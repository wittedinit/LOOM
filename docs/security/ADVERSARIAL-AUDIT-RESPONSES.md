# Adversarial Security Audit Responses (H-01 through H-22)

> **Status:** Decided -- ready for implementation
> **Date:** 2026-03-21
> **Source:** Adversarial security audit -- 22 HIGH severity findings, 7 exploit chains
> **Scope:** All findings addressed with concrete fixes, Go types, and exploit chain breakdowns

---

## H-01: SecureCredential.Close() Race Condition

**Finding:** `SecureCredential.Close()` and the auto-wipe timer goroutine can race on the `Payload.data` slice. A consumer reading `Payload.Bytes()` while `Close()` or the timer fires reads partially-zeroed data or panics.

**Fix:** Add `sync.RWMutex` to `SecureCredential`. `Close()` acquires the write lock before zeroing. The timer goroutine acquires the write lock before auto-wipe. Consumers must hold the read lock while accessing `Payload.data`.

**Fixed type:**

```go
package vault

import (
    "sync"
    "time"
)

// SecureCredential is the return type from Vault.Retrieve(). Thread-safe.
//
// Callers MUST use this pattern:
//
//     cred, err := vault.Retrieve(ctx, tenantID, ref)
//     if err != nil { ... }
//     defer cred.Close()
//     cred.WithPayload(func(data []byte) {
//         // use data within this callback; do NOT retain the slice
//     })
type SecureCredential struct {
    Type     CredentialType
    Name     string
    Username string
    Payload  *SecureBytes
    Metadata map[string]string

    mu          sync.RWMutex  // protects Payload access and closed state
    maxLifetime time.Duration
    createdAt   time.Time
    timer       *time.Timer
    closed      bool
}

// WithPayload provides safe read access to the credential payload.
// The callback holds the read lock -- the payload cannot be wiped
// while the callback is executing. Callers MUST NOT retain the
// []byte slice beyond the callback scope.
func (sc *SecureCredential) WithPayload(fn func(data []byte)) error {
    sc.mu.RLock()
    defer sc.mu.RUnlock()
    if sc.closed {
        return fmt.Errorf("vault: credential already closed")
    }
    fn(sc.Payload.Bytes())
    return nil
}

// Close zeroes all sensitive bytes. Safe to call multiple times.
func (sc *SecureCredential) Close() error {
    sc.mu.Lock()
    defer sc.mu.Unlock()
    if sc.closed {
        return nil
    }
    if sc.timer != nil {
        sc.timer.Stop()
    }
    sc.Payload.Wipe()
    sc.closed = true
    return nil
}

// startAutoWipe is called by NewSecureCredential to schedule the TTL wipe.
// The timer goroutine acquires the write lock before wiping.
func (sc *SecureCredential) startAutoWipe() {
    sc.timer = time.AfterFunc(sc.maxLifetime, func() {
        sc.mu.Lock()
        defer sc.mu.Unlock()
        if !sc.closed {
            sc.Payload.Wipe()
            sc.closed = true
        }
    })
}
```

**Breaking change:** Direct `cred.Payload.Bytes()` access is replaced by `cred.WithPayload(func(data []byte) { ... })`. All call sites must migrate to the callback pattern.

**Cross-references:** VAULT-ARCHITECTURE.md Section 3 (SecureCredential), SECURITY-MODEL.md Section 4.

---

## H-02: AES-GCM Nonce Collision During Mass KEK Rotation

**Finding:** During bulk KEK rotation, re-encrypting >10K credentials under a single new KEK generates many nonces in a short window. The concern is birthday-bound collision on 96-bit random nonces.

**Analysis:**

The birthday bound for a 96-bit random nonce is approximately 2^48 (281 trillion). LOOM's worst-case nonce consumption:

```
100K credentials x 4 KEK rotations/year = 400K nonces/key/year
2^32 nonces ≈ 4.3 billion (NIST recommended limit per key)
400K nonces/year ÷ 4.3B limit = 0.009% of limit per year
Time to reach 2^32: 10,750 years
Time to reach birthday bound (2^48): 703 million years
```

**Decision:** Standard AES-256-GCM with 96-bit random nonces is safe for LOOM's scale. AES-256-GCM-SIV is not required.

**Mitigations implemented:**

1. **Document the math** in VAULT-ARCHITECTURE.md Section 2 (Algorithms).
2. **Monitoring alert:** Add Prometheus counter `loom_vault_encryptions_total{tenant_id, kek_version}`. Alert if any single KEK version exceeds 2^32 (4,294,967,296) encryptions -- this is unreachable under normal operation but provides defense-in-depth.
3. **Per-credential DEK architecture already limits exposure:** Each credential has its own DEK. During KEK rotation, DEKs are re-wrapped (not re-encrypted with new nonces for the credential data itself). The nonces consumed are for DEK wrapping only, not credential re-encryption.

**Metric:**

```
loom_vault_encryptions_total{tenant_id, kek_version}  # Counter
Alert: kek_encryption_count_high
  expr: loom_vault_encryptions_total > 4000000000
  severity: critical
```

**Cross-references:** VAULT-ARCHITECTURE.md Section 2 (Encryption Flow).

---

## H-03: Audit Hash Chain Bypassable by Superuser

**Finding:** A PostgreSQL superuser (or DBA with full table access) can rewrite the audit hash chain and recalculate hashes, leaving no evidence of tampering.

**Fix:** External anchoring is MANDATORY for production deployments. Optional anchoring is not acceptable -- a tampered audit trail is an existential threat.

**Anchoring strategy (all three required in production):**

1. **Daily Merkle root to separate PostgreSQL instance.** Different credentials, different network segment, different admin team. The anchor DB stores only Merkle roots (date, root_hash, record_count), not full audit data.

2. **Daily Merkle root to append-only cloud storage.** S3 with Object Lock (Governance or Compliance mode), Azure Immutable Blob Storage, or GCP Bucket Lock. Object Lock prevents deletion for the configured retention period.

3. **Weekly Merkle root to blockchain-style append-only log** (optional, for regulated environments). Timestamped witness cosignature via Sigstore Rekor or similar transparency log.

**Why both (a) and (b):** A DBA who compromises the primary PostgreSQL cannot also delete S3 Object Lock-protected objects (different credential domain). An AWS admin who compromises S3 cannot also rewrite the anchor PostgreSQL (different credential domain). Both must be compromised simultaneously to forge the audit trail.

```go
type AuditAnchor struct {
    Date          time.Time `json:"date"`
    MerkleRoot    []byte    `json:"merkle_root"`    // BLAKE3 Merkle root of day's records
    RecordCount   int64     `json:"record_count"`
    FirstRecordID string    `json:"first_record_id"`
    LastRecordID  string    `json:"last_record_id"`
    Signature     []byte    `json:"signature"`       // Ed25519 signature by anchor service
}

type AuditAnchorConfig struct {
    // ExternalDB is MANDATORY in production (Mode 2+).
    ExternalDB       *ExternalDBAnchor
    // CloudStorage is MANDATORY in production (Mode 2+).
    CloudStorage     *CloudStorageAnchor
    // TransparencyLog is optional, for regulated environments.
    TransparencyLog  *TransparencyLogAnchor
}

type ExternalDBAnchor struct {
    DSN             string        // separate PostgreSQL, different credentials
    PublishInterval time.Duration // 24 hours
}

type CloudStorageAnchor struct {
    Provider    string // "s3", "azure_blob", "gcs"
    Bucket      string
    ObjectLock  bool   // MUST be true for production
    RetentionDays int  // minimum 365
}
```

**Enforcement:** LOOM refuses to start in Mode 2+ (production) without at least `ExternalDB` AND `CloudStorage` anchor configuration. Mode 1 (development) logs a `WARN` if anchoring is not configured.

**Cross-references:** AUDIT-MODEL.md, SECURITY-MODEL.md Section 4.

---

## H-04: CRL 1-Hour Propagation Delay

**Finding:** The 1-hour CRL polling interval in SECURITY-HARDENING-RESPONSES.md Section 1 means a revoked certificate remains trusted for up to 60 minutes. An attacker with a stolen agent certificate has a 1-hour window.

**Fix:** Add NATS push notification for certificate revocation. The 1-hour poll becomes a backup mechanism, not the primary.

**Revocation flow:**

1. CA revokes certificate (added to CRL + OCSP updated).
2. Hub publishes `RevocationNotification` to NATS subject `loom.pki.revocation`.
3. All edge agents subscribe to `loom.pki.revocation` and update their local revocation cache immediately.
4. Agents that missed the push notification (offline, NATS partition) catch up on the next 1-hour CRL poll.

```go
// RevocationNotification is pushed via NATS to all agents immediately
// upon certificate revocation. Agents subscribe to "loom.pki.revocation".
type RevocationNotification struct {
    SerialNumber string    `json:"serial_number"` // hex-encoded certificate serial
    Reason       string    `json:"reason"`        // "key_compromise", "ca_compromise", "affiliation_changed",
                                                   // "superseded", "cessation_of_operation", "privilege_withdrawn"
    RevokedAt    time.Time `json:"revoked_at"`
    Issuer       string    `json:"issuer"`        // CA distinguished name
    NotifyID     string    `json:"notify_id"`     // UUID for deduplication
}
```

**Worst-case propagation:**

| Scenario | Propagation Time |
|----------|-----------------|
| Agent connected to NATS | < 1 second |
| Agent reconnects after short partition | Immediate on reconnection (NATS delivers buffered messages) |
| Agent offline > JetStream retention | Next CRL poll (1 hour max) |

**Cross-references:** SECURITY-HARDENING-RESPONSES.md Section 1, EDGE-AGENT-SECURITY.md.

---

## H-05: 7-Day Grace Period Weaponizable

**Finding:** SECURITY-HARDENING-RESPONSES.md Section 1 defines a 7-day grace period after certificate expiry. An attacker who compromises an expired certificate has 7 days of full operational access.

**Fix:** Reduce grace period to 24 hours. During the grace period, only read-only operations are permitted.

**Changes:**

| Parameter | Old Value | New Value |
|-----------|-----------|-----------|
| Grace period duration | 7 days | 24 hours |
| Operations during grace | Full access with warning | Read-only (no mutations) |
| Alert level | Warning | P0 alert (page on-call) |

**Behavior during 24-hour grace:**

1. Agent presents expired certificate (within 24h of expiry).
2. TLS handshake succeeds with `WARN` log.
3. Authorization middleware restricts to read-only operations: health checks, state reads, telemetry push.
4. All mutation requests (config push, power control, credential retrieval) are rejected with `CERTIFICATE_EXPIRED_GRACE`.
5. P0 alert fires: "Agent {agent_id} operating on expired certificate. Renewal required within {hours_remaining}h."

```go
type GracePeriodPolicy struct {
    Duration       time.Duration // 24 hours (was 7 days)
    AllowedActions []string      // ["health_check", "state_read", "telemetry_push"]
    DeniedActions  []string      // ["config_push", "power_control", "credential_retrieve", "workflow_execute"]
    AlertSeverity  string        // "P0"
}
```

**Cross-references:** SECURITY-HARDENING-RESPONSES.md Section 1, EDGE-AGENT-SECURITY.md.

---

## H-06: ConfigSnapshot Checksum as Oracle

**Finding:** `ConfigSnapshot` uses SHA-256 checksum for integrity. An attacker who knows the plaintext format can hash candidate configs and compare against the stored checksum to verify guesses about device configuration.

**Fix:** Replace SHA-256 checksum with HMAC-SHA256 keyed with the tenant's DEK. Without the key, an attacker cannot verify guesses.

```go
type ConfigSnapshot struct {
    DeviceID      uuid.UUID
    Timestamp     time.Time
    Format        string        // "cisco-ios", "junos-xml", "arista-json"
    EncryptedData EncryptedBlob // AES-256-GCM encrypted config (per H14 in SECURITY-HARDENING-RESPONSES.md)
    // Integrity was: Checksum []byte (SHA-256)
    // Now: HMAC-SHA256 keyed with tenant DEK. Attacker without key cannot verify guesses.
    HMAC          []byte        // HMAC-SHA256(tenant_dek, plaintext_config)
    HMACKeyRef    string        // reference to the DEK version used for HMAC
    SnapshotBy    string        // actor who captured this snapshot
}
```

**Migration:** Existing SHA-256 checksums are recomputed as HMAC-SHA256 during the next ConfigSnapshot encryption migration (SECURITY-HARDENING-RESPONSES.md Section 6). No separate migration needed.

**Cross-references:** SECURITY-HARDENING-RESPONSES.md Section 6 (ConfigSnapshot Encryption), VAULT-ARCHITECTURE.md.

---

## H-07: Identity Spoofing via Low-Confidence Fields

**Finding:** IDENTITY-MODEL.md auto-merge threshold of 0.70 with +0.15 corroboration bonus allows low-confidence fields (hostname at 0.40, FQDN at 0.40) to trigger auto-merge: 0.40 + 0.15 + 0.15 = 0.70. An attacker controlling DNS can spoof hostname/FQDN to merge attacker-controlled device with a legitimate device.

**Fix:**

1. Raise auto-merge threshold from 0.70 to 0.85.
2. Cap corroboration bonus at +0.10 (was +0.15).
3. Fields below 0.50 base confidence cannot trigger auto-merge regardless of corroboration.

**Impact on merge behavior:**

| Identity Type | Base Confidence | Max with Corroboration | Can Auto-Merge? |
|--------------|-----------------|------------------------|-----------------|
| Serial Number | 0.95 | 0.95 | Yes |
| BIOS UUID | 0.90 | 0.90 | Yes |
| MAC Address | 0.85 | 0.95 | Yes |
| BMC IP | 0.70 | 0.80 | No (< 0.85 threshold) |
| Asset Tag | 0.65 | 0.75 | No (< 0.85 threshold) |
| Management IP | 0.50 | 0.60 | No (< 0.85 threshold) |
| Hostname | 0.40 | N/A | No (< 0.50 floor) |
| FQDN | 0.40 | N/A | No (< 0.50 floor) |

```go
const (
    AutoMergeThreshold       = 0.85 // was 0.70
    CorroborationBonusCap    = 0.10 // was 0.15
    MinConfidenceForAutoMerge = 0.50 // fields below this NEVER trigger auto-merge
)
```

**Result:** Only serial number, BIOS UUID, and MAC address can trigger auto-merge. All other identity types require human review. This eliminates the DNS-spoofing attack vector.

**Cross-references:** IDENTITY-MODEL.md Sections 2 and 3.

---

## H-08: Approval Gate Bypass

**Finding:** `ApprovalDecision` contains `DecidedBy ActorRef` but the workflow trusts the signal content without server-side validation. An attacker who can send Temporal signals can craft an `ApprovalDecision` with a spoofed `DecidedBy` and bypass the approval gate.

**Fix:** `ApprovalDecision` MUST include an `ApproverID` that is validated server-side against the auth service. The workflow does not trust signal content for identity -- it verifies independently.

```go
// ApprovalDecision is the signal sent back to the workflow.
// The workflow MUST validate ApproverID via the auth service before proceeding.
type ApprovalDecision struct {
    Approved    bool      `json:"approved"`
    ApproverID  string    `json:"approver_id"`    // JWT subject of the approver
    Reason      string    `json:"reason"`
    DecidedAt   time.Time `json:"decided_at"`
    // Signature is an HMAC of (workflow_id + approved + approver_id + decided_at)
    // using a shared secret between the approval service and the workflow worker.
    // Prevents signal forgery even if an attacker can send Temporal signals.
    Signature   []byte    `json:"signature"`
}
```

**Validation sequence (inside workflow):**

1. Receive `ApprovalDecision` signal.
2. Verify `Signature` using the shared HMAC key (rejects forged signals).
3. Call auth service: `ValidateApprover(ctx, ApproverID, TenantID, OperationType)`.
4. Auth service confirms: ApproverID has `ApproverRole` for this tenant AND this operation type.
5. If validation fails, log security event, reject the decision, and re-emit the approval request.

**Cross-references:** WORKFLOW-CONTRACT.md (ApprovalDecision type), ATTACK-SURFACE-TEMPORAL.md Section 2.

---

## H-09: Adapter Registration Without Signing

**Finding:** `Registry.Register()` accepts any `AdapterRegistration` without verifying the adapter binary's integrity. A compromised or malicious adapter can register and receive credentials for managed devices.

**Fix:** All adapter binaries MUST be cosign-verified before `Registry.Register()` accepts them. An allowlist of permitted adapter names restricts which adapters can register.

```go
// SecureRegistry wraps the adapter Registry with signature verification.
type SecureRegistry struct {
    inner            Registry
    allowedAdapters  map[string]bool   // e.g., {"redfish": true, "ssh": true, "snmp": true, "ipmi": true}
    cosignVerifier   CosignVerifier
}

// Register verifies the adapter binary signature and checks the allowlist
// before delegating to the inner registry.
func (sr *SecureRegistry) Register(reg AdapterRegistration, binaryPath string) error {
    // 1. Check allowlist
    if !sr.allowedAdapters[reg.Name] {
        emitSecurityAlert("adapter_registration_rejected_unknown", reg.Name)
        return fmt.Errorf("adapter %q not in allowlist", reg.Name)
    }

    // 2. Verify cosign signature on the adapter binary
    if err := sr.cosignVerifier.Verify(binaryPath); err != nil {
        emitSecurityAlert("adapter_registration_rejected_unsigned", reg.Name)
        return fmt.Errorf("adapter %q signature verification failed: %w", reg.Name, err)
    }

    // 3. Register
    return sr.inner.Register(reg)
}

type CosignVerifier interface {
    // Verify checks that the binary at path has a valid cosign signature
    // matching the expected identity and OIDC issuer.
    Verify(binaryPath string) error
}
```

**Configuration:**

```yaml
adapters:
  allowlist:
    - redfish
    - ssh
    - snmp
    - ipmi
    - gnmi
    - netconf
    - pikvm
  cosign:
    certificate_identity: "https://github.com/wittedinit/LOOM/.github/workflows/release.yml@refs/tags/*"
    certificate_oidc_issuer: "https://token.actions.githubusercontent.com"
```

**Cross-references:** ADAPTER-CONTRACT.md (Registry), CI-CD-PIPELINE.md (cosign signing).

---

## H-10: Lock Extend with No Maximum

**Finding:** DEVICE-LOCKING.md allows lock extensions with no absolute maximum. A buggy or malicious workflow can hold a device lock indefinitely by repeatedly extending.

**Fix:** Maximum lock duration = 30 minutes absolute (regardless of extensions). Maximum 3 extensions per lock. After the absolute maximum, the lock auto-releases.

```go
const (
    MaxLockDuration   = 30 * time.Minute // absolute maximum, regardless of extensions
    MaxLockExtensions = 3                 // maximum number of extend calls per lock
)

type LockResult struct {
    Acquired       bool          `json:"acquired"`
    LockID         string        `json:"lock_id"`
    ExpiresAt      time.Time     `json:"expires_at"`
    AbsoluteExpiry time.Time     `json:"absolute_expiry"` // NEW: hard ceiling
    ExtensionsUsed int           `json:"extensions_used"` // NEW: tracks extensions
    HeldBy         *LockHolder   `json:"held_by"`
    RetryAfter     time.Duration `json:"retry_after"`
}

// ExtendLock extends a lock's expiry. Fails if the absolute maximum
// would be exceeded or if the extension count is exhausted.
func (lm *LockManager) ExtendLock(ctx context.Context, lockID string, extension time.Duration) (*LockResult, error) {
    lock := lm.getLock(lockID)
    if lock.ExtensionsUsed >= MaxLockExtensions {
        return nil, fmt.Errorf("lock %s: maximum extensions (%d) exhausted", lockID, MaxLockExtensions)
    }
    newExpiry := time.Now().Add(extension)
    if newExpiry.After(lock.AbsoluteExpiry) {
        return nil, fmt.Errorf("lock %s: extension would exceed absolute maximum (%v)", lockID, MaxLockDuration)
    }
    lock.ExpiresAt = newExpiry
    lock.ExtensionsUsed++
    return lock, nil
}
```

**Workflows that need longer:** Must release the lock and re-acquire. The re-acquisition is visible in audit and allows other waiting workflows to proceed.

**Cross-references:** DEVICE-LOCKING.md.

---

## H-11: DeviceStatus Lacks State Machine

**Finding:** `DeviceStatus` in DOMAIN-MODEL.md is a string enum with no transition validation. Any status can transition to any other status, including invalid transitions like `decommissioned -> active` (resurrecting a wiped device).

**Fix:** Add explicit state machine. `Decommissioned` is terminal. Credentials are purged on decommission.

**Valid transitions:**

```
discovered ──► active
active ──► maintenance
maintenance ──► active
active ──► decommissioned
maintenance ──► decommissioned
```

```go
// DeviceStatusTransitions defines the valid state machine.
// Any transition not in this map is rejected.
var DeviceStatusTransitions = map[DeviceStatus][]DeviceStatus{
    DeviceStatusDiscovered:     {DeviceStatusActive},
    DeviceStatusActive:         {DeviceStatusMaintenance, DeviceStatusDecommissioned},
    DeviceStatusMaintenance:    {DeviceStatusActive, DeviceStatusDecommissioned},
    DeviceStatusDecommissioned: {}, // terminal -- no transitions out
}

// ValidateTransition checks whether a status transition is permitted.
func ValidateTransition(from, to DeviceStatus) error {
    allowed, exists := DeviceStatusTransitions[from]
    if !exists {
        return fmt.Errorf("unknown device status: %s", from)
    }
    for _, target := range allowed {
        if target == to {
            return nil
        }
    }
    return fmt.Errorf("invalid device status transition: %s -> %s", from, to)
}
```

**Decommission side effects:**

1. All credentials for the device are revoked in the vault (DEKs destroyed).
2. All active locks on the device are released.
3. All running workflows targeting the device are cancelled.
4. Edge agent credential caches for the device are wiped (via NATS command).
5. Device is retained in the database (for audit history) but marked terminal.

**Note:** `DeviceStatusDegraded` and `DeviceStatusUnreachable` from the original DOMAIN-MODEL.md are reclassified as operational signals (endpoint-level), not device lifecycle states. A device with an unreachable endpoint remains `active` -- the endpoint status tracks reachability.

**Cross-references:** DOMAIN-MODEL.md (DeviceStatus enum).

---

## H-12: DiscoveryScope.Filter Injection

**Finding:** `DiscoveryScope` accepts an untyped string filter that is passed through to protocol adapters. An attacker who controls the filter string can inject SNMP OIDs, NETCONF XPath expressions, or gNMI paths that exfiltrate data beyond the intended discovery scope.

**Fix:** Replace untyped string with a typed `DiscoveryFilter` struct. No raw string passthrough.

```go
// DiscoveryFilter replaces the untyped string filter in DiscoveryScope.
// All fields are validated before being passed to protocol adapters.
type DiscoveryFilter struct {
    // SubnetCIDR restricts discovery to a specific subnet.
    // Validated as a well-formed CIDR (e.g., "10.0.1.0/24").
    SubnetCIDR net.IPNet `json:"subnet_cidr"`

    // DeviceTypes restricts which device types to discover.
    // Uses the DeviceType enum -- no arbitrary strings.
    DeviceTypes []DeviceType `json:"device_types,omitempty"`

    // Protocols restricts which protocols to probe.
    // Validated against ProbeMethodOrder -- only known protocols accepted.
    Protocols []ProbeMethod `json:"protocols,omitempty"`

    // PortOverrides allows non-standard ports per protocol.
    // Keys must be valid ProbeMethod values. Values must be 1-65535.
    PortOverrides map[ProbeMethod]int `json:"port_overrides,omitempty"`

    // ExcludeAddresses lists specific IPs to skip during subnet scan.
    ExcludeAddresses []net.IP `json:"exclude_addresses,omitempty"`
}

// Validate ensures all filter fields contain safe, expected values.
func (f *DiscoveryFilter) Validate() error {
    for _, dt := range f.DeviceTypes {
        if !dt.IsValid() {
            return fmt.Errorf("invalid device type: %s", dt)
        }
    }
    for _, p := range f.Protocols {
        if !p.IsValid() {
            return fmt.Errorf("invalid protocol: %s", p)
        }
    }
    for proto, port := range f.PortOverrides {
        if port < 1 || port > 65535 {
            return fmt.Errorf("invalid port %d for protocol %s", port, proto)
        }
    }
    return nil
}
```

**Cross-references:** DISCOVERY-CONTRACT.md, ADAPTER-CONTRACT.md (Discoverer interface).

---

## H-13: Mode 1 sslmode=disable

**Finding:** DEPLOYMENT-TOPOLOGY.md Mode 1 example uses `sslmode=disable` in the PostgreSQL connection string. This encourages copy-paste into production.

**Fix:** Remove `sslmode=disable` from ALL examples. Default to `sslmode=verify-full`. Mode 1 (development) auto-generates a self-signed TLS certificate on first run.

**Changed examples in DEPLOYMENT-TOPOLOGY.md:**

```bash
# Mode 1 (development) -- was: sslmode=disable
# Now: self-signed cert auto-generated on first run
./loom serve \
  --db "postgres://loom:secret@localhost:5432/loom?sslmode=verify-full&sslrootcert=.loom/dev-ca.crt" \
  --nats-embedded \
  --temporal-embedded
```

**First-run behavior:**

1. LOOM detects no TLS certificates in `.loom/` directory.
2. Generates a self-signed CA + server certificate for PostgreSQL.
3. Configures PostgreSQL to require TLS (if LOOM manages the dev PostgreSQL instance).
4. Writes `dev-ca.crt` and `dev-server.key` to `.loom/`.
5. Logs: `INFO: Generated self-signed TLS certificates for development. See .loom/dev-ca.crt`

**Enforcement:** LOOM refuses to connect to PostgreSQL with `sslmode=disable` or `sslmode=prefer`. Minimum accepted: `sslmode=require`. Recommended: `sslmode=verify-full`.

**Cross-references:** DEPLOYMENT-TOPOLOGY.md Mode 1, SECURITY-MODEL.md Section 2.

---

## H-14: Edge Auto-Update Without Signature Verification

**Finding:** Edge agent auto-update downloads and applies new binaries without verifying signatures before binary replacement. A MITM or compromised update server can push malicious binaries.

**Fix:** Mandatory signature verification before binary replacement. Agent refuses unsigned binaries.

**Update sequence (hardened):**

1. Agent fetches TUF metadata from hub (timestamp -> snapshot -> targets).
2. Agent downloads new binary to a staging directory (not the running binary path).
3. Agent verifies Ed25519 signature of the binary against the TUF targets key.
4. Agent verifies SHA-256 hash matches the TUF targets metadata.
5. **Only if both checks pass:** Agent replaces the running binary and restarts.
6. If either check fails: Agent logs a security alert, discards the download, and continues running the current version.

```go
// VerifyAndApplyUpdate is the mandatory pre-update verification.
// The binary is NEVER replaced without passing all checks.
func (agent *EdgeAgent) VerifyAndApplyUpdate(update UpdateManifest, binaryPath string) error {
    // 1. Verify TUF metadata chain (timestamp -> snapshot -> targets)
    if err := agent.tufClient.VerifyMetadataChain(); err != nil {
        agent.emitSecurityAlert("update_metadata_invalid", err)
        return fmt.Errorf("TUF metadata verification failed: %w", err)
    }

    // 2. Verify binary hash matches TUF targets
    hash := sha256.Sum256(readFile(binaryPath))
    if hex.EncodeToString(hash[:]) != update.SHA256 {
        agent.emitSecurityAlert("update_hash_mismatch", nil)
        return fmt.Errorf("binary hash mismatch: expected %s", update.SHA256)
    }

    // 3. Verify Ed25519 signature
    for _, sig := range update.Signatures {
        if !ed25519.Verify(agent.trustedKey, hash[:], sig.Signature) {
            agent.emitSecurityAlert("update_signature_invalid", nil)
            return fmt.Errorf("binary signature verification failed")
        }
    }

    // 4. All checks passed -- apply update
    return agent.replaceBinary(binaryPath)
}
```

**Cross-references:** EDGE-AGENT-SECURITY.md Section 2 (TUF integration), BINARY-PROTECTION.md.

---

## H-15: No Security-Specific Metrics

**Finding:** OBSERVABILITY-STRATEGY.md defines operational metrics but lacks security-specific metrics. Auth failures, authz denials, credential access, tenant boundary violations, and adapter anomalies have no observability.

**Fix:** Add the following security metrics to OBSERVABILITY-STRATEGY.md:

| Metric | Type | Labels | Description |
|--------|------|--------|-------------|
| `loom_auth_failures_total` | Counter | `source`, `reason` | Authentication failures (invalid JWT, expired token, bad password) |
| `loom_authz_denials_total` | Counter | `tenant`, `resource`, `action` | Authorization denials (user lacks permission) |
| `loom_credential_access_total` | Counter | `tenant`, `path`, `actor` | Every credential retrieval from the vault |
| `loom_tenant_boundary_violations_total` | Counter | `source_tenant`, `target_tenant` | Attempts to access another tenant's resources |
| `loom_adapter_anomaly_score` | Gauge | `adapter`, `device` | Statistical anomaly score for adapter behavior (response time, error rate, data volume) |
| `loom_certificate_expiry_seconds` | Gauge | `component`, `serial_number` | Time until certificate expiry (for proactive alerting) |
| `loom_vault_encryptions_total` | Counter | `tenant_id`, `kek_version` | Encryption operations per KEK (for H-02 nonce monitoring) |

**Alert rules:**

```yaml
groups:
  - name: loom_security
    rules:
      - alert: HighAuthFailureRate
        expr: rate(loom_auth_failures_total[5m]) > 10
        for: 2m
        labels:
          severity: warning
      - alert: TenantBoundaryViolation
        expr: increase(loom_tenant_boundary_violations_total[1h]) > 0
        labels:
          severity: critical
      - alert: CertificateExpiringSoon
        expr: loom_certificate_expiry_seconds < 86400
        labels:
          severity: P0
      - alert: AnomalousAdapterBehavior
        expr: loom_adapter_anomaly_score > 3.0
        for: 5m
        labels:
          severity: warning
```

**Cross-references:** OBSERVABILITY-STRATEGY.md Section 1.

---

## H-16: Trace Propagation Leaks to LLM Provider

**Finding:** OpenTelemetry trace context (`traceparent` header) is propagated to outbound LLM API calls. This leaks internal trace IDs, and LLM provider-side logging may capture `tenant_id`, `workflow_id`, or `device_id` from trace span attributes.

**Fix:** Strip `traceparent` headers from outbound LLM API calls. Create a new, isolated trace span at the LLM boundary.

```go
// LLMTraceBoundary creates a sanitized span for LLM API calls.
// Only non-sensitive attributes are propagated.
func LLMTraceBoundary(ctx context.Context, requestType string, model string) (context.Context, trace.Span) {
    // 1. Strip the incoming trace context -- do NOT propagate traceparent
    cleanCtx := trace.ContextWithSpan(context.Background(), trace.SpanFromContext(ctx))

    // 2. Start a new span linked (not child-of) to the original
    tracer := otel.Tracer("loom.llm.boundary")
    newCtx, span := tracer.Start(cleanCtx, "llm.request",
        trace.WithAttributes(
            attribute.String("request_type", requestType), // "classification", "placement", "config_suggestion"
            attribute.String("model", model),               // "claude-3-5-sonnet", "llama-3", etc.
            // NO tenant_id, workflow_id, device_id, correlation_id
        ),
    )

    return newCtx, span
}

// StripTraceHeaders removes traceparent and tracestate from outbound HTTP headers.
func StripTraceHeaders(headers http.Header) {
    headers.Del("traceparent")
    headers.Del("tracestate")
    headers.Del("baggage")
}
```

**Cross-references:** OBSERVABILITY-STRATEGY.md Section 2 (Distributed Tracing), LLM-BOUNDARIES.md, ATTACK-SURFACE-LLM.md.

---

## H-17: Security Events Dropped During NATS Outage

**Finding:** During a NATS outage, all events (including security-critical ones like auth failures and credential access) are buffered in the same queue. Security events may be dropped if the buffer fills with operational events.

**Fix:** Separate high-priority ring buffer for security events with 10x capacity. Security events are flushed first on reconnection.

```go
// PriorityEventBuffer separates security events from operational events.
// Security events have 10x buffer capacity and are flushed first on reconnection.
type PriorityEventBuffer struct {
    // SecuritySlots holds auth failures, credential access, tenant violations,
    // certificate revocations, and adapter anomaly events.
    SecuritySlots *RingBuffer // capacity: 100,000 events (10x operational)

    // OperationalSlots holds device state changes, workflow events,
    // discovery results, and other non-security events.
    OperationalSlots *RingBuffer // capacity: 10,000 events

    mu sync.Mutex
}

// Publish routes events to the appropriate buffer based on event domain.
func (peb *PriorityEventBuffer) Publish(event Event) error {
    peb.mu.Lock()
    defer peb.mu.Unlock()

    if isSecurityEvent(event) {
        return peb.SecuritySlots.Push(event)
    }
    return peb.OperationalSlots.Push(event)
}

// FlushOnReconnect drains buffers to NATS. Security events first.
func (peb *PriorityEventBuffer) FlushOnReconnect(conn *nats.Conn) error {
    peb.mu.Lock()
    defer peb.mu.Unlock()

    // Security events flushed FIRST -- higher priority
    for event, ok := peb.SecuritySlots.Pop(); ok; event, ok = peb.SecuritySlots.Pop() {
        if err := conn.Publish(event.Subject(), event.Data()); err != nil {
            peb.SecuritySlots.Push(event) // re-queue on failure
            return err
        }
    }

    // Then operational events
    for event, ok := peb.OperationalSlots.Pop(); ok; event, ok = peb.OperationalSlots.Pop() {
        if err := conn.Publish(event.Subject(), event.Data()); err != nil {
            peb.OperationalSlots.Push(event)
            return err
        }
    }
    return nil
}

// isSecurityEvent returns true for events that must not be dropped.
func isSecurityEvent(e Event) bool {
    switch {
    case strings.HasPrefix(e.Action(), "auth."):
        return true
    case strings.HasPrefix(e.Action(), "credential."):
        return true
    case e.Action() == "tenant.boundary_violation":
        return true
    case e.Action() == "certificate.revoked":
        return true
    case e.Action() == "adapter.anomaly_detected":
        return true
    default:
        return false
    }
}
```

**Cross-references:** EVENT-MODEL.md, ATTACK-SURFACE-NATS.md.

---

## H-18: CI/CD Tool Installation Without Pinning

**Finding:** CI/CD pipeline installs tools (staticcheck, golangci-lint, cosign) without version pinning or checksum verification. A supply chain attack on any tool's `latest` release compromises the build.

**Fix:** All tool installations use pinned versions with SHA-256 checksum verification. No `curl | sh` from main branches.

**Checksum file:** `.github/tool-checksums.json`

```json
{
  "tools": {
    "staticcheck": {
      "version": "2024.1.1",
      "checksums": {
        "linux-amd64": "sha256:a1b2c3d4e5f6..."
      }
    },
    "golangci-lint": {
      "version": "1.62.2",
      "checksums": {
        "linux-amd64": "sha256:b2c3d4e5f6a7..."
      }
    },
    "cosign": {
      "version": "2.4.1",
      "checksums": {
        "linux-amd64": "sha256:c3d4e5f6a7b8..."
      }
    },
    "goreleaser": {
      "version": "2.5.0",
      "checksums": {
        "linux-amd64": "sha256:d4e5f6a7b8c9..."
      }
    }
  }
}
```

**CI/CD changes:**

```yaml
# BEFORE (vulnerable):
- run: go install honnef.co/go/tools/cmd/staticcheck@latest

# AFTER (pinned + verified):
- name: Install staticcheck
  run: |
    VERSION="2024.1.1"
    EXPECTED_SHA=$(jq -r '.tools.staticcheck.checksums["linux-amd64"]' .github/tool-checksums.json)
    curl -fsSL -o staticcheck.tar.gz \
      "https://github.com/dominikh/go-tools/releases/download/${VERSION}/staticcheck_linux_amd64.tar.gz"
    ACTUAL_SHA="sha256:$(sha256sum staticcheck.tar.gz | cut -d' ' -f1)"
    if [ "$ACTUAL_SHA" != "$EXPECTED_SHA" ]; then
      echo "CHECKSUM MISMATCH for staticcheck: expected $EXPECTED_SHA, got $ACTUAL_SHA"
      exit 1
    fi
    tar xzf staticcheck.tar.gz
    mv staticcheck/staticcheck /usr/local/bin/
```

**Cross-references:** CI-CD-PIPELINE.md Section 2.

---

## H-19: cosign Regexp Too Broad

**Finding:** CI-CD-PIPELINE.md uses `--certificate-identity-regexp ".*github.com/.*"` for cosign verification. This accepts signatures from ANY GitHub Actions workflow in ANY repository, not just LOOM's release workflow.

**Fix:** Use exact certificate identity matching:

```bash
# BEFORE (too broad -- accepts any GitHub repo):
cosign verify-blob \
  --certificate-identity-regexp ".*github.com/.*" \
  --certificate-oidc-issuer "https://token.actions.githubusercontent.com" \
  loom_0.3.0_linux_amd64.tar.gz

# AFTER (exact repo + workflow):
cosign verify-blob \
  --certificate-identity "https://github.com/wittedinit/LOOM/.github/workflows/release.yml@refs/tags/*" \
  --certificate-oidc-issuer "https://token.actions.githubusercontent.com" \
  loom_0.3.0_linux_amd64.tar.gz
```

**Update in all locations:**

| File | Change |
|------|--------|
| CI-CD-PIPELINE.md | Replace `--certificate-identity-regexp` with exact `--certificate-identity` |
| BINARY-PROTECTION.md | Update cosign verification example |
| RELEASE-PROCESS.md | Update release checklist verification step |
| SECURITY-POLICY.md | Update binary signing documentation |

**Cross-references:** CI-CD-PIPELINE.md Section 8 (Signed Binaries), RELEASE-PROCESS.md.

---

## H-20: Binary Self-Hash Trivially Bypassable

**Finding:** Binary self-hash (computing own hash at runtime and comparing to embedded expected hash) is trivially defeated: an attacker patches both the binary code AND the embedded hash.

**Fix:** Acknowledge the limitation. Self-hash is defense-in-depth layer, not primary protection.

**Documented in BINARY-PROTECTION.md:**

> **Self-hash integrity check: acknowledged limitations**
>
> The runtime self-hash check detects casual tampering (accidental corruption, unsophisticated modification) but does NOT protect against a determined attacker who can patch both the binary and its embedded hash. This is a fundamental limitation of self-verification without external trust anchors.
>
> **Primary protection (strong):** Code signing via cosign + Sigstore transparency log. The signature is verified by an external party (the deployment system, Kubernetes admission controller, or TUF client) BEFORE the binary executes. An attacker cannot forge a cosign signature without the OIDC identity.
>
> **Secondary protection (moderate):** Distribution integrity via TUF (edge agents) or container image signatures (hub services). The binary is verified before it is placed on the target system.
>
> **Tertiary protection (weak, defense-in-depth only):** Runtime self-hash. Detects corruption and unsophisticated tampering. Does not resist a targeted attacker. Retained because it costs nothing and catches a class of accidents.

**Cross-references:** BINARY-PROTECTION.md Section 5.

---

## H-21: LLM Exfiltration via Reasoning Field

**Finding:** `LLMRecommendation.Reasoning` stores the LLM's free-text explanation. A prompt injection attack could cause the LLM to encode exfiltrated data (credentials, device configs, tenant data) in the Reasoning field. If Reasoning is stored in the audit trail or workflow history, the exfiltrated data persists in a queryable store.

**Fix:** Reasoning/thinking fields from LLM responses are NOT stored in the audit trail or workflow history. Only the structured JSON output is persisted.

**Data flow:**

```
LLM Response
  ├── Structured Output (Type, Confidence, Recommendation, Fallback)
  │     └── Persisted in: audit trail, workflow history, database
  │
  └── Reasoning field (free text)
        └── Written to: ephemeral debug log ONLY
              ├── Retention: 24 hours
              ├── Queryable: No (not indexed, not searchable)
              ├── Storage: local filesystem, rotated by logrotate
              └── Access: root only, not exposed via API
```

```go
// LLMRecommendation as persisted in audit trail and workflow history.
// Note: Reasoning field is EXCLUDED from persistence.
type LLMRecommendation struct {
    Type           string          `json:"type"`
    Confidence     float64         `json:"confidence"`
    Provenance     []string        `json:"provenance"`
    Recommendation any             `json:"recommendation"`
    Fallback       any             `json:"fallback"`
    // Reasoning is deliberately excluded from JSON serialization.
    // It is logged to the ephemeral debug log only.
    Reasoning      string          `json:"-"`
}
```

**Cross-references:** LLM-BOUNDARIES.md Rule 3, ATTACK-SURFACE-LLM.md, AUDIT-MODEL.md.

---

## H-22: IPAM Race Between Built-In and External

**Finding:** If both built-in IPAM and external IPAM (e.g., NetBox, Infoblox) are active for the same tenant, IP allocation races cause address conflicts and double-allocation.

**Fix:** IPAM mode is exclusive per tenant. A tenant uses either built-in OR external IPAM, never both simultaneously.

```go
type IPAMMode string

const (
    IPAMBuiltIn  IPAMMode = "builtin"   // LOOM manages IP allocations internally
    IPAMExternal IPAMMode = "external"  // External IPAM (NetBox, Infoblox, etc.)
)

type TenantIPAMConfig struct {
    TenantID    uuid.UUID `json:"tenant_id"`
    Mode        IPAMMode  `json:"mode"`
    // ExternalConfig is populated only when Mode == IPAMExternal.
    ExternalConfig *ExternalIPAMConfig `json:"external_config,omitempty"`
}

type ExternalIPAMConfig struct {
    Provider    string `json:"provider"`    // "netbox", "infoblox", "phpipam"
    Endpoint    string `json:"endpoint"`    // API URL
    CredentialRef string `json:"credential_ref"` // vault path for API credentials
}
```

**Migration between modes:** Requires an explicit cutover workflow:

1. Administrator initiates `IPAMCutoverWorkflow(tenantID, fromMode, toMode)`.
2. Workflow enters maintenance window (no new IP allocations during cutover).
3. All existing allocations are exported from the source IPAM.
4. Allocations are imported into the target IPAM.
5. Verification: every allocation in the source exists in the target.
6. Mode switch is committed atomically.
7. Maintenance window ends.

**Enforcement:** The IPAM allocation API checks `TenantIPAMConfig.Mode` on every request. If `Mode == IPAMExternal`, built-in allocation endpoints return `IPAM_MODE_MISMATCH` error.

**Cross-references:** GAP-ANALYSIS.md C6 (IPAM integration), DOMAIN-MODEL.md.

---

## Exploit Chain Mitigations

The adversarial audit identified 7 exploit chains (Alpha through Eta). Each chain is broken by at least 2 independent fixes from the findings above.

### Chain Alpha: Credential Exfiltration via Compromised Adapter

**Attack path:** Malicious adapter registers (H-09) -> receives device credentials -> exfiltrates via LLM reasoning field (H-21) -> persisted in audit trail.

**Broken by:**
1. **H-09 (adapter signing):** Malicious adapter cannot register without valid cosign signature and allowlist entry.
2. **H-21 (reasoning field):** Even if adapter registers, LLM reasoning is not persisted in queryable storage.
3. **H-15 (security metrics):** `loom_adapter_anomaly_score` detects unusual adapter behavior patterns.

### Chain Beta: Cross-Tenant Device Takeover via Identity Spoofing

**Attack path:** Attacker controls DNS (spoofs FQDN) -> low-confidence auto-merge (H-07) -> merges attacker device with victim device -> accesses victim's credentials.

**Broken by:**
1. **H-07 (identity thresholds):** FQDN (confidence 0.40) is below the 0.50 floor and cannot trigger auto-merge.
2. **H-15 (security metrics):** `loom_tenant_boundary_violations_total` alerts on cross-tenant access attempts.

### Chain Gamma: Approval Gate Bypass to Unauthorized Provisioning

**Attack path:** Attacker gains NATS access -> sends forged ApprovalDecision signal (H-08) -> workflow proceeds with unauthorized provisioning.

**Broken by:**
1. **H-08 (approval validation):** ApprovalDecision requires HMAC signature + server-side auth service validation.
2. **H-09 (adapter signing):** Even with approval bypass, the provisioning adapter must be legitimately signed.
3. **H-15 (security metrics):** `loom_authz_denials_total` catches validation failures.

### Chain Delta: Persistent Access via Expired Certificate

**Attack path:** Attacker steals agent certificate -> certificate expires -> 7-day grace period (H-05) -> attacker retains full operational access for 7 days.

**Broken by:**
1. **H-05 (grace period):** Reduced to 24 hours, read-only only. No mutations during grace.
2. **H-04 (CRL push):** Certificate revocation propagates in < 1 second via NATS push, not 1-hour poll.
3. **H-15 (security metrics):** `loom_certificate_expiry_seconds` triggers P0 alert.

### Chain Epsilon: Audit Trail Forgery to Cover Tracks

**Attack path:** DBA compromises primary PostgreSQL -> rewrites audit hash chain (H-03) -> covers tracks of other attacks.

**Broken by:**
1. **H-03 (external anchoring):** Merkle roots in separate PostgreSQL + S3 Object Lock. DBA cannot rewrite both.
2. **H-15 (security metrics):** `loom_credential_access_total` provides independent monitoring outside the audit trail.

### Chain Zeta: Supply Chain Attack via CI/CD

**Attack path:** Attacker compromises tool `latest` release (H-18) -> injects code into CI/CD build -> signs binary with broad cosign identity (H-19) -> pushes malicious binary to edge agents (H-14).

**Broken by:**
1. **H-18 (tool pinning):** Pinned versions + SHA-256 checksums prevent compromised `latest` releases from affecting builds.
2. **H-19 (cosign identity):** Exact certificate identity rejects signatures from non-LOOM workflows.
3. **H-14 (update verification):** Edge agents verify TUF metadata + Ed25519 signatures before applying updates.

### Chain Eta: Device Lock Starvation for Denial of Service

**Attack path:** Attacker triggers workflow on target device -> extends lock indefinitely (H-10) -> legitimate workflows for the device are blocked.

**Broken by:**
1. **H-10 (lock limits):** Maximum 30-minute absolute lock duration, maximum 3 extensions.
2. **H-11 (state machine):** Device can be moved to maintenance status, which cancels all running workflows.

---

## Unclosed Gaps from PR #1

### C1: Workflow State Reconciliation

**Status:** Added to EDGE-AGENT-SECURITY.md scope.

Workflow state reconciliation after network partitions is addressed by the reconciliation protocol in EDGE-AGENT-SECURITY.md Section 3 (three modes: hub-wins, edge-wins, and manual-merge). The specific addition: when an edge agent reconnects after a partition, it reports all locally-executed workflow steps (buffered in SQLite) to the hub. The hub's Temporal workflows are updated via `ReportWorkflowCompletion` activity to reconcile the edge-side execution with the hub-side workflow state. Conflicts (hub dispatched the same work to another agent during the partition) are resolved using the reconciliation mode configured for the tenant.

### H11 (from PR #1): LLM Cross-Tenant Context

**Fix:** Mandatory single-turn mode for all LLM interactions. No conversation history is maintained across requests. Each LLM call includes only the data for the requesting tenant.

**Per-tenant RAG isolation:** The pgvector-backed RAG index is partitioned by `tenant_id`. The LLM query builder enforces `WHERE tenant_id = $1` on every vector similarity search. Cross-tenant document retrieval is impossible at the query level (enforced by the same tenant scoping as all other LOOM queries).

```go
// LLMQueryContext is constructed per-request. No state is carried between requests.
type LLMQueryContext struct {
    TenantID     uuid.UUID       // strict tenant scope for RAG retrieval
    RequestType  string          // classification, placement, etc.
    InputData    any             // typed input for this specific request
    // NO ConversationHistory field -- single-turn only
    // NO CrossTenantContext field -- tenant-scoped RAG only
}
```

**Cross-references:** LLM-BOUNDARIES.md, ATTACK-SURFACE-LLM.md.

### H14 (from PR #1): ConfigSnapshot Encryption

**Status:** Fully addressed in SECURITY-HARDENING-RESPONSES.md Section 6. ConfigSnapshot encryption uses AES-256-GCM with a per-snapshot DEK wrapped by the tenant KEK (same envelope encryption pattern as the credential vault). The HMAC change in H-06 above (replacing SHA-256 checksum with HMAC-SHA256) is an additional hardening on top of the encryption.

**Cross-reference:** See [SECURITY-HARDENING-RESPONSES.md Section 6](../SECURITY-HARDENING-RESPONSES.md#6-configsnapshot-encryption-h14-sh-6).

### M11: GDPR

**Status:** Fully addressed in MEDIUM-FINDINGS-RESPONSES.md M11. Device metadata (hostnames, IPs, MAC addresses) is classified as PII under GDPR when it can identify natural persons. Right-to-erasure is implemented via tenant decommission (`TenantDecommissionResult`), which purges all tenant data from all stores (PostgreSQL, NATS, Temporal, Valkey, edge agent caches).

**Cross-reference:** See [MEDIUM-FINDINGS-RESPONSES.md M11](../MEDIUM-FINDINGS-RESPONSES.md#m11-gdpr-data-classification-and-right-to-erasure).

### Bootstrap Credential Paradox

**Problem:** LOOM manages device credentials in its vault, but LOOM itself needs credentials to connect to PostgreSQL, NATS, and Temporal. Where are LOOM's own credentials stored?

**Decision:** LOOM's own infrastructure credentials (database password, NATS NKey, Temporal mTLS certs) are stored in environment variables or Kubernetes secrets. They are NOT stored in the LOOM vault.

**Rationale:** The vault is for managed device credentials. LOOM's own credentials are bootstrap dependencies -- the vault cannot decrypt anything until LOOM connects to PostgreSQL (where the encrypted DEKs/KEKs are stored). Storing LOOM's DB password in the vault creates a circular dependency.

**Credential sources by deployment mode:**

| Mode | Credential Source | Example |
|------|------------------|---------|
| Mode 1 (dev) | Environment variables | `LOOM_DB_URL`, `LOOM_NATS_URL` |
| Mode 2 (HA) | Kubernetes Secrets (or HashiCorp Vault Agent injector) | `secretKeyRef: loom-db-credentials` |
| Mode 3 (edge hub) | Kubernetes Secrets + sealed-secrets for GitOps | Encrypted in git, decrypted by controller |
| Mode 4 (multi-region) | External secret manager (AWS Secrets Manager, Azure Key Vault) | CSI driver or init container injection |

**Security controls for bootstrap credentials:**

1. Environment variables are never logged (LOOM's logger redacts any env var containing `PASSWORD`, `SECRET`, `KEY`, `TOKEN`).
2. Kubernetes Secrets use `secretKeyRef` (not `configMapKeyRef`) and are mounted as files (not env vars) where possible.
3. Bootstrap credentials are rotated independently of device credentials, on a schedule determined by the infrastructure team.
4. LOOM does not manage, rotate, or audit its own bootstrap credentials -- that is the responsibility of the deployment platform.

---

## Implementation Priority

| Finding | Phase | Effort | Dependencies |
|---------|-------|--------|-------------|
| H-01 (SecureCredential mutex) | 1B | 2 days | Vault implementation |
| H-02 (nonce monitoring) | 1B | 1 day | Observability setup |
| H-03 (audit anchoring) | 2 | 1 week | External PostgreSQL + S3 |
| H-04 (CRL push) | 2 | 3 days | NATS PKI topic |
| H-05 (grace period) | 2 | 1 day | PKI implementation |
| H-06 (HMAC checksum) | 2 | 2 days | Vault DEK access |
| H-07 (identity thresholds) | 2 | 1 day | Identity model |
| H-08 (approval validation) | 4 | 3 days | Auth service, Temporal |
| H-09 (adapter signing) | 1B | 3 days | cosign integration |
| H-10 (lock limits) | 4 | 1 day | Device locking |
| H-11 (state machine) | 1B | 2 days | Domain model |
| H-12 (filter injection) | 3 | 2 days | Discovery contract |
| H-13 (sslmode) | 1A | 1 day | PostgreSQL setup |
| H-14 (update signing) | 3 | 3 days | TUF client |
| H-15 (security metrics) | 1B | 3 days | Prometheus setup |
| H-16 (trace stripping) | 4 | 1 day | OTel integration |
| H-17 (priority buffer) | 1B | 3 days | NATS client |
| H-18 (CI pinning) | 0 | 1 day | CI/CD pipeline |
| H-19 (cosign identity) | 0 | 30 min | CI/CD pipeline |
| H-20 (self-hash docs) | 0 | 30 min | Documentation |
| H-21 (reasoning field) | 4 | 1 day | LLM integration |
| H-22 (IPAM exclusion) | 3 | 2 days | IPAM module |

---

## Cross-Reference Index

| Document | Findings Addressed |
|----------|-------------------|
| VAULT-ARCHITECTURE.md | H-01, H-02, H-06 |
| AUDIT-MODEL.md | H-03, H-21 |
| SECURITY-HARDENING-RESPONSES.md | H-04, H-05, H-06, H-14 (ConfigSnapshot) |
| EDGE-AGENT-SECURITY.md | H-04, H-05, H-14, C1 |
| IDENTITY-MODEL.md | H-07 |
| WORKFLOW-CONTRACT.md | H-08 |
| ADAPTER-CONTRACT.md | H-09, H-12 |
| DEVICE-LOCKING.md | H-10 |
| DOMAIN-MODEL.md | H-11, H-22 |
| DISCOVERY-CONTRACT.md | H-12 |
| DEPLOYMENT-TOPOLOGY.md | H-13 |
| BINARY-PROTECTION.md | H-14, H-20 |
| OBSERVABILITY-STRATEGY.md | H-15, H-16 |
| LLM-BOUNDARIES.md | H-16, H-21, H11 (PR#1) |
| EVENT-MODEL.md | H-17 |
| CI-CD-PIPELINE.md | H-18, H-19 |
| MEDIUM-FINDINGS-RESPONSES.md | M11 (GDPR) |
| SECURITY-MODEL.md | Bootstrap credential paradox |
