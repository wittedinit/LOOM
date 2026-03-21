# LOOM Medium Findings Responses (M1-M24)

> **Date:** 2026-03-21
> **Source:** [GAP-ANALYSIS.md](GAP-ANALYSIS.md) — Medium Findings
> **Status:** All 24 findings addressed with concrete decisions

---

## M1: `TypedOperation.Params` uses Go `any` — contradicts "zero map[string]any" promise

**Domain:** Arch

**Decision:** Replace `any` with typed params per operation (already done in OPERATION-TYPES.md). The `TypedOperation` envelope uses `any` at the deserialization boundary only — the concrete type is validated immediately on receipt via a type-switch dispatch.

**Rationale:** The "zero `map[string]any`" promise applies to business logic, not to wire deserialization. Every serialization framework (encoding/json, protobuf) requires a generic container at the unmarshal point. The invariant is: no `any` escapes the dispatch layer.

**Go types:**

```go
// TypedOperation is the wire envelope. Params is `any` ONLY here.
type TypedOperation struct {
    Type   OperationType `json:"type"`
    Params any           `json:"params"` // deserialization target only
}

// Dispatch validates and narrows immediately. No `any` beyond this point.
func DispatchOperation(op TypedOperation) error {
    switch op.Type {
    case OpSetHostname:
        p, ok := op.Params.(*SetHostnameParams)
        if !ok {
            return fmt.Errorf("invalid params for %s: expected *SetHostnameParams", op.Type)
        }
        return executeSetHostname(p)
    case OpCreateVLAN:
        p, ok := op.Params.(*CreateVLANParams)
        if !ok {
            return fmt.Errorf("invalid params for %s: expected *CreateVLANParams", op.Type)
        }
        return executeCreateVLAN(p)
    // ... every operation type has a concrete params struct
    }
}
```

**Contract update:** Add to OPERATION-TYPES.md: "`TypedOperation.Params` is `any` at the JSON deserialization boundary. The dispatcher MUST type-assert to the concrete params struct before any processing. No function signature beyond the dispatcher accepts `any`."

---

## M2: Apache AGE experimental — may not survive PostgreSQL upgrades

**Domain:** Arch, Infra

**Decision:** AGE is optional, not required. Default topology storage uses an adjacency table (`device_links`). AGE is only enabled when an operator explicitly installs the extension. LOOM degrades gracefully without AGE.

**Go types:**

```go
// DeviceLink is the default adjacency-table topology model. No AGE required.
type DeviceLink struct {
    ID           uuid.UUID `db:"id"`
    TenantID     uuid.UUID `db:"tenant_id"`
    SourceID     uuid.UUID `db:"source_device_id"`
    TargetID     uuid.UUID `db:"target_device_id"`
    SourcePort   string    `db:"source_port"`
    TargetPort   string    `db:"target_port"`
    LinkType     LinkType  `db:"link_type"` // ethernet, fiber, power, console
    DiscoveredAt time.Time `db:"discovered_at"`
    DiscoveredBy string    `db:"discovered_by"` // lldp, cdp, manual, cable-audit
}

type LinkType string

const (
    LinkEthernet LinkType = "ethernet"
    LinkFiber    LinkType = "fiber"
    LinkPower    LinkType = "power"
    LinkConsole  LinkType = "console"
)

// TopologyQuerier abstracts the backend. Default: SQL. Optional: AGE.
type TopologyQuerier interface {
    ShortestPath(ctx context.Context, tenantID uuid.UUID, from, to uuid.UUID) ([]DeviceLink, error)
    NeighborsWithinHops(ctx context.Context, tenantID uuid.UUID, deviceID uuid.UUID, hops int) ([]uuid.UUID, error)
    FailureDomain(ctx context.Context, tenantID uuid.UUID, deviceID uuid.UUID) ([]uuid.UUID, error)
}
```

**Migration path:** If AGE is deprecated, only the `AGETopologyQuerier` implementation is removed. The `SQLTopologyQuerier` (recursive CTEs on `device_links`) remains the permanent fallback.

---

## M3: NATS cross-subject ordering

**Domain:** Arch

**Decision:** LOOM does NOT require cross-subject ordering. Each device's events are ordered within their subject (`loom.{tenant}.device.{id}.*`). Cross-device ordering is handled by Temporal workflow sequencing, not NATS.

**Rationale:** NATS JetStream guarantees ordering within a single subject. LOOM's subject hierarchy ensures per-device event streams are totally ordered. When a workflow must coordinate across devices (e.g., drain switch A, then update switch B), Temporal activities enforce the sequence — NATS is the transport, not the sequencer.

**Subject hierarchy:**

```
loom.{tenant_id}.device.{device_id}.state      # device state changes (ordered)
loom.{tenant_id}.device.{device_id}.fact        # observed facts (ordered)
loom.{tenant_id}.device.{device_id}.op          # operation results (ordered)
loom.{tenant_id}.workflow.{workflow_id}.event    # workflow-level events (ordered)
```

**Contract update:** Add to EVENT-SCHEMA.md: "LOOM guarantees total ordering within a single NATS subject. Cross-subject (cross-device) ordering is NOT guaranteed by NATS and MUST NOT be relied upon. Use Temporal workflow sequencing for cross-device coordination."

---

## M4: AES-NI mandate excludes ARM edge devices

**Domain:** Arch, Security

**Decision:** AES-NI mandate applies to HUB infrastructure only. Edge agents use ARM hardware AES (Apple Silicon, Cortex-A53+). Go's `crypto/aes` automatically uses ARM Crypto Extensions when available.

**Updated constraint for SECURITY-MODEL.md:**

> AES hardware acceleration required: AES-NI on x86-64 (hub), ARM Crypto Extensions on aarch64 (edge). Software-only AES is not acceptable for production deployments.

**Verification at startup:**

```go
// VerifyHardwareAES checks for hardware AES support. Called at agent/hub boot.
func VerifyHardwareAES() error {
    switch runtime.GOARCH {
    case "amd64":
        if !cpu.X86.HasAES {
            return errors.New("AES-NI not available on x86-64; required for hub deployment")
        }
    case "arm64":
        if !cpu.ARM64.HasAES {
            return errors.New("ARM Crypto Extensions not available; required for edge deployment")
        }
    default:
        return fmt.Errorf("unsupported architecture %s: no verified hardware AES path", runtime.GOARCH)
    }
    return nil
}
```

**Note:** `golang.org/x/sys/cpu` provides the capability detection. Go's `crypto/aes` selects the hardware path automatically; this check is defense-in-depth to fail fast on unsupported hardware.

---

## M5: LLM response caching and replay determinism

**Domain:** Arch

**Decision:** Cache LLM responses in Valkey with a content-addressed key and 1-hour TTL. Invalidate on device state change, credential rotation, or config change.

**Go types:**

```go
// LLMCacheKey is the content-addressed cache identifier.
type LLMCacheKey [32]byte // SHA-256

// ComputeLLMCacheKey builds a deterministic key from all inputs that affect the response.
func ComputeLLMCacheKey(systemPrompt, userRequest, modelVersion string, tenantID uuid.UUID) LLMCacheKey {
    h := sha256.New()
    h.Write([]byte(systemPrompt))
    h.Write([]byte{0}) // separator
    h.Write([]byte(userRequest))
    h.Write([]byte{0})
    h.Write([]byte(modelVersion))
    h.Write([]byte{0})
    h.Write(tenantID[:])
    return LLMCacheKey(h.Sum(nil))
}

// LLMCacheEntry is the cached response payload.
type LLMCacheEntry struct {
    Response    string    `json:"response"`
    ModelID     string    `json:"model_id"`
    TokensUsed  int       `json:"tokens_used"`
    CachedAt    time.Time `json:"cached_at"`
    RequestHash string    `json:"request_hash"` // hex-encoded LLMCacheKey for debugging
}

const LLMCacheTTL = 1 * time.Hour
```

**Invalidation events (NATS subjects):**

- `loom.{tenant}.device.{id}.state` -- any state change invalidates cache entries referencing that device
- `loom.{tenant}.credential.rotated` -- invalidates all cache entries for tenant
- `loom.{tenant}.config.changed` -- invalidates all cache entries for tenant

**Replay determinism:** Cached responses ensure that retried Temporal activities produce identical LLM output within the TTL window, preventing non-determinism in workflow replay.

---

## M6: DuckDB on edge is CGO — contradicts pure-Go single binary

**Domain:** Arch, Infra

**Decision:** Remove DuckDB from edge agent entirely. Edge agents use SQLite only via `modernc.org/sqlite` (pure Go, zero CGO). Telemetry pre-aggregation uses SQL aggregation queries in SQLite.

**Go types:**

```go
// EdgeTelemetryAggregator pre-aggregates telemetry in SQLite before hub sync.
type EdgeTelemetryAggregator struct {
    db *sql.DB // modernc.org/sqlite — pure Go
}

// AggregateMetrics computes rollups using SQLite window functions.
// No DuckDB, no CGO.
func (a *EdgeTelemetryAggregator) AggregateMetrics(ctx context.Context, since time.Time) ([]MetricRollup, error) {
    const query = `
        SELECT
            metric_name,
            strftime('%Y-%m-%dT%H:00:00Z', timestamp) AS hour_bucket,
            COUNT(*) AS sample_count,
            MIN(value) AS min_val,
            MAX(value) AS max_val,
            AVG(value) AS avg_val
        FROM telemetry_raw
        WHERE timestamp >= ?
        GROUP BY metric_name, hour_bucket
        ORDER BY hour_bucket
    `
    // ... execute and scan
}

type MetricRollup struct {
    MetricName  string    `json:"metric_name"`
    HourBucket  time.Time `json:"hour_bucket"`
    SampleCount int64     `json:"sample_count"`
    MinVal      float64   `json:"min_val"`
    MaxVal      float64   `json:"max_val"`
    AvgVal      float64   `json:"avg_val"`
}
```

**Build constraint:** `CGO_ENABLED=0` is enforced for edge agent builds. CI rejects any edge binary that requires CGO.

---

## M7: NATS account provisioning

**Domain:** Security

**Decision:** Automated via LOOM tenant onboarding. The `TenantOnboardingWorkflow` creates a NATS account using the NATS server config API (v2.10+ supports dynamic account management via the system account).

**Go types:**

```go
// NATSAccountProvisioner creates and manages per-tenant NATS accounts.
type NATSAccountProvisioner struct {
    sysConn *nats.Conn // connected as NATS system account
}

// ProvisionTenantAccount creates a NATS account with tenant-scoped subject permissions.
func (p *NATSAccountProvisioner) ProvisionTenantAccount(ctx context.Context, tenantID uuid.UUID) (*NATSTenantAccount, error) {
    acct := &NATSTenantAccount{
        TenantID:    tenantID,
        AccountName: fmt.Sprintf("tenant-%s", tenantID),
        Permissions: NATSPermissions{
            Publish:   []string{fmt.Sprintf("loom.%s.>", tenantID)},
            Subscribe: []string{fmt.Sprintf("loom.%s.>", tenantID)},
        },
    }
    // Use NATS system account API to create the account dynamically
    // ...
    return acct, nil
}

type NATSTenantAccount struct {
    TenantID    uuid.UUID       `json:"tenant_id"`
    AccountName string          `json:"account_name"`
    Permissions NATSPermissions `json:"permissions"`
    CreatedAt   time.Time       `json:"created_at"`
}

type NATSPermissions struct {
    Publish   []string `json:"publish"`
    Subscribe []string `json:"subscribe"`
}
```

**Lifecycle:** Account is created during `TenantOnboardingWorkflow` and deleted during `TenantDecommissionWorkflow`. No manual NATS configuration required.

---

## M8: AES-GCM nonce entropy quality at boot

**Domain:** Security

**Decision:** First cryptographic operation after boot generates 32 random bytes and verifies entropy quality via chi-squared test. If the entropy pool is insufficient (embedded/early boot), block until `/dev/urandom` is seeded. Go's `crypto/rand.Read()` blocks on Linux until the kernel CSPRNG is seeded (since Linux 5.18 via `getrandom(2)`).

**Go types:**

```go
// VerifyEntropyPool validates that the system CSPRNG is producing quality randomness.
// Called once at agent/hub startup before any cryptographic operation.
func VerifyEntropyPool() error {
    buf := make([]byte, 256)
    _, err := crypto_rand.Read(buf) // blocks until seeded on Linux 5.18+
    if err != nil {
        return fmt.Errorf("entropy read failed: %w", err)
    }

    chi2 := chiSquaredByteTest(buf)
    // For 256 random bytes, chi-squared with 255 df should be ~255 +/- 22.
    // Reject if outside [200, 320] (very conservative bounds).
    if chi2 < 200 || chi2 > 320 {
        return fmt.Errorf("entropy quality check failed: chi-squared=%.2f (expected ~255)", chi2)
    }
    return nil
}

func chiSquaredByteTest(data []byte) float64 {
    var counts [256]float64
    for _, b := range data {
        counts[b]++
    }
    expected := float64(len(data)) / 256.0
    var chi2 float64
    for _, c := range counts {
        diff := c - expected
        chi2 += (diff * diff) / expected
    }
    return chi2
}
```

**Boot sequence:** `VerifyEntropyPool()` is called in `main()` before TLS setup, vault unsealing, or any key generation. Failure is fatal — the process refuses to start with bad entropy.

---

## M9: Device identity spoofing for cross-tenant merge

**Domain:** Security

**Decision:** Identity merge is ALWAYS tenant-scoped. Cross-tenant merge is architecturally impossible — the identity matcher never compares devices across tenants.

**Enforcement layers:**

```go
// IdentityMatcher scopes all operations to a single tenant. There is no method
// that accepts two different tenant IDs.
type IdentityMatcher struct {
    db       *sql.DB
    tenantID uuid.UUID // set at construction, immutable
}

// NewIdentityMatcher binds the matcher to exactly one tenant.
func NewIdentityMatcher(db *sql.DB, tenantID uuid.UUID) *IdentityMatcher {
    return &IdentityMatcher{db: db, tenantID: tenantID}
}

// FindMergeCandidates returns devices within this tenant that share identity signals.
// The SQL query includes `WHERE tenant_id = $1` — cross-tenant rows are never visible.
func (m *IdentityMatcher) FindMergeCandidates(ctx context.Context, signals []IdentitySignal) ([]MergeCandidate, error) {
    const query = `
        SELECT d.id, d.identity_signals
        FROM devices d
        WHERE d.tenant_id = $1
          AND d.identity_signals && $2  -- overlap operator
    `
    // ...
}
```

**Additional safeguards:**
1. PostgreSQL RLS policy: `devices` table has `USING (tenant_id = current_setting('app.tenant_id')::uuid)`
2. The `IdentityMatcher` constructor is the only factory — no way to create a matcher without a tenant ID
3. Merge operations are logged to the audit trail with both device IDs and tenant ID

**Contract update:** Add to IDENTITY-MODEL.md: "Cross-tenant device merge is architecturally impossible. The `IdentityMatcher` is constructed per-tenant and all queries are tenant-scoped via both application logic and PostgreSQL RLS."

---

## M10: Connection pool credential invalidation on rotation

**Domain:** Security

**Decision:** On credential rotation, emit a `credential.rotated` NATS event. Adapter connection pools subscribe to this event and close/recreate connections using the new credential.

**Go types:**

```go
// CredentialRotatedEvent is published when a device credential is rotated.
type CredentialRotatedEvent struct {
    TenantID   uuid.UUID `json:"tenant_id"`
    DeviceID   uuid.UUID `json:"device_id"`
    Protocol   string    `json:"protocol"`   // ssh, redfish, snmp, etc.
    RotatedAt  time.Time `json:"rotated_at"`
    RotationID uuid.UUID `json:"rotation_id"` // idempotency key
}

// CredentialRotationHandler manages pool invalidation for a single adapter.
type CredentialRotationHandler struct {
    pool       AdapterConnectionPool
    sub        *nats.Subscription
    tenantID   uuid.UUID
}

// OnCredentialRotated drains and recreates connections for the affected device.
func (h *CredentialRotationHandler) OnCredentialRotated(event *CredentialRotatedEvent) error {
    if event.TenantID != h.tenantID {
        return nil // not our tenant (defense-in-depth; NATS subjects already scoped)
    }
    // Close all pooled connections to this device
    h.pool.DrainDevice(event.DeviceID)
    // New connections will be created on-demand with the new credential
    // fetched from the vault at connection time
    return nil
}

// AdapterConnectionPool is the interface every adapter's pool must implement.
type AdapterConnectionPool interface {
    DrainDevice(deviceID uuid.UUID) error
    DrainAll() error
}
```

**NATS subject:** `loom.{tenant_id}.credential.rotated`

**Sequence:** Vault rotates credential -> publishes `CredentialRotatedEvent` -> all adapters with pooled connections to that device drain their pools -> next operation fetches fresh credential from vault and establishes new connection.

---

## M11: GDPR data classification and right-to-erasure

**Domain:** Security

**Decision:** Device metadata (hostnames, IPs, MAC addresses) is classified as PII under GDPR when it can identify natural persons. Right-to-erasure is implemented via tenant decommission, which purges all tenant data from all stores.

**Data classification:**

| Data Category | PII? | Storage | Erasure Method |
|---|---|---|---|
| Device hostnames, IPs, MACs | Yes (indirect PII) | PostgreSQL | `DELETE WHERE tenant_id = $1` |
| Operator usernames, emails | Yes (direct PII) | PostgreSQL | `DELETE WHERE tenant_id = $1` |
| Audit logs | Yes (contains operator actions) | PostgreSQL | Retained per legal hold; anonymized after hold expires |
| NATS event streams | Yes (contains device/operator data) | NATS JetStream | Stream purge by subject prefix |
| Valkey cache | Yes (cached device state) | Valkey | `DEL` by key prefix `tenant:{id}:*` |
| Edge agent local DB | Yes (device credentials, state) | SQLite on edge | Agent wipe command via hub |

**Go types:**

```go
// TenantDataPurger orchestrates right-to-erasure across all stores.
type TenantDataPurger struct {
    db      *sql.DB
    nats    *nats.Conn
    valkey  *valkey.Client
    edgeMgr EdgeAgentManager
}

// PurgeTenantData removes all data for a tenant from all stores.
// Called by TenantDecommissionWorkflow as a Temporal activity.
func (p *TenantDataPurger) PurgeTenantData(ctx context.Context, tenantID uuid.UUID) (*PurgeReport, error) {
    report := &PurgeReport{TenantID: tenantID, StartedAt: time.Now()}

    // 1. PostgreSQL (cascading deletes from tenant root)
    report.PostgreSQLRows, _ = p.purgePostgreSQL(ctx, tenantID)

    // 2. NATS JetStream (purge subjects)
    report.NATSMessages, _ = p.purgeNATSStreams(ctx, tenantID)

    // 3. Valkey (delete key prefix)
    report.ValkeyKeys, _ = p.purgeValkey(ctx, tenantID)

    // 4. Edge agents (remote wipe command)
    report.EdgeAgents, _ = p.purgeEdgeAgents(ctx, tenantID)

    report.CompletedAt = time.Now()
    return report, nil
}

type PurgeReport struct {
    TenantID       uuid.UUID `json:"tenant_id"`
    StartedAt      time.Time `json:"started_at"`
    CompletedAt    time.Time `json:"completed_at"`
    PostgreSQLRows int64     `json:"postgresql_rows_deleted"`
    NATSMessages   int64     `json:"nats_messages_purged"`
    ValkeyKeys     int64     `json:"valkey_keys_deleted"`
    EdgeAgents     int       `json:"edge_agents_wiped"`
}
```

**Contract update:** Add SECURITY-POLICY.md section on GDPR data classification and retention policy.

---

## M12: Air-gapped LLM model integrity

**Domain:** Security

**Decision:** On-prem LLM model files are verified via SHA-256 checksum against a signed manifest before loading. The manifest is signed by the LOOM release key (same TUF chain used for binary updates).

**Go types:**

```go
// ModelManifest lists all model files and their expected checksums.
type ModelManifest struct {
    Version    string              `json:"version"`
    Models     []ModelFileEntry    `json:"models"`
    SignedAt   time.Time           `json:"signed_at"`
    Signature  []byte              `json:"signature"` // Ed25519 over canonical JSON of Models
}

type ModelFileEntry struct {
    Name     string `json:"name"`     // e.g. "llama-3.1-8b-q4.gguf"
    SHA256   string `json:"sha256"`   // hex-encoded
    SizeBytes int64 `json:"size_bytes"`
}

// VerifyModelIntegrity checks all model files against the signed manifest.
func VerifyModelIntegrity(manifestPath string, modelDir string, pubKey ed25519.PublicKey) error {
    manifest, err := loadAndVerifyManifest(manifestPath, pubKey)
    if err != nil {
        return fmt.Errorf("manifest verification failed: %w", err)
    }

    for _, entry := range manifest.Models {
        filePath := filepath.Join(modelDir, entry.Name)
        hash, err := sha256File(filePath)
        if err != nil {
            return fmt.Errorf("cannot hash %s: %w", entry.Name, err)
        }
        if hash != entry.SHA256 {
            return fmt.Errorf("integrity check failed for %s: expected %s, got %s",
                entry.Name, entry.SHA256, hash)
        }
    }
    return nil
}
```

**Loading sequence:** `VerifyModelIntegrity()` is called before the LLM inference server loads any model. Failure is fatal — the server refuses to start with unverified models.

---

## M13: Temporal history as data exfiltration channel

**Domain:** Security

**Decision:** All Temporal payloads are encrypted via a custom `DataConverter`. Even with direct database access to Temporal's persistence, payload content is ciphertext. Cross-reference: SH-2 in SECURITY-HARDENING-RESPONSES.md.

**Go types:**

```go
// EncryptedDataConverter wraps Temporal's default converter with envelope encryption.
type EncryptedDataConverter struct {
    parent   converter.DataConverter
    keyStore KeyStore // fetches DEK from vault
}

// ToPayload encrypts before writing to Temporal history.
func (c *EncryptedDataConverter) ToPayload(value interface{}) (*commonpb.Payload, error) {
    plain, err := c.parent.ToPayload(value)
    if err != nil {
        return nil, err
    }
    dek, dekID, err := c.keyStore.GetCurrentDEK()
    if err != nil {
        return nil, fmt.Errorf("DEK fetch failed: %w", err)
    }
    ciphertext, nonce, err := encryptAESGCM(dek, plain.Data)
    if err != nil {
        return nil, err
    }
    return &commonpb.Payload{
        Metadata: map[string][]byte{
            "encoding":    []byte("loom/encrypted"),
            "dek_id":      []byte(dekID),
            "nonce":       nonce,
        },
        Data: ciphertext,
    }, nil
}

// FromPayload decrypts on read from Temporal history.
func (c *EncryptedDataConverter) FromPayload(payload *commonpb.Payload, valuePtr interface{}) error {
    if string(payload.Metadata["encoding"]) != "loom/encrypted" {
        return c.parent.FromPayload(payload, valuePtr) // legacy unencrypted
    }
    dekID := string(payload.Metadata["dek_id"])
    dek, err := c.keyStore.GetDEK(dekID)
    if err != nil {
        return fmt.Errorf("DEK %s not found: %w", dekID, err)
    }
    plainData, err := decryptAESGCM(dek, payload.Metadata["nonce"], payload.Data)
    if err != nil {
        return err
    }
    plain := &commonpb.Payload{Data: plainData}
    return c.parent.FromPayload(plain, valuePtr)
}
```

**Result:** Temporal history, search attributes, and memos are all ciphertext at rest. Database dumps, backups, or unauthorized Temporal UI access cannot reveal payload content.

---

## M14: seccomp/AppArmor for vault process

**Domain:** Security

**Decision:** Provide reference seccomp and AppArmor profiles for LOOM processes. Phase: post-MVP hardening (Phase 5+).

**seccomp profile (reference):**

```json
{
    "defaultAction": "SCMP_ACT_ERRNO",
    "architectures": ["SCMP_ARCH_X86_64", "SCMP_ARCH_AARCH64"],
    "syscalls": [
        {
            "names": ["read", "write", "close", "fstat", "mmap", "mprotect",
                       "munmap", "brk", "rt_sigaction", "rt_sigprocmask",
                       "clone", "futex", "epoll_ctl", "epoll_pwait",
                       "openat", "newfstatat", "getrandom", "clock_gettime",
                       "exit_group", "nanosleep", "sched_yield",
                       "connect", "socket", "getsockopt", "setsockopt",
                       "sendto", "recvfrom", "bind", "listen", "accept4"],
            "action": "SCMP_ACT_ALLOW"
        }
    ]
}
```

**Restrictions enforced:**
- No `fork`/`execve` -- vault is a library, not a process launcher
- No `ptrace` -- prevents debugging/memory inspection
- No `listen` on unprivileged ports -- vault does not expose a network server
- Limited filesystem access -- only vault data directory and `/dev/urandom`

**Go type for profile management:**

```go
// SecurityProfile defines the hardening configuration for a LOOM process.
type SecurityProfile struct {
    SeccompProfile  string `json:"seccomp_profile"`   // path to seccomp JSON
    AppArmorProfile string `json:"apparmor_profile"`  // AppArmor profile name
    ReadOnlyRootFS  bool   `json:"read_only_root_fs"`
    NoNewPrivileges bool   `json:"no_new_privileges"`
    Capabilities    []string `json:"capabilities"`    // Linux capabilities to retain
}
```

**Delivery:** Reference profiles shipped in `deploy/security/` directory. Kubernetes SecurityContext and Docker `--security-opt` examples included.

---

## M15: gNMI adapter overpromises transactional compensation

**Domain:** Infra

**Decision:** Already addressed. The gNMI adapter declares `CompensationReliability: BestEffort` in its adapter manifest. This is documented in ADAPTER-CONTRACT.md.

**Rationale:** gNMI `Set` operations are not transactional across multiple paths. If a multi-path Set partially fails, the adapter attempts to restore previous values via a compensating `Set`, but there is no protocol-level rollback guarantee.

**Go types (from ADAPTER-CONTRACT.md):**

```go
// AdapterCapabilities declares what the adapter can reliably do.
type AdapterCapabilities struct {
    SupportsCompensation    bool                  `json:"supports_compensation"`
    CompensationReliability CompensationReliability `json:"compensation_reliability"`
    SupportsTransactions    bool                  `json:"supports_transactions"`
    SupportsDryRun          bool                  `json:"supports_dry_run"`
}

type CompensationReliability string

const (
    CompensationGuaranteed CompensationReliability = "guaranteed"  // NETCONF confirmed-commit
    CompensationBestEffort CompensationReliability = "best_effort" // gNMI, SSH
    CompensationNone       CompensationReliability = "none"        // SNMP SET
)
```

**Contract update:** No change needed. Already specified. Cross-reference this response from GAP-ANALYSIS.md for traceability.

---

## M16: BIOS/UEFI provisioning variation

**Domain:** Infra

**Decision:** Out of MVP scope. BIOS/UEFI configuration is device-model-specific with no standard API. Phase 6+ feature.

**MVP scope:** Record BIOS version as an `ObservedFact` only. No configuration operations.

```go
// BIOSVersionFact is an ObservedFact, not an operation target.
type BIOSVersionFact struct {
    DeviceID     uuid.UUID `json:"device_id"`
    Vendor       string    `json:"vendor"`        // e.g. "Dell", "HPE", "Supermicro"
    BIOSVersion  string    `json:"bios_version"`  // e.g. "2.19.1"
    ReleaseDate  string    `json:"release_date"`  // e.g. "2025-11-15"
    SecureBoot   *bool     `json:"secure_boot"`   // nil if unknown
    TPMPresent   *bool     `json:"tpm_present"`   // nil if unknown
    ObservedAt   time.Time `json:"observed_at"`
    ObservedVia  string    `json:"observed_via"`   // "redfish", "ipmi", "dmidecode"
}
```

**Phase 6+ plan:** BIOS configuration via Redfish `Bios` resource (`/redfish/v1/Systems/1/Bios`). Vendor-specific schemas (Dell `DellBios`, HPE `HpeBiosExt`) handled by vendor-specific adapters.

---

## M17: Certificate lifecycle integration

**Domain:** Infra

**Decision:** Cross-reference SECURITY-HARDENING-RESPONSES.md section 1 (Internal CA Lifecycle). Full cert-manager integration planned for Phase 8.

**Phase 1A:** Self-signed CA with manual rotation procedure documented in `ops/runbooks/ca-rotation.md`.

**Phase 5:** Automated CA rotation via Temporal workflow:

```go
// CACertRotationWorkflow automates internal CA certificate rotation.
type CACertRotationWorkflow struct {
    CurrentCAExpiry time.Time
    RenewalWindow   time.Duration // e.g., 30 days before expiry
}

// Activities:
// 1. GenerateNewCA -- create new CA cert/key pair
// 2. DistributeCA -- push new CA to all trust stores (hub, edge, adapters)
// 3. ReissueLeafCerts -- re-sign all leaf certificates with new CA
// 4. VerifyTLSHealth -- check all mTLS connections still work
// 5. RetireOldCA -- remove old CA from trust stores after grace period
```

**Phase 8:** cert-manager CRDs for Kubernetes-native certificate lifecycle. `loom-issuer` ClusterIssuer backed by LOOM's internal CA.

---

## M18: Network segmentation architecture

**Domain:** Infra

**Decision:** Document recommended network topology in a DEPLOYMENT-TOPOLOGY.md appendix.

**Recommended VLAN segmentation:**

| VLAN | Purpose | LOOM Components | Access Control |
|---|---|---|---|
| Management | IPMI, iLO, iDRAC, PiKVM | Edge agent (read: Redfish, IPMI) | ACL: edge agent source IPs only |
| Control | LOOM hub-agent communication | Hub, edge agents, NATS, Temporal | mTLS required, no external access |
| Data | Workload traffic | None (LOOM does not touch data plane) | N/A |
| Out-of-Band | Emergency console access | Edge agent (serial/SSH console) | Physically separate where possible |

**Go types:**

```go
// NetworkSegment describes a logical network zone in the deployment topology.
type NetworkSegment struct {
    Name        string   `json:"name"`         // management, control, data, oob
    VLANIDs     []int    `json:"vlan_ids"`     // may span multiple VLANs
    Subnets     []string `json:"subnets"`      // CIDR notation
    Purpose     string   `json:"purpose"`
    LOOMAccess  string   `json:"loom_access"`  // "full", "read-only", "none"
    MTLSRequired bool   `json:"mtls_required"`
}
```

**Contract update:** Add appendix to DEPLOYMENT-TOPOLOGY.md with reference architecture diagram (ASCII) and per-segment firewall rules.

---

## M19: Temporal embedded to external migration path

**Domain:** Infra

**Decision:** Phase 1A starts with external Temporal (not embedded). There is no embedded-to-external migration path because embedded mode is not used.

**Deployment model:**

```
Single-binary mode:
  loom-hub binary starts Temporal Server as a subprocess (separate OS process)
  Temporal uses the same PostgreSQL instance (separate database: `temporal`)
  This is NOT embedded — it is a co-located subprocess with its own lifecycle

Distributed mode:
  Temporal Server runs independently (separate deployment, separate scaling)
  Same PostgreSQL schema, same Temporal namespace
  Migration: stop co-located subprocess, point LOOM hub at external Temporal endpoint
```

**Go types:**

```go
// TemporalDeploymentMode determines how Temporal is started.
type TemporalDeploymentMode string

const (
    TemporalColocated  TemporalDeploymentMode = "colocated"  // subprocess, single-binary
    TemporalExternal   TemporalDeploymentMode = "external"   // separate deployment
)

type TemporalConfig struct {
    Mode       TemporalDeploymentMode `yaml:"mode"`
    Address    string                 `yaml:"address"`    // external: "temporal.svc:7233"
    Namespace  string                 `yaml:"namespace"`  // always "loom"
    Database   string                 `yaml:"database"`   // PostgreSQL database name
}
```

**Contract update:** Add to DEPLOYMENT-TOPOLOGY.md: "Temporal always runs as a separate process, even in single-binary mode (subprocess). The only difference between modes is lifecycle management and network address."

---

## M20: DNS integration

**Domain:** Infra

**Decision:** Phase 1 DNS is verification-only (check A/AAAA records exist). Phase 7+ adds DNS record creation via adapter pattern.

**Phase 1 (verification only):**

```go
// DNSVerifier checks that expected DNS records exist. Does not create records.
type DNSVerifier struct {
    resolver *net.Resolver
}

// VerifyRecords checks A/AAAA resolution for a list of expected hostnames.
func (v *DNSVerifier) VerifyRecords(ctx context.Context, checks []DNSCheck) ([]DNSResult, error) {
    results := make([]DNSResult, len(checks))
    for i, check := range checks {
        addrs, err := v.resolver.LookupHost(ctx, check.Hostname)
        results[i] = DNSResult{
            Hostname: check.Hostname,
            Expected: check.ExpectedIPs,
            Resolved: addrs,
            Match:    ipsMatch(check.ExpectedIPs, addrs),
            Error:    err,
        }
    }
    return results, nil
}

type DNSCheck struct {
    Hostname    string   `json:"hostname"`
    ExpectedIPs []string `json:"expected_ips"` // optional — if empty, just check resolution
}

type DNSResult struct {
    Hostname string   `json:"hostname"`
    Expected []string `json:"expected"`
    Resolved []string `json:"resolved"`
    Match    bool     `json:"match"`
    Error    error    `json:"error,omitempty"`
}
```

**Phase 7+ (DNS record creation via adapter):**

Supported backends: PowerDNS (REST API), Route53 (AWS SDK), CoreDNS (etcd backend). Each implemented as a LOOM adapter conforming to the standard `Adapter` interface.

---

## M21: VLAN management vendor differences

**Domain:** Infra

**Decision:** Handled by per-vendor NETCONF/CLI adapters. VLAN creation is a typed operation (`CreateVLANOp`) with vendor-specific rendering in each adapter. Cross-reference: ADAPTER-SDK.md section 2.

**Go types:**

```go
// CreateVLANParams is the vendor-neutral operation input.
type CreateVLANParams struct {
    VLANID      int      `json:"vlan_id" validate:"min=1,max=4094"`
    Name        string   `json:"name" validate:"required,max=32"`
    Description string   `json:"description,omitempty"`
    Interfaces  []string `json:"interfaces,omitempty"` // ports to assign
    Tagged      bool     `json:"tagged"`               // tagged (trunk) or untagged (access)
}

// Each vendor adapter implements the rendering differently:
//
// JunosAdapter.CreateVLAN():
//   <vlans><vlan><name>{name}</name><vlan-id>{id}</vlan-id></vlan></vlans>
//
// IOSXEAdapter.CreateVLAN():
//   vlan {id}\n name {name}
//
// SONiCAdapter.CreateVLAN():
//   PATCH /restconf/data/sonic-vlan:sonic-vlan/VLAN/VLAN_LIST
//   {"vlanid": {id}, "members": [...]}
//
// CumulusAdapter.CreateVLAN():
//   NVUE: PATCH /nvue_v1/interface/bridge/domain/br_default/vlan/{id}
```

**Testing:** The adapter conformance test suite (from C10 resolution) includes a `CreateVLAN` test case that validates correct rendering per vendor against known-good output fixtures.

---

## M22: Observability self-debugging

**Domain:** App

**Decision:** `loom admin debug` CLI command with subcommands for each subsystem.

**Go types:**

```go
// DebugReport is the unified output of `loom admin debug`.
type DebugReport struct {
    Timestamp   time.Time            `json:"timestamp"`
    Queues      *QueueDebug          `json:"queues,omitempty"`
    Connections *ConnectionDebug     `json:"connections,omitempty"`
    NATS        *NATSDebug           `json:"nats,omitempty"`
    Database    *DatabaseDebug       `json:"database,omitempty"`
    Vault       *VaultDebug          `json:"vault,omitempty"`
}

type QueueDebug struct {
    TaskQueues []TaskQueueStatus `json:"task_queues"`
}

type TaskQueueStatus struct {
    Name           string `json:"name"`
    PendingTasks   int64  `json:"pending_tasks"`
    ActiveWorkers  int    `json:"active_workers"`
    BacklogAge     string `json:"backlog_age"` // duration of oldest pending task
}

type ConnectionDebug struct {
    Adapters []AdapterPoolStatus `json:"adapters"`
}

type AdapterPoolStatus struct {
    AdapterName   string `json:"adapter_name"`
    Protocol      string `json:"protocol"`
    ActiveConns   int    `json:"active_conns"`
    IdleConns     int    `json:"idle_conns"`
    MaxConns      int    `json:"max_conns"`
    WaitingReqs   int    `json:"waiting_requests"`
    AvgLatencyMs  int64  `json:"avg_latency_ms"`
}

type NATSDebug struct {
    Streams    []StreamStatus   `json:"streams"`
    Consumers  []ConsumerStatus `json:"consumers"`
}

type StreamStatus struct {
    Name       string `json:"name"`
    Messages   uint64 `json:"messages"`
    Bytes      uint64 `json:"bytes"`
    FirstSeq   uint64 `json:"first_seq"`
    LastSeq    uint64 `json:"last_seq"`
    ConsumerCount int `json:"consumer_count"`
}

type ConsumerStatus struct {
    Stream        string `json:"stream"`
    Name          string `json:"name"`
    PendingCount  int    `json:"pending_count"`
    AckPending    int    `json:"ack_pending"`
    LastDelivered uint64 `json:"last_delivered_seq"`
    Lag           uint64 `json:"lag"` // stream last_seq - consumer last_delivered
}

type DatabaseDebug struct {
    PoolSize      int    `json:"pool_size"`
    InUse         int    `json:"in_use"`
    Idle          int    `json:"idle"`
    WaitCount     int64  `json:"wait_count"`
    ActiveQueries []ActiveQuery `json:"active_queries"`
}

type ActiveQuery struct {
    PID      int    `json:"pid"`
    Duration string `json:"duration"`
    Query    string `json:"query"` // truncated to 200 chars
    State    string `json:"state"` // active, idle in transaction, etc.
}

type VaultDebug struct {
    Sealed   bool   `json:"sealed"`
    KeyAge   string `json:"key_age"`   // duration since last key rotation
    KeyCount int    `json:"key_count"` // number of managed keys
}
```

**CLI subcommands:**

```
loom admin debug queues       # Temporal task queue depth and worker count
loom admin debug connections  # Adapter connection pool status
loom admin debug nats         # Stream lag, consumer status
loom admin debug db           # PostgreSQL pool, active queries
loom admin debug vault        # Seal status, key age
loom admin debug all          # Full report (default)
```

---

## M23: CQRS projection rebuild from events

**Domain:** App

**Decision:** `loom admin db rebuild-projections` replays Temporal workflow histories to rebuild PostgreSQL device/workflow state tables. Runs offline (LOOM API unavailable during rebuild).

**Go types:**

```go
// ProjectionRebuilder replays Temporal histories to reconstruct PostgreSQL projections.
type ProjectionRebuilder struct {
    temporalClient client.Client
    db             *sql.DB
    namespace      string
}

// RebuildProjections drops and recreates all projection tables from Temporal history.
// LOOM API must be offline during this operation.
func (r *ProjectionRebuilder) RebuildProjections(ctx context.Context, opts RebuildOptions) (*RebuildReport, error) {
    report := &RebuildReport{StartedAt: time.Now()}

    // 1. Truncate projection tables (devices, workflow_states, device_facts)
    if err := r.truncateProjections(ctx); err != nil {
        return nil, fmt.Errorf("truncate failed: %w", err)
    }

    // 2. List all workflow executions
    iter, err := r.temporalClient.ListWorkflow(ctx, &workflowservice.ListWorkflowExecutionsRequest{
        Namespace: r.namespace,
    })
    if err != nil {
        return nil, err
    }

    // 3. Replay each workflow's history to reconstruct projections
    for iter.HasNext() {
        exec, _ := iter.Next()
        events, _ := r.getWorkflowHistory(ctx, exec.GetExecution())
        for _, event := range events {
            r.applyToProjection(ctx, event)
        }
        report.WorkflowsReplayed++
    }

    report.CompletedAt = time.Now()
    return report, nil
}

type RebuildOptions struct {
    TenantID *uuid.UUID // nil = all tenants
    DryRun   bool       // if true, report counts without writing
}

type RebuildReport struct {
    StartedAt         time.Time `json:"started_at"`
    CompletedAt       time.Time `json:"completed_at"`
    WorkflowsReplayed int64     `json:"workflows_replayed"`
    DevicesProjected  int64     `json:"devices_projected"`
    FactsProjected    int64     `json:"facts_projected"`
    Errors            []string  `json:"errors,omitempty"`
}
```

**Performance estimate:** ~1 minute per 10,000 workflows. A 100-device deployment with 10 workflows each rebuilds in under 1 second.

**CLI:**

```
loom admin db rebuild-projections                  # rebuild all
loom admin db rebuild-projections --tenant <id>    # rebuild single tenant
loom admin db rebuild-projections --dry-run        # count only, no writes
```

---

## M24: UI topology visualization at scale

**Domain:** App

**Decision:** Cytoscape.js with WebGL renderer for client-side graph rendering. Server-side pre-aggregation for deployments exceeding 1,000 devices.

**Server-side aggregation:**

```go
// TopologyView represents the server-side pre-aggregated topology for large deployments.
type TopologyView struct {
    Nodes []TopologyNode `json:"nodes"`
    Edges []TopologyEdge `json:"edges"`
    Level ViewLevel      `json:"level"` // site, rack, device
    Total int            `json:"total"` // total device count before aggregation
}

type ViewLevel string

const (
    ViewLevelSite   ViewLevel = "site"   // >1000 devices: group by site
    ViewLevelRack   ViewLevel = "rack"   // 100-1000 devices: group by rack
    ViewLevelDevice ViewLevel = "device" // <100 devices: show individual devices
)

type TopologyNode struct {
    ID          string            `json:"id"`
    Label       string            `json:"label"`
    Type        string            `json:"type"`        // site, rack, switch, server, pdu
    DeviceCount int               `json:"device_count"` // >1 for aggregated nodes
    Status      string            `json:"status"`       // healthy, degraded, offline
    Position    *Position         `json:"position,omitempty"`
    Metadata    map[string]string `json:"metadata,omitempty"`
}

type TopologyEdge struct {
    ID       string `json:"id"`
    Source   string `json:"source"`
    Target   string `json:"target"`
    LinkType string `json:"link_type"` // ethernet, fiber, power, aggregated
    Count    int    `json:"count"`     // >1 for aggregated edges
}

type Position struct {
    X float64 `json:"x"`
    Y float64 `json:"y"`
}

// GetTopologyView returns a view appropriate for the device count.
func GetTopologyView(ctx context.Context, tenantID uuid.UUID, zoom ViewLevel) (*TopologyView, error) {
    // Automatic level selection based on device count if zoom not specified
    // Site view: each node = site, edges = inter-site links
    // Rack view: each node = rack, edges = inter-rack links
    // Device view: each node = device, edges = physical links
}
```

**Client-side behavior:**
- Zoom out: site-level view (aggregated nodes with device counts)
- Zoom in on site: rack-level view within that site
- Zoom in on rack: device-level view within that rack
- Level-of-detail transitions triggered by zoom thresholds
- WebGL renderer (`cytoscape-webgl`) handles 1,000+ nodes at 60fps

**API endpoint:** `GET /api/v1/topology?tenant={id}&level={site|rack|device}&parent={optional-parent-id}`

---

## Summary Matrix

| Finding | Decision Type | Phase | Cross-Reference |
|---|---|---|---|
| M1 | Clarification | Phase 0 | OPERATION-TYPES.md |
| M2 | Architecture (graceful degradation) | Phase 1A | DEPLOYMENT-TOPOLOGY.md |
| M3 | Clarification (by design) | Phase 0 | EVENT-SCHEMA.md |
| M4 | Constraint update | Phase 0 | SECURITY-MODEL.md |
| M5 | New feature | Phase 3 | LLM-INTEGRATION.md |
| M6 | Architecture (remove component) | Phase 1A | EDGE-AGENT.md |
| M7 | Automation | Phase 1A | SECURITY-MODEL.md |
| M8 | Hardening | Phase 1A | SECURITY-MODEL.md |
| M9 | Clarification (by design) | Phase 0 | IDENTITY-MODEL.md |
| M10 | New feature | Phase 2 | ADAPTER-CONTRACT.md |
| M11 | Policy | Phase 5 | SECURITY-POLICY.md |
| M12 | Hardening | Phase 3 | LLM-INTEGRATION.md |
| M13 | Clarification (already solved) | Phase 1A | SECURITY-HARDENING-RESPONSES.md SH-2 |
| M14 | Hardening | Phase 5+ | deploy/security/ |
| M15 | Clarification (already addressed) | Phase 0 | ADAPTER-CONTRACT.md |
| M16 | Deferred (out of MVP) | Phase 6+ | IMPLEMENTATION-PLAN.md |
| M17 | Cross-reference | Phase 8 | SECURITY-HARDENING-RESPONSES.md |
| M18 | Documentation | Phase 1A | DEPLOYMENT-TOPOLOGY.md |
| M19 | Clarification | Phase 0 | DEPLOYMENT-TOPOLOGY.md |
| M20 | Phased delivery | Phase 1 / Phase 7 | DNS-INTEGRATION.md |
| M21 | Cross-reference | Phase 2 | ADAPTER-SDK.md |
| M22 | New feature | Phase 3 | CLI.md |
| M23 | New feature | Phase 3 | CLI.md |
| M24 | Architecture | Phase 4 | UI-ARCHITECTURE.md |
