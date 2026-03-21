# Wire Protocol Versioning

> **Status:** Decided
> **Addresses:** GAP-ANALYSIS.md C2 (wire protocol versioning absent), H3, H5, H6, H22, H23, and research items WPL-1 through WPL-12
> **Source research:** [RESEARCH-wire-protocol-lifecycle.md](RESEARCH-wire-protocol-lifecycle.md)
> **Last updated:** 2026-03-21

---

## Overview

LOOM has five wire protocol boundaries. Four of them lacked versioning contracts. This document defines the versioning strategy for each boundary, plus the data lifecycle policies (retention, idempotency, continue-as-new) that the research surfaced alongside them.

| Boundary | Transport | Versioning Strategy |
|----------|-----------|-------------------|
| REST API | HTTP | `/api/v1/` (already defined, not covered here) |
| Temporal activities | Temporal SDK | `workflow.GetVersion()` branching (Section 1, 8) |
| NATS events | NATS JetStream | Envelope `Version` field + `SchemaID` (Section 2) |
| Adapter plugins | gRPC (go-plugin) | `HandshakeConfig` protocol version negotiation (Section 3) |
| Edge agent sync | NATS leaf node | Protocol version in sync envelope header (Section 4) |

---

## 1. Temporal Activity Versioning

**Gap:** C2, H23, WPL-1, WPL-10

Temporal replays workflow execution from event history. Changing an activity's input or output struct breaks replay of in-flight workflows. All LOOM workflows MUST use `workflow.GetVersion()` before calling activities whose signatures have changed.

### Version Constants

All version constants live in a single file for discoverability:

```go
// internal/workflow/versions.go

package workflow

// Version constants for activity signature changes.
// Never delete a constant while any open workflow may reference it.
const (
    ActivityVersion_CreateVLAN_V2       = 2  // added timeout parameter
    ActivityVersion_PowerCycle_V2       = 2  // added force flag
    ActivityVersion_OSInstall_V3        = 3  // switched to staged rollout
    ActivityVersion_HealthCheck_V2      = 2  // added pre-provisioning checks
)
```

### Version Branching Pattern

```go
func ProvisionWorkflow(ctx workflow.Context, input ProvisionInput) error {
    v := workflow.GetVersion(ctx, "add-health-check-step", workflow.DefaultVersion, 1)

    if v == workflow.DefaultVersion {
        // Old path: no health check (replays old workflows correctly)
        err = workflow.ExecuteActivity(ctx, ProvisionActivityV1, oldParams).Get(ctx, &result)
    } else {
        // New path (v1): added pre-provisioning health check
        err = workflow.ExecuteActivity(ctx, RunHealthCheck, healthParams).Get(ctx, nil)
        if err != nil {
            return err
        }
        err = workflow.ExecuteActivity(ctx, ProvisionActivityV2, newParams).Get(ctx, &result)
    }
    return err
}
```

### Activity Struct Rules

- Activity input/output structs are **additive only**: new fields with zero-value defaults, never remove or rename fields.
- Every activity struct carries a version comment: `// ActivityVersion: 1`.
- Breaking changes (type changes, removed fields) require a new activity function and `GetVersion` branching.

### SafeActivity Helper

```go
// SafeActivity wraps activity execution with version tracking metadata.
func SafeActivity[T any](ctx workflow.Context, changeID string, minVersion, maxVersion int,
    oldFn func() (T, error), newFn func() (T, error)) (T, error) {

    v := workflow.GetVersion(ctx, changeID, minVersion, maxVersion)
    if v == minVersion {
        return oldFn()
    }
    return newFn()
}
```

### Replay Safety

Old workflow histories replay correctly because `GetVersion` returns the version recorded in the event history for that change ID. New workflows get the latest version. Both code paths must remain in the codebase until all old workflows have completed.

### Version Cleanup Procedure

1. Query Temporal visibility API for open workflows with version < N: `WorkflowType = "ProvisionWorkflow" AND CloseTime = missing`.
2. If zero open workflows reference version N-1, the old code path may be removed.
3. Run this check as a scheduled job (weekly) and log results.
4. After removing old paths, update the `GetVersion` call to set `minVersion` to the new baseline.

---

## 2. NATS Event Envelope Versioning

**Gap:** C2, WPL-2

The `Event` envelope struct in [EVENT-MODEL.md](EVENT-MODEL.md) now includes a `Version` field (added as part of this decision).

### Updated Envelope

```go
type Event struct {
    ID            string    `json:"id"`
    Version       int       `json:"version"`         // Envelope schema version (start at 1)
    Timestamp     time.Time `json:"timestamp"`
    TenantID      string    `json:"tenant_id"`
    Actor         Actor     `json:"actor"`
    Action        string    `json:"action"`
    ResourceType  string    `json:"resource_type"`
    ResourceID    string    `json:"resource_id"`
    Data          any       `json:"data"`
    CorrelationID string    `json:"correlation_id"`
    CausationID   string    `json:"causation_id"`
}
```

### Schema Evolution Rules

- **Additive only**: New fields with zero-value defaults are non-breaking. All consumers MUST ignore unknown fields (use `json:",omitempty"` and lenient deserialization).
- **Removal or rename is breaking**: Renaming or removing a field requires a new envelope version AND a new NATS subject namespace (e.g., `loom.v2.{tenant_id}.{domain}.{action}`).
- **Payload schemas** are identified by `ResourceType` + `Action` combination. Each combination maps to a well-known Go struct type.

### Consumer Version Compatibility

- Consumers MUST handle unknown fields gracefully: unmarshal with `json.Decoder` using `DisallowUnknownFields` = false (the default).
- Consumers MUST check `event.Version` before processing. If `Version` is higher than the consumer understands, log a warning and skip (do not crash).
- Consumers SHOULD be updated to the latest version within one minor release cycle.

### Schema Registry

- JSON Schema definitions are stored per event type in PostgreSQL (`event_schemas` table) with columns: `resource_type`, `action`, `version`, `json_schema`, `created_at`.
- Schema validation is optional at publish time (enabled via config flag) and mandatory in CI tests.
- Schema registry is queried at startup by consumers for documentation, not for runtime deserialization (Go structs are the source of truth).

### Breaking Change Procedure

When a breaking envelope change is unavoidable:

1. Create new subjects under `loom.v2.{tenant_id}.{domain}.{action}`.
2. Publish to both old and new subjects during a migration window (one minor release).
3. Consumers migrate to new subjects.
4. Stop publishing to old subjects after the migration window closes.

---

## 3. Adapter gRPC Protocol Versioning

**Gap:** C2, WPL-3

Adapter plugins communicate over gRPC via HashiCorp go-plugin. Version negotiation happens at connection time.

### HandshakeConfig

```go
var AdapterHandshake = plugin.HandshakeConfig{
    ProtocolVersion:  2,                    // current protocol version
    MagicCookieKey:   "LOOM_ADAPTER",
    MagicCookieValue: "loom-adapter-v1",
}
```

### Version Negotiation

```protobuf
service AdapterPlugin {
    rpc Handshake(HandshakeRequest) returns (HandshakeResponse);
}

message HandshakeRequest {
    int32 min_protocol_version = 1;  // minimum version host supports
    int32 max_protocol_version = 2;  // maximum version host supports
}

message HandshakeResponse {
    int32 negotiated_version = 1;    // version both sides agree on
    repeated string capabilities = 2; // adapter's declared capabilities
}
```

- The adapter reports its supported version range.
- LOOM selects the highest version both sides support.
- If no overlap exists, the adapter is rejected with a clear error message including the version mismatch details.

### Breaking Change Policy

- LOOM supports **N-1 adapter protocol versions** for one minor release cycle.
- When protocol version N is released, adapters at version N-1 continue to work. Adapters at version N-2 are rejected.
- Adapter authors receive a deprecation warning in logs one release before an old version is dropped.
- Capability declaration (`repeated string capabilities`) allows feature detection without protocol version bumps for non-breaking additions.

---

## 4. Edge Sync Protocol Versioning

**Gap:** C2, WPL-4

Edge agents and the hub negotiate protocol version when the agent establishes a NATS leaf node connection.

### Sync Envelope

```go
type SyncEnvelope struct {
    AgentVersion    string          `json:"agent_version"`    // semver of agent binary
    ProtocolVersion int             `json:"protocol_version"` // sync protocol version
    HubMinVersion   int             `json:"hub_min_version"`  // minimum protocol version hub accepts
    Payload         json.RawMessage `json:"payload"`
}
```

### Compatibility Window

- The hub supports the **current + 2 previous** agent protocol versions.
- Agents with `ProtocolVersion < HubMinVersion` are rejected.
- Rejected agents are directed to auto-update (see edge agent security model) before reconnecting.

### Sync Messages

- Every sync message carries the `ProtocolVersion` in its header.
- The hub validates `ProtocolVersion` on every message, not just at connection time (guards against mid-session version confusion after a hub upgrade).

### Incompatible Agent Handling

1. Agent connects with `ProtocolVersion` below `HubMinVersion`.
2. Hub responds with an `UpdateRequired` message containing the download URL and expected version.
3. Agent downloads, verifies signature (dual-key, see edge agent security model), installs, and reconnects.
4. Full sync does not proceed until the agent is at a compatible version.

---

## 5. ObservedFact Retention Policy

**Gap:** H3, WPL-6

ObservedFact is append-only. At scale (10,000 devices, 10 facts per device per minute), this produces 144 million rows per day without retention controls.

### Retention Tiers

| Tier | Duration | Resolution | Storage Mechanism |
|------|----------|------------|-------------------|
| Hot | 7 days | Full resolution (every fact) | PostgreSQL uncompressed partitions |
| Warm | 30 days | Hourly aggregates (min/max/avg per fact type per device) | PostgreSQL compressed partitions |
| Cold | 1 year | Daily aggregates | PostgreSQL compressed partitions |
| Archive | 7 years | Monthly Parquet exports | S3 / object storage |

### Go Type

```go
type RetentionPolicy struct {
    HotDays           int           `json:"hot_days"`            // default: 7
    WarmDays          int           `json:"warm_days"`           // default: 30
    ColdDays          int           `json:"cold_days"`           // default: 365
    AggregateInterval time.Duration `json:"aggregate_interval"`  // default: 1h for warm, 24h for cold
}
```

### Implementation

- ObservedFact table is partitioned by `observed_at` using `pg_partman`.
- `pg_partman` automatically detaches partitions that exit the hot tier.
- Detached hot partitions are aggregated into the warm tier by a scheduled job.
- Warm partitions are further aggregated into the cold tier on the same schedule.
- Archive exports run monthly, writing Parquet files partitioned by `tenant_id/year/month`.
- Tenants may customize retention via the `RetentionPolicy` on their tenant config (subject to minimum floors set by the operator).

---

## 6. Continue-As-New Strategy

**Gap:** H5, WPL-7

Temporal has a ~50K event hard limit per workflow execution. Long-running LOOM workflows (discovery scans, continuous monitoring) require `continue-as-new`.

### Trigger Conditions

- Workflow event history exceeds **5,000 events** (well below the hard limit, providing safety margin), OR
- **30 days** have elapsed since the workflow execution started.

### Checkpoint Procedure

1. **Persist state**: Write current workflow progress to PostgreSQL via an activity (`PersistCheckpointActivity`).
2. **Emit audit event**: Publish a `workflow.continued` event linking the old and new run IDs.
3. **Drain signals**: Consume all pending signals from the channel before calling `continue-as-new` (see below).
4. **Start new execution**: Call `workflow.NewContinueAsNewError` with a `WorkflowCheckpoint` carrying the state reference.

### Signal Handling

```go
// Before continue-as-new: drain all pending signals
func drainSignals(ctx workflow.Context, ch workflow.ReceiveChannel) []PendingSignal {
    var pending []PendingSignal
    for {
        var signal SignalPayload
        if !ch.ReceiveAsync(&signal) {
            break
        }
        pending = append(pending, PendingSignal{Signal: signal})
    }
    return pending
}
```

Pending signals are included in the checkpoint state and re-processed at the start of the new execution.

### Projection Updates

Before calling `continue-as-new`, the workflow emits a reconciliation event so that DB projections reflect the current state. The `WorkflowStatus` projection tracks `CurrentRunID` (updated on continue) and `OriginalRunID` (never changes).

### Audit Continuity

New workflows link to their parent via `ContinuedFromWorkflowID` and `ContinuedFromRunID` in the checkpoint. The `CorrelationID` is preserved across all continuations so that the full chain is traceable.

---

## 7. Idempotency Cache

**Gap:** H6, WPL-8

Adapter operations must be idempotent. The idempotency cache provides deduplication across retries and workflow replays.

### Two-Tier Architecture

| Tier | Store | TTL | Purpose |
|------|-------|-----|---------|
| Fast path | Valkey | 24 hours | Sub-millisecond lookups for hot operations |
| Durable | PostgreSQL | 7 days | Survives Valkey restarts, covers long-paused workflows |

### Read/Write Flow

- **Write**: Store in both tiers simultaneously (PostgreSQL first for durability, then Valkey best-effort).
- **Read**: Check Valkey first. On miss, check PostgreSQL. On PostgreSQL hit, backfill Valkey.

### Key Format

```
{tenant_id}:{device_id}:{operation_type}:{content_hash}
```

- `content_hash` is a SHA-256 of the operation parameters, ensuring that different parameters for the same operation type are not incorrectly deduplicated.

### Degradation

- **Valkey unavailable**: Degrade to PostgreSQL-only (slower but correct). Log warning, increment metric.
- **Both unavailable**: Fail closed (reject the operation). Double-execution risk is worse than temporary unavailability for destructive operations like firmware updates and power cycles.

### TTL Rules

| Scenario | TTL |
|----------|-----|
| Default operations | 24 hours |
| Operations behind approval gates | Max pause duration + 24 hours |
| Emergency / break-glass operations | 1 hour |
| Discovery operations | 4 hours (rediscovery is safe) |

---

## 8. Temporal GetVersion Integration

**Gap:** H23, WPL-10

Every LOOM workflow that calls activities MUST use `GetVersion` when activity signatures change. This section complements Section 1 with the developer workflow.

### Developer Rules

1. Every workflow change that affects activity execution MUST use `GetVersion` with a descriptive, unique change ID (e.g., `"add-verification-step-to-provisioning"`, not `"v2"` or `"fix"`).
2. Activity input/output structs are append-only. New fields get zero-value defaults. To change a field's type, add a new field and deprecate the old one.
3. Old code paths are NEVER deleted while in-flight workflows exist. Use Temporal's visibility API to verify.
4. Each workflow maintains a version history comment block at the top of its file.

### Version History Template

```go
// WorkflowVersionHistory:
// DefaultVersion (initial): basic provisioning with power-on, OS install
// Version 1 (2026-04-15): added pre-provisioning health check activity
// Version 2 (2026-05-01): changed OS install to use staged rollout
// Version 3 (2026-06-12): added post-provisioning verification step
```

### Version Constants File

All version constants are centralized in `internal/workflow/versions.go` (see Section 1).

### Version Cleanup

After all workflows at version N-1 have completed (verified via Temporal visibility query), the N-1 branch may be removed. A weekly scheduled job runs the visibility query and logs which versions are safe to clean up.

---

## 9. Schedule Risk Acknowledgment

**Gap:** H22, WPL-9

The original 12-phase plan estimated 68-95 weeks for 2-3 senior Go engineers. After accounting for this document's findings and other research gaps, the risk-adjusted estimate is **95-127 weeks**.

### Mitigation Strategy

| Mitigation | Impact |
|------------|--------|
| MVP scope limited to Phases 0-5 only | Reduces critical path to ~40-55 weeks |
| Parallel workstreams where possible (e.g., adapter dev alongside Temporal integration) | ~15% schedule compression |
| Adapter community contributions post-SDK release | Reduces Phase 7+ effort |
| Phase gates enforce quality over speed | Prevents rework-driven schedule explosion |

### Key Schedule Risks

- **Phase 4 (Temporal workflows)**: Highest risk. No prior Temporal experience. Recommend reordering before Phase 3.
- **Phase 7 (Network domain)**: NETCONF vendor fragmentation. MVP should target 1 vendor (SONiC via gNMI).
- **Phase 11 (Edge)**: This document's findings add 4-6 weeks to the original estimate.

---

## 10. Activity Timeout Budget for LLM

**Gap:** WPL-12

LLM-backed activities (natural language plan generation, anomaly explanation) have different latency characteristics than device operations.

### Timeout Configuration

| Parameter | Value | Rationale |
|-----------|-------|-----------|
| Activity timeout | 120 seconds | Model inference can take 30-90s for complex plans |
| Heartbeat interval | 15 seconds | LLM activity heartbeats during inference to prevent premature timeout |
| Start-to-close timeout | 180 seconds | Allows for one retry within the schedule-to-close window |

### Fallback

If the LLM activity times out after exhausting retries, the workflow falls back to a deterministic rule engine that produces a conservative but correct plan. The fallback is transparent to the caller -- the result struct is identical, with a `Source` field indicating `"llm"` or `"rule_engine"`.

### Heartbeat Pattern

```go
func LLMPlanActivity(ctx context.Context, input PlanInput) (PlanOutput, error) {
    // Start LLM inference in background
    resultCh := make(chan PlanOutput, 1)
    errCh := make(chan error, 1)
    go func() {
        result, err := llmClient.GeneratePlan(ctx, input)
        if err != nil {
            errCh <- err
            return
        }
        resultCh <- result
    }()

    // Heartbeat while waiting
    ticker := time.NewTicker(15 * time.Second)
    defer ticker.Stop()
    for {
        select {
        case result := <-resultCh:
            return result, nil
        case err := <-errCh:
            return PlanOutput{}, err
        case <-ticker.C:
            activity.RecordHeartbeat(ctx, "llm-inference-in-progress")
        case <-ctx.Done():
            return PlanOutput{}, ctx.Err()
        }
    }
}
```

---

## References

- [EVENT-MODEL.md](EVENT-MODEL.md) -- NATS event envelope (updated with `Version` field)
- [WORKFLOW-CONTRACT.md](WORKFLOW-CONTRACT.md) -- Temporal integration rules
- [ADAPTER-CONTRACT.md](ADAPTER-CONTRACT.md) -- Adapter interfaces and idempotency
- [RESEARCH-wire-protocol-lifecycle.md](RESEARCH-wire-protocol-lifecycle.md) -- Full research analysis
- [GAP-ANALYSIS.md](GAP-ANALYSIS.md) -- Consolidated gap findings (C2, H3, H5, H6, H22, H23)
- [Temporal Versioning Guide](https://docs.temporal.io/workflows#workflow-versioning)
