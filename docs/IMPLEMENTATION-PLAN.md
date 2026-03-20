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
