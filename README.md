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
- **[Competitive Analysis](docs/COMPETITIVE-ANALYSIS.md)** — 100+ tools analyzed across 10 domains
- **[Request-to-Device Flow](docs/REQUEST-TO-DEVICE-FLOW.md)** — End-to-end orchestration from intent to verified delivery
- **[Research](research/)** — Comprehensive landscape research across all orchestration domains

## Research Coverage

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
