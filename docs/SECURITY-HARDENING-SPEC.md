# Security Hardening Specification

> **Status:** Approved -- implementation-ready
> **Closes:** RESEARCH-security-hardening.md gaps H9-H14, H17, H18
> **Author:** Principal Security Architect
> **Date:** 2026-03-21

This document contains definitive design decisions and implementation-ready specifications
for every open gap identified in RESEARCH-security-hardening.md. All "open questions" from
that document are answered here. No further design discussion is needed -- only implementation.

---

## Table of Contents

1. [Internal CA Lifecycle (H9)](#1-internal-ca-lifecycle-h9)
2. [Temporal DataConverter Key Management (H10)](#2-temporal-dataconverter-key-management-h10)
3. [LLM Cross-Tenant Context Isolation (H11)](#3-llm-cross-tenant-context-isolation-h11)
4. [Break-Glass Token Lifecycle (H12)](#4-break-glass-token-lifecycle-h12)
5. [Audit Trail Integrity (H13)](#5-audit-trail-integrity-h13)
6. [ConfigSnapshot Encryption (H14)](#6-configsnapshot-encryption-h14)
7. [Valkey Security Posture (H17)](#7-valkey-security-posture-h17)
8. [OIDC Tenant Claim Verification (H18)](#8-oidc-tenant-claim-verification-h18)

---

## 1. Internal CA Lifecycle (H9)

### Decision

Built-in CA using step-ca (smallstep) as an embedded library for Phases 1-5.
External CA integration (HashiCorp Vault PKI secrets engine) available from Phase 10+.
LOOM never implements its own CA -- step-ca is battle-tested, ACME-compliant, and Go-native.

Certificate transparency logging is NOT required. LOOM operates an internal PKI;
CT is for the public web PKI ecosystem. Internal auditing via the LOOM audit trail
is sufficient.

### Certificate Types and Lifetimes

| Certificate Type | Lifetime | Renewal Window | Use Case |
|-----------------|----------|----------------|----------|
| Service cert | 24 hours | Renew at 16 hours (2/3 lifetime) | API, workers, Temporal, NATS inter-node |
| Edge agent cert | 365 days | Renew at 30 days before expiry | Edge agents connecting via NATS leaf |
| CA intermediate | 5 years | Rotate at 4 years | Signing authority for service + agent certs |
| CA root | 10 years | Rotate at 8 years via cross-signing | Root of trust |

### ACME Protocol for Automated Issuance

All service certificates are issued and renewed automatically via ACME. The embedded
step-ca instance exposes an ACME directory endpoint on the internal network only.
Edge agents use the ACME protocol over NATS (tunneled ACME-over-NATS for agents that
cannot reach the CA directly).

```go
package ca

import (
    "context"
    "crypto/ed25519"
    "crypto/x509"
    "crypto/x509/pkix"
    "time"

    "github.com/google/uuid"
)

// CAConfig defines the internal CA configuration.
type CAConfig struct {
    // RootCertPath is the path to the root CA certificate PEM file.
    RootCertPath string `yaml:"root_cert_path"`

    // RootKeyPath is the path to the root CA private key.
    // For HSM deployments, this is a PKCS#11 URI (e.g., "pkcs11:token=loom-ca;id=1").
    RootKeyPath string `yaml:"root_key_path"`

    // IntermediateCertPath is the path to the intermediate CA certificate.
    IntermediateCertPath string `yaml:"intermediate_cert_path"`

    // IntermediateKeyPath is the path to the intermediate CA private key.
    IntermediateKeyPath string `yaml:"intermediate_key_path"`

    // ACMEListenAddr is the ACME directory endpoint address (internal only).
    // Default: "127.0.0.1:9443"
    ACMEListenAddr string `yaml:"acme_listen_addr" default:"127.0.0.1:9443"`

    // ServiceCertLifetime is the default lifetime for service certificates.
    ServiceCertLifetime time.Duration `yaml:"service_cert_lifetime" default:"24h"`

    // AgentCertLifetime is the default lifetime for edge agent certificates.
    AgentCertLifetime time.Duration `yaml:"agent_cert_lifetime" default:"8760h"` // 365 days

    // GracePeriod is the duration after certificate expiry during which
    // the certificate is still accepted with a CRITICAL alert.
    GracePeriod time.Duration `yaml:"grace_period" default:"24h"`

    // CRLLifetime is the validity period of each issued CRL.
    CRLLifetime time.Duration `yaml:"crl_lifetime" default:"1h"`

    // CRLDistributionNATSSubject is the NATS subject for CRL push distribution.
    CRLDistributionNATSSubject string `yaml:"crl_nats_subject" default:"loom.system.crl"`

    // OCSPEnabled enables the embedded OCSP responder.
    OCSPEnabled bool `yaml:"ocsp_enabled" default:"true"`

    // OCSPListenAddr is the OCSP responder listen address.
    OCSPListenAddr string `yaml:"ocsp_listen_addr" default:"127.0.0.1:9080"`

    // CrossSignOverlapDays is the overlap period during CA rotation
    // where both old and new CA certificates are trusted.
    CrossSignOverlapDays int `yaml:"cross_sign_overlap_days" default:"30"`
}

// IssuedCertificate represents a certificate issued by the internal CA.
type IssuedCertificate struct {
    SerialNumber string            `json:"serial_number"`
    Subject      pkix.Name         `json:"subject"`
    NotBefore    time.Time         `json:"not_before"`
    NotAfter     time.Time         `json:"not_after"`
    CertPEM      []byte            `json:"-"` // PEM-encoded certificate
    Fingerprint  string            `json:"fingerprint"` // SHA-256 fingerprint
    CertType     CertificateType   `json:"cert_type"`
    RevokedAt    *time.Time        `json:"revoked_at,omitempty"`
    RevokeReason *RevocationReason `json:"revoke_reason,omitempty"`
}

// CertificateType classifies issued certificates.
type CertificateType string

const (
    CertTypeService CertificateType = "service"
    CertTypeAgent   CertificateType = "agent"
    CertTypeCA      CertificateType = "ca"
)

// RevocationReason follows RFC 5280 CRLReason.
type RevocationReason int

const (
    RevocationUnspecified          RevocationReason = 0
    RevocationKeyCompromise        RevocationReason = 1
    RevocationCACompromise         RevocationReason = 2
    RevocationAffiliationChanged   RevocationReason = 3
    RevocationSuperseded           RevocationReason = 4
    RevocationCessationOfOperation RevocationReason = 5
)

// InternalCA is the CA lifecycle manager.
type InternalCA interface {
    // IssueCertificate issues a new certificate signed by the intermediate CA.
    IssueCertificate(ctx context.Context, csr *x509.CertificateRequest, certType CertificateType) (*IssuedCertificate, error)

    // RevokeCertificate revokes a certificate by serial number.
    RevokeCertificate(ctx context.Context, serialNumber string, reason RevocationReason) error

    // GenerateCRL generates a new CRL and publishes it to NATS.
    GenerateCRL(ctx context.Context) ([]byte, error)

    // VerifyCertificate checks a presented certificate against the CA trust chain,
    // CRL, and OCSP status. Returns nil if valid, ErrCertExpiredGracePeriod if
    // expired but within grace period, or an error if revoked/invalid.
    VerifyCertificate(ctx context.Context, cert *x509.Certificate) error

    // RotateIntermediate generates a new intermediate CA certificate,
    // cross-signed by the current root, with an overlap period.
    RotateIntermediate(ctx context.Context) (*IssuedCertificate, error)

    // RotateRoot performs root CA rotation with cross-signing.
    // The old root remains trusted for CrossSignOverlapDays.
    RotateRoot(ctx context.Context) (*IssuedCertificate, error)
}
```

### CRL Distribution via NATS

CRLs are generated hourly (or on-demand after any revocation) and published to the
NATS subject `loom.system.crl`. All LOOM components and edge agents subscribe to this
subject. The CRL is a DER-encoded X.509 CRL signed by the intermediate CA.

Maximum acceptable latency between CRL generation and edge agent receipt: **5 minutes**.
NATS JetStream provides at-least-once delivery with persistence. If an edge agent is
offline, it receives the CRL upon reconnection.

```go
// CRLDistributor publishes CRLs via NATS and manages local CRL caches.
type CRLDistributor struct {
    natsConn   *nats.Conn
    subject    string // "loom.system.crl"
    localCache []byte // most recent CRL DER bytes
    lastUpdate time.Time
}

// PublishCRL publishes a new CRL to all subscribers.
func (d *CRLDistributor) PublishCRL(ctx context.Context, crlDER []byte) error {
    d.localCache = crlDER
    d.lastUpdate = time.Now()
    _, err := d.natsConn.JetStream().Publish(d.subject, crlDER)
    return err
}

// SubscribeCRL registers a handler for incoming CRL updates.
// Used by edge agents and service components.
func (d *CRLDistributor) SubscribeCRL(ctx context.Context, handler func(crlDER []byte)) error {
    js, err := d.natsConn.JetStream()
    if err != nil {
        return err
    }
    _, err = js.Subscribe(d.subject, func(msg *nats.Msg) {
        handler(msg.Data)
        msg.Ack()
    }, nats.DeliverLast(), nats.Durable("crl-consumer"))
    return err
}
```

### OCSP Stapling

The embedded step-ca instance runs an OCSP responder. Hub-to-agent TLS connections
use OCSP stapling: the hub server staples the OCSP response to its TLS handshake,
eliminating the need for agents to contact the OCSP responder directly. This is
critical for edge agents with intermittent connectivity.

OCSP responses have a 1-hour validity. The OCSP responder is reachable only on the
internal network. Services cache their stapled OCSP response and refresh every 30 minutes.

### Certificate Expiry Grace Period

When a certificate has expired but is within the 24-hour grace period:

1. The TLS handshake succeeds (custom `VerifyPeerCertificate` callback).
2. A `CRITICAL` audit event `ca.cert_expired_grace` is emitted.
3. An alert fires to the operations team.
4. The connection is tagged with `expired_grace=true` in metadata for monitoring.
5. After the 24-hour grace period, the certificate is rejected.

This prevents operational outages from clock skew or delayed ACME renewals while
maintaining security accountability.

### Edge Agent Offline Resilience

Edge agents are pre-issued certificates with 365-day lifetimes. The ACME renewal
process attempts renewal at 30 days before expiry. If the agent is offline for
up to 335 days, it can still authenticate when it reconnects. If the agent exceeds
365 days offline, it requires manual re-enrollment:

1. Generate new CSR on the agent.
2. Admin approves the CSR via the LOOM control plane.
3. New certificate is issued and delivered out-of-band (USB, secure copy).

### CA Rotation Procedure

**Intermediate CA rotation (every 5 years):**

1. Generate new intermediate key pair (Ed25519).
2. Create new intermediate certificate signed by the current root CA.
3. Both old and new intermediate certificates are trusted for 30 days.
4. During the overlap, new certificates are issued by the new intermediate.
5. All services renew their certificates (24-hour lifetime ensures full rollover within 24 hours).
6. Edge agents renew on their next ACME cycle (within 30 days).
7. After 30 days, the old intermediate is removed from the trust bundle.

**Root CA rotation (every 10 years):**

1. Generate new root key pair (Ed25519).
2. Cross-sign: new root CA signs the existing intermediate, and old root CA signs a
   certificate for the new root. This creates a trust bridge.
3. Both root certificates are distributed in the trust bundle for 30 days.
4. New intermediate CA is issued under the new root.
5. After 30 days, the old root is removed from the trust bundle.
6. Old root private key is destroyed (HSM key deletion ceremony).

---

## 2. Temporal DataConverter Key Management (H10)

### Decision

Per-namespace key derived from the tenant KEK via HKDF-SHA256. This integrates with
the existing 3-tier envelope encryption hierarchy (MEK -> KEK -> DEK) documented in
VAULT-ARCHITECTURE.md.

### Design Answers

- **Temporal's DataConverter CAN access namespace context.** The `DataConverter` interface
  receives a `context.Context` in the `ToPayload`/`FromPayload` methods. The Temporal SDK's
  interceptor chain provides namespace context. The `TenantDataConverter` extracts the
  namespace from context and uses it as the HKDF info parameter.

- **Key rotation and workflow replay are compatible.** Old payloads reference a `kek_version`
  in their metadata header. On replay, the DataConverter reads the version, derives the
  correct DEK from the corresponding KEK version, and decrypts. Old KEK versions are retained
  in the `tenant_keks` table with status `retired` for exactly this purpose.

- **Search attributes use a separate encryption key** derived with a different HKDF info
  parameter (`"temporal-search-attr"` instead of `"temporal-payload"`). This is because search
  attributes are indexed by Temporal's visibility store and may have different access patterns.

### Key Derivation

```go
package temporal

import (
    "context"
    "crypto/sha256"
    "sync"
    "time"

    "github.com/google/uuid"
    "golang.org/x/crypto/hkdf"
    commonpb "go.temporal.io/api/common/v1"
    "go.temporal.io/sdk/converter"
)

// TenantDataConverter encrypts Temporal payloads with per-namespace keys
// derived from the tenant's KEK via HKDF-SHA256.
type TenantDataConverter struct {
    parent    converter.DataConverter
    vaultSvc  VaultKeyService
    keyCache  *dekCache
}

// VaultKeyService provides access to tenant KEKs for key derivation.
// This is a narrow interface -- only what the DataConverter needs.
type VaultKeyService interface {
    // GetActiveKEK returns the active KEK for a tenant, unwrapped via MEK.
    // The returned bytes are on mlock'd pages and must be wiped after use.
    GetActiveKEK(ctx context.Context, tenantID uuid.UUID) (plaintextKEK []byte, version int, err error)

    // GetKEKByVersion returns a specific KEK version (for replaying old payloads).
    GetKEKByVersion(ctx context.Context, tenantID uuid.UUID, version int) (plaintextKEK []byte, err error)
}

// dekCacheEntry holds a derived DEK with its expiry time.
type dekCacheEntry struct {
    dek        []byte // 32-byte AES-256 key
    kekVersion int
    expiresAt  time.Time
}

// dekCache provides short-lived in-memory caching of derived DEKs
// to avoid vault round-trips on every Temporal activity.
type dekCache struct {
    mu      sync.RWMutex
    entries map[string]*dekCacheEntry // key: namespace + info
    ttl     time.Duration             // 5 minutes
}

const (
    dekCacheTTL = 5 * time.Minute

    // HKDF info parameters -- distinct per key purpose.
    hkdfInfoPayload    = "loom-temporal-payload-v1"
    hkdfInfoSearchAttr = "loom-temporal-search-attr-v1"
)

// NewTenantDataConverter creates a DataConverter that encrypts per namespace.
func NewTenantDataConverter(parent converter.DataConverter, vault VaultKeyService) *TenantDataConverter {
    return &TenantDataConverter{
        parent:   parent,
        vaultSvc: vault,
        keyCache: &dekCache{
            entries: make(map[string]*dekCacheEntry),
            ttl:     dekCacheTTL,
        },
    }
}

// deriveKey derives a 256-bit AES key from a tenant KEK using HKDF-SHA256.
// The namespace is used as the salt, and the info parameter distinguishes
// payload keys from search attribute keys.
func deriveKey(kek []byte, namespace string, info string) ([]byte, error) {
    salt := []byte(namespace)
    reader := hkdf.New(sha256.New, kek, salt, []byte(info))
    derived := make([]byte, 32) // AES-256
    if _, err := reader.Read(derived); err != nil {
        return nil, err
    }
    return derived, nil
}

// getOrDeriveKey returns a cached DEK or derives a new one.
func (c *TenantDataConverter) getOrDeriveKey(ctx context.Context, namespace string, info string) ([]byte, int, error) {
    cacheKey := namespace + ":" + info

    // Check cache first.
    c.keyCache.mu.RLock()
    if entry, ok := c.keyCache.entries[cacheKey]; ok && time.Now().Before(entry.expiresAt) {
        dek := make([]byte, len(entry.dek))
        copy(dek, entry.dek)
        version := entry.kekVersion
        c.keyCache.mu.RUnlock()
        return dek, version, nil
    }
    c.keyCache.mu.RUnlock()

    // Cache miss -- derive from KEK.
    tenantID, err := tenantIDFromNamespace(namespace)
    if err != nil {
        return nil, 0, err
    }

    kek, version, err := c.vaultSvc.GetActiveKEK(ctx, tenantID)
    if err != nil {
        return nil, 0, err
    }
    defer wipeBytes(kek)

    dek, err := deriveKey(kek, namespace, info)
    if err != nil {
        return nil, 0, err
    }

    // Cache the derived key.
    c.keyCache.mu.Lock()
    c.keyCache.entries[cacheKey] = &dekCacheEntry{
        dek:        dek,
        kekVersion: version,
        expiresAt:  time.Now().Add(c.keyCache.ttl),
    }
    c.keyCache.mu.Unlock()

    result := make([]byte, len(dek))
    copy(result, dek)
    return result, version, nil
}

// deriveKeyForVersion derives a DEK from a specific KEK version (for replay).
func (c *TenantDataConverter) deriveKeyForVersion(ctx context.Context, namespace string, info string, kekVersion int) ([]byte, error) {
    tenantID, err := tenantIDFromNamespace(namespace)
    if err != nil {
        return nil, err
    }

    kek, err := c.vaultSvc.GetKEKByVersion(ctx, tenantID, kekVersion)
    if err != nil {
        return nil, err
    }
    defer wipeBytes(kek)

    return deriveKey(kek, namespace, info)
}

// EncryptedPayloadHeader is prepended to every encrypted Temporal payload.
// It allows the DataConverter to select the correct decryption key on replay.
type EncryptedPayloadHeader struct {
    Version    int    `json:"v"`   // envelope format version (currently 1)
    KEKVersion int    `json:"kv"`  // which KEK version was used for derivation
    Nonce      []byte `json:"n"`   // AES-GCM 96-bit nonce
    Info       string `json:"i"`   // HKDF info parameter used
}

// tenantIDFromNamespace extracts the tenant UUID from a Temporal namespace.
// Namespace format: "loom-{tenant_uuid}"
func tenantIDFromNamespace(namespace string) (uuid.UUID, error) {
    const prefix = "loom-"
    if len(namespace) <= len(prefix) {
        return uuid.Nil, ErrInvalidNamespace
    }
    return uuid.Parse(namespace[len(prefix):])
}

// wipeBytes overwrites a byte slice with zeros using a constant-time
// operation that the compiler cannot optimize away.
func wipeBytes(b []byte) {
    for i := range b {
        b[i] = 0
    }
    // Compiler barrier: use the slice so the zero writes are not elided.
    _ = b[0]
}
```

### Key Rotation

When a tenant's KEK rotates:

1. The new KEK is generated and stored as `active` in `tenant_keks`.
2. The old KEK is marked `retired` but NOT deleted.
3. New Temporal payloads are encrypted with DEKs derived from the new KEK.
4. Old Temporal payloads (in workflow history) remain decryptable because:
   - Each payload's `EncryptedPayloadHeader` records the `KEKVersion` used.
   - On replay, the DataConverter retrieves the retired KEK by version.
   - The retired KEK derives the same DEK (HKDF is deterministic).

### Retention of Old KEKs

Old KEK versions are retained for: `maximum_workflow_retention_period + 30 days`.

The default Temporal namespace retention is 30 days (configurable per tenant). With the
30-day buffer, old KEKs are retained for 60 days minimum. Tenants with longer retention
(e.g., compliance: 7 years) retain their KEKs accordingly. The `tenant_keks` table tracks
`retired_at` and `retain_until` timestamps. A background job purges KEKs past their
retention window by overwriting the encrypted key material with random bytes and then
deleting the row.

```sql
-- Add retention tracking to existing tenant_keks table.
ALTER TABLE tenant_keks
    ADD COLUMN retain_until TIMESTAMPTZ,
    ADD COLUMN purged_at    TIMESTAMPTZ;

-- When a KEK is retired, set retain_until.
-- retain_until = NOW() + workflow_retention_period + 30 days
```

---

## 3. LLM Cross-Tenant Context Isolation (H11)

### Decision

Stateless single-turn calls only. No conversation history is maintained between
LLM requests, regardless of tenant. Multi-turn conversation is NOT supported and
is NOT needed -- LOOM's LLM use cases (classification, placement, config suggestion,
oversight, cost optimization) are all single-turn request/response patterns.

LOOM supports a "sensitive mode" deployment option that disables cloud LLM providers
and requires local inference (vLLM, Ollama, or llama.cpp).

RAG retrieval is partitioned by tenant using pgvector with PostgreSQL RLS -- not
separate collections. This provides cryptographic-grade isolation using the same
RLS mechanism as all other LOOM data.

### Go Types

```go
package llm

import (
    "context"
    "time"

    "github.com/google/uuid"
)

// LLMRequest is a single-turn, stateless LLM request.
// Each request is independent -- no session, no history, no context carryover.
type LLMRequest struct {
    // RequestID is a unique identifier for this request (for audit trail).
    RequestID uuid.UUID `json:"request_id"`

    // TenantID scopes all data in this request to a single tenant.
    TenantID uuid.UUID `json:"tenant_id"`

    // SystemPrompt is generic and tenant-agnostic. It is loaded from versioned
    // configuration and NEVER contains tenant-specific data.
    SystemPrompt string `json:"system_prompt"`

    // UserMessage contains tenant-scoped data. This is the ONLY place
    // tenant data appears in the LLM request.
    UserMessage string `json:"user_message"`

    // ExpectedResponseType constrains what the LLM should return.
    // Used for schema validation of the response.
    ExpectedResponseType string `json:"expected_response_type"`

    // Temperature controls randomness. Default 0.1 for deterministic outputs.
    Temperature float64 `json:"temperature"`

    // MaxTokens limits response length.
    MaxTokens int `json:"max_tokens"`

    // CorrelationID for tracing through the system.
    CorrelationID string `json:"correlation_id"`
}

// LLMResponse is the validated, typed response from a single-turn LLM call.
type LLMResponse struct {
    // RequestID matches the originating request.
    RequestID uuid.UUID `json:"request_id"`

    // TenantID is echoed back for verification.
    TenantID uuid.UUID `json:"tenant_id"`

    // RawResponse is the LLM's raw text output (before parsing).
    // Stored for audit but NOT returned to the caller.
    RawResponse string `json:"-"`

    // ParsedResponse is the typed, validated response struct.
    // This is the only field the caller uses.
    ParsedResponse any `json:"parsed_response"`

    // Confidence is the LLM's self-reported confidence (0.0 to 1.0).
    Confidence float64 `json:"confidence"`

    // ModelID identifies which model produced this response.
    ModelID string `json:"model_id"`

    // LatencyMs is the round-trip time for the LLM call.
    LatencyMs int64 `json:"latency_ms"`

    // UsedFallback indicates the deterministic fallback was used
    // instead of the LLM (LLM unavailable or low confidence).
    UsedFallback bool `json:"used_fallback"`
}

// LLMClient enforces single-turn, tenant-isolated LLM calls.
type LLMClient interface {
    // Call executes a single-turn LLM request. No session state is
    // carried between calls. Each call creates a new inference session.
    //
    // The implementation MUST:
    // 1. Create a fresh HTTP connection or inference session (no reuse between tenants).
    // 2. Include only the system prompt and user message (no history).
    // 3. Validate the response against the expected schema.
    // 4. Return UsedFallback=true if the LLM is unavailable or confidence < threshold.
    Call(ctx context.Context, req LLMRequest) (*LLMResponse, error)
}

// SensitiveModeConfig controls whether cloud LLM providers are allowed.
type SensitiveModeConfig struct {
    // Enabled disables all cloud LLM providers. Only local inference is allowed.
    Enabled bool `yaml:"enabled" json:"enabled"`

    // AllowedProviders is the list of allowed LLM provider types when
    // sensitive mode is enabled. Only "local" providers are permitted.
    AllowedProviders []string `yaml:"allowed_providers" json:"allowed_providers"`
    // Valid values: "vllm", "ollama", "llamacpp"
    // Invalid (rejected when sensitive mode is on): "openai", "anthropic", "azure_openai"
}
```

### RAG Tenant Isolation

```sql
-- pgvector embeddings table with tenant isolation via RLS.
CREATE TABLE rag_embeddings (
    id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id   UUID NOT NULL REFERENCES tenants(id),
    document_id UUID NOT NULL,
    chunk_index INT NOT NULL,
    content     TEXT NOT NULL,
    embedding   vector(1536) NOT NULL,  -- dimension matches embedding model
    metadata    JSONB DEFAULT '{}',
    created_at  TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- RLS enforcement: same mechanism as all other LOOM tables.
ALTER TABLE rag_embeddings ENABLE ROW LEVEL SECURITY;

CREATE POLICY tenant_isolation ON rag_embeddings
    USING (tenant_id = current_setting('app.current_tenant_id')::uuid);

-- Similarity search index (per-tenant via partial index is not needed;
-- RLS filters results automatically).
CREATE INDEX idx_rag_embedding_vector ON rag_embeddings
    USING ivfflat (embedding vector_cosine_ops) WITH (lists = 100);

CREATE INDEX idx_rag_tenant ON rag_embeddings (tenant_id);
```

### Enforcement Rules

1. **No session reuse.** Each `LLMClient.Call()` creates a new HTTP request. No
   `session_id`, no `conversation_id`, no `thread_id` is ever sent to the LLM provider.

2. **System prompt is immutable per version.** System prompts are loaded from a
   versioned config file (`llm_prompts_v{N}.yaml`). They contain instructions about
   output format and domain knowledge. They NEVER contain tenant names, device IPs,
   credentials, or any tenant-specific data.

3. **Tenant data only in user message.** The user message is constructed at call time
   from tenant-scoped data (device metadata, observed facts, etc.). It is discarded
   after the call completes.

4. **Response validation.** Every LLM response is parsed into a typed Go struct
   (`LLMRecommendation` from LLM-BOUNDARIES.md) and validated against the same
   pipeline as human input. Responses that fail validation are discarded and the
   deterministic fallback is used.

5. **vLLM shared KV cache risk is accepted.** When using a shared vLLM instance,
   there is a theoretical risk of cross-tenant information leakage through continuous
   batching. This risk is mitigated by: (a) single-turn calls eliminate conversation
   context, (b) system prompts are tenant-agnostic, (c) response validation prevents
   cross-tenant data from being used. For deployments where this risk is unacceptable,
   sensitive mode forces local inference with a dedicated model instance.

---

## 4. Break-Glass Token Lifecycle (H12)

### Decision

- **TTL: 15 minutes** (standardize on the shorter value from SECURITY-MODEL.md).
- **Generation: on-demand** (not pre-generated). Pre-generated sealed tokens exist
  only for the air-gapped physical-safe procedure (separate flow).
- **Dual authorization: required**. Single-admin override is allowed ONLY with an
  `emergency` flag, extended audit logging, and mandatory post-incident review
  within 24 hours.

### Complete Token Specification

```go
package breakglass

import (
    "context"
    "crypto/ed25519"
    "time"

    "github.com/google/uuid"
)

// BreakGlassToken is a time-limited emergency access token that bypasses
// normal RBAC. It requires dual authorization and produces enhanced audit.
type BreakGlassToken struct {
    // JTI is the unique token identifier (JWT ID). Used for replay prevention.
    JTI string `json:"jti"`

    // IssuedAt is when the token was created.
    IssuedAt time.Time `json:"iat"`

    // ExpiresAt is IssuedAt + 15 minutes. Non-negotiable.
    ExpiresAt time.Time `json:"exp"`

    // InitiatedBy is the admin who started the break-glass ceremony.
    InitiatedBy AdminIdentity `json:"initiated_by"`

    // ApprovedBy is the second admin who co-signed. Required unless Emergency is true.
    ApprovedBy *AdminIdentity `json:"approved_by,omitempty"`

    // Emergency indicates single-admin override mode. When true:
    // - ApprovedBy is nil
    // - Extended audit logging is enabled (every action logged at CRITICAL)
    // - A mandatory post-incident review task is created automatically
    // - The token scope is further restricted (no tenant.manage actions)
    Emergency bool `json:"emergency"`

    // Scope restricts what this token can do.
    Scope BreakGlassScope `json:"scope"`

    // Used indicates the token has been consumed. Single-use enforcement.
    Used   bool       `json:"used"`
    UsedAt *time.Time `json:"used_at,omitempty"`

    // Signature is the Ed25519 signature over the token body.
    // Signed with the break-glass signing key (separate from the JWT signing key).
    Signature []byte `json:"sig"`
}

// AdminIdentity identifies an admin participating in the break-glass ceremony.
type AdminIdentity struct {
    UserID      string `json:"user_id"`      // from OIDC subject claim
    DisplayName string `json:"display_name"`
    IPAddress   string `json:"ip_address"`   // source IP at ceremony time
    UserAgent   string `json:"user_agent"`
}

// BreakGlassScope defines the permitted actions for a break-glass token.
type BreakGlassScope struct {
    // Actions is the explicit list of permitted actions.
    // Never "*". Examples: "vault.unseal", "credential.read", "device.power_cycle".
    Actions []string `json:"actions"`

    // TenantIDs scopes the token to specific tenants. Never empty, never "*".
    TenantIDs []uuid.UUID `json:"tenant_ids"`

    // DeviceIDs optionally restricts to specific devices. Empty means all
    // devices within the scoped tenants.
    DeviceIDs []uuid.UUID `json:"device_ids,omitempty"`

    // MaxUses is the maximum number of times this token can be used.
    // Default 1 (single-use). Maximum 10 (for batch emergency operations).
    MaxUses int `json:"max_uses"`

    // UseCount tracks how many times the token has been used.
    UseCount int `json:"use_count"`
}

// BreakGlassSigningKey is the Ed25519 key pair used exclusively for signing
// break-glass tokens. This key is separate from the JWT signing key and
// is stored in the vault (encrypted by MEK).
type BreakGlassSigningKey struct {
    PublicKey  ed25519.PublicKey  `json:"public_key"`
    PrivateKey ed25519.PrivateKey `json:"-"` // never serialized
    KeyID      string             `json:"key_id"`
    CreatedAt  time.Time          `json:"created_at"`
    RotatedAt  *time.Time         `json:"rotated_at,omitempty"`
}
```

### JTI Replay Detection

JTI replay detection uses PostgreSQL, NOT Valkey. The break-glass system must
function even during a Valkey outage (which might be the reason break-glass
is needed in the first place).

```sql
-- Break-glass token replay prevention.
-- PostgreSQL table, NOT Valkey. Must survive infrastructure failures.
CREATE TABLE break_glass_tokens (
    jti             TEXT PRIMARY KEY,
    initiated_by    TEXT NOT NULL,
    approved_by     TEXT,             -- NULL for emergency single-admin
    emergency       BOOLEAN NOT NULL DEFAULT FALSE,
    scope           JSONB NOT NULL,
    issued_at       TIMESTAMPTZ NOT NULL,
    expires_at      TIMESTAMPTZ NOT NULL,
    used            BOOLEAN NOT NULL DEFAULT FALSE,
    used_at         TIMESTAMPTZ,
    use_count       INT NOT NULL DEFAULT 0,
    max_uses        INT NOT NULL DEFAULT 1,
    revoked         BOOLEAN NOT NULL DEFAULT FALSE,
    revoked_at      TIMESTAMPTZ,
    revoked_by      TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- Index for fast JTI lookup during token validation.
CREATE INDEX idx_bgt_jti_expires ON break_glass_tokens (jti, expires_at);

-- Automatic cleanup: partition by month or TTL-based deletion.
-- Tokens are retained for 90 days for audit, then purged.
CREATE INDEX idx_bgt_created ON break_glass_tokens (created_at);
```

### Ceremony Flow

```
Step 1: Initiation
  Admin A authenticates via OIDC and calls POST /api/v1/break-glass/initiate
  Body: { "scope": { "actions": [...], "tenant_ids": [...] }, "reason": "..." }
  Result: ceremony_id (UUID) created, status = "pending_approval"
  Audit: "break_glass.initiated" at CRITICAL severity

Step 2a: Co-Signing (normal flow)
  Admin B (different user, different session) authenticates and calls
  POST /api/v1/break-glass/{ceremony_id}/approve
  Body: { "approved": true, "reason": "..." }
  The system verifies Admin B != Admin A (same user cannot self-approve).
  Result: BreakGlassToken issued, signed with break-glass signing key
  Audit: "break_glass.approved" at CRITICAL severity

Step 2b: Emergency Override (single-admin flow)
  Admin A calls POST /api/v1/break-glass/initiate with "emergency": true
  The system issues the token immediately with restricted scope:
    - No "tenant.manage" actions allowed
    - MaxUses capped at 1
  A post-incident review task is created automatically in the task system.
  Audit: "break_glass.emergency_override" at CRITICAL severity
  Alert: immediate page to all other admins

Step 3: Token Use
  Admin presents the break-glass token in the Authorization header:
    Authorization: BreakGlass <token>
  The middleware:
    1. Verifies the Ed25519 signature using the break-glass public key
    2. Checks JTI against PostgreSQL (replay prevention)
    3. Checks expiry (15-minute TTL)
    4. Checks use_count < max_uses
    5. Verifies the requested action is in scope
    6. Increments use_count in PostgreSQL
    7. Produces an audit record for every action taken
  Audit: "break_glass.used" at CRITICAL severity for every action

Step 4: Post-Incident Review (mandatory for emergency overrides)
  Within 24 hours, the admin who used the emergency override must complete
  a post-incident review:
    POST /api/v1/break-glass/{ceremony_id}/review
    Body: { "summary": "...", "root_cause": "...", "corrective_actions": "..." }
  If the review is not completed within 24 hours:
    - Alert escalation to all admins
    - The admin's account is flagged for review
  Audit: "break_glass.review_completed" or "break_glass.review_overdue"
```

### Air-Gapped Sealed Token Procedure

For air-gapped deployments (Mode 4) where the OIDC provider may be unreachable:

1. During initial deployment, generate N sealed break-glass tokens (default: 3).
2. Each token is encrypted with a passphrase using Argon2id + AES-256-GCM.
3. The encrypted tokens are printed and placed in a physical safe.
4. Each token has a 30-day TTL from the moment it is unsealed (not from generation).
5. To use: enter the passphrase to decrypt the token, which starts the 30-day clock.
6. These tokens are strictly supplementary to the on-demand flow and are
   intended for catastrophic infrastructure failure scenarios only.

---

## 5. Audit Trail Integrity (H13)

### Decision

BLAKE3 hash chain with external anchoring to S3 Object Lock. Every audit record
includes a sequence number, previous hash, and record hash, forming a tamper-evident
chain per tenant. Integrity verification runs daily (full chain) and hourly
(tail verification of the last 1000 records).

S3 Object Lock is sufficient as the external anchor. An independent RFC 3161
timestamping authority is NOT required -- S3 Object Lock provides immutable,
time-stamped storage that satisfies compliance requirements for SOC 2, ISO 27001,
and PCI DSS. If a specific tenant requires RFC 3161, it can be added as an
optional anchor target.

### Schema Changes

```sql
-- Add integrity chain fields to audit_records table.
ALTER TABLE audit_records
    ADD COLUMN sequence_num   BIGINT,
    ADD COLUMN previous_hash  TEXT,
    ADD COLUMN record_hash    TEXT;

-- Sequence number is monotonically increasing per tenant.
-- Using a sequence per tenant ensures independent chains.
CREATE SEQUENCE audit_seq_tenant OWNED BY NONE;

-- Unique constraint: one sequence number per tenant.
CREATE UNIQUE INDEX idx_audit_tenant_seq ON audit_records (tenant_id, sequence_num);

-- Index for chain verification queries.
CREATE INDEX idx_audit_chain ON audit_records (tenant_id, sequence_num ASC);

-- External anchor records.
CREATE TABLE audit_anchors (
    id               UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id        TEXT NOT NULL,
    anchor_type      TEXT NOT NULL CHECK (anchor_type IN ('s3_object_lock', 'rfc3161')),
    first_seq        BIGINT NOT NULL,
    last_seq         BIGINT NOT NULL,
    record_count     INT NOT NULL,
    merkle_root      TEXT NOT NULL,       -- hex-encoded BLAKE3 Merkle root
    signature        BYTEA NOT NULL,      -- Ed25519 signature of merkle_root
    signing_key_id   TEXT NOT NULL,
    s3_bucket        TEXT,
    s3_key           TEXT,
    s3_version_id    TEXT,
    anchored_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    verified_at      TIMESTAMPTZ,         -- last successful verification
    verification_ok  BOOLEAN
);

CREATE INDEX idx_anchor_tenant ON audit_anchors (tenant_id, last_seq DESC);
```

### Updated AuditRecord Struct

```go
package audit

import (
    "encoding/hex"
    "strconv"
    "time"

    "github.com/google/uuid"
    "github.com/zeebo/blake3"
)

// AuditRecord is the immutable record of a single action in LOOM.
// Extended with integrity chain fields (H13).
type AuditRecord struct {
    ID            string            `json:"id" db:"id"`
    Timestamp     time.Time         `json:"timestamp" db:"timestamp"`
    TenantID      string            `json:"tenant_id" db:"tenant_id"`
    ActorType     string            `json:"actor_type" db:"actor_type"`
    ActorID       string            `json:"actor_id" db:"actor_id"`
    Action        string            `json:"action" db:"action"`
    ResourceType  string            `json:"resource_type" db:"resource_type"`
    ResourceID    string            `json:"resource_id" db:"resource_id"`
    BeforeState   any               `json:"before_state" db:"before_state"`
    AfterState    any               `json:"after_state" db:"after_state"`
    CorrelationID string            `json:"correlation_id" db:"correlation_id"`
    CausationID   string            `json:"causation_id" db:"causation_id"`
    Outcome       string            `json:"outcome" db:"outcome"`
    Metadata      map[string]string `json:"metadata" db:"metadata"`

    // Integrity chain fields (H13).
    SequenceNum  int64  `json:"sequence_num" db:"sequence_num"`
    PreviousHash string `json:"previous_hash" db:"previous_hash"`
    RecordHash   string `json:"record_hash" db:"record_hash"`
}

// ComputeRecordHash computes the BLAKE3 hash of an audit record.
// The hash covers all fields that constitute the record's identity and content.
// RecordHash itself is excluded (it is the output of this function).
//
// Field order is fixed and must never change. Adding new fields requires
// appending them to the end of the hash input.
func ComputeRecordHash(record AuditRecord, previousHash string) string {
    h := blake3.New()

    // Fixed-order field hashing. Order must never change.
    h.Write([]byte(record.TenantID))
    h.Write([]byte{0}) // separator
    h.Write([]byte(strconv.FormatInt(record.SequenceNum, 10)))
    h.Write([]byte{0})
    h.Write([]byte(record.ID))
    h.Write([]byte{0})
    h.Write([]byte(record.Timestamp.Format(time.RFC3339Nano)))
    h.Write([]byte{0})
    h.Write([]byte(record.ActorType))
    h.Write([]byte{0})
    h.Write([]byte(record.ActorID))
    h.Write([]byte{0})
    h.Write([]byte(record.Action))
    h.Write([]byte{0})
    h.Write([]byte(record.ResourceType))
    h.Write([]byte{0})
    h.Write([]byte(record.ResourceID))
    h.Write([]byte{0})
    h.Write([]byte(record.CorrelationID))
    h.Write([]byte{0})
    h.Write([]byte(record.CausationID))
    h.Write([]byte{0})
    h.Write([]byte(record.Outcome))
    h.Write([]byte{0})
    h.Write([]byte(previousHash))

    return hex.EncodeToString(h.Sum(nil))
}

// VerifyAuditChain verifies the integrity of the audit hash chain for a tenant.
// Returns nil if the chain is intact, or an error describing the first break.
func VerifyAuditChain(ctx context.Context, db AuditStore, tenantID string, fromSeq, toSeq int64) error {
    const batchSize = 1000
    previousHash := ""

    // If fromSeq > 0, we need the hash of the record before fromSeq to verify the chain.
    if fromSeq > 0 {
        prev, err := db.GetAuditRecord(ctx, tenantID, fromSeq-1)
        if err != nil {
            return fmt.Errorf("cannot retrieve record at seq %d: %w", fromSeq-1, err)
        }
        previousHash = prev.RecordHash
    }

    for seq := fromSeq; seq <= toSeq; seq += int64(batchSize) {
        endSeq := seq + int64(batchSize) - 1
        if endSeq > toSeq {
            endSeq = toSeq
        }

        records, err := db.GetAuditRecordRange(ctx, tenantID, seq, endSeq)
        if err != nil {
            return fmt.Errorf("failed to fetch records [%d, %d]: %w", seq, endSeq, err)
        }

        for _, record := range records {
            expected := ComputeRecordHash(record, previousHash)
            if record.RecordHash != expected {
                return &AuditChainError{
                    TenantID:      tenantID,
                    SequenceNum:   record.SequenceNum,
                    ExpectedHash:  expected,
                    ActualHash:    record.RecordHash,
                    PreviousHash:  previousHash,
                }
            }
            previousHash = record.RecordHash
        }
    }

    return nil
}

// AuditChainError indicates a break in the audit hash chain.
type AuditChainError struct {
    TenantID     string
    SequenceNum  int64
    ExpectedHash string
    ActualHash   string
    PreviousHash string
}

func (e *AuditChainError) Error() string {
    return fmt.Sprintf(
        "audit chain broken at tenant=%s seq=%d: expected=%s actual=%s",
        e.TenantID, e.SequenceNum, e.ExpectedHash, e.ActualHash,
    )
}

// AuditStore is the interface for audit record persistence.
type AuditStore interface {
    GetAuditRecord(ctx context.Context, tenantID string, seq int64) (*AuditRecord, error)
    GetAuditRecordRange(ctx context.Context, tenantID string, fromSeq, toSeq int64) ([]AuditRecord, error)
    GetLatestSequenceNum(ctx context.Context, tenantID string) (int64, error)
}
```

### External Anchoring

```go
package audit

import (
    "crypto/ed25519"
    "encoding/hex"

    "github.com/zeebo/blake3"
)

// AnchorConfig defines external anchoring configuration.
type AnchorConfig struct {
    // Enabled turns on external anchoring.
    Enabled bool `yaml:"enabled" default:"true"`

    // Interval is the anchoring frequency.
    // Anchor every N records OR every Duration, whichever comes first.
    IntervalRecords int           `yaml:"interval_records" default:"1000"`
    IntervalTime    time.Duration `yaml:"interval_time" default:"1h"`

    // S3 configuration for Object Lock storage.
    S3Bucket     string `yaml:"s3_bucket"`
    S3Region     string `yaml:"s3_region"`
    S3Endpoint   string `yaml:"s3_endpoint"`   // for MinIO in air-gapped
    S3ObjectLock bool   `yaml:"s3_object_lock"` // must be true for compliance

    // SigningKeyID identifies the Ed25519 key used to sign anchors.
    SigningKeyID string `yaml:"signing_key_id"`
}

// ComputeMerkleRoot builds a Merkle tree from record hashes and returns the root.
// Uses BLAKE3 for all internal nodes.
func ComputeMerkleRoot(recordHashes []string) string {
    if len(recordHashes) == 0 {
        return ""
    }
    if len(recordHashes) == 1 {
        return recordHashes[0]
    }

    // Pad to power of 2 by duplicating the last hash.
    for len(recordHashes)&(len(recordHashes)-1) != 0 {
        recordHashes = append(recordHashes, recordHashes[len(recordHashes)-1])
    }

    level := recordHashes
    for len(level) > 1 {
        var nextLevel []string
        for i := 0; i < len(level); i += 2 {
            h := blake3.New()
            left, _ := hex.DecodeString(level[i])
            right, _ := hex.DecodeString(level[i+1])
            h.Write(left)
            h.Write(right)
            nextLevel = append(nextLevel, hex.EncodeToString(h.Sum(nil)))
        }
        level = nextLevel
    }

    return level[0]
}

// SignAnchor signs a Merkle root with the audit signing key.
func SignAnchor(merkleRoot string, privateKey ed25519.PrivateKey) []byte {
    rootBytes, _ := hex.DecodeString(merkleRoot)
    return ed25519.Sign(privateKey, rootBytes)
}

// VerifyAnchorSignature verifies an anchor's Ed25519 signature.
func VerifyAnchorSignature(merkleRoot string, signature []byte, publicKey ed25519.PublicKey) bool {
    rootBytes, _ := hex.DecodeString(merkleRoot)
    return ed25519.Verify(publicKey, rootBytes, signature)
}
```

### Verification Schedule

| Verification Type | Frequency | Scope | Action on Failure |
|-------------------|-----------|-------|-------------------|
| Tail verification | Hourly | Last 1000 records per tenant | CRITICAL alert, incident created |
| Full chain verification | Daily (02:00 UTC) | All records per tenant | CRITICAL alert, incident created, anchor comparison |
| Anchor verification | Weekly | Compare DB chain against S3 anchors | CRITICAL alert, forensic investigation |

Full chain verification scales linearly with audit record volume. At 90 days of hot
storage with an estimated 100,000 records per tenant, full verification takes approximately
2-5 seconds per tenant (BLAKE3 hashes at ~1GB/s on modern hardware with AVX2). For tenants
with millions of records, verification is parallelized across sequence ranges.

---

## 6. ConfigSnapshot Encryption (H14)

### Decision

Encrypt ConfigSnapshot data with the tenant's KEK before storage. Credential scrubbing
is applied as defense-in-depth BEFORE encryption (belt and suspenders). Snapshots are
NOT stored in the vault -- they are encrypted and stored in Temporal workflow history
(via the TenantDataConverter from H10) and optionally in the database for point-in-time
rollback.

The Temporal `TenantDataConverter` (H10) handles encryption transparently for
workflow-stored snapshots. Database-stored snapshots use the same envelope encryption
pattern.

### Updated ConfigSnapshot Struct

```go
package adapter

import (
    "crypto/aes"
    "crypto/cipher"
    "crypto/rand"
    "regexp"
    "time"

    "github.com/google/uuid"
)

// ConfigSnapshot is the captured device configuration state.
// Data is always encrypted at rest. Credentials are scrubbed before encryption.
type ConfigSnapshot struct {
    // AdapterName identifies which adapter produced this snapshot.
    AdapterName string `json:"adapter_name" db:"adapter_name"`

    // DeviceID is the LOOM internal device ID.
    DeviceID uuid.UUID `json:"device_id" db:"device_id"`

    // TenantID scopes this snapshot to a tenant.
    TenantID uuid.UUID `json:"tenant_id" db:"tenant_id"`

    // Scope defines what was captured.
    Scope SnapshotScope `json:"scope" db:"scope"`

    // Format describes the original plaintext format.
    Format string `json:"format" db:"format"` // "text/plain", "application/json", "application/xml"

    // EncryptedData is the AES-256-GCM encrypted, credential-scrubbed config.
    EncryptedData []byte `json:"-" db:"encrypted_data"`

    // DEKWrapped is the per-snapshot DEK, wrapped by the tenant's KEK.
    DEKWrapped []byte `json:"-" db:"dek_wrapped"`

    // Nonce is the AES-GCM 96-bit nonce for this encryption.
    Nonce []byte `json:"-" db:"nonce"`

    // KEKVersion identifies which KEK version wrapped the DEK.
    KEKVersion int `json:"kek_version" db:"kek_version"`

    // Checksum is the BLAKE3 hash of the PLAINTEXT config (before scrubbing).
    // Used for change detection between snapshots without decryption.
    Checksum string `json:"checksum" db:"checksum"`

    // ScrubChecksum is the BLAKE3 hash of the scrubbed config (after scrubbing,
    // before encryption). Used for integrity verification after decryption.
    ScrubChecksum string `json:"scrub_checksum" db:"scrub_checksum"`

    // SizeBytes is the plaintext size (before encryption).
    SizeBytes int `json:"size_bytes" db:"size_bytes"`

    // CapturedAt is when the snapshot was taken.
    CapturedAt time.Time `json:"captured_at" db:"captured_at"`

    // ExpiresAt is when this snapshot should be purged. Default: CapturedAt + 90 days.
    ExpiresAt time.Time `json:"expires_at" db:"expires_at"`
}
```

### Credential Scrubbing

Credential scrubbing is defense-in-depth. Even if the encryption is somehow bypassed,
the stored config does not contain plaintext credentials.

```go
package adapter

import (
    "regexp"
    "strings"
)

// credentialPatterns defines regex patterns for credentials in device configs.
// Each pattern captures a prefix group and replaces the secret value with [SCRUBBED].
// Patterns are ordered from most specific to least specific.
var credentialPatterns = []*credentialPattern{
    // Cisco IOS / IOS-XE / NX-OS
    {regexp.MustCompile(`(?im)((?:enable|username\s+\S+)\s+(?:password|secret)\s+\d\s+)\S+`), "${1}[SCRUBBED]"},
    {regexp.MustCompile(`(?im)(snmp-server\s+community\s+)\S+`), "${1}[SCRUBBED]"},
    {regexp.MustCompile(`(?im)(crypto\s+isakmp\s+key\s+)\S+`), "${1}[SCRUBBED]"},
    {regexp.MustCompile(`(?im)(ip\s+ospf\s+authentication-key\s+)\S+`), "${1}[SCRUBBED]"},
    {regexp.MustCompile(`(?im)(neighbor\s+\S+\s+password\s+)\S+`), "${1}[SCRUBBED]"},

    // Junos
    {regexp.MustCompile(`(?im)(encrypted-password\s+")\S+(";)`), "${1}[SCRUBBED]${2}"},
    {regexp.MustCompile(`(?im)(pre-shared-key\s+(?:ascii-text|hexadecimal)\s+")\S+(";)`), "${1}[SCRUBBED]${2}"},
    {regexp.MustCompile(`(?im)(community\s+")\S+(";)`), "${1}[SCRUBBED]${2}"},
    {regexp.MustCompile(`(?im)(authentication-key\s+")\S+(";)`), "${1}[SCRUBBED]${2}"},

    // Arista EOS
    {regexp.MustCompile(`(?im)(secret\s+sha512\s+)\S+`), "${1}[SCRUBBED]"},

    // Generic patterns (catch-all, applied last)
    {regexp.MustCompile(`(?im)((?:password|secret|key|community|pre-shared-key|auth-password|priv-password)\s+)\S+`), "${1}[SCRUBBED]"},
    {regexp.MustCompile(`(?im)((?:password|secret|token|api[_-]?key)\s*[=:]\s*)\S+`), "${1}[SCRUBBED]"},
}

type credentialPattern struct {
    regex       *regexp.Regexp
    replacement string
}

// ScrubCredentials removes credential values from a device configuration.
// Applied before encryption as defense-in-depth.
func ScrubCredentials(config string) string {
    result := config
    for _, p := range credentialPatterns {
        result = p.regex.ReplaceAllString(result, p.replacement)
    }
    return result
}
```

### Encryption Flow

```go
package adapter

import (
    "crypto/aes"
    "crypto/cipher"
    "crypto/rand"
    "time"

    "github.com/google/uuid"
    "github.com/zeebo/blake3"
)

// EncryptSnapshot encrypts a plaintext config snapshot.
// Flow: capture -> checksum -> scrub -> scrub checksum -> generate DEK ->
//       encrypt -> wrap DEK with KEK -> assemble snapshot -> zero plaintext
func EncryptSnapshot(
    plaintext []byte,
    tenantID uuid.UUID,
    deviceID uuid.UUID,
    adapterName string,
    format string,
    scope SnapshotScope,
    kek []byte,       // tenant's active KEK (plaintext, mlock'd)
    kekVersion int,
) (*ConfigSnapshot, error) {
    now := time.Now().UTC()

    // Step 1: Compute checksum of original plaintext.
    checksum := computeBLAKE3(plaintext)

    // Step 2: Scrub credentials.
    scrubbed := []byte(ScrubCredentials(string(plaintext)))
    scrubChecksum := computeBLAKE3(scrubbed)

    // Step 3: Generate per-snapshot DEK (AES-256).
    dek := make([]byte, 32)
    if _, err := rand.Read(dek); err != nil {
        return nil, fmt.Errorf("failed to generate snapshot DEK: %w", err)
    }
    defer wipeBytes(dek)

    // Step 4: Encrypt scrubbed config with DEK (AES-256-GCM).
    block, err := aes.NewCipher(dek)
    if err != nil {
        return nil, fmt.Errorf("failed to create AES cipher: %w", err)
    }
    gcm, err := cipher.NewGCM(block)
    if err != nil {
        return nil, fmt.Errorf("failed to create GCM: %w", err)
    }

    nonce := make([]byte, gcm.NonceSize()) // 12 bytes for AES-GCM
    if _, err := rand.Read(nonce); err != nil {
        return nil, fmt.Errorf("failed to generate nonce: %w", err)
    }

    // AAD binds the ciphertext to this tenant and device.
    aad := []byte(tenantID.String() + ":" + deviceID.String())
    encryptedData := gcm.Seal(nil, nonce, scrubbed, aad)

    // Step 5: Wrap DEK with tenant KEK (AES-256-GCM).
    kekBlock, err := aes.NewCipher(kek)
    if err != nil {
        return nil, fmt.Errorf("failed to create KEK cipher: %w", err)
    }
    kekGCM, err := cipher.NewGCM(kekBlock)
    if err != nil {
        return nil, fmt.Errorf("failed to create KEK GCM: %w", err)
    }
    dekNonce := make([]byte, kekGCM.NonceSize())
    if _, err := rand.Read(dekNonce); err != nil {
        return nil, fmt.Errorf("failed to generate DEK nonce: %w", err)
    }
    dekWrapped := kekGCM.Seal(dekNonce, dekNonce, dek, nil) // prepend nonce to wrapped DEK

    // Step 6: Zero plaintext from memory.
    wipeBytes(scrubbed)
    // Caller is responsible for wiping the original plaintext.

    return &ConfigSnapshot{
        AdapterName:   adapterName,
        DeviceID:      deviceID,
        TenantID:      tenantID,
        Scope:         scope,
        Format:        format,
        EncryptedData: encryptedData,
        DEKWrapped:    dekWrapped,
        Nonce:         nonce,
        KEKVersion:    kekVersion,
        Checksum:      checksum,
        ScrubChecksum: scrubChecksum,
        SizeBytes:     len(plaintext),
        CapturedAt:    now,
        ExpiresAt:     now.Add(90 * 24 * time.Hour), // 90-day retention
    }, nil
}

func computeBLAKE3(data []byte) string {
    h := blake3.New()
    h.Write(data)
    return hex.EncodeToString(h.Sum(nil))
}
```

### Retention Policy

Config snapshots are retained for **90 days**, matching the warm tier
ObservedFact retention defined in AUDIT-MODEL.md. After 90 days:

1. Snapshot records are eligible for purging.
2. A background job deletes expired snapshots weekly.
3. The encrypted data, DEK, and nonce are overwritten with random bytes before deletion.
4. The metadata (checksum, size, captured_at) is retained in the audit trail permanently.

```sql
-- Config snapshot storage (for DB-stored snapshots).
CREATE TABLE config_snapshots (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL,
    device_id       UUID NOT NULL,
    adapter_name    TEXT NOT NULL,
    format          TEXT NOT NULL,
    encrypted_data  BYTEA NOT NULL,
    dek_wrapped     BYTEA NOT NULL,
    nonce           BYTEA NOT NULL,
    kek_version     INT NOT NULL,
    checksum        TEXT NOT NULL,
    scrub_checksum  TEXT NOT NULL,
    size_bytes      INT NOT NULL,
    captured_at     TIMESTAMPTZ NOT NULL,
    expires_at      TIMESTAMPTZ NOT NULL
);

ALTER TABLE config_snapshots ENABLE ROW LEVEL SECURITY;

CREATE POLICY tenant_isolation ON config_snapshots
    USING (tenant_id = current_setting('app.current_tenant_id')::uuid);

CREATE INDEX idx_snapshot_device ON config_snapshots (device_id, captured_at DESC);
CREATE INDEX idx_snapshot_expires ON config_snapshots (expires_at);
```

---

## 7. Valkey Security Posture (H17)

### Decision

Add Valkey to the LOOM security model as a first-class component with explicit controls.
Valkey stores session tokens, cached data, idempotency keys, rate limit counters, and
pub/sub messages for WebSocket fan-out. It is an attack vector and must be hardened.

Session tokens stored in Valkey use opaque JWT format. The JWT is already signed (Ed25519),
so additional encryption in Valkey is not required. The signature provides integrity and
authenticity. Confidentiality is provided by TLS in transit and Valkey's network isolation
at rest.

### Required Controls

| Control | Phase | Configuration |
|---------|-------|---------------|
| TLS 1.3 | 1A | `tls-port 6379`, `port 0` (disable plaintext port) |
| ACL per service | 1A | Separate user per LOOM service (see below) |
| Network isolation | 1A | `bind 10.x.x.x` (internal network only, not `0.0.0.0`) |
| Command restriction | 1A | Disable dangerous commands via ACL |
| Memory limit | 1A | `maxmemory 4gb` (adjust per deployment), `maxmemory-policy allkeys-lru` |
| Key expiry | 1B | All session tokens have TTL matching JWT expiry (15 minutes) |
| Persistence encryption | 5 | RDB/AOF on LUKS-encrypted volume |
| Monitoring | 2 | Alert on auth failures, connection spikes, memory >80% |
| Sentinel HA | 2+ | Valkey Sentinel for Mode 2+ deployments |

### Valkey ACL Configuration

```
# /etc/valkey/users.acl
# Each LOOM service gets a dedicated user with least-privilege access.

# API Server: session management, caching, tenant membership cache
user api-server on >$LOOM_VALKEY_API_PASSWORD ~session:* ~cache:* ~membership:* &pubsub:* +get +set +del +expire +ttl +exists +publish +@read +@write -@admin -@dangerous

# WebSocket: pub/sub for real-time event fan-out
user websocket on >$LOOM_VALKEY_WS_PASSWORD ~pubsub:* +subscribe +unsubscribe +publish +psubscribe +punsubscribe +ping -@admin -@dangerous -@write

# Worker: idempotency keys, rate limiting, distributed locks
user worker on >$LOOM_VALKEY_WORKER_PASSWORD ~idempotency:* ~ratelimit:* ~lock:* +get +set +del +expire +ttl +exists +incr +decr +setnx +pexpire -@admin -@dangerous

# Default user: disabled. No anonymous access.
user default off nopass ~* -@all
```

### Disabled Commands

The following commands are disabled via ACL category `@dangerous` and explicit denial:

| Command | Reason |
|---------|--------|
| `FLUSHALL` | Deletes all data across all databases |
| `FLUSHDB` | Deletes all data in the current database |
| `CONFIG` | Can modify runtime configuration including security settings |
| `DEBUG` | Exposes internal state, can crash the server |
| `KEYS` | O(N) scan, causes performance degradation, use `SCAN` instead |
| `SCRIPT` | Arbitrary Lua execution |
| `EVAL` / `EVALSHA` | Arbitrary Lua execution |
| `MODULE` | Can load arbitrary shared libraries |
| `SHUTDOWN` | Can stop the server |
| `SLAVEOF` / `REPLICAOF` | Can redirect replication |
| `CLUSTER` | Can modify cluster topology |
| `MIGRATE` | Can exfiltrate data to external servers |
| `SAVE` / `BGSAVE` | Uncontrolled disk writes |

### Valkey TLS Configuration

```
# /etc/valkey/valkey.conf (TLS section)

# Disable plaintext port.
port 0

# Enable TLS port.
tls-port 6379

# Server certificate and key.
tls-cert-file /etc/valkey/tls/server.crt
tls-key-file /etc/valkey/tls/server.key

# CA certificate for client verification (mTLS).
tls-ca-cert-file /etc/valkey/tls/ca.crt

# Require client certificates.
tls-auth-clients yes

# TLS 1.3 minimum.
tls-protocols "TLSv1.3"

# Network binding (internal only).
bind 10.0.0.0/8
```

### Attack Surface Analysis -- Valkey

| # | Vector | Description | Defense |
|---|--------|-------------|---------|
| V1 | Unauthenticated access | Default Valkey has no auth | ACL with per-service users, default user disabled |
| V2 | Network exposure | Valkey on public interface | `bind` to internal network only, firewall rules |
| V3 | Plaintext transit | Data intercepted on the wire | TLS 1.3 with mTLS client certs |
| V4 | Command injection | Dangerous commands exfiltrate data | ACL command restrictions, disable `EVAL`/`SCRIPT` |
| V5 | Memory exhaustion | Unbounded growth causes OOM | `maxmemory` limit with LRU eviction policy |
| V6 | Session fixation | Stolen session token replayed | JWT signature verification + TTL + Valkey TTL alignment |
| V7 | Data persistence leak | RDB/AOF files contain plaintext | LUKS encryption on Valkey data volume |
| V8 | Replication hijack | Attacker becomes master via `REPLICAOF` | `REPLICAOF` disabled via ACL, Sentinel controls replication |

### Monitoring Alerts

```yaml
# Prometheus alerting rules for Valkey.
groups:
  - name: valkey_security
    rules:
      - alert: ValkeyAuthFailures
        expr: rate(valkey_rejected_connections_total[5m]) > 5
        for: 2m
        labels:
          severity: warning
        annotations:
          summary: "Valkey authentication failures detected"

      - alert: ValkeyConnectionSpike
        expr: valkey_connected_clients > 200
        for: 1m
        labels:
          severity: warning
        annotations:
          summary: "Unusual number of Valkey connections"

      - alert: ValkeyMemoryHigh
        expr: valkey_memory_used_bytes / valkey_memory_max_bytes > 0.8
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Valkey memory usage above 80%"

      - alert: ValkeyMemoryCritical
        expr: valkey_memory_used_bytes / valkey_memory_max_bytes > 0.95
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "Valkey memory usage above 95%"

      - alert: ValkeyNoTLS
        expr: valkey_tls_enabled == 0
        for: 0s
        labels:
          severity: critical
        annotations:
          summary: "Valkey running without TLS"
```

### Sentinel Configuration for HA (Mode 2+)

```
# /etc/valkey/sentinel.conf
sentinel monitor loom-valkey 10.0.1.10 6379 2
sentinel auth-pass loom-valkey $LOOM_VALKEY_SENTINEL_PASSWORD
sentinel down-after-milliseconds loom-valkey 5000
sentinel failover-timeout loom-valkey 60000
sentinel parallel-syncs loom-valkey 1

# Sentinel TLS
tls-port 26379
port 0
tls-cert-file /etc/valkey/tls/sentinel.crt
tls-key-file /etc/valkey/tls/sentinel.key
tls-ca-cert-file /etc/valkey/tls/ca.crt
tls-auth-clients yes
tls-repl-certs yes
```

---

## 8. OIDC Tenant Claim Verification (H18)

### Decision

User-tenant membership is verified on every request via a `tenant_memberships` table.
The JWT `tenant_id` claim is NOT trusted as the sole authority -- it is verified against
the membership table. If the user is not a member of the claimed tenant, the request
is rejected with 403 Forbidden.

Membership verification is cached in Valkey with a 5-minute TTL to avoid per-request
database queries. Cache invalidation occurs on membership changes via NATS events.

The first admin is added to a new tenant via CLI during tenant creation (NOT via API).
This prevents the chicken-and-egg bootstrap problem where no admin exists to grant
the first admin access.

### TenantMembership Type

```go
package auth

import (
    "context"
    "encoding/json"
    "fmt"
    "time"

    "github.com/google/uuid"
)

// TenantMembership represents a user's verified membership in a tenant.
type TenantMembership struct {
    // ID is the membership record's unique identifier.
    ID uuid.UUID `json:"id" db:"id"`

    // UserID is the OIDC subject claim ("sub").
    UserID string `json:"user_id" db:"user_id"`

    // TenantID is the tenant the user belongs to.
    TenantID uuid.UUID `json:"tenant_id" db:"tenant_id"`

    // Role is the user's role within this tenant.
    Role Role `json:"role" db:"role"`

    // GrantedBy is the user who added this membership.
    // For bootstrap: "system:cli:tenant-create".
    GrantedBy string `json:"granted_by" db:"granted_by"`

    // GrantedAt is when membership was granted.
    GrantedAt time.Time `json:"granted_at" db:"granted_at"`

    // ExpiresAt is an optional expiry for time-limited access.
    // NULL means permanent (until explicitly revoked).
    ExpiresAt *time.Time `json:"expires_at,omitempty" db:"expires_at"`

    // RevokedAt is set when membership is revoked.
    RevokedAt *time.Time `json:"revoked_at,omitempty" db:"revoked_at"`

    // RevokedBy is the user who revoked this membership.
    RevokedBy *string `json:"revoked_by,omitempty" db:"revoked_by"`
}
```

### Database Schema

```sql
-- Tenant membership table.
CREATE TABLE tenant_memberships (
    id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id     TEXT NOT NULL,
    tenant_id   UUID NOT NULL REFERENCES tenants(id),
    role        TEXT NOT NULL CHECK (role IN ('admin', 'operator', 'viewer', 'auditor')),
    granted_by  TEXT NOT NULL,
    granted_at  TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    expires_at  TIMESTAMPTZ,
    revoked_at  TIMESTAMPTZ,
    revoked_by  TEXT,
    UNIQUE (user_id, tenant_id)  -- one active membership per user per tenant
);

-- Fast lookup by user + tenant (the hot path for every request).
CREATE INDEX idx_membership_user_tenant ON tenant_memberships (user_id, tenant_id)
    WHERE revoked_at IS NULL;

-- List members of a tenant.
CREATE INDEX idx_membership_tenant ON tenant_memberships (tenant_id)
    WHERE revoked_at IS NULL;

-- Expiry cleanup.
CREATE INDEX idx_membership_expires ON tenant_memberships (expires_at)
    WHERE expires_at IS NOT NULL AND revoked_at IS NULL;
```

### Middleware Implementation

```go
package auth

import (
    "context"
    "encoding/json"
    "errors"
    "fmt"
    "net/http"
    "time"

    "github.com/google/uuid"
)

var (
    ErrAccessDenied       = errors.New("access denied: user is not a member of the claimed tenant")
    ErrMembershipExpired  = errors.New("access denied: tenant membership has expired")
    ErrMembershipRevoked  = errors.New("access denied: tenant membership has been revoked")
)

// MembershipVerifier checks user-tenant membership on every request.
type MembershipVerifier struct {
    db    MembershipStore
    cache MembershipCache
}

// MembershipStore is the database interface for membership queries.
type MembershipStore interface {
    GetActiveMembership(ctx context.Context, userID string, tenantID uuid.UUID) (*TenantMembership, error)
}

// MembershipCache provides short-lived caching to avoid per-request DB queries.
// Implemented with Valkey, 5-minute TTL.
type MembershipCache interface {
    Get(ctx context.Context, userID string, tenantID uuid.UUID) (*TenantMembership, bool, error)
    Set(ctx context.Context, membership *TenantMembership, ttl time.Duration) error
    Invalidate(ctx context.Context, userID string, tenantID uuid.UUID) error
}

const membershipCacheTTL = 5 * time.Minute

// VerifyMembership checks that the authenticated user is a member of the
// claimed tenant. Called on every API request after JWT validation.
func (v *MembershipVerifier) VerifyMembership(ctx context.Context, claims Claims) (*TenantMembership, error) {
    // Step 1: Check cache.
    if membership, ok, err := v.cache.Get(ctx, claims.Subject, claims.TenantID); err == nil && ok {
        if err := validateMembership(membership); err != nil {
            return nil, err
        }
        return membership, nil
    }

    // Step 2: Cache miss -- query database.
    membership, err := v.db.GetActiveMembership(ctx, claims.Subject, claims.TenantID)
    if err != nil {
        return nil, ErrAccessDenied
    }

    // Step 3: Validate membership state.
    if err := validateMembership(membership); err != nil {
        return nil, err
    }

    // Step 4: Cache the valid membership.
    _ = v.cache.Set(ctx, membership, membershipCacheTTL)

    return membership, nil
}

// validateMembership checks expiry and revocation.
func validateMembership(m *TenantMembership) error {
    if m.RevokedAt != nil {
        return ErrMembershipRevoked
    }
    if m.ExpiresAt != nil && m.ExpiresAt.Before(time.Now()) {
        return ErrMembershipExpired
    }
    return nil
}

// TenantMembershipMiddleware is the HTTP middleware that verifies
// tenant membership on every request.
func TenantMembershipMiddleware(verifier *MembershipVerifier) func(http.Handler) http.Handler {
    return func(next http.Handler) http.Handler {
        return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
            // Extract claims from context (set by JWT validation middleware).
            claims, ok := ClaimsFromContext(r.Context())
            if !ok {
                http.Error(w, "unauthorized: no claims in context", http.StatusUnauthorized)
                return
            }

            // Verify membership.
            membership, err := verifier.VerifyMembership(r.Context(), claims)
            if err != nil {
                // Audit the denied access attempt.
                auditDeniedAccess(r.Context(), claims, err)
                http.Error(w, err.Error(), http.StatusForbidden)
                return
            }

            // Set verified tenant context for downstream handlers.
            // The role from the membership table takes precedence over JWT roles
            // if they differ (defense against OIDC claim manipulation).
            ctx := WithVerifiedTenant(r.Context(), membership.TenantID, membership.Role)
            next.ServeHTTP(w, r.WithContext(ctx))
        })
    }
}

// WithVerifiedTenant sets the verified tenant ID and role in the context.
// Downstream code MUST use this function to get the tenant ID, never
// extract it directly from JWT claims.
func WithVerifiedTenant(ctx context.Context, tenantID uuid.UUID, role Role) context.Context {
    ctx = context.WithValue(ctx, verifiedTenantKey, tenantID)
    ctx = context.WithValue(ctx, verifiedRoleKey, role)
    return ctx
}

// VerifiedTenantFromContext retrieves the verified tenant ID from context.
// Returns an error if the tenant has not been verified (middleware not applied).
func VerifiedTenantFromContext(ctx context.Context) (uuid.UUID, Role, error) {
    tenantID, ok := ctx.Value(verifiedTenantKey).(uuid.UUID)
    if !ok {
        return uuid.Nil, "", errors.New("no verified tenant in context")
    }
    role, ok := ctx.Value(verifiedRoleKey).(Role)
    if !ok {
        return uuid.Nil, "", errors.New("no verified role in context")
    }
    return tenantID, role, nil
}

type contextKey string

var (
    verifiedTenantKey contextKey = "verified_tenant_id"
    verifiedRoleKey   contextKey = "verified_role"
)

// auditDeniedAccess logs a denied access attempt to the audit trail.
func auditDeniedAccess(ctx context.Context, claims Claims, err error) {
    // Emit audit event: "auth.membership_denied"
    // Include: claims.Subject, claims.TenantID, error message, source IP
}
```

### Bootstrap: First Admin

The first admin for a new tenant is added via CLI during tenant creation. This is
NOT an API operation -- it runs as a direct database insertion by the `loom` binary,
authenticated via local file-system access (the operator must have shell access to
the LOOM control plane).

```bash
# Create a new tenant with initial admin.
./loom tenant create \
    --name "Acme Corp" \
    --admin-user-id "auth0|abc123" \
    --admin-display-name "Jane Smith"

# This command:
# 1. Creates the tenant record.
# 2. Generates the tenant KEK (encrypted by MEK).
# 3. Creates the Temporal namespace "loom-<tenant_id>".
# 4. Creates the NATS authorization for "loom.<tenant_id>.>".
# 5. Inserts a TenantMembership record for the admin user.
# 6. Emits audit event "tenant.created" with actor "system:cli:tenant-create".
```

After the initial admin is created, they can add more users via the API:

```
POST /api/v1/tenants/{tenant_id}/memberships
Authorization: Bearer <jwt>
{
    "user_id": "auth0|def456",
    "role": "operator",
    "expires_at": "2027-01-01T00:00:00Z"  // optional
}
```

### NATS Integration

NATS account JWTs include tenant claims that are verified against the membership table
at issuance time. When a service or edge agent requests a NATS credential:

1. The service presents its JWT to the LOOM API.
2. The API verifies tenant membership (via `MembershipVerifier`).
3. The API issues a NATS account JWT scoped to `loom.<tenant_id>.>`.
4. The NATS account JWT is signed by the LOOM NATS operator key.
5. The NATS server verifies the account JWT signature on connection.

If a user's membership is revoked, the NATS account JWT remains valid until it expires
(short-lived, 1-hour TTL). For immediate revocation, the LOOM control plane publishes
a revocation entry to the NATS account revocation list.

```go
// NATSCredentialIssuer issues tenant-scoped NATS credentials.
type NATSCredentialIssuer struct {
    membershipVerifier *MembershipVerifier
    operatorKey        ed25519.PrivateKey
    accountJWTTTL      time.Duration // 1 hour
}

// IssueNATSCredential issues a NATS account JWT after verifying membership.
func (i *NATSCredentialIssuer) IssueNATSCredential(ctx context.Context, claims Claims) (*NATSCredential, error) {
    // Verify membership.
    membership, err := i.membershipVerifier.VerifyMembership(ctx, claims)
    if err != nil {
        return nil, fmt.Errorf("cannot issue NATS credential: %w", err)
    }

    // Build NATS account JWT with tenant-scoped permissions.
    accountJWT := buildNATSAccountJWT(
        membership.TenantID,
        membership.UserID,
        membership.Role,
        i.accountJWTTTL,
        i.operatorKey,
    )

    return &NATSCredential{
        AccountJWT: accountJWT,
        TenantID:   membership.TenantID,
        ExpiresAt:  time.Now().Add(i.accountJWTTTL),
    }, nil
}

// NATSCredential is the issued NATS connection credential.
type NATSCredential struct {
    AccountJWT string    `json:"account_jwt"`
    TenantID   uuid.UUID `json:"tenant_id"`
    ExpiresAt  time.Time `json:"expires_at"`
}
```

### Cache Invalidation

When membership changes occur (grant, revoke, expire), the Valkey cache is invalidated
via NATS events:

1. Membership change occurs (API call or background expiry job).
2. An event is published to `loom.system.membership.changed`.
3. All API server instances subscribe to this event.
4. On receipt, they invalidate the corresponding Valkey cache entry.

This ensures cache consistency across horizontally scaled API instances with a maximum
staleness of the NATS delivery latency (typically <10ms within a cluster).

---

## Implementation Priority

| Item | Phase | Effort | Risk if Deferred |
|------|-------|--------|------------------|
| Valkey security hardening (H17) | 1A | 1 week | HIGH -- exposed attack surface |
| OIDC tenant membership (H18) | 1B | 2 weeks | HIGH -- OIDC misconfiguration = cross-tenant breach |
| ConfigSnapshot encryption (H14) | 2 | 2 weeks | MEDIUM -- plaintext credentials at rest |
| Audit trail hash chain (H13) | 2 | 3 weeks | MEDIUM -- tamper detection required for compliance |
| Break-glass specification (H12) | 2 | 1 week | LOW -- emergency access undefined |
| Temporal DataConverter keys (H10) | 4 | 3 weeks | MEDIUM -- single key exposes all tenants |
| LLM single-turn enforcement (H11) | 8 | 1 week | LOW -- no multi-turn exists yet |
| External audit anchoring (H13 ext) | 10 | 2 weeks | LOW -- depends on hash chain |
| Internal CA lifecycle (H9) | 10 | 4 weeks | LOW -- self-signed certs work for early phases |

---

## References

- [SECURITY-MODEL.md](SECURITY-MODEL.md) -- Core security architecture, 3-tier encryption, RBAC
- [VAULT-ARCHITECTURE.md](VAULT-ARCHITECTURE.md) -- Credential vault, MEK/KEK/DEK hierarchy, memory protection
- [AUDIT-MODEL.md](AUDIT-MODEL.md) -- Audit record structure, immutability rules, retention
- [ADAPTER-CONTRACT.md](ADAPTER-CONTRACT.md) -- ConfigSnapshot definition, SnapshotCapable interface
- [DEPLOYMENT-TOPOLOGY.md](DEPLOYMENT-TOPOLOGY.md) -- Deployment modes, NATS topology, edge agents
- [WORKFLOW-CONTRACT.md](WORKFLOW-CONTRACT.md) -- Temporal integration, DataConverter, namespace isolation
- [LLM-BOUNDARIES.md](LLM-BOUNDARIES.md) -- LLM safety rules, typed responses, confidence thresholds
- [RESEARCH-security-hardening.md](RESEARCH-security-hardening.md) -- Original gap analysis and open questions
