# LOOM Security Governance Policy

> LOOM manages credentials for every piece of managed infrastructure. This policy defines the governance processes that maintain LOOM's security posture across its lifecycle: from vulnerability disclosure to incident response, from dependency management to penetration testing.

**Revision:** 2026-03-21
**Classification:** PUBLIC (this document is intentionally public to support responsible disclosure)
**Owner:** Security Engineering
**Review Cadence:** Quarterly, or immediately following any security incident

---

## 1. Vulnerability Disclosure Process

### Responsible Disclosure

LOOM operates a responsible disclosure program. We ask security researchers to report vulnerabilities privately before public disclosure.

**Report a vulnerability:**

- **Email:** security@loom-project.dev
- **PGP Key:** Available at `https://loom-project.dev/.well-known/security.txt` and on public keyservers (key ID published in repository)
- **PGP Fingerprint:** Published in the repository root `SECURITY.md`
- **Preferred format:** Encrypted email with PGP. If PGP is not available, plaintext email is accepted.

### Disclosure Timeline

| Day | Action |
|-----|--------|
| 0 | Vulnerability report received; acknowledgment sent within 24 hours |
| 1-3 | Triage: severity assessment, reproduction, assignment to engineering team |
| 7 | Initial status update to reporter (confirmed/rejected, expected fix timeline) |
| 30 | Progress update to reporter |
| 60 | Fix developed, tested, and staged for release |
| 90 | **Coordinated public disclosure.** Fix released. CVE assigned if applicable. Reporter credited (unless they request anonymity). |

**Exceptions:**

- If the vulnerability is actively exploited in the wild, the fix timeline is accelerated (see Section 5: Security Update Policy).
- If the reporter does not respond to status updates for 30 days, LOOM reserves the right to proceed with disclosure.
- The 90-day timeline may be extended by mutual agreement if the fix requires architectural changes.

### Scope

The following are in scope for vulnerability reports:

- LOOM core components (API, vault, adapters, workflow engine, web UI)
- LOOM's cryptographic implementation (envelope encryption, key management)
- Multi-tenant isolation mechanisms (RLS, NATS accounts, Temporal namespaces)
- LLM integration security (prompt injection, credential exfiltration, tool abuse)
- Supply chain integrity (dependency management, build pipeline, container images)

The following are out of scope:

- Vulnerabilities in upstream dependencies (report these to the upstream project; notify LOOM if the vulnerability affects LOOM's usage)
- Social engineering attacks against LOOM operators
- Denial of service via volumetric network floods (infrastructure-level mitigation)
- Findings from automated scanners without a demonstrated exploit path

### Safe Harbor

LOOM will not pursue legal action against researchers who:

- Act in good faith to avoid privacy violations, data destruction, and service disruption
- Report findings promptly and do not publicly disclose before the coordinated disclosure date
- Do not access, modify, or exfiltrate data belonging to LOOM users or tenants
- Do not exploit vulnerabilities beyond what is necessary to demonstrate the issue

---

## 2. FIPS 140-2 Compatibility

### Current Status

LOOM uses Go's `crypto/tls` and `crypto/aes` packages. FIPS 140-2 compliance is achievable through Go's BoringCrypto integration.

### BoringCrypto Mode

Go supports FIPS 140-2 validated cryptography via the BoringCrypto module (a FIPS-validated fork of BoringSSL linked into the Go runtime). When built with the `GOEXPERIMENT=boringcrypto` build tag:

- `crypto/tls` uses BoringSSL for TLS handshakes
- `crypto/aes`, `crypto/cipher` use BoringSSL's AES-GCM implementation
- `crypto/ecdsa`, `crypto/rsa` use BoringSSL's implementations
- `crypto/rand` sources entropy through BoringSSL's CSPRNG

**Build command for FIPS mode:**

```bash
GOEXPERIMENT=boringcrypto CGO_ENABLED=1 go build -o loom-fips ./cmd/loom
```

### Algorithm Restrictions Under FIPS

When FIPS mode is enabled, the following algorithm restrictions apply:

| LOOM Component | Standard Algorithm | FIPS-Compliant Alternative | Notes |
|---------------|-------------------|--------------------------|-------|
| Credential encryption | AES-256-GCM | AES-256-GCM | No change required; AES-GCM is FIPS-approved |
| Key derivation | Argon2id | PBKDF2-HMAC-SHA-256 | Argon2id is not FIPS-approved; PBKDF2 must be used in FIPS mode with minimum 100,000 iterations |
| Integrity hashing | BLAKE3 | SHA-256 or SHA-384 | BLAKE3 is not FIPS-approved; SHA-2 family required |
| Digital signatures | Ed25519 | ECDSA P-256 or P-384 | Ed25519 is not FIPS-approved (as of FIPS 186-5 draft); ECDSA is the compliant alternative |
| Key exchange | X25519 | ECDH P-256 or P-384 | X25519 is not FIPS-approved; NIST curves required |
| TLS | TLS 1.2+ | TLS 1.2+ | No change; restrict cipher suites to FIPS-approved (AES-GCM, ECDHE) |

**Implementation note:** LOOM's `SECURITY-MODEL.md` specifies compile-time algorithm constants. FIPS mode requires a separate build target (`loom-fips`) with the FIPS-compliant algorithm set. Both builds are tested in CI.

### FIPS Deployment Checklist

- [ ] Built with `GOEXPERIMENT=boringcrypto`
- [ ] BoringCrypto module version is FIPS 140-2 validated (check NIST CMVP certificate)
- [ ] Algorithm constants switched to FIPS-approved set
- [ ] TLS cipher suites restricted to FIPS-approved algorithms
- [ ] KDF switched from Argon2id to PBKDF2-HMAC-SHA-256 (minimum 100,000 iterations)
- [ ] FIPS mode flag logged at startup for audit trail

---

## 3. Supply Chain Security

### Dependency Vetting Policy

All dependencies (Go modules and npm packages) must meet the following criteria before inclusion:

**Go Modules:**

| Criterion | Requirement |
|-----------|-------------|
| License | OSI-approved license compatible with LOOM's license |
| Maintenance | Active maintainer(s) with commits in the last 6 months, or declared stable/frozen |
| Security history | No unpatched critical CVEs |
| Transitive dependencies | Reviewed for the same criteria (recursive) |
| Checksum verification | `go.sum` committed to version control; `go mod verify` passes in CI |
| Proxy configuration | `GOPRIVATE` set for internal modules; `GONOSUMDB` and `GONOSUMCHECK` must NOT be set |
| New dependency approval | New direct dependencies require security team review before merge |

**npm Packages:**

| Criterion | Requirement |
|-----------|-------------|
| Lock file | `package-lock.json` committed; CI uses `npm ci` (never `npm install`) |
| Audit | `npm audit --audit-level=critical` must pass in CI |
| Lifecycle scripts | `ignore-scripts=true` by default; exceptions documented and approved |
| Provenance | Prefer packages with npm provenance attestations |
| New dependency approval | New direct dependencies require security team review |
| Continuous monitoring | Socket.dev or Snyk configured for supply chain attack detection (obfuscated code, unexpected network access, filesystem access) |

### SBOM Generation

LOOM generates a Software Bill of Materials (SBOM) for every release:

**Go backend:**

```bash
# Generate SPDX SBOM from Go binary
go version -m loom | syft packages -o spdx-json > sbom-backend.spdx.json
```

**npm frontend:**

```bash
# Generate CycloneDX SBOM from npm
npx @cyclonedx/cdxgen -o sbom-frontend.cdx.json
```

**SBOM distribution:**

- SBOMs are published alongside release artifacts
- SBOMs are signed with the same cosign key as the release binaries
- SBOMs are stored in an OCI registry as attestations (via `cosign attach sbom`)

### Binary Signing

All release binaries and container images are signed using Sigstore/cosign with keyless signing:

```bash
# Sign binary
cosign sign-blob --yes \
  --output-signature loom.sig \
  --output-certificate loom.cert \
  loom

# Sign container image
cosign sign --yes ghcr.io/loom-project/loom:v1.2.3
```

**Verification is mandatory before deployment:**

```bash
# Verify binary
cosign verify-blob \
  --signature loom.sig \
  --certificate loom.cert \
  --certificate-identity=ci@loom-project.dev \
  --certificate-oidc-issuer=https://token.actions.githubusercontent.com \
  loom

# Verify container image
cosign verify \
  --certificate-identity=ci@loom-project.dev \
  --certificate-oidc-issuer=https://token.actions.githubusercontent.com \
  ghcr.io/loom-project/loom:v1.2.3
```

### Reproducible Builds

LOOM's Go backend produces reproducible binaries:

```bash
CGO_ENABLED=0 GOOS=linux GOARCH=amd64 \
  go build -trimpath -ldflags='-s -w -buildid=' -o loom ./cmd/loom
```

Any party can verify that the published binary was built from the published source code by reproducing the build and comparing checksums.

---

## 4. Incident Response Procedure

### Phase 1: Detection

| Source | Detection Mechanism |
|--------|-------------------|
| Credential access anomaly | Vault audit log rate monitoring (ref: ATTACK-SURFACE-VAULT.md) |
| Cross-tenant access attempt | RLS violation alerts, NATS permissions violations, Temporal authorization denials |
| Authentication brute force | Per-IP and per-account rate limiting alerts (ref: ATTACK-SURFACE-API.md) |
| LLM manipulation | Confidence calibration gap alerts (ref: ATTACK-SURFACE-LLM.md) |
| Adapter compromise | Circuit breaker state changes, SSH host key changes (ref: ATTACK-SURFACE-ADAPTERS.md) |
| Supply chain | Dependency audit failures, SBOM drift, signature verification failures |
| External report | security@loom-project.dev, bug bounty program |

### Phase 2: Containment (Target: within 1 hour of detection)

**Immediate containment actions by severity:**

| Severity | Containment Action |
|----------|-------------------|
| **CRITICAL** (active credential exfiltration, cross-tenant breach) | Rotate all affected credentials immediately. Revoke all active sessions for affected tenants. Isolate compromised components at the network level. Page security on-call. |
| **HIGH** (authentication bypass, privilege escalation, adapter compromise) | Revoke affected sessions. Block source IPs/accounts. Disable affected feature/endpoint. Notify security on-call within 15 minutes. |
| **MEDIUM** (information disclosure, DoS, non-credential data exposure) | Block source. Enable enhanced logging. Notify security team during business hours. |
| **LOW** (minor information leak, non-exploitable finding) | Log finding. Schedule fix in next sprint. |

**Containment checklist:**

- [ ] Affected tenant(s) identified and notified
- [ ] Blast radius assessed (which devices, credentials, and data were potentially accessed)
- [ ] Evidence preserved (logs, memory dumps, network captures) before any remediation
- [ ] Containment action logged in incident tracking system
- [ ] Legal and compliance notified if regulated data (PII, PCI, HIPAA) is involved

### Phase 3: Eradication

- Identify root cause through log analysis, code review, and forensic examination
- Develop and test fix in isolated environment
- Verify fix addresses root cause, not just symptoms
- Verify fix does not introduce new vulnerabilities (security review required)
- If supply chain compromise: audit all dependencies, rebuild from verified sources

### Phase 4: Recovery

- Deploy fix through standard change management (expedited for CRITICAL/HIGH)
- Rotate all potentially compromised credentials (device passwords, API keys, TLS certificates, NKeys)
- Verify tenant data integrity (compare against known-good backups if available)
- Re-enable any disabled features/endpoints after fix verification
- Monitor for recurrence with enhanced alerting for 30 days

### Phase 5: Post-Mortem

A blameless post-mortem is conducted within 5 business days of incident resolution:

**Post-mortem template:**

1. **Timeline:** Minute-by-minute chronology from first indicator to full resolution
2. **Impact:** Tenants affected, data exposed, service disruption duration
3. **Root cause:** Technical root cause with supporting evidence
4. **Detection gap:** How long was the vulnerability exploitable before detection? Why?
5. **Response assessment:** What went well? What could be improved?
6. **Action items:** Concrete, assignable tasks with deadlines to prevent recurrence
7. **Attack surface update:** Update the relevant `ATTACK-SURFACE-*.md` document with the new finding

**Distribution:** Post-mortem shared with engineering, security, and affected tenant contacts (redacted version for tenants that excludes internal details).

---

## 5. Security Update Policy

### Patching Timelines

| Severity | SLA | Definition |
|----------|-----|-----------|
| **CRITICAL** | Patch within **48 hours** | Remote code execution, credential exfiltration, cross-tenant data breach, authentication bypass, active exploitation in the wild |
| **HIGH** | Patch within **7 days** | Privilege escalation, significant information disclosure, denial of service against critical functions, single-tenant data exposure |
| **MEDIUM** | Patch within **30 days** | Limited information disclosure, non-critical DoS, defense-in-depth bypass |
| **LOW** | Patch within **90 days** | Minor information leaks, hardening improvements, non-exploitable findings |

### Process

1. **Triage:** Security team assesses severity within 4 hours of report/detection
2. **Assign:** Fix assigned to owning team with SLA deadline
3. **Develop:** Fix developed on a private branch (not public until disclosure)
4. **Review:** Security team reviews fix for completeness and absence of regressions
5. **Test:** Fix verified against original exploit (if available) and regression test suite
6. **Release:** Emergency release for CRITICAL; next scheduled release for HIGH/MEDIUM/LOW (unless SLA requires earlier)
7. **Notify:** Affected users notified via security advisory. CVE requested if applicable.
8. **Disclose:** Public advisory published with fix details, impact assessment, and mitigation guidance

### Dependency CVE Response

When a CVE is published for a LOOM dependency:

| Dependency CVE Severity | Action |
|------------------------|--------|
| CRITICAL with known exploit | Update dependency within 48 hours; emergency release |
| CRITICAL without known exploit | Update dependency within 7 days |
| HIGH | Update dependency within 14 days |
| MEDIUM/LOW | Update in next scheduled release |

LOOM's CI pipeline runs `trivy image`, `npm audit`, and `govulncheck` on every build. CRITICAL findings fail the build.

---

## 6. Penetration Testing

### Third-Party Penetration Testing

| Requirement | Detail |
|-------------|--------|
| **Cadence** | Annual comprehensive pentest by an independent third-party firm |
| **Scope** | All LOOM components: API, vault, adapters, web UI, Temporal, NATS, PostgreSQL, LLM integration |
| **Methodology** | OWASP Testing Guide, PTES (Penetration Testing Execution Standard), NIST SP 800-115 |
| **Focus areas** | Multi-tenant isolation, credential vault security, LLM prompt injection, adapter protocol security |
| **Deliverables** | Executive summary, detailed findings with CVSS scores, remediation recommendations, retest of critical findings |
| **Remediation** | Critical findings remediated before retest; all findings tracked to resolution |
| **Retest** | Critical and high findings retested within 30 days of remediation |

### Continuous Automated Testing

| Test Type | Tool | Cadence |
|-----------|------|---------|
| DAST (Dynamic Application Security Testing) | OWASP ZAP / Burp Suite automation | Weekly against staging |
| SAST (Static Application Security Testing) | Semgrep, gosec | Every CI build |
| Dependency scanning | govulncheck, npm audit, Trivy | Every CI build |
| Container scanning | Trivy | Every container image build |
| Infrastructure scanning | tfsec / Checkov (IaC), kube-bench (Kubernetes) | Weekly |
| Credential scanning | gitleaks, trufflehog | Every commit (pre-commit hook) |
| LLM security testing | Custom test suite (prompt injection, credential exfiltration, tool abuse) | Every CI build |

### Internal Red Team Exercises

Quarterly red team exercises targeting:

- Cross-tenant isolation (attempt to access another tenant's devices/credentials)
- Credential exfiltration (attempt to extract plaintext credentials from vault, logs, memory, or workflow history)
- LLM manipulation (attempt to use prompt injection to execute unauthorized operations)
- Adapter exploitation (attempt to pivot from a compromised device into LOOM)
- Supply chain (attempt to introduce a malicious dependency)

Results are documented in the same format as third-party pentest reports and tracked to resolution.

---

## 7. Data Classification

All data handled by LOOM is classified according to the following scheme. Classification determines encryption requirements, access controls, retention policies, and incident response priority.

### Classification Levels

| Level | Label | Definition | Examples in LOOM |
|-------|-------|-----------|-----------------|
| **1** | **TOP SECRET** | Compromise grants direct authenticated access to managed infrastructure. Breach is catastrophic. | Device credentials (passwords, SSH keys, API tokens, SNMP community strings), MEK/KEK/DEK key material, break-glass recovery keys |
| **2** | **CONFIDENTIAL** | Compromise reveals infrastructure architecture, enables targeted attacks, or exposes regulated data. | Device configurations (ACLs, routing tables, firewall rules), network topology maps, tenant metadata, workflow execution history with device details, audit logs |
| **3** | **INTERNAL** | Operational data with limited sensitivity. Useful for reconnaissance but not directly exploitable. | Telemetry and metrics (CPU, memory, interface counters), workflow status (running/failed/completed without details), system health data, non-sensitive logs |
| **4** | **PUBLIC** | Information intended for public consumption. | API documentation, security policy (this document), public-facing status page data |

### Controls by Classification

| Control | TOP SECRET | CONFIDENTIAL | INTERNAL | PUBLIC |
|---------|-----------|--------------|----------|--------|
| **Encryption at rest** | AES-256-GCM with 3-tier envelope encryption (DEK/KEK/MEK) | AES-256-GCM (database-level or volume encryption) | Volume encryption (LUKS/BitLocker) | None required |
| **Encryption in transit** | TLS 1.2+ mandatory; mTLS preferred | TLS 1.2+ mandatory | TLS 1.2+ mandatory | TLS 1.2+ recommended |
| **Access control** | Role-based + per-credential audit record + rate limiting | Role-based + tenant scoping | Role-based | Public |
| **Retention** | Encrypted at rest; plaintext only in mlock'd memory; zeroed after use | Standard retention policy (configurable per tenant) | 90-day default retention | Indefinite |
| **Audit logging** | Every access logged with requester identity, timestamp, and operation | Access logged at API level | Aggregated metrics only | None |
| **Incident priority** | CRITICAL -- 48-hour patch SLA | HIGH -- 7-day patch SLA | MEDIUM -- 30-day patch SLA | LOW |
| **Backup handling** | Backups contain only ciphertext; backup encryption key stored separately from backup media | Encrypted backups | Standard backup procedures | N/A |
| **Memory protection** | mlock, PR_SET_DUMPABLE=0, explicit zeroing, no GC-managed strings | Standard process isolation | Standard process isolation | N/A |
| **Log redaction** | MUST be redacted (`***REDACTED***`) in all log output, error messages, and serialization | Tenant-scoped; cross-tenant redaction | No special redaction | N/A |

### Classification Responsibilities

- **Data owners** (engineering team leads) are responsible for classifying data their components handle.
- **Security engineering** reviews classifications quarterly and after any schema change.
- **All engineers** are responsible for handling data according to its classification and escalating if classification is unclear.
- **New data fields** added to the schema must be classified before the PR is merged. The PR template includes a data classification section.

---

## References

- [SECURITY-MODEL.md](../SECURITY-MODEL.md) -- Cryptographic primitives, threat model, hardware requirements
- [VAULT-ARCHITECTURE.md](../VAULT-ARCHITECTURE.md) -- Envelope encryption, key hierarchy, MEK providers
- [ATTACK-SURFACE-POSTGRESQL.md](ATTACK-SURFACE-POSTGRESQL.md) -- Database attack vectors and defenses
- [ATTACK-SURFACE-NATS.md](ATTACK-SURFACE-NATS.md) -- Message bus attack vectors and defenses
- [ATTACK-SURFACE-TEMPORAL.md](ATTACK-SURFACE-TEMPORAL.md) -- Workflow engine attack vectors and defenses
- [ATTACK-SURFACE-API.md](ATTACK-SURFACE-API.md) -- REST API attack vectors and defenses
- [ATTACK-SURFACE-VAULT.md](ATTACK-SURFACE-VAULT.md) -- Credential vault attack vectors and defenses
- [ATTACK-SURFACE-LLM.md](ATTACK-SURFACE-LLM.md) -- LLM integration attack vectors and defenses
- [ATTACK-SURFACE-ADAPTERS.md](ATTACK-SURFACE-ADAPTERS.md) -- Protocol adapter attack vectors and defenses
- [ATTACK-SURFACE-UI-SUPPLY-CHAIN.md](ATTACK-SURFACE-UI-SUPPLY-CHAIN.md) -- Web UI and supply chain attack vectors and defenses
