# LOOM Credential Vault Architecture

> **This is the most security-critical component in LOOM.** The vault holds credentials for every piece of managed infrastructure: IPMI passwords, SSH keys, SNMP community strings, Redfish tokens, cloud API keys, network device credentials, vSphere sessions. If this vault is compromised, the entire infrastructure estate is compromised. Every design decision in this document prioritizes defense over convenience.

---

## 1. Design Philosophy

### Core Principles

1. **Defense in depth.** Multiple independent layers of protection. No single compromised layer grants access to plaintext credentials. Compromise of the database yields ciphertext. Compromise of the application yields encrypted blobs without master key access. Compromise of a single tenant's KEK yields nothing from other tenants.

2. **Zero plaintext at rest — ever, anywhere.** Credentials never exist as plaintext in:
   - Database columns or rows
   - Filesystem (no temp files, no config files, no log files)
   - Swap space (memory-locked pages)
   - Core dumps (disabled for the vault process)
   - Log output (masked as `***REDACTED***`)
   - Error messages or stack traces
   - JSON serialization output
   - Debug/pprof endpoints

3. **Hardware-accelerated cryptography as the default path.** AES-256-GCM via AES-NI instruction set on all modern Intel/AMD x86-64 processors. BLAKE3 via AVX-512 where available. OS CSPRNG backed by RDRAND/RDSEED hardware random number generators. Software fallback exists but triggers an audit warning.

4. **Memory protection as a first-class concern.** Credential bytes live on `mlock`'d pages that cannot be swapped to disk. Credentials are zeroed immediately after use via constant-time wipe functions that the compiler cannot optimize away. Finalizers provide a safety net but are not the primary mechanism.

5. **Comparable to Apple's Data Protection architecture** — but designed for multi-tenant server infrastructure rather than single-user mobile devices. Apple's Secure Enclave maps to LOOM's HSM/TPM integration. Apple's per-file keys map to per-credential DEKs. Apple's class keys map to per-tenant KEKs. LOOM goes beyond Apple in multi-tenant isolation, automatic credential rotation, protocol-aware lifecycle management, and full audit trail on every access.

### Invariants (enforced in code, not policy)

- A credential's plaintext MUST NOT exist in more than one memory location simultaneously.
- A credential's plaintext MUST NOT persist in memory longer than its `MaxLifetime` TTL.
- A credential's plaintext MUST NOT appear in any output (logs, errors, serialization, traces).
- Every credential retrieval MUST produce an immutable audit record before the plaintext is returned.
- Cross-tenant credential access MUST be impossible — enforced at the type level, not just authorization checks.

---

## 2. Encryption Architecture

### Envelope Encryption (3-Tier Key Hierarchy)

```
Master Key (MEK)
  └── Key Encryption Keys (KEK) — one per tenant
        └── Data Encryption Keys (DEK) — one per credential
```

This is the same pattern used by AWS KMS, Google Cloud KMS, and Apple's Data Protection. The critical property: no single key compromise reveals all credentials. The MEK protects KEKs. Each KEK protects only its tenant's DEKs. Each DEK protects only one credential.

### Master Key (MEK)

The MEK is the root of trust. It never exists as plaintext on disk. It is used exclusively to wrap and unwrap KEKs. The MEK source is selected at deployment time, in order of preference:

| Priority | Source | Security Level | Use Case |
|----------|--------|---------------|----------|
| 1 | Hardware Security Module (HSM) via PKCS#11 | Highest | Enterprise on-prem, regulated environments |
| 2 | HashiCorp Vault Transit engine | High | Organizations already running Vault |
| 3 | Cloud KMS (AWS KMS / Azure Key Vault / GCP Cloud KMS) | High | Cloud-native deployments |
| 4 | TPM 2.0 sealed storage | Medium-High | Single-server deployments with TPM hardware |
| 5 | Local file-based key (Argon2id-derived from passphrase) | Medium | Air-gapped environments, development only |

```go
// MasterKeyProvider abstracts the source of the master key.
// The MEK never leaves the provider in the HSM and Cloud KMS cases —
// wrap/unwrap operations are performed by the provider itself.
type MasterKeyProvider interface {
    // WrapKEK encrypts a KEK using the master key.
    // For HSM/KMS providers, the plaintext KEK is sent to the provider,
    // encrypted there, and the ciphertext returned.
    WrapKEK(ctx context.Context, plaintextKEK []byte) (ciphertextKEK []byte, err error)

    // UnwrapKEK decrypts a KEK using the master key.
    UnwrapKEK(ctx context.Context, ciphertextKEK []byte) (plaintextKEK []byte, err error)

    // KeyID returns the identifier of the current master key (for key rotation tracking).
    KeyID() string

    // Provider returns the provider type for audit logging.
    Provider() MasterKeyProviderType
}

type MasterKeyProviderType string

const (
    MasterKeyProviderHSM      MasterKeyProviderType = "hsm_pkcs11"
    MasterKeyProviderVault    MasterKeyProviderType = "hashicorp_vault"
    MasterKeyProviderAWSKMS   MasterKeyProviderType = "aws_kms"
    MasterKeyProviderAzureKV  MasterKeyProviderType = "azure_keyvault"
    MasterKeyProviderGCPKMS   MasterKeyProviderType = "gcp_kms"
    MasterKeyProviderTPM      MasterKeyProviderType = "tpm2"
    MasterKeyProviderLocal    MasterKeyProviderType = "local_passphrase"
)
```

**HSM Integration (PKCS#11):** Supported devices include Thales Luna, Entrust nShield, YubiHSM 2, and AWS CloudHSM. The MEK is generated inside the HSM and never exported. All KEK wrap/unwrap operations execute inside the HSM boundary. Go library: `github.com/miekg/pkcs11` with LOOM's own wrapper for connection pooling and health checks.

**HashiCorp Vault Transit:** The Transit engine performs encrypt/decrypt operations using a key that Vault manages. LOOM never sees the transit key. Go library: `github.com/hashicorp/vault/api/v2`.

**Cloud KMS:** Each cloud provider's KMS performs the wrap/unwrap. The master key material never leaves the KMS boundary. Go libraries: `github.com/aws/aws-sdk-go-v2/service/kms`, `github.com/Azure/azure-sdk-for-go/sdk/security/keyvault/azkeys`, `cloud.google.com/go/kms/apiv1`.

**TPM 2.0:** The MEK is sealed to the TPM's Storage Root Key (SRK). Unsealing requires the TPM to be in the expected PCR state (measured boot). Go library: `github.com/google/go-tpm/tpm2`.

<!-- AUDIT-FIX: C-01 — Passphrase-derived MEK hardening -->

**Local passphrase (development only — NOT for production):** The MEK is derived from a passphrase using Argon2id with the following parameters: `time=3, memory=256MB, threads=4, keyLen=32`. The derived key is held in mlock'd memory and never written to disk. The salt is stored alongside the encrypted KEK table.

> **WARNING: `local_passphrase` is development-only. Production deployments MUST use HSM, Vault, or Cloud KMS.** The passphrase-derived MEK exists in process memory for the lifetime of the vault process. Any memory dump, core dump, or swap leak exposes the MEK and, transitively, every credential in the system. HSM/KMS providers avoid this entirely because the MEK never enters the LOOM process.

**Startup gate:** The `local_passphrase` provider **refuses to start** unless the environment variable `LOOM_ALLOW_INSECURE_MEK=true` is explicitly set. This prevents accidental production deployments with a passphrase-derived MEK.

**Startup logging:** When `local_passphrase` is active, LOOM logs a **CRITICAL** alert on every startup:

```
CRITICAL: Vault MEK provider is local_passphrase. The master encryption key exists in process memory.
          This configuration is for DEVELOPMENT AND TESTING ONLY.
          Production deployments MUST use HSM (hsm_pkcs11), HashiCorp Vault (hashicorp_vault), or Cloud KMS.
          Set LOOM_ALLOW_INSECURE_MEK=true to acknowledge this risk (already set).
```

This message is logged at `slog.LevelError` (not `slog.LevelWarn`) and is emitted to the audit trail as `vault.insecure_mek_startup`.

```go
// PassphraseMEKConfig configures the local passphrase MEK provider.
// This provider is for development and testing only.
type PassphraseMEKConfig struct {
    Salt              []byte `json:"salt"`
    Time              uint32 `json:"time"`    // Argon2id time (default: 3)
    Memory            uint32 `json:"memory"`  // Argon2id memory in KiB (default: 262144 = 256MB)
    Threads           uint8  `json:"threads"` // Argon2id parallelism (default: 4)
    NonProductionOnly bool   `json:"non_production_only"` // MUST be true; startup fails if false
}
```

`NonProductionOnly` defaults to `true` and **cannot be set to `false`**. It exists as a machine-readable marker so that deployment validation tools, CI/CD gates, and compliance scanners can programmatically reject any configuration that uses `local_passphrase` in a production context.

<!-- END AUDIT-FIX: C-01 -->

### Key Encryption Key (KEK) — Per Tenant

- One AES-256 key per tenant.
- Generated using `crypto/rand` (OS CSPRNG).
- Encrypted (wrapped) by the MEK and stored in the `tenant_keks` table.
- Rotated quarterly by default (configurable per tenant).
- On rotation: a new KEK is generated, all DEKs for the tenant are re-encrypted under the new KEK, old KEK is retained (marked `retired`) until all DEKs have been re-wrapped, then securely deleted.
- KEK version is tracked — each encrypted DEK references which KEK version encrypted it.

```go
// KEKRecord is stored in the tenant_keks table.
type KEKRecord struct {
    ID             uuid.UUID             `db:"id"`
    TenantID       uuid.UUID             `db:"tenant_id"`
    Version        int                   `db:"version"`          // monotonically increasing
    EncryptedKey   []byte                `db:"encrypted_key"`    // AES-256 key, wrapped by MEK
    MasterKeyID    string                `db:"master_key_id"`    // which MEK version wrapped this
    Status         KEKStatus             `db:"status"`           // active, rotating, retired
    CreatedAt      time.Time             `db:"created_at"`
    RotatedAt      *time.Time            `db:"rotated_at"`
    ExpiresAt      time.Time             `db:"expires_at"`
}

type KEKStatus string

const (
    KEKStatusActive   KEKStatus = "active"
    KEKStatusRotating KEKStatus = "rotating"
    KEKStatusRetired  KEKStatus = "retired"
)
```

### Data Encryption Key (DEK) — Per Credential

- One AES-256-GCM key per credential.
- Generated using `crypto/rand`.
- Encrypted (wrapped) by the tenant's active KEK.
- Stored alongside the encrypted credential blob.
- Rotated whenever the credential itself is rotated.
- The DEK never appears in logs, errors, or any output.

### Algorithms — Concrete Choices

| Purpose | Algorithm | Implementation | Hardware Acceleration |
|---------|-----------|---------------|----------------------|
| Symmetric encryption | AES-256-GCM | `crypto/aes` + `crypto/cipher` (Go stdlib) | AES-NI on x86-64 (automatic) |
| Key derivation (passphrase) | Argon2id | `golang.org/x/crypto/argon2` | Memory-hard, resistant to GPU/ASIC |
| Key exchange | X25519 | `golang.org/x/crypto/curve25519` | Constant-time implementation |
| Digital signatures | Ed25519 | `crypto/ed25519` (Go stdlib) | Used for audit record signing |
| Integrity hashing | BLAKE3 | `github.com/zeebo/blake3` | AVX-512, AVX2, SSE4.1 auto-detected |
| CSPRNG | `crypto/rand` | OS CSPRNG (`/dev/urandom` on Linux) | RDRAND/RDSEED on supported CPUs |
| Authenticated encryption nonce | 96-bit random | `crypto/rand` | Per AES-256-GCM spec |

### Hardware Instruction Set Detection

Go's `crypto/aes` package automatically uses AES-NI instructions on x86-64 processors. No code changes are needed — the Go runtime detects AES-NI at startup. However, LOOM explicitly verifies and logs which instruction sets are active for audit and compliance purposes.

```go
package vault

import (
    "log/slog"
    "runtime"

    "golang.org/x/sys/cpu"
)

// CryptoCapabilities reports what hardware acceleration is available.
type CryptoCapabilities struct {
    AESNI    bool `json:"aes_ni"`
    AVX2     bool `json:"avx2"`
    AVX512   bool `json:"avx512"`
    RDRAND   bool `json:"rdrand"`
    RDSEED   bool `json:"rdseed"`
    Platform string `json:"platform"`
}

// DetectCryptoCapabilities checks CPU features and logs results at startup.
// This MUST be called during vault initialization and the result recorded
// in the startup audit log entry.
func DetectCryptoCapabilities(logger *slog.Logger) CryptoCapabilities {
    caps := CryptoCapabilities{
        Platform: runtime.GOARCH,
    }

    if runtime.GOARCH == "amd64" {
        caps.AESNI = cpu.X86.HasAES
        caps.AVX2 = cpu.X86.HasAVX2
        caps.AVX512 = cpu.X86.HasAVX512F && cpu.X86.HasAVX512BW
        // RDRAND/RDSEED are used by the OS CSPRNG, not directly by Go,
        // but we report their presence for the audit record.
        // Note: cpu.X86 does not expose RDRAND directly in all Go versions;
        // presence is inferred from kernel usage of /dev/urandom.
    }

    logger.Info("crypto hardware capabilities detected",
        "aes_ni", caps.AESNI,
        "avx2", caps.AVX2,
        "avx512", caps.AVX512,
        "platform", caps.Platform,
    )

    if !caps.AESNI {
        logger.Warn("AES-NI not detected — AES operations will use software fallback, " +
            "this is slower and not recommended for production")
    }

    return caps
}
```

### Encryption Flow

**Storing a credential:**

```
1. Generate DEK: 32 random bytes via crypto/rand
2. Encrypt credential plaintext with DEK using AES-256-GCM
   - 96-bit random nonce (never reused — new nonce per encryption)
   - Produces: ciphertext + 128-bit authentication tag
3. Fetch tenant's active KEK (unwrap from DB using MEK)
4. Encrypt DEK with KEK using AES-256-GCM
5. Store in DB: encrypted_credential_blob, encrypted_dek, nonce, kek_version
6. Zero the plaintext DEK and credential from memory immediately
```

**Retrieving a credential:**

```
1. Verify caller identity + tenant match + RBAC permission
2. Write audit record BEFORE returning credential
3. Fetch encrypted blob + encrypted DEK from DB
4. Unwrap KEK using MEK (via MasterKeyProvider)
5. Decrypt DEK using KEK
6. Decrypt credential blob using DEK
7. Return SecureCredential (mlock'd, auto-zeroing, TTL-bounded)
8. Zero the DEK from memory immediately
```

---

## 3. Go Types

All types in this section are concrete and implementable. Package: `internal/vault`.

### Vault Interface

```go
package vault

import (
    "context"
    "io"
    "time"

    "github.com/google/uuid"
)

// Vault is the sole interface through which credentials are stored,
// retrieved, rotated, and revoked. There is no other path to credential
// data. All methods enforce tenant isolation and produce audit records.
type Vault interface {
    // Store encrypts and persists a credential for a tenant.
    // Returns a CredentialRef that is the only way to retrieve it later.
    Store(ctx context.Context, tenantID uuid.UUID, cred Credential) (CredentialRef, error)

    // Retrieve decrypts and returns a credential wrapped in a SecureCredential.
    // The caller MUST call Close() on the returned SecureCredential when done.
    // The SecureCredential auto-zeroes after MaxLifetime even if Close() is not called.
    Retrieve(ctx context.Context, tenantID uuid.UUID, ref CredentialRef) (*SecureCredential, error)

    // Rotate replaces a credential with a new value. The old credential is
    // marked as retired but retained (encrypted) for audit purposes.
    // A new DEK is generated for the new credential.
    Rotate(ctx context.Context, tenantID uuid.UUID, ref CredentialRef, newCred Credential) error

    // Revoke permanently disables a credential. The encrypted blob is retained
    // for audit but marked as revoked and can never be decrypted again
    // (the DEK is destroyed).
    Revoke(ctx context.Context, tenantID uuid.UUID, ref CredentialRef) error

    // List returns metadata (never plaintext) for all credentials belonging to a tenant.
    List(ctx context.Context, tenantID uuid.UUID) ([]CredentialMetadata, error)
}
```

### Credential

```go
// Credential is the plaintext credential to be stored. This type exists
// only transiently — it is zeroed immediately after encryption.
type Credential struct {
    // Type identifies the authentication mechanism (ssh_key, snmp_community, etc.).
    // Uses the CredentialType enum from the domain model.
    Type CredentialType `json:"type"`

    // Name is a human-readable label (e.g., "prod-switch-01 SSH root").
    Name string `json:"name"`

    // Payload is the sensitive credential material.
    // For passwords: the password string as bytes.
    // For SSH keys: the PEM-encoded private key.
    // For SNMP: the community string.
    // For certificates: the PEM-encoded certificate + private key.
    // For API keys: the key string.
    //
    // This field is NEVER serialized to JSON, logs, or errors.
    Payload SecureBytes `json:"-"`

    // Username is the associated username, if applicable. Not all credential
    // types have a username (e.g., SNMP community strings, API keys).
    Username string `json:"username,omitempty"`

    // Metadata holds non-sensitive attributes for search and display.
    Metadata map[string]string `json:"metadata,omitempty"`

    // RotationPolicy defines how and when this credential should be rotated.
    RotationPolicy *RotationPolicy `json:"rotation_policy,omitempty"`

    // DeviceIDs lists the devices this credential is used for.
    // A credential MAY apply to multiple devices (e.g., shared SNMP community).
    DeviceIDs []uuid.UUID `json:"device_ids,omitempty"`
}
```

### SecureBytes

```go
// SecureBytes wraps sensitive byte data with protections:
// - Custom MarshalJSON that always returns "***REDACTED***"
// - Custom String() that always returns "***REDACTED***"
// - Wipe() method that zeroes the underlying bytes using constant-time operations
//
// SecureBytes MUST be allocated on mlock'd pages via NewSecureBytes().
type SecureBytes struct {
    data   []byte
    locked bool // true if mlock'd
    wiped  bool // true if already zeroed
}

// NewSecureBytes allocates len bytes on mlock'd memory pages.
// The caller MUST call Wipe() when done.
func NewSecureBytes(data []byte) (*SecureBytes, error) {
    sb := &SecureBytes{
        data: make([]byte, len(data)),
    }
    copy(sb.data, data)

    // Lock the memory page to prevent swapping to disk
    if err := mlock(sb.data); err != nil {
        // Wipe and fail — do not proceed with unlocked credential memory
        wipeBytes(sb.data)
        return nil, fmt.Errorf("mlock failed for credential memory: %w", err)
    }
    sb.locked = true

    return sb, nil
}

// Bytes returns the underlying data. Panics if already wiped.
func (s *SecureBytes) Bytes() []byte {
    if s.wiped {
        panic("vault: attempt to read wiped SecureBytes")
    }
    return s.data
}

// Wipe zeroes all bytes using a constant-time operation that the compiler
// cannot optimize away, then unlocks the memory pages.
func (s *SecureBytes) Wipe() {
    if s.wiped {
        return
    }
    wipeBytes(s.data)
    if s.locked {
        _ = munlock(s.data) // best-effort unlock
    }
    s.wiped = true
}

// MarshalJSON always returns the string "***REDACTED***".
// SecureBytes MUST NOT be serialized. This method exists as a safety net
// to prevent accidental serialization from leaking credential material.
func (s *SecureBytes) MarshalJSON() ([]byte, error) {
    return []byte(`"***REDACTED***"`), nil
}

// String always returns "***REDACTED***".
func (s *SecureBytes) String() string {
    return "***REDACTED***"
}

// GoString always returns "***REDACTED***" (for %#v formatting).
func (s *SecureBytes) GoString() string {
    return "***REDACTED***"
}

// Format implements fmt.Formatter to prevent any format verb from leaking data.
func (s *SecureBytes) Format(f fmt.State, verb rune) {
    fmt.Fprint(f, "***REDACTED***")
}
```

### SecureCredential

```go
// SecureCredential is the return type from Vault.Retrieve(). It wraps decrypted
// credential material with automatic protections:
//
// - Implements io.Closer — calling Close() zeroes all sensitive bytes
// - Uses mlock to prevent swapping
// - Has a TTL — auto-zeroes after MaxLifetime even if Close() is not called
// - runtime.SetFinalizer as a last-resort safety net
//
// Callers MUST use this pattern:
//
//     cred, err := vault.Retrieve(ctx, tenantID, ref)
//     if err != nil { ... }
//     defer cred.Close()
//     password := cred.Payload.Bytes()
//     // use password
//     // cred.Close() is called by defer, zeroing password
type SecureCredential struct {
    Type     CredentialType
    Name     string
    Username string
    Payload  *SecureBytes
    Metadata map[string]string

    // maxLifetime is the TTL after which the credential auto-zeroes.
    maxLifetime time.Duration
    // createdAt is when this SecureCredential was instantiated (not when the
    // credential was first stored — when this in-memory copy was created).
    createdAt time.Time
    // timer fires at maxLifetime to auto-wipe.
    timer *time.Timer
    // closed tracks whether Close() has been called.
    closed bool
}

// NewSecureCredential creates a SecureCredential with TTL-based auto-wipe.
// maxLifetime controls how long the plaintext persists in memory.
// A runtime.SetFinalizer is registered as a safety net.
func NewSecureCredential(
    credType CredentialType,
    name string,
    username string,
    payload *SecureBytes,
    metadata map[string]string,
    maxLifetime time.Duration,
) *SecureCredential {
    sc := &SecureCredential{
        Type:        credType,
        Name:        name,
        Username:    username,
        Payload:     payload,
        Metadata:    metadata,
        maxLifetime: maxLifetime,
        createdAt:   time.Now(),
    }

    // Auto-wipe timer: credential is zeroed after maxLifetime
    sc.timer = time.AfterFunc(maxLifetime, func() {
        sc.Close()
    })

    // Safety net finalizer: if Close() is never called and the timer
    // somehow fails, the GC will zero the credential.
    runtime.SetFinalizer(sc, func(s *SecureCredential) {
        s.Close()
    })

    return sc
}

// Close zeroes all sensitive bytes and stops the auto-wipe timer.
// Safe to call multiple times.
func (sc *SecureCredential) Close() error {
    if sc.closed {
        return nil
    }
    sc.closed = true
    if sc.timer != nil {
        sc.timer.Stop()
    }
    if sc.Payload != nil {
        sc.Payload.Wipe()
    }
    return nil
}

// Ensure SecureCredential implements io.Closer at compile time.
var _ io.Closer = (*SecureCredential)(nil)
```

### CredentialRef and CredentialMetadata

```go
// CredentialRef is an opaque reference to a stored credential.
// It contains no sensitive material and is safe to log, serialize, and pass around.
type CredentialRef struct {
    ID       uuid.UUID `json:"id" db:"id"`
    TenantID uuid.UUID `json:"tenant_id" db:"tenant_id"`
}

// CredentialMetadata is the non-sensitive metadata about a stored credential.
// Returned by Vault.List(). Contains no credential material — only labels,
// types, and lifecycle timestamps.
type CredentialMetadata struct {
    ID              uuid.UUID          `json:"id" db:"id"`
    TenantID        uuid.UUID          `json:"tenant_id" db:"tenant_id"`
    Type            CredentialType     `json:"type" db:"type"`
    Name            string             `json:"name" db:"name"`
    Username        string             `json:"username,omitempty" db:"username"`
    DeviceIDs       []uuid.UUID        `json:"device_ids,omitempty"`
    RotationPolicy  *RotationPolicy    `json:"rotation_policy,omitempty"`
    Status          CredentialStatus   `json:"status" db:"status"`
    CreatedAt       time.Time          `json:"created_at" db:"created_at"`
    UpdatedAt       time.Time          `json:"updated_at" db:"updated_at"`
    LastRotatedAt   *time.Time         `json:"last_rotated_at,omitempty" db:"last_rotated_at"`
    NextRotationAt  *time.Time         `json:"next_rotation_at,omitempty" db:"next_rotation_at"`
    LastAccessedAt  *time.Time         `json:"last_accessed_at,omitempty" db:"last_accessed_at"`
    AccessCount     int64              `json:"access_count" db:"access_count"`
    KEKVersion      int                `json:"kek_version" db:"kek_version"`
    Metadata        map[string]string  `json:"metadata,omitempty"`
}

type CredentialStatus string

const (
    CredentialStatusActive   CredentialStatus = "active"
    CredentialStatusRotating CredentialStatus = "rotating"
    CredentialStatusRetired  CredentialStatus = "retired"
    CredentialStatusRevoked  CredentialStatus = "revoked"
)
```

### EncryptedBlob

```go
// EncryptedBlob is the at-rest representation of an encrypted credential.
// This is what gets stored in the database. It contains only ciphertext —
// no plaintext ever touches the database.
type EncryptedBlob struct {
    ID            uuid.UUID `db:"id"`
    TenantID      uuid.UUID `db:"tenant_id"`

    // Ciphertext is the AES-256-GCM encrypted credential payload.
    Ciphertext    []byte    `db:"ciphertext"`
    // Nonce is the 96-bit random nonce used for this encryption.
    Nonce         []byte    `db:"nonce"`
    // AuthTag is the 128-bit GCM authentication tag (appended to ciphertext by Go's GCM).
    // In Go's crypto/cipher GCM implementation, the auth tag is appended to the
    // ciphertext by Seal(). This field documents that it is present.

    // EncryptedDEK is the AES-256 data encryption key, encrypted by the tenant's KEK.
    EncryptedDEK  []byte    `db:"encrypted_dek"`
    // DEKNonce is the nonce used to encrypt the DEK with the KEK.
    DEKNonce      []byte    `db:"dek_nonce"`

    // KEKVersion identifies which version of the tenant's KEK encrypted this DEK.
    KEKVersion    int       `db:"kek_version"`

    // IntegrityHash is a BLAKE3 hash of (ciphertext || encrypted_dek || kek_version)
    // computed at write time and verified at read time to detect tampering.
    IntegrityHash []byte    `db:"integrity_hash"`

    CreatedAt     time.Time `db:"created_at"`
    UpdatedAt     time.Time `db:"updated_at"`
}
```

### KeyMetadata and RotationPolicy

```go
// KeyMetadata tracks the lifecycle of any key in the hierarchy.
type KeyMetadata struct {
    KeyID        string        `json:"key_id"`
    KeyType      KeyType       `json:"key_type"`       // "mek", "kek", "dek"
    Algorithm    string        `json:"algorithm"`      // "AES-256-GCM"
    CreatedAt    time.Time     `json:"created_at"`
    ExpiresAt    time.Time     `json:"expires_at"`
    RotatedAt    *time.Time    `json:"rotated_at,omitempty"`
    Status       string        `json:"status"`         // "active", "rotating", "retired", "destroyed"
    Provider     string        `json:"provider"`       // MEK provider type for audit
}

type KeyType string

const (
    KeyTypeMEK KeyType = "mek"
    KeyTypeKEK KeyType = "kek"
    KeyTypeDEK KeyType = "dek"
)

// RotationPolicy defines the rotation schedule and strategy for a credential.
type RotationPolicy struct {
    // AutoRotate enables automatic rotation for protocols that support it.
    AutoRotate bool `json:"auto_rotate" db:"auto_rotate"`

    // Interval is the time between rotations (e.g., 90 days, 24 hours).
    Interval time.Duration `json:"interval" db:"interval"`

    // MaxAge is the absolute maximum age before the credential MUST be rotated.
    // If rotation fails, alerts are generated.
    MaxAge time.Duration `json:"max_age" db:"max_age"`

    // Strategy defines how rotation is performed.
    Strategy RotationStrategy `json:"strategy" db:"strategy"`

    // NotifyBeforeDays sends alerts this many days before rotation is due.
    NotifyBeforeDays int `json:"notify_before_days" db:"notify_before_days"`
}

type RotationStrategy string

const (
    // RotationStrategyInPlace: generate new credential, update on device, update in vault.
    RotationStrategyInPlace RotationStrategy = "in_place"
    // RotationStrategyBlueGreen: create new credential, test it, then deactivate old one.
    RotationStrategyBlueGreen RotationStrategy = "blue_green"
    // RotationStrategyManual: alert operator, do not auto-rotate.
    RotationStrategyManual RotationStrategy = "manual"
)
```

### Memory Wipe Primitives

```go
package vault

import (
    "crypto/subtle"
    "syscall"
    "unsafe"
)

// wipeBytes zeroes a byte slice using crypto/subtle to prevent
// the compiler from optimizing away the write.
// This is the ONLY function that should be used to zero credential material.
func wipeBytes(b []byte) {
    // subtle.ConstantTimeCopy with a slice of zeroes.
    // This cannot be optimized away because the compiler does not know
    // whether the source and destination overlap.
    zeros := make([]byte, len(b))
    subtle.ConstantTimeCopy(1, b, zeros)
}

// mlock locks memory pages to prevent swapping to disk.
// Uses syscall.Mlock on Linux/macOS.
func mlock(b []byte) error {
    if len(b) == 0 {
        return nil
    }
    return syscall.Mlock(b)
}

// munlock unlocks previously locked memory pages.
func munlock(b []byte) error {
    if len(b) == 0 {
        return nil
    }
    return syscall.Munlock(b)
}

// disableCoreDumps prevents the process from producing core dumps
// that could contain credential material.
// On Linux: prctl(PR_SET_DUMPABLE, 0)
// On macOS: this is a no-op (macOS does not support PR_SET_DUMPABLE,
// but core dumps are off by default and the process is sandboxed).
func disableCoreDumps() error {
    // Linux implementation:
    // _, _, errno := syscall.Syscall(syscall.SYS_PRCTL,
    //     uintptr(syscall.PR_SET_DUMPABLE), 0, 0)
    // if errno != 0 {
    //     return fmt.Errorf("prctl PR_SET_DUMPABLE failed: %w", errno)
    // }
    // Build-tagged implementations in:
    //   vault_linux.go  — calls prctl
    //   vault_darwin.go — no-op with log
    return nil
}
```

---

## 4. Memory Protection

Memory protection is not optional. Every credential byte in the LOOM vault process follows these rules.

### Locked Pages

All `SecureBytes` allocations use `syscall.Mlock()` to pin memory pages in RAM. This prevents the operating system from swapping credential material to disk, where it could be recovered by an attacker with physical access.

```go
// On startup, the vault process raises its RLIMIT_MEMLOCK to allow
// sufficient locked memory for the expected credential working set.
func raiseMemlockLimit() error {
    var rlim syscall.Rlimit
    if err := syscall.Getrlimit(syscall.RLIMIT_MEMLOCK, &rlim); err != nil {
        return fmt.Errorf("getrlimit MEMLOCK: %w", err)
    }

    // Request 64 MB of lockable memory. This supports approximately
    // 16,000 concurrent decrypted credentials (4 KB each average).
    const desiredLimit = 64 * 1024 * 1024
    if rlim.Cur < desiredLimit {
        rlim.Cur = desiredLimit
        if rlim.Max < desiredLimit {
            rlim.Max = desiredLimit
        }
        if err := syscall.Setrlimit(syscall.RLIMIT_MEMLOCK, &rlim); err != nil {
            return fmt.Errorf("setrlimit MEMLOCK to %d: %w — run with CAP_IPC_LOCK or increase limits in /etc/security/limits.conf", desiredLimit, err)
        }
    }
    return nil
}
```

### Zeroing Guarantees

1. **Primary mechanism:** `SecureCredential.Close()` calls `SecureBytes.Wipe()` which uses `subtle.ConstantTimeCopy` with a zero buffer. The `subtle` package is designed for security-sensitive operations; the Go compiler is not permitted to optimize away its writes.

2. **TTL enforcement:** `time.AfterFunc(maxLifetime, ...)` auto-wipes credentials that outlive their TTL. Default `maxLifetime` is 5 minutes. Configurable per credential type — short-lived tokens may have a 30-second TTL.

3. **Finalizer safety net:** `runtime.SetFinalizer` is registered on every `SecureCredential`. If neither `Close()` nor the TTL timer fires (programming error), the garbage collector triggers zeroing. This is a last resort — it is not deterministic and MUST NOT be relied upon.

4. **DEK zeroing:** After encrypting or decrypting a credential, the DEK plaintext is immediately wiped. The DEK exists in memory only for the duration of the encrypt/decrypt operation — typically less than 1 microsecond on modern hardware with AES-NI.

### Core Dump Prevention

```go
// vault_linux.go
//go:build linux

package vault

import "syscall"

func disableCoreDumps() error {
    _, _, errno := syscall.RawSyscall(syscall.SYS_PRCTL,
        4 /* PR_SET_DUMPABLE */, 0, 0)
    if errno != 0 {
        return errno
    }
    return nil
}
```

### Log / Serialization Protection

The `SecureBytes` type prevents credential leakage through every Go output mechanism:

| Output Path | Protection |
|------------|-----------|
| `json.Marshal` | `MarshalJSON()` returns `"***REDACTED***"` |
| `fmt.Println` / `%s` / `%v` | `String()` returns `"***REDACTED***"` |
| `fmt.Printf("%#v")` | `GoString()` returns `"***REDACTED***"` |
| `fmt.Printf("%x")` / any verb | `Format()` writes `"***REDACTED***"` |
| `slog` / structured logging | `LogValue()` returns `slog.StringValue("***REDACTED***")` |
| Stack traces | Type is opaque — no exported byte fields |
| `pprof` heap dump | Memory is wiped by TTL before most heap profiles |

The `Credential` struct tags `Payload` as `json:"-"` so it is excluded from JSON serialization entirely. The `SecureBytes.MarshalJSON()` is a secondary defense in case the field is ever included by reflection-based serializers.

---

## 5. Access Control

### Authentication Chain

Every `Vault.Retrieve()` call requires all of the following, checked in order:

1. **Authenticated identity.** The `context.Context` must carry a verified identity (JWT claims validated by the API gateway). Unauthenticated requests are rejected before reaching the vault.

2. **Tenant match.** The requesting identity's tenant ID must match the credential's tenant ID. This is enforced at the type level: `CredentialRef` includes `TenantID`, and the vault implementation compares it against the credential's `TenantID` in the database. A mismatch returns `ErrTenantMismatch` — not "not found" (to avoid oracle attacks, this error is converted to a generic 403 at the API boundary).

3. **RBAC permission.** The identity must hold the `credential:read` permission for the specific credential or a wildcard permission for the tenant. Permissions are defined in the LOOM identity model.

4. **Audit record written.** The audit record is written BEFORE the credential is returned. If the audit write fails, the credential is not returned. This ensures no credential access is unaudited.

```go
// retrieveWithAccessControl implements the full access control chain.
func (v *vaultImpl) retrieveWithAccessControl(
    ctx context.Context,
    tenantID uuid.UUID,
    ref CredentialRef,
) (*SecureCredential, error) {
    // 1. Extract and validate caller identity
    identity, err := auth.IdentityFromContext(ctx)
    if err != nil {
        return nil, fmt.Errorf("vault: unauthenticated: %w", err)
    }

    // 2. Tenant isolation check
    if identity.TenantID != tenantID || ref.TenantID != tenantID {
        return nil, ErrTenantMismatch // converted to 403 at API boundary
    }

    // 3. RBAC permission check
    if !identity.HasPermission("credential:read", ref.ID.String()) {
        return nil, ErrPermissionDenied
    }

    // 4. Rate limiting check
    if err := v.rateLimiter.Check(identity.ID, "credential:read"); err != nil {
        return nil, fmt.Errorf("vault: rate limit exceeded: %w", err)
    }

    // 5. Write audit record BEFORE returning credential
    auditRecord := audit.AuditRecord{
        TenantID:     tenantID.String(),
        ActorType:    string(identity.Type),
        ActorID:      identity.ID,
        Action:       "credential.retrieved",
        ResourceType: "credential",
        ResourceID:   ref.ID.String(),
        Outcome:      "success",
    }
    if err := v.auditor.Write(ctx, auditRecord); err != nil {
        // Audit write failure — DO NOT return the credential
        return nil, fmt.Errorf("vault: audit write failed, credential access denied: %w", err)
    }

    // 6. Decrypt and return
    return v.decrypt(ctx, tenantID, ref)
}
```

### Anti-Bulk-Export

Credentials cannot be bulk-exported. The `Vault` interface provides no "export all" or "dump" method. The `List()` method returns only `CredentialMetadata` (never plaintext). `Retrieve()` returns one credential at a time.

Rate limiting enforces this at runtime:

```go
// CredentialRateLimiter tracks per-identity credential access rates.
type CredentialRateLimiter struct {
    // MaxRetrievalsPerWindow is the maximum number of credential retrievals
    // allowed per identity in a sliding window.
    MaxRetrievalsPerWindow int           // default: 50
    Window                 time.Duration // default: 5 minutes
    // AlertThreshold triggers a security alert when reached.
    AlertThreshold         int           // default: 20 (triggers at 40% of max)
}
```

If a single identity retrieves more than `AlertThreshold` credentials in the window, a security alert is emitted to the audit log and the configured alerting channel (PagerDuty, Slack, etc.). If `MaxRetrievalsPerWindow` is exceeded, further retrievals are denied until the window slides.

### Break Glass Emergency Access

<!-- AUDIT-FIX: C-07 — Removed contradictory break-glass design (pre-generated tokens, 1-hour TTL, physical safe storage) -->
<!-- SUPERSEDED: Previous break-glass design replaced by SECURITY-HARDENING-RESPONSES.md Section 4 -->

Break-glass tokens: see **SECURITY-HARDENING-RESPONSES.md Section 4** (authoritative). Summary of the current design:

1. **TTL: 15 minutes.** Tokens expire 15 minutes after generation. This is not configurable.
2. **Generated on-demand with dual authorization.** Two authorized operators must independently approve the break-glass request. Tokens are **NOT pre-generated** and are **NOT stored offline**.
3. Break-glass access bypasses RBAC but **NOT** tenant isolation and **NOT** audit.
4. Every break-glass access produces an enhanced audit record with `action: "credential.break_glass_retrieved"`.
5. A mandatory notification is sent to all tenant administrators and the security team.
6. Break-glass tokens are single-use — each token has a unique `jti` that is recorded on first use and rejected on subsequent attempts.

<!-- END AUDIT-FIX: C-07 -->

---

## 6. Rotation

### Automatic Rotation (Protocol-Aware)

LOOM performs zero-downtime credential rotation for protocols that support programmatic credential management. The rotation follows a test-before-commit pattern:

#### SSH Key Rotation

```
1. Generate new Ed25519 keypair in vault
2. Deploy new public key to target device(s) via existing SSH session
   (using the current key that still works)
3. Verify: open a NEW SSH session using the new private key
4. If verify succeeds: remove old public key from device, store new private key
   in vault, mark old credential as retired
5. If verify fails: remove new public key from device, alert operator,
   keep old credential active
```

#### SNMP Community String Rotation

```
1. Generate new random community string (32+ characters, alphanumeric)
2. Update community string on target device via SNMP SET or SSH CLI
3. Verify: perform SNMP GET using new community string
4. If verify succeeds: store new string in vault, mark old as retired
5. If verify fails: revert device to old community string, alert operator
```

#### Cloud API Key Rotation

```
1. Create new API key via cloud provider API (AWS IAM, GCP Service Account, etc.)
2. Verify: make a test API call using new key
3. If verify succeeds: store new key in vault, delete old key via provider API
4. If verify fails: delete new key via provider API, alert operator
```

#### HTTP Bearer Token / Redfish Session Rotation

```
1. Create new session / request new token using existing credentials
2. Verify: make authenticated API call with new token
3. If verify succeeds: store new token, invalidate old session
4. If verify fails: keep old session, alert operator
```

### Manual Rotation Workflow

For credential types that do not support programmatic rotation (e.g., IPMI passwords on older hardware, vendor portal credentials):

1. LOOM generates the alert: "Credential X is due for rotation (last rotated N days ago)."
2. Operator updates the credential on the device manually.
3. Operator stores the new credential in LOOM via API or UI.
4. LOOM verifies the new credential against the device.
5. Old credential is marked as retired.

### Rotation Tracking

```go
// RotationTracker monitors credential rotation status across all tenants.
type RotationTracker struct {
    // CheckInterval is how often the tracker scans for overdue rotations.
    CheckInterval time.Duration // default: 1 hour

    // OverdueThreshold defines when a credential is considered overdue
    // relative to its rotation policy interval.
    OverdueThreshold float64 // default: 1.5 (150% of interval = overdue)

    // CriticalThreshold defines when an overdue credential escalates
    // to a critical alert.
    CriticalThreshold float64 // default: 2.0 (200% of interval = critical)
}
```

The rotation tracker produces these audit actions:
- `credential.rotation_due` — credential is approaching rotation deadline.
- `credential.rotation_overdue` — credential has passed its rotation deadline.
- `credential.rotation_critical` — credential is critically overdue, escalated alert.
- `credential.rotated` — credential was successfully rotated.
- `credential.rotation_failed` — rotation attempt failed, old credential still active.

---

## 7. Vault Backend Implementations

### Backend 1: PostgreSQL + Envelope Encryption (Default)

This is the default backend for LOOM deployments. All encryption and decryption happens in the Go process — PostgreSQL stores and retrieves opaque ciphertext only.

**Database schema:**

```sql
-- Encrypted credential storage. PostgreSQL sees ONLY ciphertext.
CREATE TABLE vault_credentials (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    type            TEXT NOT NULL,
    name            TEXT NOT NULL,
    username        TEXT,
    ciphertext      BYTEA NOT NULL,           -- AES-256-GCM encrypted payload
    nonce           BYTEA NOT NULL,           -- 96-bit GCM nonce
    encrypted_dek   BYTEA NOT NULL,           -- DEK encrypted by tenant KEK
    dek_nonce       BYTEA NOT NULL,           -- nonce for DEK encryption
    kek_version     INTEGER NOT NULL,         -- which KEK version encrypted the DEK
    integrity_hash  BYTEA NOT NULL,           -- BLAKE3 integrity hash
    status          TEXT NOT NULL DEFAULT 'active',
    device_ids      UUID[] DEFAULT '{}',
    rotation_policy JSONB,
    metadata        JSONB DEFAULT '{}',
    access_count    BIGINT DEFAULT 0,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    last_rotated_at TIMESTAMPTZ,
    last_accessed_at TIMESTAMPTZ
);

-- Tenant Key Encryption Keys — encrypted by MEK.
CREATE TABLE vault_tenant_keks (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    version         INTEGER NOT NULL,
    encrypted_key   BYTEA NOT NULL,           -- AES-256 key wrapped by MEK
    master_key_id   TEXT NOT NULL,            -- identifies which MEK wrapped this
    status          TEXT NOT NULL DEFAULT 'active',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    rotated_at      TIMESTAMPTZ,
    expires_at      TIMESTAMPTZ NOT NULL,
    UNIQUE (tenant_id, version)
);

-- Index for fast tenant-scoped queries.
CREATE INDEX idx_vault_credentials_tenant ON vault_credentials(tenant_id);
CREATE INDEX idx_vault_credentials_status ON vault_credentials(tenant_id, status);
CREATE INDEX idx_vault_tenant_keks_active ON vault_tenant_keks(tenant_id, status)
    WHERE status = 'active';

-- Row-level security: each tenant's DB user can only see their own rows.
ALTER TABLE vault_credentials ENABLE ROW LEVEL SECURITY;
CREATE POLICY tenant_isolation ON vault_credentials
    USING (tenant_id = current_setting('app.current_tenant')::UUID);

ALTER TABLE vault_tenant_keks ENABLE ROW LEVEL SECURITY;
CREATE POLICY tenant_kek_isolation ON vault_tenant_keks
    USING (tenant_id = current_setting('app.current_tenant')::UUID);
```

**Security properties of this backend:**
- A stolen database dump yields only ciphertext. Each credential is encrypted with its own DEK. Each DEK is encrypted with a tenant KEK. The KEK is encrypted with the MEK. Without the MEK (held in HSM/KMS/memory), the data is unrecoverable.
- PostgreSQL row-level security provides an additional isolation layer at the database level.
- BLAKE3 integrity hashes detect any tampering with ciphertext or encrypted keys.
- The PostgreSQL connection uses TLS (`sslmode=verify-full`) with certificate pinning.

### Backend 2: HashiCorp Vault Integration

For organizations running HashiCorp Vault, LOOM delegates all cryptographic operations and credential storage to Vault:

```go
// HashiCorpVaultBackend stores credentials in Vault's KV v2 engine
// and uses Vault's Transit engine for key operations.
type HashiCorpVaultBackend struct {
    client       *vaultapi.Client
    kvMountPath  string // e.g., "secret" (KV v2 mount)
    transitPath  string // e.g., "transit" (Transit engine mount)
    transitKey   string // e.g., "loom-master" (Transit key name)
}
```

- **Credential storage:** Vault KV v2 engine at path `secret/data/loom/{tenant_id}/{credential_id}`.
- **Key operations:** Vault Transit engine performs encrypt/decrypt. LOOM sends plaintext to Transit over mTLS; Transit encrypts it with Vault-managed keys and returns ciphertext.
- **Key rotation:** Vault Transit handles key rotation natively. LOOM calls `transit/keys/loom-master/rotate` and Vault re-wraps with the new key version.
- **Audit:** Both LOOM's audit log and Vault's native audit log capture every operation.
- **Lease management:** Vault dynamic secrets can be used for credentials that support them (e.g., database passwords, cloud IAM tokens).

**Trade-off:** This backend provides the highest security (Vault is FIPS 140-2 compliant in Enterprise) but adds an operational dependency on a Vault cluster.

### Backend 3: HSM-Backed (PKCS#11)

The HSM-backed backend keeps the master key entirely within the HSM. KEK and DEK operations use AES-NI in the Go process, but the MEK never leaves the HSM boundary.

```go
// HSMBackend uses PKCS#11 for master key operations.
type HSMBackend struct {
    pkcs11Ctx   *pkcs11.Ctx
    session     pkcs11.SessionHandle
    mekHandle   pkcs11.ObjectHandle // handle to the MEK inside the HSM
    store       CredentialStore     // PostgreSQL or other storage for encrypted blobs
}
```

- **Supported HSMs:** Thales Luna Network HSM, Entrust nShield Connect, YubiHSM 2, AWS CloudHSM.
- **MEK lifecycle:** The MEK is generated inside the HSM (`C_GenerateKey`), never exported. Wrap/unwrap operations are performed via `C_WrapKey` / `C_UnwrapKey` PKCS#11 calls.
- **Connection pooling:** PKCS#11 sessions are pooled (HSMs have limited concurrent session counts). A background health checker verifies HSM connectivity every 30 seconds.
- **Failover:** If the primary HSM is unreachable, failover to a replica HSM (HSMs typically support clustering). If all HSMs are unreachable, the vault enters read-only mode — existing decrypted credentials in memory remain available (within their TTL), but no new decrypt or encrypt operations are possible until HSM connectivity is restored.

---

## 8. Comparison to Apple's Data Protection Architecture

Apple's Data Protection is the gold standard for consumer device credential management. LOOM's vault is designed to match or exceed Apple's architecture for server infrastructure management.

| Apple Data Protection | LOOM Vault | Notes |
|----------------------|-----------|-------|
| **Secure Enclave** — dedicated security processor with hardware-fused UID key | **HSM / TPM 2.0** — dedicated hardware security module with hardware-bound master key | Both ensure the root key never exists in main memory. LOOM supports multiple HSM vendors. |
| **Per-file keys** — every file encrypted with unique key | **Per-credential DEKs** — every credential encrypted with unique key | Identical concept. Compromise of one DEK reveals one credential only. |
| **Class keys** — keys grouped by protection class (A/B/C/D) | **Per-tenant KEKs** — keys grouped by tenant | LOOM's tenant isolation is stronger: Apple's classes are on one device; LOOM's tenants are cryptographically isolated across the entire system. |
| **Hardware UID key** — fused into silicon, never extractable | **MEK sealed to HSM** — generated inside HSM, never exported | Functionally equivalent. Both ensure the root of trust is hardware-bound. |
| **Effaceable Storage** — instant key destruction = instant data destruction | **Credential revocation** — DEK destruction makes credential unrecoverable | Same pattern: destroy the key, the data becomes cryptographic garbage. |
| **Key wrapping chain** — UID → class key → per-file key | **Key wrapping chain** — MEK → KEK → DEK | Same 3-tier envelope encryption. |
| **Passcode-derived key** — PBKDF2 from user passcode mixed with UID | **Argon2id-derived key** — memory-hard KDF (stronger than PBKDF2) for local fallback | LOOM uses a stronger KDF. Argon2id is resistant to GPU/ASIC attacks; PBKDF2 is not. |

### Where LOOM Goes Beyond Apple

1. **Multi-tenant isolation.** Apple protects one user's data on one device. LOOM protects thousands of tenants' credentials across a distributed infrastructure, with cryptographic enforcement that one tenant cannot access another's credentials even if they compromise the application layer.

2. **Automatic credential rotation.** Apple does not rotate per-file keys. LOOM automatically rotates credentials on a configurable schedule, with protocol-aware rotation strategies that verify the new credential works before retiring the old one.

3. **Protocol-aware credential lifecycle.** Apple treats all files the same. LOOM understands that an SSH key requires a different rotation procedure than an SNMP community string, which is different from a cloud API key. Each credential type has a lifecycle handler.

4. **Full audit trail.** Apple does not audit per-file key access. LOOM records every credential retrieval with actor identity, timestamp, and correlation to the workflow that requested it.

5. **Distributed key hierarchy.** Apple's key hierarchy exists on a single device. LOOM's key hierarchy spans HSMs, KMS services, and multiple application instances, with key material never stored in plaintext at rest in any tier.

---

## 9. Threat Scenarios and Mitigations

### Threat 1: Database Dump Stolen

**Attack:** An attacker obtains a full backup or live dump of the PostgreSQL database, including the `vault_credentials` and `vault_tenant_keks` tables.

**What the attacker has:**
- Encrypted credential blobs (AES-256-GCM ciphertext)
- Encrypted DEKs (each encrypted by a tenant KEK)
- Encrypted KEKs (each encrypted by the MEK)
- Nonces and integrity hashes

**What the attacker does NOT have:**
- The MEK (held in HSM/KMS/memory, never stored in the database)
- Any plaintext credential material
- Any plaintext key material

**Defense layers:**
1. Each credential is encrypted with its own unique DEK — no two credentials share a key.
2. Each DEK is encrypted with a tenant-specific KEK — even if one KEK is somehow derived, other tenants are unaffected.
3. Each KEK is encrypted with the MEK — which is in the HSM/KMS, not the database.
4. AES-256-GCM provides authenticated encryption — ciphertext cannot be modified without detection.
5. BLAKE3 integrity hashes provide an independent tamper-detection layer.

**Residual risk:** None, assuming AES-256 remains unbroken. The attacker faces a 2^256 keyspace with no known shortcut.

### Threat 2: Memory Dump of Running Process

**Attack:** An attacker with root access captures a memory dump of the LOOM vault process (via `/proc/pid/mem`, `gcore`, or a debugger).

**Defenses:**
1. `prctl(PR_SET_DUMPABLE, 0)` prevents core dumps and restricts `/proc/pid/mem` access on Linux.
2. Credential plaintext exists in memory only for the duration of active use (bounded by `MaxLifetime` TTL, default 5 minutes).
3. All credential memory is allocated on `mlock`'d pages (not in swap).
4. Credentials are zeroed immediately after `Close()` or TTL expiry using `subtle.ConstantTimeCopy` with zero buffer.
5. DEKs exist in memory for microseconds (only during encrypt/decrypt operations).
6. The KEK is held in memory only during active credential operations, then wiped.

**Residual risk:** If the attacker captures memory at the exact moment a credential is decrypted and in use, they can read that one credential. This window is minimized (TTL-bounded, typically seconds to minutes). The MEK may be in memory if using the local passphrase provider — HSM/KMS providers avoid this entirely.

### Threat 3: Compromised Tenant Attempting Cross-Tenant Access

**Attack:** A tenant's credentials (API tokens, user accounts) are compromised. The attacker attempts to access other tenants' credentials.

**Defenses:**
1. **Type-level isolation:** `CredentialRef` includes `TenantID`. The vault verifies `ref.TenantID == tenantID` (from the authenticated context) on every operation. A mismatch returns `ErrTenantMismatch`.
2. **Cryptographic isolation:** Each tenant has its own KEK. Even if an attacker bypasses the application-level check, they cannot decrypt another tenant's DEKs without that tenant's KEK.
3. **Database-level isolation:** PostgreSQL row-level security (RLS) ensures queries can only return rows matching the current tenant. Even a SQL injection cannot read other tenants' rows.
4. **No enumeration:** The `List()` method is tenant-scoped. There is no API to list credentials across tenants.

**Residual risk:** If the attacker compromises the LOOM application process itself (not just tenant credentials), they could potentially access the MEK and derive all KEKs. This is mitigated by the HSM/KMS backend, where the MEK never enters the application process.

### Threat 4: Insider with Database Access

**Attack:** A DBA or infrastructure administrator with direct PostgreSQL access attempts to read credential values.

**Defenses:**
1. All credential material in the database is encrypted. The DBA sees only ciphertext.
2. The MEK is not stored in the database and is not accessible via any database query.
3. The database user used by LOOM has minimal privileges (SELECT, INSERT, UPDATE on vault tables only — no DDL, no superuser, no `COPY TO`).
4. The audit table is append-only — the LOOM database user does not have UPDATE or DELETE on the audit table (per the Audit Model).
5. Database access is logged at the PostgreSQL level (via `pgaudit` extension).
6. PostgreSQL connections use TLS with mutual certificate authentication — connection logs identify the connecting principal.

**Residual risk:** An insider with access to both the database AND the HSM/KMS (or the application server's memory) could theoretically reconstruct credentials. This is mitigated by separation of duties: database administrators should not have HSM/KMS administrative access.

### Threat 5: Compromised LLM Attempting to Exfiltrate Credentials

**Attack:** The LLM decision engine (per LLM-BOUNDARIES.md) is compromised or manipulated via prompt injection. It attempts to instruct LOOM to retrieve and exfiltrate credentials.

**Defenses:**
1. **The LLM never has direct access to the vault.** Per LLM-BOUNDARIES.md, the LLM produces typed recommendation structs. It cannot call `Vault.Retrieve()`. The vault interface is not exposed to the LLM integration layer.
2. **The LLM never sees credential plaintext.** Even if the LLM is involved in a workflow that uses credentials (e.g., "configure this switch"), the workflow engine retrieves the credential, uses it via the adapter, and zeroes it — the LLM sees only the workflow status, never the credential.
3. **Credential references are opaque.** The LLM may see `CredentialRef` (UUID + tenant ID) in workflow plans but cannot use them to retrieve plaintext without going through the full authentication + RBAC + audit chain.
4. **Rate limiting detects bulk retrieval.** Even if the LLM somehow triggers credential retrievals through the workflow engine, the rate limiter and anomaly detection will flag unusual patterns.

**Residual risk:** Extremely low. The LLM is architecturally separated from the vault by multiple layers. An attacker would need to compromise the LLM, the workflow engine, the authentication layer, and the RBAC layer simultaneously.

### Threat 6: Side-Channel Attacks (Timing, Cache)

**Attack:** An attacker with co-tenancy on the same physical server uses timing analysis or cache-line analysis to extract key material during cryptographic operations.

**Defenses:**
1. **AES-NI execution:** AES-256-GCM operations via AES-NI execute in constant time in dedicated CPU instructions, not in software lookup tables. This eliminates cache-timing side channels that affect software AES implementations.
2. **Constant-time comparisons:** All credential comparisons (e.g., HMAC verification) use `crypto/subtle.ConstantTimeCompare`, which does not leak information via timing.
3. **Constant-time zeroing:** `subtle.ConstantTimeCopy` for memory wiping does not short-circuit.
4. **X25519/Ed25519:** These algorithms are designed to be constant-time. Go's implementations follow this property.
5. **No branching on secret data:** The vault implementation avoids `if/else` branches based on credential values. Error paths use constant-time operations where possible.

**Residual risk:** Spectre/Meltdown-class attacks may still be possible on shared hardware. Mitigation: run the vault process on dedicated (non-shared) infrastructure with microcode patches applied. In cloud environments, use dedicated instances (e.g., AWS `metal` instances, Azure Dedicated Hosts).

### Threat 7: Supply Chain Attack on LOOM Binary

**Attack:** An attacker compromises the LOOM build pipeline and injects malicious code into the vault binary that exfiltrates credentials.

**Defenses:**
1. **Reproducible builds:** LOOM uses Go's reproducible build system. The same source code produces the same binary, verifiable by independent parties.
2. **Binary signatures:** Release binaries are signed with a LOOM release key (Ed25519). Signatures are verified before deployment.
3. **SBOM (Software Bill of Materials):** Every release includes an SBOM listing all dependencies with checksums. Dependencies are pinned to exact versions in `go.sum`.
4. **Dependency audit:** Critical dependencies (crypto libraries, HSM drivers) are vendored and audited before version bumps.
5. **SLSA Level 3 compliance:** Build provenance is generated and published with each release, documenting the build environment, source commit, and builder identity.
6. **Runtime integrity:** On startup, the vault binary verifies its own checksum against a signed manifest (defense against on-disk binary replacement).

**Residual risk:** A sophisticated attacker who compromises the Go compiler itself or the hardware the build runs on. This is mitigated by cross-compilation and verification on multiple independent build environments.

<!-- AUDIT-FIX: C-01 — Additional threat scenario for passphrase-derived MEK -->

### Threat 8: MEK in Process Memory (Passphrase Provider)

**Attack:** An attacker with access to the host (root shell, container escape, or memory forensics on a seized machine) extracts the passphrase-derived MEK from the LOOM vault process memory, gaining the ability to decrypt all KEKs and, transitively, all credentials.

**Defenses (when using HSM/KMS — the production path):**
1. The MEK **never enters the LOOM process**. All wrap/unwrap operations execute inside the HSM/KMS boundary.
2. Even with full memory access, the attacker obtains only plaintext of currently-active credentials (bounded by TTL), not the MEK itself.

**Defenses (when using local_passphrase — development only):**
1. `LOOM_ALLOW_INSECURE_MEK=true` environment variable must be explicitly set — prevents accidental production use.
2. CRITICAL alert logged on every startup.
3. `NonProductionOnly: true` in config enables automated compliance rejection.
4. `mlock`'d memory prevents swap exposure.
5. Core dumps disabled via `prctl`.

**Residual risk:** **HIGH for local_passphrase.** If the attacker captures process memory at any point during runtime, the MEK is exposed. This risk is **fully mitigated by HSM/KMS providers** where the MEK never enters the process. This is the primary reason `local_passphrase` is development-only.

<!-- END AUDIT-FIX: C-01 -->

---

## 10. Implementation Sequence

The vault will be implemented in this order, with each phase fully tested before proceeding:

1. **Phase 1 — Types and memory protection.** Define all types in this document. Implement `SecureBytes`, `wipeBytes`, `mlock`/`munlock`, core dump prevention. Unit test that zeroing actually works (read memory after wipe, verify all zero bytes).

2. **Phase 2 — Encryption primitives.** Implement AES-256-GCM encrypt/decrypt, DEK generation, BLAKE3 integrity hashing. Implement hardware detection and startup logging. Unit test with known test vectors from NIST.

3. **Phase 3 — Key hierarchy.** Implement the MEK/KEK/DEK envelope encryption chain. Implement the `MasterKeyProvider` interface with the local passphrase backend first (simplest to test). Implement KEK generation, wrapping, unwrapping.

4. **Phase 4 — PostgreSQL backend.** Implement `Store`, `Retrieve`, `List`, `Revoke` against PostgreSQL. Implement the database schema. Integration test: store a credential, retrieve it, verify plaintext matches, verify database contains only ciphertext.

5. **Phase 5 — Access control and audit.** Wire up authentication, tenant isolation, RBAC checks, and audit logging. Integration test: verify cross-tenant access is denied, verify audit records are written before credential return.

6. **Phase 6 — Rotation.** Implement rotation tracking and the SSH key rotation workflow as the first protocol-specific rotator.

7. **Phase 7 — HSM and KMS backends.** Implement the HashiCorp Vault backend, then the PKCS#11 HSM backend, then cloud KMS backends.

---

## Appendix A: Go Module Dependencies

```
# Core crypto (Go stdlib — no external dependency)
crypto/aes
crypto/cipher
crypto/ed25519
crypto/rand
crypto/subtle

# Extended crypto
golang.org/x/crypto/argon2        # Argon2id key derivation
golang.org/x/crypto/curve25519    # X25519 key exchange
golang.org/x/sys/cpu              # CPU feature detection

# Hashing
github.com/zeebo/blake3           # BLAKE3 with AVX-512 support

# HSM
github.com/miekg/pkcs11           # PKCS#11 for HSM integration

# HashiCorp Vault
github.com/hashicorp/vault/api/v2 # Vault API client

# Cloud KMS
github.com/aws/aws-sdk-go-v2/service/kms
cloud.google.com/go/kms/apiv1
github.com/Azure/azure-sdk-for-go/sdk/security/keyvault/azkeys

# TPM
github.com/google/go-tpm/tpm2     # TPM 2.0 support

# Core
github.com/google/uuid            # UUID v7 generation
```

## Appendix B: Configuration

```yaml
vault:
  # Master key provider: hsm_pkcs11, hashicorp_vault, aws_kms, azure_keyvault, gcp_kms, tpm2, local_passphrase
  master_key_provider: hsm_pkcs11

  # Provider-specific configuration
  hsm:
    library_path: /usr/lib/softhsm/libsofthsm2.so  # PKCS#11 library
    slot: 0
    pin_env_var: LOOM_HSM_PIN  # PIN read from environment variable, never from config file

  hashicorp_vault:
    address: https://vault.internal:8200
    transit_mount: transit
    transit_key: loom-master
    kv_mount: secret
    auth_method: kubernetes  # or approle, token
    role: loom-vault

  aws_kms:
    key_id: arn:aws:kms:us-east-1:123456789:key/abcd-1234
    region: us-east-1

  # Memory and security settings
  memory:
    mlock_limit_bytes: 67108864    # 64 MB
    credential_max_lifetime: 5m    # TTL for decrypted credentials in memory
    dek_max_lifetime: 1s           # TTL for DEK plaintext in memory

  # Key rotation
  rotation:
    kek_rotation_interval: 2160h   # 90 days
    default_credential_rotation: 2160h
    rotation_check_interval: 1h
    overdue_threshold: 1.5         # 150% of interval
    critical_threshold: 2.0        # 200% of interval

  # Rate limiting
  rate_limit:
    max_retrievals_per_window: 50
    window: 5m
    alert_threshold: 20

  # Audit
  audit:
    sign_records: true             # Ed25519 sign each audit record
    signing_key_path: /etc/loom/audit-signing-key.pem
```
