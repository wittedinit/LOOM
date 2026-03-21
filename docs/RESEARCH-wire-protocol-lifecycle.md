# Research: Wire Protocol & Data Lifecycle

> **Status:** Open — requires design decisions before Phase 1A
> **Addresses:** GAP-ANALYSIS.md findings C2, H1, H3, H5, H6, H22, H23
> **Owner:** TBD
> **Last updated:** 2026-03-21

---

## 1. Problem Statement

LOOM has multiple inter-component communication boundaries that function as wire protocols but lack versioning contracts. Additionally, several data lifecycle questions are unanswered: how long is data retained, how are caches located, and how does Temporal's `continue-as-new` interact with the rest of the system.

1. **Wire protocol versioning** — Adapter gRPC, NATS events, Temporal activities, edge sync — all unversioned
2. **Temporal-PostgreSQL shared fate** — Single failure domain for execution truth and application state
3. **ObservedFact retention** — Append-only facts with no retention policy = unbounded growth
4. **Temporal `continue-as-new`** — Cascading implications for signals, projections, and audit trails
5. **Idempotency cache** — Location and durability unspecified
6. **Schedule risk** — 12-phase plan is 16-22 months for 2-3 people
7. **Temporal workflow versioning** — `GetVersion` API not mentioned anywhere

---

## 2. Wire Protocol Versioning

### 2.1 The Problem

LOOM has five wire protocol boundaries with no versioning strategy:

| Boundary | Transport | Current Versioning | Risk |
|----------|-----------|-------------------|------|
| REST API | HTTP | `/api/v1/` (defined) | Low — well-understood |
| Adapter plugins | gRPC (go-plugin) | None | High — binary incompatibility |
| NATS events | NATS JetStream | None | High — field additions break consumers |
| Temporal activities | Temporal SDK | None | Critical — replay breaks on struct changes |
| Edge agent sync | NATS + custom | None | High — hub/agent version mismatch |

### 2.2 Temporal Activity Versioning (Most Critical)

Temporal replays workflow execution from event history. If you change an activity's input or output struct, replay of in-flight workflows fails. Temporal provides `workflow.GetVersion()` for this:

```go
func MyWorkflow(ctx workflow.Context, input WorkflowInput) error {
    // Version 1: original power cycle
    v := workflow.GetVersion(ctx, "power-cycle-params", workflow.DefaultVersion, 1)

    if v == workflow.DefaultVersion {
        // Old code path (for replaying old workflows)
        err = workflow.ExecuteActivity(ctx, PowerCycleV1, oldParams).Get(ctx, &result)
    } else {
        // New code path (v1: added timeout parameter)
        err = workflow.ExecuteActivity(ctx, PowerCycleV2, newParams).Get(ctx, &result)
    }
}
```

**Rules for LOOM:**
- Every activity input/output struct gets a version comment: `// ActivityVersion: 1`
- Struct fields are additive only (new fields with defaults, never remove/rename)
- Breaking changes require `workflow.GetVersion()` branching
- Old code paths are retained until all in-flight workflows using them have completed
- A "version cleanup" workflow drains old versions periodically

### 2.3 NATS Event Envelope Versioning

```go
type EventEnvelope struct {
    Version     int       `json:"version"`      // Envelope version (start at 1)
    SchemaID    string    `json:"schema_id"`     // e.g., "device.created.v2"
    Data        json.RawMessage `json:"data"`    // Payload (schema-versioned)
    // ... existing fields (timestamp, tenant, correlation_id)
}
```

**Rules:**
- Envelope changes are backward-compatible (additive fields only)
- Payload schema identified by `SchemaID` (consumers check version before deserializing)
- Consumers must handle unknown schema versions gracefully (log + skip, not crash)
- Schema registry (in-code, not external): maps `SchemaID` to Go struct type

### 2.4 Adapter gRPC Versioning

```protobuf
// Adapter plugin protocol version
service AdapterPlugin {
    rpc Handshake(HandshakeRequest) returns (HandshakeResponse);
    // Handshake includes protocol version negotiation
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

**Rules:**
- Host and plugin negotiate protocol version at connection time
- Host maintains backward compatibility for N-1 protocol versions
- Capability declaration allows feature detection without version bumps

### 2.5 Edge Agent Sync Versioning

```go
type SyncEnvelope struct {
    AgentVersion    string    // semver of agent binary
    ProtocolVersion int       // sync protocol version
    HubMinVersion   int       // minimum protocol version hub accepts
    Payload         json.RawMessage
}
```

**Rules:**
- Hub rejects agents with `ProtocolVersion < HubMinVersion`
- Rejected agents are directed to auto-update before reconnecting
- Hub maintains backward compatibility for N-1 protocol versions

### 2.6 Open Questions

- [ ] How long must old Temporal workflow versions be maintained? (Until all in-flight workflows complete — what is the maximum workflow duration?)
- [ ] Should NATS use protobuf instead of JSON for event payloads? (Performance vs. debuggability tradeoff)
- [ ] Should the adapter protocol version be a semver or an integer?

---

## 3. Temporal-PostgreSQL Shared Fate

### 3.1 The Problem

Temporal Server and LOOM share the same PostgreSQL instance (separate schemas). When PostgreSQL goes down, both the execution source of truth (Temporal) and the application projection (LOOM DB) are simultaneously unavailable.

### 3.2 Analysis

| Aspect | Shared PostgreSQL | Separate PostgreSQL |
|--------|------------------|-------------------|
| Operational complexity | Lower (one backup, one monitor) | Higher (two instances to manage) |
| Connection pool contention | Risk of Temporal starving LOOM or vice versa | Isolated pools |
| Failure correlation | Correlated (both fail together) | Independent |
| Recovery | Single restore, but both must be consistent | Independent recovery |
| Cost | Lower (one instance) | ~2x PostgreSQL cost |

### 3.3 Recommendations

**For Mode 1-2 (single site, <1000 devices):** Shared PostgreSQL is acceptable.
- Mitigation: Separate connection pools with hard limits (Temporal: 20 connections, LOOM: 60 connections out of max 100)
- Mitigation: Temporal schema in a separate PostgreSQL database (not just schema) on the same instance, enabling independent backup/restore

**For Mode 3-4 (multi-region, >1000 devices):** Separate PostgreSQL instances.
- Temporal Server uses its own PostgreSQL (or Cassandra for large-scale)
- LOOM application database is independent
- Separate monitoring and alerting

### 3.4 Open Questions

- [ ] What is Temporal's connection pool requirement under load? (Measure in Phase 4)
- [ ] Can a long LOOM migration block Temporal operations on the same instance?
- [ ] Should the deployment topology document specify the separation boundary?

---

## 4. ObservedFact Retention

### 4.1 The Problem

ObservedFact is append-only and immutable. At scale (10,000 devices, 10 facts per device per minute), this generates 144 million rows per day with no retention policy.

### 4.2 Recommended Retention Strategy

```
Hot tier (PostgreSQL, uncompressed):
  - Duration: 7 days
  - Use case: real-time queries, recent state, drift detection
  - Storage: ~10 GB/day at 10K devices

Warm tier (TimescaleDB compressed hypertable):
  - Duration: 90 days
  - Use case: trend analysis, historical comparison
  - Compression: ~10:1 ratio → ~1 GB/day
  - Total: ~90 GB

Cold tier (TimescaleDB continuous aggregate or export):
  - Duration: 1 year
  - Use case: compliance audit, long-term trends
  - Aggregation: hourly summaries (min/max/avg per fact type per device)
  - Storage: ~365 MB

Archive (S3/object storage):
  - Duration: 7 years (compliance)
  - Format: Parquet files, partitioned by tenant/month
  - Storage: ~50 MB/month compressed
```

### 4.3 Latest Fact Materialized View

For fast current-state queries:

```sql
CREATE MATERIALIZED VIEW latest_facts AS
SELECT DISTINCT ON (device_id, fact_type)
    device_id, fact_type, value, confidence, observed_at, source
FROM observed_facts
ORDER BY device_id, fact_type, observed_at DESC;

-- Refresh periodically (every 60 seconds) or use TimescaleDB continuous aggregate
```

### 4.4 Implementation

- ObservedFact table becomes a TimescaleDB hypertable (partitioned by `observed_at`)
- Compression policy: compress chunks older than 7 days
- Retention policy: drop raw data older than 90 days
- Continuous aggregate: hourly summaries retained for 1 year
- Export job: monthly Parquet export to object storage

### 4.5 Open Questions

- [ ] Should ObservedFact be a TimescaleDB hypertable from day one, or start as a plain table?
- [ ] What is the acceptable latency for `latest_facts` queries? (Materialized view refresh interval)
- [ ] Should tenants have configurable retention policies?

---

## 5. Temporal `continue-as-new` Design

### 5.1 The Problem

Temporal has a ~50K event hard limit per workflow execution. Long-running workflows (discovery scans, continuous monitoring) require `continue-as-new`, which creates a new execution with cascading implications.

### 5.2 Checkpoint State Format

```go
type WorkflowCheckpoint struct {
    // Carry-over state
    CorrelationID   string              // audit trail continuity
    OriginalRunID   string              // link back to first execution
    ContinueCount   int                 // how many times continued
    EventCount      int                 // events in current execution

    // Workflow-specific state (varies by workflow type)
    Progress        json.RawMessage     // opaque progress data
    PendingSignals  []PendingSignal     // signals that haven't been processed
    ActiveTimers    []ActiveTimer       // timers to re-register
}
```

### 5.3 Signal Handling Across `continue-as-new`

```go
// Before continue-as-new: drain pending signals
func drainSignals(ctx workflow.Context, ch workflow.ReceiveChannel) []PendingSignal {
    var pending []PendingSignal
    for {
        var signal SignalPayload
        ok := ch.ReceiveAsync(&signal)
        if !ok {
            break
        }
        pending = append(pending, PendingSignal{Signal: signal})
    }
    return pending
}

// After continue-as-new: re-process pending signals
func reprocessSignals(ctx workflow.Context, pending []PendingSignal) {
    for _, ps := range pending {
        handleSignal(ctx, ps.Signal)
    }
}
```

### 5.4 DB Projection Update

```go
// WorkflowStatus projection must track the latest RunID
type WorkflowStatus struct {
    WorkflowID      string
    CurrentRunID    string    // updated on continue-as-new
    OriginalRunID   string    // never changes
    ContinueCount   int
    // ... status fields
}

// Activity that updates the projection after continue-as-new
func UpdateProjectionActivity(ctx context.Context, params ProjectionUpdate) error {
    return db.UpdateWorkflowRunID(params.WorkflowID, params.NewRunID, params.ContinueCount)
}
```

### 5.5 Audit Trail Continuity

```go
// Audit records reference OriginalRunID for chain continuity
type AuditMetadata struct {
    WorkflowID      string
    OriginalRunID   string    // always the first execution's run ID
    CurrentRunID    string    // current execution's run ID
    ContinueCount   int
    CorrelationID   string    // same across all continuations
}
```

### 5.6 Open Questions

- [ ] What is the event threshold for `continue-as-new`? (10K? 25K? Configurable?)
- [ ] How do approval signals work across `continue-as-new`? (Approval service must re-signal the new execution)
- [ ] Should `continue-as-new` be transparent to the API consumer? (They should only see WorkflowID, not RunID changes)

---

## 6. Idempotency Cache

### 6.1 The Problem

The adapter contract says "adapters maintain an idempotency cache with a configurable TTL" but never specifies where, how it's shared across workers, or how it interacts with workflow pause duration.

### 6.2 Recommended Design: PostgreSQL + Valkey

```go
type IdempotencyRecord struct {
    Key         string        // idempotency key (from operation)
    TenantID    uuid.UUID
    Result      []byte        // serialized OperationResult
    CreatedAt   time.Time
    ExpiresAt   time.Time     // TTL-based expiry
}

// Two-tier cache:
// 1. Valkey (hot): sub-millisecond lookups, ~1 hour TTL
// 2. PostgreSQL (durable): long-lived records, TTL matches max workflow pause duration

func (c *IdempotencyCache) Check(ctx context.Context, key string) (*OperationResult, error) {
    // Check Valkey first (fast path)
    if result, err := c.valkey.Get(ctx, key); err == nil {
        return result, nil
    }
    // Fall back to PostgreSQL (durable path)
    return c.db.GetIdempotencyRecord(ctx, key)
}

func (c *IdempotencyCache) Store(ctx context.Context, key string, result OperationResult, ttl time.Duration) error {
    // Write to both (PostgreSQL first for durability)
    if err := c.db.StoreIdempotencyRecord(ctx, key, result, ttl); err != nil {
        return err
    }
    // Best-effort Valkey cache (short TTL for hot reads)
    _ = c.valkey.Set(ctx, key, result, min(ttl, 1*time.Hour))
    return nil
}
```

### 6.3 TTL Rules

| Scenario | TTL |
|----------|-----|
| Default operations | 24 hours |
| Operations in workflows with approval gates | Max workflow pause duration + 24 hours |
| Emergency/break-glass operations | 1 hour (short-lived) |
| Discovery operations | 4 hours (rediscovery is safe) |

### 6.4 Open Questions

- [ ] Should the idempotency cache be per-tenant or global? (Per-tenant keys prevent cross-tenant cache poisoning)
- [ ] What happens when both Valkey and PostgreSQL caches are unavailable? (Fail open = risk double execution; fail closed = operations blocked)
- [ ] Should Temporal's built-in activity result caching replace the external idempotency cache for workflow-initiated operations?

---

## 7. Schedule Risk Assessment

### 7.1 The Problem

The 12-phase plan estimates 68-95 weeks for 2-3 senior Go engineers. Several phases are underestimated.

### 7.2 Risk-Adjusted Timeline

| Phase | Original Estimate | Risk-Adjusted | Key Risk |
|-------|------------------|---------------|----------|
| 1A: Skeleton | 2-3 weeks | 3-4 weeks | Temporal embedded mode setup |
| 1B: Schema + Identity | 3-4 weeks | 5-6 weeks | Domain model gaps (this document) |
| 2: SSH + SNMP | 4-6 weeks | 5-7 weeks | First adapter is always hardest |
| 3: Redfish + IPMI | 4-6 weeks | 6-8 weeks | OEM divergence (see RESEARCH-protocol-adapter-realism.md) |
| 4: Temporal Workflows | 6-8 weeks | 8-12 weeks | No Temporal experience on team |
| 5: REST API + Auth | 4-6 weeks | 5-7 weeks | Multi-tenant RBAC complexity |
| 6: UI | 6-8 weeks | 8-10 weeks | Topology visualization, real-time updates |
| 7: Network Domain | 10-14 weeks | 14-20 weeks | NETCONF vendor fragmentation |
| 8: LLM + Cost | 6-8 weeks | 8-10 weeks | LLM integration latency, cost model |
| 9: Verification + Graph | 4-6 weeks | 5-7 weeks | AGE viability |
| 10: HA + Security | 6-8 weeks | 8-12 weeks | mTLS, vault, audit, Helm |
| 11: Edge | 6-8 weeks | 10-14 weeks | This document's findings |
| 12: Polish | 4-6 weeks | 4-6 weeks | — |
| **Total** | **68-95 weeks** | **95-127 weeks** | |

### 7.3 Recommendations

1. **Add 30% buffer per phase** — Standard for greenfield projects with unfamiliar technology (Temporal).

2. **Reorder: Phase 4 before Phase 3** — Validate Temporal (hardest architectural risk) before investing in more adapters. If Temporal doesn't work as assumed, the adapter investment is protected.

3. **Identify MVP-critical vs MVP-optional within phases:**
   - Phase 7 (Network): MVP needs 1 vendor (SONiC via gNMI), not 4
   - Phase 8 (LLM): MVP can use deterministic fallback only, defer LLM to post-MVP
   - Phase 9 (Graph): MVP can use recursive CTEs, defer AGE to post-MVP

4. **Build the device simulator in Phase 1B** — Enables scale testing throughout all subsequent phases.

### 7.4 Open Questions

- [ ] Is the team willing to extend the timeline to 2+ years, or should scope be cut?
- [ ] Which phases can be parallelized with additional engineers?
- [ ] Should Phase 7 (Network) target 1 vendor for MVP and expand post-MVP?

---

## 8. Temporal `GetVersion` Integration

### 8.1 The Problem

No LOOM document mentions Temporal's `workflow.GetVersion()` API, which is essential for evolving workflow definitions without breaking in-flight executions.

### 8.2 Rules for LOOM Workflow Development

```go
// Rule 1: Every workflow change that affects activity execution must use GetVersion
v := workflow.GetVersion(ctx, "change-id", workflow.DefaultVersion, currentVersion)

// Rule 2: Version change IDs are descriptive and unique
// Good: "add-verification-step-to-provisioning"
// Bad: "v2", "fix", "update"

// Rule 3: Old code paths are never deleted while in-flight workflows exist
// Use Temporal's visibility API to check: are there any open workflows with version < N?

// Rule 4: Activity input/output structs are append-only
// New fields get default values. Old fields are never removed.
// If a field must change type, add a new field and deprecate the old one.
```

### 8.3 Workflow Versioning Documentation Template

Each workflow should maintain a version history:

```go
// WorkflowVersionHistory:
// DefaultVersion (initial): basic provisioning with power-on, OS install
// Version 1 (2026-04-15): added pre-provisioning health check activity
// Version 2 (2026-05-01): changed OS install to use staged rollout
// Version 3 (2026-06-12): added post-provisioning verification step
```

### 8.4 Open Questions

- [ ] Should LOOM have a workflow version registry (database table tracking active versions)?
- [ ] How are old workflow versions cleaned up? (Periodic job that checks for zero open workflows?)
- [ ] Should the CI pipeline block deployment if a workflow change doesn't include `GetVersion`?

---

## 9. Implementation Priority

| Item | Phase | Effort | Dependencies |
|------|-------|--------|--------------|
| NATS event envelope versioning | 1A | 1 week | Event model finalized |
| Temporal `GetVersion` guidelines | 1A | 1 week | Documentation only |
| Adapter gRPC protocol versioning | 1B | 1-2 weeks | go-plugin integration |
| ObservedFact TimescaleDB hypertable | 1B | 1-2 weeks | TimescaleDB extension |
| Idempotency cache (PG + Valkey) | 2 | 1-2 weeks | Valkey deployment |
| `continue-as-new` checkpoint design | 4 | 2-3 weeks | First Temporal workflow |
| Latest facts materialized view | 2 | 1 week | ObservedFact hypertable |
| Edge sync protocol versioning | 11 | 1-2 weeks | Edge agent design |
| Schedule re-baseline | Now | 1 week | All research docs complete |

---

## 10. References

- WORKFLOW-CONTRACT.md — Temporal integration rules
- EVENT-MODEL.md — NATS event structure
- ADAPTER-CONTRACT.md — Adapter interfaces and idempotency
- ADR-004 — PostgreSQL as primary database
- ADR-008 — Temporal as execution truth
- DOMAIN-MODEL.md — ObservedFact definition
- IMPLEMENTATION-PLAN.md — Phase schedule and estimates
- [Temporal Versioning Guide](https://docs.temporal.io/workflows#workflow-versioning)
