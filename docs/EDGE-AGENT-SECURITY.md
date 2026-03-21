# Edge Agent Security Model

> **Status:** Decided
> **Addresses:** GAP-ANALYSIS.md findings C1, C4, C5; RESEARCH-edge-agent-security.md findings EAS-1 through EAS-10
> **Last updated:** 2026-03-21

---

## 1. Edge Credential Cache (C4, EAS-1, EAS-2, EAS-3)

### Decision

Credentials cached on edge agents are stored as **AES-256-GCM encrypted columns in the agent's local SQLite database**. Each credential value is encrypted with a per-credential Data Encryption Key (DEK), and each DEK is wrapped by the agent's Key Encryption Key (KEK).

### Key Derivation and Credential Caching Mode

<!-- AUDIT-FIX: C-02 — TPM is MANDATORY for credential caching; non-TPM agents use fetch-on-demand -->

The agent's credential handling mode depends on hardware capability:

- **TPM available (`CachedWithTPM` mode):** KEK is sealed to the device's TPM 2.0 module. The sealed key can only be unsealed on the specific physical device, binding credentials to hardware identity. **Only agents with TPM may cache credentials locally.** This is no longer optional — TPM is MANDATORY for any agent that caches credentials.
- **No TPM (`FetchOnDemand` mode):** Agents without TPM **do not cache credentials locally**. Instead, they retrieve credentials from the hub for each operation and discard them from memory immediately after use. Credentials are never written to the local SQLite database.

> **Design decision (revised):** The previous design allowed non-TPM agents to cache credentials using Argon2id-derived encryption. This was identified as a single point of compromise: if the enrollment secret is leaked, all cached credentials on every non-TPM agent are recoverable. The enrollment secret is a shared secret distributed at provisioning time and is not bound to hardware identity.
>
> **New rule:** TPM is MANDATORY for credential caching. Non-TPM agents operate in fetch-on-demand mode.

**Tradeoff:** Fetch-on-demand mode has higher latency per operation (one round-trip to hub per credential retrieval) and requires hub connectivity for any operation that needs credentials. Non-TPM agents **cannot perform credentialed operations while offline**. This is an acceptable tradeoff because the alternative (caching with a software-derived key) provides insufficient protection against enrollment secret compromise.

```go
// EdgeCredentialMode determines how the agent handles credentials.
type EdgeCredentialMode string

const (
    // CachedWithTPM — credentials are cached locally in encrypted SQLite,
    // with KEK sealed to TPM 2.0. Requires TPM hardware.
    CachedWithTPM EdgeCredentialMode = "cached_with_tpm"

    // FetchOnDemand — credentials are retrieved from the hub for each
    // operation. Never cached locally. Used when TPM is not available.
    FetchOnDemand EdgeCredentialMode = "fetch_on_demand"
)
```

<!-- END AUDIT-FIX: C-02 -->

### Go Types

```go
// EdgeCredentialCache manages encrypted credential storage on the edge agent.
type EdgeCredentialCache struct {
    DB         *sql.DB
    KeyProvider EdgeKeyProvider
    MaxOffline time.Duration // Default: 7 days. After expiry, cache is wiped.
}

// EdgeKeyProvider abstracts the KEK source. Implementations are selected
// at agent startup based on hardware detection.
type EdgeKeyProvider interface {
    // UnwrapDEK decrypts a per-credential DEK using the agent KEK.
    UnwrapDEK(wrappedDEK []byte) ([]byte, error)
    // WrapDEK encrypts a per-credential DEK using the agent KEK.
    WrapDEK(dek []byte) ([]byte, error)
    // Available reports whether this provider can operate on the current hardware.
    Available() bool
    // Destroy zeroes all key material held in memory.
    Destroy()
}

// TPMKeyProvider seals/unseals the agent KEK via TPM 2.0.
type TPMKeyProvider struct {
    DevicePath string // e.g., "/dev/tpmrm0"
    PCRSlots   []int  // PCR slots for sealing policy
}

// PassphraseKeyProvider derives the agent KEK from the enrollment secret via Argon2id.
type PassphraseKeyProvider struct {
    Salt       []byte // Unique per agent, generated at enrollment
    Time       uint32 // Argon2id time parameter (default: 3)
    Memory     uint32 // Argon2id memory in KiB (default: 65536)
    Threads    uint8  // Argon2id parallelism (default: 4)
}
```

### Credential Manifest

The hub tracks an `EdgeCredentialManifest` for every enrolled agent. This manifest records which credentials have been distributed, when, and a hash of each credential value.

```go
// EdgeCredentialManifest is maintained by the hub, one per edge agent.
type EdgeCredentialManifest struct {
    AgentID       string
    Entries       []ManifestEntry
    LastSyncedAt  time.Time
}

type ManifestEntry struct {
    CredentialRef string    // Reference to the vault credential
    DistributedAt time.Time // When the credential was sent to the agent
    ValueHash     []byte    // BLAKE3 hash of the credential value at distribution time
    ExpiresAt     time.Time // When the cached copy must be wiped
}
```

### Agent Decommission

When an agent is decommissioned:

1. Hub marks all credentials in that agent's manifest for rotation.
2. Hub revokes the agent's mTLS certificate (added to CRL).
3. Hub sends a credential-wipe command over NATS (best-effort if agent is reachable).
4. Agent, upon receiving the wipe command (or upon next hub contact), zeroes all DEKs and deletes all encrypted credential rows from local SQLite.
5. If the agent is unreachable, the hub proceeds with credential rotation for all affected devices regardless — the cached copies become useless once rotated at the source.

### Offline Expiry

The credential cache has a configurable maximum offline duration (default: 7 days). After this period without hub contact, the agent:

1. Wipes all cached credentials from SQLite.
2. Enters **degraded mode**: monitoring only, no mutations, no device authentication.
3. Continues attempting hub reconnection.

---

## 2. Auto-Update Security (C5, EAS-4, EAS-5, EAS-6)

### Decision

Edge agent binary distribution uses **TUF (The Update Framework)**, the CNCF-graduated standard for secure software updates. TUF addresses single-key compromise, replay attacks, rollback attacks, and stale metadata in a single framework.

### Key Hierarchy

| Role | Storage | Purpose | Rotation |
|------|---------|---------|----------|
| Root key | Offline HSM, air-gapped ceremony | Signs root metadata, delegates all other roles | Annual ceremony, cross-signed by outgoing root |
| Targets key | CI/CD pipeline HSM or KMS | Signs binary artifacts for day-to-day releases | Delegated via TUF, rotatable without root ceremony |
| Snapshot key | Automated, short-lived | Ensures metadata consistency (atomic view of repo) | Automated, rotated per-release |
| Timestamp key | Automated, short-lived (24h expiry) | Prevents stale metadata replay | Automated, rotated daily |

### Multi-Signature

Root metadata requires **2-of-3 signatures**. No single key compromise can result in fleet-wide code execution. The three root key holders are distinct individuals with distinct HSM tokens.

### Staged Rollout

Updates deploy in stages with health check gates between each:

| Stage | Scope | Gate to advance |
|-------|-------|-----------------|
| Canary | 1 agent (tagged at provisioning) | Agent reports healthy within 5 minutes |
| 10% | 10% of fleet (deterministic hash-based selection) | All agents healthy after configurable soak (default: 1 hour) |
| 50% | 50% of fleet | All agents healthy after configurable soak (default: 1 hour) |
| 100% | Remaining fleet | Operator approval or auto-promote after 24 hours |

### Health Check

After applying an update, each agent must report the following via NATS within 5 minutes:

- Binary version (must match expected)
- Hub connectivity (NATS heartbeat successful)
- At least one managed device reachable
- No crash loops detected (agent uptime > 2 minutes)

If the health report is not received within 5 minutes, the agent's systemd watchdog restarts with the previous binary (`/opt/loom/agent.prev`).

### Rollback

- **Automatic:** If health check fails post-update, the agent's systemd watchdog reverts to the previous binary. The hub records the failure and halts the rollout stage.
- **Manual:** Hub operator can issue a fleet-wide rollback command that instructs all agents to revert to a specified version.
- **TUF protection:** Snapshot metadata includes version numbers. Agents reject downgrades unless explicitly instructed by a rollback command signed with the targets key.

### Key Rotation

TUF root key rotation follows a ceremony process:

1. New root metadata is cross-signed by both the outgoing and incoming root keys.
2. Agents fetch new root metadata as part of their normal TUF metadata refresh.
3. Old root key is revoked in the new root metadata.
4. All subordinate keys (targets, snapshot, timestamp) can be rotated independently via TUF delegation without a root ceremony.

### Go Types

```go
// UpdatePolicy defines the rollout strategy for a release.
type UpdatePolicy struct {
    ReleaseID       string
    TargetVersion   string
    Stages          []RolloutStage
    RequireApproval bool          // If true, final stage requires operator approval
    SoakDuration    time.Duration // Health check soak period between stages
    AutoRollback    bool          // If true, failed health checks trigger automatic rollback
}

// RolloutStage represents one phase of a staged rollout.
type RolloutStage struct {
    Name          string  // e.g., "canary", "10pct", "50pct", "100pct"
    Percentage    float64 // Fraction of fleet (0.0-1.0)
    AgentSelector string  // Tag-based selector for canary, hash-based for percentage
    Status        string  // "pending", "in_progress", "healthy", "failed", "rolled_back"
    StartedAt     time.Time
    CompletedAt   time.Time
}

// AgentHealth is reported by each agent after an update.
type AgentHealth struct {
    AgentID         string
    BinaryVersion   string
    HubConnected    bool
    DevicesReachable int
    UptimeSeconds   int64
    NATSLatencyMs   int64
    ReportedAt      time.Time
}

// UpdateManifest is the TUF targets metadata for a single release.
type UpdateManifest struct {
    Version       string
    SHA256        string
    Length        int64
    Signatures    []TUFSignature
    RolloutPolicy UpdatePolicy
    PublishedAt   time.Time
}

type TUFSignature struct {
    KeyID     string
    Method    string // "ed25519"
    Signature []byte
}
```

### Phase Commitment

- **Phase 3:** Signed binary verification (Ed25519 multi-sig check before applying any update).
- **Phase 7 (post-MVP):** Full TUF repository integration with staged rollout, health gates, and automated key rotation.

---

## 3. Split-Brain Reconciliation (C1, EAS-7, EAS-8, EAS-9, EAS-10)

### Reconciliation Protocol

When connectivity is restored after a partition, the following protocol executes:

1. **Agent sends `ReconciliationRequest`** containing:
   - Local state hash (BLAKE3 hash of the agent's device state table)
   - Last-seen hub sequence number (the last NATS JetStream sequence the agent acknowledged)
   - Buffered event count by priority class
   - Clock status (see Section 3.4)

2. **Hub computes delta** between its own state and the agent-reported state hash. If hashes match (no divergence), reconciliation is skipped.

3. **Hub applies one of three reconciliation modes** (configurable per tenant):

| Mode | Behavior | Use case |
|------|----------|----------|
| `AutoReconcile` | Hub wins for configuration and desired state. Agent wins for observed state and facts. Conflicting mutations are resolved by timestamp with clock-skew correction. | Default for most tenants |
| `HumanApproval` | Conflicts are queued in a reconciliation inbox for operator review. Non-conflicting changes apply automatically. | High-security or compliance-sensitive tenants |
| `AgentAutonomous` | Agent decisions made during partition stand as-is. Hub records them but does not override. | Remote sites with trusted local operators |

```go
// ReconciliationRequest is sent by the agent when partition heals.
type ReconciliationRequest struct {
    AgentID            string
    LocalStateHash     []byte    // BLAKE3 hash of local device state
    LastHubSequence    uint64    // Last NATS JetStream sequence acknowledged
    BufferedEventCount map[Priority]int
    ClockStatus        ClockStatus
    PartitionStartedAt time.Time // When the agent first lost hub contact
}

// ReconciliationMode determines how conflicts are resolved.
type ReconciliationMode string

const (
    AutoReconcile   ReconciliationMode = "auto"
    HumanApproval   ReconciliationMode = "human"
    AgentAutonomous ReconciliationMode = "agent_autonomous"
)
```

### Priority-Based Offline Queue

The offline queue replaces the previous FIFO design with priority classes that prevent critical events from being evicted by telemetry volume.

| Priority | Class | Contents | Eviction | Storage |
|----------|-------|----------|----------|---------|
| P0 | Safety | Circuit breaker trips, emergency shutdown, discovery events, state transitions, errors | **Never evicted from memory.** Overflows to SQLite WAL if memory queue full. | Memory + SQLite spill |
| P1 | Operational | Scheduled maintenance results, credential rotation requests, configuration change results | Evicted only after all P2 events are gone. Max 1,000 items. | Memory |
| P2 | Informational | Telemetry, metrics, routine heartbeats, status updates | Evicted first (ring buffer). Oldest dropped when full. | Memory ring buffer |

**Disk spill:** P0 events that exceed the memory queue capacity are written to the agent's local SQLite database in WAL mode. These are replayed to the hub on reconnection before any other events.

```go
type Priority int

const (
    P0Safety        Priority = 0
    P1Operational   Priority = 1
    P2Informational Priority = 2
)

// OfflineEvent represents a queued event during partition.
type OfflineEvent struct {
    ID        string
    Priority  Priority
    Timestamp time.Time
    Payload   []byte
    Persisted bool // true if spilled to SQLite
}
```

### Autonomous Action Boundaries

The edge agent's offline authority is bounded by a per-tenant policy:

| Category | Actions | Authorization |
|----------|---------|---------------|
| **ALLOWED offline** | Discovery, monitoring, read-only state collection, emergency power-off (if safety trigger fires) | Always permitted, no hub contact needed |
| **REQUIRES HUB** | Configuration changes, provisioning, credential rotation, workflow execution | Queued until hub contact restored |
| **CONFIGURABLE** | Per-tenant policy sets the boundary. Some tenants may allow additional autonomy (e.g., restart-service, pre-approved workflow templates). | Defined at provisioning, stored in agent policy file |

The `AgentAutonomyPolicy` is set per tenant at provisioning time and can be updated by the hub:

```go
// AgentAutonomyPolicy defines what the agent may do offline.
type AgentAutonomyPolicy struct {
    TenantID            string
    Mode                string   // "observe_only", "pre_approved", "emergency", "full_autonomy"
    AllowedWorkflows    []string // Workflow template IDs permitted offline (for "pre_approved" mode)
    EmergencyThresholds map[string]float64 // e.g., {"health_pct": 10.0} — triggers emergency actions
}
```

### Clock Skew

Edge sites (especially Raspberry Pi or low-cost hardware without RTC batteries) can experience significant clock drift during extended partitions.

**Mitigation:**

- Every message from the agent includes a `ClockStatus` with the agent's local timestamp and NTP synchronization status.
- The hub **rejects messages with >30 seconds of clock skew** (configurable per tenant). Rejected messages are logged for investigation but not applied.
- The agent runs an NTP sync check every 5 minutes. If drift exceeds 5 seconds, the agent emits a P0 alert event.
- During reconciliation, the hub uses the agent's last-known-good NTP sync timestamp to estimate drift and adjust event ordering accordingly.

```go
// ClockStatus is included in every agent-to-hub message.
type ClockStatus struct {
    LocalTime     time.Time // Agent's local wall clock at message creation
    NTPSynced     bool      // Whether the agent's last NTP check succeeded
    DriftEstimate time.Duration // Agent's self-assessed drift from NTP (positive = ahead)
    LastNTPSync   time.Time // When the agent last successfully synced with NTP
}
```

---

## 4. Traceability Matrix

| Decision | Gap Analysis Finding | Research Finding | Section |
|----------|---------------------|------------------|---------|
| AES-256-GCM encrypted SQLite | C4 | EAS-1 (credential storage) | 1 |
| TPM-sealed or Argon2id-derived KEK | C4 | EAS-2 (hardware binding) | 1 |
| Credential manifest per agent | C4 | EAS-3 (revocation on decommission) | 1 |
| TUF for binary distribution | C5 | EAS-4 (multi-signature) | 2 |
| 2-of-3 root key threshold | C5 | EAS-5 (key compromise mitigation) | 2 |
| Staged rollout with health gates | C5 | EAS-6 (rollback and staged deployment) | 2 |
| Reconciliation protocol with three modes | C1 | EAS-7 (post-partition reconciliation) | 3 |
| Priority-based offline queue (P0/P1/P2) | C1 | EAS-8 (critical event eviction) | 3 |
| Autonomous action boundaries per tenant | C1 | EAS-9 (offline authority limits) | 3 |
| Clock skew detection and rejection | C1 | EAS-10 (timestamp drift) | 3 |

---

## 5. References

- [GAP-ANALYSIS.md](GAP-ANALYSIS.md) — Consolidated expert review (C1, C4, C5)
- [RESEARCH-edge-agent-security.md](RESEARCH-edge-agent-security.md) — Detailed research and threat models
- [The Update Framework (TUF)](https://theupdateframework.io/) — CNCF graduated project
- [Uptane](https://uptane.github.io/) — TUF for automotive OTA updates
- [Argon2](https://www.rfc-editor.org/rfc/rfc9106.html) — RFC 9106, password hashing
- DEPLOYMENT-TOPOLOGY.md Section 3: Edge Agent Architecture
- VAULT-ARCHITECTURE.md Section 5: Break Glass Emergency Access
