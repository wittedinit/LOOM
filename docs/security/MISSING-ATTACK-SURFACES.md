# Missing Attack Surfaces

> **Date:** 2026-03-21
> **Source:** Adversarial security audit — 7 attack surfaces not covered by existing threat models
> **Status:** All 7 surfaces addressed with concrete hardening requirements
> **Severity:** These are systemic gaps, not individual findings. Each represents an entire category of attacks that the existing security documentation does not model.

---

## 1. Kubernetes Hardening

**Gap:** LOOM runs on Kubernetes but has no Kubernetes-specific security requirements documented. The existing attack surface analyses cover PostgreSQL, NATS, Vault, Temporal, LLM, UI, and adapters — but not the orchestration platform itself.

### Requirements

**Network Policies (deny-all default):**

Every LOOM namespace starts with a deny-all ingress and egress policy. Explicit allow rules are added per-service:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-all
spec:
  podSelector: {}
  policyTypes:
    - Ingress
    - Egress
```

Allowed traffic (explicit policies per service):

| Source | Destination | Port | Justification |
|--------|-------------|------|---------------|
| loom-api | loom-db | 5432 | Database access |
| loom-api | loom-nats | 4222 | Event publish |
| loom-api | loom-valkey | 6379 | Session cache |
| loom-worker | loom-db | 5432 | Workflow state |
| loom-worker | loom-temporal | 7233 | Workflow engine |
| loom-worker | loom-vault | 8200 | Credential retrieval |
| loom-agent | loom-nats | 4222 | Event relay |
| ingress | loom-api | 8443 | External API access |

All other traffic is denied. No pod-to-pod communication outside explicit rules.

**PodSecurityStandards (restricted profile):**

All LOOM pods run under the `restricted` PodSecurityStandard:
- `runAsNonRoot: true`
- `allowPrivilegeEscalation: false`
- `readOnlyRootFilesystem: true` (with explicit volume mounts for writable paths)
- `seccompProfile: RuntimeDefault`
- `capabilities.drop: ["ALL"]`
- No `hostNetwork`, `hostPID`, or `hostIPC`
- No privileged containers

**RBAC minimums:**

Per-service Kubernetes service accounts. No shared service account across services. Each service account has the minimum RBAC permissions needed:

| Service Account | Permissions | Scope |
|----------------|------------|-------|
| loom-api | get/list pods (health checks only) | loom namespace |
| loom-worker | get/list/create jobs (for batch adapters) | loom namespace |
| loom-agent | none | loom-edge namespace |
| loom-db | none | loom namespace |

No service account has `cluster-admin`, `edit`, or `admin` ClusterRoleBindings.

**etcd encryption:** Enable encryption at rest for Kubernetes secrets using AES-CBC or AES-GCM with a KMS provider. LOOM secrets stored in Kubernetes (TLS certs, service account tokens) are encrypted in etcd, not stored as base64-encoded plaintext.

**Admission controllers:** Deploy OPA/Gatekeeper or Kyverno with policies that enforce:
- No `latest` image tags (all images must be pinned to SHA digest)
- No privileged containers
- No host path mounts outside explicit allowlist
- Resource limits required on all containers (prevents fork bombs and resource exhaustion)
- Image registry restricted to signed images from trusted registries

**Cross-reference:** DEPLOYMENT-TOPOLOGY.md, CI-CD-PIPELINE.md (image signing)

---

## 2. DNS Attacks

**Gap:** No documentation addresses DNS-based attacks despite LOOM relying on DNS for service discovery (Kubernetes CoreDNS), external lookups (device hostname resolution), and OIDC provider discovery.

### Requirements

**DNSSEC validation for external lookups:** All DNS lookups to external domains (OIDC providers, NTP servers, cloud APIs) MUST be validated with DNSSEC where the domain supports it. The LOOM resolver validates DNSSEC signatures and rejects responses with invalid signatures.

**Internal DNS pinning:** For Kubernetes-internal service discovery, DNS responses are validated against known service IPs from the Kubernetes API. If DNS returns an IP that does not match any known endpoint for that service, the request is rejected and an alert is emitted. This prevents DNS cache poisoning within the cluster.

**SSRF filter bypass awareness:** DNS rebinding attacks can bypass SSRF filters by resolving to an internal IP after the initial validation check resolves to an external IP. LOOM's HTTP client validates the resolved IP at connection time (not just at DNS resolution time). Internal CIDR ranges (10.0.0.0/8, 172.16.0.0/12, 192.168.0.0/16, 127.0.0.0/8, 169.254.169.254/32) are blocked for outbound connections unless explicitly configured.

```go
type DNSSecurityConfig struct {
    ValidateDNSSEC      bool     // true for external lookups
    PinInternalDNS      bool     // true — validate against K8s API
    BlockedCIDRs        []string // RFC 1918, loopback, link-local, metadata
    AllowedInternalCIDRs []string // explicit exceptions for known internal services
}
```

**Cross-reference:** DEPLOYMENT-TOPOLOGY.md, SECURITY-MODEL.md

---

## 3. Insider Threat Model

**Gap:** The security model addresses external attackers and compromised components but does not model threats from authorized LOOM operators or developers with legitimate access.

### Requirements

**Least privilege per role:**

| Role | Can Deploy | Can Approve Workflows | Can Access Credentials | Can Modify Audit | Can Access Vault Keys |
|------|-----------|----------------------|----------------------|-----------------|---------------------|
| Developer | Yes (non-prod) | No | No | No | No |
| Operator | Yes (all) | Yes | Read-only (masked) | No | No |
| Security Admin | No | Yes | Yes (plaintext) | View only | Yes (with ceremony) |
| Super Admin | Yes | Yes | Yes | View only | Yes (with ceremony) |

No single role can deploy code AND approve workflows AND access credentials. This prevents a single compromised account from executing a complete attack chain.

**Separation of duties:**
- Code deployment requires approval from someone who did NOT author the code
- Workflow approval for production devices requires approval from someone who did NOT create the workflow
- Vault key access (MEK/KEK operations) requires the break-glass ceremony (dual authorization)
- Audit log export requires security admin approval

**Mandatory code review:** All changes to LOOM core (hub, worker, vault, auth middleware) require at least one reviewer who is not the author. Direct push to main/release branches is blocked.

**Background checks:** Personnel with access to vault keys, HSM PINs, or production database credentials undergo background checks appropriate to the sensitivity level. This is a policy requirement, not a technical control.

**Cross-reference:** SECURITY-MODEL.md (role hierarchy), SECURITY-POLICY.md

---

## 4. Physical Edge Security

**Gap:** Edge agents are deployed in potentially hostile physical environments (remote data centers, edge cabinets, customer premises). Physical access attacks are not modeled.

### Requirements

**TPM mandatory for credential caching:** Already addressed in C-02 and EDGE-AGENT-SECURITY.md. TPM 2.0 is the preferred KEK source. When TPM is unavailable, Argon2id-derived encryption is the mandatory minimum.

**Tamper-evident seals:** Edge agent hardware SHOULD have tamper-evident seals on the chassis. On boot, the agent checks TPM PCR values against expected measurements. If PCR values have changed (indicating hardware tampering, BIOS modification, or boot chain alteration), the agent:
1. Refuses to unseal cached credentials
2. Reports the tamper event to the hub (if reachable)
3. Enters a degraded mode: can receive commands from the hub but cannot execute operations requiring cached credentials

**Cold boot mitigation:** Where available, enable AMD SME (Secure Memory Encryption) or Intel TME (Total Memory Encryption) to encrypt DRAM contents. This prevents cold boot attacks that freeze RAM and read credential material from memory.

These are hardware-dependent features and cannot be mandated universally. The agent SHOULD detect and enable them where available, and report their availability to the hub as a security posture fact.

```go
type EdgeSecurityPosture struct {
    TPMAvailable      bool
    TPMVersion        string // "2.0" or ""
    MemoryEncryption  string // "amd_sme", "intel_tme", "none"
    SecureBootEnabled bool
    PCRIntegrity      bool   // true if PCR values match expected
}
```

**Cross-reference:** EDGE-AGENT-SECURITY.md, VAULT-ARCHITECTURE.md (TPM integration)

---

## 5. OIDC Provider Compromise

**Gap:** If the OIDC identity provider is compromised, an attacker can mint arbitrary JWTs that LOOM accepts. The existing mitigation (tenant membership verification in SECURITY-HARDENING-RESPONSES.md Section 8) limits blast radius but does not address the root cause.

### Requirements

**JWKS endpoint pinning:** The JWKS (JSON Web Key Set) endpoint URL is configured at deployment time and does NOT follow redirects. LOOM does not dynamically discover JWKS endpoints via `.well-known/openid-configuration` in production. This prevents an attacker who compromises DNS or the OIDC provider's web server from redirecting JWKS fetches to attacker-controlled keys.

**JWKS key pinning:** In addition to endpoint pinning, LOOM pins the expected JWKS key IDs (`kid`). If the OIDC provider presents a JWKS containing an unknown `kid`, the key is NOT trusted until an operator explicitly approves it. This prevents an attacker who compromises the OIDC provider from adding a new signing key.

**Multiple OIDC providers for redundancy:** Support configuration of 2+ OIDC providers. If the primary provider is unavailable or compromised (detected via JWKS pinning violation), authentication falls back to the secondary provider. This prevents a single OIDC provider compromise from being a complete authentication bypass.

**Token binding to client certificate:** For high-security deployments, JWT tokens are bound to the client's mTLS certificate. The JWT `cnf` (confirmation) claim contains the certificate thumbprint. A token presented without the matching client certificate is rejected. This prevents stolen tokens from being used on a different machine.

```go
type OIDCProviderConfig struct {
    ProviderURL     string   // pinned, no redirect following
    JWKSEndpoint    string   // pinned URL
    PinnedKeyIDs    []string // expected kid values
    Priority        int      // lower = preferred
    TokenBindingCNF bool     // require certificate thumbprint in cnf claim
}
```

**Cross-reference:** SECURITY-HARDENING-RESPONSES.md Section 8 (OIDC tenant verification), SECURITY-MODEL.md

---

## 6. Backup Security

**Gap:** No documentation addresses the security of LOOM backups. Database backups, vault snapshots, and configuration exports contain the same sensitive data as the live system but may be stored with weaker protections.

### Requirements

**Separate encryption key:** Backups are encrypted with a dedicated backup encryption key, NOT the same MEK used for the live vault. If the MEK is compromised, backups remain protected (and vice versa). The backup key is stored in a separate key management path with independent access controls.

**Backup access logging:** Every backup creation, access, download, and restore operation is logged to the audit trail. The log entry includes who initiated the operation, when, and from what source IP.

**Restore testing:** Backup restores are tested quarterly in an isolated environment. The test validates:
1. Backup decryption succeeds
2. Database schema integrity is maintained
3. Vault credential retrieval works after restore
4. Audit trail hash chain is intact after restore

**Storage access restriction:** Backup storage (S3 bucket, NFS share, or local path) has independent access controls from the production system. Production service accounts cannot access backup storage. Backup operations use a dedicated service account with write-only access (cannot read existing backups — only the restore service account can read).

```go
type BackupSecurityConfig struct {
    EncryptionKeyPath  string // vault path, separate from MEK
    StoragePath        string // S3 URI, NFS path, or local path
    WriteOnlyAccount   string // can create backups, cannot read
    ReadOnlyAccount    string // can read for restore, cannot delete
    RetentionDays      int    // default: 90
    RestoreTestCron    string // "0 0 1 */3 *" — quarterly
}
```

**Cross-reference:** VAULT-ARCHITECTURE.md (encryption hierarchy), SECURITY-CHECKLIST.md (add backup verification items)

---

## 7. Application-as-Vault-Accessor

**Gap:** The LOOM application process holds vault credentials and can access any credential in its tenant scope. A compromised application process becomes a credential oracle — it can retrieve any credential for any device the tenant manages.

### Acknowledgment

This is an architectural limitation shared by every application that integrates with a secrets manager. The application must be able to retrieve credentials to use them. There is no architecture where the application can use credentials it cannot access.

### Mitigations

**Per-call authentication (not ambient):** Vault operations require per-call authentication tokens with short TTLs (5 minutes). The application does NOT hold a long-lived vault token. Each credential retrieval requires a fresh token obtained via the Kubernetes auth method or AppRole auth method.

**Rate limiting on vault API:** The vault enforces rate limits per client identity:
- Maximum 100 credential retrievals per minute per service account
- Maximum 10 unique credentials per minute (prevents bulk enumeration)
- Anomaly alert if a service account accesses more than 5x its historical average in a 5-minute window

**Anomaly detection on access patterns:** The vault logs all credential access and runs pattern analysis:
- A service that normally accesses 3 credentials suddenly accessing 50 triggers an alert
- Access to credentials outside the service's historical pattern triggers an alert
- Access during unusual hours (relative to baseline) triggers an alert

These mitigations do not prevent a compromised application from accessing credentials it legitimately needs. They limit the blast radius by detecting and rate-limiting abnormal access patterns.

```go
type VaultAccessPolicy struct {
    MaxRetrievalsPerMinute    int // 100
    MaxUniqueCredsPerMinute   int // 10
    AnomalyMultiplier         float64 // 5.0x historical average
    TokenTTL                  time.Duration // 5 minutes
    AuthMethod                string // "kubernetes" or "approle"
}
```

**Cross-reference:** VAULT-ARCHITECTURE.md (access controls), SECURITY-MODEL.md

---

## FALSE ASSURANCES

The following are claims or implications in LOOM's documentation that overstate the actual security guarantee. Each is acknowledged honestly with the actual guarantee level documented.

---

### FA-01: "Zero Plaintext at Rest — Ever, Anywhere"

**Claim:** VAULT-ARCHITECTURE.md states credentials "never exist as plaintext" at rest.

**Actual guarantee:** Credentials are encrypted at rest in the database, encrypted in transit, and wiped from memory after use. However:
- During the brief window of active use (decryption to re-encryption), the plaintext exists in process memory. Memory-locking (`mlock`) prevents swap, but the plaintext is readable by a process with `CAP_SYS_PTRACE` or a kernel exploit.
- Core dumps are disabled, but a sufficiently privileged attacker (root on the host) can read process memory via `/proc/[pid]/mem`.
- Hardware memory encryption (AMD SME/Intel TME) mitigates physical access but is not universally available.

**Honest guarantee level:** Credentials are encrypted at rest and in transit. In-memory exposure is minimized to the shortest possible window with OS-level protections (mlock, no core dumps). A kernel-level or hypervisor-level compromise can read plaintext credentials during the use window.

---

### FA-02: "Saga Compensation Provides Rollback"

**Claim:** WORKFLOW-CONTRACT.md implies saga compensation can reliably roll back infrastructure changes.

**Actual guarantee:** Saga compensation is best-effort for most adapter types. Only adapters that pass the compensation conformance test (M-12) earn the `"guaranteed"` reliability level. For many protocols (IPMI, SSH, basic REST APIs), compensation is "attempt to reverse" — not a transactional rollback. If compensation fails, the system enters a `compensation_partial` state requiring operator intervention.

**Honest guarantee level:** Compensation provides automated rollback attempts. For verified adapters on supported protocols (NETCONF, Terraform, Ansible), rollback reliability is high. For other protocols, compensation is best-effort with operator alerting on failure.

---

### FA-03: "LLM Cannot Cause Outages Due to Validation Pipeline"

**Claim:** LLM-BOUNDARIES.md describes a multi-stage validation pipeline (schema check, syntax check, Batfish verification, dry-run, human approval) that prevents LLM-generated configs from causing harm.

**Actual guarantee:** The validation pipeline is sound for vendors supported by Batfish (Cisco IOS/NX-OS, Junos, Arista EOS). For the majority of LOOM's target platforms (MikroTik, SONiC, Cumulus, UniFi, OPNsense), Batfish verification is unavailable. These platforms get syntax checking and human review, but no semantic verification. A syntactically valid but logically wrong config (e.g., `permit` instead of `deny` in an ACL) can pass all automated checks.

**Honest guarantee level:** For Batfish-supported vendors, the pipeline provides strong protection against LLM errors. For other vendors, protection is limited to syntax validation and mandatory human review. Semantic correctness depends on the human reviewer.

---

### FA-04: "Binary Protection Prevents Reverse Engineering"

**Claim:** BINARY-PROTECTION.md describes garble, anti-debug, and string encryption measures.

**Actual guarantee:** These measures raise the cost of casual reverse engineering from "trivial" (1-2 hours) to "moderate" (1-2 days for a skilled reverse engineer). They do NOT prevent reverse engineering by a determined attacker with appropriate tools (IDA Pro, Ghidra, custom deobfuscation scripts). Go's runtime metadata requirements mean that significant structural information is always recoverable regardless of obfuscation.

**Honest guarantee level:** Binary protection deters casual inspection and automated tooling. It does not protect against targeted reverse engineering. Intellectual property protection ultimately depends on legal mechanisms (EULA, trade secrets, copyright), not binary obfuscation.

---

### FA-05: "Multi-Tenant Isolation Is Complete"

**Claim:** Various documents describe NATS accounts, RLS policies, per-tenant encryption keys, and namespace isolation as providing complete tenant isolation.

**Actual guarantee:** Tenant isolation is enforced at multiple layers (database, messaging, vault, Temporal). However:
- Shared physical infrastructure (Kubernetes nodes, network switches, hypervisors) means side-channel attacks are theoretically possible (cache timing, memory bus contention).
- Shared LLM inference (even with KV cache disabled) processes tenant data on the same GPU, where speculative execution or cache timing could theoretically leak information.
- The LOOM hub process itself is a shared component that holds all tenants' contexts in the same process memory space.
- Advisory lock hash collisions (M-10) can cause cross-tenant interference (denial of service, not data leak).

**Honest guarantee level:** Tenant isolation is enforced at the application, database, messaging, and encryption layers. It is NOT enforced at the hardware layer (CPU cache, GPU memory, network fabric). For deployments requiring hardware-level isolation, dedicated infrastructure per tenant is necessary. LOOM's isolation model is comparable to major SaaS platforms (AWS multi-tenant services, Snowflake) — strong logical isolation, shared physical infrastructure.

---

## References

- [ADVERSARIAL-AUDIT-MEDIUM-LOW.md](ADVERSARIAL-AUDIT-MEDIUM-LOW.md) — Medium and low findings from same audit
- [ADVERSARIAL-REVIEW.md](../ADVERSARIAL-REVIEW.md) — Original adversarial architecture review
- [SECURITY-MODEL.md](../SECURITY-MODEL.md) — Core security architecture
- [VAULT-ARCHITECTURE.md](../VAULT-ARCHITECTURE.md) — Credential vault design
- [EDGE-AGENT-SECURITY.md](../EDGE-AGENT-SECURITY.md) — Edge agent security model
- [DEPLOYMENT-TOPOLOGY.md](../DEPLOYMENT-TOPOLOGY.md) — Kubernetes deployment model
- [BINARY-PROTECTION.md](../BINARY-PROTECTION.md) — Binary protection measures
- [LLM-BOUNDARIES.md](../LLM-BOUNDARIES.md) — LLM validation pipeline
- [SECURITY-CHECKLIST.md](SECURITY-CHECKLIST.md) — Pre-deployment checklist
