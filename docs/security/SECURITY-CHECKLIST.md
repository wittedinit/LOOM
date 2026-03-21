# LOOM Pre-Deployment Security Audit Checklist

> This checklist must be completed before any LOOM deployment is placed into production. Each item references the specific attack surface document and section that justifies its inclusion. A deployment with any unchecked CRITICAL item is not authorized for production use.

**Revision:** 2026-03-21
**Classification:** INTERNAL -- Operations
**Owner:** Security Engineering

---

## How to Use This Checklist

1. Complete every item in order. Items marked **(CRITICAL)** are blocking -- the deployment MUST NOT proceed if any are unchecked.
2. For each item, record the date verified, the engineer who verified it, and any notes.
3. Re-verify after any infrastructure change (upgrades, migrations, scaling events).
4. Store the completed checklist alongside the deployment record in the change management system.

---

## 1. Infrastructure Prerequisites

### PostgreSQL

- [ ] **(CRITICAL)** PostgreSQL TLS enabled with `sslmode=verify-full` and CA certificate pinning
  - *Ref: [ATTACK-SURFACE-POSTGRESQL.md, Attack 4: Connection String Injection](ATTACK-SURFACE-POSTGRESQL.md) -- SSL downgrade attack enables MITM and credential interception*
  - Verify: `pg_hba.conf` contains `clientcert=verify-full` for all LOOM roles

- [ ] **(CRITICAL)** Password encryption set to `scram-sha-256` in `postgresql.conf`
  - *Ref: [ATTACK-SURFACE-POSTGRESQL.md, Hardened PostgreSQL Configuration](ATTACK-SURFACE-POSTGRESQL.md) -- `password_encryption = scram-sha-256` prevents hash downgrade*
  - Verify: `SHOW password_encryption;` returns `scram-sha-256`

- [ ] **(CRITICAL)** Row-Level Security (RLS) policies active and FORCED on all tenant-scoped tables
  - *Ref: [ATTACK-SURFACE-POSTGRESQL.md, Attack 2: Cross-Tenant Data Access](ATTACK-SURFACE-POSTGRESQL.md) -- missing WHERE tenant_id clause leaks data across tenants*
  - Verify: `SELECT relname, relrowsecurity, relforcerowsecurity FROM pg_class WHERE relname IN ('devices','credentials','workflows',...);` -- both columns must be `true`

- [ ] **(CRITICAL)** Application role (`loom_app`) is `NOSUPERUSER` with no file access roles
  - *Ref: [ATTACK-SURFACE-POSTGRESQL.md, Attack 6: COPY Command File Exfiltration](ATTACK-SURFACE-POSTGRESQL.md) -- superuser can read/write arbitrary files and execute OS commands*
  - Verify: `SELECT rolsuper, rolcreatedb, rolcreaterole FROM pg_roles WHERE rolname='loom_app';` -- all false
  - Verify: `pg_read_server_files`, `pg_write_server_files`, `pg_execute_server_program` revoked

- [ ] Application role `search_path` set to `loom_schema, pg_catalog` (no `public`)
  - *Ref: [ATTACK-SURFACE-POSTGRESQL.md, Attack 5: Privilege Escalation via search_path](ATTACK-SURFACE-POSTGRESQL.md) -- function shadowing via public schema*
  - Verify: `SHOW search_path;` for each LOOM role

- [ ] `CREATE` privilege revoked on `public` schema
  - *Ref: [ATTACK-SURFACE-POSTGRESQL.md, Attack 5](ATTACK-SURFACE-POSTGRESQL.md) -- prevents rogue object creation in public schema*

- [ ] Extension whitelist enforced (`timescaledb`, `age`, `vector`, `pgaudit`, `pgcrypto` only); untrusted languages removed
  - *Ref: [ATTACK-SURFACE-POSTGRESQL.md, Attack 7: Extension-Based Code Execution](ATTACK-SURFACE-POSTGRESQL.md) -- plpythonu enables reverse shell, dblink enables lateral movement*

- [ ] `pgaudit` enabled for `write`, `ddl`, and `role` operations
  - *Ref: [ATTACK-SURFACE-POSTGRESQL.md, Hardened PostgreSQL Configuration](ATTACK-SURFACE-POSTGRESQL.md)*

- [ ] `pg_hba.conf` restricts connections to known CIDR ranges; rejects all other connections
  - *Ref: [ATTACK-SURFACE-POSTGRESQL.md, Hardened PostgreSQL Configuration](ATTACK-SURFACE-POSTGRESQL.md) -- last line must be `host all all 0.0.0.0/0 reject`*

### NATS JetStream

- [ ] **(CRITICAL)** NATS accounts configured per tenant (not just subject-based isolation)
  - *Ref: [ATTACK-SURFACE-NATS.md, Attack 1: Cross-Tenant Event Snooping](ATTACK-SURFACE-NATS.md) -- subject-based isolation provides ZERO security; any client can subscribe to `loom.>` and see all tenants*
  - Verify: `nats server info` shows per-tenant accounts; test that `loom.>` wildcard from one account cannot see another

- [ ] **(CRITICAL)** NKey authentication enabled (no plaintext passwords in any config file)
  - *Ref: [ATTACK-SURFACE-NATS.md, Attack 4: Credential Exposure in NATS Config](ATTACK-SURFACE-NATS.md) -- plaintext passwords in config files are trivially extracted from git history, Docker layers, process listings*
  - Verify: No `password:` or `token:` directives in any NATS config; all authentication uses `nkey:` directives
  - Verify: NKey seed files have `0400` (owner-read-only) permissions

- [ ] **(CRITICAL)** TLS enabled for all client and leaf node connections
  - *Ref: [ATTACK-SURFACE-NATS.md, Attack 6: Leaf Node Man-in-the-Middle](ATTACK-SURFACE-NATS.md) -- plaintext leaf connections expose all device credentials, firmware URLs, and allow message injection*
  - Verify: All connection URLs use `tls://` scheme, never `nats://`
  - Verify: Leaf node TLS block has `verify: true` for mutual TLS

- [ ] Per-account JetStream resource limits configured (memory, storage, streams, consumers)
  - *Ref: [ATTACK-SURFACE-NATS.md, Attack 5: Denial of Service via Slow Consumer](ATTACK-SURFACE-NATS.md) -- one tenant's misbehaving consumer can exhaust server memory and degrade all tenants*
  - Verify: Each account has `max_mem`, `max_file`, `max_streams`, `max_consumers` set

- [ ] Per-account connection and subscription limits configured
  - *Ref: [ATTACK-SURFACE-NATS.md, Attack 5](ATTACK-SURFACE-NATS.md) -- connection flood and subscription exhaustion attacks*
  - Verify: Each account has `max_connections`, `max_subscriptions`, `max_payload` set

- [ ] System account (`SYS`) is separate from tenant accounts
  - *Ref: [ATTACK-SURFACE-NATS.md, Hardened Configuration Reference](ATTACK-SURFACE-NATS.md)*

### Temporal

- [ ] **(CRITICAL)** Mutual TLS (mTLS) enabled on all Temporal frontend and internode connections
  - *Ref: [ATTACK-SURFACE-TEMPORAL.md, Attack 6: gRPC API Exploitation](ATTACK-SURFACE-TEMPORAL.md) -- without mTLS, any process on the network can perform any operation including terminating workflows and reading execution histories*
  - Verify: `temporal-server.yaml` has `requireClientAuth: true` on both `frontend` and `internode` TLS blocks

- [ ] **(CRITICAL)** Custom Authorizer and ClaimMapper plugins configured
  - *Ref: [ATTACK-SURFACE-TEMPORAL.md, Attack 4: Namespace Escape](ATTACK-SURFACE-TEMPORAL.md) -- without authorization, any authenticated client can access any namespace including `temporal-system`*
  - Verify: `temporal-server.yaml` contains `authorizer: loom` and `claimMapper: loom`

- [ ] **(CRITICAL)** Namespace-per-tenant configured with `AllowedWorkerCertSANs`
  - *Ref: [ATTACK-SURFACE-TEMPORAL.md, Attack 4](ATTACK-SURFACE-TEMPORAL.md) -- namespaces are NOT a security boundary without authorization plugins*
  - Verify: Each tenant has a dedicated namespace; worker SANs are restricted per namespace

- [ ] Per-namespace rate limits configured (`frontend.namespaceRPS`)
  - *Ref: [ATTACK-SURFACE-TEMPORAL.md, Attack 5: Workflow Bombing](ATTACK-SURFACE-TEMPORAL.md) -- uncapped workflow creation exhausts Temporal resources*
  - Verify: Dynamic config contains `frontend.namespaceRPS`, `history.rps`, `matching.rps`

- [ ] Temporal frontend (port 7233) network-segmented to LOOM API and worker pods only
  - *Ref: [ATTACK-SURFACE-TEMPORAL.md, Attack 6](ATTACK-SURFACE-TEMPORAL.md) -- default Temporal is equivalent to an unauthenticated database*

### Valkey (Redis-Compatible Cache)

- [ ] **(CRITICAL)** `requirepass` set with a strong password (minimum 32 characters, random)
  - *Ref: [SECURITY-MODEL.md, Attack Surface](../SECURITY-MODEL.md) -- Valkey stores ephemeral data including nonce tracking, JTI blacklists, and rate limit state*

- [ ] **(CRITICAL)** TLS enabled for all Valkey client connections
  - *Ref: [SECURITY-MODEL.md](../SECURITY-MODEL.md) -- Valkey traffic contains security-sensitive data (deduplication keys, rate limit state)*

- [ ] Valkey not exposed to untrusted networks (bind to internal interfaces only)

---

## 2. Vault Configuration

- [ ] **(CRITICAL)** AES-NI detected and confirmed at startup; LOOM refuses to start without it
  - *Ref: [SECURITY-MODEL.md, Section 3: Hardware Instruction Set Requirements](../SECURITY-MODEL.md) -- AES-NI is mandatory; there is no degraded mode*
  - *Ref: [ATTACK-SURFACE-VAULT.md, Attack 9: Encryption Downgrade](ATTACK-SURFACE-VAULT.md) -- software AES without hardware acceleration is unacceptable for credential volumes*
  - Verify: Startup logs contain `INFO: AES-NI detected`; absence produces `FATAL` and exit

- [ ] **(CRITICAL)** MEK provider configured: HSM (PKCS#11) > HashiCorp Vault Transit > Cloud KMS > TPM 2.0 > file-based (air-gapped only)
  - *Ref: [VAULT-ARCHITECTURE.md, Section 2: Master Key](../VAULT-ARCHITECTURE.md) -- MEK source determines root-of-trust security level*
  - *Ref: [ATTACK-SURFACE-VAULT.md, Attack 10: Vault Backend Substitution](ATTACK-SURFACE-VAULT.md) -- backend is immutable after initialization; change attempts generate CRITICAL alerts*
  - Verify: Startup logs show `INFO: vault backend initialized: {provider_type}`
  - Verify: If `local_passphrase` in production, document explicit justification and risk acceptance

- [ ] **(CRITICAL)** Per-tenant KEK generated and encrypted by MEK; KEK status is `active`
  - *Ref: [VAULT-ARCHITECTURE.md, Section 2: Key Encryption Key](../VAULT-ARCHITECTURE.md) -- each tenant has independent KEK; cross-tenant credential access is structurally impossible*
  - Verify: `SELECT tenant_id, version, status FROM tenant_keks;` -- every tenant has at least one `active` KEK

- [ ] Credential rotation policies defined (DEK rotation on credential update; KEK rotation quarterly)
  - *Ref: [VAULT-ARCHITECTURE.md, Section 2: DEK and KEK](../VAULT-ARCHITECTURE.md) -- stale keys increase exposure window*
  - *Ref: [ATTACK-SURFACE-VAULT.md, Attack 8: AES-GCM Nonce Reuse](ATTACK-SURFACE-VAULT.md) -- DEK rotation every 90 days stays within NIST 2^32 invocation safety margin*

- [ ] Break-glass procedure documented and tested (MEK recovery from escrow, KEK re-wrap)
  - *Ref: [VAULT-ARCHITECTURE.md, Section 2: Master Key](../VAULT-ARCHITECTURE.md) -- if MEK provider becomes unavailable, all credentials are inaccessible without recovery procedure*
  - Verify: Break-glass procedure has been tested in a staging environment within the last 90 days

- [ ] **(CRITICAL)** Core dumps disabled: `PR_SET_DUMPABLE=0` (Linux) and `RLIMIT_CORE=0`
  - *Ref: [ATTACK-SURFACE-VAULT.md, Attack 2: Memory Dump Credential Extraction](ATTACK-SURFACE-VAULT.md) -- Go's GC does NOT zero freed memory; core dumps expose credentials*
  - *Ref: [ATTACK-SURFACE-VAULT.md, Attack 4: Swap File Credential Leakage](ATTACK-SURFACE-VAULT.md) -- mlock prevents swap, PR_SET_DUMPABLE prevents core dumps*
  - Verify: `/proc/<pid>/status` shows `Dumpable: 0` on Linux

- [ ] `RLIMIT_MEMLOCK` set high enough for expected credential count (SecureBytes requires mlock'd pages)
  - *Ref: [ATTACK-SURFACE-VAULT.md, Attack 4](ATTACK-SURFACE-VAULT.md) -- LOOM must refuse to start if RLIMIT_MEMLOCK is too low*

- [ ] Cryptographic algorithms are compile-time constants (AES-256-GCM, BLAKE3, Argon2id); no runtime configuration
  - *Ref: [ATTACK-SURFACE-VAULT.md, Attack 9: Encryption Downgrade](ATTACK-SURFACE-VAULT.md) -- config-driven algorithm selection allows downgrade to AES-128, SHA-1, PBKDF2, or NULL encryption*

---

## 3. API Security

- [ ] **(CRITICAL)** JWT validation with strict algorithm allowlist: Ed25519 (`EdDSA`) and ES256 only; `alg: none` and `HS256` explicitly rejected
  - *Ref: [ATTACK-SURFACE-API.md, Section 1: JWT Token Attacks](ATTACK-SURFACE-API.md) -- `alg: none` bypasses all authentication; HS256-with-RSA-public-key allows signature forgery*
  - Verify: JWT middleware calls `jwt.WithValidMethods([]string{"EdDSA", "ES256"})`

- [ ] **(CRITICAL)** Tenant ID extracted exclusively from JWT claims, never from headers, query params, or request body
  - *Ref: [ATTACK-SURFACE-API.md, Section 2: IDOR](ATTACK-SURFACE-API.md) -- tenant_id from X-Tenant-ID header allows trivial cross-tenant access*
  - *Ref: [ATTACK-SURFACE-API.md, Section 5: Request Smuggling](ATTACK-SURFACE-API.md) -- X-Tenant-ID injection*

- [ ] CORS allowlist configured with specific origins (no `Access-Control-Allow-Origin: *`)
  - *Ref: [ATTACK-SURFACE-API.md, Section 6: CORS Misconfiguration](ATTACK-SURFACE-API.md) -- wildcard CORS allows malicious websites to steal data via logged-in user's session*
  - Verify: CORS middleware has an explicit origin allowlist; unknown origins receive no CORS headers

- [ ] Rate limiting enabled per-IP and per-tenant
  - *Ref: [ATTACK-SURFACE-API.md, Section 4: Rate Limiting and Brute Force](ATTACK-SURFACE-API.md) -- credential stuffing and API key enumeration*
  - Verify: Rate limiter is configured with per-IP token bucket and per-account lockout (5 failures = 15-minute lockout)

- [ ] Content Security Policy headers set: `script-src 'self'` (no `unsafe-inline`, no `unsafe-eval`)
  - *Ref: [ATTACK-SURFACE-UI-SUPPLY-CHAIN.md, Section 11: Content Security Policy](ATTACK-SURFACE-UI-SUPPLY-CHAIN.md) -- CSP mitigates XSS, clickjacking, and data exfiltration*
  - Verify: Response headers include full CSP with `script-src 'self'`, `frame-ancestors 'none'`, `object-src 'none'`

- [ ] HSTS enabled with `max-age=31536000; includeSubDomains; preload`
  - *Ref: [ATTACK-SURFACE-UI-SUPPLY-CHAIN.md, Section 11: Additional Security Headers](ATTACK-SURFACE-UI-SUPPLY-CHAIN.md) -- all infrastructure management traffic must be encrypted*
  - Verify: `Strict-Transport-Security` header present on all responses

- [ ] Webhook URLs restricted to HTTPS + domain allowlist; no IP-based URLs; DNS resolution checked against private ranges
  - *Ref: [ATTACK-SURFACE-API.md, Section 8: Webhook / Callback SSRF](ATTACK-SURFACE-API.md) -- SSRF via webhook URLs can leak cloud metadata, access internal services, read local files*
  - Verify: Webhook validator rejects `http://`, `file://`, all private IP ranges, and cloud metadata endpoints

- [ ] `X-Frame-Options: DENY` and `X-Content-Type-Options: nosniff` headers set
  - *Ref: [ATTACK-SURFACE-UI-SUPPLY-CHAIN.md, Section 3: Clickjacking](ATTACK-SURFACE-UI-SUPPLY-CHAIN.md)*

- [ ] Server technology headers stripped (`Server`, `X-Powered-By`, `X-Runtime`)
  - *Ref: [ATTACK-SURFACE-API.md, Section 10: Error Message Information Disclosure](ATTACK-SURFACE-API.md)*

- [ ] GraphQL introspection disabled in production; query depth limited to 5; batch size limited to 5
  - *Ref: [ATTACK-SURFACE-API.md, Section 7: GraphQL / API Introspection Abuse](ATTACK-SURFACE-API.md)*

- [ ] Pagination page size capped (max 100); cursors HMAC-signed with tenant scope
  - *Ref: [ATTACK-SURFACE-API.md, Section 9: Pagination / Export Data Leakage](ATTACK-SURFACE-API.md)*

- [ ] WebSocket upgrade handler validates Origin header against allowlist
  - *Ref: [ATTACK-SURFACE-UI-SUPPLY-CHAIN.md, Section 4: WebSocket Hijacking](ATTACK-SURFACE-UI-SUPPLY-CHAIN.md) -- WebSocket connections are NOT subject to same-origin policy*

- [ ] Session tokens stored in `httpOnly; Secure; SameSite=Strict` cookies, not localStorage
  - *Ref: [ATTACK-SURFACE-UI-SUPPLY-CHAIN.md, Section 5: Local Storage Token Theft](ATTACK-SURFACE-UI-SUPPLY-CHAIN.md) -- any XSS escalates to full account takeover if tokens are in localStorage*

---

## 4. LLM Security

- [ ] **(CRITICAL)** LLM output validation pipeline active: schema validation -> guardrail checks -> Batfish simulation -> diff review
  - *Ref: [ATTACK-SURFACE-LLM.md, Section 1: Prompt Injection](ATTACK-SURFACE-LLM.md) -- prompt injection cannot be fully prevented at the application layer; defense-in-depth validation pipeline is mandatory*
  - *Ref: [ATTACK-SURFACE-LLM.md, Implementation Requirements](ATTACK-SURFACE-LLM.md) -- LLM output is untrusted input*
  - Verify: Every LLM-generated configuration passes through typed struct parsing, not freeform text execution

- [ ] **(CRITICAL)** Tool call sequence analysis enabled (privilege escalation chain detection)
  - *Ref: [ATTACK-SURFACE-LLM.md, Section 3: Tool Use Abuse](ATTACK-SURFACE-LLM.md) -- even individually valid tool calls can form attack patterns: `get_config -> update_credentials -> execute_command`*
  - Verify: `ToolValidator` is active with tenant scope enforcement, parameter validation, operation ordering checks, and danger level classification

- [ ] Confidence calibration thresholds set (LLM self-assessment weighted at 0.20 or less)
  - *Ref: [ATTACK-SURFACE-LLM.md, Section 7: Confidence Score Manipulation](ATTACK-SURFACE-LLM.md) -- LLM-claimed confidence of 0.98 on novel configs was calibrated down to 0.00; never trust LLM self-assessed confidence alone*
  - Verify: `ConfidenceCalibrator` uses multi-signal scoring: LLM (0.20), RAG (0.30), template (0.25), historical (0.25)
  - Verify: Novelty penalty (-0.30) applied when historical success is 0
  - Verify: Manipulation detection alerts when LLM-claimed vs. calibrated gap exceeds 0.30

- [ ] **(CRITICAL)** Credential exfiltration scanner active (11 regex patterns)
  - *Ref: [ATTACK-SURFACE-LLM.md, Section 2: Credential Exfiltration via LLM Output](ATTACK-SURFACE-LLM.md) -- LLM can embed Cisco passwords, API keys, private keys, SNMP communities, JWT tokens, and database URLs in generated configurations*
  - Verify: `OutputSanitizer` scans ALL LLM output (reasoning, config, JSON fields) before storage, display, or execution
  - Verify: Scanner covers all 11 patterns: Cisco passwords, SNMP communities, Bearer/API tokens, AWS keys, database URLs, PEM private keys, JWT tokens, generic password fields, enable secrets, SSH keys, connection strings

- [ ] Model routing validated (no unauthorized model substitution)
  - *Ref: [ATTACK-SURFACE-LLM.md, Section 5: Model Substitution Attack](ATTACK-SURFACE-LLM.md) -- compromised local model can inject backdoor ACLs, DNS exfiltration, and security downgrades*
  - Verify: `ModelRegistry` restricts `config_suggestion` to verified providers only; local models limited to classification
  - Verify: Response HMAC-SHA256 signatures verified against registered provider signing keys

- [ ] Circuit breaker active for LLM tool calls (max 25 calls/request, max 3 duplicate calls, 5-minute timeout)
  - *Ref: [ATTACK-SURFACE-LLM.md, Section 4: Denial of Service via LLM Looping](ATTACK-SURFACE-LLM.md) -- infinite refresh loops, config oscillation, and cascading failures exhaust API credits*

- [ ] RAG corpus integrity checking enabled (source trust, SHA-256 checksums, dangerous pattern scanning)
  - *Ref: [ATTACK-SURFACE-LLM.md, Section 6: RAG Poisoning](ATTACK-SURFACE-LLM.md) -- poisoned documents can inject backdoor ACLs, DNS exfiltration, and weakened crypto into LLM recommendations*

- [ ] LLM input sanitization pipeline active for all device-sourced strings
  - *Ref: [ATTACK-SURFACE-LLM.md, Section 1](ATTACK-SURFACE-LLM.md) -- hostname injection, sysDescr HTML comments, XML tag injection, JSON role injection, newline instruction injection*

---

## 5. Network Security

- [ ] **(CRITICAL)** Management VLAN isolated for IPMI/SNMPv2c devices; only LOOM adapter IPs can reach management ports
  - *Ref: [ATTACK-SURFACE-ADAPTERS.md, Section 1: IPMI Protocol Vulnerabilities](ATTACK-SURFACE-ADAPTERS.md) -- IPMI RAKP hash leak (CVE-2013-4786) is a protocol-level flaw that cannot be patched; network isolation is the only defense*
  - *Ref: [ATTACK-SURFACE-ADAPTERS.md, Section 2: SNMP Community String Exposure](ATTACK-SURFACE-ADAPTERS.md) -- SNMPv2c sends community strings in plaintext UDP*
  - Verify: Firewall rules restrict UDP/623 (IPMI) and UDP/161 (SNMP) to LOOM adapter source IPs only

- [ ] **(CRITICAL)** All adapter connections use TLS where the protocol supports it
  - *Ref: [ATTACK-SURFACE-ADAPTERS.md, Cross-Cutting Security Recommendations](ATTACK-SURFACE-ADAPTERS.md) -- Redfish (HTTPS), gNMI (mTLS), WS-Management (HTTPS), NETCONF (over SSH)*
  - Verify: No `InsecureSkipVerify: true` in any TLS configuration
  - Verify: Minimum TLS 1.2, prefer TLS 1.3

- [ ] **(CRITICAL)** SSH host key database populated; `InsecureIgnoreHostKey` is never used
  - *Ref: [ATTACK-SURFACE-ADAPTERS.md, Section 3: SSH Man-in-the-Middle](ATTACK-SURFACE-ADAPTERS.md) -- InsecureIgnoreHostKey makes SSH MITM trivial via ARP/DNS spoofing*
  - Verify: TOFU mode is configured (`TOFUReject` for high-security, `TOFUAcceptAndFlag` for standard)
  - Verify: SSH client restricts key exchange to `curve25519-sha256`, `ecdh-sha2-nistp256`; bans `ssh-rsa` and `ssh-dss`

- [ ] gNMI mutual TLS (mTLS) configured for all gNMI-capable devices
  - *Ref: [ATTACK-SURFACE-ADAPTERS.md, Section 6: gNMI/gRPC Without TLS](ATTACK-SURFACE-ADAPTERS.md) -- without mTLS, any client can delete all ACLs, add rogue static routes, and enable port mirroring*
  - Verify: No `insecure.NewCredentials()` in gNMI client configuration

- [ ] IPMI cipher suite 0 banned; minimum cipher suite 3 (AES) enforced
  - *Ref: [ATTACK-SURFACE-ADAPTERS.md, Section 1b: Cipher Suite Zero](ATTACK-SURFACE-ADAPTERS.md) -- cipher suite 0 provides no authentication, integrity, or confidentiality*

- [ ] Default credentials audit enabled for BMC devices; devices with defaults blocked from management
  - *Ref: [ATTACK-SURFACE-ADAPTERS.md, Section 1c: Default Credentials](ATTACK-SURFACE-ADAPTERS.md) -- well-known defaults (ADMIN/ADMIN, root/calvin, etc.) are trivially exploitable*

- [ ] Protocol-specific timeouts configured for all adapter connections
  - *Ref: [ATTACK-SURFACE-ADAPTERS.md, Section 10: Adapter Timeout / Slowloris](ATTACK-SURFACE-ADAPTERS.md) -- compromised devices holding connections open exhaust goroutine pool; 5-10% compromised devices can take down entire adapter fleet*
  - Verify: Every protocol has connect, read, write, idle, and total operation timeouts configured

- [ ] Per-device circuit breakers active (3 consecutive timeouts or 5 failures opens circuit)
  - *Ref: [ATTACK-SURFACE-ADAPTERS.md, Section 10](ATTACK-SURFACE-ADAPTERS.md)*

- [ ] Connection pool limits set (max 500 total, max 3 per device, 15-minute TTL)
  - *Ref: [ATTACK-SURFACE-ADAPTERS.md, Section 10](ATTACK-SURFACE-ADAPTERS.md)*

---

## 6. Monitoring and Alerting

- [ ] **(CRITICAL)** Credential access rate limiting alerts configured
  - *Ref: [ATTACK-SURFACE-VAULT.md, Attack 1: Envelope Encryption Key Recovery](ATTACK-SURFACE-VAULT.md) -- brute-force attempts against credential store must be detected*
  - *Ref: [VAULT-ARCHITECTURE.md, Invariants](../VAULT-ARCHITECTURE.md) -- every credential retrieval produces an immutable audit record*
  - Verify: Alerts fire when credential access rate exceeds baseline by 3x within a 5-minute window

- [ ] **(CRITICAL)** Cross-tenant access attempt alerts active
  - *Ref: [ATTACK-SURFACE-POSTGRESQL.md, Attack 2: Cross-Tenant Data Access](ATTACK-SURFACE-POSTGRESQL.md) -- RLS violation attempts must generate alerts*
  - *Ref: [ATTACK-SURFACE-TEMPORAL.md, Attack 4: Namespace Escape](ATTACK-SURFACE-TEMPORAL.md) -- cross-namespace access attempts*
  - *Ref: [ATTACK-SURFACE-NATS.md, Attack 1: Cross-Tenant Event Snooping](ATTACK-SURFACE-NATS.md) -- cross-account subscription attempts*
  - Verify: Alerts configured for RLS policy violations, Temporal authorization denials, and NATS permissions violations

- [ ] **(CRITICAL)** Failed authentication alerts active (API, database, NATS, Temporal)
  - *Ref: [ATTACK-SURFACE-API.md, Section 4: Rate Limiting and Brute Force](ATTACK-SURFACE-API.md) -- credential stuffing detection*
  - Verify: Alerts fire after 10 failed authentication attempts from a single source within 1 minute

- [ ] Workflow bombing detection active (rate of workflow creation, task queue depth monitoring)
  - *Ref: [ATTACK-SURFACE-TEMPORAL.md, Attack 5: Workflow Bombing](ATTACK-SURFACE-TEMPORAL.md) -- 10,000+ workflows in rapid succession exhaust persistence storage*
  - Verify: `QueueDepthMonitor` checks task queue depths every 30 seconds; alerts on zero pollers

- [ ] Vault backend change attempt alerts configured (CRITICAL severity)
  - *Ref: [ATTACK-SURFACE-VAULT.md, Attack 10: Vault Backend Substitution](ATTACK-SURFACE-VAULT.md) -- `BACKEND_CHANGE_REJECTED` events indicate active attack*

- [ ] SSH host key change alerts configured
  - *Ref: [ATTACK-SURFACE-ADAPTERS.md, Section 3: SSH Man-in-the-Middle](ATTACK-SURFACE-ADAPTERS.md) -- host key changes may indicate MITM attack*

- [ ] LLM confidence manipulation alerts configured (gap > 0.30 between claimed and calibrated)
  - *Ref: [ATTACK-SURFACE-LLM.md, Section 7: Confidence Score Manipulation](ATTACK-SURFACE-LLM.md)*

- [ ] Adapter circuit breaker state changes logged and alerted
  - *Ref: [ATTACK-SURFACE-ADAPTERS.md, Section 10](ATTACK-SURFACE-ADAPTERS.md) -- multiple circuit breakers opening simultaneously may indicate coordinated attack*

---

## 7. Supply Chain Security

- [ ] Go module integrity verified (`go mod verify` passes in CI)
  - *Ref: [ATTACK-SURFACE-UI-SUPPLY-CHAIN.md, Section 7: Dependency Confusion](ATTACK-SURFACE-UI-SUPPLY-CHAIN.md)*
  - Verify: `GONOSUMDB` and `GONOSUMCHECK` are NOT set; `GOPRIVATE` is configured for internal modules

- [ ] npm lock file (`package-lock.json`) committed and `npm ci` used in CI (not `npm install`)
  - *Ref: [ATTACK-SURFACE-UI-SUPPLY-CHAIN.md, Section 8: npm Supply Chain](ATTACK-SURFACE-UI-SUPPLY-CHAIN.md)*

- [ ] Container images built from distroless/scratch base images with pinned digests (not tags)
  - *Ref: [ATTACK-SURFACE-UI-SUPPLY-CHAIN.md, Section 10: Container Image Attacks](ATTACK-SURFACE-UI-SUPPLY-CHAIN.md) -- mutable tags can be silently replaced*

- [ ] Container images signed with cosign/sigstore; admission controller enforces signatures
  - *Ref: [ATTACK-SURFACE-UI-SUPPLY-CHAIN.md, Section 10](ATTACK-SURFACE-UI-SUPPLY-CHAIN.md)*

- [ ] Binary builds are reproducible (`-trimpath -ldflags='-s -w -buildid='` with `CGO_ENABLED=0`)
  - *Ref: [ATTACK-SURFACE-UI-SUPPLY-CHAIN.md, Section 9: Binary Tampering](ATTACK-SURFACE-UI-SUPPLY-CHAIN.md)*

- [ ] Vulnerability scanning (`trivy`, `npm audit`) runs in CI with CRITICAL severity failing the build
  - *Ref: [ATTACK-SURFACE-UI-SUPPLY-CHAIN.md, Sections 8 and 10](ATTACK-SURFACE-UI-SUPPLY-CHAIN.md)*

---

## Sign-Off

| Role | Name | Date | Signature |
|------|------|------|-----------|
| Security Lead | | | |
| Infrastructure Lead | | | |
| Platform Engineering Lead | | | |
| CISO / Delegate | | | |
