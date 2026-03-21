# LOOM Implementation Plan v3: Seams Locked

## Context

LOOM is a greenfield Go project — the "orchestrator of orchestrators." After two rounds of review, the plan has been tightened from "bloated ambition" to "real execution plan." This version locks the remaining seam risks identified in review: identity, typed operations, verification semantics, audit semantics, and ADRs. These are the four things that kill control planes — not bad headlines, but ambiguity at the seams.

---

## Phase 0: Contracts — Freeze All Seams Before Code
**Scope: M | Output: 11 documents + ADR folder. No code. All seams locked.**

### Documents to Produce

**`docs/DOMAIN-MODEL.md`** — Canonical resource model with Go-style structs:
- `Target`, `Device`, `Endpoint`, `ObservedFact`, `Capability`, `CredentialRef`, `StateSnapshot`, `Relationship`, `TenantScope`
- Also includes `DesiredState` and `VerificationResult` — needed by Phase 4

**`docs/IDENTITY-MODEL.md`** — The most dangerous seam. Must define:
- `DeviceID` = LOOM internal UUID (immutable, generated at creation)
- `ExternalIdentity` set = `{type, value, source, confidence}` (serial number, MAC, hostname, BMC IP, management IP)
- Identity source priority order (e.g., serial > MAC > BMC_IP > hostname)
- Merge rules: when two discoveries match on high-confidence identity → merge into one device, attach both endpoints
- Split rules: when identity conflict detected → create separate devices, flag for human review
- Enrichment rules: additional protocol discovery adds endpoints/facts to existing device, not new device
- Mutation handling: hostname change, IP change → update external identity, keep DeviceID stable
- Tombstoning: superseded identities marked, never deleted
- Confidence scoring: each identity source has a confidence weight

**`docs/ADAPTER-CONTRACT.md`** — Operation-family model:
- `Connector`: `Connect`, `Disconnect`, `Ping` (required for all adapters)
- `Discoverer`: `Discover(scope DiscoveryScope) (*DiscoveryResult, error)` (optional)
- `Executor`: `Execute(op TypedOperation) (*OperationResult, error)` (optional)
- `StateReader`: `ReadState(resourceID string) (*StateSnapshot, error)` (optional)
- `Watcher`: `Watch(resourceID string) (<-chan StateEvent, error)` (optional, only for streaming protocols)
- Adapters implement only what they support
- Capability declaration: adapter registers `[]Capability` at registration time
- Error classification: `TransientError`, `PermanentError`, `PartialError`, `ValidationError`
- Idempotency: every operation carries `IdempotencyKey`; re-execution safe
- Compensatability: every executor declares its compensation op (or `CompensationNone`)

**`docs/OPERATION-TYPES.md`** — Typed operations (no `map[string]any`):
- Every executor operation has a typed request struct and typed response struct
- Operation types defined: `PowerCycleOp`, `SetBootDeviceOp`, `ReadInterfacesOp`, `CreateVLANOp`, `ConfigurePortOp`, `SetACLOp`, `ReadSensorsOp`, `ExecuteCommandOp`
- Each type specifies: `OperationType`, `TargetResourceRef`, `Parameters` (typed struct), `IdempotencyKey`, `RequestedBy`, `TenantID`, `Timeout`, `DryRunSupported`, `CompensationRef`, `ValidationResult`
- Validation occurs before execution dispatch — always

**`docs/WORKFLOW-CONTRACT.md`** — Workflow execution rules:
- Temporal is execution source of truth; DB is searchable projection/reporting layer
- One workflow step = one meaningful business action
- One activity = one retryable side effect
- Verification = its own activity (mandatory, not optional)
- Compensation = its own activity where needed
- Every workflow declares expected outcome (used by verification)
- Approval gates are Temporal signals, not polling
- Workflow status projection to DB is async (not blocking)

**`docs/VERIFICATION-MODEL.md`** — Verification semantics:
- Each workflow declares `ExpectedOutcome` at submission time
- Verification checks map to expected outcomes (not random health checks)
- `VerificationResult` = `{check_name, expected, observed, pass/fail, evidence, timestamp}`
- Verification results stored separately from raw observed facts
- **Failed verification = failed workflow**, even if execution returned success
- Verification is convergence proof: "desired state matches observed state"

**`docs/EVENT-MODEL.md`** — Event naming and envelope:
- Subject: `loom.{tenant}.{domain}.{action}` (e.g., `loom.prod.device.discovered`)
- Envelope: `{id, timestamp, tenant_id, actor, action, resource_type, resource_id, data, correlation_id, causation_id}`
- Actor identity always present (user JWT subject or `system:workflow:<id>`)
- Events are append-only, immutable after publish

**`docs/AUDIT-MODEL.md`** — Audit envelope:
- Actor identity: always present, never empty
- System-initiated actions use reserved actor type `system:{component}:{workflow_id}`
- Every workflow step gets `correlation_id` (links all steps in one request) + `causation_id` (links to parent step)
- Audit events are immutable — no UPDATE, no DELETE
- Audit envelope includes: `{id, timestamp, tenant_id, actor_type, actor_id, action, resource_type, resource_id, before_state, after_state, correlation_id, causation_id, outcome}`

**`docs/ERROR-MODEL.md`** — Error taxonomy:
- `TransientError` (retry with backoff) — network timeout, rate limit, temporary unavailability
- `PermanentError` (do not retry) — auth failure, resource not found, unsupported operation
- `PartialError` (some ops succeeded) — compensation needed for completed parts
- `ValidationError` (input rejected) — schema violation, policy violation, conflict
- Every error carries: `code`, `message`, `retryable`, `adapter`, `operation`, `correlation_id`

**`docs/LLM-BOUNDARIES.md`** — Hard enforceable rules:
- LLM never performs execution directly
- LLM never bypasses validation pipeline
- LLM output must be typed into a schema struct (never freeform execution)
- All LLM recommendations carry: `confidence`, `provenance[]`, `deterministic_fallback`
- Every execution path must work without the LLM (degraded but functional)
- LLM suggests; deterministic systems decide, validate, and execute
- These rules are enforced in code review, not just documented

**`docs/API-CONVENTIONS.md`** — REST API contract stability:
- Versioning: `/api/v1/` prefix, backward-compatible within major version
- Pagination: cursor-based (`?cursor=<token>&limit=50`)
- Error response: `{error: {code, message, details[], correlation_id}}`
- Correlation ID: `X-Correlation-ID` header propagated through all layers
- Idempotency: `Idempotency-Key` header for mutating requests
- Filtering: query params with consistent naming (`?tenant_id=&type=&status=`)

**`docs/adr/`** — Architecture Decision Records:
- ADR-001: Why Go as core language
- ADR-002: Why Temporal as workflow engine
- ADR-003: Why NATS JetStream as event backbone
- ADR-004: Why PostgreSQL as control-plane database
- ADR-005: Why tenant scoping at row/event/workflow level from day one
- ADR-006: Why adapter operation-family model (not one universal interface)
- ADR-007: LLM boundaries — suggests only, deterministic fallback required
- ADR-008: Temporal as execution truth, DB as projection
- ADR-009: Identity model — DeviceID + ExternalIdentity with confidence scoring
- ADR-010: Verification as mandatory post-execution activity

### Acceptance Criteria
- All 11 documents + 10 ADRs committed to repo
- Domain model has concrete Go-style struct definitions (field names, types, JSON tags)
- Identity model answers: "what happens when same server discovered via SSH and Redfish?" unambiguously
- Operation types have typed request/response structs — zero `map[string]any`
- Verification model defines: expected outcome → check mapping → convergence proof
- Audit model defines: actor identity for both human and system actions
- Another AI agent can read Phase 0 docs and implement Phase 1A without inventing local truths

---

## Phase 1A: Bootstrap Runtime
**Scope: S | Output: Binary starts, connects to infra, health check, clean shutdown**

Only:
- `go mod init github.com/wittedinit/loom`
- Cobra CLI: `loom serve`, `loom version`, `loom config validate`
- Viper config (YAML + env vars + flags)
- slog structured logging
- PostgreSQL connection (pgxpool) — connect, ping, close
- NATS connection — connect, ping, close
- `GET /healthz` — 200 when all deps healthy
- Graceful shutdown (SIGTERM/SIGINT)
- docker-compose.yml: PostgreSQL 16, NATS, Valkey
- Makefile: `build`, `test`, `lint`, `docker-up`, `docker-down`
- Dockerfile (multi-stage), `.golangci.yml`

**NOT in this phase**: No migrations, no schema, no models, no adapters, no NATS streams, no middleware beyond logging. TimescaleDB/AGE/pgvector present in docker-compose but not used in code.

### Acceptance Criteria
- `./loom serve` → connects, serves `/healthz` → 200, SIGTERM → clean shutdown
- `go test ./...` passes, `golangci-lint run` passes

---

## Phase 1B: Core Persistence + Registry
**Scope: M | Output: Create/list devices, register adapters, publish events, audit**

Only:
- Canonical model types (from Phase 0 DOMAIN-MODEL.md)
- goose migrations: `tenants`, `devices`, `endpoints`, `credentials`, `observed_facts`, `audit_events`
- Every table has `tenant_id` — enforced
- Repository layer: `DeviceRepository`, `TenantRepository`, `AuditRepository`
- Adapter registry (from Phase 0 ADAPTER-CONTRACT.md): register by protocol, lookup by capability
- Mock adapter implementing Connector + Discoverer
- NATS JetStream stream + subject setup (from Phase 0 EVENT-MODEL.md)
- Event publisher (audit envelope from Phase 0 AUDIT-MODEL.md)
- CLI: `loom migrate up/down/status`

**Keep it dumb**: persist rows, publish envelopes, register adapters, migrate schema. No business orchestration logic. No smart event processing. No subscribers unless strictly needed.

### Acceptance Criteria
- `./loom migrate up` creates all tables with `tenant_id` on every row
- Create tenant → create device (scoped) → list devices (filtered) → works
- Mock adapter registers with `[connect, discover]`, found by registry lookup
- Device creation publishes `loom.{tenant}.device.created` to JetStream
- Audit recorder persists with full envelope (actor, correlation_id, causation_id)
- Schema matches Phase 0 domain model doc

---

## Phase 2: Inventory Vertical Slice — SSH + Redfish
**Scope: M | Output: Discover one real host, normalize facts, store, show via CLI**

**One target. One adapter. One path. No subnet scanning. No LLM. No multi-provider failover.**

- SSH adapter: Connector + Discoverer (OS, hostname, CPU, memory, disk, NICs)
- Redfish adapter: Connector + Discoverer + Executor (BMC inventory, power control, boot device)
- Discovery service with precedence rules (from Phase 0 IDENTITY-MODEL.md):
  - First successful primary identity → create device
  - Additional protocols → enrich (add endpoint + facts), not duplicate
  - Merge based on identity confidence rules
- CLI: `loom discover --target <ip> --protocol ssh|redfish`
- CLI: `loom devices list`, `loom devices show <id>`
- Facts stored as `ObservedFact` with adapter provenance

### Acceptance Criteria
- `./loom discover --target <ip> --protocol ssh` → discovers, stores, shows
- `./loom discover --target <ip> --protocol redfish` → discovers, stores, shows
- Same host discovered via both → merged into one device (not duplicated)
- Facts are tenant-scoped, audited, event-published

---

## Phase 3: Expanded Discovery — SNMP, Scanner, Classifier
**Scope: M | Output: Scan subnets, fingerprint, deduplicate, optional LLM enhancement**

Internal sequencing (enforced):
1. SNMP adapter (Connector + Discoverer)
2. Subnet scanner (port probe → detect available protocols)
3. Deterministic fingerprinter (sysObjectID → vendor/model, Redfish schema → type)
4. Dedup/merge logic (identity rules from Phase 0)
5. Optional LLM classifier (enhances, does not replace deterministic classification)

LLM classifier outputs: `{category, vendor, model, confidence, evidence[]}` — not just a label.

LLM is optional: system works without it. If unavailable, deterministic classification only.

### Acceptance Criteria
- Subnet scan discovers devices via SSH/Redfish/SNMP
- Multi-protocol device merged, not duplicated
- Fingerprinter classifies known vendor/model from sysObjectID
- LLM (when available) improves confidence; system works without it

---

## Phase 4: Durable Workflows — Temporal, Approvals, Rollback, Verification
**Scope: L | Output: Durable orchestration with saga, approval, verification**

- Temporal client + worker
- Discovery workflow (scan → probe → classify → store) — Temporal durable
- Provisioning workflow with saga (plan → approve → execute → **verify** → rollback on failure)
- Power-cycle workflow: off → wait → on → verify reachable
- Verification activities (from Phase 0 VERIFICATION-MODEL.md):
  - Workflow declares `ExpectedOutcome`
  - Checks map to expected outcome
  - `VerificationResult` stored separately
  - **Failed verification = failed workflow**
- IPMI adapter (Connector + Executor for power control)
- Human approval via Temporal signals
- Workflow status projection to DB (Temporal = truth, DB = projection)
- CLI: `loom workflow run/status/list/cancel/signal`

Activity granularity rules:
- One step = one meaningful business action
- One activity = one retryable side effect
- Verification = its own activity
- Compensation = its own activity

### Acceptance Criteria
- Workflows visible in Temporal UI
- Power-cycle: off → on → **verification confirms reachable** → success
- Failed step → saga compensates (LIFO)
- Approval signal resumes paused workflow
- Kill + restart → workflow resumes from exact point
- System works with no LLM configured

---

## Phase 5: Operator API — REST, Auth, Tenant Scoping, SSE
**Scope: L | Output: REST API with auth, workflow submission, real-time updates**

- REST API (chi router, `/api/v1/`)
- JWT auth + RBAC (admin, operator, viewer per tenant)
- API conventions from Phase 0 (cursor pagination, correlation ID, idempotency header, error schema)
- Handlers: devices, workflows, discovery, events (SSE)
- Valkey cache for hot device reads
- Rate limiting (Valkey-backed)

### Acceptance Criteria
- Tenant filtering enforced on all endpoints
- Unauthenticated → 401, wrong tenant → 403
- SSE streams workflow updates in real time
- API follows conventions doc (pagination, errors, idempotency)

---

## Phase 6: Thin UI — Inventory, Workflows, Approvals
**Scope: M | Output: Minimal operator lens, not a product stream**

- React + Next.js + shadcn/ui
- Device inventory (TanStack Table)
- Workflow list + detail (step-by-step, logs)
- Approval button (triggers Temporal signal)
- Dark mode, mobile-responsive

**NOT**: No Cytoscape.js, no React Flow, no ECharts. Step list is sufficient. Add visualization when operators need it.

### Acceptance Criteria
- UI shows devices, workflows, allows approvals
- Dark mode works, tablet usable

---

## Phase 7: Network Domain — Multi-Vendor Config Translation
**Scope: XL | Output: Configure switches across 4+ vendors from universal model**

- Universal network model: VLAN, Interface, Route, ACL, Subnet
- Config translation engine: universal → vendor-specific
- NETCONF, gNMI, eAPI, NX-API adapters
- Vendor translators: SONiC, EOS, IOS-XE, NX-OS, Junos, MikroTik
- Network provisioning workflow (multi-vendor in one Temporal workflow)
- AMT/PiKVM are NOT in this phase (separate concern → Phase 11)

### Acceptance Criteria
- Same VLAN on 3+ vendors from universal model
- Network workflow: VLANs + ports + ACLs across multi-vendor

---

## Phase 8: Cost Engine + Lifecycle Management
**Scope: L | Output: Cost tracking, TTL/budget/idle teardown**

- Cost model (bare metal amortization, VM fractional, cloud pricing)
- Per-tenant/resource cost tracking
- Lifecycle policies: TTL, idle detection, budget caps
- Teardown workflows (Temporal)
- Cost dashboard (ECharts — first real UI chart use)
- Use TimescaleDB only if volume justifies it; plain indexed timestamps otherwise

### Acceptance Criteria
- TTL enforcement works, budget cap enforcement works
- Cost dashboard shows spend over time

---

## Phase 9: Verification Pipeline + Drift + Compliance
**Scope: L | Output: Structured reports, drift, compliance, graph queries**

Internal sequencing:
1. Expanded verification reports (structured JSON + markdown)
2. Drift detection (periodic adapter queries vs declared state)
3. Compliance policy engine (data residency, encryption)
4. AGE graph queries (blast radius)
5. Topology visualization in UI (Cytoscape.js — now there's data)
6. Request gateway with intent parsing (mark as "intent system seed" — scope carefully)

RAG is **optional and non-blocking** — convenience for operator assistance, not a core dependency.

### Acceptance Criteria
- Structured verification report after provisioning
- Drift detected when manual change made
- Blast radius query returns correct set via AGE

---

## Phase 10: Hardening — HA, Security, Observability
**Scope: L | Output: Production-ready core**

- HA: leader election (PostgreSQL advisory locks)
- TLS everywhere, mTLS between components
- Secrets: Vault or encrypted-at-rest
- Row-level security in PostgreSQL
- Prometheus metrics, OpenTelemetry tracing
- Tamper-evident audit log (hash chain)
- Helm chart + systemd units

### Acceptance Criteria
- Kill leader → standby takes over ≤30s
- All communication TLS, credentials encrypted at rest
- Full OTel trace from request → workflow → adapter → verification

---

## Phase 11: Edge Agents + Remaining Adapters
**Scope: L | Output: Edge deployment, full adapter coverage**

- Edge agent (`loom-edge`): SQLite + NATS leaf node + store-and-forward
- Remaining adapters: AMT, PiKVM, vSphere, Proxmox, libvirt, RouterOS, NVUE, OPNsense, UniFi
- pgvector RAG pipeline (optional, non-blocking — vendor doc embedding for LLM grounding)
- Comprehensive e2e test suite

### Acceptance Criteria
- Edge agent buffers when disconnected, syncs when reconnected
- E2e tests pass full flow across 3+ infrastructure types

---

## Phase Dependency Graph

```
Phase 0 (Contracts) ← design only, no code
    ↓
Phase 1A (Bootstrap Runtime)
    ↓
Phase 1B (Core Persistence + Registry)
    ↓
Phase 2 (SSH + Redfish Vertical Slice)
    ↓
Phase 3 (SNMP + Scanner + Classifier)
    ↓
Phase 4 (Temporal Workflows + Verification)
    ↓
Phase 5 (REST API + Auth)
    ↓
Phase 6 (Thin UI)
    ↓
Phase 7 (Network Domain)
    ↓
Phase 8 (Cost + Lifecycle)
    ↓
Phase 9 (Verification + Drift + Compliance)
    ↓
Phase 10 (HA + Security + Observability)
    ↓
Phase 11 (Edge + Remaining Adapters)
```

## Implementation Watches

1. **Phase 1B scope**: Keep it dumb. Persist rows, publish envelopes, register adapters. No orchestration logic.
2. **Adapter semantics**: Enforce typed operations from Phase 0. No `map[string]any`. Code review rule.
3. **Discovery duplicates**: Identity rules from Phase 0 must be implemented before Phase 2 can merge.
4. **Temporal abuse**: Not every minor action needs a workflow. Apply granularity rules from Phase 0.
5. **LLM creep**: Keep it boxed. Deterministic fallback required. Enforce in code review.
6. **TimescaleDB/AGE/pgvector**: Present in docker-compose. Used in code only when the phase requires it.
7. **RAG**: Optional, non-blocking. Not a core dependency.
8. **Request gateway** (Phase 9): This is the seed of "LOOM as intent system." Scope very carefully.

---

## Appendix A: Resource Estimates

| Phase | Engineers | Weeks | Skills Required | External Dependencies |
|-------|-----------|-------|-----------------|----------------------|
| **0: Contracts** | 1–2 | 3–4 | Go backend (senior), network engineering (reviewer) | None — design only. Second AI agent for contract review. |
| **1A: Bootstrap Runtime** | 1–2 | 2–3 | Go backend | PostgreSQL 16, NATS server, Valkey — all via docker-compose. CI platform (GitHub Actions or similar). |
| **1B: Core Persistence + Registry** | 2–3 | 4–5 | Go backend (2), security (RLS review) | Same as 1A. goose migration tooling. |
| **2: SSH + Redfish Vertical** | 2–3 | 5–6 | Go backend (1–2), network engineering (1) | At least 1 physical server with BMC (iDRAC/iLO) for Redfish testing. SSH-accessible Linux host. Lab hardware or cloud bare metal. |
| **3: SNMP + Scanner + Classifier** | 2–3 | 5–7 | Go backend (1–2), network engineering (1) | SNMP-enabled switches/routers in lab. LLM API key or local GPU (16GB+ VRAM) for optional classifier. Vendor MIB files. |
| **4: Temporal Workflows** | 2–3 | 6–8 | Go backend (2, Temporal experience preferred), network engineering (1) | Temporal server (self-hosted). IPMI-capable hardware for power control testing. Integration test environment with real/simulated devices. |
| **5: REST API + Auth** | 2–3 | 4–6 | Go backend (1–2), security (1, JWT/RBAC) | JWT identity provider (Keycloak, Auth0, or equivalent) for testing. Load testing tools. |
| **6: Thin UI** | 1–2 | 4–5 | React frontend (1–2) | API from Phase 5 must be stable and documented. Design system decisions (shadcn/ui). |
| **7: Network Domain** | 3–5 | 10–14 | Go backend (2), network engineering (2–3, multi-vendor) | Physical or virtual switches from 4+ vendors (SONiC, EOS, IOS-XE, NX-OS, Junos, MikroTik). NETCONF/gNMI/eAPI/NX-API access. Vendor documentation. Lab network with multi-vendor topology. |
| **8: Cost + Lifecycle** | 2–3 | 5–7 | Go backend (1–2), frontend (1) | Cloud billing API access (if tracking cloud costs). TimescaleDB extension validation. Historical cost data for testing models. |
| **9: Verification + Drift + Compliance** | 3–4 | 8–10 | Go backend (2), network engineering (1), security (1) | Apache AGE extension for PostgreSQL. pgvector if RAG pipeline included. Compliance policy templates. Production-like dataset for graph queries. |
| **10: HA + Security + Hardening** | 2–4 | 6–8 | Go backend (1–2), security (1–2), SRE/platform (1) | HashiCorp Vault (or equivalent). TLS certificate infrastructure (CA). Prometheus + Grafana + OTel collector. Penetration testing engagement. FIPS-validated crypto libraries if enterprise customers require it. |
| **11: Edge + Remaining Adapters** | 3–5 | 8–12 | Go backend (2–3), network engineering (1–2) | Edge hardware (Raspberry Pi, NUC, or equivalent). AMT-capable Intel vPro system. PiKVM/BliKVM unit. vSphere/Proxmox/libvirt hypervisors. RouterOS/NVUE/OPNsense/UniFi devices. GPU server for pgvector RAG pipeline (optional). |

### Summary Totals

- **Total calendar time (sequential)**: ~68–95 weeks (~16–22 months)
- **Peak team size**: 5–6 engineers (Phases 7, 9, 11)
- **Minimum viable team**: 2 senior Go engineers + 1 network engineer from Phase 0 through Phase 4, scaling up for Phase 7+
- **Critical hiring**: Go engineers with infrastructure/systems experience are the hardest to find. Network engineers who can also write Go are rarer still. Start recruiting during Phase 0.
- **Parallelization opportunity**: Phases 6 (UI) can overlap with Phase 5 tail-end. Phase 8 (Cost) can partially overlap with Phase 7 if separate engineers are available. This can compress the timeline by 4–8 weeks.

---

## Appendix B: Risk Register

| # | Risk | Likelihood | Impact | Mitigation | Owner |
|---|------|-----------|--------|------------|-------|
| 1 | **Temporal learning curve delays Phase 4** — Team has no Temporal experience; Go SDK semantics (deterministic workflow code, replay constraints) are non-obvious | High | High | Allocate 2 weeks for Temporal training before Phase 4 starts. Build a throwaway prototype workflow in Phase 1B timeframe. Hire or contract one Temporal-experienced engineer. | Tech Lead |
| 2 | **Multi-vendor network config translation is harder than expected** — Vendor CLI/API semantics differ in undocumented ways (e.g., VLAN trunk behavior on EOS vs IOS-XE vs NX-OS) | High | High | Start with 2 vendors in Phase 7, not 4. Build vendor-specific integration test suites with real devices. Budget 2x the estimated time for the third and fourth vendors. Engage vendor TAC contacts early. | Network Lead |
| 3 | **LLM hallucination rate too high for production config generation** — Generated network/server configs contain plausible but incorrect values | Medium | Critical | Enforce multi-layer validation pipeline (schema check → syntax parse → dry-run → human approval). Never allow LLM-generated configs to execute without deterministic validation. Track hallucination rate metrics. Maintain deterministic fallback for every LLM code path. | LLM Lead |
| 4 | **Go protocol libraries have undocumented bugs at scale** — gosnmp, gofish, go-wsman-messages work in testing but fail with specific firmware versions or at high concurrency | Medium | High | Pin library versions. Build adapter-level integration tests against multiple firmware versions. Maintain a device compatibility matrix. Budget time for upstream bug reports and local patches. Vendor device farms for regression testing. | Adapter Lead |
| 5 | **PostgreSQL + 3 extensions creates upgrade complexity** — TimescaleDB, AGE, and pgvector may not all support the same PostgreSQL major version simultaneously, blocking upgrades | Medium | High | Track extension compatibility matrices quarterly. Test upgrades in staging 60 days before production. Defer TimescaleDB/AGE/pgvector usage until the phase that requires them (not Phase 1). Maintain a rollback-tested upgrade runbook. Consider managed PostgreSQL with extension support. | Platform Lead |
| 6 | **Community/open-source dependencies change licensing** — Key dependency (e.g., Valkey fork instability, Temporal licensing change, NATS license shift) disrupts project | Low | High | Audit all dependency licenses quarterly. Maintain a vendor exit plan for each critical dependency. Prefer Apache 2.0 / MIT / BSD / PostgreSQL-licensed projects. Monitor Temporal's BSL-to-Apache timeline. Evaluate whether Valkey community remains healthy. | Tech Lead |
| 7 | **Hiring Go + infrastructure engineers is competitive** — Senior Go engineers with network/systems experience command $200K+ and are scarce | High | High | Start recruiting during Phase 0. Consider remote-first to widen pool. Offer open-source contribution visibility as a hiring perk. Cross-train Python/Rust engineers into Go during Phase 1. Budget for 2–3 months of hiring pipeline lead time per engineer. | Engineering Manager |
| 8 | **Air-gapped LLM performance is insufficient** — 7B/14B local models produce config recommendations too unreliable for operator trust | Medium | Medium | Benchmark local models against cloud models on LOOM-specific tasks during Phase 3. Fine-tune a domain-specific model on network config datasets. Define minimum acceptable confidence threshold — below it, fall back to deterministic-only. Document which tasks require 70B+ models. | LLM Lead |
| 9 | **Enterprise customers require FIPS 140-2 that Go crypto doesn't fully support** — Go's standard crypto library is not FIPS-certified; BoringCrypto fork has limited support | Medium | High | Use Go's BoringCrypto build mode (`GOEXPERIMENT=boringcrypto`) where possible. Document which crypto operations are covered vs not. Engage with customers early to understand exact FIPS requirements (full validation vs FIPS-like). Budget for external FIPS validation consulting if needed. | Security Lead |
| 10 | **Temporal's event history size limits force architectural rework** — Workflows with 1000+ activities hit Temporal's 50K event limit, requiring child workflow decomposition patterns | Medium | Medium | Design workflow decomposition rules in Phase 0 contracts. Enforce maximum activity count per workflow (e.g., 200). Use ContinueAsNew for long-running workflows. Build this into the workflow contract so it's not a surprise in Phase 4. | Tech Lead |
| 11 | **Identity model merge/split rules produce incorrect device deduplication** — Real-world device identity is messier than modeled (e.g., shared BMC IPs, hostname collisions, DHCP churn) | High | Medium | Build extensive integration tests with adversarial identity scenarios. Implement manual merge/split override in UI (Phase 6). Log all merge/split decisions for audit. Start with conservative merge rules (high confidence only) and loosen based on operational feedback. | Backend Lead |
| 12 | **Phase 0 contracts are incomplete — seams discovered during implementation** — Despite thorough Phase 0, real code reveals interaction patterns not anticipated in contracts | Medium | Medium | Treat Phase 0 contracts as living documents, not frozen specs. Allow contract amendments during implementation with ADR justification. Assign contract review as a continuous responsibility, not a one-time phase. Each phase retrospective should check for contract gaps. | Tech Lead |
| 13 | **NATS JetStream message ordering/delivery semantics misunderstood** — Event-driven flows depend on ordering guarantees that JetStream provides only under specific configurations | Medium | Medium | Build a JetStream behavior test suite in Phase 1B that validates exactly-once, ordering, and replay semantics. Document required stream/consumer configuration. Avoid designs that require global ordering — use per-device or per-tenant streams. | Backend Lead |
| 14 | **UI becomes a development bottleneck** — Single frontend engineer cannot keep pace with API changes from Phases 5–9, creating growing feature backlog | Medium | Medium | Keep UI minimal (Phase 6 scope). Use auto-generated API clients (OpenAPI → TypeScript). Defer advanced visualizations (Cytoscape, React Flow, ECharts) until Phase 9 when there is real data. Consider hiring a second frontend engineer before Phase 7 if API surface grows. | Engineering Manager |
| 15 | **Scope creep from "orchestrator of orchestrators" vision** — Each phase invites expansion ("why not also manage DNS/DHCP/IPAM/monitoring/backup?") that delays core delivery | High | High | Enforce phase acceptance criteria strictly — done means the listed criteria pass, not "done plus bonus features." Maintain a backlog of out-of-scope requests for future phases. Tech Lead has veto on scope additions mid-phase. Every scope addition requires an ADR. | Tech Lead |

---

## Appendix C: Go/No-Go Criteria

| Phase | Go Criteria | No-Go Triggers |
|-------|------------|----------------|
| **0: Contracts** | All 11 contract documents + 10 ADRs committed. Second AI agent reviews all contracts and finds zero ambiguous seams. Domain model has Go-style struct definitions with field names, types, and JSON tags. Identity model unambiguously answers: "same server discovered via SSH and Redfish" → single device. Operation types have zero `map[string]any`. Every contract answers its acceptance criteria. | Any contract document has unresolved TODOs or "TBD" placeholders. Reviewers identify ambiguous interaction between two contracts (e.g., audit model references an actor type not defined in identity model). Domain model structs would force adapter implementers to invent local types. |
| **1A: Bootstrap** | `./loom serve` starts, connects to PostgreSQL and NATS, serves `GET /healthz` → 200, handles SIGTERM with clean shutdown and zero goroutine leaks. `./loom version` prints version, commit SHA, and build date. `go test ./...` passes with >80% line coverage. `golangci-lint run` passes with zero warnings. CI pipeline (GitHub Actions) builds on Linux, macOS, and Windows. Docker-compose brings up all infrastructure dependencies. Makefile targets `build`, `test`, `lint`, `docker-up`, `docker-down` all work. | Binary panics on startup with any supported config combination. Health check returns 200 when PostgreSQL or NATS is unreachable. Graceful shutdown takes >5 seconds or leaks connections. Test coverage <80%. Lint violations present. CI does not build on all three platforms. |
| **1B: Core Persistence** | `./loom migrate up` creates all tables with `tenant_id` on every row. CRUD operations work for tenants, devices, endpoints, credentials, observed facts. Row-level tenant isolation verified: Tenant A cannot read Tenant B's devices via repository layer. Mock adapter registers with `[connect, discover]` capabilities and is found by registry lookup. Device creation publishes `loom.{tenant}.device.created` to JetStream (verified by test consumer). Audit recorder persists full envelope with actor, correlation_id, causation_id. Schema matches Phase 0 domain model — zero drift. Test coverage >80%. | Any table missing `tenant_id`. Cross-tenant data leakage possible through any repository method. Event envelope missing required fields from Phase 0 EVENT-MODEL.md. Adapter registry allows registration without capability declaration. Schema diverges from Phase 0 domain model without an approved ADR. |
| **2: Inventory Vertical** | SSH adapter discovers host facts (OS, hostname, CPU, memory, disk, NICs) from a real Linux host. Redfish adapter discovers BMC inventory and can execute power control (on/off/reset) on a real server. Same host discovered via SSH and Redfish produces exactly one device (not two) with both endpoints attached. Facts are tenant-scoped, audited, and event-published. `loom discover --target <ip> --protocol ssh` and `loom discover --target <ip> --protocol redfish` both work end-to-end. `loom devices list` and `loom devices show <id>` display correct, merged data. Integration tests pass against real or simulated hardware. | SSH adapter fails on common Linux distributions (Ubuntu 22.04, RHEL 9, Debian 12). Redfish adapter fails on any of Dell iDRAC 9, HPE iLO 5, or Supermicro BMC. Dual-protocol discovery creates duplicate devices. Facts from one protocol overwrite (not supplement) facts from another. No integration tests against real/simulated hardware. |
| **3: Expanded Discovery** | SNMP adapter discovers device facts via SNMPv2c and SNMPv3 from real switches/routers. Subnet scanner detects available protocols on a /24 subnet in <30 seconds. Deterministic fingerprinter correctly classifies known vendor/model from sysObjectID for 10+ common device types. Multi-protocol device discovery (SSH + Redfish + SNMP) merges into one device. LLM classifier (when available) improves classification confidence; system produces correct classifications without LLM. | SNMP adapter fails with common enterprise switches (Cisco, Arista, Juniper). Subnet scan takes >2 minutes for a /24. Fingerprinter misclassifies common devices. LLM unavailability prevents any classification. |
| **4: Temporal Workflows** | Workflows visible and manageable in Temporal UI. Power-cycle workflow (off → wait → on → verify reachable) completes successfully on real hardware. Failed workflow step triggers saga compensation in LIFO order. Human approval signal pauses workflow indefinitely and resumes correctly. Kill LOOM binary mid-workflow + restart → workflow resumes from exact interruption point. Verification activity confirms expected outcome; failed verification = failed workflow even if execution returned success. All workflows run without LLM configured (deterministic mode). p99 workflow start latency <500ms. Event history per workflow stays under 10K events for standard operations. | Workflows do not survive LOOM restart. Saga compensation executes out of order or skips steps. Verification passes when actual state does not match expected state. Any workflow requires LLM to function. Single workflow exceeds 50K Temporal events without using ContinueAsNew. Approval signal lost or processed incorrectly. |
| **5: REST API + Auth** | All endpoints enforce tenant scoping — no cross-tenant data leakage. Unauthenticated requests return 401. Requests to wrong tenant return 403. JWT validation works with at least one standard IdP (Keycloak or Auth0). Cursor-based pagination works correctly on all list endpoints. `X-Correlation-ID` header propagated through all layers to audit log. `Idempotency-Key` header prevents duplicate mutations. SSE endpoint streams workflow updates in real time with <1s latency. Error responses match Phase 0 API-CONVENTIONS.md schema. Rate limiting prevents >100 req/s from single client. API handles 1000 concurrent connections without degradation. | Any endpoint returns data from another tenant. JWT bypass possible through any code path. Pagination returns duplicate or missing results. Correlation ID lost between API and audit log. SSE latency >5s. Rate limiting absent or bypassable. |
| **6: Thin UI** | Device inventory table renders 1000+ devices with <100ms interaction latency (TanStack Table virtualization). Workflow list shows status, steps, and timing. Approval button triggers Temporal signal and workflow resumes within 5 seconds. Dark mode renders correctly on all views. Tablet viewport (768px) is usable without horizontal scrolling. UI communicates only through Phase 5 REST API — no direct database or NATS access. | UI makes direct backend calls bypassing the API layer. Table unusable at 500+ rows (jank, freezing). Approval button does not trigger workflow resumption. Dark mode has unreadable text or missing contrast. Mobile/tablet layout is broken. |
| **7: Network Domain** | Universal network model (VLAN, Interface, Route, ACL) translates correctly to at least 3 vendor-specific configs (e.g., SONiC + EOS + IOS-XE). Same VLAN created on 3+ vendors from one universal model definition. Network provisioning workflow configures VLANs + ports + ACLs across multi-vendor topology in one Temporal workflow. Config translation produces syntactically valid vendor configs (verified by vendor CLI parser or simulation). At least 4 adapters (NETCONF, gNMI, eAPI, NX-API) pass integration tests. Rollback removes configs cleanly on workflow failure. | Config translation produces invalid vendor syntax. Same universal model produces conflicting configs across vendors. Workflow cannot handle partial failure (2 of 3 switches configured, 1 fails). No integration tests against real or simulated network devices. Fewer than 3 vendors working. |
| **8: Cost + Lifecycle** | Cost model calculates per-resource and per-tenant costs. TTL enforcement automatically triggers teardown workflow when resource expires. Budget cap enforcement prevents provisioning when tenant exceeds budget. Idle detection identifies resources with no traffic/usage for configurable threshold. Cost dashboard displays spend-over-time with correct aggregations. Teardown workflows execute with proper dependency ordering (apps before compute before network). | Cost calculations produce negative values or obvious errors. TTL enforcement has >1 hour drift from configured deadline. Budget cap can be bypassed by concurrent requests. Teardown workflow deletes network before applications. |
| **9: Verification + Drift + Compliance** | Structured verification reports generated as JSON + human-readable markdown after every provisioning workflow. Drift detection catches manual changes within one polling interval (configurable, default 5 minutes). Compliance policy engine evaluates data residency and encryption rules. AGE graph query returns correct blast radius for a given router/switch failure. Topology visualization renders in UI with Cytoscape.js showing real device relationships. | Drift detection misses manual configuration changes. Blast radius query returns incorrect device set. Compliance engine produces false negatives (passes non-compliant configs). Verification reports missing for any workflow type. |
| **10: HA + Security + Hardening** | Leader election failover completes in ≤30 seconds when primary is killed. All inter-component communication uses TLS (mTLS between LOOM components). Credentials encrypted at rest in Vault (or equivalent). Row-level security in PostgreSQL prevents direct SQL access to other tenants' data. Prometheus metrics and OTel traces available for full request lifecycle (API → workflow → adapter → verification). Audit log tamper-evident (hash chain verifiable). Helm chart deploys complete stack on Kubernetes. systemd units deploy on bare Linux. | Failover takes >60 seconds or requires manual intervention. Any inter-component communication is plaintext. Credentials stored in plaintext anywhere. RLS bypassable via direct SQL. No observability for adapter-level operations. Audit log entries can be modified without detection. |
| **11: Edge + Remaining Adapters** | Edge agent (`loom-edge`) buffers operations when disconnected from central LOOM. Edge agent syncs buffered data when connection restored — zero data loss. At least 5 new adapters (AMT, PiKVM, vSphere, Proxmox, libvirt) pass integration tests. End-to-end test suite passes full flow (discover → provision → verify → teardown) across 3+ infrastructure types (bare metal + VM + network). Edge agent runs on Raspberry Pi 4 (4GB RAM) with acceptable performance. | Edge agent loses buffered data on reconnection. Any new adapter crashes LOOM core process (isolation failure). E2E test suite does not cover full lifecycle. Edge agent requires >8GB RAM or dedicated GPU. |
