# LOOM

**The Orchestrator of Orchestrators**

LOOM doesn't replace Kubernetes, Terraform, Ansible, or any tool that already does its job well. It **orchestrates them**. LOOM is the meta-orchestration layer that weaves together existing tools, infrastructure, and workflows — public cloud, private cloud, bare metal, network devices, edge nodes — into a single, intelligent control plane. It coordinates orchestrators, fills the gaps between them, and makes decisions no individual tool can make because no individual tool has the full picture.

---

## Start Here

| Goal | Documents |
|------|-----------|
| **Understand the Problem** | [Competitive Analysis](docs/COMPETITIVE-ANALYSIS.md) -- [Research](research/) |
| **Understand the Solution** | [Architecture](docs/ARCHITECTURE.md) -- [Tech Stack](docs/TECH-STACK.md) -- [Request-to-Device Flow](docs/REQUEST-TO-DEVICE-FLOW.md) |
| **Understand the Contracts** | [Domain Model](docs/DOMAIN-MODEL.md) -- [Adapter Contract](docs/ADAPTER-CONTRACT.md) -- [Workflow Contract](docs/WORKFLOW-CONTRACT.md) |
| **Understand the Security** | [Security Model](docs/SECURITY-MODEL.md) -- [Vault Architecture](docs/VAULT-ARCHITECTURE.md) -- [Attack Surfaces](docs/security/) |
| **Understand the Plan** | [Implementation Plan](docs/IMPLEMENTATION-PLAN.md) -- [Deployment Topology](docs/DEPLOYMENT-TOPOLOGY.md) |

---

## Tech Stack

| Component | Choice | Why |
|-----------|--------|-----|
| Language | **Go** | Protocol libraries, goroutines, single binary deploys |
| Workflow Engine | **Temporal** | Durable execution, saga pattern, signals, visibility |
| Event Bus | **NATS JetStream** | 14MB binary, edge leaf nodes, 150K msgs/sec |
| Database | **PostgreSQL + TimescaleDB + pgvector** | Multi-model (relational + time-series + vector) in one system |
| UI | **React + Next.js + shadcn/ui** | Largest ecosystem, topology visualization support |
| LLM | **Claude API + local vLLM** | Provider-agnostic, air-gapped capable |

Full rationale: [Tech Stack Decision](docs/TECH-STACK.md) | Research: [Language](research/tech-stack/01-core-language.md), [Data Store](research/tech-stack/02-data-store.md), [LLM](research/tech-stack/03-llm-integration.md), [Workflow](research/tech-stack/04-workflow-engine.md), [UI](research/tech-stack/05-ui-dashboard.md)

---

## Security Posture

**71 attack vectors** tested across **8 attack surfaces** with working exploit and defense code for each:

| Surface | Vectors | Surface | Vectors |
|---------|---------|---------|---------|
| [Protocol Adapters](docs/security/ATTACK-SURFACE-ADAPTERS.md) | 10 | [NATS JetStream](docs/security/ATTACK-SURFACE-NATS.md) | 8 |
| [REST API](docs/security/ATTACK-SURFACE-API.md) | 10 | [PostgreSQL](docs/security/ATTACK-SURFACE-POSTGRESQL.md) | 7 |
| [LLM Integration](docs/security/ATTACK-SURFACE-LLM.md) | 8 | [Temporal](docs/security/ATTACK-SURFACE-TEMPORAL.md) | 8 |
| [Credential Vault](docs/security/ATTACK-SURFACE-VAULT.md) | 10 | [UI & Supply Chain](docs/security/ATTACK-SURFACE-UI-SUPPLY-CHAIN.md) | 10 |

The [credential vault](docs/VAULT-ARCHITECTURE.md) implements enterprise-grade encryption comparable to Apple's Data Protection architecture:
- **AES-256-GCM** with AES-NI hardware acceleration on all modern x86-64 processors
- **Envelope encryption**: per-credential DEKs wrapped in per-tenant KEKs wrapped in a master key
- **Zero plaintext credentials** — ever — in database, filesystem, swap, logs, error messages, or serialization output
- Memory-locked pages, constant-time wipes, core dump protection

Full model: [Security Model](docs/SECURITY-MODEL.md) | [Vault Architecture](docs/VAULT-ARCHITECTURE.md)

---

## Project Status

**Phase:** 0 — Contracts (Freeze All Seams Before Code)

**Completed:**
- Competitive landscape research across 10 domains and 100+ tools
- Architecture design and system decomposition
- Tech stack selection with ADR-backed decisions
- All 22 contract and specification documents
- 11 Architecture Decision Records
- Security analysis: 71 attack vectors across 8 surfaces
- Vault architecture with hardware-accelerated encryption design
- 12-phase implementation plan (v3, AI-reviewed)

**Next:** Phase 1 — Skeleton (binary boots, Temporal connects, NATS connects, single table migrates)

Full plan: [Implementation Plan](docs/IMPLEMENTATION-PLAN.md)

---

## Vision

- **Universal Reach** — Orchestrate anything: VMs, containers, bare metal, switches, routers, serverless functions, SaaS APIs, and custom binaries. No infrastructure is out of scope.
- **LLM-Driven Decision Heuristics** — Core scheduling and placement decisions powered by large language models, enabling context-aware reasoning over complex, multi-dimensional trade-offs.
- **Cost, Availability & Reachability Aware** — Every decision factors in cost dimensions, availability zones, network reachability, and latency constraints.
- **Capacity Scheduling & Reservation** — Schedule and reserve capacity across heterogeneous infrastructure with automated teardown and cleanup semantics built in.
- **Single Pane of Glass** — Unified view into health, cost, and performance across your entire estate — from cloud regions to rack-level hardware.
- **Workflow Orchestration** — Not just infrastructure — orchestrate application-level workflows, CI/CD pipelines, data flows, and multi-step processes end to end.

## Architecture

```
┌─────────────────────────────────────────────────────────┐
│                      LOOM Core                          │
│  ┌─────────────┐  ┌──────────────┐  ┌───────────────┐  │
│  │  LLM Engine  │  │  Scheduler   │  │  Cost Engine  │  │
│  │  (Heuristics)│  │  (Capacity)  │  │  (Optimizer)  │  │
│  └──────┬──────┘  └──────┬───────┘  └───────┬───────┘  │
│         └────────────────┼──────────────────┘           │
│                    ┌─────┴─────┐                        │
│                    │  Planner  │                        │
│                    └─────┬─────┘                        │
├──────────────────────────┼──────────────────────────────┤
│                   Adapter Layer                         │
│  ┌────────┐ ┌────────┐ ┌────────┐ ┌────────┐ ┌──────┐ │
│  │  Cloud  │ │  K8s   │ │ Bare   │ │Network │ │ App  │ │
│  │  (AWS/  │ │        │ │ Metal  │ │(Switch/│ │ & Bin│ │
│  │  GCP/Az)│ │        │ │        │ │Router) │ │      │ │
│  └────────┘ └────────┘ └────────┘ └────────┘ └──────┘ │
├─────────────────────────────────────────────────────────┤
│               Observability & Control                   │
│  ┌──────────┐  ┌─────────────┐  ┌───────────────────┐  │
│  │  Health   │  │ Performance │  │  Cost Dashboard   │  │
│  └──────────┘  └─────────────┘  └───────────────────┘  │
└─────────────────────────────────────────────────────────┘
```

## Key Differentiators

| Capability | Existing Tools | LOOM |
|---|---|---|
| **Scope** | Narrow silos (containers OR network OR VMs) | Universal — containers, VMs, bare metal, switches, routers, firewalls, binaries |
| **Decision Making** | Static rules (affinity, resource requests) | LLM-driven heuristics across cost, availability, reachability, compliance |
| **Cost** | Separate FinOps tools that observe, not control | Cost embedded as first-class scheduling dimension |
| **Network Devices** | Separate network automation (15+ NOS dialects) | Unified adapter layer across SONiC, Cumulus, IOS, Junos, EOS, MikroTik, etc. |
| **Remote Management** | Per-protocol tooling (Redfish, IPMI, AMT, PiKVM) | Auto-detects and uses correct protocol per device |
| **Verification** | Manual / hope / separate monitoring | Automated end-to-end verification with oversight detection |
| **Lifecycle** | Manual teardown, forgotten resources | Automated teardown with budget, TTL, and idle triggers |
| **Multi-Tenancy** | Namespace-level at best | First-class tenant model with network segmentation and firewall orchestration |

---

## Why LOOM Exists: The Problem

After exhaustive research across **10 domains** and **100+ tools**, one conclusion is clear: **no single tool — or combination of tools — provides universal infrastructure orchestration.** The market is deeply fragmented into vertical silos that don't communicate, don't share context, and can't make coordinated decisions.

An enterprise typically operates **15-30 separate management tools** with zero coordination between them.

### The 8 Systemic Gaps Every Tool Shares

#### 1. No Cross-Domain Orchestration

Every tool operates within its silo. Kubernetes orchestrates containers — not switches. Terraform provisions cloud resources — doesn't know what happens after. Ansible configures servers — has no state, no cost awareness. Cisco NSO configures network devices — blind to compute and applications. Airflow orchestrates workflows — assumes infrastructure already exists.

**The handoff gap is where things break.** Terraform creates a VM. Ansible configures it. But who ensures the switch port is in the right VLAN? That the firewall rule allows the traffic? That DNS resolves? **Nobody.** That gap is stitched together with brittle bash scripts and Jenkins pipelines.

#### 2. Zero Native Cost Awareness

**Not a single orchestrator embeds cost in scheduling decisions.** Kubernetes doesn't know an m5.xlarge costs $0.192/hr vs spot at $0.06/hr. Nomad's bin-packing optimizes utilization, not spend. Terraform provisions where you tell it, not where it's cheapest.

FinOps tools (Kubecost, CloudHealth, Infracost, Vantage) **observe and report** costs — but they're separate systems that can't influence placement decisions. The decision-maker and the cost-knower are always different tools.

Organizations waste an average of **28% of cloud spend** (Flexera). Nobody tracks bare metal costs (power, cooling, depreciation, rack fees) alongside cloud costs.

#### 3. No Network Device Orchestration From Any Platform

Across **all 100+ tools analyzed** — not one platform manages network devices as first-class orchestrable resources alongside compute.

The NOS fragmentation is staggering:
- **15+ distinct CLI dialects** (Cisco IOS vs Junos vs EOS vs NX-OS vs MikroTik vs SONiC vs Cumulus NVUE...)
- **8+ API protocols** (REST, NETCONF, gNMI, SNMP, gRPC, SSH/CLI, JSON-RPC, vendor binary)
- **5 distinct Cisco NOS families alone** (IOS, IOS-XE, IOS-XR, NX-OS, ACI — all different)

No existing abstraction (NAPALM, Nornir, Netmiko, Ansible Network) covers both broad vendor coverage AND deep feature access AND real-time state.

#### 4. Remote Management Protocol Fragmentation

LOOM must support **25-30 distinct protocol/API combinations** to truly manage all infrastructure:

| Category | Protocols | Challenge |
|----------|-----------|-----------|
| Server BMC | Redfish, IPMI, **Intel AMT** (non-Redfish, SOAP), OpenBMC | AMT uses WS-Management — completely different stack |
| KVM-over-IP | PiKVM (REST), BliKVM, TinyPilot, Raritan (proprietary) | No standard, no fleet management |
| Network | SNMP, NETCONF, gNMI, SSH/CLI, vendor REST APIs | 6+ protocols per network deployment |
| Hypervisor | vSphere, Proxmox, libvirt, Hyper-V (WMI), Nutanix | Each with different API paradigms |
| IoT/Edge | LwM2M, MQTT, TR-069/USP | Completely different paradigm from data center |

#### 5. No LLM-Driven Decision Intelligence

Every orchestrator uses **static rules**: affinity, resource requests, priority classes. No orchestrator can reason about workload patterns, predict demand, or make context-aware tradeoffs.

AIOps tools (Dynatrace Davis, BigPanda, Moogsoft) bolt on AI for **alerting and correlation** but never for **core scheduling decisions**. They identify problems but don't fix them. They observe but don't act.

#### 6. No Automated Lifecycle / Teardown Semantics

No tool natively supports: "provision this environment, run the workload, verify success, tear everything down." Kubernetes has TTL for finished Jobs — nothing for running workloads or namespaces. Terraform can `destroy` but doesn't do it automatically. Nobody tracks idle environments or enforces budget caps.

#### 7. No Request Verification or Oversight Detection

No tool can take a complex request, execute it across all layers, and then **prove every element was delivered correctly**. Terraform says "apply complete." Ansible says "ok." But did the VLAN actually propagate? Is the firewall actually blocking cross-tenant traffic?

More critically — **no tool questions the request itself**: "You asked for 30 VMs but didn't specify DNS — shall I configure it?"

#### 8. The Observe-Act Chasm

Observability tools (Datadog, Grafana, Dynatrace, Splunk) — **11 out of 13 have zero orchestration capability**. They watch but cannot act. Orchestrators act but have limited observability. Watch and act are always separate systems with a human bridge.

---

## How LOOM Solves This

| Principle | What It Means |
|-----------|---------------|
| **Orchestrator of orchestrators** | Doesn't replace K8s/Terraform/Ansible — coordinates them as steps in a single plan |
| **LLM as core decision engine** | Not "AI-powered" marketing — the LLM IS the reasoning engine that weighs cost, availability, reachability, compliance simultaneously |
| **Universal adapter layer** | Pluggable adapters for every protocol (Redfish, IPMI, AMT, PiKVM, SNMP, NETCONF, gNMI, SSH/CLI, vSphere, Proxmox...) behind a unified resource model |
| **Cost as first-class dimension** | Every decision factors in cost — not a separate FinOps report |
| **Verification-first** | Automated end-to-end verification proving every element was delivered, with LLM-powered oversight detection that questions missing requirements |
| **Observe AND act** | Single system that sees the problem, reasons about it, and executes the fix across all infrastructure types |
| **Lifecycle semantics** | TTL, budget caps, idle detection, scheduled teardown — built into the orchestration primitive |

**The bottom line:** The market has mature "eyes" (observability), capable "hands" (automation tools), but no "brain" that spans all infrastructure types and makes holistic decisions. LOOM is that brain.

---

## Documentation Index

### Foundation
| Document | Description |
|----------|-------------|
| [Competitive Analysis](docs/COMPETITIVE-ANALYSIS.md) | 100+ tools analyzed across 10 domains — the gap analysis that justifies LOOM |
| [Architecture](docs/ARCHITECTURE.md) | System design, adapter layer, LLM decision engine, component decomposition |
| [Tech Stack](docs/TECH-STACK.md) | Language, database, LLM, workflow engine, and UI decisions with full rationale |
| [Deployment Topology](docs/DEPLOYMENT-TOPOLOGY.md) | Single-laptop to globally distributed fleet — 4 deployment modes with resource sizing |

### Contracts & Models
| Document | Description |
|----------|-------------|
| [Domain Model](docs/DOMAIN-MODEL.md) | Canonical resource types: Target, Device, Endpoint, Capability, StateSnapshot |
| [Adapter Contract](docs/ADAPTER-CONTRACT.md) | Operation-family interfaces: Connector, Discoverer, Executor, StateReader, Watcher |
| [Workflow Contract](docs/WORKFLOW-CONTRACT.md) | Temporal integration rules: one step = one business action, one activity = one side effect |
| [Discovery Contract](docs/DISCOVERY-CONTRACT.md) | Device discovery behavioral contract: probe, identify, classify, enrich |
| [Operation Types](docs/OPERATION-TYPES.md) | Typed operation structs — no `map[string]any` — compile-time correctness |
| [Identity Model](docs/IDENTITY-MODEL.md) | The most dangerous seam: DeviceID stability, merge/split rules, confidence scoring |
| [Event Model](docs/EVENT-MODEL.md) | NATS JetStream subject hierarchy, envelope format, stream configuration |
| [Error Model](docs/ERROR-MODEL.md) | Error taxonomy, propagation rules, retry policies across all components |
| [Verification Model](docs/VERIFICATION-MODEL.md) | How LOOM proves convergence: desired state matches observed state |
| [Audit Model](docs/AUDIT-MODEL.md) | Compliance audit trail: who did what, when, to which device, with what result |
| [API Conventions](docs/API-CONVENTIONS.md) | REST API versioning, pagination, filtering, error response contracts |
| [Request-to-Device Flow](docs/REQUEST-TO-DEVICE-FLOW.md) | End-to-end orchestration path from user intent to verified device state |

### Security
| Document | Description |
|----------|-------------|
| [Security Model](docs/SECURITY-MODEL.md) | Threat model, authentication, authorization, network segmentation |
| [Vault Architecture](docs/VAULT-ARCHITECTURE.md) | Envelope encryption, memory protection, hardware acceleration, key rotation |
| [LLM Boundaries](docs/LLM-BOUNDARIES.md) | Hard rules: LLM suggests, never executes — enforced in code, not policy |
| [Failure Modes](docs/FAILURE-MODES.md) | Every failure mode cataloged: detection, impact, automatic response, operator action |
| [Attack Surfaces](docs/security/) | 71 vectors across 8 surfaces with exploit and defense code |

### Operations
| Document | Description |
|----------|-------------|
| [Implementation Plan](docs/IMPLEMENTATION-PLAN.md) | 12-phase roadmap from contracts to production (v3, AI-reviewed) |
| [Testing Strategy](docs/TESTING-STRATEGY.md) | Unit through E2E against simulated device fleets — all must pass before merge |

### Architecture Decision Records
| ADR | Decision |
|-----|----------|
| [ADR-001](docs/adr/ADR-001-go-language.md) | Go as primary language |
| [ADR-002](docs/adr/ADR-002-temporal-workflow.md) | Temporal for workflow orchestration |
| [ADR-003](docs/adr/ADR-003-nats-jetstream.md) | NATS JetStream for messaging |
| [ADR-004](docs/adr/ADR-004-postgresql-database.md) | PostgreSQL as primary database |
| [ADR-005](docs/adr/ADR-005-tenant-scoping.md) | Tenant scoping from day one |
| [ADR-006](docs/adr/ADR-006-adapter-families.md) | Operation-family adapter model |
| [ADR-007](docs/adr/ADR-007-llm-boundaries.md) | LLM suggests, never executes |
| [ADR-008](docs/adr/ADR-008-temporal-truth-db-projection.md) | Temporal as execution truth, database as projection |
| [ADR-009](docs/adr/ADR-009-identity-model.md) | DeviceID + ExternalIdentity with confidence |
| [ADR-010](docs/adr/ADR-010-verification-mandatory.md) | Verification is mandatory |
| [ADR-011](docs/adr/ADR-011-vault-architecture.md) | Enterprise-grade vault with hardware-accelerated encryption |

### Research
| Document | Scope |
|----------|-------|
| [Cloud-Native Orchestrators](research/01-cloud-native-orchestrators.md) | Kubernetes, Nomad, OpenShift, Rancher, Tanzu, Docker Swarm, Mesos |
| [IaC & Provisioning](research/02-iac-provisioning-tools.md) | Terraform, Pulumi, Crossplane, Ansible, CloudFormation, SaltStack |
| [Workflow Orchestrators](research/03-workflow-orchestrators.md) | Airflow, Temporal, Prefect, Argo, Dagster, Step Functions, NiFi |
| [Network & Bare Metal](research/04-network-bare-metal-orchestrators.md) | Cisco NSO, MAAS, Tinkerbell, Ironic, NetBox, Apstra, CloudVision |
| [Meta-Orchestrators](research/05-meta-orchestrators.md) | Backstage, Kratix, Humanitec, Port, Upbound, Spacelift |
| [AI/LLM-Driven Ops](research/06-ai-llm-driven-ops.md) | Kubiya, Sedai, Cast.ai, Harness, Dynatrace Davis, Karpenter |
| [FinOps & Cost](research/07-finops-cost-platforms.md) | CloudHealth, Kubecost, Infracost, Vantage, OpenCost, Spot.io |
| [Observability](research/08-observability-platforms.md) | Datadog, Grafana, New Relic, Splunk, Dynatrace, Zabbix |
| [Remote Management](research/09-remote-management-protocols.md) | IPMI, Redfish, Intel AMT, PiKVM, iDRAC, iLO, SNMP, NETCONF, gNMI |
| [Network OS & Switches](research/10-network-os-switch-platforms.md) | SONiC, Cumulus, Cisco IOS/NX-OS/ACI, Junos, EOS, MikroTik, VyOS |
| [Tech Stack Research](research/tech-stack/) | Language, data store, LLM, workflow engine, UI framework evaluations |

---

## License

TBD
