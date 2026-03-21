# LOOM Gap Analysis — Consolidated Expert Review

> **Date:** 2026-03-21
> **Reviewers:** Systems Architect, Security Architect, Infrastructure Engineer, Application Architect
> **Scope:** All Phase 0 contracts, ADRs, research documents, implementation plan
> **Method:** Independent review by four domain experts, findings consolidated and cross-referenced

---

## Executive Summary

Four independent expert reviews identified **89 gaps** across the LOOM Phase 0 documentation. After deduplication and cross-referencing, these consolidate into **10 critical**, **23 high**, **24 medium**, and **8 low** severity findings organized into six research areas.

The Phase 0 contracts are exceptionally thorough — among the best pre-code documentation any reviewer had seen. The gaps are not signs of carelessness but of a project whose ambition (25+ protocol adapters, 4 deployment modes, multi-tenant, LLM-driven, air-gapped capable) creates a surface area that no single pass can fully cover.

**Three systemic themes emerge:**

1. **The research-to-contract gap** — LOOM's competitive research honestly documents vendor fragmentation and protocol limitations, but the adapter contracts and architecture sometimes smooth over those problems with cleaner abstractions than reality permits.

2. **Core vs. edge security asymmetry** — The security perimeter is drawn tightly around the control plane (vault, mTLS, RLS, RBAC) but leaves significant gaps at the edges: edge agent credentials, auto-update trust, adapter plugin isolation, and Valkey.

3. **Over-specified runtime, under-specified developer experience** — The runtime architecture is detailed down to struct fields, but there is no adapter SDK, no scaffolding tool, no testing harness, and no contributor guide. A project that needs 25+ adapters cannot afford to make adapter development difficult.

---

## Critical Findings (Resolve Before Phase 1A)

| # | Finding | Domains | Research Doc |
|---|---------|---------|--------------|
| C1 | **Edge-Hub conflict resolution undefined** — No reconciliation protocol for post-partition workflow state vs. edge observed state. FIFO queue eviction drops critical events. | Arch, Infra | [RESEARCH-edge-agent-security.md](RESEARCH-edge-agent-security.md) |
| C2 | **Wire protocol versioning absent** — No versioning for adapter gRPC, NATS events, Temporal activity types, or edge sync protocol. Temporal replay breaks on struct changes. | Arch, App | [RESEARCH-wire-protocol-lifecycle.md](RESEARCH-wire-protocol-lifecycle.md) |
| C3 | **Multi-tenant shared device model missing** — `Device` has single `TenantID`. Every shared switch breaks the model. RC-04 proposes options but doesn't commit. | Arch, Infra, App | [RESEARCH-domain-model-gaps.md](RESEARCH-domain-model-gaps.md) |
| C4 | **Edge agent credential cache unencrypted** — No local vault, no KEK, no TPM binding. Physical access to edge site exposes all device credentials. | Security | [RESEARCH-edge-agent-security.md](RESEARCH-edge-agent-security.md) |
| C5 | **Auto-update as privilege escalation vector** — Single signing key, no multi-sig, no staged rollout, no rollback. Compromised key = code execution on entire fleet. | Security | [RESEARCH-edge-agent-security.md](RESEARCH-edge-agent-security.md) |
| C6 | **No IPAM integration or strategy** — Decomposition examples assign subnets but no IP allocation model, no DHCP integration, no collision prevention. | Infra | [RESEARCH-domain-model-gaps.md](RESEARCH-domain-model-gaps.md) |
| C7 | **NETCONF vendor model fragmentation severely underestimated** — Single adapter claimed for IOS-XE, IOS-XR, Junos, Nokia. YANG models are radically different. Confirmed-commit support varies. | Infra | [RESEARCH-protocol-adapter-realism.md](RESEARCH-protocol-adapter-realism.md) |
| C8 | **No physical topology model** — No rack, PDU, cooling zone, or cable types. Cannot assess power failure domains or make meaningful placement decisions. | Infra | [RESEARCH-domain-model-gaps.md](RESEARCH-domain-model-gaps.md) |
| C9 | **Firmware lifecycle completely absent** — `FirmwareVersion` recorded but no update operations, compatibility matrices, staging, or rollback defined. | Infra | [RESEARCH-domain-model-gaps.md](RESEARCH-domain-model-gaps.md) |
| C10 | **No adapter SDK or developer experience** — No scaffolding tool, no example walkthrough, no conformance test suite, no contributor guide. Adapter count = LOOM's value. | App | [RESEARCH-protocol-adapter-realism.md](RESEARCH-protocol-adapter-realism.md) |

---

## High Findings (Resolve Before Phase 2)

| # | Finding | Domains | Research Doc |
|---|---------|---------|--------------|
| H1 | Temporal-PostgreSQL shared fate (single failure domain) | Arch | [RESEARCH-wire-protocol-lifecycle.md](RESEARCH-wire-protocol-lifecycle.md) |
| H2 | Identity matcher has no concurrency control (RC-06 unsolved) | Arch | [RESEARCH-domain-model-gaps.md](RESEARCH-domain-model-gaps.md) |
| H3 | ObservedFact table grows without bound (no retention policy) | Arch | [RESEARCH-wire-protocol-lifecycle.md](RESEARCH-wire-protocol-lifecycle.md) |
| H4 | DesiredState `map[string]string` vs heterogeneous ObservedFacts — comparison undefined | Arch | [RESEARCH-domain-model-gaps.md](RESEARCH-domain-model-gaps.md) |
| H5 | `continue-as-new` cascading complexity (signals, projections, audit) | Arch | [RESEARCH-wire-protocol-lifecycle.md](RESEARCH-wire-protocol-lifecycle.md) |
| H6 | Idempotency cache location and durability unspecified | Arch | [RESEARCH-wire-protocol-lifecycle.md](RESEARCH-wire-protocol-lifecycle.md) |
| H7 | DesiredState conflict resolution for overlapping declarations | Arch | [RESEARCH-domain-model-gaps.md](RESEARCH-domain-model-gaps.md) |
| H8 | No adapter rate limiting — device management plane overload | Arch, Infra | [RESEARCH-protocol-adapter-realism.md](RESEARCH-protocol-adapter-realism.md) |
| H9 | Internal CA lifecycle (no OCSP, no CRL distribution, no CA rotation) | Security | [RESEARCH-security-hardening.md](RESEARCH-security-hardening.md) |
| H10 | Temporal DataConverter encryption key management unspecified | Security | [RESEARCH-security-hardening.md](RESEARCH-security-hardening.md) |
| H11 | LLM context window leaks across tenants | Security, App | [RESEARCH-security-hardening.md](RESEARCH-security-hardening.md) |
| H12 | Break-glass token TTL inconsistency (15min vs 1hr), jti cache durability | Security | [RESEARCH-security-hardening.md](RESEARCH-security-hardening.md) |
| H13 | Audit trail BLAKE3 chain referenced but not in schema, no external anchoring | Security | [RESEARCH-security-hardening.md](RESEARCH-security-hardening.md) |
| H14 | ConfigSnapshot stored as unencrypted `[]byte` (contains device secrets) | Security | [RESEARCH-security-hardening.md](RESEARCH-security-hardening.md) |
| H15 | RawConfigPush bypasses all validation — privileged escape hatch | Security | [RESEARCH-protocol-adapter-realism.md](RESEARCH-protocol-adapter-realism.md) |
| H16 | Adapter plugin trust boundary undefined (go-plugin gRPC) | Security, App | [RESEARCH-protocol-adapter-realism.md](RESEARCH-protocol-adapter-realism.md) |
| H17 | Valkey security posture completely absent from security model | Security | [RESEARCH-security-hardening.md](RESEARCH-security-hardening.md) |
| H18 | OIDC tenant claim verification — no user-to-tenant membership check | Security | [RESEARCH-security-hardening.md](RESEARCH-security-hardening.md) |
| H19 | Redfish OEM extension divergence (Dell/HPE/Supermicro) | Infra | [RESEARCH-protocol-adapter-realism.md](RESEARCH-protocol-adapter-realism.md) |
| H20 | Storage orchestration model skeletal (no volume/LUN/pool types) | Infra | [RESEARCH-domain-model-gaps.md](RESEARCH-domain-model-gaps.md) |
| H21 | 100-device MVP insufficient for architectural validation | Infra, App | [RESEARCH-protocol-adapter-realism.md](RESEARCH-protocol-adapter-realism.md) |
| H22 | 12-phase plan schedule risk extreme (16-22 months, 2-3 people) | Arch | [RESEARCH-wire-protocol-lifecycle.md](RESEARCH-wire-protocol-lifecycle.md) |
| H23 | Temporal `GetVersion` / workflow versioning strategy missing | App | [RESEARCH-wire-protocol-lifecycle.md](RESEARCH-wire-protocol-lifecycle.md) |

---

## Medium Findings

| # | Finding | Domains |
|---|---------|---------|
| M1 | `TypedOperation.Params` uses Go `any` — contradicts "zero map[string]any" promise | Arch |
| M2 | Apache AGE experimental — may not survive PostgreSQL upgrades | Arch, Infra |
| M3 | NATS event ordering weaker than architecture requires (cross-subject) | Arch |
| M4 | AES-NI mandate excludes ARM edge devices (Raspberry Pi) | Arch, Security |
| M5 | LLM response caching and replay determinism unspecified | Arch |
| M6 | DuckDB on edge is CGO — contradicts pure-Go single binary | Arch, Infra |
| M7 | NATS account provisioning not automated (static config) | Security |
| M8 | AES-GCM nonce entropy quality at boot | Security |
| M9 | Device identity spoofing for cross-tenant merge | Security |
| M10 | Connection pool credential invalidation on rotation | Security |
| M11 | No GDPR data classification or right-to-erasure handling | Security |
| M12 | Air-gapped LLM model integrity unverified | Security |
| M13 | Temporal history as data exfiltration channel | Security |
| M14 | No seccomp/AppArmor profile for vault process | Security |
| M15 | gNMI adapter overpromises transactional compensation | Infra |
| M16 | BIOS/UEFI provisioning variation not addressed | Infra |
| M17 | Certificate lifecycle management not integrated | Infra |
| M18 | Network segmentation / firewall orchestration architecture missing | Infra |
| M19 | Temporal embedded mode → external migration path undefined | Infra |
| M20 | DNS integration is verification-only (no record creation) | Infra |
| M21 | VLAN management vendor differences at scale | Infra |
| M22 | Observability self-debugging workflows missing | App |
| M23 | CQRS projection rebuild from events not designed | App |
| M24 | UI real-time updates and topology visualization at scale | App |

---

## Low Findings

| # | Finding | Domains |
|---|---------|---------|
| L1 | MikroTik REST API limitations acknowledged but understated | Infra |
| L2 | Bare metal cost model undefined | Infra |
| L3 | Out-of-band management network isolation unspecified | Infra |
| L4 | Feature flag infrastructure absent | App |
| L5 | Cost data bootstrapping for first-time deployment | App |
| L6 | NATS subject hierarchy design not formalized | App |
| L7 | PostgreSQL extension co-habitation risk | App |
| L8 | WebSocket contract for real-time UI not specified | App |

---

## Research Documents

The following research documents provide detailed analysis and recommendations for each gap cluster:

| Document | Covers | Critical Gaps |
|----------|--------|---------------|
| [RESEARCH-edge-agent-security.md](RESEARCH-edge-agent-security.md) | Edge credential caching, auto-update trust, split-brain conflict resolution, offline autonomy boundaries | C1, C4, C5 |
| [RESEARCH-protocol-adapter-realism.md](RESEARCH-protocol-adapter-realism.md) | NETCONF/gNMI vendor fragmentation, Redfish OEM divergence, adapter SDK/DX, rate limiting, plugin trust, MVP validation scale | C7, C10, H8, H15, H16, H19, H21 |
| [RESEARCH-domain-model-gaps.md](RESEARCH-domain-model-gaps.md) | IPAM, physical topology, firmware lifecycle, storage model, shared devices, DesiredState semantics, identity concurrency | C3, C6, C8, C9, H2, H4, H7, H20 |
| [RESEARCH-wire-protocol-lifecycle.md](RESEARCH-wire-protocol-lifecycle.md) | API/adapter/event versioning, ObservedFact retention, Temporal patterns, idempotency, schedule risk, backward compatibility | C2, H1, H3, H5, H6, H22, H23 |
| [RESEARCH-security-hardening.md](RESEARCH-security-hardening.md) | Vault isolation, Valkey security, audit integrity, ConfigSnapshot encryption, CA lifecycle, GDPR, break-glass, LLM tenant isolation | H9-H14, H17, H18 |

---

## Recommended Action Plan

### Before Phase 1A (Immediate)
1. Resolve C1-C5 (edge agent security model) — these affect foundational trust boundaries
2. Resolve C2 (wire protocol versioning) — affects every inter-component interface
3. Resolve C3 (shared device model) — affects the core domain model schema

### Before Phase 1B (Schema Migration)
4. Resolve C6-C9 (domain model gaps) — IPAM, topology, firmware, storage types needed in schema
5. Resolve H1-H7 (architectural high gaps) — before Temporal integration locks patterns

### Before Phase 2 (First Adapters)
6. Resolve C7, C10 (adapter realism + SDK) — before writing SSH/Redfish/SNMP adapters
7. Resolve H8, H15, H16 (adapter operational gaps) — rate limiting, RawConfigPush, plugin trust

### Before Phase 5 (Production Target)
8. Resolve all High security gaps (H9-H18) — before any production deployment
9. Address Medium gaps as encountered during implementation

---

## Cross-Reference Matrix

Gaps that were independently identified by multiple reviewers (higher confidence):

| Gap | Architect | Security | Infra | App |
|-----|-----------|----------|-------|-----|
| Shared device model | GAP-11 | — | Gap 13 | Gap 10 |
| Edge split-brain | GAP-03 | — | Gap 6 | — |
| DuckDB CGO contradiction | GAP-17 | — | Gap 7 | — |
| Wire protocol versioning | GAP-07 | — | — | Gap 8 |
| Apache AGE risk | GAP-08 | — | Gap 18 | Gap 21 |
| Adapter rate limiting | GAP-18 | — | — | Gap 18 |
| AES-NI vs ARM | GAP-14 | Gap 12 | — | — |
| Adapter plugin security | — | Gap 11 | — | Gap 2 |
| LLM tenant isolation | — | Gap 4 | — | Gap 5 |

---

## Methodology

Each reviewer was given the persona of a senior domain expert and asked to independently review all Phase 0 documents. Reviews were conducted in parallel with no cross-communication. Findings were then consolidated, deduplicated, and cross-referenced. Severity ratings reflect the consensus across reviewers — gaps identified by multiple reviewers independently were elevated in priority.

**Individual review details:**
- **Systems Architect** — 18 gaps (3 critical, 8 high, 5 medium, 2 low)
- **Security Architect** — 23 gaps (2 critical, 11 high, 10 medium)
- **Infrastructure Engineer** — 21 gaps (4 critical, 6 high, 7 medium, 4 low)
- **Application Architect** — 27 gaps (1 critical, 8 high, 12 medium, 6 low)
- **Application Architect detail:** [APPLICATION-LAYER-GAP-ANALYSIS.md](APPLICATION-LAYER-GAP-ANALYSIS.md)
