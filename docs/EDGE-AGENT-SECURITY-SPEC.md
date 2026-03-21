# Edge Agent Security Specification

> **Status:** Approved — Phase 0 Contract
> **Resolves:** GAP-ANALYSIS.md findings C1, C4, C5
> **Derived from:** RESEARCH-edge-agent-security.md
> **Owner:** Security Architecture
> **Last updated:** 2026-03-21

---

## 1. Scope

This specification defines the security architecture for LOOM edge agents operating at remote sites with intermittent connectivity. It covers three domains:

1. **Credential Cache** (C4) -- How the edge agent encrypts, stores, and manages device credentials offline.
2. **Auto-Update Trust** (C5) -- How agent binary updates are signed, distributed, verified, and rolled back.
3. **Split-Brain Conflict Resolution** (C1) -- How the hub and edge agent reconcile divergent state after a network partition.

Each section contains concrete Go type definitions, protocol flows, state machines, threat mitigations, failure modes, and integration points. The specification is implementation-ready.

---

## 2. Credential Cache

### 2.1 Decision

**TPM 2.0 is RECOMMENDED, not required.** Edge agents operate on diverse hardware -- x86 servers with TPM 2.0, Raspberry Pi 5 with optional external TPM modules, and commodity ARM devices without hardware security. Mandating TPM would exclude a significant portion of the target hardware.

**Fallback: passphrase-derived KEK using Argon2id.** When TPM is unavailable, the agent derives its KEK from a provisioning passphrase using Argon2id (memory-hard, side-channel resistant). This passphrase is entered once during agent provisioning and never stored in plaintext. The derived KEK is held in mlock'd memory during agent runtime.

**Rationale:** This mirrors the VAULT-ARCHITECTURE.md master key provider hierarchy (HSM > Vault Transit > Cloud KMS > TPM > Local passphrase). The edge agent operates at Tier 4-5 of this hierarchy. Making TPM optional with a secure fallback maximizes deployment reach without compromising the fundamental encryption guarantees.

### 2.2 Specification

#### 2.2.1 Key Hierarchy (Edge Agent)

```
Agent KEK (Key Encryption Key)
  └── sealed to TPM SRK (if TPM available)
  └── OR derived from provisioning passphrase via Argon2id
        └── Credential DEK (Data Encryption Key) — one per cached credential
              └── AES-256-GCM encrypted credential payload
```

The edge agent key hierarchy is a simplified 2-tier version of the hub's 3-tier hierarchy. The agent KEK fills the role of both MEK and KEK -- this is acceptable because the agent is single-tenant (scoped to one site within one tenant) and managing a bounded credential set.

#### 2.2.2 Go Types

```go
package agent

import (
    "crypto/aes"
    "crypto/cipher"
    "crypto/rand"
    "time"

    "github.com/google/uuid"
)

// AgentKEKProvider abstracts the source of the agent's Key Encryption Key.
// Implementations: TPMKEKProvider, PassphraseKEKProvider.
type AgentKEKProvider interface {
    // Unseal returns the agent KEK in mlock'd memory.
    // For TPM: unseals from the TPM's Storage Root Key.
    // For passphrase: derives from the passphrase using Argon2id.
    Unseal() (*SecureBytes, error)

    // ProviderType returns the KEK source for audit logging.
    ProviderType() AgentKEKProviderType

    // Destroy zeroes the KEK from memory and (if TPM) flushes the handle.
    Destroy()
}

type AgentKEKProviderType string

const (
    AgentKEKProviderTPM        AgentKEKProviderType = "tpm2"
    AgentKEKProviderPassphrase AgentKEKProviderType = "argon2id_passphrase"
)

// TPMKEKProvider seals/unseals the agent KEK to the TPM 2.0 SRK.
// Library: github.com/google/go-tpm/tpm2
type TPMKEKProvider struct {
    // DevicePath is the TPM device (e.g., "/dev/tpmrm0").
    DevicePath string
    // SealedKeyPath is the file containing the TPM-sealed KEK blob.
    SealedKeyPath string
    // PCRSelection defines which PCRs must match for unseal (measured boot).
    // Default: PCRs 0, 2, 4, 7 (BIOS, option ROMs, boot loader, Secure Boot state).
    PCRSelection []int
}

// PassphraseKEKProvider derives the agent KEK from a provisioning passphrase.
type PassphraseKEKProvider struct {
    // SaltPath is the file containing the Argon2id salt (32 bytes, random).
    // Generated once at provisioning time and stored on disk.
    SaltPath string
    // Passphrase is provided at startup (stdin, environment, or systemd credential).
    // Zeroed from memory immediately after KEK derivation.
    passphrase []byte
}

// Argon2idParams are fixed and non-configurable. Matching VAULT-ARCHITECTURE.md.
var Argon2idParams = struct {
    Time    uint32
    Memory  uint32 // KiB
    Threads uint8
    KeyLen  uint32
}{
    Time:    3,
    Memory:  256 * 1024, // 256 MiB
    Threads: 4,
    KeyLen:  32, // 256-bit KEK
}

// CachedCredential is a single credential stored in the agent's encrypted cache.
type CachedCredential struct {
    // ID matches the CredentialRef.ID from the hub's vault.
    ID uuid.UUID `db:"id"`

    // SiteID scopes this credential to a specific site.
    SiteID string `db:"site_id"`

    // TenantID scopes this credential to a specific tenant.
    TenantID uuid.UUID `db:"tenant_id"`

    // CredentialType identifies the authentication mechanism.
    CredentialType string `db:"credential_type"`

    // Ciphertext is the AES-256-GCM encrypted credential payload.
    Ciphertext []byte `db:"ciphertext"`

    // Nonce is the 96-bit random nonce used for AES-256-GCM encryption.
    Nonce []byte `db:"nonce"`

    // EncryptedDEK is the per-credential DEK, encrypted by the agent KEK.
    EncryptedDEK []byte `db:"encrypted_dek"`

    // DEKNonce is the nonce used to encrypt the DEK with the KEK.
    DEKNonce []byte `db:"dek_nonce"`

    // AAD is the additional authenticated data bound to this ciphertext.
    // Contains: credential_id || site_id || tenant_id (prevents relocation attacks).
    AAD []byte `db:"aad"`

    // IntegrityHash is a BLAKE3 hash of (ciphertext || encrypted_dek || aad).
    IntegrityHash []byte `db:"integrity_hash"`

    // HubVersion is the credential version as known by the hub at distribution time.
    HubVersion int `db:"hub_version"`

    // CachedAt is when this credential was cached locally.
    CachedAt time.Time `db:"cached_at"`

    // ExpiresAt is the hard expiry for this cached copy.
    // Default: CachedAt + 7 days. After this time, the credential is wiped.
    ExpiresAt time.Time `db:"expires_at"`

    // DeviceIDs lists which devices at this site use this credential.
    // Stored as JSON array in SQLite.
    DeviceIDs []uuid.UUID `db:"device_ids"`
}

// CredentialCacheConfig defines the credential cache behavior.
type CredentialCacheConfig struct {
    // MaxOfflineDuration is the maximum time credentials remain cached
    // without hub contact. Default: 7 days (168 hours).
    // After this duration, all cached credentials are wiped and the agent
    // enters degraded mode (monitoring only, no device mutations).
    MaxOfflineDuration time.Duration `yaml:"max_offline_duration" default:"168h"`

    // WipeOnTamper controls whether the cache is wiped when tamper
    // indicators are detected (TPM PCR mismatch, integrity hash failure).
    WipeOnTamper bool `yaml:"wipe_on_tamper" default:"true"`

    // DBPath is the path to the encrypted credential cache SQLite file.
    DBPath string `yaml:"db_path" default:"/var/lib/loom/credential-cache.db"`
}
```

#### 2.2.3 Credential Distribution Flow

```
Hub                                    Edge Agent
 │                                         │
 │  1. Agent connects via mTLS             │
 │◄────────────────────────────────────────│
 │                                         │
 │  2. Hub verifies agent certificate      │
 │     - Extracts SiteID from cert SAN     │
 │     - Validates cert is not revoked     │
 │     - Checks cert scope matches         │
 │       requested credentials             │
 │                                         │
 │  3. Agent requests credentials for      │
 │     its site's devices                  │
 │────────────────────────────────────────►│
 │                                         │
 │  4. Hub queries credential manifest:    │
 │     - Which credentials are assigned    │
 │       to devices at this site?          │
 │     - Which credentials have changed    │
 │       since the agent's last sync?      │
 │                                         │
 │  5. Hub decrypts credentials from       │
 │     vault and sends over mTLS channel   │
 │     (each credential individually,      │
 │     never bulk)                         │
 │────────────────────────────────────────►│
 │                                         │
 │  6. Agent receives each credential:     │
 │     a. Generates random DEK (256-bit)   │
 │     b. Encrypts payload with DEK        │
 │        (AES-256-GCM, random nonce)      │
 │     c. Encrypts DEK with agent KEK      │
 │        (AES-256-GCM, random nonce)      │
 │     d. Computes BLAKE3 integrity hash   │
 │     e. Writes CachedCredential to       │
 │        SQLite                           │
 │     f. Zeroes plaintext from memory     │
 │                                         │
 │  7. Hub updates credential manifest:    │
 │     - Records: credential_id,           │
 │       agent_id, site_id, version,       │
 │       distributed_at                    │
 │────────────────────────────────────────►│
 │                                         │
```

#### 2.2.4 Per-Site Credential Scoping

**Decision:** Per-site credential scoping -- the agent only receives credentials for devices at its assigned site.

The hub maintains a `credential_manifest` table that tracks which credentials have been distributed to which agents:

```go
// CredentialManifestEntry records a credential distribution to an agent.
// The hub uses this table to:
// 1. Limit credential distribution to site-scoped agents
// 2. Track which agents hold which credential versions
// 3. Trigger credential rotation on agent decommission
type CredentialManifestEntry struct {
    ID            uuid.UUID `db:"id"`
    CredentialID  uuid.UUID `db:"credential_id"`
    AgentID       uuid.UUID `db:"agent_id"`
    SiteID        string    `db:"site_id"`
    TenantID      uuid.UUID `db:"tenant_id"`
    Version       int       `db:"version"`        // credential version at distribution time
    DistributedAt time.Time `db:"distributed_at"`
    RevokedAt     *time.Time `db:"revoked_at"`     // set on agent decommission
}
```

**Authentication of credential requests:** The agent authenticates to the hub via mTLS. The agent's client certificate contains the SiteID in the Subject Alternative Name (SAN) extension as a URI: `loom://site/{site_id}/agent/{agent_id}`. The hub extracts the SiteID from the certificate and enforces that the agent can only request credentials for devices assigned to that site. A rogue agent cannot fetch credentials for a different site because its certificate SAN will not match.

```go
// AgentCertificateScope extracts the site scope from an agent's mTLS certificate.
type AgentCertificateScope struct {
    AgentID  uuid.UUID
    SiteID   string
    TenantID uuid.UUID
    // IssuedAt and ExpiresAt from the certificate itself.
    IssuedAt  time.Time
    ExpiresAt time.Time
}

// ExtractAgentScope parses the agent's mTLS client certificate and returns
// the scope fields. Returns an error if the certificate SAN format is invalid
// or the certificate is revoked.
func ExtractAgentScope(cert *x509.Certificate, crl *x509.RevocationList) (*AgentCertificateScope, error)
```

#### 2.2.5 Credential Cache Lifecycle State Machine

```
                    ┌──────────────────────────────┐
                    │           ONLINE              │
                    │  Credentials synced from hub  │
                    │  Cache refreshed on each sync │
                    └──────────┬───────────────────┘
                               │
                    Hub connection lost
                               │
                    ┌──────────▼───────────────────┐
                    │       OFFLINE_CACHED          │
                    │  Using cached credentials     │
                    │  Timer: MaxOfflineDuration     │
                    │  Full device operations        │
                    └──────────┬───────────────────┘
                               │
               ┌───────────────┼──────────────────┐
               │               │                  │
        Hub reconnects    Timer expires    Tamper detected
               │               │            (WipeOnTamper)
               │               │                  │
    ┌──────────▼──────┐  ┌─────▼──────────┐  ┌───▼──────────────┐
    │     ONLINE      │  │   DEGRADED     │  │    WIPED         │
    │ (re-sync creds) │  │  Creds wiped   │  │  Creds destroyed │
    │                 │  │  Monitor only  │  │  Agent halted    │
    │                 │  │  No mutations  │  │  Requires re-    │
    │                 │  │  No device     │  │  provisioning    │
    │                 │  │  auth possible │  │                  │
    └─────────────────┘  └───────┬────────┘  └──────────────────┘
                                 │
                          Hub reconnects
                                 │
                         ┌───────▼────────┐
                         │    ONLINE      │
                         │ (re-sync all   │
                         │  credentials)  │
                         └────────────────┘
```

**MaxOfflineDuration: 7 days.** After 7 days without hub contact, all cached credentials are wiped and the agent enters degraded mode. This limits the exposure window from a stolen device. The 7-day window is a balance between operational continuity (edge sites may lose connectivity for days) and security (limiting the time stolen credentials remain usable).

**Degraded mode behavior:**
- Discovery scans continue (no credentials needed for ICMP/ARP scanning).
- SNMP community strings are wiped -- SNMP discovery and monitoring stop.
- SSH/Redfish/IPMI operations are impossible -- no credentials.
- The agent continues buffering events and telemetry from passive sources.
- The agent reports its degraded status in heartbeat messages.
- On hub reconnect, the agent re-syncs all credentials and exits degraded mode.

#### 2.2.6 Credential Cache Wipe Triggers

The credential cache is wiped (all rows deleted, SQLite VACUUM'd) under these conditions:

| Trigger | Condition | Behavior |
|---------|-----------|----------|
| **Time expiry** | `now() > ExpiresAt` on any credential | Individual credential wiped. If all credentials expired, enter DEGRADED. |
| **Offline duration** | Hub unreachable for > `MaxOfflineDuration` | All credentials wiped. Enter DEGRADED. |
| **Tamper detection** | TPM PCR mismatch (boot chain changed) | All credentials wiped. Enter WIPED. Requires re-provisioning. |
| **Integrity failure** | BLAKE3 hash mismatch on cache read | Affected credential wiped. Alert logged. If >50% of credentials fail integrity, wipe all and enter WIPED. |
| **Hub-initiated revocation** | Hub sends revocation command via NATS | Specific credential wiped. Agent re-requests if needed. |
| **Agent decommission** | Hub sends decommission command | All credentials wiped. Agent certificate revoked. Agent shuts down. |

#### 2.2.7 Decommission Protocol

When an edge agent is decommissioned:

1. Hub marks the agent's certificate as revoked in the CRL.
2. Hub sends a `loom.hub.commands.{site_id}.agent_decommission` NATS message.
3. Agent receives the message and:
   a. Wipes all cached credentials (SQLite DELETE + VACUUM).
   b. Zeroes the agent KEK from memory.
   c. If TPM: destroys the sealed KEK blob.
   d. If passphrase: deletes the Argon2id salt file.
   e. Deletes the agent certificate and private key.
   f. Logs the decommission event to the local decision log.
   g. Exits the process.
4. Hub queries the credential manifest for all credentials distributed to this agent.
5. Hub triggers credential rotation for every credential in the manifest.
6. Hub marks all manifest entries as revoked.

If the agent is offline during decommission (cannot receive the NATS message):
- The hub revokes the certificate and rotates all distributed credentials.
- When the agent eventually reconnects, its mTLS handshake fails (revoked certificate).
- The agent detects the authentication failure and self-wipes.

### 2.3 Threat Mitigations

| Threat | Mitigation |
|--------|------------|
| **Physical theft of edge device** | Credentials encrypted at rest with AES-256-GCM. KEK sealed to TPM (hardware-bound) or derived from passphrase (not on disk). Attacker must defeat TPM or brute-force Argon2id (256 MiB memory, 3 iterations). 7-day expiry limits exposure window. |
| **Root access via OS vulnerability** | KEK in mlock'd memory (not swappable). Core dumps disabled (PR_SET_DUMPABLE=0). Credential plaintext zeroed after use. Attacker with root can dump memory but the credential lifetime in memory is bounded to active-use duration. |
| **Supply chain compromise of edge hardware** | TPM PCR validation detects boot chain modification. On PCR mismatch, credentials are wiped. Without TPM: supply chain risk is accepted at the passphrase-derived tier. |
| **Disk forensics after decommissioning** | Decommission protocol wipes credentials, deletes KEK material, and the hub rotates all distributed credentials. Even if disk is recovered, the credentials found are no longer valid. SQLite VACUUM prevents deleted data recovery from database pages. |
| **Rogue agent fetching wrong-site credentials** | mTLS certificate SAN contains SiteID. Hub enforces scope: credential requests outside the agent's site are rejected. Certificate revocation prevents decommissioned agents from authenticating. |
| **Credential staleness after hub rotation** | Hub tracks credential versions in the manifest. On reconnect, the agent syncs and receives updated credentials. Stale credentials may fail to authenticate to devices -- the agent reports the failure and the hub pushes the updated credential. |

### 2.4 Failure Modes

| Failure | Impact | Behavior | Recovery |
|---------|--------|----------|----------|
| **TPM unavailable at runtime** | Agent cannot unseal KEK. Cannot decrypt any cached credentials. | Agent enters DEGRADED mode immediately. Logs `FATAL: TPM unseal failed`. Monitoring-only operation. | Investigate TPM failure (hardware, driver, PCR state). If TPM is permanently lost, re-provision agent with passphrase-derived KEK. |
| **TPM PCR mismatch** | Boot chain changed (kernel update, firmware change, or tampering). | Agent wipes all credentials. Enters WIPED state. Requires re-provisioning. | If legitimate change (kernel update): re-provision agent with new PCR baseline. If tampering: investigate, re-image device, re-provision. |
| **Passphrase lost** | Agent cannot derive KEK after restart. | Agent cannot start credential cache. Enters DEGRADED mode. | Re-provision agent with a new passphrase. Hub re-distributes all credentials. |
| **SQLite credential cache corrupted** | Credential reads fail with database errors. | Agent wipes corrupted cache, creates new empty database. Enters DEGRADED until hub reconnect. | Automatic on hub reconnect -- credentials are re-distributed. |
| **Hub unreachable for >7 days** | Credentials expire and are wiped. | Agent enters DEGRADED mode. Monitoring only. Events buffered. | Hub reconnect triggers full credential re-sync. Agent exits DEGRADED. |
| **Clock drift causes premature expiry** | Credentials wiped before 7 days (if clock jumps forward). | Agent enters DEGRADED mode prematurely. | NTP correction fixes the clock. Hub reconnect re-syncs credentials. |

### 2.5 Integration Points

| System | Integration |
|--------|-------------|
| **Hub Vault** (VAULT-ARCHITECTURE.md) | Hub decrypts credentials from vault using its 3-tier key hierarchy, then transmits plaintext to the agent over mTLS. The agent re-encrypts with its own 2-tier key hierarchy. The hub vault's CredentialService.Retrieve() is called once per credential per sync. |
| **NATS** (DEPLOYMENT-TOPOLOGY.md) | Credential revocation and decommission commands sent via NATS hub-to-edge subjects. Credential sync can also occur via NATS request/reply for delta updates between full syncs. |
| **Agent Certificate** (SECURITY-MODEL.md) | Agent mTLS certificate serves dual purpose: NATS leaf node authentication and credential request scoping. Certificate lifecycle managed by hub's internal CA (step-ca). |
| **SQLite** (DEPLOYMENT-TOPOLOGY.md) | Credential cache stored in a separate SQLite database from the agent's operational state database. This isolates credential wipe operations from operational state. |

### 2.6 Implementation Notes

**Libraries:**
- TPM 2.0: `github.com/google/go-tpm/tpm2` (pure Go, no CGO)
- Argon2id: `golang.org/x/crypto/argon2`
- AES-256-GCM: `crypto/aes` + `crypto/cipher` (Go stdlib, AES-NI accelerated)
- BLAKE3: `github.com/zeebo/blake3`
- SQLite: `modernc.org/sqlite` (pure Go, no CGO, consistent with DEPLOYMENT-TOPOLOGY.md)
- mlock/munlock: `syscall.Mlock` / `syscall.Munlock`

**Constraints:**
- AES-NI is NOT mandatory for edge agents (unlike the hub vault). Edge agents may run on ARM devices (Raspberry Pi) without AES-NI. Go's crypto/aes uses constant-time software fallback on non-x86 architectures. The agent logs a warning if AES-NI is absent but continues operating.
- The passphrase delivery mechanism for PassphraseKEKProvider uses systemd's `LoadCredential=` directive (systemd v250+). The passphrase is read from a credential file that can be provisioned via systemd-creds or manually placed. It is NOT passed via environment variable (visible in /proc/PID/environ).

---

## 3. Auto-Update Trust

### 3.1 Decision

**TUF (The Update Framework) for update distribution.** TUF is a CNCF graduated project, battle-tested in Docker Content Trust, PyPI, Uptane (automotive OTA), and Sigstore. It addresses every identified gap: threshold signatures, key rotation without fleet redeployment, rollback protection, replay protection, and consistent snapshots.

**2-of-3 threshold signatures on the targets role.** No single key compromise allows pushing a malicious binary to the fleet. Three signing keys are held by three different individuals/roles (e.g., engineering lead, security engineer, release manager). Any two must sign a release.

**Root keys stored on air-gapped machine, ceremony-based rotation.** Root key material never touches a networked computer. Root key ceremonies follow a documented procedure with witnesses, as is standard for PKI root CA operations.

**Rationale:** TUF's complexity is justified by the threat model. A compromised auto-update mechanism gives an attacker code execution on every edge agent in the fleet. The complexity cost is borne by the build/release pipeline, not the edge agent -- the agent embeds a TUF client that verifies metadata, which is straightforward to implement.

### 3.2 Specification

#### 3.2.1 TUF Role Hierarchy

```
Root Role (offline, air-gapped)
  ├── Threshold: 2-of-3 (root key holders)
  ├── Rotation: ceremony-based, annual or on compromise
  ├── Signs: targets role key, snapshot role key, timestamp role key
  │
  ├── Targets Role (build pipeline)
  │     ├── Threshold: 2-of-3 (signing keys)
  │     ├── Rotation: via root role delegation (old root signs new targets key)
  │     ├── Signs: agent binary metadata (hash, size, version, rollout config)
  │     └── Custom metadata: rollout stage, platform, architecture
  │
  ├── Snapshot Role (automated)
  │     ├── Threshold: 1-of-1 (automated key in CI pipeline)
  │     ├── Rotation: via root role
  │     ├── Signs: snapshot of all targets metadata versions
  │     └── Purpose: atomicity — ensures targets and delegations are consistent
  │
  └── Timestamp Role (automated)
        ├── Threshold: 1-of-1 (automated key in CI pipeline)
        ├── Rotation: via root role
        ├── Signs: timestamp + snapshot version
        ├── Expiry: 24 hours (agent rejects stale timestamps)
        └── Purpose: freshness — prevents freeze attacks
```

#### 3.2.2 Go Types

```go
package autoupdate

import (
    "time"
)

// TUFConfig configures the agent's TUF client.
type TUFConfig struct {
    // RepositoryURL is the base URL of the TUF repository on the hub.
    // Example: "https://hub.example.com/api/v1/updates/tuf"
    RepositoryURL string `yaml:"repository_url"`

    // LocalMetadataPath stores the agent's local copy of TUF metadata.
    // Used for rollback protection (version comparison against local state).
    LocalMetadataPath string `yaml:"local_metadata_path" default:"/var/lib/loom/tuf-metadata"`

    // CheckInterval is how often the agent checks for updates.
    CheckInterval time.Duration `yaml:"check_interval" default:"1h"`

    // MaxTimestampAge is the maximum acceptable age of the timestamp metadata.
    // If the timestamp is older than this, the agent rejects the update
    // (protects against freeze attacks). Default: 24 hours.
    MaxTimestampAge time.Duration `yaml:"max_timestamp_age" default:"24h"`

    // RolloutGroup determines which rollout stage this agent belongs to.
    // Set at provisioning time. Values: "canary", "early", "standard", "conservative".
    RolloutGroup string `yaml:"rollout_group" default:"standard"`

    // PreviousBinaryPath is where the agent stores the previous binary for rollback.
    PreviousBinaryPath string `yaml:"previous_binary_path" default:"/opt/loom/agent.prev"`
}

// UpdateTarget represents a signed binary in the TUF repository.
type UpdateTarget struct {
    // Name is the target filename (e.g., "loom-agent-v1.2.3-linux-amd64").
    Name string `json:"name"`

    // Version is the semantic version of the binary.
    Version string `json:"version"`

    // Length is the size of the binary in bytes.
    Length int64 `json:"length"`

    // Hashes contains the cryptographic hashes of the binary.
    // TUF requires at least SHA-256 and SHA-512.
    Hashes map[string]string `json:"hashes"`

    // Custom contains LOOM-specific metadata for the target.
    Custom UpdateTargetCustom `json:"custom"`
}

// UpdateTargetCustom carries rollout and compatibility metadata.
type UpdateTargetCustom struct {
    // RolloutConfig defines the staged rollout parameters.
    Rollout RolloutConfig `json:"rollout"`

    // Platform is the target OS (e.g., "linux").
    Platform string `json:"platform"`

    // Architecture is the target CPU arch (e.g., "amd64", "arm64").
    Architecture string `json:"architecture"`

    // MinAgentVersion is the minimum current version required for this update.
    // Prevents skipping versions that include migration steps.
    MinAgentVersion string `json:"min_agent_version,omitempty"`

    // ReleaseNotes is a human-readable summary for audit.
    ReleaseNotes string `json:"release_notes,omitempty"`
}

// RolloutConfig defines the staged rollout parameters.
type RolloutConfig struct {
    // Stages defines the rollout progression.
    Stages []RolloutStage `json:"stages"`

    // CurrentStage is the index into Stages that is currently active.
    // Updated by the hub as health gates pass.
    CurrentStage int `json:"current_stage"`

    // HaltedReason is non-empty if the rollout has been halted.
    HaltedReason string `json:"halted_reason,omitempty"`
}

// RolloutStage defines a single stage in the rollout progression.
type RolloutStage struct {
    // Name identifies the stage (e.g., "canary", "early_adopter", "majority", "full").
    Name string `json:"name"`

    // Percentage is the percentage of agents in this group that should update.
    Percentage int `json:"percentage"`

    // Groups lists which RolloutGroup values are eligible at this stage.
    Groups []string `json:"groups"`

    // HealthGateDuration is how long the stage must remain healthy
    // before advancing to the next stage.
    HealthGateDuration time.Duration `json:"health_gate_duration"`

    // StartedAt is when this stage was activated. Zero means not yet started.
    StartedAt *time.Time `json:"started_at,omitempty"`
}

// AgentHealthReport is sent by the agent after applying an update.
type AgentHealthReport struct {
    // AgentID identifies the reporting agent.
    AgentID string `json:"agent_id"`

    // SiteID identifies the agent's site.
    SiteID string `json:"site_id"`

    // Version is the agent version after the update.
    Version string `json:"version"`

    // PreviousVersion is the agent version before the update.
    PreviousVersion string `json:"previous_version"`

    // BootSuccess indicates the agent process started without errors.
    BootSuccess bool `json:"boot_success"`

    // HubConnectivity indicates the agent can reach the hub NATS cluster.
    HubConnectivity bool `json:"hub_connectivity"`

    // DeviceReachable indicates at least one managed device is reachable.
    DeviceReachable bool `json:"device_reachable"`

    // HealthCheckPassed indicates all three conditions above are true.
    HealthCheckPassed bool `json:"health_check_passed"`

    // UpdateAppliedAt is when the binary was replaced.
    UpdateAppliedAt time.Time `json:"update_applied_at"`

    // ReportedAt is when this health report was generated.
    ReportedAt time.Time `json:"reported_at"`
}
```

#### 3.2.3 Staged Rollout Progression

```
Stage 1: Canary (5%)
  Groups: ["canary"]
  Health gate: 1 hour
  ─────────────────────────────────────────
  │ 5% of agents update                    │
  │ Health check: boot + hub + 1 device    │
  │ If any canary fails: HALT rollout      │
  │ If all canary healthy for 1h: advance  │
  ─────────────────────────────────────────
                    │
                    ▼
Stage 2: Early Adopter (25%)
  Groups: ["canary", "early"]
  Health gate: 1 hour
  ─────────────────────────────────────────
  │ 25% of agents update                   │
  │ Same health check                      │
  │ If >10% fail: HALT rollout             │
  │ If <10% fail for 1h: advance           │
  ─────────────────────────────────────────
                    │
                    ▼
Stage 3: Majority (50%)
  Groups: ["canary", "early", "standard"]
  Health gate: 1 hour
  ─────────────────────────────────────────
  │ 50% of agents update                   │
  │ Same health check                      │
  │ If >5% fail: HALT rollout              │
  │ If <5% fail for 1h: advance            │
  ─────────────────────────────────────────
                    │
                    ▼
Stage 4: Full (100%)
  Groups: ["canary", "early", "standard", "conservative"]
  Health gate: none (final stage)
  ─────────────────────────────────────────
  │ All remaining agents update             │
  │ Health monitoring continues             │
  │ Hub can issue fleet-wide rollback       │
  ─────────────────────────────────────────
```

**Health check definition:** An update is considered healthy when all three conditions are met within 5 minutes of the binary being replaced:

1. **Boot success** -- The agent process starts without fatal errors.
2. **Hub connectivity** -- The agent's NATS leaf node connects to the hub.
3. **Device reachable** -- At least one managed device at the site responds to a health probe (ICMP ping, SNMP sysUpTime, or Redfish GET /redfish/v1).

**Failure thresholds per stage:**
- Stage 1 (canary): Any single failure halts the rollout.
- Stage 2 (25%): >10% failure rate halts the rollout.
- Stage 3 (50%): >5% failure rate halts the rollout.
- Stage 4 (100%): No automatic halt. Hub monitors and can issue manual rollback.

#### 3.2.4 Agent Update Flow

```
Agent                           Hub TUF Repository
  │                                      │
  │ 1. Check for update (every 1h)       │
  │   GET /tuf/timestamp.json            │
  │─────────────────────────────────────►│
  │◄─────────────────────────────────────│
  │                                      │
  │ 2. Compare timestamp version with    │
  │    local metadata (rollback check)   │
  │    If timestamp version <= local:    │
  │    skip (no update or freeze attack) │
  │                                      │
  │ 3. Verify timestamp signature        │
  │    Check timestamp age < 24h         │
  │                                      │
  │ 4. Fetch snapshot.json               │
  │   GET /tuf/snapshot.json             │
  │─────────────────────────────────────►│
  │◄─────────────────────────────────────│
  │                                      │
  │ 5. Verify snapshot signature         │
  │    Compare snapshot version against  │
  │    local (rollback protection)       │
  │                                      │
  │ 6. Fetch targets.json               │
  │   GET /tuf/targets.json             │
  │─────────────────────────────────────►│
  │◄─────────────────────────────────────│
  │                                      │
  │ 7. Verify targets signature(s)       │
  │    Require 2-of-3 threshold          │
  │    Compare target version against    │
  │    current agent version             │
  │                                      │
  │ 8. Check rollout eligibility:        │
  │    - Is my RolloutGroup included     │
  │      in the current stage's Groups?  │
  │    - If not: skip, check again later │
  │                                      │
  │ 9. Download binary                   │
  │   GET /tuf/targets/{binary_name}     │
  │─────────────────────────────────────►│
  │◄─────────────────────────────────────│
  │                                      │
  │ 10. Verify binary hashes match       │
  │     targets metadata (SHA-256 +      │
  │     SHA-512)                         │
  │                                      │
  │ 11. Apply update:                    │
  │   a. Copy current binary to          │
  │      agent.prev                      │
  │   b. Write new binary to agent.new   │
  │   c. Verify new binary checksum      │
  │      on disk (re-read and hash)      │
  │   d. Rename agent.new → agent        │
  │      (atomic on Linux)               │
  │   e. Signal systemd to restart       │
  │                                      │
  │ 12. Systemd restarts agent with      │
  │     ExecStartPre watchdog            │
  │                                      │
  │ 13. New agent runs health check      │
  │     (boot + hub + device)            │
  │                                      │
  │ 14. If healthy within 5 minutes:     │
  │   a. Report AgentHealthReport        │
  │   b. Update local TUF metadata       │
  │   c. Continue normal operation       │
  │                                      │
  │ 14b. If NOT healthy within 5 min:    │
  │   a. Watchdog detects health failure │
  │   b. Rename agent.prev → agent       │
  │   c. Restart with previous binary   │
  │   d. Report rollback event to hub   │
  │                                      │
```

#### 3.2.5 Rollback Mechanism

The rollback mechanism uses a systemd `ExecStartPre` watchdog that validates health before declaring the update successful:

```ini
# /etc/systemd/system/loom-agent.service
[Unit]
Description=LOOM Edge Agent
After=network-online.target
Wants=network-online.target

[Service]
Type=notify
ExecStartPre=/opt/loom/agent-watchdog check-previous-health
ExecStart=/opt/loom/agent
WatchdogSec=300
Restart=on-failure
RestartSec=5

# Credential delivery for passphrase-based KEK
LoadCredential=agent-kek-passphrase:/etc/credstore/loom-agent-passphrase

[Install]
WantedBy=multi-user.target
```

```go
// WatchdogConfig defines the health watchdog behavior.
type WatchdogConfig struct {
    // HealthCheckTimeout is how long the watchdog waits for the agent
    // to report healthy after startup. If the timeout expires without
    // a healthy report, the watchdog triggers a rollback.
    HealthCheckTimeout time.Duration `yaml:"health_check_timeout" default:"5m"`

    // PreviousBinaryPath is the location of the previous agent binary.
    PreviousBinaryPath string `yaml:"previous_binary_path" default:"/opt/loom/agent.prev"`

    // CurrentBinaryPath is the location of the current agent binary.
    CurrentBinaryPath string `yaml:"current_binary_path" default:"/opt/loom/agent"`

    // MaxRollbackAttempts is the maximum number of consecutive rollbacks
    // before the agent halts and requires manual intervention.
    // Prevents infinite rollback loops.
    MaxRollbackAttempts int `yaml:"max_rollback_attempts" default:"3"`
}

// RollbackState tracks the rollback history, persisted to disk.
// This file survives binary replacement.
type RollbackState struct {
    // ConsecutiveRollbacks counts how many times the agent has rolled back
    // without a successful health check in between.
    ConsecutiveRollbacks int `json:"consecutive_rollbacks"`

    // LastSuccessfulVersion is the most recent version that passed health checks.
    LastSuccessfulVersion string `json:"last_successful_version"`

    // LastAttemptedVersion is the most recent version that was attempted.
    LastAttemptedVersion string `json:"last_attempted_version"`

    // LastRollbackAt is when the most recent rollback occurred.
    LastRollbackAt *time.Time `json:"last_rollback_at,omitempty"`

    // History records the last 10 update/rollback events for debugging.
    History []RollbackEvent `json:"history"`
}

// RollbackEvent records a single update or rollback.
type RollbackEvent struct {
    Timestamp   time.Time `json:"timestamp"`
    FromVersion string    `json:"from_version"`
    ToVersion   string    `json:"to_version"`
    Action      string    `json:"action"` // "update_applied", "rollback", "health_passed"
    Reason      string    `json:"reason,omitempty"`
}
```

#### 3.2.6 Air-Gapped Update Mechanism

Air-gapped environments use a USB-delivered TUF repository. The agent's TUF client operates identically -- it verifies the metadata chain regardless of transport.

```
Air-Gapped Update Procedure:

1. On internet-connected build machine:
   $ loom-release build --version v1.2.3 --platforms linux/amd64,linux/arm64
   $ loom-release sign --key signing-key-1.pem
   $ loom-release sign --key signing-key-2.pem
   $ loom-release publish --repo /path/to/tuf-repo
   $ cp -r /path/to/tuf-repo /media/usb/loom-updates/

2. Transfer USB to air-gapped environment.

3. On air-gapped hub:
   $ loom admin update import --from /media/usb/loom-updates/
   (Hub verifies TUF metadata chain before importing)

4. Agents pick up the update from the hub's TUF repository on their
   next check interval.

Note: The USB-delivered TUF repository contains the full metadata chain
(root.json, targets.json, snapshot.json, timestamp.json). The agent
verifies the same trust chain as in connected mode.
```

#### 3.2.7 Key Rotation

TUF handles key rotation without fleet redeployment:

**Targets key rotation (routine):**
1. New targets key pair generated.
2. The root role signs a new root.json that delegates to the new targets key.
3. Old root.json and new root.json are both published (TUF cross-signing).
4. Agents fetch new root.json, verify it is signed by the old root key, and adopt the new targets key.
5. Old targets key is retired.

**Root key rotation (ceremony):**
1. Air-gapped ceremony with witnesses.
2. New root keys generated on air-gapped machine.
3. Old root signs new root.json containing new root keys.
4. New root.json published. Agents verify the chain: old root -> new root.
5. Old root keys are destroyed (or archived in a safe).

**Baked-in key bootstrap:** The agent binary ships with a pinned root.json (the initial root of trust). On first boot, the agent uses this pinned root to bootstrap trust. Subsequent root rotations are verified through the TUF chain. This solves the chicken-and-egg problem: the first binary's pinned root is trusted because the binary itself was distributed through a trusted channel (provisioning). All subsequent updates are verified against the TUF chain rooted in the original root.

### 3.3 Threat Mitigations

| Threat | Mitigation |
|--------|------------|
| **Signing key compromise** | 2-of-3 threshold prevents single-key compromise from being exploitable. An attacker must compromise 2 of 3 independent key holders. Each key is stored on a separate security device (YubiKey, hardware token). |
| **Hub compromise** | Even if the hub is compromised, the attacker cannot push a malicious binary without 2 signing keys. The hub serves TUF metadata and binaries, but the signing keys are not on the hub. The hub is a distribution channel, not a trust anchor. |
| **MITM on update channel** | TUF metadata is signed with Ed25519. Binary hashes are in the signed metadata. TLS protects the transport. An MITM can delay updates (detected by timestamp expiry) but cannot substitute binaries. |
| **Malicious insider** | No single insider can sign a release (2-of-3 threshold). The release process requires two independent approvals. Audit trail records who signed each release. |
| **Rollback attack** | TUF metadata contains monotonically increasing version numbers. The agent rejects any metadata with a version <= its locally stored version. An attacker cannot serve an older (vulnerable) binary. |
| **Freeze attack** | TUF timestamp role has a 24-hour expiry. If the agent receives a timestamp older than 24 hours, it rejects the update channel as potentially compromised and logs a warning. The agent continues running its current version. |
| **Compromised update corrupts rollback** | The rollback binary (agent.prev) is written before the update is applied. If the new binary corrupts the rollback path, the systemd watchdog detects health failure and restarts with agent.prev. MaxRollbackAttempts prevents infinite loops. |

### 3.4 Failure Modes

| Failure | Impact | Behavior | Recovery |
|---------|--------|----------|----------|
| **Hub TUF repository unreachable** | Agent cannot check for updates. | Agent continues running current version. Retries on next check interval. No degradation of operational capability. | Automatic when hub is reachable. |
| **TUF metadata verification fails** | Agent rejects the update. | Agent logs the verification failure with details (which signature failed, which version mismatch). Continues running current version. Reports the failure to the hub (if reachable). | Investigate: was it a legitimate signing error or an attack? Fix and re-sign the release. |
| **Binary hash mismatch after download** | Agent rejects the downloaded binary. | Agent discards the downloaded binary. Retries on next check interval. Reports the mismatch to the hub. | Likely download corruption. Automatic retry resolves transient network issues. If persistent, the TUF repository binary is corrupt -- re-upload and re-sign. |
| **New binary fails health check** | Agent rolls back to previous binary. | Watchdog detects health failure within 5 minutes. Renames agent.prev to agent. Restarts. Reports rollback to hub. | Hub receives rollback notification. If multiple agents roll back, the rollout stage is halted. Investigate the binary issue. |
| **Consecutive rollback loop** | Agent exhausts MaxRollbackAttempts (3). | Agent enters a permanent halt state. Logs `FATAL: max rollback attempts exceeded`. Requires manual intervention. | Operator must manually deploy a known-good binary and reset the RollbackState. |
| **Timestamp expired (freeze attack or clock drift)** | Agent rejects update channel. | Agent logs `WARN: TUF timestamp expired, possible freeze attack`. Continues running current version. Does not accept any updates until a fresh timestamp is received. | If clock drift: fix NTP. If legitimate freeze attack: investigate hub security. |
| **One signing key compromised** | Attacker has 1 of 3 keys. | No immediate impact -- 2-of-3 threshold prevents exploitation. | Rotate the compromised key via root ceremony. Revoke the compromised key. Generate and distribute a replacement. |

### 3.5 Integration Points

| System | Integration |
|--------|-------------|
| **Hub API** | Hub hosts the TUF repository at `/api/v1/updates/tuf/`. The hub's CI pipeline publishes signed releases to this repository. The hub tracks rollout stage progression and agent health reports. |
| **NATS** | Agent health reports sent via `loom.{site_id}.agent.health_report`. Rollout stage advancement commands sent via `loom.hub.agent.{site_id}.update`. Rollback notifications sent via `loom.{site_id}.agent.rollback`. |
| **Temporal** | Rollout orchestration is a Temporal workflow: `AgentRolloutWorkflow`. It manages stage progression, health gate timers, halt/resume, and fleet-wide rollback. Each stage is an activity with a health gate timer side effect. |
| **systemd** | Agent binary managed by systemd unit `loom-agent.service`. Watchdog uses `WatchdogSec` and `sd_notify` for health signaling. |

### 3.6 Implementation Notes

**Libraries:**
- TUF client: `github.com/theupdateframework/go-tuf/v2` (official Go implementation)
- Ed25519: `crypto/ed25519` (Go stdlib)
- SHA-256/SHA-512: `crypto/sha256`, `crypto/sha512` (Go stdlib)
- systemd notification: `github.com/coreos/go-systemd/v22/daemon`

**Signing key storage:**
- Each of the 3 targets signing keys is stored on a YubiKey 5 NFC (Ed25519 key generation on device, private key never exported).
- Root keys are stored on a dedicated air-gapped laptop in a safe. The laptop runs a minimal Linux installation with only the TUF CLI tools.
- The snapshot and timestamp keys are automated (stored as encrypted secrets in the CI pipeline, accessible only to the release job).

---

## 4. Split-Brain Conflict Resolution

### 4.1 Decision

**Priority-based eviction** replaces FIFO eviction for the offline event queue. Critical events (device discovery, state transitions, mutations, errors) are never evicted. Telemetry and heartbeats are evicted first.

**NTP is REQUIRED for edge sites.** Clock skew > 30 seconds triggers a warning; > 5 minutes triggers autonomous action suspension. Timestamp integrity is foundational to the "latest timestamp wins" conflict resolution rule -- without reliable time, that rule produces incorrect results.

**Reconciliation is automatic for non-conflicting changes, requires human approval for competing mutations.** The hub runs a reconciliation workflow after partition healing. Non-conflicting changes (new discoveries, telemetry updates) are merged automatically. Competing mutations (hub issued a teardown while the edge agent observed the device as active) require operator review.

**Autonomous action policy is configurable per-site with 3 levels.** Different sites have different risk profiles. A tier-1 data center may be observe-only, while a remote unmanned site may need emergency autonomy.

**Edge agent maintains a local decision log (SQLite).** Every autonomous action, every state transition, and every conflict is recorded locally for hub review on reconnect. This provides accountability and audit trail for actions taken during partition.

### 4.2 Specification

#### 4.2.1 Priority-Based Offline Queue

```go
package agent

import (
    "time"

    "github.com/google/uuid"
)

// EventPriority defines the eviction priority for offline queue events.
type EventPriority int

const (
    // PriorityP0 events are NEVER evicted from the queue.
    // If P0 events exceed memory, they spill to disk (SQLite).
    // Examples: device discovery, state transitions, mutations, errors,
    //           security events, autonomous action records.
    PriorityP0 EventPriority = 0

    // PriorityP1 events are evicted after all P2 events are evicted.
    // Examples: configuration changes, health status changes,
    //           credential sync requests, non-critical alerts.
    PriorityP1 EventPriority = 1

    // PriorityP2 events are evicted first.
    // Examples: telemetry data points, periodic metrics,
    //           routine heartbeats, discovery scan progress.
    PriorityP2 EventPriority = 2
)

// OfflineQueueConfig configures the priority-based offline event queue.
type OfflineQueueConfig struct {
    // MaxMemoryEvents is the maximum number of events held in memory
    // across all priority levels. Default: 10,000.
    MaxMemoryEvents int `yaml:"max_memory_events" default:"10000"`

    // P0DiskSpillPath is the SQLite database path for P0 event overflow.
    // When P0 events exceed their memory share, they spill to disk.
    // P0 events are NEVER dropped. Default: "/var/lib/loom/p0-events.db"
    P0DiskSpillPath string `yaml:"p0_disk_spill_path" default:"/var/lib/loom/p0-events.db"`

    // MaxP0DiskEvents is the maximum number of P0 events on disk.
    // If exceeded, the oldest P0 disk events are compacted (summarized)
    // into a single digest event, not dropped.
    // Default: 100,000.
    MaxP0DiskEvents int `yaml:"max_p0_disk_events" default:"100000"`

    // MemoryAllocation defines the memory share per priority level.
    // These are soft limits -- P0 can overflow into P1/P2 space.
    MemoryAllocation QueueMemoryAllocation `yaml:"memory_allocation"`
}

// QueueMemoryAllocation defines how the memory budget is split.
type QueueMemoryAllocation struct {
    // P0Percent is the percentage of MaxMemoryEvents reserved for P0.
    // Default: 30% (3,000 of 10,000).
    P0Percent int `yaml:"p0_percent" default:"30"`

    // P1Percent is the percentage reserved for P1.
    // Default: 30% (3,000 of 10,000).
    P1Percent int `yaml:"p1_percent" default:"30"`

    // P2Percent is the percentage reserved for P2.
    // Default: 40% (4,000 of 10,000).
    // P2 has the most space because telemetry is the highest-volume category.
    P2Percent int `yaml:"p2_percent" default:"40"`
}

// QueuedEvent is an event waiting in the offline queue.
type QueuedEvent struct {
    // ID is a unique identifier for this event.
    ID uuid.UUID `json:"id" db:"id"`

    // SequenceNumber is a monotonically increasing per-agent sequence number.
    // Used for ordering during replay.
    SequenceNumber int64 `json:"sequence_number" db:"sequence_number"`

    // Priority determines eviction order.
    Priority EventPriority `json:"priority" db:"priority"`

    // Category classifies the event for conflict detection.
    Category EventCategory `json:"category" db:"category"`

    // Subject is the NATS subject this event targets.
    Subject string `json:"subject" db:"subject"`

    // Payload is the serialized event data.
    Payload []byte `json:"payload" db:"payload"`

    // WallClockTimestamp is the wall clock time when the event was created.
    WallClockTimestamp time.Time `json:"wall_clock_timestamp" db:"wall_clock_timestamp"`

    // MonotonicOffset is the offset in nanoseconds from the agent's
    // monotonic clock start (time.Since(bootTime)). Used by the hub
    // to reconstruct event ordering independent of wall clock drift.
    MonotonicOffset int64 `json:"monotonic_offset" db:"monotonic_offset"`

    // LastNTPSyncAt is the wall clock time of the agent's last successful
    // NTP synchronization. Used by the hub to estimate clock drift.
    LastNTPSyncAt time.Time `json:"last_ntp_sync_at" db:"last_ntp_sync_at"`

    // NTPOffset is the measured offset from NTP at last sync (nanoseconds).
    // Positive means agent clock is ahead of NTP.
    NTPOffset int64 `json:"ntp_offset" db:"ntp_offset"`

    // IsAutonomousAction indicates this event records an autonomous action
    // taken by the edge agent during partition.
    IsAutonomousAction bool `json:"is_autonomous_action" db:"is_autonomous_action"`

    // ResourceID identifies the primary resource this event affects
    // (e.g., device UUID). Used for conflict detection.
    ResourceID *uuid.UUID `json:"resource_id,omitempty" db:"resource_id"`

    // Spilled indicates this event was spilled to disk (P0 overflow).
    Spilled bool `json:"-" db:"spilled"`
}

// EventCategory classifies events for conflict detection and priority assignment.
type EventCategory string

const (
    EventCategoryDiscovery       EventCategory = "discovery"         // P0
    EventCategoryStateTransition EventCategory = "state_transition"  // P0
    EventCategoryMutation        EventCategory = "mutation"          // P0
    EventCategoryError           EventCategory = "error"             // P0
    EventCategorySecurity        EventCategory = "security"          // P0
    EventCategoryAutonomousAction EventCategory = "autonomous_action" // P0
    EventCategoryConfigChange    EventCategory = "config_change"     // P1
    EventCategoryHealthChange    EventCategory = "health_change"     // P1
    EventCategoryAlert           EventCategory = "alert"             // P1
    EventCategoryTelemetry       EventCategory = "telemetry"         // P2
    EventCategoryMetric          EventCategory = "metric"            // P2
    EventCategoryHeartbeat       EventCategory = "heartbeat"         // P2
)

// PriorityForCategory returns the eviction priority for an event category.
func PriorityForCategory(cat EventCategory) EventPriority {
    switch cat {
    case EventCategoryDiscovery, EventCategoryStateTransition,
        EventCategoryMutation, EventCategoryError,
        EventCategorySecurity, EventCategoryAutonomousAction:
        return PriorityP0
    case EventCategoryConfigChange, EventCategoryHealthChange,
        EventCategoryAlert:
        return PriorityP1
    case EventCategoryTelemetry, EventCategoryMetric,
        EventCategoryHeartbeat:
        return PriorityP2
    default:
        return PriorityP1 // conservative default
    }
}
```

**Eviction algorithm:**

```
When queue is full (total events >= MaxMemoryEvents):

1. Count events by priority: n_p0, n_p1, n_p2
2. If n_p2 > 0: evict oldest P2 event. Done.
3. If n_p2 == 0 and n_p1 > 0: evict oldest P1 event.
   Log warning: "P1 events being evicted — high event volume during partition."
4. If n_p2 == 0 and n_p1 == 0: P0 overflow.
   Spill oldest in-memory P0 event to disk (P0DiskSpillPath SQLite).
   Log warning: "P0 events spilling to disk — extended partition with high critical event volume."
5. P0 events are NEVER dropped. If P0 disk exceeds MaxP0DiskEvents:
   Compact: summarize the oldest 1000 P0 events into a single digest event
   that preserves the count, time range, categories, and affected resource IDs
   but discards the full payloads.
   Log warning: "P0 events compacted — extremely extended partition."
```

#### 4.2.2 Clock Skew Handling

```go
// ClockState tracks the agent's clock synchronization status.
type ClockState struct {
    // NTPSynced indicates whether the agent has successfully synced with NTP.
    NTPSynced bool `json:"ntp_synced"`

    // LastNTPSync is the wall clock time of the last successful NTP sync.
    LastNTPSync time.Time `json:"last_ntp_sync"`

    // MeasuredOffset is the offset from NTP at last sync (nanoseconds).
    // Positive: agent clock is ahead. Negative: agent clock is behind.
    MeasuredOffset int64 `json:"measured_offset"`

    // EstimatedDrift is the estimated drift rate (nanoseconds per second)
    // calculated from the last two NTP syncs.
    EstimatedDrift float64 `json:"estimated_drift"`

    // SkewLevel classifies the current clock state.
    SkewLevel ClockSkewLevel `json:"skew_level"`
}

// ClockSkewLevel classifies clock synchronization quality.
type ClockSkewLevel string

const (
    // ClockSkewNormal indicates skew < 30 seconds. Normal operation.
    ClockSkewNormal ClockSkewLevel = "normal"

    // ClockSkewWarning indicates 30s <= skew < 5 minutes.
    // Agent continues operating but logs warnings.
    // Hub is notified via heartbeat.
    ClockSkewWarning ClockSkewLevel = "warning"

    // ClockSkewCritical indicates skew >= 5 minutes.
    // Autonomous actions are suspended.
    // Agent enters observe-only mode regardless of AutonomousActionPolicy.
    // Device mutations are blocked.
    // Timestamp-dependent conflict resolution is unreliable.
    ClockSkewCritical ClockSkewLevel = "critical"
)

// ClockSkewThresholds defines the thresholds for clock skew classification.
var ClockSkewThresholds = struct {
    WarningThreshold  time.Duration
    CriticalThreshold time.Duration
}{
    WarningThreshold:  30 * time.Second,
    CriticalThreshold: 5 * time.Minute,
}
```

**NTP requirement enforcement:**

- Edge agents check NTP synchronization status at startup and periodically (every 5 minutes).
- NTP status is queried via `adjtimex(2)` system call on Linux (checks `TIME_OK` status) or by running `chronyc tracking` and parsing the output.
- If NTP has never synced (fresh boot, no RTC battery on Raspberry Pi), the agent starts in `ClockSkewCritical` state and operates in observe-only mode until NTP syncs.
- Every event in the offline queue carries `LastNTPSyncAt` and `NTPOffset` so the hub can compensate for drift during reconciliation.

#### 4.2.3 Autonomous Action Policy

```go
// AutonomousActionPolicy defines what the edge agent is allowed to do
// during a network partition. Configured per-site at provisioning time.
type AutonomousActionPolicy struct {
    // Level is the autonomy level for this site.
    Level AutonomyLevel `yaml:"level" json:"level"`

    // PreApprovedActions lists specific action types the agent may execute
    // autonomously when Level is PreApproved. Ignored for other levels.
    PreApprovedActions []PreApprovedAction `yaml:"pre_approved_actions,omitempty" json:"pre_approved_actions,omitempty"`

    // EmergencyThresholds defines the conditions under which the agent
    // may take emergency action when Level is Emergency.
    EmergencyThresholds *EmergencyThresholds `yaml:"emergency_thresholds,omitempty" json:"emergency_thresholds,omitempty"`
}

// AutonomyLevel defines the three levels of edge agent autonomy.
type AutonomyLevel string

const (
    // AutonomyObserveOnly: Agent monitors, collects telemetry, runs health
    // checks, and buffers events. It does NOT execute any mutations against
    // devices. It does NOT start/stop/restart any device services.
    // Suitable for: tier-1 data centers with on-site staff, high-compliance
    // environments, sites where human approval is always required.
    AutonomyObserveOnly AutonomyLevel = "observe_only"

    // AutonomyPreApproved: Agent executes a pre-defined list of action types
    // autonomously. The list is configured per-site. Common pre-approved
    // actions: restart-service, clear-error, power-cycle-device.
    // Every pre-approved action is recorded in the decision log.
    // Suitable for: staffed remote sites, environments with well-understood
    // failure modes and documented remediation procedures.
    AutonomyPreApproved AutonomyLevel = "pre_approved"

    // AutonomyEmergency: Agent can take any action from the pre-approved list
    // PLUS emergency actions when emergency thresholds are breached.
    // Emergency actions include: power-cycle unresponsive devices, isolate
    // devices exhibiting anomalous behavior, restart local services.
    // Emergency actions require at least one threshold to be breached.
    // Every action is recorded in the decision log with full justification.
    // Suitable for: unmanned remote sites, edge deployments with limited
    // connectivity, sites where downtime cost exceeds risk of autonomous action.
    AutonomyEmergency AutonomyLevel = "emergency"
)

// PreApprovedAction defines a specific action the agent may take autonomously.
type PreApprovedAction struct {
    // ActionType is the operation type (e.g., "restart_service", "power_cycle",
    // "clear_error", "reset_interface").
    ActionType string `yaml:"action_type" json:"action_type"`

    // MaxExecutionsPerHour limits how often this action can be executed.
    // Prevents action storms. Default: 5.
    MaxExecutionsPerHour int `yaml:"max_executions_per_hour" json:"max_executions_per_hour" default:"5"`

    // RequiresCondition defines the condition that must be true before
    // the action is executed (e.g., "device_health < 20",
    // "service_status == failed").
    RequiresCondition string `yaml:"requires_condition" json:"requires_condition"`

    // CooldownAfterExecution is the minimum time between executions of
    // this action on the same device. Default: 15 minutes.
    CooldownAfterExecution time.Duration `yaml:"cooldown_after_execution" json:"cooldown_after_execution" default:"15m"`
}

// EmergencyThresholds defines when emergency actions are permitted.
type EmergencyThresholds struct {
    // DeviceHealthBelow triggers emergency action when a device's health
    // score falls below this value (0-100). Default: 10.
    DeviceHealthBelow int `yaml:"device_health_below" json:"device_health_below" default:"10"`

    // DeviceUnreachableFor triggers emergency action when a device has been
    // unreachable for this duration. Default: 30 minutes.
    DeviceUnreachableFor time.Duration `yaml:"device_unreachable_for" json:"device_unreachable_for" default:"30m"`

    // AllDevicesUnreachable triggers a site-level emergency when all
    // managed devices are unreachable simultaneously.
    AllDevicesUnreachable bool `yaml:"all_devices_unreachable" json:"all_devices_unreachable" default:"true"`
}
```

#### 4.2.4 Local Decision Log

Every autonomous action, state observation, and conflict-relevant event is recorded in a local SQLite decision log. This log is synced to the hub on reconnect for review.

```go
// DecisionLogEntry records an autonomous action or observation during partition.
type DecisionLogEntry struct {
    // ID is a unique identifier for this log entry.
    ID uuid.UUID `db:"id"`

    // SequenceNumber is a monotonically increasing counter, consistent
    // with the offline queue sequence numbers.
    SequenceNumber int64 `db:"sequence_number"`

    // Timestamp is the wall clock time of the decision.
    Timestamp time.Time `db:"timestamp"`

    // MonotonicOffset is the monotonic clock offset for ordering accuracy.
    MonotonicOffset int64 `db:"monotonic_offset"`

    // EntryType classifies the log entry.
    EntryType DecisionLogEntryType `db:"entry_type"`

    // ActionType is the specific action taken (for action entries).
    // e.g., "power_cycle", "restart_service", "clear_error".
    ActionType string `db:"action_type"`

    // DeviceID is the device affected by this action, if applicable.
    DeviceID *uuid.UUID `db:"device_id"`

    // AutonomyLevel is the autonomy level that permitted this action.
    AutonomyLevel AutonomyLevel `db:"autonomy_level"`

    // Justification is a structured explanation of why the action was taken.
    // Includes: triggering condition, threshold values, device state at time
    // of decision, and the policy rule that authorized the action.
    Justification DecisionJustification `db:"justification"` // stored as JSON

    // Outcome records the result of the action.
    Outcome DecisionOutcome `db:"outcome"` // stored as JSON

    // HubReviewed indicates whether the hub has reviewed this entry.
    // Set to true during reconciliation.
    HubReviewed bool `db:"hub_reviewed"`

    // HubVerdict is the hub's assessment of this action after review.
    // null until reviewed.
    HubVerdict *string `db:"hub_verdict"` // "approved", "flagged", "reverted"
}

// DecisionLogEntryType classifies decision log entries.
type DecisionLogEntryType string

const (
    DecisionLogAction       DecisionLogEntryType = "action"       // autonomous action taken
    DecisionLogObservation  DecisionLogEntryType = "observation"  // state change observed
    DecisionLogSkipped      DecisionLogEntryType = "skipped"      // action considered but not taken (policy denied)
    DecisionLogConflict     DecisionLogEntryType = "conflict"     // conflicting state detected
    DecisionLogDegraded     DecisionLogEntryType = "degraded"     // agent entered degraded mode
)

// DecisionJustification explains why an autonomous action was taken.
type DecisionJustification struct {
    // TriggerCondition is the condition that triggered the action.
    TriggerCondition string `json:"trigger_condition"`

    // ThresholdValue is the threshold that was breached.
    ThresholdValue string `json:"threshold_value"`

    // ObservedValue is the actual observed value.
    ObservedValue string `json:"observed_value"`

    // PolicyRule is the specific policy rule that authorized the action.
    PolicyRule string `json:"policy_rule"`

    // DeviceState is a snapshot of the device's state at decision time.
    DeviceState map[string]interface{} `json:"device_state"`

    // PartitionDuration is how long the agent has been disconnected.
    PartitionDuration time.Duration `json:"partition_duration"`
}

// DecisionOutcome records the result of an autonomous action.
type DecisionOutcome struct {
    // Success indicates whether the action completed without error.
    Success bool `json:"success"`

    // Error is the error message if the action failed.
    Error string `json:"error,omitempty"`

    // DeviceStateAfter is a snapshot of the device's state after the action.
    DeviceStateAfter map[string]interface{} `json:"device_state_after,omitempty"`

    // DurationMs is how long the action took in milliseconds.
    DurationMs int64 `json:"duration_ms"`
}
```

#### 4.2.5 Reconciliation Protocol

After a network partition heals and the NATS leaf node reconnects, the following reconciliation protocol executes:

```
Edge Agent                          Hub
     │                                │
     │ 1. NATS leaf reconnects        │
     │────────────────────────────────►│
     │                                │
     │ 2. Agent sends reconnect       │
     │    metadata:                    │
     │    - partition_start            │
     │    - partition_end              │
     │    - last_ntp_sync_at           │
     │    - ntp_offset                 │
     │    - clock_skew_level           │
     │    - event_count_by_priority    │
     │    - autonomous_action_count    │
     │    - decision_log_count         │
     │────────────────────────────────►│
     │                                │
     │ 3. Hub estimates clock drift:  │
     │    drift = (partition_end -     │
     │    last_ntp_sync) * drift_rate  │
     │    If drift > 5min: flag all    │
     │    timestamps as unreliable     │
     │                                │
     │ 4. Agent replays buffered      │
     │    events in sequence order     │
     │    (P0 first, then P1, then P2) │
     │────────────────────────────────►│
     │                                │
     │ 5. Hub receives events and     │
     │    classifies each:            │
     │                                │
     │    NON-CONFLICTING:            │
     │    - New device discovery      │
     │      (no hub-side mutation)    │
     │    - Telemetry/metrics         │
     │    - Health observations       │
     │    → Auto-merged. No approval. │
     │                                │
     │    CONFLICTING:                │
     │    - Device was decommissioned │
     │      by hub but edge observed  │
     │      it as active              │
     │    - Edge modified device      │
     │      config that hub also      │
     │      modified                  │
     │    - Edge took autonomous      │
     │      action that contradicts   │
     │      a hub workflow            │
     │    → Flagged for human review. │
     │                                │
     │    STALE:                       │
     │    - Edge events with          │
     │      timestamps older than     │
     │      hub's last state change   │
     │      (and clock drift is       │
     │      accounted for)            │
     │    → Logged but not applied.   │
     │                                │
     │ 6. Hub sends decision log      │
     │    review request              │
     │◄────────────────────────────────│
     │                                │
     │ 7. Agent sends decision log    │
     │    entries                     │
     │────────────────────────────────►│
     │                                │
     │ 8. Hub reviews decision log:   │
     │    - Pre-approved actions      │
     │      within policy: auto-      │
     │      approve                   │
     │    - Emergency actions: flag   │
     │      for human review          │
     │    - Policy violations: flag   │
     │      and alert                 │
     │                                │
     │ 9. For conflicting events:     │
     │    Hub creates Reconciliation  │
     │    Workflow (Temporal)          │
     │                                │
     │ 10. Reconciliation Workflow:   │
     │    a. Presents conflicts to    │
     │       operator via approval    │
     │       signal                   │
     │    b. Operator reviews:        │
     │       - Hub state vs edge      │
     │         observation            │
     │       - Decision log entries   │
     │       - Clock drift estimate   │
     │    c. Operator resolves each   │
     │       conflict:                │
     │       - Accept hub state       │
     │       - Accept edge state      │
     │       - Manual merge           │
     │    d. Resolved state is        │
     │       applied                  │
     │    e. Agent notified of        │
     │       resolution               │
     │◄────────────────────────────────│
     │                                │
     │ 11. Agent updates local state  │
     │     to match reconciled state  │
     │                                │
```

#### 4.2.6 Reconciliation Workflow (Temporal)

```go
// ReconciliationWorkflowInput is the input to the hub-side reconciliation workflow.
type ReconciliationWorkflowInput struct {
    // AgentID identifies the reconnecting agent.
    AgentID uuid.UUID `json:"agent_id"`

    // SiteID identifies the agent's site.
    SiteID string `json:"site_id"`

    // PartitionStart is the estimated start of the partition.
    PartitionStart time.Time `json:"partition_start"`

    // PartitionEnd is when the partition healed (reconnect time).
    PartitionEnd time.Time `json:"partition_end"`

    // PartitionDuration is the total partition duration.
    PartitionDuration time.Duration `json:"partition_duration"`

    // ClockDriftEstimate is the estimated clock drift in nanoseconds.
    // Positive: agent clock was ahead. Negative: agent clock was behind.
    ClockDriftEstimate int64 `json:"clock_drift_estimate"`

    // ClockDriftReliable indicates whether the drift estimate is reliable.
    // False if the agent's last NTP sync was more than 24 hours before
    // the partition started.
    ClockDriftReliable bool `json:"clock_drift_reliable"`

    // Conflicts lists the detected conflicts requiring resolution.
    Conflicts []StateConflict `json:"conflicts"`

    // DecisionLogSummary summarizes the autonomous actions taken.
    DecisionLogSummary DecisionLogSummary `json:"decision_log_summary"`
}

// StateConflict represents a single conflict between hub state and edge state.
type StateConflict struct {
    // ID uniquely identifies this conflict.
    ID uuid.UUID `json:"id"`

    // ResourceType is the type of resource in conflict (e.g., "device", "config").
    ResourceType string `json:"resource_type"`

    // ResourceID is the specific resource identifier.
    ResourceID uuid.UUID `json:"resource_id"`

    // HubState is the hub's view of the resource at partition end.
    HubState ConflictState `json:"hub_state"`

    // EdgeState is the edge agent's view of the resource at partition end.
    EdgeState ConflictState `json:"edge_state"`

    // ConflictType classifies the conflict.
    ConflictType ConflictType `json:"conflict_type"`

    // AutoResolvable indicates whether this conflict can be resolved
    // without human input. True for non-competing changes (e.g.,
    // hub added metadata field A, edge added metadata field B).
    AutoResolvable bool `json:"auto_resolvable"`

    // SuggestedResolution is the system's suggested resolution.
    SuggestedResolution string `json:"suggested_resolution"`
}

// ConflictState captures one side's view of a resource.
type ConflictState struct {
    // State is the resource state (e.g., "active", "decommissioned").
    State string `json:"state"`

    // Timestamp is when this state was set (adjusted for clock drift).
    Timestamp time.Time `json:"timestamp"`

    // Source identifies who set this state (e.g., "hub_workflow:wf-123",
    // "edge_agent:agent-456", "edge_autonomous:power_cycle").
    Source string `json:"source"`

    // Properties are the key-value properties of the resource at this state.
    Properties map[string]interface{} `json:"properties"`
}

// ConflictType classifies the nature of a conflict.
type ConflictType string

const (
    // ConflictCompetingMutation: both hub and edge modified the same resource.
    // Requires human approval.
    ConflictCompetingMutation ConflictType = "competing_mutation"

    // ConflictStateDesyncLifecycle: hub and edge disagree on lifecycle state
    // (e.g., hub says decommissioned, edge says active).
    // Requires human approval.
    ConflictStateDesyncLifecycle ConflictType = "lifecycle_desync"

    // ConflictStaleHubAssumption: hub made a decision based on stale state
    // that the edge has since updated. May be auto-resolvable.
    ConflictStaleHubAssumption ConflictType = "stale_hub_assumption"

    // ConflictMetadataMerge: hub and edge both enriched metadata for the
    // same resource with non-overlapping fields. Auto-resolvable.
    ConflictMetadataMerge ConflictType = "metadata_merge"
)

// DecisionLogSummary provides an overview of autonomous actions for the operator.
type DecisionLogSummary struct {
    // TotalActions is the number of autonomous actions taken.
    TotalActions int `json:"total_actions"`

    // ActionsByType groups actions by type.
    ActionsByType map[string]int `json:"actions_by_type"`

    // SuccessRate is the percentage of actions that succeeded.
    SuccessRate float64 `json:"success_rate"`

    // PolicyViolations is the number of actions that violated policy
    // (should be zero in normal operation).
    PolicyViolations int `json:"policy_violations"`

    // HighestAutonomyUsed is the highest autonomy level exercised.
    HighestAutonomyUsed AutonomyLevel `json:"highest_autonomy_used"`
}
```

#### 4.2.7 Conflict Resolution Rules

The reconciliation workflow applies these deterministic rules before escalating to a human:

| Conflict Type | Rule | Auto-Resolve? |
|---------------|------|---------------|
| **Metadata merge** (non-overlapping fields) | Union of hub and edge metadata. For each field, take the value from whichever side set it. | Yes |
| **Metadata merge** (overlapping fields, same value) | No conflict -- both sides agree. | Yes |
| **Metadata merge** (overlapping fields, different values) | If clock drift is reliable: latest timestamp wins (adjusted for drift). If clock drift unreliable: flag for human review. | Conditional |
| **Stale hub assumption** | If the edge's observation is more recent (accounting for drift) and the hub's decision was based on the stale state: present both to the operator with a recommendation to accept edge state. | No (requires confirmation) |
| **Lifecycle desync** | Always requires human approval. Present: hub's lifecycle action (with workflow ID), edge's observation (with evidence), and the decision log entries during the relevant time window. | No |
| **Competing mutation** | Always requires human approval. Present: both mutations side-by-side, timestamps (adjusted for drift), and affected downstream state. | No |

#### 4.2.8 State Machine: Agent Partition Mode

```
                    ┌───────────────────────────┐
                    │          ONLINE            │
                    │  Connected to hub          │
                    │  Full operation            │
                    │  Events sent in real time  │
                    └──────────┬────────────────┘
                               │
                    Hub connection lost
                    (NATS leaf disconnect)
                               │
                    ┌──────────▼────────────────┐
                    │      PARTITION_DETECTED    │
                    │  Assess autonomy level     │
                    │  Check clock skew          │
                    │  Start buffering events    │
                    └──────────┬────────────────┘
                               │
              ┌────────────────┼───────────────────┐
              │                │                   │
        observe_only     pre_approved         emergency
              │                │                   │
    ┌─────────▼──────┐  ┌──────▼────────┐  ┌──────▼──────────┐
    │  OBSERVE_ONLY  │  │ PRE_APPROVED  │  │   EMERGENCY     │
    │  Monitor only  │  │ Execute pre-  │  │  Full pre-      │
    │  No mutations  │  │ approved acts │  │  approved +     │
    │  Buffer events │  │ Log decisions │  │  emergency acts  │
    │                │  │ Buffer events │  │  Log decisions   │
    │                │  │               │  │  Buffer events   │
    └────────┬───────┘  └──────┬────────┘  └──────┬──────────┘
             │                 │                   │
             │    Clock skew > 5 minutes           │
             │    ┌────────────┼───────────────┐   │
             │    │            │               │   │
             │    ▼            ▼               ▼   │
             │  ┌──────────────────────────────┐   │
             ├──│    CLOCK_SKEW_SUSPENDED      │───┤
             │  │  Autonomous actions halted   │   │
             │  │  Observe-only regardless of  │   │
             │  │  configured policy           │   │
             │  │  Buffer events               │   │
             │  │  Log clock skew event (P0)   │   │
             │  └──────────────────────────────┘   │
             │                                     │
             │    Credential cache expired          │
             │    ┌────────────────────────────┐   │
             │    │                            │   │
             ▼    ▼                            ▼   │
           ┌──────────────────────────────────────┐│
           │         DEGRADED                      ││
           │  Credentials wiped                    ││
           │  Cannot authenticate to devices       ││
           │  Passive monitoring only (ICMP/ARP)   ││
           │  Buffer events                        ││
           └──────────┬───────────────────────────┘│
                      │                             │
                      │    Hub reconnects           │
                      │    ┌────────────────────┐   │
                      │    │                    │   │
                      ▼    ▼                    ▼   ▼
                    ┌───────────────────────────────┐
                    │       RECONCILING              │
                    │  Send reconnect metadata       │
                    │  Replay buffered events        │
                    │  Send decision log             │
                    │  Receive reconciliation result │
                    │  Re-sync credentials           │
                    │  Update local state            │
                    └──────────┬────────────────────┘
                               │
                    Reconciliation complete
                               │
                    ┌──────────▼────────────────┐
                    │          ONLINE            │
                    └───────────────────────────┘
```

### 4.3 Threat Mitigations

| Threat | Mitigation |
|--------|------------|
| **Critical event eviction** (Scenario 3 from research) | Priority-based eviction ensures P0 events (discoveries, state transitions, mutations) are never dropped. P0 events spill to disk when memory is full. Only P2 telemetry/heartbeats are evicted first, then P1 health/config changes. |
| **Clock skew corruption** (Scenario 2 from research) | NTP is required. Every event carries monotonic offset and last NTP sync time. Hub estimates drift and adjusts timestamps during reconciliation. Clock skew > 5 minutes suspends autonomous actions. |
| **Competing mutations** (Scenario 1 from research) | Hub detects competing mutations during reconciliation. These are never auto-resolved -- they require human review. The reconciliation workflow presents both sides (hub workflow vs. edge observation) with full context. |
| **State machine divergence** (Scenario 4 from research) | Edge agent replays all state transitions in sequence order during reconciliation. Hub replays edge events against hub state to identify divergence points. The reconciliation workflow presents the full timeline for operator review. |
| **Autonomous action abuse** | Three-level autonomy policy limits what agents can do. Rate limits per action type prevent action storms. Decision log provides full audit trail. Hub reviews every autonomous action on reconnect. |
| **Partition abuse by attacker** | An attacker who partitions an edge site intentionally to exploit autonomous actions is limited by the autonomy policy. observe_only sites take no actions. pre_approved sites only execute whitelisted actions with rate limits. emergency sites require threshold breach. All actions are logged and reviewed. |

### 4.4 Failure Modes

| Failure | Impact | Behavior | Recovery |
|---------|--------|----------|----------|
| **P0 disk spill SQLite full** | Cannot store more P0 events on disk. | Compaction activates: oldest 1000 P0 events are summarized into a digest. Space is reclaimed. | Automatic via compaction. Operator should investigate why so many critical events are accumulating (possible event storm). |
| **NTP unreachable during partition** | Clock drift accumulates. | Agent tracks time since last NTP sync. If drift exceeds 5 minutes (estimated), autonomous actions are suspended. All events are tagged with "clock drift unreliable". | NTP restoration fixes the clock. Hub uses monotonic offsets for event ordering during reconciliation. |
| **Reconciliation workflow times out** | Conflicting state remains unresolved. | Reconciliation workflow has a 72-hour timeout (configurable). If no human resolves conflicts within this window, the workflow auto-resolves using the conservative rule: hub state wins for all conflicts. Edge observations are preserved as historical records but do not override hub state. | Operator reviews the auto-resolved conflicts retrospectively. |
| **Decision log SQLite corrupted** | Autonomous action audit trail lost. | Agent creates a new decision log database. Logs the corruption event. The lost entries are flagged in the reconciliation workflow as "audit gap". | Hub flags the gap for security review. The actions themselves may have produced observable side effects that the hub can detect during reconciliation (e.g., device state changes). |
| **Agent takes action beyond its policy** | Policy violation. | This is a software bug. The policy engine should prevent this. If it occurs: the decision log records the action as a policy violation. The hub flags it during reconciliation with elevated severity. | Investigate the policy engine bug. Assess whether the unauthorized action caused harm. The hub can revert the action if needed. |
| **Hub issues workflow during partition targeting edge devices** | Workflow cannot reach edge devices. | The Temporal workflow for the edge devices will fail at the adapter activity (device unreachable). The workflow enters retry/compensation depending on configuration. When partition heals, the workflow can be re-submitted. The reconciliation workflow detects the stale hub assumption and presents it to the operator. | Operator decides whether to re-submit the workflow or accept the edge state. |

### 4.5 Integration Points

| System | Integration |
|--------|-------------|
| **NATS** (DEPLOYMENT-TOPOLOGY.md) | Offline queue replaces NATS publishing during partition. On reconnect, buffered events are replayed through the NATS leaf node. Reconnect metadata sent via `loom.{site_id}.agent.reconnected`. |
| **Temporal** (WORKFLOW-CONTRACT.md) | Reconciliation is a Temporal workflow (`ReconciliationWorkflow`). Conflict resolution uses Temporal's signal-based human approval pattern (not polling). The workflow has a configurable timeout (default 72h). |
| **SQLite** (DEPLOYMENT-TOPOLOGY.md) | P0 disk spill and decision log are stored in separate SQLite databases. The decision log database is not wiped during credential cache wipe -- it is an audit trail that must survive. |
| **NATS Subject Mapping** (DEPLOYMENT-TOPOLOGY.md) | Decision log sync uses `loom.{site_id}.agent.decision_log`. Reconciliation results received via `loom.hub.commands.{site_id}.reconciliation_result`. |
| **FAILURE-MODES.md Section 3.5** | The partition behavior described here supersedes the simpler description in FAILURE-MODES.md 3.5. That section should reference this specification for the complete protocol. |

### 4.6 Implementation Notes

**Libraries:**
- SQLite: `modernc.org/sqlite` (pure Go, consistent with agent architecture)
- NTP status: `golang.org/x/sys/unix` for `adjtimex(2)` syscall on Linux. On non-Linux platforms, shell out to `chronyc tracking` or `ntpq -p`.
- Monotonic clock: `time.Since()` provides monotonic durations in Go stdlib.
- UUID generation: `github.com/google/uuid` v4.

**Decision log storage:**
- Stored at `/var/lib/loom/decision-log.db`.
- Separate from the credential cache (`/var/lib/loom/credential-cache.db`).
- Separate from the agent operational state (`/var/lib/loom/agent.db`).
- Retained for 90 days after hub review (configurable).
- Each entry is ~1-2 KB. At 100 actions/day for 7 days, this is ~700 KB -- negligible.

**P0 disk spill storage:**
- Stored at `/var/lib/loom/p0-events.db`.
- Separate database to avoid impacting agent operational state during high-volume event spill.
- Each event is ~0.5-5 KB. 100,000 events = 50-500 MB. Acceptable for edge devices with >1 GB free disk space (agent should monitor disk space and alert if <500 MB free).

---

## 5. Cross-Cutting Concerns

### 5.1 Audit Trail Integration

All three subsystems produce audit events that integrate with LOOM's audit model:

| Event | Subject | Priority | Source |
|-------|---------|----------|--------|
| `agent.credential_cache.synced` | `loom.{site_id}.security` | P1 | Credential cache |
| `agent.credential_cache.expired` | `loom.{site_id}.security` | P0 | Credential cache |
| `agent.credential_cache.wiped` | `loom.{site_id}.security` | P0 | Credential cache |
| `agent.credential_cache.tamper_detected` | `loom.{site_id}.security` | P0 | Credential cache |
| `agent.update.applied` | `loom.{site_id}.agent` | P0 | Auto-update |
| `agent.update.rollback` | `loom.{site_id}.agent` | P0 | Auto-update |
| `agent.update.health_passed` | `loom.{site_id}.agent` | P1 | Auto-update |
| `agent.partition.detected` | `loom.{site_id}.agent` | P0 | Split-brain |
| `agent.partition.healed` | `loom.{site_id}.agent` | P0 | Split-brain |
| `agent.autonomous_action` | `loom.{site_id}.agent` | P0 | Split-brain |
| `agent.reconciliation.started` | `loom.{site_id}.agent` | P0 | Split-brain |
| `agent.reconciliation.completed` | `loom.{site_id}.agent` | P0 | Split-brain |
| `agent.clock_skew.warning` | `loom.{site_id}.agent` | P1 | Split-brain |
| `agent.clock_skew.critical` | `loom.{site_id}.agent` | P0 | Split-brain |

### 5.2 Metrics

All three subsystems expose Prometheus metrics at the agent's metrics endpoint:

```
# Credential cache
loom_agent_credential_cache_size{site_id}                    gauge
loom_agent_credential_cache_oldest_seconds{site_id}          gauge
loom_agent_credential_cache_state{site_id,state}             gauge  # 0/1 per state
loom_agent_credential_kek_provider{site_id,provider}         gauge  # 0/1 per provider

# Auto-update
loom_agent_version_info{site_id,version,rollout_group}       gauge  # always 1
loom_agent_update_last_check_timestamp{site_id}              gauge
loom_agent_update_rollbacks_total{site_id}                   counter
loom_agent_update_health_check_duration_seconds{site_id}     histogram

# Split-brain
loom_agent_offline_queue_size{site_id,priority}              gauge
loom_agent_offline_queue_evictions_total{site_id,priority}   counter
loom_agent_offline_queue_p0_disk_spill_size{site_id}         gauge
loom_agent_partition_duration_seconds{site_id}               gauge  # 0 when online
loom_agent_autonomous_actions_total{site_id,action_type}     counter
loom_agent_clock_skew_seconds{site_id}                       gauge
loom_agent_decision_log_size{site_id}                        gauge
```

### 5.3 Agent Configuration Extension

The existing `AgentConfig` (DEPLOYMENT-TOPOLOGY.md Section 3) is extended with the fields defined in this specification:

```go
// AgentSecurityConfig extends AgentConfig with security-specific fields.
// These fields are added to the existing AgentConfig struct.
type AgentSecurityConfig struct {
    // CredentialCache configures the encrypted credential cache.
    CredentialCache CredentialCacheConfig `yaml:"credential_cache"`

    // KEKProvider configures the KEK source (TPM or passphrase).
    KEKProvider AgentKEKProviderConfig `yaml:"kek_provider"`

    // AutoUpdate configures the TUF-based auto-update system.
    AutoUpdate TUFConfig `yaml:"auto_update"`

    // AutonomousPolicy configures the autonomous action policy.
    AutonomousPolicy AutonomousActionPolicy `yaml:"autonomous_policy"`

    // OfflineQueue configures the priority-based offline event queue.
    OfflineQueue OfflineQueueConfig `yaml:"offline_queue"`

    // DecisionLogPath is the path to the decision log SQLite database.
    DecisionLogPath string `yaml:"decision_log_path" default:"/var/lib/loom/decision-log.db"`

    // ClockSkewCheckInterval is how often to check NTP synchronization.
    ClockSkewCheckInterval time.Duration `yaml:"clock_skew_check_interval" default:"5m"`
}

// AgentKEKProviderConfig selects and configures the KEK provider.
type AgentKEKProviderConfig struct {
    // Type selects the KEK provider: "tpm2" or "passphrase".
    Type AgentKEKProviderType `yaml:"type" default:"passphrase"`

    // TPM configures the TPM 2.0 KEK provider (used when Type is "tpm2").
    TPM *TPMKEKProviderConfig `yaml:"tpm,omitempty"`

    // Passphrase configures the passphrase KEK provider (used when Type is "passphrase").
    Passphrase *PassphraseKEKProviderConfig `yaml:"passphrase,omitempty"`
}

type TPMKEKProviderConfig struct {
    DevicePath    string `yaml:"device_path" default:"/dev/tpmrm0"`
    SealedKeyPath string `yaml:"sealed_key_path" default:"/var/lib/loom/tpm-sealed-kek.blob"`
    PCRSelection  []int  `yaml:"pcr_selection" default:"[0,2,4,7]"`
}

type PassphraseKEKProviderConfig struct {
    SaltPath string `yaml:"salt_path" default:"/var/lib/loom/kek-salt.bin"`
    // Passphrase is delivered via systemd LoadCredential, not config file.
}
```

---

## 6. Implementation Priority

This specification updates the phasing from RESEARCH-edge-agent-security.md based on the design decisions made:

| Item | Phase | Effort | Dependencies | Notes |
|------|-------|--------|--------------|-------|
| Agent KEK (passphrase-derived) + encrypted SQLite credential cache | 1A | 2 weeks | Vault architecture finalized | Core security requirement. Must ship before any production edge deployment. |
| Per-site credential scoping + credential manifest | 1A | 1 week | Agent provisioning flow, internal CA | Ensures blast radius is bounded to one site. |
| Priority-based offline queue (replace FIFO) | 1A | 1 week | NATS event model finalized | Prevents critical event loss during partition. |
| Clock skew detection + autonomous action suspension | 1A | 3 days | NTP infrastructure at edge sites | Simple implementation with high safety impact. |
| Local decision log | 1B | 1 week | SQLite agent architecture | Audit trail for autonomous actions. |
| Autonomous action policy engine | 1B | 2 weeks | Decision log, pre-approved action list | Three-level policy with rate limiting. |
| Reconciliation protocol (hub-side) | 1B | 2-3 weeks | Temporal workflow patterns, conflict detection | Core post-partition recovery mechanism. |
| TUF client integration in agent | 1B | 2-3 weeks | CI pipeline, TUF repository on hub | Threshold signing + metadata verification. |
| Staged rollout orchestration | 1B | 1-2 weeks | TUF integration, health reporting | Temporal workflow for rollout progression. |
| Rollback watchdog (systemd) | 1B | 3 days | TUF integration | ExecStartPre health check + binary swap. |
| TPM 2.0 KEK provider | 2 | 2 weeks | TPM hardware availability for testing | Hardware-bound key enhancement. |
| Hub-initiated credential revocation + rotation on decommission | 2 | 1 week | Credential manifest, internal CA CRL | Automated blast radius containment. |
| Air-gapped TUF repository + USB import | 2 | 1 week | TUF integration | Mode 4 deployment support. |
| Root key ceremony tooling + documentation | 2 | 1 week | TUF integration | Operational procedure, not code. |

---

## 7. References

- [The Update Framework (TUF)](https://theupdateframework.io/) -- CNCF graduated project
- [Uptane](https://uptane.github.io/) -- TUF for automotive OTA updates
- [go-tuf v2](https://github.com/theupdateframework/go-tuf) -- Official Go TUF implementation
- [go-tpm](https://github.com/google/go-tpm) -- Google's Go TPM 2.0 library
- DEPLOYMENT-TOPOLOGY.md Section 3: Edge Agent Architecture
- VAULT-ARCHITECTURE.md: Encryption Architecture, Key Hierarchy
- SECURITY-MODEL.md Section 2: Cryptographic Primitives
- FAILURE-MODES.md Section 3.5: Leaf Node Disconnected from Hub
- EDGE-CASES.md: RC-04 (Shared Device), RC-05 (Out-of-Order Events), RC-06 (Duplicate Discovery)
- GAP-ANALYSIS.md: C1, C4, C5
