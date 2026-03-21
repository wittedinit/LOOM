# Documentation Map

> Navigation guide to all LOOM documentation. Organized by layer, from foundational concepts through contracts, security, and operations. Read each layer before moving to the next.

---

## Reading Order

Start with the Foundation Layer to understand what LOOM is and why it exists. Then read the Architecture Layer for the big picture. The Contract Layer defines every behavioral guarantee. The Security Layer covers threat defense. Operations covers failure handling and API surface. ADRs explain the "why" behind key decisions. Research contains the competitive landscape analysis.

---

## Foundation Layer

Read these first. They establish context, competitive positioning, and technology choices.

| Document | Description |
|----------|-------------|
| [README.md](../README.md) | Project overview, vision, and current status |
| [COMPETITIVE-ANALYSIS.md](COMPETITIVE-ANALYSIS.md) | Analysis of 100+ tools across 10 domains; why no existing tool solves the universal orchestration problem |
| [TECH-STACK.md](TECH-STACK.md) | Technology decisions: Go, PostgreSQL, Temporal, NATS, Valkey, and rationale for each |
| [GLOSSARY.md](GLOSSARY.md) | Definitions of every LOOM-specific term with links to formal specifications |

---

## Architecture Layer

The structural design. How components fit together, what the domain model looks like, and how LOOM deploys.

| Document | Description |
|----------|-------------|
| [ARCHITECTURE.md](ARCHITECTURE.md) | Core architecture: control plane, decision engine, adapter runtime, and the orchestrator-of-orchestrators philosophy |
| [DOMAIN-MODEL.md](DOMAIN-MODEL.md) | Canonical resource types: Device, Endpoint, Credential, Target, and all supporting enums |
| [DEPLOYMENT-TOPOLOGY.md](DEPLOYMENT-TOPOLOGY.md) | Four deployment modes from single binary to globally distributed fleet; edge agent architecture |
| [REQUEST-TO-DEVICE-FLOW.md](REQUEST-TO-DEVICE-FLOW.md) | End-to-end flow from user request through intent parsing, planning, execution, and verification |

---

## Contract Layer

Behavioral contracts define the guarantees each subsystem provides. These are the specifications that implementation must satisfy.

| Document | Description |
|----------|-------------|
| [ADAPTER-CONTRACT.md](ADAPTER-CONTRACT.md) | Operation families (Connector, Discoverer, Executor, StateReader, Watcher), capability declaration, error classification, idempotency, and compensation |
| [IDENTITY-MODEL.md](IDENTITY-MODEL.md) | Device identity: DeviceID stability, ExternalIdentity types, confidence-based matching, merge/split rules |
| [DISCOVERY-CONTRACT.md](DISCOVERY-CONTRACT.md) | Protocol probe sequence (gNMI through PiKVM), timeouts, retry policy, caching, batch scanning, NATS events |
| [OPERATION-TYPES.md](OPERATION-TYPES.md) | Typed operation requests and responses: power control, boot control, network config, command execution |
| [WORKFLOW-CONTRACT.md](WORKFLOW-CONTRACT.md) | Temporal integration rules: workflow submission, step/activity model, verification, compensation, approval gates |
| [VERIFICATION-MODEL.md](VERIFICATION-MODEL.md) | Convergence proof: expected vs observed state, verification checks, evidence capture, failure semantics |
| [EVENT-MODEL.md](EVENT-MODEL.md) | NATS JetStream event subjects, envelope format, domain/action hierarchy, consumer configuration |
| [ERROR-MODEL.md](ERROR-MODEL.md) | Error taxonomy: transient, permanent, partial, validation errors; retry policies and propagation rules |
| [AUDIT-MODEL.md](AUDIT-MODEL.md) | Immutable audit trail: actor identity, correlation/causation chains, before/after state capture |
| [COST-MODEL.md](COST-MODEL.md) | Cost-aware orchestration: bare metal amortization, cloud pricing, network costs, budget enforcement, LLM integration |
| [LLM-BOUNDARIES.md](LLM-BOUNDARIES.md) | Hard rules for LLM integration: no direct execution, typed output, confidence thresholds, deterministic fallbacks |
| [API-CONVENTIONS.md](API-CONVENTIONS.md) | REST API contract: versioning, pagination, filtering, error response format, rate limiting |
| [FAILURE-MODES.md](FAILURE-MODES.md) | Every failure mode in the stack: detection, impact, automatic behavior, operator recovery procedures |

---

## Security Layer

Defense architecture and threat analysis for every attack surface.

| Document | Description |
|----------|-------------|
| [SECURITY-MODEL.md](SECURITY-MODEL.md) | Threat model, crown jewels analysis, defense-in-depth architecture, authentication/authorization |
| [VAULT-ARCHITECTURE.md](VAULT-ARCHITECTURE.md) | Credential vault: envelope encryption (DEK/KEK/MEK), memory protection, HSM integration, zero-plaintext-at-rest |
| [security/ATTACK-SURFACE-API.md](security/ATTACK-SURFACE-API.md) | API attack surface: injection, auth bypass, rate limiting, input validation |
| [security/ATTACK-SURFACE-ADAPTERS.md](security/ATTACK-SURFACE-ADAPTERS.md) | Adapter attack surface: credential theft, protocol downgrade, MITM, command injection |
| [security/ATTACK-SURFACE-LLM.md](security/ATTACK-SURFACE-LLM.md) | LLM attack surface: prompt injection, data exfiltration, recommendation manipulation |
| [security/ATTACK-SURFACE-NATS.md](security/ATTACK-SURFACE-NATS.md) | NATS attack surface: unauthorized subscription, message tampering, replay attacks |
| [security/ATTACK-SURFACE-POSTGRESQL.md](security/ATTACK-SURFACE-POSTGRESQL.md) | PostgreSQL attack surface: SQL injection, privilege escalation, data leakage |
| [security/ATTACK-SURFACE-TEMPORAL.md](security/ATTACK-SURFACE-TEMPORAL.md) | Temporal attack surface: workflow hijacking, activity spoofing, history tampering |
| [security/ATTACK-SURFACE-UI-SUPPLY-CHAIN.md](security/ATTACK-SURFACE-UI-SUPPLY-CHAIN.md) | UI and supply chain attack surface: XSS, dependency poisoning, build pipeline compromise |
| [security/ATTACK-SURFACE-VAULT.md](security/ATTACK-SURFACE-VAULT.md) | Vault attack surface: key extraction, memory scraping, side-channel attacks |

---

## Decision Records

Architectural Decision Records explain why specific technology and design choices were made. Each ADR is a permanent record -- superseded ADRs are marked as such, never deleted.

| Document | Decision |
|----------|----------|
| [adr/ADR-001-go-language.md](adr/ADR-001-go-language.md) | Go as the implementation language |
| [adr/ADR-002-temporal-workflow.md](adr/ADR-002-temporal-workflow.md) | Temporal for workflow orchestration |
| [adr/ADR-003-nats-jetstream.md](adr/ADR-003-nats-jetstream.md) | NATS JetStream for event streaming |
| [adr/ADR-004-postgresql-database.md](adr/ADR-004-postgresql-database.md) | PostgreSQL as the primary data store |
| [adr/ADR-005-tenant-scoping.md](adr/ADR-005-tenant-scoping.md) | Tenant-scoped everything: row-level isolation |
| [adr/ADR-006-adapter-families.md](adr/ADR-006-adapter-families.md) | Operation family model over uniform interface |
| [adr/ADR-007-llm-boundaries.md](adr/ADR-007-llm-boundaries.md) | LLM as advisor, not executor |
| [adr/ADR-008-temporal-truth-db-projection.md](adr/ADR-008-temporal-truth-db-projection.md) | Temporal as source of truth, DB as projection |
| [adr/ADR-009-identity-model.md](adr/ADR-009-identity-model.md) | Confidence-based identity matching |
| [adr/ADR-010-verification-mandatory.md](adr/ADR-010-verification-mandatory.md) | Mandatory verification after every mutation |
| [adr/ADR-011-vault-architecture.md](adr/ADR-011-vault-architecture.md) | Envelope encryption with per-credential DEKs |

---

## Research

Competitive landscape research across 10 domains plus technology stack evaluation. These informed the decisions recorded in ADRs.

| Document | Domain |
|----------|--------|
| [research/01-cloud-native-orchestrators.md](../research/01-cloud-native-orchestrators.md) | Kubernetes, Nomad, Docker Swarm, Mesos |
| [research/02-iac-provisioning-tools.md](../research/02-iac-provisioning-tools.md) | Terraform, Pulumi, CloudFormation, Crossplane |
| [research/03-workflow-orchestrators.md](../research/03-workflow-orchestrators.md) | Temporal, Airflow, Argo, Step Functions |
| [research/04-network-bare-metal-orchestrators.md](../research/04-network-bare-metal-orchestrators.md) | MAAS, Ironic, Tinkerbell, RackHD |
| [research/05-meta-orchestrators.md](../research/05-meta-orchestrators.md) | Cisco NSO, Crossplane, Krateo |
| [research/06-ai-llm-driven-ops.md](../research/06-ai-llm-driven-ops.md) | AIOps platforms, LLM-driven infrastructure |
| [research/07-finops-cost-platforms.md](../research/07-finops-cost-platforms.md) | CloudHealth, Kubecost, Vantage, FinOps tools |
| [research/08-observability-platforms.md](../research/08-observability-platforms.md) | Prometheus, Grafana, Datadog, OpenTelemetry |
| [research/09-remote-management-protocols.md](../research/09-remote-management-protocols.md) | IPMI, Redfish, AMT, PiKVM, iLO, iDRAC |
| [research/10-network-os-switch-platforms.md](../research/10-network-os-switch-platforms.md) | SONiC, Cumulus, IOS, Junos, EOS, NX-OS |
| [research/README.md](../research/README.md) | Research summary and findings overview |
| [research/tech-stack/01-core-language.md](../research/tech-stack/01-core-language.md) | Language evaluation: Go vs Rust vs Python vs Java |
| [research/tech-stack/02-data-store.md](../research/tech-stack/02-data-store.md) | Database evaluation: PostgreSQL vs CockroachDB vs others |
| [research/tech-stack/03-llm-integration.md](../research/tech-stack/03-llm-integration.md) | LLM provider evaluation: local-first, provider-agnostic |
| [research/tech-stack/04-workflow-engine.md](../research/tech-stack/04-workflow-engine.md) | Workflow engine evaluation: Temporal vs Argo vs custom |
| [research/tech-stack/05-ui-dashboard.md](../research/tech-stack/05-ui-dashboard.md) | UI framework evaluation for the operations dashboard |

---

## Implementation

| Document | Description |
|----------|-------------|
| [IMPLEMENTATION-PLAN.md](IMPLEMENTATION-PLAN.md) | Contracts-first, vertical-slice driven implementation plan with phases and milestones |
