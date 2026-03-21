# ADR-011: Enterprise-Grade Credential Vault with Hardware-Accelerated Encryption

## Status

Accepted

## Context

LOOM manages credentials for every piece of managed infrastructure — IPMI passwords, SSH keys, SNMP community strings, Redfish tokens, cloud API keys, network device credentials. Compromise of this credential store means compromise of the entire infrastructure estate. Security must be equivalent to or exceed Apple's Data Protection architecture.

## Decision

The credential vault is a library embedded in LOOM's process with defense-in-depth across encryption, key management, memory protection, access control, and LLM isolation.

### Encryption

- **AES-256-GCM** for all symmetric encryption, hardware-accelerated via AES-NI (mandatory — LOOM refuses to start without AES-NI)
- **3-tier envelope encryption**: Master Key → Tenant KEK → Credential DEK
- **Argon2id** for any passphrase-derived keys
- **BLAKE3** for integrity verification (uses AVX-512 when available)
- **X25519** for key exchange, **Ed25519** for signatures

### Key Management Backends (pluggable, ordered by preference)

1. HSM via PKCS#11 (Thales Luna, YubiHSM, AWS CloudHSM)
2. HashiCorp Vault Transit engine
3. Cloud KMS (AWS KMS, Azure Key Vault, GCP Cloud KMS)
4. TPM 2.0 sealed storage
5. Local file-based (Argon2id-derived from passphrase) — air-gapped fallback only

### Memory Protection

- Credential bytes on **mlock'd pages** (no swap)
- **Explicit zeroing** after use (not optimizable away)
- Core dumps disabled (`PR_SET_DUMPABLE=0`)
- Custom `MarshalJSON` that refuses to serialize credential values
- Credentials never appear in logs, errors, or stack traces

### Access Control

- Every credential retrieval: authn + tenant match + RBAC + rate limit + audit entry
- **No bulk export API** — architecturally impossible
- Break-glass requires dual authorization + enhanced audit

### LLM Isolation

- LLM receives only **opaque CredentialRef UUIDs**
- No vault method returns credentials in LLM-consumable format
- Adapter uses credential directly; LLM never sees the value

## Consequences

- Higher confidence in credential security — compromise requires defeating HSM + envelope encryption + memory protection + access control simultaneously
- Per-tenant KEK means tenant compromise is isolated; one tenant's breach does not expose another's credentials
- Hardware acceleration means encryption is effectively free (>1M ops/sec for AES-256-GCM with AES-NI)
- HSM dependency adds operational complexity for highest-security deployments
- mlock has system-wide limits (`RLIMIT_MEMLOCK`) that may need tuning on hosts with large credential sets
- AES-NI mandate excludes older hardware (pre-2010) — acceptable given enterprise hardware target
- LLM isolation eliminates an entire class of prompt-injection credential exfiltration attacks

## Alternatives Considered

1. **Storing credentials in HashiCorp Vault only** — Rejected: adds a hard external dependency; LOOM must work standalone in air-gapped environments.
2. **PostgreSQL-native encryption (pgcrypto)** — Rejected: encryption in the database means the database has access to plaintext; LOOM encrypts before storing.
3. **Software-only AES without AES-NI requirement** — Rejected: enterprise hardware mandate; software AES is 10-50x slower and vulnerable to timing attacks.
4. **Separate credential microservice** — Rejected: adds network hop and complexity; vault is a library embedded in LOOM's process for minimal attack surface.
