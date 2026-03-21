# Security Checklist

> Pre-deployment security verification checklist for LOOM. All items must be checked before production deployment. Items sourced from adversarial security audit findings.

---

## Cryptography & Credentials

- [ ] HSM PIN rotated within 90 days
- [ ] Edge agents use TPM for credential caching
- [ ] AES-NI hardware acceleration verified on all nodes
- [ ] Master encryption key (MEK) stored in HSM (not file-based) for production

## Observability & Monitoring

- [ ] Security-specific Prometheus metrics enabled (`loom_auth_failures_total`, `loom_authz_denials_total`, `loom_credential_access_total`, `loom_tenant_boundary_violations_total`, `loom_adapter_anomaly_score`, `loom_break_glass_used_total`, `loom_certificate_expiry_days`, `loom_lock_contention_total`)
- [ ] External audit anchoring configured (mandatory for production)
- [ ] `/healthz` returns no component topology (only `ok`/`degraded`)
- [ ] Component health details only exposed on authenticated `/api/v1/health`
- [ ] Log scrubbing uses allowlist model (unknown fields redacted by default)
- [ ] `traceparent` header stripped from outbound LLM API calls

## Network & TLS

- [ ] All PostgreSQL connections use `sslmode=verify-full` (no `sslmode=disable` anywhere)
- [ ] NATS system subjects (`$SYS.*`, `loom._system.*`) restricted to service accounts
- [ ] Kubernetes NetworkPolicies applied (deny-all default)
- [ ] Mutual TLS configured for NETCONF and gNMI device connections
- [ ] NATS inter-node and leaf connections use TLS

## Multi-Tenancy

- [ ] Per-tenant LLM token budgets configured
- [ ] Tenant boundary violation alerts active
- [ ] Per-tenant credential access profiling enabled

## Supply Chain & CI/CD

- [ ] cosign identity scoped to exact repo (no wildcard `--certificate-identity-regexp`)
- [ ] `go mod verify` passes (module integrity check in CI)
- [ ] No `curl | sh` patterns in CI (all tools pinned with SHA-256 checksums)
- [ ] SBOM generated and published with each release
- [ ] Container images signed with cosign and digest-pinned in deployment manifests

## Edge & Device Security

- [ ] Edge agents report behavioral anomaly scores
- [ ] Device-side mutual authentication enabled where firmware supports it
- [ ] IPMI/SNMP traffic restricted to dedicated management VLAN
- [ ] Offline command queue bounded by `offline_queue_max`

## Incident Response

- [ ] Security event bus (`LOOM_SECURITY` NATS stream) configured and isolated from operational backpressure
- [ ] Break-glass procedure documented and tested
- [ ] Security events forwarded to external SIEM
