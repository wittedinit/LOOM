# LOOM Adversarial Architecture Review

> Written by a hostile AI architect whose job is to find every conceptual flaw, implementation impossibility, hand-wave, and gap in this plan. If the plan survives this review intact, it deserves to be built. It will not survive intact.

**Documents reviewed:** All 22 docs in `/docs/`, 11 ADRs, 8 attack surface analyses, `README.md`.

**Verdict:** The contracts and domain model are unusually well-defined for a project at this stage. The identity model and error taxonomy are genuinely strong. But the plan has serious structural problems in seven categories. Several claims are borderline impossible without enormous scope reduction. The competitive positioning overpromises relative to what the implementation plan can deliver in any reasonable timeline.

---

## 1. Architectural Impossibilities

### 1.1 Transactional Rollback Across Heterogeneous Infrastructure

**Gap:** WORKFLOW-CONTRACT.md claims saga-pattern compensation across Terraform + Ansible + network switch configs + firewall rules + OS installs. ADAPTER-CONTRACT.md defines `CompensationInfo` per operation. But this assumes every target system supports idempotent undo -- and many do not.

**What actually breaks:**
- MikroTik RouterOS REST API has no transactional semantics. If you push a firewall rule and then need to roll back, you must identify and delete the specific rule by its `.id` field, which is assigned server-side and unpredictable. If the original push partially applied (e.g., 3 of 5 rules), you have no atomic way to know which 3.
- IPMI has no "undo." `PowerCycleOp` compensation says `"off"` compensates `"on"` -- but what if the server was already on before the workflow started? The compensation puts it in a *different* state than the pre-workflow state. The `PreviousState` field in `PowerCycleResult` captures this, but the compensation operation type is hardcoded to the reverse action, not "restore previous state."
- SSH `ExecuteCommandOp` is explicitly marked `CompensationNone` -- but provisioning workflows use SSH for post-install configuration (`ConfigureHost` step). If step 6 fails, the compensation for step 5 (`RevertHostConfig`) requires SSH commands to undo other SSH commands. That is a custom undo script per operation, which is not captured in any typed operation.
- Cisco IOS-XE NETCONF supports candidate/running config separation, but Arista eAPI does not have a native rollback-to-checkpoint mechanism. EOS `configure session` is the closest, but it is not used in the adapter translation examples in REQUEST-TO-DEVICE-FLOW.md.

**Impact:** Compensation will silently leave infrastructure in inconsistent states. The "saga compensates in LIFO order" claim becomes "saga *attempts* compensation and logs failures" -- which is monitoring, not rollback.

**Recommendation:**
1. Add a `PreOpSnapshot` field to `OperationResult` that captures the exact state before the operation. Compensation restores this snapshot, not a generic reverse.
2. For every adapter, document explicitly which operations are truly compensatable and which are "best-effort undo." Add a `CompensationReliability` enum: `guaranteed`, `best_effort`, `manual_required`.
3. Add a `CompensationFailureStrategy` to the workflow contract: what happens when compensation itself fails? Currently WORKFLOW-CONTRACT.md says "logged and alerted but do not block other compensations" -- this needs to be elevated to a first-class workflow state (`compensation_partial`) with operator remediation procedures.

---

### 1.2 LLM Config Generation Without Hallucination Causing Outages

**Gap:** ARCHITECTURE.md claims the LLM generates "vendor-specific config from universal intent." LLM-BOUNDARIES.md has a hallucination prevention pipeline (JSON schema -> syntax check -> Batfish verification -> dry-run -> human approval -> live execution). This pipeline is sound in theory but has critical gaps.

**What actually breaks:**
- **Batfish does not support MikroTik RouterOS, OPNsense, SONiC, Cumulus NVUE, or UniFi.** Batfish supports Cisco IOS/NX-OS, Junos, Arista EOS, and a few others. For the majority of LOOM's target device list, "Batfish-style offline verification" is marketing. You would need to build a config simulator for each unsupported NOS.
- The confidence threshold for config generation is 0.9 in production (LLM-BOUNDARIES.md). But confidence is self-reported by the LLM. An LLM that confidently generates a syntactically valid but semantically wrong ACL rule (e.g., `permit` instead of `deny`) will pass JSON schema validation, pass syntax checking, and score 0.95 confidence. The hallucination is in the *logic*, not the *syntax*.
- The fallback is "template-based generation" -- but templates must exist for every vendor/model/firmware combination. For a project that targets 15+ NOS dialects, maintaining templates is the same amount of work as building the adapters themselves. The fallback is not "free."

**Impact:** Without offline semantic verification for most target platforms, the LLM config path is either: (a) restricted to the 4-5 vendors Batfish supports, or (b) dependent on human review for every config change, which eliminates the automation benefit.

**Recommendation:**
1. Be honest about Batfish coverage in the docs. List which vendors have offline verification and which do not.
2. For vendors without Batfish support, require a "config diff review" step that shows the operator exactly what will change, rather than the full config. This is achievable and doesn't require Batfish.
3. Add a `VerificationDepth` enum to each adapter: `offline_verified` (Batfish), `syntax_verified` (parser only), `schema_verified` (JSON schema only), `unverified` (human approval mandatory). This makes the trust level explicit per vendor.

---

### 1.3 Real-Time Cost Optimization

**Gap:** ARCHITECTURE.md and COMPETITIVE-ANALYSIS.md claim "cost as a first-class scheduling dimension" and "continuous reasoning about tradeoffs." TECH-STACK.md mentions "cost analytics" with ECharts. IMPLEMENTATION-PLAN.md Phase 8 is "Cost Engine + Lifecycle Management."

**What actually breaks:**
- Cloud pricing APIs (AWS, GCP, Azure) have 24-48 hour lag for actual billing data. Spot prices change every 5 minutes but the actual cost of a running instance is not known until the billing API catches up.
- Bare metal cost calculation requires: power draw (requires PDU integration or IPMI power sensor data), cooling overhead (a facility-level metric, not per-server), depreciation schedule (an accounting construct, not a technical one), rack rental costs (a contract term, not an API call), and network transit costs (depends on ISP billing model). None of these have standard APIs. IMPLEMENTATION-PLAN.md Phase 8 says "cost model (bare metal amortization, VM fractional, cloud pricing)" but does not explain where any of these numbers come from.
- The LLM reasoning example in ARCHITECTURE.md mentions "spot prices have been stable for 72 hours" -- but this requires historical spot price tracking, which means building or integrating a spot price history service. This is not mentioned anywhere in the implementation plan.

**Impact:** The "cost-aware placement" feature will, in practice, be limited to: cloud list prices (not actual spend), user-provided bare metal cost estimates (not real-time data), and best-guess amortization. This is useful but far from the "first-class decision dimension" the docs promise.

**Recommendation:**
1. Define explicit cost data sources in a new `COST-MODEL.md` doc: what data is available in real-time vs. estimated vs. user-provided.
2. Separate "cost estimation at placement time" (feasible with list prices) from "cost tracking over time" (requires billing API integration). These are different features with different accuracy guarantees.
3. For bare metal costs, require operators to configure cost-per-server-per-hour as a static input. Do not pretend this can be calculated automatically.

---

### 1.4 The "Single Pane of Glass"

**Gap:** README.md and COMPETITIVE-ANALYSIS.md claim LOOM provides a "single pane of glass" into "health, cost, and performance across your entire estate." IMPLEMENTATION-PLAN.md Phase 6 is a "thin UI" with device inventory, workflow list, and approvals. Phase 9 adds topology visualization.

**What actually breaks:**
- A single pane of glass for heterogeneous infrastructure requires real-time state from every managed device. That means continuous polling or streaming telemetry from 10K+ devices. TECH-STACK.md mentions gNMI Watcher for streaming, but only 3 adapters implement Watcher (gNMI, vSphere, libvirt). The other 10+ adapters require polling.
- Polling 10K devices at 1-minute intervals means 10K adapter calls per minute. Each call requires credential retrieval (vault round-trip), connection setup, data collection, and state comparison. At 30 seconds per call (generous for SNMP, tight for SSH), you need 5,000 concurrent connections sustained indefinitely.
- The UI (Phase 6) is a "thin" React app. The "single pane of glass" requires Cytoscape.js topology (Phase 9), ECharts cost analytics (Phase 8), React Flow workflow visualization (not scheduled), and real-time telemetry dashboards. The full vision requires Phases 6 + 8 + 9 + additional work.

**Impact:** The "single pane of glass" will, at best, be a device inventory table with workflow status -- which is NetBox + Temporal UI, not a paradigm shift. The full vision is 6-12 months past Phase 6.

**Recommendation:**
1. Stop calling it a "single pane of glass" until Phase 9 is complete. Call Phase 6 what it is: an operator console.
2. Define the telemetry ingestion architecture for continuous monitoring. How many devices can be polled per worker? What is the concurrency model? This is absent from the docs.
3. Consider whether Grafana embedding (mentioned in TECH-STACK.md as "optional") should be the primary dashboard for telemetry, with LOOM's UI focused on orchestration and workflows.

---

## 2. Missing "How" for Every "What"

### 2.1 Protocol Auto-Detection

**Gap:** ARCHITECTURE.md says adapters "auto-detect" whether a device uses Redfish, IPMI, AMT, or PiKVM. REQUEST-TO-DEVICE-FLOW.md says "auto-detected per device." The `Target` struct in DOMAIN-MODEL.md has a `ProtocolHint` field that is "optional."

**How does auto-detection actually work?** The docs never say. Options:
- Port scanning (probe 443 for Redfish, 623 for IPMI, 16992 for AMT, 8080 for PiKVM) -- but many devices expose multiple protocols on non-standard ports.
- Protocol-specific handshake (try Redfish `/redfish/v1`, IPMI `Get Channel Auth`, AMT SOAP discovery) -- requires attempting connections with potentially wrong credentials.
- SNMP sysObjectID fingerprinting -- requires SNMP access, which requires community strings or SNMPv3 credentials you might not have yet.

**Impact:** Without a defined auto-detection algorithm, the "auto-detect" claim is a TODO. In practice, operators will need to provide `ProtocolHint` for most devices, which makes auto-detection a nice-to-have, not a core feature.

**Recommendation:** Write a `PROTOCOL-DETECTION.md` doc that specifies the exact detection algorithm: which ports are probed, in what order, with what timeouts, what constitutes a positive identification, and what happens when detection is ambiguous. IMPLEMENTATION-PLAN.md Phase 3 mentions a "subnet scanner (port probe -> detect available protocols)" -- this needs to be fleshed out into a contract.

---

### 2.2 Multi-Vendor Network Config Translation

**Gap:** REQUEST-TO-DEVICE-FLOW.md shows the same VLAN creation translated to SONiC (gNMI/OpenConfig), Cisco IOS-XE (NETCONF/YANG), Arista EOS (eAPI), and MikroTik (REST). This looks clean for VLAN creation. But VLANs are the *simplest* network operation.

**Where it falls apart:**
- **ACLs** have fundamentally different semantics across vendors. Cisco uses numbered/named ACLs bound to interfaces. Junos uses firewall filters with terms. Arista uses standard/extended ACLs *or* traffic policies. SONiC uses ACL tables mapped to ports. A "universal ACL model" that maps cleanly to all four is a research project, not an engineering task.
- **Routing protocols** (BGP, OSPF) have vendor-specific knobs that differ in ways that matter for production. Cisco BGP uses `address-family` submode; Junos uses `routing-options` and `protocols bgp` as separate hierarchies. A universal model either loses vendor-specific features or becomes a vendor-specific superset -- which defeats the purpose.
- **QoS/traffic shaping** is so vendor-specific that even NAPALM (which LOOM references as insufficient) punted on it entirely.

**Impact:** Phase 7 is scoped as "XL" but the docs underestimate the difficulty. VLAN + port + basic ACL translation is achievable. Routing, QoS, and advanced features are multi-year projects per vendor.

**Recommendation:**
1. Scope Phase 7 explicitly: "VLAN, interface, static route, and simple ACL translation for SONiC, EOS, IOS-XE, and Junos." Document what is *not* in scope.
2. For advanced features (BGP, QoS, policy-based routing), adopt a pass-through model: the operator provides vendor-native config, and LOOM pushes it without translation. This is honest and useful.
3. Add a `TranslationCoverage` matrix to each network adapter: which operations have universal model support and which require native config.

---

### 2.3 LLM Conflict Resolution

**Gap:** ARCHITECTURE.md says the LLM "reasons across multiple dimensions simultaneously" -- cost, availability, reachability, capacity, compliance, performance. But what happens when these dimensions conflict?

- Cheapest placement is in a single AZ (no redundancy).
- Best availability requires 3 AZs (expensive).
- Compliance requires EU data residency (eliminates all US AZs).
- Lowest latency requires same rack (single failure domain).
- Tenant isolation requires dedicated hosts (high cost).

**How does the LLM decide?** The docs never define a priority order for dimensions, a weighting scheme, or a conflict resolution strategy. LLM-BOUNDARIES.md says every LLM recommendation has a `Confidence` score and `Fallback` -- but the fallback is "rule-based placement (capacity + affinity)" which also has the same multi-dimensional conflict problem.

**Impact:** Without explicit constraint priorities, the LLM's decisions will be unpredictable and non-reproducible. Two identical requests might get different placements depending on the LLM's mood (temperature setting). Operators will not trust the system.

**Recommendation:**
1. Define a constraint priority order in a `PLACEMENT-POLICY.md` doc. Example: compliance (hard constraint, must satisfy) > availability (SLA-based, configurable) > cost (soft constraint, optimize) > performance (best-effort).
2. Hard constraints must be enforced deterministically *before* the LLM runs. The LLM optimizes within the feasible space, not across constraint boundaries.
3. LLM placement decisions must be deterministic given the same inputs. This means fixing the temperature to 0 for placement decisions and caching results.

---

### 2.4 Edge Agent Autonomy During Partitions

**Gap:** DEPLOYMENT-TOPOLOGY.md Mode 3 describes edge agents with SQLite + NATS leaf nodes that "operate autonomously during network partitions." IMPLEMENTATION-PLAN.md Phase 11 says "edge agent buffers when disconnected, syncs when reconnected."

**What autonomous operations can an edge agent actually perform?** The docs never say.
- Can it execute workflows? It has no Temporal client.
- Can it make LLM decisions? It may not have a local LLM.
- Can it push config to devices? With what authorization? Cached credentials?
- Can it handle conflicts when the same device is managed by the hub and the edge simultaneously?
- How does credential rotation work if the edge is partitioned for weeks?

**Impact:** Without defining the scope of autonomous operation, the edge agent is either: (a) a dumb telemetry buffer (useful but not "autonomous"), or (b) a full LOOM instance (which means running PostgreSQL + Temporal locally, defeating the "lightweight" claim).

**Recommendation:**
1. Define exactly which operations the edge agent can perform autonomously: discovery (yes), telemetry collection (yes), workflow execution (no -- requires Temporal), config push (only pre-approved/cached operations).
2. Define the conflict resolution strategy for split-brain scenarios: hub takes precedence, edge changes are flagged for review on reconnection.
3. Define credential caching policy: how long are cached credentials valid? What happens when they expire during a partition?

---

## 3. Scale Assumptions That Don't Hold

### 3.1 PostgreSQL + TimescaleDB for 10K Device Telemetry

**Gap:** TECH-STACK.md claims PostgreSQL + TimescaleDB handles telemetry. DEPLOYMENT-TOPOLOGY.md Mode 2 targets "100-10,000 devices." The scale-out path says "add VictoriaMetrics when telemetry exceeds 500K datapoints/sec."

**The math:**
- 10K devices, each reporting 20 metrics (CPU, memory, disk, network in/out, temperatures, power), at 1-minute intervals = 200K inserts/minute = 3,333 inserts/second.
- Each insert is ~200 bytes (timestamp + device_id + metric_name + value + tags). That is 667 KB/s sustained write throughput.
- TimescaleDB on a single node with good I/O can handle 100K-500K inserts/second. So 3.3K/s is fine for raw inserts.
- But: continuous aggregates, retention policies, and read queries for dashboards all compete for I/O. A query like "show me the 95th percentile CPU across all devices in rack A3 for the last 7 days" on 10K devices is non-trivial.
- The real concern: TimescaleDB shares the PostgreSQL instance with device inventory, workflow projections, audit records, topology (AGE), and embeddings (pgvector). All of these compete for connection pool slots (default 25 per FAILURE-MODES.md), buffer cache, and WAL bandwidth.

**Impact:** At 10K devices, a single PostgreSQL instance will struggle -- not because TimescaleDB can't handle the inserts, but because the combined workload (OLTP inventory + time-series telemetry + graph queries + vector search) creates I/O and memory contention. The "single backup/restore pipeline" advantage becomes a "single point of failure for all workloads" disadvantage.

**Recommendation:**
1. Be explicit about the break-even point. At what device count does the single-PostgreSQL model stop working? My estimate: ~2,000 devices with active telemetry. Above that, separate the telemetry workload.
2. Define the telemetry schema now (it is absent from the docs -- no table definitions for metrics/telemetry appear anywhere). Without a schema, you cannot estimate storage or query performance.
3. Move TimescaleDB to a separate PostgreSQL instance at Mode 2 (production) deployments. The "single engine" argument does not justify running OLTP and time-series on the same instance in production.

---

### 3.2 Temporal at 1000+ Concurrent Workflows

**Gap:** The docs target environments with 10K devices. A large provisioning request could generate hundreds of concurrent workflows (one per device or per group). Drift detection at 1-minute intervals on 10K devices could generate 10K workflow executions per minute if each drift check is a workflow.

**What actually constrains Temporal:**
- Temporal Server uses PostgreSQL for persistence. Every activity start, heartbeat, and completion is a DB write. At 1000 concurrent workflows with 5 activities each, that is 5000+ DB round-trips per workflow lifecycle, multiplied by 1000 = 5M DB operations for the batch.
- Temporal's history service is the bottleneck in most deployments. Each shard handles a subset of workflows. Default shard count is 4. At 1000 concurrent workflows, each shard handles 250 -- manageable but not trivial.
- WORKFLOW-CONTRACT.md rule 9 requires `continue-as-new` at ~10K events. A discovery workflow scanning 10K devices will hit this quickly if each device scan is an activity (10K activities = 20K+ events including starts and completions).

**Impact:** Temporal will work at moderate scale (100-500 concurrent workflows). At 1000+ concurrent workflows, you need Temporal cluster tuning (shard count, DB connection pool, matching service partitions) that is not mentioned in the docs.

**Recommendation:**
1. Add a `TEMPORAL-SIZING.md` doc that covers: expected workflow volume per deployment size, recommended shard count, DB connection pool sizing for Temporal's PostgreSQL, and worker concurrency limits.
2. Not every operation needs to be a Temporal workflow. Drift checks should be lightweight polling loops, not full workflow executions. Reserve Temporal for multi-step operations with compensation requirements.
3. Define a batching strategy: instead of one workflow per device for discovery, use one workflow per subnet or per rack with child workflows.

---

### 3.3 NATS JetStream Event Volume

**Gap:** EVENT-MODEL.md configures the LOOM_EVENTS stream with 50 GB max, 90-day retention, 3 replicas. Every mutation produces an event. Every audit record produces an event. Every telemetry data point could produce an event.

**The math at 10K devices:**
- Device state events: 10K devices * 1 update/minute = 10K events/minute
- Telemetry events: 10K devices * 20 metrics * 1/minute = 200K events/minute (if each metric is a NATS event)
- Audit events: every workflow step, every adapter call, every credential access = potentially 100K events/minute during provisioning bursts
- Total: 300K+ events/minute sustained = 5K events/second

Each event is ~1 KB (JSON envelope). That is 5 MB/s write throughput, replicated 3x = 15 MB/s I/O. Over 90 days, that is ~40 TB of data -- far exceeding the 50 GB limit.

**Impact:** The 50 GB MaxBytes will be exhausted in hours at scale. The DiscardOld policy will silently drop events. The DB projector consumer must keep up or lose events permanently.

**Recommendation:**
1. Do not publish individual telemetry data points as NATS events. Telemetry should be written directly to TimescaleDB (or VictoriaMetrics). NATS events should be reserved for state changes (device status transitions, workflow lifecycle, audit records).
2. Recalculate the MaxBytes and retention based on actual event volume (state changes only). 50 GB may be sufficient for state change events at 10K devices, but the docs should show the math.
3. Add a `loom admin events stats` command that reports current event volume, storage utilization, and projected time to limit.

---

## 4. Integration Fantasies

### 4.1 Seamless LLM Failover (Cloud to Local)

**Gap:** TECH-STACK.md describes automatic failover from cloud LLM to local model. "Cloud unavailable -> local model." LLM-BOUNDARIES.md says every path works without LLM (deterministic fallback).

**What actually happens during failover:**
- Claude Opus generates a sophisticated multi-dimensional placement decision with 0.92 confidence.
- Cloud goes down. Failover to local Llama 4 8B (4-bit quantized, running on CPU).
- The local 8B model generates a syntactically valid but strategically inferior placement decision. Or it generates garbage because the prompt was tuned for a 200B+ parameter model.
- The confidence threshold is 0.8 for placement (LLM-BOUNDARIES.md). The local model may report 0.85 confidence on a bad decision because confidence calibration is model-dependent.

**Impact:** "Seamless failover" is misleading. The quality of decisions will drop dramatically when falling back to a local 7-8B model. If the system makes bad decisions silently, operators lose trust.

**Recommendation:**
1. Define quality tiers explicitly: "Tier 1: cloud model (full reasoning), Tier 2: local large model (70B+, reduced reasoning), Tier 3: local small model (7-14B, basic classification only), Tier 4: deterministic fallback (no LLM)."
2. When failover occurs, log it prominently and degrade functionality gracefully. Do not let a 7B model make production placement decisions that a 200B model was trusted for.
3. Add model-specific confidence calibration. A 0.8 from Claude Opus is not the same as a 0.8 from Llama 8B. Adjust thresholds per model.

---

### 4.2 Air-Gapped Deployment with Full Feature Parity

**Gap:** TECH-STACK.md claims "Air-gapped cost: $0/day" and "Zero external network calls." DEPLOYMENT-TOPOLOGY.md Mode 1 runs everything locally. README.md says "No internet required. No API keys required."

**What you lose in air-gapped mode:**
- No cloud pricing APIs (cost optimization is limited to static estimates).
- No cloud LLM APIs (limited to local model quality -- see 4.1).
- No pgvector embeddings for vendor documentation RAG (unless you pre-embed all docs before going air-gapped).
- No software updates, no CVE patches, no model updates.
- No remote attestation for security compliance.

**Impact:** "Full feature parity" is false. Air-gapped mode is a different product with significantly reduced intelligence. This is fine -- many customers need air-gapped operation -- but the docs should be honest about what is lost.

**Recommendation:** Create a `FEATURE-MATRIX.md` that explicitly lists which features are available in each deployment mode: cloud-connected, hybrid, and air-gapped. Mark each feature as "full," "degraded," or "unavailable" per mode.

---

### 4.3 Multi-Vendor ACL Semantic Equivalence

**Gap:** REQUEST-TO-DEVICE-FLOW.md shows firewall rules (web->app ALLOW, app->db ALLOW, web->db DENY) applied via OPNsense REST API. OPERATION-TYPES.md defines `SetACLOp` with a universal `ACLRule` struct. VERIFICATION-MODEL.md defines `ACLEnforcementCheck`.

**The fantasy:** A universal ACL rule translates cleanly to every vendor's ACL implementation.

**The reality:**
- Cisco IOS ACLs are sequence-numbered and order-dependent. Junos firewall filters use "terms" that are named and order-independent within a filter. The same intent requires different structural representations.
- Stateful vs. stateless: OPNsense uses pf (stateful firewall). Cisco ACLs are stateless by default (stateful requires ZBF). A "permit TCP 8080" rule on a stateful firewall allows return traffic automatically; on a stateless ACL, you need an explicit return rule.
- The `ACLRule` struct in OPERATION-TYPES.md has `Action`, `Protocol`, `Source`, `Dest`, `Port`. It is missing: state tracking, connection tracking, logging, rate limiting, time-based rules, object groups, NAT integration -- all of which vary by vendor.

**Impact:** The universal ACL model will work for trivial cases (permit/deny IP/port). Any real-world firewall policy requires vendor-specific extensions that break the universal model.

**Recommendation:**
1. Rename `SetACLOp` to `SetBasicACLOp` and document its limitations explicitly.
2. Add a `SetNativeACLOp` that accepts vendor-native config as a raw string/struct for complex rules. This is the escape hatch.
3. The verification step (`ACLEnforcementCheck`) is the right place to ensure correctness -- focus there rather than trying to make the universal model cover all cases.

---

## 5. Chicken-and-Egg Problems

### 5.1 LOOM Orchestrates Infrastructure, But Needs Infrastructure

**Gap:** LOOM requires PostgreSQL, Temporal (which requires PostgreSQL), NATS, and optionally Valkey. In Mode 2 (production), this is a Patroni cluster, a 3-node NATS cluster, and a 3-node Valkey cluster. Who provisions and manages these?

- If LOOM manages them, who manages LOOM while LOOM is bootstrapping?
- If they are managed manually, then LOOM's "single pane of glass" has a blind spot for its own infrastructure.
- If they are managed by Terraform/Ansible, then the operator needs Terraform/Ansible expertise *in addition to* LOOM -- which undermines the "orchestrator of orchestrators" value prop.

**Impact:** LOOM's own infrastructure is the most critical infrastructure in the stack, and it is unmanaged by LOOM. This is a philosophical gap that customers will immediately ask about.

**Recommendation:**
1. Document the bootstrap procedure explicitly in a `BOOTSTRAP.md` doc. "Before LOOM can run, you need: PostgreSQL (here's how to set it up), NATS (here's the config), Temporal (here's how to deploy it)."
2. Consider providing a `loom bootstrap` command that deploys LOOM's own dependencies using Docker Compose or Helm, so the operator does not need separate tooling.
3. Add LOOM's own infrastructure to its monitoring scope *after* bootstrap. LOOM should monitor its own PostgreSQL, NATS, and Temporal health and alert on degradation.

---

### 5.2 Credential Bootstrap Paradox

**Gap:** LOOM stores credentials in an AES-256-GCM encrypted vault with 3-tier envelope encryption (VAULT-ARCHITECTURE.md). The MEK is in HSM/TPM. The KEK is per-tenant. The DEK is per-credential. But:

- To connect to PostgreSQL (where credentials are stored), LOOM needs a database password.
- To connect to NATS, LOOM needs NKey credentials.
- To connect to Temporal, LOOM needs connection credentials.
- These bootstrap credentials cannot be stored in LOOM's vault because the vault is *in* PostgreSQL.

**Where do the bootstrap credentials live?**
- Environment variables? Then they are in process memory unencrypted, visible in `/proc/<pid>/environ`, and logged by container runtimes.
- Config file? Then there is a plaintext secret on disk, violating the "zero plaintext at rest" invariant.
- External secret manager (Vault, AWS Secrets Manager)? Then LOOM depends on an external secret manager, complicating the air-gapped deployment story.

**Impact:** The security model has a bootstrap exception that is not documented. The most security-sensitive credentials (database access) are handled with the *least* security.

**Recommendation:**
1. Document the bootstrap credential chain explicitly. Accept that bootstrap credentials have a different security model than managed credentials.
2. Recommend mTLS for PostgreSQL and NATS connections (certificate-based auth, no passwords). Certificates can be provisioned by an init container or systemd service.
3. For the air-gapped case, use TPM-sealed storage for bootstrap credentials (the same TPM used for the MEK).

---

### 5.3 Adapter Deployment Before Device Discovery

**Gap:** LOOM discovers devices and then selects the appropriate adapter. But adapters must be compiled into the binary (Go static linking). If a customer has a device type that LOOM does not have a compiled adapter for, discovery will detect the device but cannot manage it.

- The docs list 13 adapter types. The implementation plan schedules SSH + Redfish in Phase 2, SNMP in Phase 3, IPMI in Phase 4, NETCONF/gNMI/eAPI/NX-API in Phase 7, and AMT/PiKVM/vSphere/Proxmox/libvirt in Phase 11.
- A customer deploying LOOM at Phase 4 cannot manage their Arista switches, Proxmox hypervisors, or PiKVM devices.

**Impact:** LOOM's value is proportional to its adapter coverage. Until Phase 11, LOOM manages a subset of infrastructure, and operators need other tools for the rest -- defeating the "single pane of glass" claim.

**Recommendation:**
1. This is inherent to the phased approach and is acceptable. But the docs should be honest: "At Phase N, LOOM supports the following adapters: [list]. For other devices, use existing tools and integrate via LOOM's API."
2. Prioritize adapters by customer demand, not by implementation convenience. If the target market has 80% Cisco switches, the NETCONF adapter should be Phase 3, not Phase 7.
3. Consider a plugin architecture (`HashiCorp go-plugin` is mentioned in TECH-STACK.md but never elaborated) to allow third-party adapters without recompilation.

---

## 6. Missing Error Paths

### 6.1 Device Returns Garbage

**Gap:** ERROR-MODEL.md defines `TransientError`, `PermanentError`, `PartialError`, and `ValidationError`. But there is no error type for "device responded but the response is nonsensical."

**Examples:**
- SNMP GET returns a sysDescr of `"\x00\x00\x00"` (corrupted MIB).
- Redfish returns `200 OK` with an empty JSON body (firmware bug).
- SSH session connects but the shell outputs binary garbage (wrong terminal encoding).
- NETCONF returns a valid XML response but the data is internally inconsistent (VLAN 100 exists in the VLAN database but no interface references it).

**Impact:** The adapter will either parse the garbage into wrong facts (silent data corruption) or crash (unhandled panic). Neither is acceptable.

**Recommendation:**
1. Add a `MalformedResponseError` to the error taxonomy: device responded but the response failed semantic validation.
2. Every adapter's `Discover` and `ReadState` methods must validate response data against expected schemas before returning facts. Invalid responses should be logged with the raw payload and discarded.
3. Add `ResponseValidation` as a step in the adapter contract: raw response -> schema validation -> semantic validation -> typed result.

---

### 6.2 LLM Is Confident But Wrong

**Gap:** LLM-BOUNDARIES.md says low confidence (< 0.7) triggers human review. But LLMs are most dangerous when confident and wrong.

**Scenario:** The LLM recommends placing a database workload on a spot instance (cost optimization). Confidence: 0.91. The recommendation passes all validation. The spot instance is terminated 2 hours later, taking the database offline. The LLM was confidently wrong because it did not understand that databases need stable compute.

**Impact:** The confidence threshold is a necessary but insufficient safeguard. It catches "I don't know" but not "I'm sure and I'm wrong."

**Recommendation:**
1. Add hard constraints that override LLM recommendations regardless of confidence: "databases never on spot instances," "production workloads never on preemptible compute," "stateful workloads never on ephemeral storage."
2. These constraints should be user-configurable policy rules, not embedded in LLM prompts. They are enforced by the deterministic validator *after* the LLM recommends and *before* execution.
3. Log every case where a hard constraint overrides an LLM recommendation. This data improves the LLM over time.

---

### 6.3 Two Tenants Conflict on Same Physical Device

**Gap:** DOMAIN-MODEL.md says "Cross-tenant access is forbidden." But physical devices can be shared: a network switch carries VLANs for multiple tenants. A hypervisor hosts VMs from multiple tenants.

**Scenario:** Tenant A requests VLAN 100 on switch SW-01. Tenant B requests VLAN 100 on the same switch SW-01. Or: Tenant A's workflow is reconfiguring SW-01's trunk ports while Tenant B's workflow is creating a VLAN on SW-01.

**Impact:** Without per-device locking, concurrent workflows from different tenants can produce race conditions on shared physical infrastructure. The adapter's `IdempotencyKey` prevents duplicate operations within a tenant, but does not prevent cross-tenant conflicts on the same device.

**Recommendation:**
1. Add a device-level distributed lock (Temporal workflow semaphore or PostgreSQL advisory lock per device). Only one workflow can mutate a specific physical device at a time, regardless of tenant.
2. For shared resources (VLAN IDs, IP ranges), add a resource allocation service that prevents conflicts. VLAN 100 on switch SW-01 must be globally unique, not per-tenant unique.
3. Document the shared-device model explicitly: which resources are tenant-scoped (VMs, credentials) and which are shared (physical switches, hypervisors, storage arrays).

---

### 6.4 Compensation Partially Fails

**Gap:** WORKFLOW-CONTRACT.md says compensation failures are "logged and alerted but do not block other compensations." But what is the workflow's final state?

**Scenario:** A 5-step provisioning workflow fails at step 4. Compensation runs for steps 3, 2, 1. Step 3 compensation succeeds. Step 2 compensation fails (the VLAN delete timed out). Step 1 compensation succeeds. The workflow is now in a state where:
- Step 1 is compensated (IPs released).
- Step 2 is NOT compensated (VLAN still exists).
- Step 3 is compensated (network config removed, but the VLAN from step 2 is still there).

**Impact:** The infrastructure is in an inconsistent state. The workflow state says "compensated" but it is partially compensated. No error type captures this. `PartialError` captures "some operations in a single step succeeded" but not "some compensations in a saga succeeded."

**Recommendation:**
1. Add a `compensation_partial` workflow state (distinct from `compensating` and `failed`).
2. When compensation partially fails, generate a `CompensationReport` that lists exactly what was undone and what remains. This report is the operator's remediation checklist.
3. Add a `loom workflow remediate <id>` command that re-attempts failed compensations.

---

## 7. Competitive Blindspots

### 7.1 Crossplane + Backstage + Kubecost = 80% of LOOM?

**Gap:** COMPETITIVE-ANALYSIS.md dismisses Crossplane ("everything must be a K8s CRD") and Backstage ("a catalog, not an orchestrator"). But the combination of:
- Crossplane (cloud + VM provisioning via K8s API)
- Backstage (service catalog + developer portal)
- Kubecost (cost allocation and optimization)
- Argo Workflows (workflow orchestration with K8s)
- Ansible (network device config)

...covers 70-80% of what LOOM promises, using mature tools with large communities. The remaining 20% (bare metal BMC, PiKVM, LLM placement, multi-vendor network translation) is genuinely unique to LOOM, but it is the most niche 20%.

**Impact:** Enterprise buyers will ask "why not Crossplane + Backstage + Kubecost?" LOOM's competitive response must be crisp and honest about where it adds value vs. where it duplicates existing tools.

**Recommendation:**
1. Add a "LOOM vs. Composite Stack" section to COMPETITIVE-ANALYSIS.md that honestly assesses Crossplane + Backstage + Kubecost + Argo + Ansible.
2. Position LOOM's unique value in three areas: (a) bare metal + BMC protocol unification, (b) multi-vendor network config translation, (c) cross-domain verification. These are the capabilities no combination of existing tools provides.
3. Consider whether LOOM should *integrate with* Crossplane/Backstage rather than *replace* them. The "orchestrator of orchestrators" vision means LOOM could orchestrate Crossplane as a provider, not compete with it.

---

### 7.2 The "We Already Have Tools" Objection

**Gap:** COMPETITIVE-ANALYSIS.md says enterprises operate "15-30 separate management tools." But enterprises have *invested* in those tools. They have team expertise, runbooks, monitoring integrations, and compliance certifications built around those tools.

**The objection:** "We already have Terraform for provisioning, Ansible for config management, Cisco NSO for network, and Datadog for monitoring. LOOM requires us to re-learn, re-certify, and re-integrate everything. What's the ROI?"

**Impact:** LOOM's adoption barrier is not technical -- it is organizational. The docs focus entirely on technical differentiation and say nothing about adoption strategy.

**Recommendation:**
1. Define an incremental adoption path: "Start with discovery only (Phase 2). Then add workflow orchestration (Phase 4). Then add network config (Phase 7)." Each phase should deliver standalone value.
2. Define explicit integration points with existing tools: "LOOM can trigger Ansible playbooks as workflow activities," "LOOM can import Terraform state for cost tracking," "LOOM can forward events to Datadog for monitoring." The "orchestrator of orchestrators" vision requires these integrations to be first-class, not afterthoughts.
3. Add an `ADOPTION-STRATEGY.md` doc that addresses the organizational dimension: who champions LOOM, which team owns it, how it integrates with existing ITSM processes.

---

### 7.3 The Scope Monster

**Gap:** LOOM claims to orchestrate: containers, VMs, bare metal, switches, routers, firewalls, serverless functions, SaaS APIs, custom binaries, and edge nodes. It claims to handle: 25+ protocols, 15+ NOS dialects, 5 hypervisor platforms, 3 cloud providers, and an LLM decision engine. It claims to provide: cost optimization, compliance verification, drift detection, topology visualization, workflow orchestration, and a UI.

**This is the scope of 10 startups, not one project.**

The implementation plan has 12 phases. At a generous estimate of 3-4 months per phase (with a small team), that is 3-4 years to complete. By Phase 11, the competitive landscape will have shifted dramatically. The tools LOOM competes with today will have absorbed some of LOOM's innovations.

**Impact:** Scope creep is the #1 killer of ambitious infrastructure projects. LOOM risks becoming permanently "90% done" on everything and "100% done" on nothing.

**Recommendation:**
1. Define an MVP that is shippable at Phase 5 (API + auth + SSH/Redfish/SNMP/IPMI + Temporal workflows + basic verification). This is "LOOM for bare metal lifecycle management" -- a narrow but valuable product.
2. Everything after Phase 5 should be driven by customer demand, not architectural ambition. If customers need network config translation (Phase 7), build it. If they need cost optimization (Phase 8), build it. Do not build speculatively.
3. Kill features that are not clearly differentiated. Serverless orchestration, SaaS API management, and IoT/edge protocols (LwM2M, TR-069) are mentioned in the README but not in the implementation plan. Remove them from the README until they are planned.

---

## Summary of Findings

| # | Category | Findings | Severity |
|---|----------|----------|----------|
| 1.1 | Impossibility | Saga rollback on devices without undo | Critical |
| 1.2 | Impossibility | Batfish does not cover most target NOS vendors | High |
| 1.3 | Impossibility | Real-time cost data does not exist for bare metal | High |
| 1.4 | Impossibility | "Single pane of glass" requires Phase 9+ | Medium |
| 2.1 | Missing How | Protocol auto-detection algorithm undefined | High |
| 2.2 | Missing How | ACL/routing/QoS translation complexity understated | Critical |
| 2.3 | Missing How | LLM conflict resolution between dimensions undefined | High |
| 2.4 | Missing How | Edge agent autonomy scope undefined | Medium |
| 3.1 | Scale | PostgreSQL combined workload at 10K devices | High |
| 3.2 | Scale | Temporal sizing for 1000+ concurrent workflows | Medium |
| 3.3 | Scale | NATS event volume miscalculated | High |
| 4.1 | Integration | Cloud-to-local LLM failover quality drop | High |
| 4.2 | Integration | Air-gapped mode is a different product | Medium |
| 4.3 | Integration | Universal ACL model too simplistic | High |
| 5.1 | Chicken-Egg | LOOM needs unmanaged infrastructure | Medium |
| 5.2 | Chicken-Egg | Bootstrap credential paradox | High |
| 5.3 | Chicken-Egg | Adapter coverage gates value | Medium |
| 6.1 | Error Path | No handling for garbage device responses | High |
| 6.2 | Error Path | Confident-but-wrong LLM decisions | Critical |
| 6.3 | Error Path | Cross-tenant conflict on shared devices | Critical |
| 6.4 | Error Path | Partial compensation leaves inconsistent state | High |
| 7.1 | Competitive | Composite stack covers 80% of value | High |
| 7.2 | Competitive | Adoption strategy absent | Medium |
| 7.3 | Competitive | Scope is 10 startups, not 1 project | Critical |

**Critical findings (5):** 1.1, 2.2, 6.2, 6.3, 7.3
**High findings (12):** 1.2, 1.3, 2.1, 2.3, 3.1, 3.3, 4.1, 4.3, 5.2, 6.1, 6.4, 7.1
**Medium findings (6):** 1.4, 2.4, 3.2, 4.2, 5.1, 5.3, 7.2

---

## Top 5 Actions (Ordered by Risk Reduction)

1. **Scope the MVP to Phase 5.** Define what is shippable. Kill features that are not differentiated. This addresses 7.3.

2. **Add device-level distributed locking for shared physical infrastructure.** This addresses 6.3 (cross-tenant conflicts) -- a data corruption risk that must be solved before multi-tenancy is real.

3. **Define hard constraints that override LLM recommendations.** This addresses 6.2 (confident-but-wrong) and 2.3 (conflict resolution). Policy rules enforced deterministically are more trustworthy than LLM confidence scores.

4. **Write the missing specification docs:** `PROTOCOL-DETECTION.md`, `COST-MODEL.md`, `PLACEMENT-POLICY.md`, `FEATURE-MATRIX.md`, `BOOTSTRAP.md`. This addresses 2.1, 1.3, 2.3, 4.2, 5.1.

5. **Add `CompensationReliability` and `PreOpSnapshot` to the adapter contract.** This addresses 1.1 (saga rollback on devices without undo) -- the difference between "we tried to roll back" and "we restored the previous state."

---

*This review is deliberately adversarial. The fact that LOOM's documentation is detailed enough to attack at this level of specificity is, itself, a positive signal. Most projects at this stage have nothing worth attacking.*
