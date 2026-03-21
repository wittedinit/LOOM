# Security Hardening Responses

> **Status:** Decided — ready for implementation
> **Addresses:** GAP-ANALYSIS.md findings H9-H14, H17, H18 and RESEARCH-security-hardening.md findings SH-1 through SH-12
> **Last updated:** 2026-03-21

---

## 1. Internal CA Lifecycle (H9, SH-1)

**Gap:** No OCSP responder, no CRL distribution mechanism, no CA rotation procedure, undefined certificate lifetimes for different component classes.

### Decisions

**OCSP responder:** Embedded in the LOOM hub process, serving an `/ocsp` endpoint. This avoids deploying a separate OCSP infrastructure while providing real-time revocation checking for hub-connected components. Edge agents that cannot reach the hub fall back to CRL-based checking.

**CRL distribution:** CRLs are published to the NATS KV bucket `loom.pki.crl`. Edge agents fetch the latest CRL on a 1-hour polling interval. On reconnection after a partition, the agent fetches immediately. Hub-connected services receive CRL updates via NATS subscription with no polling delay.

**CA rotation:** When the current CA approaches end-of-life, a new CA is generated and cross-signed by the old CA. Both CAs are trusted during a 90-day overlap window. Agents and services accept certificates signed by either CA during the transition. The old CA is removed from trust stores only after all certificates it issued have expired or been re-issued.

**Certificate lifetimes:**

| Component Class | Lifetime | Rationale |
|----------------|----------|-----------|
| Hub services | 90 days | Short-lived, auto-renewed, always online |
| Edge agent certs | 365 days | Must survive extended offline periods |
| Device adapter certs | 30 days | Tightest rotation — closest to untrusted devices |

**Grace period:** 7 days after certificate expiry. During the grace period, operations are allowed with a warning-level alert emitted on every TLS handshake using an expired certificate. This is NOT a hard-fail — it prevents operational disruption while providing strong signal to renew. After the 7-day grace, the certificate is rejected.

### Go Types

```go
type InternalCA struct {
    CACert          *x509.Certificate
    CAKey           crypto.Signer
    CRLDistribution CRLDistribution
    OCSPResponder   OCSPResponder
    RotationPlan    *CertRotationPlan
}

type CRLDistribution struct {
    NATSBucket      string        // "loom.pki.crl"
    PublishInterval time.Duration // 1 hour
    LastPublished   time.Time
}

type OCSPResponder struct {
    Endpoint   string // "/ocsp"
    SigningKey  crypto.Signer
    CacheMaxAge time.Duration // 5 minutes
}

type CertRotationPlan struct {
    CurrentCA      *x509.Certificate
    NextCA         *x509.Certificate // nil if no rotation in progress
    OverlapDays    int               // 90
    TransitionStart time.Time
    TransitionEnd   time.Time
}
```

### Open Questions Resolved

| Question | Decision |
|----------|----------|
| step-ca or custom? | Built-in CA for Phase 1-5; external CA integration (Vault PKI, cert-manager) deferred to Phase 10+ |
| Max CRL propagation delay? | 1 hour (polling interval) + NATS delivery latency |
| Edge agent cert lifetime? | 365 days with automatic renewal at 30 days before expiry |
| Certificate transparency? | Not required for internal CA; revisit if external-facing services are added |

---

## 2. Temporal DataConverter Encryption (H10, SH-2)

**Gap:** Encryption key scope (global vs. per-tenant) and rotation procedure for the Temporal DataConverter are undefined.

### Decisions

**Key scope:** Per-namespace encryption key, one per tenant. Each Temporal namespace maps 1:1 to a LOOM tenant, so namespace-scoped keys provide tenant isolation without the complexity of per-workflow keys.

**Key derivation:** The tenant KEK is retrieved from the vault. A DataConverter-specific DEK is derived via HKDF-SHA256 using the context string `"temporal-dataconverter"` and the namespace name as the info parameter. This ensures the DataConverter key is cryptographically distinct from other keys derived from the same KEK.

**Implementation:** A custom `DataConverter` wrapping Temporal's `codec.PayloadCodec` interface. Each payload is encrypted with AES-256-GCM using the namespace-specific DEK. The encrypted payload includes a key version identifier in the header to support decryption with older keys during replay.

**Key rotation:** When a tenant KEK rotates, new DataConverter DEKs are automatically derived. Old key versions are retained (the vault keeps previous KEK versions) so that Temporal's workflow replay can decrypt historical payloads. Old key versions are never purged while any workflow using them remains in the retention window.

### Go Types

```go
type EncryptedDataConverter struct {
    keyProvider DataConverterKeyProvider
    codec       *encryptedCodec
}

type DataConverterKeyProvider interface {
    // GetKey returns the current DEK for a namespace, along with its version.
    GetKey(ctx context.Context, namespace string) (key []byte, version int, err error)
    // GetKeyByVersion returns a specific DEK version (for replay decryption).
    GetKeyByVersion(ctx context.Context, namespace string, version int) ([]byte, error)
}
```

### Open Questions Resolved

| Question | Decision |
|----------|----------|
| Can DataConverter access namespace? | Yes — Temporal's `converter.EncodingPayloadConverter` receives context including namespace |
| Key rotation vs. replay? | Old KEK versions retained; DEKs re-derived on demand for historical decryption |
| Separate key for search attributes? | No — search attributes use the same per-namespace DEK; revisit if search attribute indexing requires server-side decryption |

---

## 3. LLM Context Isolation (H11, SH-3)

**Gap:** Shared LLM provider (vLLM or cloud) may carry tenant data across requests via conversation history or KV cache reuse.

### Decisions

**Stateless single-turn:** Every LLM call is a stateless single-turn request. No conversation history is shared across tenants, and no conversation history is shared across requests for the same tenant. If multi-turn context is needed, it is reconstructed from Temporal workflow state, NOT from LLM memory.

**vLLM configuration:** Use `--disable-kv-cache-reuse` to prevent KV cache sharing between requests from different tenants. If tenant volume and security requirements justify it, deploy separate vLLM instances per tenant (isolated inference). The default is shared vLLM with cache reuse disabled.

**Context construction:** Each LLM request is assembled from exactly three sources:
1. **System prompt** — generic, tenant-agnostic, loaded from versioned configuration
2. **RAG results** — tenant-scoped (pgvector query filtered by `tenant_id`)
3. **Current request** — the user's immediate request or the workflow's current decision point

**Tenant ID handling:** The tenant ID is included in the LLM request metadata for audit and routing purposes, but is NOT included in the prompt content itself. This prevents the LLM from learning tenant identifiers or associating data with specific tenants.

### Go Types

```go
type LLMRequestContext struct {
    TenantID     uuid.UUID `json:"-"` // metadata only, not in prompt
    SystemPrompt string
    RAGResults   []RAGResult
    UserRequest  string
    MaxTokens    int
    Temperature  float64
}

type RAGResult struct {
    Content    string
    Source     string  // document reference
    Score      float64 // relevance score
}
```

### Open Questions Resolved

| Question | Decision |
|----------|----------|
| Single-turn sufficient? | Yes — multi-turn is reconstructed from workflow state; LLM never maintains session |
| Sensitive mode? | Deferred to Phase 10+; for now, local vLLM is the default and cloud LLM is opt-in with contractual DPA |
| RAG partitioning? | pgvector with RLS (row-level security) filtered by `tenant_id`; separate collections not needed initially |

---

## 4. Break-Glass Token (H12, SH-4)

**Gap:** TTL mismatch between SECURITY-MODEL.md (15 minutes) and VAULT-ARCHITECTURE.md (1 hour). JTI cache durability, ceremony, and rate limiting undefined.

### Decisions

**TTL:** 15 minutes. The shorter TTL is chosen because break-glass tokens bypass normal RBAC and represent the highest-risk credential in the system. If an emergency operation requires more than 15 minutes, a new break-glass token must be issued through the full ceremony.

**Token format:** JWT with the following claims:

```json
{
  "sub": "<admin-user-id>",
  "tenant_id": "<tenant-uuid>",
  "break_glass": true,
  "iat": 1711036800,
  "exp": 1711037700,
  "jti": "<unique-token-id>",
  "approvers": ["<admin-a-uuid>", "<admin-b-uuid>"]
}
```

**JTI replay cache:** PostgreSQL table `break_glass_tokens`, NOT Valkey. Break-glass tokens must survive Valkey restarts and must be auditable. The table schema includes `jti`, `issued_at`, `expires_at`, `issued_by`, `approved_by`, `used`, `used_at`, and `revoked`.

**Ceremony:** Dual authorization — requires 2 approvers. Both must authenticate via separate MFA challenges. The ceremony flow is:
1. Admin A initiates break-glass request (provides justification)
2. Admin B receives notification, reviews justification, authenticates with MFA, and co-signs
3. Token is generated, signed with a dedicated break-glass signing key (separate from normal JWT signing)
4. Token usage is logged to the tamper-evident audit trail

**Audit:** Every break-glass token usage emits a `security.break_glass.used` event. This event is delivered to ALL tenant administrators AND the security team (via a dedicated NATS subject `loom.security.break_glass`).

**Rate limit:** Maximum 3 break-glass tokens per tenant per 24-hour rolling window. This prevents abuse while allowing legitimate emergencies. The rate limit is enforced in PostgreSQL (count of non-expired tokens in the last 24 hours).

### Go Types

```go
type BreakGlassRequest struct {
    RequestedBy   uuid.UUID
    TenantID      uuid.UUID
    Justification string
    RequestedAt   time.Time
}

type BreakGlassToken struct {
    JTI         string
    TenantID    uuid.UUID
    IssuedBy    uuid.UUID
    ApprovedBy  uuid.UUID
    IssuedAt    time.Time
    ExpiresAt   time.Time // IssuedAt + 15 minutes
    Used        bool
    UsedAt      *time.Time
}

type BreakGlassCeremony struct {
    Request       BreakGlassRequest
    Approvers     [2]uuid.UUID // exactly 2 required
    MFACompleted  [2]bool
    SigningKey     crypto.Signer // dedicated break-glass key
}
```

### Open Questions Resolved

| Question | Decision |
|----------|----------|
| Single admin available? | Not permitted — break-glass requires dual authorization without exception; plan staffing accordingly |
| Pre-generated or on-demand? | On-demand only — pre-generated tokens are a storage risk |
| 15 minutes sufficient? | Yes — issue a new token if more time is needed; the ceremony is fast when both admins are available |

---

## 5. Audit Trail Integrity Chain (H13, SH-5)

**Gap:** BLAKE3 integrity chain referenced in SECURITY-MODEL.md but not present in the `AuditRecord` struct or database schema. No external anchoring mechanism.

### Decisions

**Hash chain:** Each audit record includes a `PreviousHash` field containing the SHA-256 hash of the previous record in the chain. SHA-256 is chosen over BLAKE3 for the persisted chain because SHA-256 is universally available in FIPS-compliant environments and supported natively by PostgreSQL. BLAKE3 may still be used for in-memory verification where performance matters.

**Schema addition:**

```sql
ALTER TABLE audit_log
    ADD COLUMN previous_hash BYTEA NOT NULL,
    ADD COLUMN record_hash BYTEA NOT NULL;

CREATE INDEX idx_audit_log_chain ON audit_log (tenant_id, sequence_num);
```

**External anchoring:** A daily Merkle root is computed from all audit records generated that day and published to an immutable store. The default immutable store is a PostgreSQL append-only table (`audit_anchors`) with separate database credentials from the main application. Optional integration with an RFC 3161 timestamping authority or blockchain timestamping service is supported but not required.

**Verification CLI:**

```
loom admin audit verify --from 2026-03-01 --to 2026-03-21
```

This command walks the hash chain for the specified date range and verifies continuity. It also checks daily Merkle roots against the anchor store. Any break in the chain is reported with the exact sequence number where tampering is detected.

### Go Types

```go
type AuditChainVerifier struct {
    DB          *sql.DB
    AnchorStore AnchorStore
}

func (v *AuditChainVerifier) VerifyRange(ctx context.Context, tenantID uuid.UUID, from, to time.Time) (*VerificationResult, error)

type MerkleAnchor struct {
    TenantID    uuid.UUID
    Date        time.Time
    MerkleRoot  []byte   // SHA-256
    RecordCount int64
    FirstSeq    int64
    LastSeq     int64
    CreatedAt   time.Time
}
```

### Open Questions Resolved

| Question | Decision |
|----------|----------|
| Continuous or on-demand? | Both — daily automated verification job + on-demand CLI for investigations |
| S3 Object Lock sufficient? | PostgreSQL append-only table is the default anchor; S3 Object Lock or RFC 3161 TSA as optional additional anchors |
| Scale of full chain verification? | Daily Merkle roots enable range-based verification without walking the entire chain; full chain verification is available but expected to be used rarely |

---

## 6. ConfigSnapshot Encryption (H14, SH-6)

**Gap:** `ConfigSnapshot.Data` is stored as unencrypted `[]byte`. Device running configs contain SNMP community strings, enable passwords, BGP authentication keys, IPsec PSKs, and management ACLs.

### Decisions

**Mandatory encryption:** `ConfigSnapshot.Data` MUST be encrypted with the tenant's DEK before storage. The `Data` field is replaced with an `EncryptedBlob` using the same envelope encryption pattern as the credential vault (AES-256-GCM with a per-snapshot DEK wrapped by the tenant KEK).

**Decryption:** Only on restore operations. Decryption requires vault access and emits an audit log entry. Read-only operations (listing snapshots, comparing metadata) do not require decryption.

**Reason:** Device running configs are among the most sensitive data in the system. A single leaked config can expose every credential on the device, every neighbor relationship, and the full network topology visible to that device.

### Go Types

```go
type ConfigSnapshot struct {
    DeviceID      uuid.UUID
    Timestamp     time.Time
    Format        string    // "cisco-ios", "junos-xml", "arista-json"

    // Encrypted payload (replaces plain Data []byte)
    EncryptedData []byte    // AES-256-GCM ciphertext
    DEKWrapped    []byte    // DEK wrapped by tenant KEK
    Nonce         []byte    // AES-GCM nonce
    KeyVersion    int       // tenant KEK version used

    // Metadata (unencrypted, for indexing)
    Checksum      string    // SHA-256 hash of plaintext config (for diff detection)
    SizeBytes     int       // plaintext size
}
```

### Open Questions Resolved

| Question | Decision |
|----------|----------|
| Store in vault or Temporal? | Temporal workflow history + database; vault is for keys, not bulk data |
| Scrubbing vs. encryption? | Full encryption — scrubbing is lossy and cannot guarantee coverage of all credential patterns |
| Retention policy? | 90 days default, configurable per tenant; oldest snapshots purged by background job |
| DataConverter or separate encryption? | Separate — ConfigSnapshot encryption uses the vault DEK pattern directly; Temporal DataConverter encrypts the workflow payload envelope around it |

---

## 7. Valkey Security (H17, SH-7)

**Gap:** Valkey stores session tokens, cached data, and pub/sub for WebSocket fan-out but is not mentioned in the security model, security checklist, or attack surface analysis.

### Decisions

**TLS:** Mandatory for all Valkey connections. Valkey is configured with `tls-port` only; the plaintext `port` is set to `0` (disabled). Certificates are issued by the internal CA with 90-day lifetime.

**Authentication:** ACL-based authentication with per-service accounts. No shared password. Each LOOM service connects with its own Valkey user:

| Service | Valkey User | Allowed Key Patterns | Allowed Commands |
|---------|-------------|---------------------|------------------|
| loom-api | `loom-api` | `session:*`, `cache:*` | `GET`, `SET`, `DEL`, `EXPIRE`, `TTL` |
| loom-worker | `loom-worker` | `idempotency:*`, `ratelimit:*` | `GET`, `SET`, `DEL`, `EXPIRE`, `INCR` |
| loom-agent | `loom-agent` | `pubsub:*` | `SUBSCRIBE`, `PUBLISH` |

**Command restrictions:** The following commands are disabled for all non-admin accounts: `FLUSHALL`, `FLUSHDB`, `CONFIG`, `DEBUG`, `KEYS`. The `default` user is disabled entirely.

**Data encryption:** Session tokens stored in Valkey are AES-256-GCM encrypted opaque blobs, not plaintext JWTs. This ensures that a Valkey compromise does not directly yield usable session tokens.

**Persistence encryption:** RDB and AOF files are stored on an encrypted filesystem (LUKS or equivalent). Valkey itself does not encrypt persistence files, so OS-level encryption is required.

**Eviction policy:** `allkeys-lru` with `maxmemory` set to 75% of available RAM. This prevents OOM conditions while maintaining useful cache data.

### Go Types

```go
type ValkeyConfig struct {
    TLSEnabled     bool              // always true
    TLSCertFile    string
    TLSKeyFile     string
    TLSCACertFile  string
    ACLUsers       []ValkeyACLUser
    MaxMemory      string            // "75%"
    EvictionPolicy string            // "allkeys-lru"
    DisabledCmds   []string          // ["FLUSHALL", "FLUSHDB", "CONFIG", "DEBUG", "KEYS"]
}

type ValkeyACLUser struct {
    Username     string
    PasswordHash string            // ACL SETUSER ... >#hash
    KeyPatterns  []string          // e.g., ["session:*", "cache:*"]
    Commands     []string          // e.g., ["+get", "+set", "+del"]
}
```

### Open Questions Resolved

| Question | Decision |
|----------|----------|
| Add to attack surface analysis? | Yes — Valkey should be added as a vector in the attack surface documentation |
| Sentinel or Cluster? | Valkey Sentinel for HA in production; Cluster mode deferred until scale requires it |
| Session token encryption? | Yes — AES-256-GCM encrypted blobs, not plaintext JWTs |

---

## 8. OIDC Tenant Verification (H18, SH-8)

**Gap:** JWT `tenant_id` claim is trusted without verifying that the user actually belongs to the claimed tenant. A misconfigured OIDC provider could allow tenant impersonation.

### Decisions

**Membership verification:** The JWT `tenant_id` claim is NOT trusted alone. After JWT signature and expiry validation, the API middleware queries the `tenant_memberships` table to confirm that `user_id` (from `sub` claim) has an active membership in `tenant_id`.

**Caching:** The membership check result is cached in Valkey with a 5-minute TTL. The cache key is `membership:<user_id>:<tenant_id>`. The cache is explicitly invalidated when a membership is added, removed, or modified (via NATS event `loom.iam.membership.changed`).

**Enforcement:** If the membership check fails (user is not a member of the claimed tenant), the request is rejected with HTTP 403 regardless of the JWT's validity. This is a hard security boundary.

### Go Types

```go
type TenantMembershipVerifier struct {
    DB    *sql.DB
    Cache ValkeyClient
}

// Verify checks that userID has an active membership in tenantID.
// Returns true if the membership exists and is not expired.
func (v *TenantMembershipVerifier) Verify(ctx context.Context, userID string, tenantID uuid.UUID) (bool, error)
```

### Open Questions Resolved

| Question | Decision |
|----------|----------|
| Sole authority or augment JWT? | Membership table is authoritative; JWT claim is a hint for routing, not a security assertion |
| First admin bootstrap? | Tenant creation API (super-admin only) creates both the tenant and the initial admin membership atomically |
| Cache membership? | Yes — Valkey, 5-minute TTL, invalidated on membership change events |
| NATS subject isolation? | NATS account permissions are configured from the membership table, not from JWT claims; same source of truth |

---

## 9. Additional Findings (SH-9 through SH-12)

### SH-9: Certificate Grace Period

**Decision:** 7-day grace period after certificate expiry. Operations are permitted with warning-level alerts. After 7 days, the certificate is hard-rejected. This balances operational continuity against security risk — a 24-hour grace (as originally proposed in the research) is too short for edge agents that may be offline for extended periods, while an unlimited grace defeats the purpose of certificate expiry.

### SH-10: Edge Certificate Lifetime

**Decision:** 365-day certificate lifetime for edge agents, with automatic renewal triggered at 30 days before expiry. The long lifetime accommodates edge sites that may lose connectivity for weeks or months. The 30-day renewal window provides ample time for the renewal process to complete even with intermittent connectivity.

### SH-11: Valkey Session Token Encryption

**Decision:** Session tokens stored in Valkey are AES-256-GCM encrypted, not plaintext JWT strings. The encryption key is derived from the service's Valkey credentials via HKDF. This ensures that a Valkey data leak (backup exposure, memory dump, replication intercept) does not directly yield usable session tokens.

### SH-12: NATS ACL Model

**Decision:** Per-account publish/subscribe permissions matching tenant scope. Each tenant gets a NATS account with permissions restricted to subjects under `loom.<tenant_id>.>`. Cross-tenant subjects (system events, CRL distribution) use a separate system account. Account configuration is generated from the `tenant_memberships` table — the same source of truth as OIDC tenant verification (see Section 8).

---

## Implementation Priority

| Item | Phase | Effort | Dependencies |
|------|-------|--------|--------------|
| Valkey security hardening (TLS, ACL, commands) | 1A | 1 week | Valkey deployment |
| OIDC tenant membership verification | 1B | 1-2 weeks | Auth middleware, `tenant_memberships` table |
| Valkey session token encryption (SH-11) | 1B | 3 days | Valkey security hardening |
| NATS ACL per-tenant accounts (SH-12) | 1B | 1 week | NATS deployment, tenant provisioning |
| ConfigSnapshot encryption | 2 | 1-2 weeks | Vault integration |
| Audit trail hash chain + schema | 2 | 2-3 weeks | Audit model |
| Break-glass ceremony specification | 2 | 1 week | Security policy, MFA integration |
| Temporal DataConverter per-namespace key | 4 | 2-3 weeks | Temporal integration, vault key derivation |
| LLM single-turn enforcement | 8 | 1 week | LLM integration |
| External audit anchoring (Merkle roots) | 10 | 1-2 weeks | Audit hash chain |
| Internal CA lifecycle (OCSP, CRL, rotation) | 10 | 3-4 weeks | PKI design |
| Edge cert 365-day lifetime + renewal (SH-10) | 10 | 1 week | Internal CA lifecycle |
| Certificate grace period enforcement (SH-9) | 10 | 3 days | Internal CA lifecycle |

---

## References

- [GAP-ANALYSIS.md](GAP-ANALYSIS.md) — Consolidated expert review (H9-H14, H17, H18)
- [RESEARCH-security-hardening.md](RESEARCH-security-hardening.md) — Detailed analysis and recommendations
- SECURITY-MODEL.md — Core security architecture
- VAULT-ARCHITECTURE.md — Credential encryption and KEK hierarchy
- AUDIT-MODEL.md — Audit record structure
- ADAPTER-CONTRACT.md — ConfigSnapshot definition
- DEPLOYMENT-TOPOLOGY.md — Component placement and TLS requirements
