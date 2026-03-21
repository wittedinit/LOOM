# LOOM Application-Layer Gap Analysis

> Principal Architect review of application-layer gaps, untested assumptions, and areas requiring further research.
> Reviewer perspective: 15+ years building enterprise platforms (developer experience, API design, observability, CI/CD, application lifecycle).

---

## Summary

LOOM's contract documentation is unusually thorough for a Phase 0 project. The domain model, error taxonomy, identity model, and adapter contracts are well-defined with concrete Go types. The adversarial review and edge case catalog show genuine self-awareness about hard problems. However, this analysis identifies **27 gaps** across 13 categories that, if left unaddressed, will compromise developer adoption, operational debuggability, and the project's ability to scale from design to production.

The most critical theme: **LOOM over-specifies the runtime architecture and under-specifies the developer experience.** There is no adapter SDK, no scaffolding tool, no testing harness documentation, no contributor guide, and no example adapter walkthrough. A project that needs 25+ protocol adapters to deliver its value proposition cannot afford to make adapter development difficult.

---

## Gap 1: No Adapter SDK or Developer Experience Story

**Where found:** ADAPTER-CONTRACT.md, IMPLEMENTATION-PLAN.md (Phases 2-11), TESTING-STRATEGY.md

**The assumption being made:** That the Go interface definitions in ADAPTER-CONTRACT.md are sufficient for someone to write a new adapter. That the five interfaces (Connector, Discoverer, Executor, StateReader, Watcher) plus error types plus idempotency contract plus compensation contract are self-explanatory.

**Why it's risky:** LOOM's entire value scales with adapter count. If writing an adapter requires reading 6 contract documents, understanding the idempotency cache TTL rule, correctly classifying errors into 4 categories, implementing compensation metadata, building a mock server from scratch, and wiring into the registry -- most potential contributors will give up. The adapter contract describes *what* to implement but never *how* to get started.

Missing:
- No `loom adapter scaffold <protocol>` CLI command that generates boilerplate
- No example adapter walkthrough (e.g., "Build an SNMP adapter in 200 lines")
- No adapter development guide ("here's the happy path, here are the 7 error patterns to handle")
- No shared test harness (each adapter builds its own mock server per TESTING-STRATEGY.md -- this is quadratic effort)
- No adapter conformance test suite ("run this against your adapter to verify it implements the contract correctly")
- No documentation on how to test an adapter against a real device in a lab vs. mock

**What research is needed:**
1. What does a minimal adapter look like? Write one (e.g., a dummy HTTP adapter) and measure: lines of code, number of contract documents referenced, time to first successful test.
2. Can a shared `MockDevice` framework reduce per-adapter test boilerplate? What protocol-specific test infrastructure is truly needed vs. generic?
3. Should the adapter conformance suite be a Go test suite that adapter authors import, or a CLI tool that exercises the adapter externally?

**Severity:** Critical

---

## Gap 2: Plugin/Adapter Lifecycle (Versioning, Hot-Reload, Dependency Isolation)

**Where found:** ADAPTER-CONTRACT.md (AdapterRegistration, Factory), ADR-012 (go-plugin mentioned), TECH-STACK.md (HashiCorp go-plugin "escape hatch"), ADVERSARIAL-REVIEW.md Section 5.3

**The assumption being made:** That all adapters are compiled into a single Go binary. The adversarial review raises this (Section 5.3) but the only response is "inherent to the phased approach."

**Why it's risky:**
- A customer discovers a bug in the Redfish adapter. They must wait for a new LOOM release that includes the fix, then upgrade the entire binary. They cannot patch one adapter independently.
- A community contributor writes a MikroTik adapter. They must fork the LOOM repo, add their adapter, and maintain their fork. They cannot distribute a standalone plugin.
- ADR-012 mentions "plugin adapters are separate binaries (via go-plugin gRPC)" as an IP protection measure, but this architecture is never specified in the adapter contract, implementation plan, or deployment topology. It is a throwaway line in a security ADR.
- No versioning scheme exists for individual adapters. If the Connector interface changes in LOOM v0.4, do all adapters break? Is there an adapter API version?

**What research is needed:**
1. Should LOOM support out-of-process adapters via go-plugin from day one, or is this a post-MVP concern? What is the performance overhead of gRPC per adapter call vs. in-process?
2. What is the adapter API stability contract? Can the Connector interface change between LOOM minor versions?
3. How do third-party adapter authors distribute their work? Go module? OCI artifact? Sidecar container?

**Severity:** High

---

## Gap 3: REST API Design Completeness

**Where found:** API-CONVENTIONS.md, API-DOCUMENTATION.md, IMPLEMENTATION-PLAN.md Phase 5

**The assumption being made:** That the API conventions document (73 lines) is a complete API contract.

**Why it's risky:** The API conventions cover pagination, error format, headers, and rate limiting -- all correct and well-specified. But the actual API surface is never defined. There is no route table. No endpoint list. No resource schema for request/response bodies. API-DOCUMENTATION.md says "huma generates OpenAPI from Go structs," which is correct, but means the API surface is undefined until code is written in Phase 5.

Missing:
- No endpoint inventory (what routes exist? `/api/v1/devices`, `/api/v1/workflows`, ... what else?)
- No CRUD contract per resource (which resources support POST? PATCH? DELETE? What are the request bodies?)
- No bulk operation design (discover 1000 devices -- is that 1000 POST requests or 1 POST with a list?)
- No webhook/callback contract (how do external systems get notified? SSE is defined but webhooks are not)
- No search/filter semantics beyond basic query params (how do I find "all switches in rack A3 with firmware < 4.0"?)
- No field selection (GraphQL-style `?fields=id,name,status` to reduce payload size for large inventories)
- No resource expansion (do I get nested endpoints when I fetch a device, or do I make N+1 requests?)
- No batch/transaction semantics (apply VLAN config to 10 switches atomically -- how?)

**What research is needed:**
1. Draft a route table with all resources, methods, and a one-line description of each endpoint. This should exist before Phase 5 coding begins.
2. Decide on bulk operation patterns: async job submission with polling, or synchronous batch with partial failure reporting?
3. Define the webhook contract: URL registration, payload format, retry policy, HMAC verification.

**Severity:** High

---

## Gap 4: Observability of LOOM Itself (Operator Debugging Story)

**Where found:** OBSERVABILITY-STRATEGY.md, FAILURE-MODES.md

**The assumption being made:** That metrics + traces + structured logs are sufficient for operators to debug LOOM. The observability strategy is one of the strongest documents in the project -- metrics are well-designed, trace propagation is specified, log correlation IDs work end-to-end.

**Why it's risky:** The documents describe observability *outputs* (metrics, traces, logs) but not *workflows*. When something goes wrong, the operator needs a debugging playbook, not a list of metric names.

Missing:
- No runbook for common failure scenarios ("workflow stuck in `awaiting_approval` for 3 hours -- how do I investigate?")
- No `loom admin debug <workflow-id>` command that aggregates logs, traces, and Temporal history for a single workflow
- No correlation between LOOM's internal observability and the managed device's health (if a Redfish adapter call fails, can the operator see the raw HTTP request/response from the trace?)
- No request replay capability (can I re-run a failed workflow step with debugging enabled?)
- No capacity planning metrics ("how close am I to hitting the PostgreSQL connection pool limit?" exists as a metric but has no dashboard panel or alert threshold guidance)
- The Grafana dashboards are described as JSON files in `deploy/grafana/dashboards/` but do not exist yet and are not in the implementation plan
- No log aggregation strategy (where do logs go in production? stdout? file? Loki? The docs say JSON to stdout but never mention aggregation)

**What research is needed:**
1. Write 5 debugging scenarios end-to-end: "adapter timeout," "saga compensation failure," "LLM confidence too low," "event consumer falling behind," "tenant isolation violation suspected." For each, trace what an operator would do with the current tooling.
2. Should LOOM embed a lightweight log viewer in its UI (Phase 6), or is Grafana/Loki the expected log exploration tool?
3. Is a `loom admin` debug toolkit (workflow inspector, event replayer, adapter call tracer) worth building before Phase 10?

**Severity:** Medium

---

## Gap 5: LLM Integration Practicalities (Latency, Cost, Context Limits)

**Where found:** TECH-STACK.md (Decision 3), LLM-BOUNDARIES.md, ADR-007, OBSERVABILITY-STRATEGY.md (LLM metrics), ADVERSARIAL-REVIEW.md Sections 1.2, 4.1, 6.2

**The assumption being made:** That LLM calls fit cleanly into Temporal activity timeout budgets, that cost is manageable, and that the provider-agnostic abstraction can hide model quality differences.

**Why it's risky:**

Latency budget is unspecified:
- A Claude Opus call with 8K context typically takes 5-15 seconds. A Temporal activity with a 30s "short" timeout has room for one LLM call with no retry. A 5-minute "medium" timeout allows 1-2 retries.
- Local 7B models on CPU take 5-10 tokens/sec. A 500-token response takes 50-100 seconds -- this exceeds the "short" activity timeout and approaches the "medium" one.
- The LLM is called inside activities (per ADR-007), so the activity timeout must account for LLM latency. But the activity timeout defaults in WORKFLOW-CONTRACT.md were designed for adapter calls, not LLM calls.

Context window limits:
- The LLM decision engine receives structured context (cost data, device inventory, placement constraints). For a 1000-device fleet, the context for a placement decision could easily exceed 100K tokens.
- No document addresses context window management: how is the LLM context constructed? Is it a fixed template? Is there a token budget? What happens when context exceeds the model's window?

Cost governance:
- OBSERVABILITY-STRATEGY.md defines `loom_llm_cost_dollars` as a counter. But there is no per-tenant cost attribution for LLM usage. If Tenant A's complex request triggers 50 LLM calls while Tenant B's simple request triggers 2, the cost model does not attribute this correctly.
- No circuit breaker exists for LLM cost. If a workflow enters a retry loop that calls the LLM on each attempt, cost can spiral.

**What research is needed:**
1. Define explicit LLM latency budgets per task type (classification: 5s, placement: 15s, config generation: 30s, oversight: 10s). Adjust Temporal activity timeouts accordingly.
2. Prototype context construction for a 1000-device placement decision. Measure token count. Design a summarization/chunking strategy.
3. Add per-tenant LLM cost tracking to the cost model. Add a per-tenant LLM spend circuit breaker.

**Severity:** High

---

## Gap 6: Multi-Tenancy Application Layer Gaps

**Where found:** ADR-005, DOMAIN-MODEL.md (Tenant Scoping), EDGE-CASES.md (RC-04), DEVICE-LOCKING.md, MVP-SCOPE.md

**The assumption being made:** That `tenant_id` on every row plus `WHERE tenant_id = ?` on every query is a complete multi-tenancy implementation.

**Why it's risky:**

The data isolation is well-specified. What is missing is everything above the data layer:

- **Quota management:** No per-tenant limits on devices, workflows, API requests, LLM calls, or storage. A single tenant can exhaust the system's capacity.
- **Resource accounting:** No way to answer "how many devices does Tenant A manage?" without a full table scan (no materialized count).
- **Tenant lifecycle:** No contract for creating, suspending, or deleting tenants. What happens when a tenant is deleted? Are their devices orphaned? Are their workflows cancelled? Are their audit records retained?
- **Tenant configuration:** No per-tenant settings beyond `TenantID`. Tenants might need different LLM providers, different approval policies, different cost budgets, different discovery schedules. Where does per-tenant config live?
- **Shared device ownership (RC-04):** The domain model has a single `TenantID` per device. Shared physical infrastructure (switches, hypervisors) serving multiple tenants is flagged in EDGE-CASES.md but the recommended fix ("virtual device record per tenant mapping to shared physical device") is not reflected in the domain model.
- **Tenant onboarding:** No documented process for "create a new tenant and set up their first discovery."

**What research is needed:**
1. Design a `TenantConfig` model that captures per-tenant settings (quotas, LLM provider preference, approval policy, cost budget, discovery schedule).
2. Decide the shared device model: does each tenant get a virtual device projection, or is there a `PhysicalDevice` -> `TenantDevice` relationship?
3. Define tenant lifecycle operations: create, suspend (disable new operations), reactivate, delete (with data retention policy).

**Severity:** High

---

## Gap 7: Testing Strategy for 25+ Protocol Adapters

**Where found:** TESTING-STRATEGY.md, ADAPTER-CONTRACT.md

**The assumption being made:** That each adapter gets its own bespoke mock server and that the simulated fleet in E2E tests (5 Redfish + 3 SNMP + 2 SSH + 1 gNMI + 1 PiKVM = 12 mock devices) is representative.

**Why it's risky:**

Scale of the testing problem:
- 13 adapter types are listed in ADAPTER-CONTRACT.md. Each needs at minimum: happy path, auth failure, timeout, partial failure, idempotency, and protocol-specific edge cases = ~7 test patterns x 13 adapters = 91 test scenarios minimum.
- Mock server quality is the bottleneck. A Redfish mock that serves DMTF-compliant JSON is a non-trivial project. A NETCONF mock that handles `<edit-config>` with candidate/running separation is even harder. Each mock is essentially a protocol simulator.
- The E2E fleet simulates 12 devices. Real deployments target 100-10,000 devices. There is no load/scale test for concurrent adapter connections at the fleet level.

Missing:
- No contract testing (verify that the mock server and the real device produce identical responses for the same operation)
- No chaos/fault injection framework (what happens when a device drops the TCP connection mid-response? When a device takes 60 seconds to respond to an SNMP GET?)
- No regression test for device firmware variations (Dell iDRAC 8 vs. iDRAC 9 vs. iDRAC X -- different Redfish schemas, different quirks)
- No test strategy for the config translation layer (Phase 7) -- how do you verify that a VLAN config template produces correct output for Cisco IOS 15.2 vs. 17.3?

**What research is needed:**
1. Can existing open-source device simulators be used? (e.g., Cisco CML for IOS/NX-OS simulation, GNS3 for network devices, dmtf/Redfish-Mockup-Server for Redfish)
2. Should LOOM maintain a "device zoo" -- a library of captured real-device responses that mock servers replay? This is the pattern used by cloud SDK test suites.
3. What is the minimum viable mock for each adapter to catch real bugs? Define "mock fidelity levels": L1 (happy path only), L2 (error paths), L3 (protocol edge cases), L4 (firmware variation coverage).

**Severity:** High

---

## Gap 8: Event Sourcing / CQRS Implications

**Where found:** ADR-008, EVENT-MODEL.md, WORKFLOW-CONTRACT.md, FAILURE-MODES.md

**The assumption being made:** That "Temporal is truth, DB is projection" (ADR-008) is a clean CQRS separation. That the DB projection can always be rebuilt from Temporal's event history.

**Why it's risky:**

The architecture is CQRS-like but does not fully commit to event sourcing, which creates ambiguity:
- **Projection rebuild is claimed but not specified.** ADR-008 says "the DB projection can always be rebuilt from Temporal." But there is no rebuild command (`loom admin rebuild-projections`), no rebuild procedure, and no estimate of how long a rebuild takes for a 10K device fleet.
- **Two event streams exist: Temporal event history and NATS JetStream events.** These are not the same. Temporal events are internal execution state (activity started, activity completed). NATS events are domain events (device.created, workflow.completed). The DB is projected from NATS events (via the DB Projector consumer), but the claim is that Temporal is the source of truth. If NATS events are lost (MaxBytes exceeded, consumer falls behind), the DB projection diverges from Temporal truth, and there is no reconciliation path defined from Temporal -> DB.
- **NATS event retention is 90 days.** If a DB projection is corrupted after 90 days, older events have been discarded. The "rebuild from events" claim has a 90-day TTL.
- **No event schema evolution strategy.** When the `DeviceCreatedData` struct changes (new field added), what happens to old events in the stream? Are they forward-compatible? Backward-compatible? The event model says events are "immutable after publish" but does not address schema versioning.

**What research is needed:**
1. Design the projection rebuild procedure. Can you replay Temporal workflow histories to regenerate NATS events? Or must you maintain a separate event archive?
2. Define event schema versioning rules: additive-only fields? Schema registry? Version field in the event envelope?
3. Calculate the NATS storage requirement for 90-day retention at various fleet sizes (100, 1K, 10K devices) with state-change-only events (per adversarial review recommendation).

**Severity:** Medium

---

## Gap 9: UI Architecture Gaps

**Where found:** TECH-STACK.md (Decision 5), IMPLEMENTATION-PLAN.md (Phase 6, 8, 9), MVP-SCOPE.md

**The assumption being made:** That a "thin UI" in Phase 6 (device list, workflow list, approval button, dark mode) is straightforward and that the complex visualizations (Cytoscape.js topology, React Flow workflows, ECharts cost analytics) can be added incrementally.

**Why it's risky:**

Real-time updates:
- The UI consumes SSE for real-time updates. But SSE is one-directional (server to client). WebSocket is mentioned for "bidirectional ops" but never specified. What bidirectional operations does the UI perform that require WebSocket? Terminal/console access to devices? Interactive approval workflows?
- No WebSocket contract exists (message format, authentication, reconnection behavior, heartbeat).
- At 10K devices, the SSE stream could produce thousands of events per second. No client-side throttling or aggregation strategy is defined.

Topology visualization at scale:
- Cytoscape.js is chosen for topology visualization (Phase 9). Cytoscape.js performs well up to ~5K nodes with WebGL rendering, but LOOM targets 10K+ devices plus network relationships between them. A 10K-device topology with 50K relationships will not render interactively in a browser without aggressive level-of-detail management.
- No topology data API is defined. How does the UI fetch the graph? One giant JSON blob? Paginated subgraphs? Server-side pre-rendered tiles?

Mobile/responsive:
- Phase 6 says "tablet usable" but there is no mobile design consideration. Is LOOM's UI a desktop-first operations console, or does it need to work on a phone for on-call incident response?

State management:
- The tech stack specifies Zustand for client state and TanStack Query for server state. But no state management architecture is defined: what data is cached client-side? How are stale caches invalidated when SSE events arrive? What is the optimistic update strategy for approval actions?

**What research is needed:**
1. Define the real-time update contract: which resources emit SSE events, what is the event payload format, how does the client throttle high-volume streams?
2. Prototype Cytoscape.js with 10K nodes and 50K edges. Measure frame rate. Design a level-of-detail strategy (cluster nodes by rack/site, expand on click).
3. Decide: is the UI a desktop operations console or does it need mobile support? This affects the component library choice and layout strategy.

**Severity:** Medium

---

## Gap 10: CI/CD for LOOM Itself

**Where found:** CI-CD-PIPELINE.md, RELEASE-PROCESS.md, TESTING-STRATEGY.md

**The assumption being made:** That GitHub Actions with Docker Compose service containers is sufficient for CI.

**Why it's risky:**

The CI/CD pipeline is well-defined for basic unit/integration/security testing. What is missing:

- **No database migration testing.** Migrations run forward (`loom migrate up`) but there is no CI step that verifies `loom migrate down` works. Migration rollback is critical for failed deployments.
- **No upgrade path testing.** When LOOM v0.3 is deployed and v0.4 is released, what happens? Is there a CI job that deploys v0.3, populates data, then upgrades to v0.4 and verifies data integrity? This is absent.
- **No multi-architecture CI.** GoReleaser builds linux/amd64 and linux/arm64, but CI tests only run on ubuntu-latest (amd64). ARM-specific bugs (common in crypto operations, memory alignment) will not be caught.
- **E2E tests are nightly-only.** E2E tests with the simulated fleet run nightly, not on PRs. A developer can merge a change that breaks the full flow and not find out until the nightly run. This is a common source of "it worked in unit tests" regressions.
- **No performance regression detection.** Benchmarks run on-demand, not on every PR. There is no baseline comparison to detect performance regressions.
- **No canary/staged release process.** RELEASE-PROCESS.md describes "tag -> build -> release" but not "release to staging -> verify -> promote to production." For infrastructure orchestration software, this is important.

**What research is needed:**
1. Should E2E tests run on every PR (cost: ~10 minutes per run, requires fleet containers) or remain nightly?
2. Design an upgrade test: `v(N-1)` deploy -> seed data -> `v(N)` upgrade -> verify. Can this be automated in CI?
3. Should LOOM use a performance regression framework (e.g., `benchstat` for Go benchmarks with GitHub Action comparisons)?

**Severity:** Medium

---

## Gap 11: "LLM Suggests Only" Boundary Under Pressure

**Where found:** ADR-007, LLM-BOUNDARIES.md, PLACEMENT-POLICY.md, ARCHITECTURE.md

**The assumption being made:** That operators will accept "the LLM suggests, a human approves" for every production decision. That the boundary between suggestion and execution will survive contact with real users.

**Why it's risky:**

The boundary is architecturally sound (ADR-007 is one of the best decisions in the project). But it will face pressure from two directions:

- **"Why do I need to approve this?"** After the LLM correctly suggests the same placement decision 100 times in a row for routine operations, operators will demand auto-approval. LLM-BOUNDARIES.md defines confidence thresholds (> 0.7 = auto-merge) but these are for discovery classification, not for execution approval. The tiered autonomy model ("routine ops = auto-execute; production changes = human-in-the-loop") from TECH-STACK.md is never formalized into a policy framework.
- **"Can the LLM just do it?"** Natural language intent parsing (REQUEST-TO-DEVICE-FLOW.md) is the gateway drug. Once an operator can say "deploy a web server," the next request is "deploy and configure a web server" -- which requires the LLM to generate config, the operator to approve it, and the system to execute it. The workflow quickly becomes: LLM generates everything, human rubber-stamps it, system executes. The "suggest only" boundary becomes a formality.

Missing:
- No auto-approval policy framework ("for operations of type X with confidence > Y, skip human approval")
- No audit of auto-approved vs. human-approved decisions (critical for trust calibration)
- No graduated autonomy model (start conservative, earn trust over time based on track record)
- No user-facing documentation of what the LLM does and does not control

**What research is needed:**
1. Design a tiered autonomy policy: Level 0 (deterministic only), Level 1 (LLM suggests, human approves all), Level 2 (LLM suggests, auto-approve routine), Level 3 (LLM decides routine, human approves changes), Level 4 (full auto with circuit breaker). Each level has explicit criteria for promotion and demotion.
2. Build an approval analytics dashboard: how often does the human agree with the LLM? How often does the human override? Track this per operation type, per tenant, per model.
3. Define the trust calibration loop: LLM accuracy over time -> adjustable confidence thresholds -> adjustable autonomy levels.

**Severity:** Medium

---

## Gap 12: Documentation and Onboarding

**Where found:** README.md, all docs

**The assumption being made:** That 22 contract documents, 12 ADRs, and 16 research documents constitute sufficient documentation for someone to understand and use LOOM.

**Why it's risky:**

The documentation is architecturally complete but developer-hostile. It is written for the architects who designed LOOM, not for the operators who will use it or the developers who will extend it.

Missing:
- **No quickstart guide.** "I have a server. Show me LOOM discovering it in 5 minutes."
- **No tutorial.** "Build your first adapter," "Run your first workflow," "Set up multi-tenant isolation."
- **No concept guide.** "What is a Target vs. a Device vs. an Endpoint? When do I care about each?"
- **No operator manual.** "How do I deploy LOOM in production? How do I back it up? How do I upgrade?"
- **No troubleshooting guide.** "My workflow is stuck. What do I check?"
- **No glossary of terms.** (GLOSSARY.md exists based on the directory listing, but was not in the reading list -- good if it exists, but it needs to be prominent.)
- **No architecture decision navigation.** An ADR index exists, but someone new to the project does not know which ADRs matter for their use case.
- **No contribution guide.** How does someone contribute an adapter? What is the PR process? What tests must pass?

**What research is needed:**
1. Write a quickstart guide that takes someone from zero to a discovered device in 5 minutes (using Mode 1 single-binary deployment). Measure how long it actually takes.
2. Identify the 3 most common "first 30 minutes" use cases and write tutorials for each.
3. Decide: is the primary audience operators (who use LOOM) or developers (who extend LOOM)? The documentation strategy differs.

**Severity:** High

---

## Gap 13: Backward Compatibility and Upgrade Paths

**Where found:** RELEASE-PROCESS.md, API-CONVENTIONS.md, CI-CD-PIPELINE.md, DOMAIN-MODEL.md

**The assumption being made:** That semantic versioning plus "database migrations are always forward-compatible within a major version" is a sufficient upgrade story.

**Why it's risky:**

- **No NATS event schema evolution.** If an event payload struct changes between versions, old consumers (edge agents running v0.3) receiving new events (from hub running v0.4) may fail to deserialize. No schema registry, no version field in the event envelope, no forward/backward compatibility rules.
- **No Temporal workflow versioning.** If a workflow definition changes between LOOM versions, running workflows started on v0.3 that are mid-execution when the system upgrades to v0.4 will use the v0.4 code for replay. Temporal has a `GetVersion` API for this exact problem, but it is never mentioned in WORKFLOW-CONTRACT.md.
- **No adapter contract versioning.** If the `Connector` interface adds a method in v0.4, all compiled adapters break. If go-plugin is used for out-of-process adapters, the gRPC contract must be versioned independently.
- **No config file migration.** The Viper config (YAML + env vars) may change between versions. No tool exists to migrate a v0.3 config to v0.4 format.
- **No database migration rollback testing.** RELEASE-PROCESS.md describes goose migrations but never tests `down` migrations. A failed upgrade that requires rollback to v(N-1) needs working down migrations.
- **Edge agent upgrade strategy is undefined.** If the hub is upgraded but edge agents are not (common in air-gapped edge sites), what is the compatibility window? Can a v0.4 hub talk to a v0.3 edge agent?

**What research is needed:**
1. Add a version field to the NATS event envelope and define forward/backward compatibility rules (additive fields only within a major version).
2. Document the Temporal `GetVersion` pattern in WORKFLOW-CONTRACT.md and require it for all workflow changes.
3. Define the hub-edge version compatibility matrix: hub v(N) must support edge agents v(N-2) through v(N).
4. Add `loom config migrate` command that converts config files between versions.

**Severity:** High

---

## Gap 14: Telemetry Schema and Ingestion Pipeline

**Where found:** TECH-STACK.md, ADVERSARIAL-REVIEW.md Section 3.1, OBSERVABILITY-STRATEGY.md

**The assumption being made:** That telemetry (device metrics) will be stored in TimescaleDB and that the schema will be designed when needed.

**Why it's risky:** The adversarial review correctly identifies that the telemetry schema is absent from the project. No table definition for device metrics exists anywhere. The data model section of the docs covers devices, endpoints, facts, and verification results -- but not time-series metrics. Without a schema, you cannot estimate storage, query performance, or retention policy.

Additionally, the boundary between "observed facts" (in ObservedFact table) and "telemetry" (in TimescaleDB) is unclear. Is CPU utilization a fact or a metric? Is power draw a fact or a metric? The domain model does not distinguish between point-in-time observations (facts) and continuous time-series data (metrics).

**What research is needed:**
1. Define the telemetry schema: table structure, hypertable partitioning (by device_id? by tenant_id? by time?), retention policy, continuous aggregates.
2. Draw a clear line between facts (low-frequency observations stored in `observed_facts`) and metrics (high-frequency time-series stored in TimescaleDB).
3. Estimate storage requirements at 100, 1K, and 10K devices with 20 metrics per device at 1-minute resolution.

**Severity:** Medium

---

## Gap 15: Graph Database (Apache AGE) Usage Specifics

**Where found:** TECH-STACK.md, IMPLEMENTATION-PLAN.md Phase 9, ADVERSARIAL-REVIEW.md Section 3.1

**The assumption being made:** That Apache AGE inside PostgreSQL can handle topology queries (blast radius, dependency graphs) at scale.

**Why it's risky:**
- Apache AGE is relatively immature compared to Neo4j. Its openCypher support is incomplete (missing some APOC-equivalent functions, limited query optimizer).
- The `Relationship` model in DOMAIN-MODEL.md is a generic edge table with string-typed source/target types. This is a relational approximation of a graph, not a native graph model. If AGE is used, the data needs to be *in* AGE's graph storage, not in a regular SQL table with AGE queries on top.
- No graph schema is defined. What are the node types? What are the edge types? What are the required traversal queries? Without these, AGE's suitability cannot be evaluated.
- The scale-out path says "add Neo4j read replica at 4+ hops / GDS algorithms." But migrating from AGE to Neo4j mid-project requires re-modeling the graph, rewriting all queries, and running dual graph stores during transition.

**What research is needed:**
1. Define the 5 most important graph queries (blast radius, dependency chain, shortest path between devices, cluster identification, topology diff). Prototype them in AGE and measure performance at 1K and 10K nodes.
2. If AGE performance is insufficient, is the fallback to use the `Relationship` SQL table with recursive CTEs instead of openCypher? Benchmark this alternative.
3. Design the AGE graph schema: node labels (Device, Switch, VLAN, Subnet, Application), edge labels (connected_to, member_of, depends_on, runs_on), and required indexes.

**Severity:** Medium

---

## Gap 16: Credential Rotation and Lifecycle in Production

**Where found:** VAULT-ARCHITECTURE.md, ADR-011, EDGE-CASES.md (RC-03), ADVERSARIAL-REVIEW.md Section 5.2

**The assumption being made:** That the vault's envelope encryption design is sufficient for production credential management.

**Why it's risky:**

The vault design (AES-256-GCM, 3-tier envelope encryption, mlock, memory wiping) is excellent from a cryptographic standpoint. What is missing is the operational lifecycle:

- **No credential rotation workflow.** When an operator rotates a switch password, what happens? Is there a `loom credential rotate` command that updates the vault entry, re-tests connectivity, and reports success/failure across all endpoints using that credential?
- **No credential health check.** How does LOOM know a credential has expired or been changed externally (someone changed a password on the device directly)? There is no periodic credential validation.
- **No credential sharing model.** If 50 switches share the same SNMP community string, is that one CredentialRef used by 50 endpoints, or 50 CredentialRefs with the same value? The domain model allows N endpoints per CredentialRef, but no tooling exists for "update this credential and see which devices are affected."
- **Edge agent credential caching** (ADVERSARIAL-REVIEW.md Section 2.4): if an edge agent caches credentials locally (in SQLite) during a network partition, and the credential is rotated on the hub during the partition, the edge agent has stale credentials. No expiry or refresh mechanism is defined.

**What research is needed:**
1. Design the credential rotation workflow: update -> test -> propagate -> verify -> audit.
2. Define credential health check intervals: how often does LOOM verify that stored credentials still work? (Periodically reconnect? On first use only?)
3. Define edge agent credential caching policy: max cache duration, refresh-on-reconnect behavior, fallback when cached credential fails.

**Severity:** Medium

---

## Gap 17: No Disaster Recovery Procedure

**Where found:** FAILURE-MODES.md, DEPLOYMENT-TOPOLOGY.md, ADR-008

**The assumption being made:** That the failure mode catalog (which is very thorough for component failures) covers disaster recovery.

**Why it's risky:** The failure modes document covers individual component failures (PostgreSQL down, Temporal down, NATS down) and their recovery procedures. But it does not cover:

- **Full-site disaster recovery.** If the entire LOOM deployment is lost (data center fire, complete host failure), what is the recovery procedure? Which data must be backed up? In what order is the system restored?
- **Backup strategy.** No backup schedule, no backup verification, no restore procedure, no RTO/RPO targets.
- **State reconstruction.** ADR-008 says Temporal is the source of truth and the DB can be rebuilt. But Temporal's state is in PostgreSQL. If PostgreSQL is lost, both Temporal state and DB projections are gone. What is the backup for that?
- **Edge agent disaster recovery.** If a hub is rebuilt from backup, how do edge agents reconnect? Are their local SQLite databases consistent with the rebuilt hub?

**What research is needed:**
1. Define backup targets: PostgreSQL (full + WAL archiving), NATS JetStream (stream snapshots), Temporal (implicit in PostgreSQL backup), Vault keyring, configuration files.
2. Define RTO/RPO targets per deployment mode (Mode 1: RTO 1h/RPO 1h, Mode 2: RTO 15m/RPO 0, etc.).
3. Write a disaster recovery runbook: from bare metal to operational LOOM with restored state.

**Severity:** Medium

---

## Gap 18: Temporal Workflow Versioning (Determinism Contract)

**Where found:** WORKFLOW-CONTRACT.md, ADR-008

**The assumption being made:** That workflow code changes are safe to deploy while workflows are running.

**Why it's risky:** Temporal replays workflow history through the current code. If workflow code changes between when a workflow was started and when it is replayed (e.g., after a LOOM restart), the replay may produce different results, causing a non-determinism error that crashes the workflow.

Temporal provides `workflow.GetVersion()` to handle this, but it is never mentioned in WORKFLOW-CONTRACT.md. This is a critical omission. Every workflow author needs to understand versioning, but the contract does not require it.

**What research is needed:**
1. Add Temporal workflow versioning rules to WORKFLOW-CONTRACT.md.
2. Define a testing pattern: "deploy workflow v2 while workflow v1 instances are running, verify v1 instances complete correctly."
3. Add a lint rule or code review checklist item: "any workflow code change must include a GetVersion guard."

**Severity:** High

---

## Gap 19: No Feature Flag Infrastructure

**Where found:** CI-CD-PIPELINE.md (Section 6)

**The assumption being made:** That feature flags stored in PostgreSQL and evaluated per-tenant are simple to implement and maintain.

**Why it's risky:** CI-CD-PIPELINE.md shows a `featureflag.Enabled("llm-cost-prediction", tenant.ID)` example, but no feature flag infrastructure exists in the domain model, implementation plan, or any other document. There is no:

- Feature flag schema in the database
- API for managing feature flags
- UI for toggling flags
- Audit trail for flag changes
- Flag cleanup process (removing flags after features GA)
- Interaction model between flags and the LLM (does the LLM know which features are enabled?)

This is mentioned casually as if it were trivial, but a production-grade feature flag system is non-trivial, especially with per-tenant scoping.

**What research is needed:**
1. Should LOOM build its own feature flag system or integrate an open-source one (OpenFeature, Flipt, Unleash)?
2. If built-in, what is the schema? When in the implementation plan is it built?
3. How do feature flags interact with Temporal workflows? (A workflow started with flag X enabled that runs for 2 hours -- does it re-evaluate the flag mid-execution?)

**Severity:** Low

---

## Gap 20: Cost Model Data Source Bootstrapping

**Where found:** COST-MODEL.md, ADVERSARIAL-REVIEW.md Section 1.3

**The assumption being made:** That cost profiles can be populated with real data.

**Why it's risky:** COST-MODEL.md (which was written in response to the adversarial review) defines a thorough cost taxonomy with CostDimension, CostProfile, CostLineItem types. But it acknowledges that bare metal costs (power, cooling, rack space, depreciation) are operator-provided estimates, not measured values.

The gap is that the cost model requires manual data entry for on-premises infrastructure, but no data entry workflow exists. An operator managing 500 bare metal servers is not going to manually enter power draw, cooling overhead, rack rental cost, and depreciation schedule for each one. The cost model will be empty for on-prem unless there is:
- Bulk import capability
- Integration with DCIM tools (NetBox, Device42) that already track this data
- Sensible defaults ("if you don't know your PUE, use 1.5")

**What research is needed:**
1. Define a cost data import format (CSV, JSON) for bulk loading on-prem cost profiles.
2. Identify common DCIM tools that track cost-relevant data and design import adapters.
3. Define sensible cost defaults per device type so the cost model is useful even with incomplete data.

**Severity:** Low

---

## Gap 21: NATS Subject Hierarchy and Tenant ID in Subjects

**Where found:** EVENT-MODEL.md, ADR-005

**The assumption being made:** That `loom.{tenant_id}.{domain}.{action}` is a good subject hierarchy.

**Why it's risky:** Putting tenant_id as the second segment of the NATS subject means:

- Subscribing to all events for a domain across tenants requires wildcard: `loom.*.device.>`. This works but exposes cross-tenant data to any consumer with this subscription.
- NATS authorization must restrict subscriptions per tenant. A consumer for Tenant A must be prevented from subscribing to `loom.tenant-b.>`. This requires NATS account/user-level authorization that is not specified in any document.
- If tenant IDs are UUIDs, subjects become very long: `loom.550e8400-e29b-41d4-a716-446655440000.device.created`. NATS handles this, but it affects readability in debugging tools.
- The `_system` reserved tenant ID for system events is an edge case in every consumer.

**What research is needed:**
1. Design the NATS authorization model: one NATS account per tenant, or shared accounts with subject-level ACLs?
2. Should tenant_id be a short alias (e.g., `tenant-a`) rather than a UUID in subjects, with the UUID in the event payload?
3. Verify that NATS JetStream subject filtering performs well with thousands of distinct tenant IDs in the subject hierarchy.

**Severity:** Low

---

## Gap 22: Apache AGE and pgvector Extension Maturity

**Where found:** TECH-STACK.md, ADR-004

**The assumption being made:** That all PostgreSQL extensions (TimescaleDB, AGE, pgvector) coexist peacefully in one instance.

**Why it's risky:**
- Apache AGE is relatively new (graduated from Apache Incubator in 2023). Its production readiness at scale is less proven than TimescaleDB or pgvector.
- Running 4 extensions (TimescaleDB, AGE, pgvector, plus core PostgreSQL) in one instance creates upgrade complexity. Each extension has its own release cycle. A PostgreSQL major version upgrade (16 -> 17) requires all 4 extensions to support the new version simultaneously.
- No document specifies which PostgreSQL version is targeted, which extension versions are required, or how extension upgrades are managed.

**What research is needed:**
1. Test all 4 extensions running simultaneously on PostgreSQL 16. Verify no conflicts in shared memory, WAL handling, or vacuum behavior.
2. Define the extension version matrix: which versions of TimescaleDB, AGE, and pgvector are compatible with which PostgreSQL version.
3. Plan the PostgreSQL major version upgrade procedure with all extensions.

**Severity:** Low

---

## Gap 23: WebSocket Contract for Real-Time Operations

**Where found:** TECH-STACK.md, DEPLOYMENT-TOPOLOGY.md

**The assumption being made:** That SSE is sufficient for real-time updates and WebSocket is only needed for "bidirectional ops."

**Why it's risky:** The tech stack says "SSE (primary) + WebSocket" but the WebSocket contract is never defined. Specific questions:
- What is the WebSocket endpoint? (`/ws/v1/events`?)
- How does WebSocket authentication work? (JWT in query param? Upgrade header? Cookie?)
- What is the message format? (JSON? Protobuf? CBOR?)
- How does the server handle thousands of concurrent WebSocket connections?
- What is the reconnection strategy? (exponential backoff? resumption token?)
- Is WebSocket used for device console access (terminal-over-websocket)?

**What research is needed:**
1. Define whether WebSocket is needed for MVP or if SSE is sufficient.
2. If needed, define the WebSocket contract: endpoint, auth, message format, reconnection.
3. Estimate WebSocket connection capacity per LOOM API server instance.

**Severity:** Low

---

## Gap 24: Shared Device Ownership Model

**Where found:** EDGE-CASES.md (RC-04), DEVICE-LOCKING.md, DOMAIN-MODEL.md

**The assumption being made:** That every device belongs to exactly one tenant.

**Why it's risky:** Physical switches, hypervisors, and storage arrays serve multiple tenants. The current model forces a device to have a single `TenantID`. EDGE-CASES.md recommends a virtual device projection model but it is not incorporated into the domain model. Until this is resolved:
- A shared switch cannot exist cleanly in the data model
- Tenant isolation is violated if Tenant A can see the physical device that Tenant B also uses
- Device locking (DEVICE-LOCKING.md) uses device_id for locking, but a virtual device model would need locking on the physical device, not the per-tenant virtual device

**What research is needed:**
1. Design the shared device model. Options: (a) PhysicalDevice + TenantDeviceProjection, (b) Device with OwnerTenantID + AuthorizedTenantIDs list, (c) Separate "infrastructure" tenant that owns shared devices.
2. Update the domain model to reflect the chosen approach.
3. Update DEVICE-LOCKING.md to handle locks on shared physical devices.

**Severity:** Medium

---

## Gap 25: No Chaos Engineering or Resilience Testing Strategy

**Where found:** TESTING-STRATEGY.md, FAILURE-MODES.md

**The assumption being made:** That the failure mode catalog (which is prescriptive and thorough) will be validated by the existing test categories.

**Why it's risky:** FAILURE-MODES.md says "Every behavior described here must be implemented and tested. If LOOM does not behave as described during a failure, that is a bug." But TESTING-STRATEGY.md does not include chaos testing. There is no:
- Chaos test framework (kill PostgreSQL mid-workflow, partition NATS, crash Temporal mid-activity)
- Failure injection in CI (even a simple "restart PostgreSQL during integration tests")
- Game day procedure (scheduled resilience testing in pre-production)

**What research is needed:**
1. Should LOOM adopt a chaos testing framework (Toxiproxy for network faults, custom Docker healthcheck manipulation for component kills)?
2. Which failure modes from FAILURE-MODES.md can be tested automatically in CI? Which require manual game days?
3. Can the E2E test suite be extended with a "chaos mode" that randomly kills components during test execution?

**Severity:** Medium

---

## Gap 26: Concurrent Discovery Race Conditions

**Where found:** DISCOVERY-CONTRACT.md, IDENTITY-MODEL.md, EDGE-CASES.md (RC-02)

**The assumption being made:** That discovery is safe to run concurrently from multiple sources.

**Why it's risky:** If two discovery workflows scan the same subnet simultaneously (e.g., a scheduled scan and an operator-triggered scan), they may both discover the same device and attempt to create it. The identity matcher handles merge/enrich for sequential discoveries, but concurrent discoveries create a race condition:
1. Discovery A probes 10.0.0.5, finds it is new, starts creating Device record
2. Discovery B probes 10.0.0.5, finds it is new (A hasn't committed yet), starts creating Device record
3. Both commit: duplicate device created

The identity matcher's merge rules assume sequential processing. No document addresses the concurrency control for the identity matching pipeline itself.

**What research is needed:**
1. Should the identity matcher use database-level uniqueness constraints (unique index on serial_number per tenant) to prevent duplicates, with a retry-and-merge loop on conflict?
2. Should discovery workflows acquire a per-subnet lock to prevent concurrent scans?
3. What is the expected concurrency model: one discovery workflow per tenant at a time, or parallel scanning with identity-level locking?

**Severity:** Medium

---

## Gap 27: No API Rate Limiting Per Operation Type

**Where found:** API-CONVENTIONS.md, IMPLEMENTATION-PLAN.md Phase 5

**The assumption being made:** That per-tenant rate limiting is sufficient.

**Why it's risky:** API-CONVENTIONS.md defines per-tenant rate limiting backed by Valkey. But all operations count equally. A tenant making 1000 read requests per minute and a tenant making 1000 discovery requests per minute have very different system impact. Discovery requests trigger adapter connections, NATS events, and potentially LLM calls. Read requests hit Valkey cache.

Missing:
- No rate limiting by operation type (reads vs. writes vs. discovery vs. workflow submission)
- No concurrency limiting (max concurrent workflows per tenant, max concurrent adapter connections per tenant)
- No cost-weighted rate limiting (an LLM-heavy operation "costs" more than a cache read)

**What research is needed:**
1. Define rate limit tiers: read operations (high limit), write operations (medium limit), discovery/workflow operations (low limit).
2. Define concurrency limits: max concurrent workflows per tenant, max concurrent adapter connections per tenant.
3. Should rate limiting be configurable per tenant (premium tenants get higher limits)?

**Severity:** Low

---

## Priority Summary

| Severity | Count | Gaps |
|----------|-------|------|
| **Critical** | 1 | #1 (Adapter SDK/DX) |
| **High** | 7 | #2 (Plugin Lifecycle), #3 (API Completeness), #5 (LLM Practicalities), #6 (Multi-Tenancy), #7 (Testing 25+ Adapters), #12 (Documentation/Onboarding), #13 (Backward Compat), #18 (Temporal Versioning) |
| **Medium** | 10 | #4 (Observability Debugging), #8 (CQRS Implications), #9 (UI Architecture), #10 (CI/CD), #11 (LLM Boundary Pressure), #14 (Telemetry Schema), #15 (AGE Usage), #16 (Credential Rotation), #17 (DR), #24 (Shared Devices), #25 (Chaos Testing), #26 (Discovery Races) |
| **Low** | 6 | #19 (Feature Flags), #20 (Cost Data Bootstrap), #21 (NATS Subjects), #22 (Extension Maturity), #23 (WebSocket Contract), #27 (Rate Limiting) |

---

## Closing Assessment

LOOM's Phase 0 documentation is among the most thorough I have seen for a project that has not yet written production code. The adversarial review, edge case catalog, and failure mode catalog demonstrate genuine architectural rigor. The decision to freeze contracts before writing code is correct and rare.

The primary risk is not architectural -- it is adoption. LOOM's value proposition requires 25+ adapters and a community of adapter authors. Without an SDK, scaffolding tools, a conformance test suite, and documentation that speaks to developers (not just architects), the adapter ecosystem will not materialize. The second-order risk is operational: the system is designed for operators, but the operator debugging experience (runbooks, admin tools, troubleshooting guides) is absent.

The recommendation is: before Phase 1A coding begins, invest in the adapter SDK design (Gap 1), the API route table (Gap 3), and the quickstart guide (Gap 12). These three deliverables de-risk the most critical adoption bottlenecks and can be done in parallel with the implementation plan.
