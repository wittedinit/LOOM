# MVP Scope

> Addresses ADVERSARIAL-REVIEW.md Section 7 (implied): LOOM's scope is enormous — "10 startups in a trench coat." This document draws a hard line between what ships in MVP and what is post-MVP evolution.

## MVP = Phases 0 through 5

MVP is complete when Phase 5 passes its go/no-go criteria. Phases 6-11 are product evolution, not launch requirements.

| Phase | Scope | What Ships |
|-------|-------|------------|
| 0 | Contracts | All contract docs, domain model, ADRs. No code. |
| 1A | Bootstrap Runtime | Binary starts, connects to PostgreSQL + NATS, health check, graceful shutdown. |
| 1B | Core Persistence | Device/tenant/audit tables, adapter registry, event publisher, mock adapter. |
| 2 | Inventory Vertical | SSH + Redfish adapters, single-host discovery, identity merge, CLI. |
| 3 | Expanded Discovery | SNMP adapter, subnet scanner, deterministic fingerprinter, optional LLM classifier. |
| 4 | Durable Workflows | Temporal integration, saga compensation, human approval, verification, IPMI adapter. |
| 5 | Operator API | REST API with JWT auth, RBAC, tenant scoping, SSE, rate limiting. |

---

## MVP Capabilities (what LOOM can do at Phase 5)

1. **Discovery:** Scan subnets, discover devices via SSH, Redfish, SNMP, and IPMI. Fingerprint vendor/model. Merge multi-protocol discoveries into a single device identity.
2. **Inventory:** Store devices, endpoints, observed facts, and external identities in PostgreSQL. Tenant-scoped, audited, event-published.
3. **Workflows:** Execute durable workflows via Temporal — discovery, provisioning, power-cycle. Saga rollback on failure (with compensation reliability per adapter — see ADAPTER-CONTRACT.md). Human approval gates via Temporal signals.
4. **Verification:** Every workflow declares an expected outcome. Verification is mandatory. Failed verification = failed workflow.
5. **API:** REST API with JWT auth, per-tenant RBAC, cursor pagination, idempotency keys, SSE for real-time updates.
6. **Adapters:** SSH, Redfish, SNMP, IPMI — 4 adapters covering server discovery, power control, and sensor reading.
7. **Single-tenant:** MVP is single-tenant. Multi-tenant isolation exists at the data layer (every table has `tenant_id`) but the operational model is one organization per deployment.
8. **Config push:** Raw config push via SSH/Redfish. No template engine, no universal config translation.
9. **Events:** NATS JetStream for internal events (device lifecycle, workflow status, audit). No external event consumers.
10. **CLI:** `loom serve`, `loom discover`, `loom devices list/show`, `loom workflow run/status/list`, `loom migrate`.

---

## Explicitly OUT of MVP

These are real capabilities that LOOM will eventually support. They are NOT in Phases 0-5.

| Feature | Phase | Why Not MVP |
|---------|-------|-------------|
| Multi-cloud cost arbitrage | 8 | Requires cloud pricing API integration, cost model definition, and billing data pipelines. Useful but not core. |
| LLM-driven placement optimization | 7+ | LLM placement requires hard constraint enforcement (PLACEMENT-POLICY.md), tested templates, and deterministic fallback. MVP uses rule-based placement only. |
| Capacity planning | 8-9 | Requires historical data, trend analysis, and cost modeling. Needs months of telemetry data to be useful. |
| Full multi-vendor network translation | 7 | Universal VLAN/ACL/route model across 6+ vendors is an XL effort. MVP uses raw config push. |
| Edge agents | 11 | Store-and-forward agents with SQLite and NATS leaf nodes. Requires conflict resolution for split-brain scenarios. |
| Topology visualization | 9 | Cytoscape.js graph rendering. Needs AGE graph data populated from discovery. |
| Drift detection | 9 | Periodic state comparison against declared state. Requires defined desired state model. |
| Compliance engine | 9 | Policy-based compliance checking (data residency, encryption). Requires policy definition framework. |
| NETCONF/gNMI/eAPI/NX-API adapters | 7 | Network protocol adapters for switch/router management. Not needed for server-focused MVP. |
| AMT/PiKVM/vSphere/Proxmox/libvirt adapters | 11 | Specialized adapters for KVM, virtualization, and out-of-band management. |
| Web UI | 6 | React + Next.js operator console. CLI is sufficient for MVP. |
| Cost dashboard | 8 | ECharts-based cost visualization. |
| HA / leader election | 10 | Single-instance is acceptable for MVP. HA adds complexity without changing core functionality. |
| mTLS between components | 10 | TLS for external API; internal communication is localhost or trusted network in MVP. |
| Helm chart | 10 | Docker Compose is sufficient for MVP deployment. |
| pgvector RAG pipeline | 11 | Vendor documentation embedding for LLM grounding. Optional even post-MVP. |

---

## MVP Success Metric

> LOOM can discover 100 devices via SSH + Redfish + SNMP + IPMI, execute a provisioning workflow with saga rollback and verification, and expose the result via a REST API with auth and tenant scoping — for one tenant, deployed via Docker Compose.

### Concrete Go/No-Go Criteria

1. **Discovery:** `loom discover --subnet 10.0.0.0/24` discovers and fingerprints all reachable devices. Multi-protocol discoveries for the same host are merged, not duplicated.
2. **Workflow execution:** A provisioning workflow (plan -> approve -> execute -> verify) completes successfully via Temporal. Steps that fail trigger saga compensation in LIFO order.
3. **Verification:** A workflow with a deliberately failed verification step transitions to `failed` state, not `succeeded`.
4. **Compensation:** A workflow with a deliberately failed step at position N triggers compensation for steps N-1 through 1. Compensation reliability is reported per adapter (see ADAPTER-CONTRACT.md). Adapters with `CompReliabilityNone` require human acknowledgment before execution.
5. **API:** All device and workflow operations are accessible via REST API with JWT auth. Tenant filtering is enforced. Unauthenticated requests return 401. Wrong-tenant requests return 403.
6. **Audit:** Every mutation produces an audit event with full envelope (actor, correlation_id, causation_id, before/after state).
7. **Events:** Device lifecycle and workflow status events are published to NATS JetStream.
8. **Scale:** The system handles 100 devices without performance degradation. (10K devices is a post-MVP target.)

---

## Post-MVP Roadmap

Phases 6-11 are product evolution. They are ordered by value, not by technical dependency (except where noted).

| Phase | Priority | Rationale |
|-------|----------|-----------|
| 6 (Thin UI) | High | Operators need a visual console. CLI-only is a barrier to adoption. |
| 7 (Network Domain) | High | Multi-vendor switch/router management is the core differentiator. |
| 8 (Cost Engine) | Medium | Cost tracking enables budget enforcement and chargeback. |
| 9 (Verification + Drift) | Medium | Drift detection and compliance are enterprise requirements. |
| 10 (Hardening) | High (before production customers) | HA, TLS, secrets management, observability are pre-production. |
| 11 (Edge + Remaining Adapters) | Medium | Edge agents and full adapter coverage expand the addressable market. |

Each post-MVP phase has its own acceptance criteria defined in IMPLEMENTATION-PLAN.md. No phase is started until the previous phase passes go/no-go.

---

## What This Means for the Docs

Several documents describe capabilities that are post-MVP:

- **ARCHITECTURE.md** describes LLM-driven placement, cost optimization, and topology visualization. These are the vision, not the MVP.
- **COMPETITIVE-ANALYSIS.md** claims "single pane of glass." MVP delivers an operator API + CLI, not a dashboard.
- **DEPLOYMENT-TOPOLOGY.md** describes Mode 2 (production HA) and Mode 3 (edge). MVP is Mode 1 (single-instance Docker Compose).

These documents are accurate descriptions of the full product vision. They are not commitments for MVP. The distinction is captured here in MVP-SCOPE.md — the authoritative source for what ships first.
