# Wire Protocol & Data Lifecycle Specification

> **Status:** Approved — all open questions resolved with definitive design decisions
> **Resolves:** GAP-ANALYSIS.md findings C2, H1, H3, H5, H6, H22, H23
> **Supersedes:** RESEARCH-wire-protocol-lifecycle.md (which remains as historical context)
> **Owner:** Principal Distributed Systems Engineer
> **Last updated:** 2026-03-21

---

## Table of Contents

1. [Wire Protocol Versioning (C2)](#1-wire-protocol-versioning-c2)
   - 1.1 Temporal Activity Versioning
   - 1.2 NATS Event Envelope Versioning
   - 1.3 Adapter gRPC Protocol Versioning
   - 1.4 Edge Agent Sync Versioning
2. [Temporal-PostgreSQL Shared Fate (H1)](#2-temporal-postgresql-shared-fate-h1)
3. [ObservedFact Retention (H3)](#3-observedfact-retention-h3)
4. [Temporal continue-as-new (H5)](#4-temporal-continue-as-new-h5)
5. [Idempotency Cache (H6)](#5-idempotency-cache-h6)
6. [Schedule Risk Recommendations (H22)](#6-schedule-risk-recommendations-h22)
7. [Temporal GetVersion Integration (H23)](#7-temporal-getversion-integration-h23)

---

## 1. Wire Protocol Versioning (C2)

LOOM has four wire protocol boundaries that require versioning contracts. The REST API (`/api/v1/`) is already versioned and excluded from this specification.

| Boundary | Transport | Versioning Mechanism | Backward Compat |
|----------|-----------|---------------------|-----------------|
| Temporal activities | Temporal SDK | `workflow.GetVersion()` | Retain until zero open workflows |
| NATS events | NATS JetStream | `EventEnvelope.SchemaID` | Additive-only, unknown = skip |
| Adapter plugins | gRPC (go-plugin) | Handshake min/max negotiation | N-1 versions |
| Edge agent sync | NATS + custom | `SyncEnvelope.ProtocolVersion` | N-1 versions |

### 1.1 Temporal Activity Versioning

**Decision:** `workflow.GetVersion()` is mandatory for every workflow change that alters activity execution, input structs, output structs, or step ordering. This is the most critical versioning boundary because Temporal replays workflow execution from event history; a struct change without `GetVersion` breaks replay of in-flight workflows.

#### 1.1.1 Activity Struct Evolution Rules

```go
// Rule 1: Every activity input/output struct carries a version annotation.
// ActivityVersion: 2
type PowerCycleParams struct {
    DeviceID    uuid.UUID     `json:"device_id"`              // v1: original
    TenantID    uuid.UUID     `json:"tenant_id"`              // v1: original
    WaitTime    time.Duration `json:"wait_time"`              // v1: original
    Timeout     time.Duration `json:"timeout,omitempty"`      // v2: added (default: 5m)
    ForceReset  bool          `json:"force_reset,omitempty"`  // v2: added (default: false)
    // Deprecated: ShutdownMode was renamed to PowerOffMethod in v3.
    // Retained for deserialization of in-flight workflows.
    ShutdownMode string       `json:"shutdown_mode,omitempty"` // v1: deprecated in v3
    PowerOffMethod string     `json:"power_off_method,omitempty"` // v3: replaces ShutdownMode
}
```

**Evolution rules (enforced by CI lint):**

| Allowed | Forbidden |
|---------|-----------|
| Add new field with zero-value default | Remove a field |
| Add new field with `omitempty` | Rename a field |
| Deprecate a field (keep it, add replacement) | Change a field's type |
| Add a new enum value to a string field | Reorder fields that affect serialization |

#### 1.1.2 `workflow.GetVersion()` Usage Rules

```go
func ProvisioningWorkflow(ctx workflow.Context, input ProvisioningInput) error {
    // RULE: Every version branch uses a descriptive, unique change ID.
    // Convention: "{verb}-{noun}-{context}" in kebab-case.
    // Good: "add-health-check-before-provisioning"
    // Bad:  "v2", "fix", "update", "change1"

    v := workflow.GetVersion(ctx, "add-health-check-before-provisioning",
        workflow.DefaultVersion, 1)

    if v >= 1 {
        // v1+: Run pre-provisioning health check
        var healthResult HealthCheckResult
        err := workflow.ExecuteActivity(ctx, PreProvisioningHealthCheck,
            HealthCheckParams{DeviceID: input.DeviceID}).Get(ctx, &healthResult)
        if err != nil {
            return fmt.Errorf("pre-provisioning health check: %w", err)
        }
    }

    // Original provisioning logic (runs for all versions)
    var provResult ProvisioningResult
    err := workflow.ExecuteActivity(ctx, ProvisionDevice,
        input.ProvisionParams).Get(ctx, &provResult)
    if err != nil {
        return err
    }

    // RULE: Multiple versions can coexist in the same workflow.
    v2 := workflow.GetVersion(ctx, "add-post-provision-verification-retry",
        workflow.DefaultVersion, 2)

    if v2 == workflow.DefaultVersion {
        // Original verification (single attempt)
        return workflow.ExecuteActivity(ctx, VerifyProvisioning,
            input.ExpectedOutcome).Get(ctx, nil)
    } else if v2 == 1 {
        // v1: verification with retry
        return verifyWithRetry(ctx, input.ExpectedOutcome, 3)
    } else {
        // v2: verification with exponential backoff retry
        return verifyWithExponentialBackoff(ctx, input.ExpectedOutcome, 5)
    }
}
```

#### 1.1.3 Version Change ID Naming Convention

Format: `{verb}-{noun}-{context}`

| Pattern | Example | When to Use |
|---------|---------|-------------|
| `add-{feature}` | `add-health-check-before-provisioning` | New activity or step added |
| `change-{param}-in-{activity}` | `change-timeout-in-power-cycle` | Activity parameter semantics changed |
| `replace-{old}-with-{new}` | `replace-single-verify-with-retry` | Activity replaced with different implementation |
| `remove-{step}-from-{workflow}` | `remove-legacy-dns-check-from-discovery` | Step removed (old path becomes no-op) |
| `reorder-{steps}-in-{workflow}` | `reorder-verify-before-register-in-provisioning` | Step execution order changed |

#### 1.1.4 Old Code Path Retention Policy

**Decision:** Old code paths are retained until zero open workflow executions reference that version. Cleanup is automated.

```go
// VersionCleanupQuery checks Temporal visibility for open workflows
// that might replay through a given version branch.
//
// Run this as a periodic cron (daily) or before a release that removes old paths.
func CanRemoveVersion(ctx context.Context, client client.Client,
    workflowType string, changeID string, maxVersion int) (bool, error) {

    // Query: are there any open workflows of this type started before the
    // version was introduced?
    query := fmt.Sprintf(
        `WorkflowType = '%s' AND ExecutionStatus = 'Running' AND StartTime < '%s'`,
        workflowType, versionIntroductionDate(changeID),
    )

    resp, err := client.ListWorkflow(ctx, &workflowservice.ListWorkflowExecutionsRequest{
        Query: query,
    })
    if err != nil {
        return false, fmt.Errorf("query open workflows: %w", err)
    }

    return len(resp.Executions) == 0, nil
}
```

**Retention rules:**

| Scenario | Retention |
|----------|-----------|
| Normal workflows (provisioning, power-cycle) | Max workflow duration is 24 hours. Old paths safe to remove 7 days after version deployment. |
| Long-running workflows (discovery, monitoring) | These use `continue-as-new` (Section 4). Max single execution is bounded by checkpoint threshold. Old paths safe to remove 7 days after last execution with old version completes. |
| Approval-gated workflows | Approval timeout is configurable (max 30 days per WORKFLOW-CONTRACT.md). Old paths retained for 37 days after version deployment. |

#### 1.1.5 CI Enforcement

```go
// Package loomvet implements a go/analysis-based linter that detects
// activity struct changes without corresponding GetVersion usage.
//
// Install: go install github.com/wittedinit/loom/tools/loomvet@latest
// Usage:  loomvet ./...
// CI:     Added to .golangci.yml as custom linter

// Detection rules:
// 1. Any file in internal/workflow/ or internal/activity/ that modifies a
//    struct tagged with "// ActivityVersion:" must also contain a
//    workflow.GetVersion() call with a new change ID not seen in the
//    previous commit.
//
// 2. Any PR that adds/removes/reorders activities in a workflow function
//    must include a corresponding GetVersion branch.
//
// 3. Any PR that changes an activity function signature triggers a
//    blocking warning requiring explicit acknowledgment.

// Implementation: AST analysis comparing the PR diff against the base branch.
// The linter examines:
//   - Struct definitions with "// ActivityVersion:" comments
//   - Function signatures matching the Temporal activity registration pattern
//   - workflow.GetVersion() call sites and their change IDs
```

**`.golangci.yml` addition:**

```yaml
linters-settings:
  govet:
    enable:
      - loomvet  # Custom analyzer for Temporal version safety

# CI pipeline step (GitHub Actions):
# - name: Temporal Version Safety Check
#   run: go vet -vettool=$(which loomvet) ./internal/workflow/... ./internal/activity/...
```

#### 1.1.6 Workflow Version History Documentation Template

Every workflow file must contain a version history block at the top of the workflow function:

```go
// ProvisioningWorkflow orchestrates end-to-end device provisioning.
//
// Version History:
// ┌─────────┬────────────┬─────────────────────────────────────────────┬───────────┐
// │ Version │ Date       │ Change ID                                   │ Status    │
// ├─────────┼────────────┼─────────────────────────────────────────────┼───────────┤
// │ Default │ 2026-04-01 │ (initial implementation)                    │ Drainable │
// │ 1       │ 2026-04-15 │ add-health-check-before-provisioning        │ Active    │
// │ 2       │ 2026-05-01 │ add-post-provision-verification-retry       │ Active    │
// │ 3       │ 2026-06-12 │ replace-single-verify-with-exponential      │ Active    │
// └─────────┴────────────┴─────────────────────────────────────────────┴───────────┘
//
// Drainable = no open workflows reference this version; old code path can be removed.
// Active    = open workflows may replay through this version; code path must be retained.
func ProvisioningWorkflow(ctx workflow.Context, input ProvisioningInput) error {
    // ...
}
```

---

### 1.2 NATS Event Envelope Versioning

**Decision:** JSON envelopes with `SchemaID`-based versioning. Protobuf is deferred to post-MVP; JSON provides superior debuggability during development and the performance difference is negligible at LOOM's event volumes (<10K events/second).

#### 1.2.1 Versioned Event Envelope

```go
package event

import (
    "encoding/json"
    "time"
)

// EventEnvelope is the wire format for all LOOM events on NATS.
// Version history:
//   EnvelopeVersion 1: initial (2026-04-01)
type EventEnvelope struct {
    // EnvelopeVersion is the version of the envelope structure itself.
    // Envelope changes are additive-only. Start at 1.
    EnvelopeVersion int `json:"envelope_version"`

    // SchemaID identifies the payload schema.
    // Format: "{domain}.{action}.v{N}" (e.g., "device.created.v1")
    // Consumers use SchemaID to select the correct deserialization struct.
    SchemaID string `json:"schema_id"`

    // Standard envelope fields (from EVENT-MODEL.md Event struct)
    ID            string          `json:"id"`
    Timestamp     time.Time       `json:"timestamp"`
    TenantID      string          `json:"tenant_id"`
    Actor         Actor           `json:"actor"`
    Action        string          `json:"action"`
    ResourceType  string          `json:"resource_type"`
    ResourceID    string          `json:"resource_id"`
    CorrelationID string          `json:"correlation_id"`
    CausationID   string          `json:"causation_id"`

    // Data is the schema-versioned payload. Consumers deserialize based on SchemaID.
    Data json.RawMessage `json:"data"`
}
```

#### 1.2.2 Schema Registry

```go
package event

import (
    "encoding/json"
    "fmt"
    "sync"
)

// SchemaRegistry maps SchemaID strings to Go struct types for deserialization.
// This is an in-code registry, not an external service. Schema definitions
// live alongside their Go struct definitions.
type SchemaRegistry struct {
    mu      sync.RWMutex
    schemas map[string]SchemaEntry
}

// SchemaEntry holds the metadata and factory for a schema version.
type SchemaEntry struct {
    SchemaID    string                    // "device.created.v1"
    GoType      string                    // "event.DeviceCreatedDataV1" (for logging)
    Factory     func() interface{}        // Returns a zero-value instance for deserialization
    Deprecated  bool                      // True if this version is superseded
    SuccessorID string                    // SchemaID of the replacement, if deprecated
}

// Global registry, populated at init time.
var Registry = &SchemaRegistry{
    schemas: make(map[string]SchemaEntry),
}

// Register adds a schema to the registry. Called from init() functions
// in each event payload package.
func (r *SchemaRegistry) Register(entry SchemaEntry) {
    r.mu.Lock()
    defer r.mu.Unlock()
    r.schemas[entry.SchemaID] = entry
}

// Decode deserializes an EventEnvelope's Data field into the registered type.
// Returns (nil, nil) if the SchemaID is unknown — consumers must handle this
// by logging and skipping, never crashing.
func (r *SchemaRegistry) Decode(env EventEnvelope) (interface{}, error) {
    r.mu.RLock()
    entry, ok := r.schemas[env.SchemaID]
    r.mu.RUnlock()

    if !ok {
        // Unknown schema: return nil, no error.
        // Consumer is responsible for logging and skipping.
        return nil, nil
    }

    target := entry.Factory()
    if err := json.Unmarshal(env.Data, target); err != nil {
        return nil, fmt.Errorf("decode schema %s: %w", env.SchemaID, err)
    }
    return target, nil
}

// Registration example (in event payload file):
func init() {
    Registry.Register(SchemaEntry{
        SchemaID: "device.created.v1",
        GoType:   "event.DeviceCreatedDataV1",
        Factory:  func() interface{} { return &DeviceCreatedDataV1{} },
    })
    Registry.Register(SchemaEntry{
        SchemaID:    "device.created.v2",
        GoType:      "event.DeviceCreatedDataV2",
        Factory:     func() interface{} { return &DeviceCreatedDataV2{} },
        Deprecated:  false,
    })
}
```

#### 1.2.3 Consumer Rules

```go
// ConsumeEvent is the standard event consumption pattern.
// RULE: Unknown SchemaID -> log + skip, NEVER crash.
// RULE: Deprecated SchemaID -> process normally, emit metric.
func ConsumeEvent(env EventEnvelope) error {
    payload, err := Registry.Decode(env)
    if err != nil {
        // Deserialization failure: log, send to dead-letter, do not crash.
        slog.Error("event deserialization failed",
            "schema_id", env.SchemaID,
            "event_id", env.ID,
            "error", err,
        )
        return err // NACK for redelivery or dead-letter
    }

    if payload == nil {
        // Unknown schema: this consumer was deployed before the producer.
        // Log at INFO (not ERROR — this is expected during rolling deployments).
        slog.Info("skipping event with unknown schema",
            "schema_id", env.SchemaID,
            "event_id", env.ID,
        )
        return nil // ACK — do not block the consumer
    }

    // Type-switch on the concrete payload type.
    switch p := payload.(type) {
    case *DeviceCreatedDataV1:
        return handleDeviceCreatedV1(env, p)
    case *DeviceCreatedDataV2:
        return handleDeviceCreatedV2(env, p)
    default:
        // Known schema, but this consumer does not handle it.
        slog.Debug("ignoring event not handled by this consumer",
            "schema_id", env.SchemaID,
        )
        return nil // ACK
    }
}
```

#### 1.2.4 Schema Evolution Rules

| Rule | Detail |
|------|--------|
| New payload version = new SchemaID | `device.created.v1` and `device.created.v2` are separate schemas |
| Old versions published concurrently during rolling deploy | Producers emit the newest version they know; consumers handle both |
| Deprecated schemas retained for 90 days | Matches NATS stream retention (EVENT-MODEL.md) |
| Envelope fields are additive-only | New envelope fields get zero-value defaults; never remove/rename |
| SchemaID format is stable | `{domain}.{action}.v{N}` — never change the format itself |

---

### 1.3 Adapter gRPC Protocol Versioning

**Decision:** Integer-based protocol versioning (not semver). Semver is overkill for a plugin boundary where the host controls both sides of the negotiation. Integers are simpler to compare and sufficient for the backward-compatibility guarantee.

#### 1.3.1 Handshake Protocol

```protobuf
syntax = "proto3";
package loom.adapter.v1;

// AdapterPlugin is the gRPC service exposed by every adapter plugin process.
// The host (LOOM core) connects to the plugin and calls Handshake first.
service AdapterPlugin {
    // Handshake negotiates protocol version and declares capabilities.
    // MUST be the first RPC call after connection.
    // If negotiation fails, the host disconnects and logs an error.
    rpc Handshake(HandshakeRequest) returns (HandshakeResponse);

    // Connect establishes a session to the target device.
    rpc Connect(ConnectRequest) returns (ConnectResponse);

    // Disconnect tears down the device session.
    rpc Disconnect(DisconnectRequest) returns (DisconnectResponse);

    // Ping checks liveness of the device session.
    rpc Ping(PingRequest) returns (PingResponse);

    // Discover collects facts about the target.
    rpc Discover(DiscoverRequest) returns (DiscoverResponse);

    // Execute performs a typed operation.
    rpc Execute(ExecuteRequest) returns (ExecuteResponse);

    // ReadState returns a point-in-time state snapshot.
    rpc ReadState(ReadStateRequest) returns (ReadStateResponse);

    // Watch streams state changes (only for adapters that support it).
    rpc Watch(WatchRequest) returns (stream WatchEvent);
}

message HandshakeRequest {
    // min_protocol_version is the oldest protocol version the host supports.
    int32 min_protocol_version = 1;

    // max_protocol_version is the newest protocol version the host supports.
    int32 max_protocol_version = 2;

    // host_version is the LOOM core version string (informational).
    string host_version = 3;
}

message HandshakeResponse {
    // negotiated_version is the protocol version both sides will use.
    // Must be: min_protocol_version <= negotiated_version <= max_protocol_version.
    // If the plugin cannot satisfy this range, it returns an error status.
    int32 negotiated_version = 1;

    // capabilities declares what this adapter can do.
    // The host uses this to route operations to the correct adapter.
    repeated AdapterCapability capabilities = 2;

    // adapter_name is the human-readable name of this adapter (e.g., "redfish-dell-idrac").
    string adapter_name = 3;

    // adapter_version is the semver of the adapter binary (informational).
    string adapter_version = 4;
}

message AdapterCapability {
    string name = 1;        // "power_control", "discover_hardware", etc.
    string protocol = 2;    // "redfish", "ssh", "snmp", etc.
    bool read_only = 3;
}
```

#### 1.3.2 Version Negotiation Logic (Host Side)

```go
package adapter

import (
    "fmt"

    pb "github.com/wittedinit/loom/internal/adapter/proto"
)

const (
    // CurrentProtocolVersion is the latest protocol version the host supports.
    CurrentProtocolVersion = 1

    // MinSupportedProtocolVersion is the oldest version the host can work with.
    // This provides N-1 backward compatibility.
    MinSupportedProtocolVersion = 1
)

// NegotiateVersion performs version handshake with a plugin.
// Returns the negotiated version or an error if incompatible.
func NegotiateVersion(client pb.AdapterPluginClient) (int32, []string, error) {
    resp, err := client.Handshake(ctx, &pb.HandshakeRequest{
        MinProtocolVersion: MinSupportedProtocolVersion,
        MaxProtocolVersion: CurrentProtocolVersion,
        HostVersion:        BuildVersion,
    })
    if err != nil {
        return 0, nil, fmt.Errorf("handshake failed: %w", err)
    }

    v := resp.NegotiatedVersion
    if v < MinSupportedProtocolVersion || v > CurrentProtocolVersion {
        return 0, nil, fmt.Errorf(
            "plugin negotiated version %d outside host range [%d, %d]",
            v, MinSupportedProtocolVersion, CurrentProtocolVersion,
        )
    }

    caps := make([]string, len(resp.Capabilities))
    for i, c := range resp.Capabilities {
        caps[i] = c.Name
    }

    slog.Info("adapter handshake complete",
        "adapter", resp.AdapterName,
        "adapter_version", resp.AdapterVersion,
        "negotiated_protocol", v,
        "capabilities", caps,
    )

    return v, caps, nil
}
```

#### 1.3.3 Version Bump Criteria

A protocol version bump (incrementing `CurrentProtocolVersion`) is required when:

| Change | Version Bump Required? |
|--------|----------------------|
| Add a new optional field to an existing request/response message | No (protobuf is additive-safe) |
| Add a new RPC method to the service | No (host does not call methods it does not know about) |
| Remove or rename an RPC method | **Yes** |
| Change the semantics of an existing field | **Yes** |
| Change the type of an existing field | **Yes** |
| Add a new required field (field that the host always sends and expects the plugin to handle) | **Yes** |
| Change error code semantics | **Yes** |

#### 1.3.4 Backward Compatibility Contract

- The host maintains compatibility with protocol version `CurrentProtocolVersion` and `CurrentProtocolVersion - 1`.
- When `CurrentProtocolVersion` is bumped, the host retains the old code path for one release cycle.
- Adapters compiled against `CurrentProtocolVersion - 2` or older are rejected at handshake.
- The host logs a warning when a plugin negotiates `CurrentProtocolVersion - 1` to signal upgrade needed.

---

### 1.4 Edge Agent Sync Versioning

**Decision:** Agent-to-hub communication uses a versioned sync envelope. The hub maintains N-1 backward compatibility and rejects agents below the minimum supported version, directing them to auto-update.

#### 1.4.1 Sync Envelope

```go
package edge

import (
    "encoding/json"
    "time"
)

// SyncEnvelope wraps all agent-to-hub and hub-to-agent messages.
type SyncEnvelope struct {
    // AgentVersion is the semver of the agent binary (e.g., "1.2.3").
    AgentVersion string `json:"agent_version"`

    // ProtocolVersion is the sync protocol version (integer, starts at 1).
    ProtocolVersion int `json:"protocol_version"`

    // HubMinVersion is the minimum protocol version the hub accepts.
    // Set by the hub in responses; agents read this to detect incompatibility.
    HubMinVersion int `json:"hub_min_version,omitempty"`

    // MessageType identifies the payload type.
    // Agent->Hub: "heartbeat", "fact_report", "status_update", "log_batch"
    // Hub->Agent: "config_push", "command", "update_directive", "ack"
    MessageType string `json:"message_type"`

    // Payload is the message-type-specific data.
    Payload json.RawMessage `json:"payload"`

    // Timestamp is when the message was created.
    Timestamp time.Time `json:"timestamp"`

    // AgentID uniquely identifies this agent instance.
    AgentID string `json:"agent_id"`

    // TenantID scopes this message to a tenant.
    TenantID string `json:"tenant_id"`
}

const (
    // CurrentSyncProtocolVersion is the latest sync protocol the hub supports.
    CurrentSyncProtocolVersion = 1

    // MinSyncProtocolVersion is the oldest version the hub accepts.
    MinSyncProtocolVersion = 1
)
```

#### 1.4.2 Hub Rejection and Auto-Update Flow

```go
package edge

import "fmt"

// SyncRejection is sent by the hub when an agent's protocol version is too old.
type SyncRejection struct {
    Reason          string `json:"reason"`
    HubMinVersion   int    `json:"hub_min_version"`
    AgentVersion    string `json:"agent_version"`
    UpdateURL       string `json:"update_url"`       // URL to download the new agent binary
    UpdateChecksum  string `json:"update_checksum"`   // SHA-256 of the new binary
    ForceUpdate     bool   `json:"force_update"`      // If true, agent must update before reconnecting
}

// ValidateAgent checks whether an agent is compatible with this hub.
func ValidateAgent(env SyncEnvelope) (*SyncRejection, error) {
    if env.ProtocolVersion < MinSyncProtocolVersion {
        return &SyncRejection{
            Reason: fmt.Sprintf(
                "agent protocol version %d below minimum %d",
                env.ProtocolVersion, MinSyncProtocolVersion,
            ),
            HubMinVersion:  MinSyncProtocolVersion,
            AgentVersion:   env.AgentVersion,
            UpdateURL:      fmt.Sprintf("/api/v1/agents/update/%s", latestAgentVersion()),
            UpdateChecksum: latestAgentChecksum(),
            ForceUpdate:    true,
        }, nil
    }

    if env.ProtocolVersion < CurrentSyncProtocolVersion {
        // Accepted but deprecated — log warning, include upgrade hint in response.
        slog.Warn("agent using deprecated protocol version",
            "agent_id", env.AgentID,
            "protocol_version", env.ProtocolVersion,
            "current_version", CurrentSyncProtocolVersion,
        )
    }

    return nil, nil // Accepted
}
```

#### 1.4.3 Store-and-Forward Compatibility

When the agent buffers messages locally (SQLite) during disconnection, the buffered messages carry the `ProtocolVersion` that was current when they were created. The hub must accept messages at any version >= `MinSyncProtocolVersion`, even if they were buffered days ago. This means:

- Hub-side deserialization handles all versions from `MinSyncProtocolVersion` to `CurrentSyncProtocolVersion`.
- Before bumping `MinSyncProtocolVersion`, the operator must ensure all edge agents have drained their local buffers.
- The hub exposes a metric `loom_edge_agent_protocol_version{agent_id}` to track fleet version distribution.

---

## 2. Temporal-PostgreSQL Shared Fate (H1)

### Decision

**Mode 1-2 (single site, <1,000 devices):** Shared PostgreSQL instance with separate databases and isolated connection pools. **Mode 3-4 (multi-region, >1,000 devices):** Separate PostgreSQL instances.

### 2.1 Connection Pool Specification

```yaml
# PostgreSQL instance: max_connections = 100
#
# Pool allocation:
#   Temporal Server:  20 connections (separate database: "temporal")
#   LOOM Application: 60 connections (database: "loom")
#   Reserved:         10 connections (monitoring, migrations, ad-hoc admin)
#   Headroom:         10 connections (burst capacity)

# LOOM application pgxpool config:
loom_pool:
  max_connections: 60
  min_connections: 10
  max_conn_idle_time: 5m
  max_conn_lifetime: 30m
  health_check_period: 30s

# Temporal Server PostgreSQL config (temporal.yaml):
temporal_pool:
  max_connections: 20
  # Temporal manages its own pool internally; these are passed
  # via the Temporal server config sql.maxConns / sql.maxIdleConns.
```

### 2.2 Database Separation

Temporal uses a separate PostgreSQL **database** (not just schema) on the same instance:

```sql
-- Instance setup (run once)
CREATE DATABASE temporal;
CREATE DATABASE temporal_visibility;
CREATE DATABASE loom;

-- Separate roles with connection limits
CREATE ROLE temporal_user WITH LOGIN PASSWORD '...' CONNECTION LIMIT 20;
CREATE ROLE loom_user WITH LOGIN PASSWORD '...' CONNECTION LIMIT 60;

GRANT ALL ON DATABASE temporal TO temporal_user;
GRANT ALL ON DATABASE temporal_visibility TO temporal_user;
GRANT ALL ON DATABASE loom TO loom_user;
```

**Why separate databases (not schemas):**
- Independent `pg_dump` / `pg_restore` for each database.
- Temporal's schema migrations cannot interfere with LOOM's tables.
- Temporal can be migrated to a separate instance later by changing only the connection string.
- Separate databases enable separate WAL archiving policies if needed.

### 2.3 Migration Safety

LOOM migrations must not hold locks that block Temporal operations:

| Rule | Detail |
|------|--------|
| No long-running `ALTER TABLE` on hot tables | Use `CREATE INDEX CONCURRENTLY`, `ALTER TABLE ... ADD COLUMN` (no default rewrite in PG 11+) |
| No `LOCK TABLE ... EXCLUSIVE` | Use advisory locks for coordination |
| Migration timeout | Set `statement_timeout = '30s'` in migration session |
| Test in staging first | Migrations run against staging with Temporal workflows active before production |
| Maintenance window for DDL | Schema changes that require `ACCESS EXCLUSIVE` (column type change, column drop) are scheduled during maintenance windows |

### 2.4 Monitoring

```yaml
# Prometheus alerts for connection pool health
groups:
  - name: loom_pg_pool
    rules:
      - alert: LOOMPoolNearCapacity
        expr: pgxpool_acquired_conns{pool="loom"} / pgxpool_max_conns{pool="loom"} > 0.8
        for: 5m
        annotations:
          summary: "LOOM connection pool at >80% capacity"

      - alert: TemporalPoolNearCapacity
        expr: sql_open_connections{database="temporal"} / 20 > 0.8
        for: 5m
        annotations:
          summary: "Temporal connection pool at >80% capacity"

      - alert: PostgreSQLTotalConnectionsHigh
        expr: pg_stat_activity_count > 85
        for: 2m
        annotations:
          summary: "PostgreSQL approaching max_connections (100)"
```

### 2.5 Mode 3-4 Separation

When scaling beyond 1,000 devices or deploying multi-region:

- Temporal Server gets its own PostgreSQL instance (or migrates to Cassandra for global deployment).
- LOOM application database remains independent.
- The transition requires only connection string changes in Temporal's config and LOOM's config.
- No code changes are needed because the databases are already separate.

---

## 3. ObservedFact Retention (H3)

### Decision

**TimescaleDB hypertable from day one.** The cost of retrofitting time-series partitioning onto an existing table with billions of rows is far greater than the cost of installing the TimescaleDB extension in Phase 1B. TimescaleDB is already in `docker-compose.yml`.

### 3.1 Retention Tiers

| Tier | Duration | Storage Format | Use Case | Estimated Storage (10K devices) |
|------|----------|---------------|----------|-------------------------------|
| Hot | 7 days | Uncompressed hypertable chunks | Real-time queries, drift detection, current state | ~70 GB |
| Warm | 90 days | TimescaleDB native compression (~10:1) | Trend analysis, historical comparison | ~90 GB (compressed) |
| Cold | 1 year | Continuous aggregate (hourly summaries) | Compliance audit, long-term trends | ~365 MB |
| Archive | 7 years | Parquet on object storage (monthly partitions) | Regulatory compliance, forensics | ~50 MB/month |

### 3.2 Hypertable Creation

```sql
-- Step 1: Create the observed_facts table
CREATE TABLE observed_facts (
    id              UUID        NOT NULL DEFAULT gen_random_uuid(),
    device_id       UUID        NOT NULL,
    endpoint_id     UUID        NOT NULL,
    fact_type       TEXT        NOT NULL,
    value           TEXT        NOT NULL,
    confidence      FLOAT8      NOT NULL DEFAULT 1.0,
    source          TEXT        NOT NULL,
    observed_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
    tenant_id       UUID        NOT NULL,

    -- TimescaleDB requires the partitioning column in any unique constraint.
    -- We use (observed_at, id) as the primary key.
    PRIMARY KEY (observed_at, id)
);

-- Step 2: Convert to hypertable
-- chunk_time_interval = 1 day (good for 10K devices writing ~144M rows/day)
SELECT create_hypertable(
    'observed_facts',
    'observed_at',
    chunk_time_interval => INTERVAL '1 day',
    migrate_data => true
);

-- Step 3: Indexes for common query patterns
CREATE INDEX idx_facts_device_type ON observed_facts (device_id, fact_type, observed_at DESC);
CREATE INDEX idx_facts_tenant ON observed_facts (tenant_id, observed_at DESC);
CREATE INDEX idx_facts_source ON observed_facts (source, observed_at DESC);
```

### 3.3 Compression Policy

```sql
-- Enable compression on chunks older than 7 days.
-- Compression ratio is typically 10:1 for time-series fact data.
ALTER TABLE observed_facts SET (
    timescaledb.compress,
    timescaledb.compress_segmentby = 'device_id, fact_type, tenant_id',
    timescaledb.compress_orderby = 'observed_at DESC'
);

-- Automatically compress chunks older than 7 days
SELECT add_compression_policy('observed_facts', INTERVAL '7 days');
```

### 3.4 Retention Policy

```sql
-- Automatically drop raw chunks older than 90 days.
-- By this point, data has been aggregated into the continuous aggregate
-- and archived to object storage.
SELECT add_retention_policy('observed_facts', INTERVAL '90 days');
```

### 3.5 Continuous Aggregate (Cold Tier)

```sql
-- Hourly summaries retained for 1 year.
-- This is the "cold" tier: enough detail for compliance audit and trend analysis,
-- but 60x fewer rows than raw data.
CREATE MATERIALIZED VIEW observed_facts_hourly
WITH (timescaledb.continuous) AS
SELECT
    time_bucket('1 hour', observed_at) AS bucket,
    device_id,
    fact_type,
    tenant_id,
    count(*)                          AS observation_count,
    -- For numeric facts (parseable to double):
    min(value)                        AS min_value,
    max(value)                        AS max_value,
    -- For non-numeric facts, keep the latest value:
    last(value, observed_at)          AS latest_value,
    last(source, observed_at)         AS latest_source,
    max(confidence)                   AS max_confidence
FROM observed_facts
GROUP BY bucket, device_id, fact_type, tenant_id;

-- Refresh policy: materialize data older than 2 hours (allow for late-arriving data)
SELECT add_continuous_aggregate_policy('observed_facts_hourly',
    start_offset    => INTERVAL '1 year',
    end_offset      => INTERVAL '2 hours',
    schedule_interval => INTERVAL '1 hour'
);

-- Retain hourly summaries for 1 year
SELECT add_retention_policy('observed_facts_hourly', INTERVAL '1 year');
```

### 3.6 Latest Facts View

```sql
-- latest_facts: a continuous aggregate that provides the most recent fact
-- for each (device_id, fact_type) pair. Refreshed every 60 seconds.
-- This replaces the materialized view from RESEARCH-wire-protocol-lifecycle.md
-- with a TimescaleDB continuous aggregate for automatic refresh.
CREATE MATERIALIZED VIEW latest_facts
WITH (timescaledb.continuous) AS
SELECT
    time_bucket('1 minute', observed_at)  AS bucket,
    device_id,
    fact_type,
    tenant_id,
    last(value, observed_at)              AS value,
    last(confidence, observed_at)         AS confidence,
    last(source, observed_at)             AS source,
    max(observed_at)                      AS observed_at
FROM observed_facts
GROUP BY bucket, device_id, fact_type, tenant_id;

-- Refresh every 60 seconds with a 5-minute end offset for data settling.
SELECT add_continuous_aggregate_policy('latest_facts',
    start_offset    => INTERVAL '1 day',
    end_offset      => INTERVAL '5 minutes',
    schedule_interval => INTERVAL '1 minute'
);
```

**For truly real-time "current state" queries** (within the 60-second refresh window), use a direct query:

```sql
-- Real-time latest fact query (used when the continuous aggregate is stale
-- or when sub-minute freshness is required).
SELECT DISTINCT ON (device_id, fact_type)
    device_id, fact_type, value, confidence, source, observed_at
FROM observed_facts
WHERE tenant_id = $1
  AND device_id = $2
  AND observed_at > now() - INTERVAL '1 day'
ORDER BY device_id, fact_type, observed_at DESC;
```

### 3.7 Archive Export Job

```go
package retention

import (
    "context"
    "fmt"
    "time"
)

// ArchiveJob exports old continuous aggregate data to Parquet files
// on object storage. Runs monthly via a cron-scheduled Temporal workflow.
type ArchiveJob struct {
    DB            *pgxpool.Pool
    ObjectStorage ObjectStorage
}

// ExportMonth exports one month of hourly summaries to Parquet.
// The Parquet file is partitioned by tenant and keyed by year-month.
func (j *ArchiveJob) ExportMonth(ctx context.Context, year int, month time.Month) error {
    start := time.Date(year, month, 1, 0, 0, 0, 0, time.UTC)
    end := start.AddDate(0, 1, 0)

    // Query hourly summaries for the month
    rows, err := j.DB.Query(ctx, `
        SELECT bucket, device_id, fact_type, tenant_id,
               observation_count, min_value, max_value,
               latest_value, latest_source, max_confidence
        FROM observed_facts_hourly
        WHERE bucket >= $1 AND bucket < $2
        ORDER BY tenant_id, bucket
    `, start, end)
    if err != nil {
        return fmt.Errorf("query hourly summaries: %w", err)
    }
    defer rows.Close()

    // Write to Parquet (using apache/arrow-go or parquet-go)
    key := fmt.Sprintf("observed-facts/%d/%02d/facts.parquet", year, month)
    writer, err := j.ObjectStorage.NewParquetWriter(ctx, key)
    if err != nil {
        return fmt.Errorf("create parquet writer: %w", err)
    }

    for rows.Next() {
        var record HourlyFactRecord
        if err := rows.Scan(
            &record.Bucket, &record.DeviceID, &record.FactType,
            &record.TenantID, &record.Count, &record.MinValue,
            &record.MaxValue, &record.LatestValue, &record.LatestSource,
            &record.MaxConfidence,
        ); err != nil {
            return fmt.Errorf("scan row: %w", err)
        }
        if err := writer.Write(record); err != nil {
            return fmt.Errorf("write parquet record: %w", err)
        }
    }

    return writer.Close()
}
```

### 3.8 Per-Tenant Configurable Retention

**Decision:** Tenants may configure retention within guardrails.

```sql
-- Tenant retention overrides (stored in the LOOM application database)
CREATE TABLE tenant_retention_config (
    tenant_id       UUID        PRIMARY KEY REFERENCES tenants(id),
    hot_days        INT         NOT NULL DEFAULT 7   CHECK (hot_days >= 7),
    warm_days       INT         NOT NULL DEFAULT 90  CHECK (warm_days >= 30),
    cold_days       INT         NOT NULL DEFAULT 365 CHECK (cold_days >= 90),
    archive_years   INT         NOT NULL DEFAULT 7   CHECK (archive_years >= 1),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Notes:
-- - Minimum hot retention: 7 days (required for drift detection)
-- - Maximum: unlimited (tenant pays for storage)
-- - The global TimescaleDB policies enforce the MINIMUM retention.
--   Tenants requesting longer retention: their data is simply not deleted
--   when the global policy runs, because a per-tenant filter is applied.
-- - Implementation: the retention job (Temporal cron workflow) reads
--   tenant_retention_config and issues DELETE queries per tenant,
--   respecting each tenant's configured retention.
```

---

## 4. Temporal `continue-as-new` (H5)

### Decision

**Checkpoint at 10,000 events** (configurable per workflow type). This is well under Temporal's ~50K hard limit, providing a 5x safety margin for event spikes caused by signal bursts or retry storms.

### 4.1 WorkflowCheckpoint Struct

```go
package workflow

import (
    "encoding/json"
    "time"

    "go.temporal.io/sdk/workflow"
)

// WorkflowCheckpoint carries all state needed to resume a workflow
// after continue-as-new. This struct is the input to the new execution.
type WorkflowCheckpoint struct {
    // CorrelationID is preserved across all continuations for audit trail continuity.
    CorrelationID string `json:"correlation_id"`

    // OriginalRunID is the RunID of the very first execution in this chain.
    // Never changes across continuations. Used for audit trail linkage.
    OriginalRunID string `json:"original_run_id"`

    // ContinueCount tracks how many times this workflow has continued.
    ContinueCount int `json:"continue_count"`

    // CheckpointedAt is when the checkpoint was created.
    CheckpointedAt time.Time `json:"checkpointed_at"`

    // PendingSignals holds signals that were received but not yet processed
    // at the time of checkpoint. These are re-processed after continue-as-new.
    PendingSignals []PendingSignal `json:"pending_signals,omitempty"`

    // ActiveTimers holds timer specifications that need to be re-registered
    // after continue-as-new.
    ActiveTimers []ActiveTimer `json:"active_timers,omitempty"`

    // Progress holds workflow-type-specific state. Opaque to the checkpoint
    // framework; each workflow type defines its own progress struct.
    Progress json.RawMessage `json:"progress"`
}

// PendingSignal is a signal that was received but not processed before checkpoint.
type PendingSignal struct {
    SignalName string          `json:"signal_name"`
    Payload    json.RawMessage `json:"payload"`
    ReceivedAt time.Time       `json:"received_at"`
}

// ActiveTimer is a timer that needs to be re-registered after continue-as-new.
type ActiveTimer struct {
    TimerID   string        `json:"timer_id"`
    Remaining time.Duration `json:"remaining"` // time left when checkpoint was taken
    Purpose   string        `json:"purpose"`   // human-readable description
}
```

### 4.2 Checkpoint Helper Function

```go
package workflow

import (
    "context"
    "encoding/json"
    "fmt"
    "time"

    "go.temporal.io/sdk/workflow"
)

// CheckpointConfig controls when continue-as-new is triggered.
type CheckpointConfig struct {
    // EventThreshold is the number of events before continue-as-new.
    // Default: 10,000. Configurable per workflow type.
    EventThreshold int

    // SignalChannelName is the name of the signal channel to drain.
    SignalChannelName string
}

// DefaultCheckpointConfig returns the default checkpoint configuration.
func DefaultCheckpointConfig() CheckpointConfig {
    return CheckpointConfig{
        EventThreshold:    10000,
        SignalChannelName: "loom-workflow-signal",
    }
}

// ShouldContinueAsNew checks whether the workflow should checkpoint and continue.
// Call this at the end of each iteration in a long-running workflow loop.
func ShouldContinueAsNew(ctx workflow.Context, config CheckpointConfig) bool {
    info := workflow.GetInfo(ctx)
    // GetInfo().GetCurrentHistoryLength() returns the number of events
    // in the current execution's history.
    return info.GetCurrentHistoryLength() >= config.EventThreshold
}

// ContinueAsNewWithCheckpoint performs the full checkpoint sequence:
// 1. Drain pending signals
// 2. Capture timer state
// 3. Update DB projection with new RunID
// 4. Execute continue-as-new
//
// This function does not return on success (Temporal terminates the current
// execution and starts a new one). On failure, it returns an error.
func ContinueAsNewWithCheckpoint(
    ctx workflow.Context,
    config CheckpointConfig,
    signalCh workflow.ReceiveChannel,
    progress json.RawMessage,
    activeTimers []ActiveTimer,
) error {
    info := workflow.GetInfo(ctx)

    // Step 1: Drain all pending signals from the channel.
    // These will be re-processed after continue-as-new.
    pendingSignals := drainSignals(ctx, signalCh)

    // Step 2: Build checkpoint.
    checkpoint := WorkflowCheckpoint{
        CorrelationID:  getCorrelationID(ctx),
        OriginalRunID:  getOriginalRunID(ctx, info),
        ContinueCount:  getContinueCount(ctx) + 1,
        CheckpointedAt: workflow.Now(ctx),
        PendingSignals: pendingSignals,
        ActiveTimers:   activeTimers,
        Progress:       progress,
    }

    // Step 3: Update DB projection with upcoming RunID change.
    // This activity runs BEFORE continue-as-new so the DB knows
    // to expect a new RunID.
    updateParams := ProjectionUpdate{
        WorkflowID:    info.WorkflowExecution.ID,
        ContinueCount: checkpoint.ContinueCount,
        CorrelationID: checkpoint.CorrelationID,
        CheckpointedAt: checkpoint.CheckpointedAt,
    }

    activityCtx := workflow.WithActivityOptions(ctx, workflow.ActivityOptions{
        StartToCloseTimeout: 30 * time.Second,
        RetryPolicy: &temporal.RetryPolicy{
            MaximumAttempts: 3,
        },
    })
    err := workflow.ExecuteActivity(activityCtx,
        UpdateProjectionForContinueAsNew, updateParams).Get(ctx, nil)
    if err != nil {
        // If projection update fails, do NOT continue-as-new.
        // The workflow stays in the current execution (safe — just approaching limit).
        return fmt.Errorf("update projection before continue-as-new: %w", err)
    }

    // Step 4: Continue as new with the checkpoint as input.
    return workflow.NewContinueAsNewError(ctx, info.WorkflowType.Name, checkpoint)
}

// drainSignals reads all pending signals from the channel without blocking.
func drainSignals(ctx workflow.Context, ch workflow.ReceiveChannel) []PendingSignal {
    var pending []PendingSignal
    for {
        var raw json.RawMessage
        ok := ch.ReceiveAsync(&raw)
        if !ok {
            break
        }
        pending = append(pending, PendingSignal{
            SignalName: "loom-workflow-signal",
            Payload:    raw,
            ReceivedAt: workflow.Now(ctx),
        })
    }
    return pending
}

// reprocessSignals re-delivers signals that were pending at checkpoint time.
// Called at the start of the new execution.
func ReprocessSignals(ctx workflow.Context, pending []PendingSignal,
    handler func(ctx workflow.Context, signal json.RawMessage) error) error {
    for _, ps := range pending {
        if err := handler(ctx, ps.Payload); err != nil {
            return fmt.Errorf("reprocess signal from %s: %w",
                ps.ReceivedAt.Format(time.RFC3339), err)
        }
    }
    return nil
}

// getOriginalRunID returns the OriginalRunID, either from the checkpoint
// input (if this is a continuation) or from the current execution (if first run).
func getOriginalRunID(ctx workflow.Context, info *workflow.Info) string {
    // If the workflow was started with a checkpoint input, use its OriginalRunID.
    // Otherwise, this is the first execution — use the current RunID.
    checkpoint, ok := getCheckpointFromInput(ctx)
    if ok && checkpoint.OriginalRunID != "" {
        return checkpoint.OriginalRunID
    }
    return info.WorkflowExecution.RunID
}
```

### 4.3 DB Projection

```go
package workflow

import (
    "time"
)

// WorkflowStatusProjection tracks the latest state of a workflow
// in the PostgreSQL reporting database.
// API consumers see WorkflowID only, never RunID.
type WorkflowStatusProjection struct {
    WorkflowID      string     `json:"workflow_id"    db:"workflow_id"`
    CurrentRunID    string     `json:"-"              db:"current_run_id"`    // Internal only, never exposed via API
    OriginalRunID   string     `json:"-"              db:"original_run_id"`   // Internal only
    ContinueCount   int        `json:"-"              db:"continue_count"`    // Internal only
    TenantID        string     `json:"tenant_id"      db:"tenant_id"`
    WorkflowType    string     `json:"workflow_type"  db:"workflow_type"`
    State           string     `json:"state"          db:"state"`
    CurrentStep     string     `json:"current_step"   db:"current_step"`
    CorrelationID   string     `json:"correlation_id" db:"correlation_id"`
    StartedAt       *time.Time `json:"started_at"     db:"started_at"`
    CompletedAt     *time.Time `json:"completed_at"   db:"completed_at"`
    Error           *string    `json:"error"          db:"error"`
    UpdatedAt       time.Time  `json:"updated_at"     db:"updated_at"`
}

// ProjectionUpdate is the activity input for updating the DB projection
// when a workflow continues-as-new.
type ProjectionUpdate struct {
    WorkflowID     string    `json:"workflow_id"`
    ContinueCount  int       `json:"continue_count"`
    CorrelationID  string    `json:"correlation_id"`
    CheckpointedAt time.Time `json:"checkpointed_at"`
}

// UpdateProjectionForContinueAsNew is a Temporal activity that updates the
// workflow projection in PostgreSQL.
//
// This runs as the last activity before continue-as-new. After CAN completes,
// a callback activity in the new execution updates CurrentRunID with the actual
// new RunID (which is not known until the new execution starts).
func UpdateProjectionForContinueAsNew(ctx context.Context,
    params ProjectionUpdate) error {

    _, err := db.Exec(ctx, `
        UPDATE workflow_status
        SET continue_count = $2,
            updated_at = $3
        WHERE workflow_id = $1
    `, params.WorkflowID, params.ContinueCount, params.CheckpointedAt)

    return err
}

// UpdateRunIDAfterContinueAsNew is called at the start of the new execution
// to record the new RunID. This is a simple activity.
func UpdateRunIDAfterContinueAsNew(ctx context.Context,
    workflowID string, newRunID string, continueCount int) error {

    _, err := db.Exec(ctx, `
        UPDATE workflow_status
        SET current_run_id = $2,
            continue_count = $3,
            updated_at = now()
        WHERE workflow_id = $1
    `, workflowID, newRunID, continueCount)

    return err
}
```

### 4.4 Audit Trail Continuity

```go
package audit

// AuditMetadata is attached to every audit event emitted by a workflow.
// CorrelationID and OriginalRunID are stable across all continuations,
// providing a single thread for audit trail queries.
type AuditMetadata struct {
    WorkflowID    string `json:"workflow_id"`
    OriginalRunID string `json:"original_run_id"` // First execution's RunID, stable forever
    CurrentRunID  string `json:"current_run_id"`   // Current execution's RunID
    ContinueCount int    `json:"continue_count"`
    CorrelationID string `json:"correlation_id"`   // Same across all continuations
}

// Query for the full audit trail of a workflow, including all continuations:
//
//   SELECT * FROM audit_events
//   WHERE correlation_id = $1
//   ORDER BY timestamp ASC;
//
// This returns all events across all continuations because CorrelationID
// is preserved.
```

### 4.5 Transparency to API Consumers

**Decision:** API consumers never see RunID. The API exposes `WorkflowID` only. `continue-as-new` is an internal implementation detail.

- `GET /api/v1/workflows/{workflow_id}` returns `WorkflowStatusProjection` which omits `CurrentRunID` and `OriginalRunID` from JSON output (via `json:"-"` tags).
- `GET /api/v1/workflows/{workflow_id}/history` queries Temporal using `WorkflowID` (which is stable across continuations). Temporal's SDK handles RunID resolution automatically.
- Signal delivery targets `WorkflowID` (Temporal routes to the latest RunID).
- Cancellation targets `WorkflowID` (same routing).

### 4.6 Approval Signals Across `continue-as-new`

When a workflow is waiting for approval and approaches the event threshold:

1. The workflow does **not** continue-as-new while blocked on an approval signal. Approval-gated workflows are typically low-event (one signal wait = few history events). The 10,000 event threshold is not reachable during a single approval wait.
2. If a long-running workflow has multiple approval gates and accumulates events between them, it checkpoints **before** the next approval gate (not during the wait).
3. Pending approval signals are drained and re-delivered after continue-as-new, just like any other signal.
4. The approval service sends signals to `WorkflowID` (not `RunID`), so continue-as-new is transparent to the approval service.

---

## 5. Idempotency Cache (H6)

### Decision

**Two-tier cache: PostgreSQL (durable) + Valkey (hot).** PostgreSQL-first writes guarantee durability. Valkey provides sub-millisecond lookups for the common case. If both caches are unavailable, the system fails CLOSED (blocks operations rather than risking double-execution).

### 5.1 IdempotencyRecord

```go
package idempotency

import (
    "context"
    "encoding/json"
    "fmt"
    "time"

    "github.com/google/uuid"
)

// IdempotencyRecord stores the result of a previously executed operation.
type IdempotencyRecord struct {
    // Key is the idempotency key from the operation request.
    // Format: "{tenant_id}:{idempotency_key}" to prevent cross-tenant poisoning.
    Key string `json:"key" db:"key"`

    // TenantID is the tenant that owns this record.
    TenantID uuid.UUID `json:"tenant_id" db:"tenant_id"`

    // OperationType identifies the operation (e.g., "power_cycle", "create_vlan").
    OperationType string `json:"operation_type" db:"operation_type"`

    // Result is the serialized OperationResult from the original execution.
    Result json.RawMessage `json:"result" db:"result"`

    // Status tracks whether the operation is in progress or completed.
    // "pending" = operation started, not yet completed (prevents concurrent execution)
    // "completed" = operation finished, result available
    // "failed" = operation failed, result contains error details
    Status string `json:"status" db:"status"`

    // CreatedAt is when the record was first created.
    CreatedAt time.Time `json:"created_at" db:"created_at"`

    // ExpiresAt is when this record should be purged.
    ExpiresAt time.Time `json:"expires_at" db:"expires_at"`

    // WorkflowID is the Temporal workflow ID that executed this operation, if any.
    WorkflowID string `json:"workflow_id,omitempty" db:"workflow_id"`
}
```

### 5.2 Cache Interface

```go
package idempotency

import (
    "context"
    "time"
)

// Cache is the two-tier idempotency cache interface.
type Cache interface {
    // Check returns the cached result for a key, or nil if not found.
    // Checks Valkey first (fast path), then PostgreSQL (durable path).
    Check(ctx context.Context, tenantID uuid.UUID, key string) (*IdempotencyRecord, error)

    // Acquire attempts to claim an idempotency key for exclusive execution.
    // Returns (true, nil) if the key was successfully claimed (no prior record exists).
    // Returns (false, record) if the key already exists (operation in progress or completed).
    // Uses PostgreSQL INSERT with ON CONFLICT for atomicity.
    Acquire(ctx context.Context, record IdempotencyRecord) (bool, *IdempotencyRecord, error)

    // Complete marks an idempotency key as completed with the given result.
    // Writes to PostgreSQL first, then best-effort to Valkey.
    Complete(ctx context.Context, tenantID uuid.UUID, key string,
        result json.RawMessage, status string) error

    // Purge removes expired records. Called by the daily cleanup job.
    Purge(ctx context.Context) (int64, error)
}
```

### 5.3 PostgreSQL Schema

```sql
CREATE TABLE idempotency_records (
    key             TEXT        NOT NULL,
    tenant_id       UUID        NOT NULL,
    operation_type  TEXT        NOT NULL,
    result          JSONB,
    status          TEXT        NOT NULL DEFAULT 'pending'
                                CHECK (status IN ('pending', 'completed', 'failed')),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    expires_at      TIMESTAMPTZ NOT NULL,
    workflow_id     TEXT,

    PRIMARY KEY (tenant_id, key)
);

-- Index for cleanup job
CREATE INDEX idx_idempotency_expires ON idempotency_records (expires_at)
    WHERE status != 'pending';

-- Index for checking in-progress operations
CREATE INDEX idx_idempotency_pending ON idempotency_records (tenant_id, key)
    WHERE status = 'pending';
```

### 5.4 Two-Tier Implementation

```go
package idempotency

import (
    "context"
    "encoding/json"
    "fmt"
    "time"

    "github.com/jackc/pgx/v5/pgxpool"
    "github.com/valkey-io/valkey-go"
)

type TwoTierCache struct {
    db    *pgxpool.Pool
    vk    valkey.Client
    clock func() time.Time // injectable for testing
}

func NewTwoTierCache(db *pgxpool.Pool, vk valkey.Client) *TwoTierCache {
    return &TwoTierCache{db: db, vk: vk, clock: time.Now}
}

// compositeKey builds a tenant-scoped cache key.
func compositeKey(tenantID uuid.UUID, key string) string {
    return fmt.Sprintf("idem:%s:%s", tenantID.String(), key)
}

func (c *TwoTierCache) Check(ctx context.Context,
    tenantID uuid.UUID, key string) (*IdempotencyRecord, error) {

    // Fast path: check Valkey
    if c.vk != nil {
        vkKey := compositeKey(tenantID, key)
        cmd := c.vk.B().Get().Key(vkKey).Build()
        result, err := c.vk.Do(ctx, cmd).AsBytes()
        if err == nil {
            var record IdempotencyRecord
            if err := json.Unmarshal(result, &record); err == nil {
                return &record, nil
            }
        }
        // Valkey miss or error: fall through to PostgreSQL
    }

    // Durable path: check PostgreSQL
    var record IdempotencyRecord
    err := c.db.QueryRow(ctx, `
        SELECT key, tenant_id, operation_type, result, status,
               created_at, expires_at, workflow_id
        FROM idempotency_records
        WHERE tenant_id = $1 AND key = $2 AND expires_at > now()
    `, tenantID, key).Scan(
        &record.Key, &record.TenantID, &record.OperationType,
        &record.Result, &record.Status, &record.CreatedAt,
        &record.ExpiresAt, &record.WorkflowID,
    )
    if err != nil {
        if err == pgx.ErrNoRows {
            return nil, nil // Not found
        }
        return nil, fmt.Errorf("check idempotency: %w", err)
    }

    // Populate Valkey cache for future lookups (best-effort)
    if c.vk != nil && record.Status == "completed" {
        c.cacheInValkey(ctx, tenantID, key, &record)
    }

    return &record, nil
}

func (c *TwoTierCache) Acquire(ctx context.Context,
    record IdempotencyRecord) (bool, *IdempotencyRecord, error) {

    // Atomic INSERT with ON CONFLICT — PostgreSQL guarantees exactly-once.
    var existing IdempotencyRecord
    err := c.db.QueryRow(ctx, `
        INSERT INTO idempotency_records
            (key, tenant_id, operation_type, status, created_at, expires_at, workflow_id)
        VALUES ($1, $2, $3, 'pending', now(), $4, $5)
        ON CONFLICT (tenant_id, key)
        DO UPDATE SET key = idempotency_records.key  -- no-op update to return existing row
        RETURNING key, tenant_id, operation_type, result, status,
                  created_at, expires_at, workflow_id
    `, record.Key, record.TenantID, record.OperationType,
        record.ExpiresAt, record.WorkflowID,
    ).Scan(
        &existing.Key, &existing.TenantID, &existing.OperationType,
        &existing.Result, &existing.Status, &existing.CreatedAt,
        &existing.ExpiresAt, &existing.WorkflowID,
    )
    if err != nil {
        return false, nil, fmt.Errorf("acquire idempotency key: %w", err)
    }

    // If the returned row has a different status or was already there,
    // the key was not freshly acquired.
    if existing.Status != "pending" || existing.CreatedAt.Before(c.clock().Add(-1*time.Second)) {
        return false, &existing, nil
    }

    return true, nil, nil
}

func (c *TwoTierCache) Complete(ctx context.Context,
    tenantID uuid.UUID, key string,
    result json.RawMessage, status string) error {

    // PostgreSQL first (durable)
    _, err := c.db.Exec(ctx, `
        UPDATE idempotency_records
        SET result = $3, status = $4
        WHERE tenant_id = $1 AND key = $2
    `, tenantID, key, result, status)
    if err != nil {
        return fmt.Errorf("complete idempotency record: %w", err)
    }

    // Valkey second (best-effort, for fast subsequent lookups)
    if c.vk != nil {
        record := IdempotencyRecord{
            Key:       key,
            TenantID:  tenantID,
            Result:    result,
            Status:    status,
        }
        c.cacheInValkey(ctx, tenantID, key, &record)
    }

    return nil
}

func (c *TwoTierCache) cacheInValkey(ctx context.Context,
    tenantID uuid.UUID, key string, record *IdempotencyRecord) {

    data, err := json.Marshal(record)
    if err != nil {
        return // Best-effort
    }

    vkKey := compositeKey(tenantID, key)
    ttl := time.Until(record.ExpiresAt)
    if ttl <= 0 {
        return
    }
    // Cap Valkey TTL at 1 hour to avoid stale cache entries
    if ttl > 1*time.Hour {
        ttl = 1 * time.Hour
    }

    cmd := c.vk.B().Set().Key(vkKey).Value(string(data)).Ex(ttl).Build()
    _ = c.vk.Do(ctx, cmd) // Best-effort
}

func (c *TwoTierCache) Purge(ctx context.Context) (int64, error) {
    tag, err := c.db.Exec(ctx, `
        DELETE FROM idempotency_records
        WHERE expires_at < now() AND status != 'pending'
    `)
    if err != nil {
        return 0, fmt.Errorf("purge expired idempotency records: %w", err)
    }
    return tag.RowsAffected(), nil
}
```

### 5.5 TTL Rules

| Scenario | TTL | Rationale |
|----------|-----|-----------|
| Default operations | 24 hours | Covers retry windows and human working day |
| Approval-gated workflows | Approval timeout + 24h (max: 31 days) | Key must survive the entire approval wait |
| Emergency/break-glass | 1 hour | Short-lived, time-critical operations |
| Discovery operations | 4 hours | Rediscovery is safe and idempotent |

### 5.6 Failure Mode

**Decision: Fail CLOSED.** If both Valkey and PostgreSQL are unavailable, operations are blocked. Double-execution of infrastructure operations (power cycling a server twice, creating duplicate VLANs) is worse than temporary unavailability.

```go
func (c *TwoTierCache) CheckWithFailClosed(ctx context.Context,
    tenantID uuid.UUID, key string) (*IdempotencyRecord, error) {

    record, err := c.Check(ctx, tenantID, key)
    if err != nil {
        // Both tiers failed — fail closed.
        return nil, fmt.Errorf(
            "idempotency cache unavailable, blocking operation to prevent "+
            "double-execution: %w", err)
    }
    return record, nil
}
```

### 5.7 Cleanup Job

```go
// IdempotencyCleanupWorkflow is a Temporal cron workflow that purges expired
// idempotency records daily.
//
// Schedule: "0 3 * * *" (3:00 AM UTC daily)
//
// This is a lightweight workflow — typically completes in seconds.
func IdempotencyCleanupWorkflow(ctx workflow.Context) error {
    activityCtx := workflow.WithActivityOptions(ctx, workflow.ActivityOptions{
        StartToCloseTimeout: 5 * time.Minute,
    })

    var purged int64
    err := workflow.ExecuteActivity(activityCtx, PurgeExpiredRecords).Get(ctx, &purged)
    if err != nil {
        return fmt.Errorf("purge expired records: %w", err)
    }

    slog.Info("idempotency cleanup completed", "records_purged", purged)
    return nil
}
```

---

## 6. Schedule Risk Recommendations (H22)

### Decision

Reorder phases (Phase 4 before Phase 3), add 30% buffer to all estimates, build a device simulator in Phase 1B, and scope Phase 7 and Phase 8 to MVP-minimal.

### 6.1 Risk-Adjusted Timeline

| Phase | Original Estimate | Buffer | Risk-Adjusted | Key Risk | Change from Original Plan |
|-------|------------------|--------|---------------|----------|--------------------------|
| 0: Contracts | 3-4 wks | 0% | 3-4 wks | None (design only) | No change |
| 1A: Bootstrap | 2-3 wks | 30% | 3-4 wks | Temporal embedded mode | No change |
| 1B: Schema + Identity | 3-4 wks | 50% | 5-6 wks | Domain model gaps, device simulator | **Add device simulator** |
| 2: SSH + Redfish | 4-6 wks | 20% | 5-7 wks | First adapter always hardest | No change |
| **4: Temporal Workflows** | **6-8 wks** | **50%** | **9-12 wks** | **No Temporal experience** | **Moved before Phase 3** |
| 3: SNMP + Scanner | 4-6 wks | 30% | 5-8 wks | Vendor MIB fragmentation | **Moved after Phase 4** |
| 5: REST API + Auth | 4-6 wks | 20% | 5-7 wks | RBAC complexity | No change |
| 6: UI | 6-8 wks | 25% | 8-10 wks | Real-time updates | No change |
| 7: Network (MVP) | 10-14 wks | 0% | **6-8 wks** | Vendor fragmentation | **Scoped to 1 vendor** |
| 8: Cost + Lifecycle (MVP) | 6-8 wks | 0% | **4-5 wks** | LLM integration | **Deterministic only, defer LLM** |
| 9: Verification + Graph | 4-6 wks | 20% | 5-7 wks | AGE viability | No change |
| 10: HA + Security | 6-8 wks | 30% | 8-10 wks | mTLS, vault | No change |
| 11: Edge + Adapters | 6-8 wks | 40% | 8-11 wks | Edge sync complexity | No change |
| 12: Polish | 4-6 wks | 0% | 4-6 wks | None | No change |
| **Total** | **68-95 wks** | | **80-105 wks** | | **15-25% shorter than unbuffered worst case** |

### 6.2 Phase Reordering Rationale: Phase 4 Before Phase 3

**Why:** Temporal is the highest architectural risk in the entire project. The team has no Temporal experience. If Temporal's execution model does not work as assumed (deterministic replay constraints, signal handling, `continue-as-new` complexity), the project needs to know before investing in more adapters.

**What changes:**
- Phase 4 (Temporal Workflows) moves to immediately after Phase 2 (SSH + Redfish).
- Phase 3 (SNMP + Scanner + Classifier) moves after Phase 4.
- This means the first Temporal workflow uses SSH and Redfish adapters only, which is sufficient for validation.
- If Temporal works, Phase 3 proceeds normally.
- If Temporal requires architectural changes, those changes are made before the team has built 5 adapters.

### 6.3 Phase 7 MVP Scope

**Original Phase 7:** 4+ vendors (SONiC, EOS, IOS-XE, NX-OS, Junos, MikroTik) with universal config translation.

**MVP Phase 7:** 1 vendor (SONiC via gNMI) with intent-based templates.

| In MVP | Post-MVP |
|--------|----------|
| SONiC via gNMI adapter | EOS via eAPI |
| VLAN create/delete/modify | NX-OS via NX-API |
| Interface config (access/trunk) | IOS-XE via NETCONF |
| Basic ACL (permit/deny) | Junos via NETCONF |
| Intent → vendor template | MikroTik via REST |
| | Universal config translation engine |
| | Multi-vendor workflow in single Temporal execution |

**Why SONiC first:** Open-source, standardized OpenConfig YANG models, gNMI is the cleanest protocol, no OEM surprises. Validates the entire intent-to-device pipeline with the least vendor friction.

### 6.4 Phase 8 MVP Scope

**Original Phase 8:** Cost model with LLM integration, TTL/budget/idle detection.

**MVP Phase 8:** Deterministic cost tracking only. LLM deferred to post-MVP.

| In MVP | Post-MVP |
|--------|----------|
| Manual cost entry per resource | Cloud pricing API integration |
| Per-tenant cost aggregation | LLM-powered cost optimization suggestions |
| TTL enforcement | Idle detection (requires baseline analysis) |
| Budget cap alerting | Automatic teardown on budget breach |
| Cost dashboard (basic bar charts) | Predictive cost forecasting |

### 6.5 Device Simulator

**Decision:** Build a configurable device simulator in Phase 1B to enable scale testing throughout all subsequent phases.

```go
package simulator

// DeviceSimulator exposes fake SSH, Redfish, SNMP, and IPMI endpoints.
// Each simulated device responds with configurable facts, supports
// the full adapter interface, and can inject faults.
//
// Deployed as a docker-compose service alongside PostgreSQL, NATS, and Valkey.
// Scales to 10,000 simulated devices on a single host.
type DeviceSimulator struct {
    // Devices maps device ID to simulated device configuration.
    Devices map[string]SimulatedDevice

    // FaultInjector controls simulated failures.
    FaultInjector FaultInjector
}

type SimulatedDevice struct {
    ID             string            `json:"id"`
    Hostname       string            `json:"hostname"`
    DeviceType     string            `json:"device_type"`    // "server", "switch"
    Vendor         string            `json:"vendor"`         // "Dell", "Cisco"
    Model          string            `json:"model"`
    Protocols      []string          `json:"protocols"`      // ["ssh", "redfish", "snmp"]
    Facts          map[string]string `json:"facts"`          // pre-configured fact values
    PowerState     string            `json:"power_state"`    // "on", "off"
    ResponseDelay  time.Duration     `json:"response_delay"` // simulated latency
}

type FaultInjector struct {
    // ErrorRate is the probability (0.0-1.0) that any operation returns an error.
    ErrorRate float64 `json:"error_rate"`

    // TimeoutRate is the probability that an operation hangs until timeout.
    TimeoutRate float64 `json:"timeout_rate"`

    // PartialFailureRate is the probability of a partial error.
    PartialFailureRate float64 `json:"partial_failure_rate"`
}

// The simulator enables:
// 1. Scale testing: run discovery against 10K simulated devices in CI.
// 2. Fault injection: test saga compensation with controlled failures.
// 3. Adapter development: build adapters without physical hardware.
// 4. Workflow testing: validate Temporal workflows against deterministic responses.
// 5. Performance benchmarking: measure throughput under controlled conditions.
```

**Simulator implementation priority:**
- Phase 1B: SSH and Redfish endpoints (mock responses, configurable facts).
- Phase 2: Expanded to support SNMP.
- Phase 4: Fault injection for Temporal workflow testing.
- Phase 7: gNMI/NETCONF endpoints for network config testing.

---

## 7. Temporal GetVersion Integration (H23)

### Decision

`workflow.GetVersion()` is mandatory for all workflow changes that affect activity execution. This is enforced by CI lint (Section 1.1.5) and documented in every workflow file (Section 1.1.6). This section provides the complete rules document for LOOM workflow developers.

### 7.1 Rules for LOOM Workflow Developers

#### Rule 1: Every activity change requires `GetVersion`

Any change to a workflow that alters which activities run, their order, their inputs, their outputs, or their retry policies requires a `workflow.GetVersion()` branch. This includes:

| Change Type | Requires GetVersion? | Example |
|-------------|---------------------|---------|
| Add a new activity | **Yes** | Adding a health check step |
| Remove an activity | **Yes** | Removing a legacy DNS check |
| Change activity input struct | **Yes** | Adding a `Timeout` field to `PowerCycleParams` |
| Change activity output struct | **Yes** | Adding a `Duration` field to `PowerCycleResult` |
| Change activity retry policy | **Yes** | Changing `MaximumAttempts` from 3 to 5 |
| Reorder activities | **Yes** | Moving verification before registration |
| Change a timer duration | **Yes** | Changing a wait from 30s to 60s |
| Change signal handling logic | **Yes** | Changing how approval signals are processed |
| Change a side-effect computation | **Yes** | Changing a `workflow.SideEffect` return value |
| Change a query handler | No | Query handlers do not affect replay |
| Change logging within a workflow | No | Logging does not affect replay |
| Change an activity's internal implementation (not its signature) | No | Refactoring inside the activity function |

#### Rule 2: Change IDs are descriptive and globally unique

```go
// GOOD change IDs:
workflow.GetVersion(ctx, "add-health-check-before-provisioning", ...)
workflow.GetVersion(ctx, "replace-single-verify-with-retry-in-power-cycle", ...)
workflow.GetVersion(ctx, "add-timeout-param-to-configure-network", ...)

// BAD change IDs (will be rejected by linter):
workflow.GetVersion(ctx, "v2", ...)           // Non-descriptive
workflow.GetVersion(ctx, "fix", ...)          // Non-descriptive
workflow.GetVersion(ctx, "update", ...)       // Non-descriptive
workflow.GetVersion(ctx, "change1", ...)      // Non-descriptive
workflow.GetVersion(ctx, "temp", ...)         // Non-descriptive
```

#### Rule 3: Old code paths are never deleted while in-flight workflows exist

```go
// WRONG: deleting old code path
func ProvisioningWorkflow(ctx workflow.Context, input ProvisioningInput) error {
    // Someone removed the DefaultVersion branch because "nobody uses it anymore"
    // This BREAKS replay for any in-flight workflow that was started before v1.
    var healthResult HealthCheckResult
    err := workflow.ExecuteActivity(ctx, PreProvisioningHealthCheck,
        HealthCheckParams{DeviceID: input.DeviceID}).Get(ctx, &healthResult)
    // ...
}

// RIGHT: keeping old code path until verified safe to remove
func ProvisioningWorkflow(ctx workflow.Context, input ProvisioningInput) error {
    v := workflow.GetVersion(ctx, "add-health-check-before-provisioning",
        workflow.DefaultVersion, 1)

    if v >= 1 {
        var healthResult HealthCheckResult
        err := workflow.ExecuteActivity(ctx, PreProvisioningHealthCheck,
            HealthCheckParams{DeviceID: input.DeviceID}).Get(ctx, &healthResult)
        if err != nil {
            return err
        }
    }
    // DefaultVersion path: no health check (original behavior, retained for replay)
    // ...
}
```

#### Rule 4: Activity structs are append-only

```go
// WRONG: breaking change
type PowerCycleParams struct {
    DeviceID uuid.UUID     // renamed from TargetID — BREAKS deserialization
    Timeout  int           // changed from time.Duration — BREAKS deserialization
    // WaitTime removed — BREAKS deserialization of in-flight workflows
}

// RIGHT: additive change
type PowerCycleParams struct {
    TargetID uuid.UUID     `json:"target_id"`              // v1: original
    WaitTime time.Duration `json:"wait_time"`              // v1: original
    DeviceID uuid.UUID     `json:"device_id,omitempty"`    // v2: added, replaces TargetID
    Timeout  time.Duration `json:"timeout,omitempty"`      // v2: added (default: 5m)
}
```

### 7.2 Version History Documentation Template

```go
// ============================================================================
// WORKFLOW VERSION HISTORY
// ============================================================================
//
// Workflow: ProvisioningWorkflow
// File:     internal/workflow/provisioning.go
//
// ┌─────────┬────────────┬─────────────────────────────────────────────────┬───────────┐
// │ Version │ Date       │ Change ID                                       │ Status    │
// ├─────────┼────────────┼─────────────────────────────────────────────────┼───────────┤
// │ Default │ 2026-04-01 │ (initial implementation)                        │ Drainable │
// │ 1       │ 2026-04-15 │ add-health-check-before-provisioning            │ Active    │
// │ 2       │ 2026-05-01 │ add-post-provision-verification-retry           │ Active    │
// │ 3       │ 2026-06-12 │ replace-single-verify-with-exponential          │ Active    │
// └─────────┴────────────┴─────────────────────────────────────────────────┴───────────┘
//
// Status legend:
//   Drainable  = CanRemoveVersion() returns true; old code path can be removed
//                in the next release.
//   Active     = Open workflows may replay through this version; code path
//                must be retained.
//   Retired    = Code path has been removed in a prior release (listed for history).
//
// ============================================================================
```

### 7.3 Version Cleanup Process

```go
package versioncleanup

import (
    "context"
    "fmt"
    "time"

    "go.temporal.io/api/workflowservice/v1"
    "go.temporal.io/sdk/client"
    "go.temporal.io/sdk/workflow"
)

// VersionCleanupWorkflow runs weekly to identify version branches that
// can be safely removed (no open workflows reference them).
//
// Schedule: "0 0 * * 0" (midnight UTC every Sunday)
//
// Output: A report of which versions are drainable, published as a
// NATS event and logged for developer visibility.
func VersionCleanupWorkflow(ctx workflow.Context) error {
    activityCtx := workflow.WithActivityOptions(ctx, workflow.ActivityOptions{
        StartToCloseTimeout: 5 * time.Minute,
    })

    var report VersionCleanupReport
    err := workflow.ExecuteActivity(activityCtx,
        ScanDrainableVersions).Get(ctx, &report)
    if err != nil {
        return fmt.Errorf("scan drainable versions: %w", err)
    }

    if len(report.DrainableVersions) > 0 {
        // Publish report as a NATS event for developer notification
        err = workflow.ExecuteActivity(activityCtx,
            PublishCleanupReport, report).Get(ctx, nil)
        if err != nil {
            return fmt.Errorf("publish cleanup report: %w", err)
        }
    }

    return nil
}

// VersionCleanupReport summarizes which workflow versions can be removed.
type VersionCleanupReport struct {
    ScannedAt          time.Time            `json:"scanned_at"`
    WorkflowTypes      int                  `json:"workflow_types_scanned"`
    DrainableVersions  []DrainableVersion   `json:"drainable_versions"`
    ActiveVersions     []ActiveVersion      `json:"active_versions"`
}

type DrainableVersion struct {
    WorkflowType     string    `json:"workflow_type"`
    ChangeID         string    `json:"change_id"`
    Version          int       `json:"version"`
    IntroducedAt     time.Time `json:"introduced_at"`
    LastOpenWorkflow time.Time `json:"last_open_workflow"` // When the last workflow using this version completed
    SafeToRemove     bool      `json:"safe_to_remove"`
}

type ActiveVersion struct {
    WorkflowType     string `json:"workflow_type"`
    ChangeID         string `json:"change_id"`
    Version          int    `json:"version"`
    OpenWorkflows    int    `json:"open_workflows"` // Count of workflows that may replay through this version
}

// ScanDrainableVersions is the activity that queries Temporal's visibility
// API to determine which versions are safe to remove.
func ScanDrainableVersions(ctx context.Context) (*VersionCleanupReport, error) {
    // Implementation queries Temporal for each registered workflow type
    // and each known version change ID, checking for open workflows
    // that were started before that version was introduced.
    //
    // The version registry (a static map in the codebase, maintained
    // alongside the version history comments) provides the list of
    // known change IDs and their introduction dates.
    //
    // For each version:
    //   1. Query: "WorkflowType = X AND ExecutionStatus = 'Running'
    //      AND StartTime < version_introduction_date"
    //   2. If count == 0 → drainable
    //   3. If count > 0 → active (report count)

    // ... implementation omitted for brevity, follows the CanRemoveVersion
    // pattern from Section 1.1.4
    return &VersionCleanupReport{}, nil
}
```

### 7.4 CI Integration

The CI pipeline includes two checks related to workflow versioning:

**Check 1: `loomvet` static analyzer** (see Section 1.1.5)

Detects activity struct changes or workflow step changes without corresponding `GetVersion` calls. Runs as part of `go vet`.

**Check 2: Change ID uniqueness check**

```bash
#!/bin/bash
# scripts/check-version-ids.sh
# Ensures all GetVersion change IDs are unique across the codebase.
# Duplicate change IDs cause non-deterministic replay behavior.

DUPES=$(grep -roh 'workflow.GetVersion(ctx, "[^"]*"' internal/workflow/ \
    | sort | uniq -d)

if [ -n "$DUPES" ]; then
    echo "ERROR: Duplicate GetVersion change IDs detected:"
    echo "$DUPES"
    exit 1
fi
```

**Check 3: Change ID format validation**

```bash
#!/bin/bash
# scripts/check-version-id-format.sh
# Validates that all GetVersion change IDs follow the naming convention.
# Pattern: {verb}-{noun}-{context}, all lowercase, kebab-case.

BAD_IDS=$(grep -roh 'workflow.GetVersion(ctx, "[^"]*"' internal/workflow/ \
    | grep -vE '"[a-z]+-[a-z]+-[a-z-]+"')

if [ -n "$BAD_IDS" ]; then
    echo "ERROR: GetVersion change IDs must follow '{verb}-{noun}-{context}' format:"
    echo "$BAD_IDS"
    exit 1
fi
```

---

## Appendix A: Open Question Resolution Log

Every open question from RESEARCH-wire-protocol-lifecycle.md is resolved below.

### Section 2.6 (Wire Protocol)

| Question | Decision | Rationale |
|----------|----------|-----------|
| How long must old Temporal versions be maintained? | Until zero open workflows, verified weekly by cleanup workflow. Max 37 days for approval-gated workflows. | Temporal's replay model requires old code paths for in-flight workflows. Automated scanning makes this manageable. |
| NATS: protobuf or JSON? | JSON for MVP. Protobuf post-MVP if performance requires it. | JSON provides superior debuggability. LOOM's event volume (<10K/s) does not justify protobuf complexity. The `SchemaID` mechanism works identically with either encoding. |
| Adapter protocol version: semver or integer? | Integer. | Simpler comparison, sufficient for N-1 compatibility. Protobuf messages are inherently additive; version bumps are rare. |

### Section 3.4 (Temporal-PostgreSQL)

| Question | Decision | Rationale |
|----------|----------|-----------|
| Temporal connection pool under load? | 20 connections allocated. Measure in Phase 4 and adjust. | Temporal's PostgreSQL driver pools internally. 20 is conservative; can increase at the expense of LOOM's 60. |
| Can LOOM migration block Temporal? | No, if migration safety rules are followed (Section 2.3). | Separate databases prevent cross-database locking. Migration `statement_timeout` prevents long-running DDL. |
| Should deployment topology specify separation boundary? | Yes, 1,000 devices is the threshold. | Below 1K: shared instance acceptable. Above 1K: separate instances required for operational independence. |

### Section 4.5 (ObservedFact Retention)

| Question | Decision | Rationale |
|----------|----------|-----------|
| TimescaleDB from day one? | Yes. | Retrofitting partitioning onto a billion-row table is operationally dangerous. TimescaleDB is already in docker-compose. The overhead for small deployments is negligible. |
| Acceptable latency for latest_facts? | 60 seconds (continuous aggregate refresh), with real-time fallback query available. | Drift detection runs every 5 minutes. 60s staleness is acceptable. Sub-minute queries use the direct fallback. |
| Per-tenant configurable retention? | Yes, within guardrails (min 7 days hot). | Tenants with compliance requirements may need longer retention. Guardrails prevent operators from setting retention too low for drift detection. |

### Section 5.6 (continue-as-new)

| Question | Decision | Rationale |
|----------|----------|-----------|
| Event threshold? | 10,000 (configurable per workflow type). | 5x safety margin below Temporal's 50K hard limit. Accounts for event spikes from signal bursts and retry storms. |
| Approval signals across CAN? | Workflow does not CAN during approval wait. Approval signals target WorkflowID (RunID resolved by Temporal). | Approval waits generate few events. Temporal's signal routing by WorkflowID handles CAN transparently. |
| Transparent to API consumer? | Yes. API exposes WorkflowID only. RunID is internal. | Consumers should not need to understand Temporal internals. WorkflowID is the stable identifier. |

### Section 6.4 (Idempotency)

| Question | Decision | Rationale |
|----------|----------|-----------|
| Per-tenant or global cache? | Per-tenant (composite key: `{tenant_id}:{idempotency_key}`). | Prevents cross-tenant cache poisoning. Tenant A's idempotency key cannot collide with Tenant B's. |
| Both caches unavailable? | Fail closed (block operations). | Double-execution of infrastructure operations (power cycles, VLAN creation) is more dangerous than temporary unavailability. |
| Use Temporal's built-in caching? | No. Temporal's activity result cache handles replay, not cross-request idempotency. The external cache is needed for API-level idempotency. | Different scope: Temporal caches within a workflow execution. The idempotency cache spans across requests and workflows. |

### Section 7.4 (Schedule)

| Question | Decision | Rationale |
|----------|----------|-----------|
| Extend to 2+ years or cut scope? | Cut scope (Phase 7, 8) and reorder (Phase 4 before Phase 3). Target 80-105 weeks. | Validate the riskiest component (Temporal) early. Ship a narrower MVP sooner. Expand post-MVP. |
| Which phases can be parallelized? | Phase 6 (UI) overlaps Phase 5 tail. Phase 8 (Cost) partially overlaps Phase 7 with separate engineers. | 4-8 weeks compression possible. |
| Phase 7 target 1 vendor for MVP? | Yes — SONiC via gNMI. | Cleanest protocol, open-source, standardized YANG models. Validates the entire pipeline with minimal vendor friction. |

### Section 8.4 (GetVersion)

| Question | Decision | Rationale |
|----------|----------|-----------|
| Workflow version registry in DB? | No. Static map in codebase alongside version history comments. | Simpler. The cleanup workflow reads the static map. Adding a DB table adds complexity without benefit. |
| How are old versions cleaned up? | Weekly cleanup workflow scans Temporal visibility API. | Automated scanning prevents manual drift. Weekly cadence is sufficient because workflows complete within days. |
| CI block on missing GetVersion? | Yes. `loomvet` analyzer blocks PRs. | Enforces the versioning contract at the earliest possible point. A missed `GetVersion` in production causes data loss (broken replay). |

---

## Appendix B: Cross-Reference to Source Documents

| This Section | Resolves | Source Document | Status |
|-------------|----------|-----------------|--------|
| 1 (Wire Protocol Versioning) | C2 | GAP-ANALYSIS.md, RESEARCH-wire-protocol-lifecycle.md Sec 2 | **Closed** |
| 2 (Temporal-PostgreSQL) | H1 | GAP-ANALYSIS.md, RESEARCH-wire-protocol-lifecycle.md Sec 3 | **Closed** |
| 3 (ObservedFact Retention) | H3 | GAP-ANALYSIS.md, RESEARCH-wire-protocol-lifecycle.md Sec 4 | **Closed** |
| 4 (continue-as-new) | H5 | GAP-ANALYSIS.md, RESEARCH-wire-protocol-lifecycle.md Sec 5 | **Closed** |
| 5 (Idempotency Cache) | H6 | GAP-ANALYSIS.md, RESEARCH-wire-protocol-lifecycle.md Sec 6 | **Closed** |
| 6 (Schedule Risk) | H22 | GAP-ANALYSIS.md, RESEARCH-wire-protocol-lifecycle.md Sec 7 | **Closed** |
| 7 (GetVersion Integration) | H23 | GAP-ANALYSIS.md, RESEARCH-wire-protocol-lifecycle.md Sec 8 | **Closed** |
