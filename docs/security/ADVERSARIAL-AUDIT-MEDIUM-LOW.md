# Adversarial Audit: Medium and Low Findings

> **Date:** 2026-03-21
> **Source:** Adversarial security audit of all LOOM design documents
> **Status:** All 27 MEDIUM and 10 LOW findings addressed with concrete decisions
> **Scope:** This document covers findings that are individually non-critical but collectively represent significant attack surface if unaddressed.

---

## MEDIUM FINDINGS

---

### M-01: NTP Manipulation on Edge Agents

**Attack:** An attacker who controls NTP responses to edge agents can skew time, causing certificate validation failures, replay window manipulation, or audit trail inconsistency.

**Decision:** Require NTS (Network Time Security, RFC 8915) for all edge agent NTP sources. NTS provides authenticated time synchronization without the vulnerabilities of NTPv4 symmetric key authentication.

**Fallback:** When NTS is unavailable (legacy environments), agents MUST configure at least 3 independent NTP sources. If any two sources disagree by more than 5 seconds, the agent rejects the time update and emits a `security.ntp.disagreement` alert. The agent continues using its last-known-good time until the discrepancy resolves.

**Go types:**

```go
type NTPConfig struct {
    Sources       []NTPSource   // Minimum 3 required
    MaxSkew       time.Duration // 5 seconds — reject if sources disagree by more
    RequireNTS    bool          // true by default; false only with explicit operator override
    LastKnownGood time.Time     // fallback if sources disagree
}

type NTPSource struct {
    Address   string // NTP server address
    NTSEnabled bool  // true if NTS is available for this source
    Priority  int    // lower = preferred
}
```

**Cross-reference:** EDGE-AGENT-SECURITY.md (time-dependent cache expiry), SECURITY-CHECKLIST.md (add NTP verification item)

---

### M-02: Valkey Session Key Derived from Valkey Credentials

**Attack:** SECURITY-HARDENING-RESPONSES.md Section SH-11 derives the session encryption key from Valkey credentials via HKDF. If Valkey credentials are compromised, both the data store AND the encryption are compromised simultaneously — no defense in depth.

**Decision:** Use a separate session encryption key retrieved from the vault, NOT derived from Valkey credentials. The session encryption key is a dedicated secret stored in the vault under `kv/loom/session-encryption-key`, rotated independently from Valkey credentials.

**Go types:**

```go
type SessionEncryptionConfig struct {
    // KeySource is always "vault" — never derived from Valkey credentials
    KeySource     string // "vault"
    VaultPath     string // "kv/loom/session-encryption-key"
    KeyVersion    int    // current active version
    RotationDays  int    // 30-day rotation
}
```

**Cross-reference:** SECURITY-HARDENING-RESPONSES.md Section 7 (Valkey Security), VAULT-ARCHITECTURE.md (key hierarchy)

---

### M-03: P0 Events Unencrypted on Edge

**Attack:** The edge agent's P0 event queue (SQLite-based, for store-and-forward during disconnection) stores high-priority events in plaintext. Physical access to the edge device exposes event contents including device state changes, credential usage records, and workflow results.

**Decision:** Encrypt the P0 event queue SQLite database using the same edge KEK that protects the credential cache. Events are encrypted at the column level (event payload) using AES-256-GCM with per-event nonces. Event metadata (timestamp, sequence number, delivery status) remains unencrypted for queue management.

**Go types:**

```go
type EncryptedP0Event struct {
    SequenceNum    int64     // unencrypted — queue ordering
    Timestamp      time.Time // unencrypted — expiry management
    DeliveryStatus string    // unencrypted — "pending", "delivered", "failed"
    EncryptedPayload []byte  // AES-256-GCM ciphertext of event payload
    Nonce          []byte    // per-event GCM nonce
    KEKVersion     int       // which KEK version encrypted this event
}
```

**Cross-reference:** EDGE-AGENT-SECURITY.md Section 1 (EdgeCredentialCache uses same KEK pattern), EVENT-MODEL.md

---

### M-04: RLS Bypass via Stale Tenant Context

**Attack:** PostgreSQL Row-Level Security depends on `current_setting('app.current_tenant_id')`. If a connection is reused from a pool and the previous request's tenant context is not cleared, the next request executes with the wrong tenant's permissions.

**Decision:** Mandatory `SET app.current_tenant_id = $1` at the start of every transaction, enforced by middleware — not by individual query code. The middleware acquires the connection, sets the tenant context, and only then passes control to the handler.

**Verification:** Add an integration test that:
1. Sets tenant context to Tenant A
2. Returns connection to pool without clearing context
3. Acquires a new connection
4. Asserts that `current_setting('app.current_tenant_id')` is empty (not Tenant A)
5. Verifies that queries fail if tenant context is not explicitly set

**Go types:**

```go
// TenantContextMiddleware ensures every DB transaction has an explicit tenant context.
// This is the ONLY code path that sets app.current_tenant_id.
type TenantContextMiddleware struct {
    DB *sql.DB
}

func (m *TenantContextMiddleware) WithTenant(ctx context.Context, tenantID uuid.UUID, fn func(tx *sql.Tx) error) error {
    tx, err := m.DB.BeginTx(ctx, nil)
    if err != nil {
        return err
    }
    defer tx.Rollback()

    // MUST be first statement in every transaction
    if _, err := tx.ExecContext(ctx, "SET LOCAL app.current_tenant_id = $1", tenantID.String()); err != nil {
        return fmt.Errorf("failed to set tenant context: %w", err)
    }

    if err := fn(tx); err != nil {
        return err
    }
    return tx.Commit()
}
```

**Cross-reference:** ATTACK-SURFACE-POSTGRESQL.md (Attack 2: Cross-Tenant Data Access), SECURITY-CHECKLIST.md (RLS verification)

---

### M-05: RAG Prompt Injection via Cross-Tenant Document Retrieval

**Attack:** If RAG retrieval does not strictly filter by tenant, Tenant A could craft documents containing prompt injection payloads that appear in Tenant B's LLM context.

**Decision:** Per-tenant RAG index isolation. Every pgvector similarity search MUST include a `WHERE tenant_id = $1` filter. This is enforced at the repository layer — no RAG query function accepts a call without a tenant_id parameter.

**Enforcement:** The `RAGRepository` interface requires `tenantID` as the first parameter on all query methods. There is no method signature that omits it.

**Go types:**

```go
type RAGRepository interface {
    // Search always filters by tenantID. There is no cross-tenant search method.
    Search(ctx context.Context, tenantID uuid.UUID, embedding []float32, limit int) ([]RAGResult, error)
    // Index stores a document scoped to a tenant.
    Index(ctx context.Context, tenantID uuid.UUID, doc RAGDocument) error
}
```

**Cross-reference:** SECURITY-HARDENING-RESPONSES.md Section 3 (LLM Context Isolation), ATTACK-SURFACE-LLM.md, LLM-BOUNDARIES.md

---

### M-06: NATS Isolation Inconsistency Across Documents

**Attack:** Some documents reference subject-based ACLs as the isolation mechanism, others reference NATS accounts. If implementation follows subject ACLs alone, any NATS client that subscribes to `loom.>` sees all tenant events.

**Decision:** NATS accounts (not just subject ACLs) are the authoritative isolation mechanism. Each tenant gets a dedicated NATS account. Subject ACLs within accounts provide fine-grained control. All documentation must be consistent on this point.

**Normative statement:** Subject-based ACLs alone are ZERO isolation. They are permission checks within an account, not security boundaries between tenants. The account boundary is the security boundary.

**Cross-reference:** ATTACK-SURFACE-NATS.md (Attack 1), SECURITY-HARDENING-RESPONSES.md Section SH-12, SECURITY-CHECKLIST.md (NATS account verification)

---

### M-07: Failed Credential Rotation Leaves Old Credential in Active State

**Attack:** If a credential rotation fails midway (new credential created on device, but vault update fails), the old credential remains marked `active` in the vault while the device has already changed to the new credential. Subsequent operations use the stale credential and fail.

**Decision:** On rotation failure:
1. Mark old credential as `rotation_failed` (NOT `active` and NOT deleted)
2. Emit `security.credential.rotation_failed` alert with device ID and credential type
3. Do NOT delete the old credential — it may still work if the device-side change did not complete
4. Schedule automatic retry in 1 hour
5. After 3 failed retries, escalate to operator for manual resolution

**Go types:**

```go
type CredentialStatus string

const (
    CredentialStatusActive         CredentialStatus = "active"
    CredentialStatusRotating       CredentialStatus = "rotating"
    CredentialStatusRotationFailed CredentialStatus = "rotation_failed"
    CredentialStatusRevoked        CredentialStatus = "revoked"
    CredentialStatusExpired        CredentialStatus = "expired"
)

type RotationFailureRecord struct {
    CredentialID uuid.UUID
    DeviceID     uuid.UUID
    FailedAt     time.Time
    Reason       string
    RetryCount   int       // max 3
    NextRetryAt  time.Time // FailedAt + (RetryCount * 1 hour)
}
```

**Cross-reference:** VAULT-ARCHITECTURE.md (credential lifecycle), ADAPTER-CONTRACT.md (rotation operations)

---

### M-08: Shared Device Lock Contention Discloses Cross-Tenant Information

**Attack:** When Tenant A holds a lock on a shared device (e.g., a network switch), the lock contention response to Tenant B could reveal Tenant A's workflow ID, tenant ID, or operation type — leaking cross-tenant operational information.

**Decision:** Lock contention responses return ONLY generic information: `"device busy, retry after {seconds}"`. No workflow ID, no tenant ID, no operation type, no holder identity. The retry-after duration is the lock's TTL — not the expected completion time (which would leak information about the operation).

**Go types:**

```go
type LockContentionResponse struct {
    DeviceBusy   bool          `json:"device_busy"`
    RetryAfterS  int           `json:"retry_after_seconds"` // lock TTL, no operation info
    // No workflow ID, no tenant ID, no operation type — intentionally absent
}
```

**Cross-reference:** DEVICE-LOCKING.md (lock granularity model)

---

### M-09: Endpoint.TenantID Divergence from Device.TenantID

**Attack:** If `Endpoint` has its own `TenantID` field separate from its parent `Device.TenantID`, a bug or migration error could cause divergence. An endpoint could appear to belong to Tenant A while its parent device belongs to Tenant B, creating RLS confusion and potential cross-tenant access.

**Decision:** Remove `Endpoint.TenantID`. Endpoints inherit tenant ownership exclusively from their parent `Device`. This is a single-source-of-truth enforcement at the type level.

**Go types:**

```go
type Endpoint struct {
    ID         uuid.UUID      `json:"id" db:"id"`
    DeviceID   uuid.UUID      `json:"device_id" db:"device_id"` // FK to devices.id
    // TenantID intentionally absent — inherited from Device via JOIN
    Protocol   string         `json:"protocol" db:"protocol"`
    Address    string         `json:"address" db:"address"`
    Port       int            `json:"port" db:"port"`
    Status     EndpointStatus `json:"status" db:"status"`
}
```

**Database enforcement:** RLS policy on `endpoints` table joins through `devices` to get `tenant_id`:

```sql
CREATE POLICY endpoints_tenant_isolation ON endpoints
    USING (device_id IN (
        SELECT id FROM devices
        WHERE tenant_id = current_setting('app.current_tenant_id')::uuid
    ));
```

**Cross-reference:** DOMAIN-MODEL.md (Endpoint struct), ATTACK-SURFACE-POSTGRESQL.md (RLS policies)

---

### M-10: Advisory Lock Hash Collisions

**Attack:** Advisory lock keys are 64-bit integers derived from device UUIDs by hashing. If the hash function has poor distribution (e.g., FNV-64a on UUIDs), collisions cause unrelated devices to contend on the same lock, creating phantom deadlocks and throughput reduction.

**Decision:** Use SHA-256 truncated to 64 bits for advisory lock key derivation. SHA-256 has excellent distribution properties for UUID inputs. Truncation to 64 bits (PostgreSQL bigint) yields a collision probability of approximately 1 in 2^32 at 65,536 devices (birthday paradox).

**Documented collision probability:**

| Device Count | Collision Probability |
|-------------|----------------------|
| 1,000 | ~0.00003% |
| 10,000 | ~0.003% |
| 100,000 | ~0.3% |
| 1,000,000 | ~23% |

At LOOM's target scale (10K-100K devices), collision risk is negligible. At 1M+ devices, consider expanding to two advisory locks per device (128-bit coverage).

**Cross-reference:** DEVICE-LOCKING.md (lock key derivation)

---

### M-11: Lock Release Across Approval Gates (TOCTOU)

**Attack:** A workflow acquires a device lock, sends a config change for human approval, and holds the lock for the entire approval duration (minutes to hours). This blocks all other workflows on that device. Alternatively, if the lock is released during approval, another workflow can change the device state between validation and execution (TOCTOU).

**Decision:** Two-phase locking:
1. **Intent lock** during approval: lightweight lock that blocks new mutating workflows but does NOT block monitoring, status checks, or read operations. The intent lock declares "a change is pending human approval for this device."
2. **Full exclusive lock** on execution: acquired after approval, before execution. If the device state has changed since validation (detected via optimistic version check), the workflow restarts from validation.

**Go types:**

```go
type LockType string

const (
    LockTypeIntent    LockType = "intent"    // blocks mutations, allows reads
    LockTypeExclusive LockType = "exclusive"  // blocks everything
)

type DeviceLockRequest struct {
    DeviceID    uuid.UUID
    WorkflowID  string
    LockType    LockType
    TTL         time.Duration // intent: 1 hour max; exclusive: 5 minutes max
}
```

**Cross-reference:** DEVICE-LOCKING.md, WORKFLOW-CONTRACT.md (approval gates)

---

### M-12: CompensationReliability Self-Declared Without Verification

**Attack:** An adapter claims `CompensationReliability: "transactional"` for marketing purposes but its rollback implementation is buggy or incomplete. The workflow engine trusts this claim and presents operations as safely reversible when they are not.

**Decision:** Verify via conformance test suite. Any adapter claiming `"transactional"` or `"guaranteed"` compensation MUST pass a standardized rollback test:
1. Execute the operation (assert success)
2. Execute compensation (assert success)
3. Assert device state matches pre-operation snapshot

If the test fails, the adapter's `CompensationReliability` is forcibly downgraded to `"best_effort"` in the adapter registry. This downgrade is logged and visible in the operator console.

**Go types:**

```go
type CompensationReliability string

const (
    CompensationGuaranteed  CompensationReliability = "guaranteed"   // passed conformance test
    CompensationBestEffort  CompensationReliability = "best_effort"  // no test or test failed
    CompensationNone        CompensationReliability = "none"         // no compensation possible
)

type CompensationConformanceResult struct {
    AdapterName   string
    OperationType string
    Claimed       CompensationReliability
    Verified      CompensationReliability // may differ from Claimed
    TestPassed    bool
    FailureReason string // empty if passed
    TestedAt      time.Time
}
```

**Cross-reference:** ADAPTER-CONTRACT.md (CompensationInfo), ADVERSARIAL-REVIEW.md Section 1.1 (rollback reliability), TESTING-STRATEGY.md

---

### M-13: ConfigIntent Template Injection

**Attack:** `ConfigIntent` uses Go `text/template` for parameter interpolation. If user-supplied parameters contain `{{` or `}}`, they can inject arbitrary template directives including function calls (`{{ .Exec "rm -rf /" }}`).

**Decision:** Two mitigations applied together:
1. **Input validation:** Reject any parameter value containing `{{` or `}}`. This blocks template injection at the input boundary.
2. **Safe templating:** Use `text/template` with `html/template`'s escaping functions applied to all parameter values via a custom `FuncMap`. Parameters are always passed through `html.EscapeString` before interpolation.

**Go types:**

```go
// ValidateConfigIntentParams rejects parameters that could inject template directives.
func ValidateConfigIntentParams(params map[string]string) error {
    for key, value := range params {
        if strings.Contains(value, "{{") || strings.Contains(value, "}}") {
            return fmt.Errorf("parameter %q contains forbidden template delimiters", key)
        }
    }
    return nil
}
```

**Cross-reference:** OPERATION-TYPES.md (ConfigIntent operations), REQUEST-TO-DEVICE-FLOW.md

---

### M-14: Workflow Event Overflow Without Compensation Checkpoint

**Attack:** Temporal workflows have a hard 10,000-event limit. If a long-running workflow approaches this limit without checkpointing its compensation state, a continue-as-new loses the compensation history, making rollback impossible.

**Decision:** Two thresholds:
- **5,000 events:** Normal continue-as-new (already specified in WORKFLOW-CONTRACT.md). Compensation state is serialized into the continue-as-new input.
- **9,000 events (emergency):** Force continue-as-new with a full compensation checkpoint. The checkpoint includes the complete LIFO compensation stack serialized as workflow input. This is a safety net — reaching 9,000 events indicates the 5,000 threshold was missed or a sub-workflow is unexpectedly large.
- **10,000 events:** MUST never be reached. The 9,000 threshold is the hard backstop.

**Go types:**

```go
const (
    ContinueAsNewThreshold     = 5000 // normal continuation
    EmergencyContinueThreshold = 9000 // forced continuation with compensation snapshot
    TemporalHardLimit          = 10000 // must never reach
)

type CompensationCheckpoint struct {
    Stack         []CompensationEntry // LIFO order
    WorkflowRunID string
    CheckpointAt  time.Time
    EventCount    int
}
```

**Cross-reference:** WORKFLOW-CONTRACT.md (continue-as-new policy), FAILURE-MODES.md

---

### M-15: Circuit Breaker Abuse

**Attack:** An attacker with tenant-level API access could intentionally trigger failures to open a circuit breaker, causing a denial-of-service for that device/adapter combination across all tenants.

**Decision:**
- **Minimum 3-minute cooldown** before a circuit breaker can transition from open to half-open
- **Circuit cannot be opened by external request** — only by observed failure rate exceeding the threshold (default: 50% failure rate over 10 requests in a 5-minute sliding window)
- **Manual override** (force-open or force-close) requires operator approval and emits an audit event
- **Per-tenant failure tracking** — Tenant A's failures do not count toward Tenant B's circuit breaker state

**Go types:**

```go
type CircuitBreakerConfig struct {
    FailureThreshold  float64       // 0.5 (50%)
    WindowSize        int           // 10 requests
    WindowDuration    time.Duration // 5 minutes
    MinCooldown       time.Duration // 3 minutes minimum
    PerTenant         bool          // true — tenant-isolated failure tracking
}
```

**Cross-reference:** FAILURE-MODES.md (circuit breaker policy), ADAPTER-CONTRACT.md

---

### M-16: DryRun Side Effects from Simulated Execution

**Attack:** An adapter without native dry-run support might "simulate" a dry-run by executing the operation and then rolling it back. The execution itself has side effects (network state changes, audit logs on the target device, SNMP traps) that leak information or cause disruption.

**Decision:** Adapters without native dry-run support MUST return `UnsupportedError` for `DryRun=true`. No simulation by execution. No "execute then rollback" pattern.

Document which adapters support native dry-run:

| Adapter | Native DryRun | Mechanism |
|---------|--------------|-----------|
| Junos NETCONF | Yes | `<commit-check/>` |
| Cisco IOS-XE NETCONF | Yes | `<validate>` on candidate config |
| Arista eAPI | Partial | `configure session` (syntax check only) |
| Redfish | No | UnsupportedError |
| SSH/CLI | No | UnsupportedError |
| IPMI | No | UnsupportedError |
| Terraform | Yes | `terraform plan` |
| Ansible | Yes | `--check` mode |

**Cross-reference:** ADAPTER-CONTRACT.md (DryRun field), OPERATION-TYPES.md

---

### M-17: /healthz Unauthenticated Information Disclosure

**Attack:** An unauthenticated `/healthz` endpoint that returns component-level health details (database status, NATS connectivity, vault reachability) gives an attacker a free reconnaissance tool.

**Decision:** Split into two endpoints:

| Endpoint | Auth | Response |
|----------|------|----------|
| `/healthz` | None | `{"status": "ok"}` or `{"status": "degraded"}` — no component details |
| `/api/v1/health` | Required (operator role) | Full component health: DB, NATS, Vault, Temporal, adapters |

The unauthenticated `/healthz` exists solely for Kubernetes liveness probes and load balancer health checks. It returns a single boolean signal with zero operational detail.

**Cross-reference:** API-CONVENTIONS.md, DEPLOYMENT-TOPOLOGY.md (Kubernetes probe configuration)

---

### M-18: `loom._system.*` NATS Subjects Lack ACLs

**Attack:** System subjects (`loom._system.>`) carry internal coordination messages (leader election, schema migrations, feature flag updates). If tenant accounts can subscribe to these subjects, they gain visibility into LOOM's internal state.

**Decision:** System subjects are restricted to LOOM service accounts only via NATS account permissions. No tenant account can publish to or subscribe to `loom._system.>`.

**NATS account configuration:**

```
accounts {
    LOOM_SYSTEM {
        users: [{ user: "loom-hub" }, { user: "loom-worker" }, { user: "loom-agent" }]
        exports: []  // no exports to tenant accounts
    }
    TENANT_TEMPLATE {
        imports: []  // no imports from LOOM_SYSTEM
        users: [{ user: "tenant-$TENANT_ID" }]
    }
}
```

**Cross-reference:** ATTACK-SURFACE-NATS.md, SECURITY-HARDENING-RESPONSES.md Section SH-12

---

### M-19: Deterministic Fallback Predictability

**Attack:** When the LLM is unavailable, LOOM falls back to deterministic template-based config generation and placement. If the deterministic logic is fully predictable, an attacker can predict exactly which server will be assigned to a workload and pre-position exploits.

**Decision:**
- Rotate template sets periodically (monthly or on operator trigger)
- Add randomization to deterministic decisions: when multiple placements are equally valid (same cost, same capacity), select randomly among them rather than using a fixed ordering
- Log the randomization seed per decision for audit reproducibility

**Cross-reference:** LLM-BOUNDARIES.md (fallback mode), PLACEMENT-POLICY.md

---

### M-20: No Per-Tenant LLM Rate Limits

**Attack:** A single tenant can exhaust the LLM token budget for the entire LOOM deployment by submitting rapid-fire complex queries. This denies LLM-assisted operations to all other tenants.

**Decision:** Per-tenant token budget enforced at the LLM gateway before the API call reaches the LLM provider. The budget is configurable per tenant with a system-wide default.

**Go types:**

```go
type LLMTenantBudget struct {
    TenantID         uuid.UUID
    TokensPerHour    int64   // default: 100,000
    TokensPerDay     int64   // default: 1,000,000
    CurrentHourUsage int64
    CurrentDayUsage  int64
    LastResetHour    time.Time
    LastResetDay     time.Time
}
```

**Cross-reference:** APPLICATION-LAYER-RESPONSES.md (AL-5 — LLM rate limiting), LLM-BOUNDARIES.md

---

### M-21: Compromised Edge Agent Overwrites Hub Data

**Attack:** A compromised edge agent could report fabricated facts (fake hardware specs, false health status, invented endpoints) that overwrite legitimate data in the hub's database.

**Decision:** Hub validates agent-reported facts against last-known state:
- **Anomaly detection:** If more than 50% of a device's facts change simultaneously in a single report, the update is flagged for human review (not automatically applied)
- **Rate limiting:** Facts for a single device can be updated at most once per minute
- **Immutable facts:** Certain identity facts (serial number, BIOS UUID) are append-only after first observation. An agent cannot overwrite them, only add new observations

**Go types:**

```go
type FactValidationResult struct {
    DeviceID       uuid.UUID
    TotalFacts     int
    ChangedFacts   int
    ChangeRatio    float64 // ChangedFacts / TotalFacts
    RequiresReview bool    // true if ChangeRatio > 0.5
    Reason         string  // "bulk_change_detected" or ""
}
```

**Cross-reference:** IDENTITY-MODEL.md (ExternalIdentity append-only), DISCOVERY-CONTRACT.md

---

### M-22: Budget Enforcement Only at Workflow Start

**Attack:** A workflow checks budget at start, gets approval for an estimated $50 operation, but the actual cost escalates to $500 as activities execute. The budget is only checked once at initiation, not during execution.

**Decision:** Add mid-execution budget check before each major activity. If the projected cost of the remaining activities exceeds the tenant's remaining budget, the workflow pauses and emits a `cost.budget.exceeded` alert. The operator can either increase the budget or cancel the workflow (triggering compensation).

**Go types:**

```go
type BudgetCheck struct {
    TenantID       uuid.UUID
    WorkflowID     string
    SpentSoFar     int64 // cents
    ProjectedTotal int64 // cents — estimated remaining + spent
    BudgetLimit    int64 // cents — tenant's configured limit
    Exceeded       bool
}
```

**Cross-reference:** COST-MODEL.md, WORKFLOW-CONTRACT.md (activity pre-checks)

---

### M-23: Cost Attribution to _platform Pseudo-Tenant

**Attack:** The `_platform` pseudo-tenant absorbs shared infrastructure costs. If regular tenants can attribute their costs to `_platform`, they avoid billing for their actual usage.

**Decision:** `_platform` cost attribution requires admin-level permissions. Regular tenant API keys cannot assign, modify, or query `_platform` costs. Cost entries with `tenant_id = _platform` are created only by the system's cost allocation engine, never by tenant-initiated API calls.

**Cross-reference:** COST-MODEL.md, SECURITY-MODEL.md (role hierarchy)

---

### M-24: Feature Flags Without Audit Trail

**Attack:** A malicious insider changes a feature flag (e.g., disabling MFA enforcement or enabling debug mode) with no record of who changed it, when, or why.

**Decision:** Feature flag changes:
1. Emit a NATS audit event on `loom._system.audit.feature_flag`
2. Feature flag table includes `changed_by` (user ID), `changed_at` (timestamp), and `reason` (mandatory text field)
3. Feature flag changes are immutable — each change creates a new row, old rows are retained for audit

**Go types:**

```go
type FeatureFlagChange struct {
    FlagName   string    `db:"flag_name"`
    OldValue   string    `db:"old_value"`
    NewValue   string    `db:"new_value"`
    ChangedBy  uuid.UUID `db:"changed_by"`
    ChangedAt  time.Time `db:"changed_at"`
    Reason     string    `db:"reason"` // mandatory — empty string rejected
}
```

**Cross-reference:** AUDIT-MODEL.md, SECURITY-MODEL.md (change tracking)

---

### M-25: No `go mod verify` in CI Pipeline

**Attack:** A supply chain attack replaces a dependency module in the Go module cache or proxy. Without `go mod verify`, the build proceeds with tampered dependencies.

**Decision:** Add `go mod verify` as a mandatory step before `go build` in the CI pipeline. If checksums do not match `go.sum`, the build fails immediately. This step is non-skippable — no CI override flag.

**CI step:**

```yaml
- name: Verify module checksums
  run: go mod verify
  # No "continue-on-error" — build MUST fail if checksums mismatch
```

**Cross-reference:** CI-CD-PIPELINE.md, RELEASE-PROCESS.md, ATTACK-SURFACE-UI-SUPPLY-CHAIN.md

---

### M-26: License Bypass via Binary Patching

**Attack:** Since Go binaries contain all licensing logic locally, an attacker can patch the binary to skip license checks.

**Decision:** Acknowledge this as an inherent limitation of client-side license enforcement. Primary protection is legal (EULA, copyright law, trade secret protections) combined with distribution control (signed binaries, authenticated download).

**Additional mitigation for cloud-connected deployments:** Add server-side license validation. The LOOM hub periodically phones home to a license validation server (opt-in for on-prem, mandatory for cloud/SaaS). If the license check fails, the hub enters a 30-day grace period with degraded functionality (no new workflow creation, existing workflows continue).

**Cross-reference:** BINARY-PROTECTION.md Section 7 (License Enforcement)

---

### M-27: Identity Phase 1 to Phase 2 Gap (TOCTOU)

**Attack:** The identity resolution pipeline has two phases: Phase 1 (collect identities from multiple sources) and Phase 2 (merge/match into a Device record). If the device is modified by another workflow between Phase 1 and Phase 2, the merge decision is based on stale data.

**Decision:** Add optimistic lock version check. The `Device` record includes a `version` column (monotonically incrementing integer). Phase 1 reads the current version. Phase 2 performs a conditional update: `UPDATE devices SET ... WHERE id = $1 AND version = $2`. If the version has changed (0 rows affected), the pipeline restarts from Phase 1 with the new state.

**Go types:**

```go
type DeviceVersionCheck struct {
    DeviceID uuid.UUID
    Version  int64 // read at Phase 1, checked at Phase 2
}

// MergeWithVersionCheck atomically merges identity data only if version matches.
// Returns ErrVersionConflict if the device was modified between Phase 1 and Phase 2.
func MergeWithVersionCheck(ctx context.Context, tx *sql.Tx, deviceID uuid.UUID, expectedVersion int64, mergeData IdentityMerge) error
```

**Cross-reference:** IDENTITY-MODEL.md (Phase 1/Phase 2 pipeline), DOMAIN-MODEL.md (Device struct)

---

## LOW FINDINGS

---

### L-01: wipeBytes Copies to Heap

**Decision:** Use `mmap`-allocated bytes (not heap) for `SecureBytes`. Already specified in VAULT-ARCHITECTURE.md. The `SecureBytes` type allocates via `syscall.Mmap` with `PROT_READ|PROT_WRITE` and `mlock` to prevent swap. The Go garbage collector never sees these bytes, so it cannot copy them.

**Cross-reference:** VAULT-ARCHITECTURE.md (SecureBytes implementation)

---

### L-02: CSP `unsafe-inline` for Styles

**Decision:** Move to nonce-based CSP. Each HTTP response generates a cryptographically random nonce. The CSP header becomes `style-src 'self' 'nonce-{random}'`. All inline `<style>` tags must include the matching `nonce` attribute. This eliminates `unsafe-inline` while permitting necessary inline styles.

**Cross-reference:** ATTACK-SURFACE-UI-SUPPLY-CHAIN.md (Content Security Policy)

---

### L-03: HSM PIN Rotation Not Scheduled

**Decision:** Add to SECURITY-CHECKLIST.md: "HSM PIN/passphrase rotation every 90 days. Rotation requires dual authorization (same ceremony as break-glass token issuance). Previous PINs are recorded in sealed audit envelope, not in plaintext logs."

**Cross-reference:** SECURITY-CHECKLIST.md, VAULT-ARCHITECTURE.md (MEK source: HSM)

---

### L-04: Per-Namespace DataConverter Key Allows Tenant Admin to Read Workflow History

**Decision:** This is by design, not a vulnerability. A tenant admin with Temporal namespace access CAN read their own tenant's workflow history (because the DataConverter key is scoped to their namespace). This is an intentional access grant — tenant admins should be able to inspect their own workflow execution details. Document this explicitly so it is not mistakenly reported as a finding in future audits.

**Cross-reference:** SECURITY-HARDENING-RESPONSES.md Section 2 (Temporal DataConverter Encryption)

---

### L-05: X-Correlation-ID Log Forging

**Decision:** Validate format: accept only UUID v4 values for `X-Correlation-ID`. If the header value is not a valid UUID v4, discard it and generate a server-side UUID v4 replacement. This prevents attackers from injecting multiline strings, ANSI escape sequences, or other log-forging payloads via the correlation ID header.

**Validation regex:** `^[0-9a-f]{8}-[0-9a-f]{4}-4[0-9a-f]{3}-[89ab][0-9a-f]{3}-[0-9a-f]{12}$`

**Cross-reference:** API-CONVENTIONS.md (request headers), OBSERVABILITY-STRATEGY.md

---

### L-06 / L-07: garble Obfuscation Limitations

**Decision:** Already documented in BINARY-PROTECTION.md as "raises the cost of reverse engineering, not bulletproof." No change needed. `garble` renames symbols and encrypts strings but cannot prevent a determined attacker with sufficient time. This is accepted — the goal is to raise the cost of casual reverse engineering, not to achieve theoretical unbreakability.

**Cross-reference:** BINARY-PROTECTION.md Sections 3-4

---

### L-08: Anti-Debug Trivially Bypassed

**Decision:** Already documented in BINARY-PROTECTION.md. Anti-debug measures (ptrace detection, timing checks) are speed bumps, not walls. They deter casual analysis and automated tooling. A skilled reverse engineer can bypass them in minutes. This is accepted.

**Cross-reference:** BINARY-PROTECTION.md Section 6

---

### L-09: Log Scrubbing Uses Blocklist Instead of Allowlist

**Decision:** Change to allowlist approach. Only fields explicitly listed in the logging allowlist are emitted in plaintext. All other fields are logged as `[FIELD:redacted]`. This inverts the security posture: instead of trying to enumerate every sensitive pattern (and missing some), we enumerate every safe pattern (and redact everything else).

The allowlist is maintained per log event type. Unknown fields in dynamic data structures are always redacted.

**Cross-reference:** OBSERVABILITY-STRATEGY.md (structured logging), SECURITY-MODEL.md

---

### L-10: Advisory Lock Cleanup 60-Second Window

**Decision:** Reduce to 15-second cleanup interval for the lock table. Stale lock detection runs every 15 seconds instead of every 60 seconds. This reduces the maximum time a stale lock can block operations from 60 seconds to 15 seconds. Session-level advisory locks are automatically released on disconnect; this cleanup targets transaction-level locks where the connection survived but the transaction was abandoned.

**Cross-reference:** DEVICE-LOCKING.md (lock cleanup)

---

## Implementation Priority

| Finding | Phase | Effort | Risk if Deferred |
|---------|-------|--------|-----------------|
| M-04 (RLS stale context) | 1A | 2 days | Critical — tenant isolation bypass |
| M-06 (NATS accounts) | 1A | 1 day | Critical — tenant isolation bypass |
| M-09 (Endpoint.TenantID) | 1A | 1 day | High — data model inconsistency |
| M-05 (RAG tenant filter) | 1B | 1 day | High — cross-tenant data leak |
| M-17 (/healthz split) | 1B | 0.5 day | Medium — reconnaissance |
| M-18 (system subject ACLs) | 1B | 0.5 day | Medium — internal state exposure |
| M-02 (Valkey session key) | 1B | 1 day | Medium — defense in depth |
| M-03 (P0 event encryption) | 2 | 2 days | Medium — edge data exposure |
| M-07 (rotation failure) | 2 | 2 days | High — credential management |
| M-08 (lock info disclosure) | 2 | 0.5 day | Low — information leak |
| M-10 (advisory lock hash) | 2 | 0.5 day | Low — phantom contention |
| M-11 (intent locks) | 2 | 3 days | Medium — TOCTOU |
| M-12 (compensation verify) | 3 | 1 week | Medium — false safety guarantee |
| M-13 (template injection) | 3 | 1 day | High — code injection |
| M-14 (event overflow) | 3 | 1 day | Medium — compensation loss |
| M-15 (circuit breaker) | 3 | 2 days | Medium — DoS vector |
| M-16 (dry-run safety) | 3 | 1 day | Medium — side effects |
| M-19 (deterministic fallback) | 4 | 1 day | Low — predictability |
| M-20 (LLM rate limits) | 8 | 2 days | Medium — resource exhaustion |
| M-21 (agent fact validation) | 2 | 3 days | High — data integrity |
| M-22 (mid-execution budget) | 8 | 2 days | Low — cost overrun |
| M-23 (platform cost) | 8 | 1 day | Low — billing manipulation |
| M-24 (feature flag audit) | 2 | 1 day | Medium — insider threat |
| M-25 (go mod verify) | 1A | 0.5 day | High — supply chain |
| M-26 (license bypass) | 10+ | 1 week | Low — accepted limitation |
| M-27 (identity TOCTOU) | 2 | 2 days | Medium — data corruption |
| M-01 (NTP) | 2 | 2 days | Medium — time manipulation |
| L-01 through L-10 | Various | 1-2 days each | Low |

---

## References

- [ADVERSARIAL-REVIEW.md](../ADVERSARIAL-REVIEW.md) — Original adversarial architecture review
- [MEDIUM-FINDINGS-RESPONSES.md](../MEDIUM-FINDINGS-RESPONSES.md) — Prior medium findings (M1-M24 from gap analysis)
- [SECURITY-HARDENING-RESPONSES.md](../SECURITY-HARDENING-RESPONSES.md) — Security hardening decisions (H9-H14, H17-H18)
- [EDGE-AGENT-SECURITY.md](../EDGE-AGENT-SECURITY.md) — Edge agent credential cache and security model
- [VAULT-ARCHITECTURE.md](../VAULT-ARCHITECTURE.md) — Credential vault encryption hierarchy
- [DEVICE-LOCKING.md](../DEVICE-LOCKING.md) — Advisory lock model
- [IDENTITY-MODEL.md](../IDENTITY-MODEL.md) — Identity resolution pipeline
- [ATTACK-SURFACE-NATS.md](ATTACK-SURFACE-NATS.md) — NATS threat model
- [ATTACK-SURFACE-POSTGRESQL.md](ATTACK-SURFACE-POSTGRESQL.md) — PostgreSQL threat model
- [BINARY-PROTECTION.md](../BINARY-PROTECTION.md) — Binary protection and license enforcement
