# LOOM

**The Orchestrator of Orchestrators**

LOOM doesn't replace Kubernetes, Terraform, Ansible, or any tool that already does its job well. It **orchestrates them**. LOOM is the meta-orchestration layer that weaves together existing tools, infrastructure, and workflows — public cloud, private cloud, bare metal, network devices, edge nodes — into a single, intelligent control plane. It coordinates orchestrators, fills the gaps between them, and makes decisions no individual tool can make because no individual tool has the full picture.

## Vision

- **Universal Reach** — Orchestrate anything: VMs, containers, bare metal, switches, routers, serverless functions, SaaS APIs, and custom binaries. No infrastructure is out of scope.
- **LLM-Driven Decision Heuristics** — Core scheduling and placement decisions powered by large language models, enabling context-aware reasoning over complex, multi-dimensional trade-offs.
- **Cost, Availability & Reachability Aware** — Every decision factors in cost dimensions, availability zones, network reachability, and latency constraints.
- **Capacity Scheduling & Reservation** — Schedule and reserve capacity across heterogeneous infrastructure with automated teardown and cleanup semantics built in.
- **Single Pane of Glass** — Unified view into health, cost, and performance across your entire estate — from cloud regions to rack-level hardware.
- **Workflow Orchestration** — Not just infrastructure — orchestrate application-level workflows, CI/CD pipelines, data flows, and multi-step processes end to end.

## Architecture (Planned)

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

## Documentation

- **[Architecture](docs/ARCHITECTURE.md)** — System design, adapter layer, LLM decision engine
- **[Tech Stack](docs/TECH-STACK.md)** — Language, database, LLM, workflow engine, and UI decisions with rationale
- **[Competitive Analysis](docs/COMPETITIVE-ANALYSIS.md)** — 100+ tools analyzed across 10 domains
- **[Request-to-Device Flow](docs/REQUEST-TO-DEVICE-FLOW.md)** — End-to-end orchestration from intent to verified delivery
- **[Research](research/)** — Comprehensive landscape research across all orchestration domains

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

## Research Coverage

10 parallel research agents analyzed **100+ tools** across the entire orchestration landscape:

| Domain | Tools Analyzed |
|--------|---------------|
| Cloud-Native Orchestrators | Kubernetes, Nomad, OpenShift, Rancher, Tanzu, Docker Swarm, Mesos |
| IaC & Provisioning | Terraform, Pulumi, Crossplane, Ansible, CloudFormation, SaltStack |
| Workflow Orchestrators | Airflow, Temporal, Prefect, Argo, Dagster, Step Functions, NiFi |
| Network & Bare Metal | Cisco NSO, MAAS, Tinkerbell, Ironic, NetBox, Apstra, CloudVision |
| Meta-Orchestrators | Backstage, Kratix, Humanitec, Port, Upbound, Spacelift |
| AI/LLM-Driven Ops | Kubiya, Sedai, Cast.ai, Harness, Dynatrace Davis, Karpenter |
| FinOps & Cost | CloudHealth, Kubecost, Infracost, Vantage, OpenCost, Spot.io |
| Observability | Datadog, Grafana, New Relic, Splunk, Dynatrace, Zabbix |
| Remote Management | IPMI, Redfish, Intel AMT, PiKVM, iDRAC, iLO, SNMP, NETCONF, gNMI |
| Network OS & Switches | SONiC, Cumulus, Cisco IOS/NX-OS/ACI, Junos, EOS, MikroTik, VyOS |

## Status

Early stage — architecture and design phase. Competitive landscape research complete.

## License

TBD
