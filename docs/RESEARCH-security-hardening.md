# Research: Security Hardening Gaps

> **Status:** Open — requires design decisions before Phase 5 (production)
> **Addresses:** GAP-ANALYSIS.md findings H9-H14, H17, H18
> **Owner:** TBD
> **Last updated:** 2026-03-21

---

## 1. Problem Statement

The LOOM security model is thorough for the control plane core but has gaps at the boundaries. Eight areas require hardening before any production deployment:

1. **Internal CA lifecycle** — No OCSP, no CRL distribution, no CA rotation procedure
2. **Temporal DataConverter key management** — Encryption key scope and rotation undefined
3. **LLM cross-tenant context leakage** — Shared LLM provider may carry tenant data across requests
4. **Break-glass token inconsistencies** — TTL mismatch, jti cache durability, ceremony undefined
5. **Audit trail integrity** — BLAKE3 chain referenced but not in schema, no external anchoring
6. **ConfigSnapshot encryption** — Device configs (containing secrets) stored as unencrypted `[]byte`
7. **Valkey security posture** — Not mentioned in security model despite storing session tokens
8. **OIDC tenant claim verification** — No user-to-tenant membership check beyond JWT claim trust

---

## 2. Internal CA Lifecycle

### 2.1 Current State

- `InternalCA` type handles certificate issuance and revocation
- `CertLifetime` defaults to 90 days with auto-renewal
- CRL path defined but no distribution mechanism
- No OCSP responder
- No CA certificate rotation procedure

### 2.2 Gaps and Recommendations

| Gap | Risk | Recommendation |
|-----|------|---------------|
| No CRL distribution to edge agents | Edge agent trusts revoked certificates until manually updated | Push CRL updates via NATS; edge agent checks CRL on every TLS handshake |
| No OCSP | Real-time revocation checking unavailable | OCSP stapling on hub-to-agent connections; OCSP responder on control plane |
| CA certificate rotation | Catastrophic event requiring simultaneous update of all components | Cross-signing: new CA signs with both old and new CA cert for transition period |
| Certificate expiry during operation | Active operations fail mid-execution | Grace period: accept expired certificates for up to 24 hours, with alerting |
| Edge agent offline > CertLifetime | Agent certificate expires, cannot reconnect | Pre-issue certificates with extended lifetime for edge agents (365 days) |

### 2.3 Recommended CA Architecture

```
Option A: Built-in CA (simple, sufficient for Phase 1-5)
  - step-ca embedded or sidecar
  - ACME protocol for automated issuance/renewal
  - Short-lived certificates (24h for services, 90 days for edge agents)
  - CRL pushed via NATS loom.system.crl subject

Option B: External CA integration (Phase 10+)
  - HashiCorp Vault PKI secrets engine
  - cert-manager for Kubernetes deployments
  - LOOM acts as RA (Registration Authority), external CA signs
```

### 2.4 Open Questions

- [ ] Is step-ca acceptable as the built-in CA, or should LOOM implement its own?
- [ ] What is the maximum acceptable time between CRL generation and edge agent receipt?
- [ ] Should edge agents use a longer certificate lifetime (365 days) to handle extended offline periods?
- [ ] Is certificate transparency logging required for compliance?

---

## 3. Temporal DataConverter Key Management

### 3.1 Current State

- `EncryptedDataConverter` encrypts Temporal payloads with AES-256-GCM
- Key source not specified
- Single key vs per-tenant key not decided
- Key rotation procedure not defined

### 3.2 Analysis

| Approach | Security | Complexity | Temporal Compatibility |
|----------|----------|-----------|----------------------|
| **Single global key** | Weakest — one key compromise exposes all tenants | Low | Works natively — DataConverter is global |
| **Per-namespace key** | Better — key per tenant namespace | Medium | Possible — DataConverter can inspect namespace |
| **Per-tenant key from vault** | Best — integrated with KEK hierarchy | High | Complex — DataConverter must resolve tenant context |

### 3.3 Recommendation

**Per-namespace key** derived from the tenant's KEK:

```go
type TenantDataConverter struct {
    vault       VaultClient
    keyCache    sync.Map  // namespace -> cached DEK (short-lived)
}

func (c *TenantDataConverter) Encode(namespace string, payloads []*commonpb.Payload) ([]*commonpb.Payload, error) {
    dek, err := c.getOrDeriveKey(namespace)
    // Encrypt each payload with namespace-specific DEK
}

func (c *TenantDataConverter) getOrDeriveKey(namespace string) ([]byte, error) {
    // Derive DEK from tenant KEK using HKDF
    // Cache for 5 minutes to avoid vault round-trips on every activity
}
```

### 3.4 Key Rotation

- DataConverter DEKs are derived (HKDF) from tenant KEKs
- When KEK rotates, new DEKs are automatically derived
- Old DEKs must be retained (or re-derivable) for replay of historical workflows
- Retention: keep old KEK versions for maximum workflow retention period + buffer

### 3.5 Open Questions

- [ ] Can Temporal's DataConverter access the namespace/tenant context for per-tenant encryption?
- [ ] How does key rotation interact with Temporal's workflow replay? (Old payloads must be decryptable)
- [ ] Should encrypted search attributes use a separate key from workflow payloads?

---

## 4. LLM Cross-Tenant Context Isolation

### 4.1 Current State

- LLM receives device metadata scoped to the requesting tenant
- Credentials excluded (opaque `CredentialRef` only)
- No specification of context isolation between tenants

### 4.2 Threat Vectors

| Vector | Risk | Mitigation |
|--------|------|-----------|
| Shared conversation context | Tenant A's device info leaks to tenant B's session | **Stateless single-turn calls only** — no conversation history |
| vLLM shared KV cache | Continuous batching may leak between tenants | Use separate inference sessions per tenant, or accept risk with monitoring |
| RAG retrieval scope | Tenant A's documents included in tenant B's context | Tenant-scoped vector store partitions (pgvector with RLS) |
| System prompt containing tenant data | System prompt cached across requests | System prompt is generic; tenant data only in user message |
| LLM provider logging | Provider logs contain tenant data | Use local LLM for sensitive deployments; contractual DPA for cloud providers |

### 4.3 Recommendation

```go
type LLMRequest struct {
    TenantID    uuid.UUID
    // Each request is a single turn — no conversation history carried
    SystemPrompt string              // generic, tenant-agnostic
    UserMessage  string              // contains tenant-scoped data
    Temperature  float64
    MaxTokens    int
}

// Enforcement:
// 1. LLM client creates a new session per request (no session reuse)
// 2. RAG retrieval filters by tenant_id (pgvector with WHERE tenant_id = ?)
// 3. System prompt is loaded from versioned config, never contains tenant data
// 4. Response is validated against schema before returning (prevents prompt injection leaking data)
```

### 4.4 Open Questions

- [ ] Is stateless single-turn sufficient, or do some use cases need multi-turn? (Cost optimization discussions?)
- [ ] Should LOOM support a "sensitive mode" that disables cloud LLM and requires local inference?
- [ ] How is the RAG vector store partitioned for multi-tenancy? (Separate collections? RLS?)

---

## 5. Break-Glass Token Lifecycle

### 5.1 Inconsistencies Found

| Aspect | SECURITY-MODEL.md | VAULT-ARCHITECTURE.md |
|--------|-------------------|----------------------|
| Token TTL | 15 minutes | 1 hour |
| Authorization | "Dual authorization required" | "Bypasses RBAC" |
| Storage | "Physical safe or dedicated secrets manager" | Not specified |
| Generation | Not specified | Not specified |

### 5.2 Recommended Specification

```go
type BreakGlassToken struct {
    JTI         string        // unique token ID (UUID)
    IssuedAt    time.Time
    ExpiresAt   time.Time     // IssuedAt + 15 minutes (standardize on shorter TTL)
    IssuedBy    string        // admin who initiated
    ApprovedBy  string        // second admin who approved (dual authorization)
    Scope       BreakGlassScope // what the token can do
    Used        bool          // single-use enforcement
    UsedAt      *time.Time
}

type BreakGlassScope struct {
    Actions     []string      // e.g., ["vault.unseal", "credential.read", "device.power_cycle"]
    TenantIDs   []uuid.UUID   // scoped to specific tenants (never "*")
    DeviceIDs   []uuid.UUID   // optional device scope
}
```

**Ceremony:**
1. Admin A initiates break-glass request (generates unsigned token)
2. Admin B reviews and co-signs (dual authorization)
3. Token is signed with a dedicated break-glass signing key (separate from normal JWT signing)
4. Token is stored in a tamper-evident log (not just audit trail)
5. Token is used within 15 minutes
6. JTI replay check: PostgreSQL table (not Valkey — must survive Valkey outage)

### 5.3 Open Questions

- [ ] What happens when only one admin is available? (Single-admin override with extended audit?)
- [ ] Should break-glass tokens be pre-generated and stored in a safe, or generated on-demand?
- [ ] Is 15 minutes sufficient for emergency operations? (Some firmware recoveries take longer)

---

## 6. Audit Trail Integrity

### 6.1 Current State

- Audit records are append-only with DB triggers preventing UPDATE/DELETE
- BLAKE3 integrity chain mentioned in SECURITY-MODEL.md but not in the `AuditRecord` struct or DB schema
- No external anchoring mechanism

### 6.2 Recommended Implementation

```go
type AuditRecord struct {
    // ... existing fields

    // Integrity chain
    SequenceNum     int64     // monotonically increasing per tenant
    PreviousHash    string    // BLAKE3 hash of previous record (chain link)
    RecordHash      string    // BLAKE3 hash of this record (excluding RecordHash itself)
}

// Hash computation
func computeHash(record AuditRecord, previousHash string) string {
    h := blake3.New()
    h.Write([]byte(record.TenantID.String()))
    h.Write([]byte(strconv.FormatInt(record.SequenceNum, 10)))
    h.Write([]byte(record.Action))
    h.Write([]byte(record.ActorID))
    h.Write(record.Details)
    h.Write([]byte(previousHash))
    h.Write([]byte(record.Timestamp.Format(time.RFC3339Nano)))
    return hex.EncodeToString(h.Sum(nil))
}
```

### 6.3 External Anchoring

```
Every 1000 records (or every 1 hour, whichever comes first):
1. Compute Merkle root of last 1000 record hashes
2. Sign the Merkle root with an audit signing key
3. Publish signed root to:
   - S3 Object Lock bucket (immutable for retention period)
   - Optional: RFC 3161 timestamping authority
   - Optional: separate audit database (different admin domain)
```

### 6.4 Tamper Detection

```go
// Periodic integrity verification job
func VerifyAuditChain(ctx context.Context, tenantID uuid.UUID) error {
    records := db.GetAuditRecords(tenantID, orderBy: "sequence_num ASC")
    previousHash := ""
    for _, record := range records {
        expected := computeHash(record, previousHash)
        if record.RecordHash != expected {
            alert.Critical("Audit chain broken at sequence %d for tenant %s",
                record.SequenceNum, tenantID)
            return ErrAuditTampered
        }
        previousHash = record.RecordHash
    }
    return nil
}
```

### 6.5 Open Questions

- [ ] Should audit chain verification run continuously or on-demand?
- [ ] Is S3 Object Lock sufficient as an external anchor, or is an independent timestamping authority required?
- [ ] How does hash chain verification scale with audit record volume? (Periodic vs. full chain)

---

## 7. ConfigSnapshot Encryption

### 7.1 The Problem

`ConfigSnapshot.Data` is an opaque `[]byte` containing device running configs. These configs contain SNMP community strings, enable passwords, BGP auth keys, IPsec PSKs, and management ACLs. They are stored unencrypted in Temporal workflow history and potentially in the database.

### 7.2 Recommendation

```go
type ConfigSnapshot struct {
    DeviceID    uuid.UUID
    Timestamp   time.Time
    Format      string          // "cisco-ios", "junos-xml", "arista-json"

    // Encrypted payload
    EncryptedData []byte         // AES-256-GCM encrypted config
    DEKWrapped    []byte         // DEK wrapped by tenant KEK
    Nonce         []byte         // AES-GCM nonce

    // Metadata (unencrypted, for indexing)
    Checksum    string          // BLAKE3 hash of plaintext config
    SizeBytes   int             // plaintext size
}
```

**Encryption flow:**
1. Adapter captures running config (plaintext)
2. Generate per-snapshot DEK
3. Encrypt config with DEK (AES-256-GCM)
4. Wrap DEK with tenant KEK
5. Store encrypted snapshot (Temporal history or database)
6. Zero plaintext from memory

**Optional: credential scrubbing before encryption:**
```go
// Scrub known credential patterns before storage
var credentialPatterns = []regexp.Regexp{
    regexp.MustCompile(`(?i)(password|secret|key)\s+\d?\s*\S+`),
    regexp.MustCompile(`(?i)community\s+\S+`),
    regexp.MustCompile(`(?i)pre-shared-key\s+\S+`),
}

func scrubCredentials(config string) string {
    for _, pattern := range credentialPatterns {
        config = pattern.ReplaceAllString(config, "$1 [SCRUBBED]")
    }
    return config
}
```

### 7.3 Open Questions

- [ ] Should snapshots be stored in the vault rather than in Temporal history?
- [ ] Is credential scrubbing sufficient, or must full configs be encrypted?
- [ ] What is the retention policy for config snapshots? (Needed for rollback, but how long?)
- [ ] Should the Temporal `EncryptedDataConverter` handle snapshot encryption, or should it be separate?

---

## 8. Valkey Security Posture

### 8.1 Current State

Valkey (Redis fork) stores session tokens, cached data, and pub/sub for WebSocket fan-out. It is not mentioned in the security model, security checklist, or any attack surface analysis.

### 8.2 Required Security Controls

| Control | Recommendation | Priority |
|---------|---------------|----------|
| **TLS** | Require TLS for all Valkey connections | Phase 1A |
| **Authentication** | Valkey ACL with per-service credentials (not shared password) | Phase 1A |
| **Network isolation** | Valkey listens on internal network only (not 0.0.0.0) | Phase 1A |
| **Persistence encryption** | RDB/AOF files encrypted at rest (OS-level LUKS or Valkey enterprise) | Phase 5 |
| **Key expiry** | All session tokens have TTL matching JWT expiry | Phase 1B |
| **Command restriction** | Disable dangerous commands: `FLUSHALL`, `FLUSHDB`, `CONFIG`, `DEBUG`, `KEYS` | Phase 1A |
| **Memory limit** | Set `maxmemory` to prevent OOM on the host | Phase 1A |
| **Monitoring** | Alert on auth failures, connection spikes, memory pressure | Phase 2 |

### 8.3 Valkey ACL Configuration

```
# Per-service ACLs
user api-server on >$API_PASSWORD ~session:* ~cache:* +get +set +del +expire +ttl
user websocket on >$WS_PASSWORD ~pubsub:* +subscribe +publish
user worker on >$WORKER_PASSWORD ~idempotency:* ~ratelimit:* +get +set +del +expire +incr
user default off  # disable default user
```

### 8.4 Open Questions

- [ ] Should Valkey be added to the attack surface analysis with its own vector set?
- [ ] Is Valkey Sentinel or Valkey Cluster required for HA deployments?
- [ ] Should session tokens in Valkey be encrypted, or is the opaque JWT format sufficient?

---

## 9. OIDC Tenant Claim Verification

### 9.1 Current State

JWTs contain a `tenant_id` claim. JWT validation checks signature, expiry, issuer, and audience. The API extracts `tenant_id` and uses it for all tenant-scoped operations. There is no independent verification that the user actually belongs to the claimed tenant.

### 9.2 Threat Scenario

1. OIDC provider is misconfigured (custom claim mapping allows self-service `tenant_id`)
2. Attacker registers with the OIDC provider
3. Attacker sets their `tenant_id` claim to a target tenant
4. JWT is validly signed by the trusted OIDC provider
5. LOOM accepts the JWT and grants access to the target tenant

### 9.3 Recommendation: User-Tenant Membership Table

```go
type TenantMembership struct {
    UserID      string        // from JWT sub claim
    TenantID    uuid.UUID
    Role        string        // admin, operator, viewer
    GrantedBy   string        // who added this user
    GrantedAt   time.Time
    ExpiresAt   *time.Time    // optional time-limited access
}

// Middleware: verify membership on every request
func VerifyTenantMembership(ctx context.Context, claims JWTClaims) error {
    membership, err := db.GetMembership(claims.UserID, claims.TenantID)
    if err != nil {
        return ErrAccessDenied  // user not a member of claimed tenant
    }
    if membership.ExpiresAt != nil && membership.ExpiresAt.Before(time.Now()) {
        return ErrMembershipExpired
    }
    // Set verified tenant context
    ctx = WithVerifiedTenant(ctx, claims.TenantID, membership.Role)
    return nil
}
```

### 9.4 Open Questions

- [ ] Should tenant membership be the sole authority, or should the JWT claim be trusted when membership exists?
- [ ] How is the first admin added to a new tenant? (Bootstrap problem)
- [ ] Should membership verification be cached (Valkey, 5-minute TTL) to avoid per-request DB queries?
- [ ] How does this interact with NATS subject isolation? (NATS trusts JWT claims — needs independent verification too)

---

## 10. Implementation Priority

| Item | Phase | Effort | Dependencies |
|------|-------|--------|--------------|
| Valkey security hardening (TLS, ACL, commands) | 1A | 1 week | Valkey deployment |
| OIDC tenant membership verification | 1B | 1-2 weeks | Auth middleware |
| ConfigSnapshot encryption | 2 | 1-2 weeks | Vault integration |
| Audit trail hash chain (schema + computation) | 2 | 2-3 weeks | Audit model |
| Break-glass specification (reconcile inconsistencies) | 2 | 1 week | Security policy review |
| Temporal DataConverter per-namespace key | 4 | 2-3 weeks | Temporal integration |
| LLM single-turn enforcement | 8 | 1 week | LLM integration |
| External audit anchoring (S3 Object Lock) | 10 | 1-2 weeks | Audit hash chain |
| Internal CA lifecycle (step-ca or equivalent) | 10 | 3-4 weeks | PKI design |

---

## 11. References

- SECURITY-MODEL.md — Core security architecture
- VAULT-ARCHITECTURE.md — Credential encryption design
- AUDIT-MODEL.md — Audit record structure
- ADAPTER-CONTRACT.md — ConfigSnapshot definition
- DEPLOYMENT-TOPOLOGY.md — Component placement and TLS
- ATTACK-SURFACE-*.md — 71-vector security analysis
- LLM-BOUNDARIES.md — LLM safety constraints
- API-CONVENTIONS.md — Authentication middleware
