# LOOM

**The Universal Infrastructure Orchestrator — Manager of Managers**

LOOM is a universal orchestration platform that weaves together any infrastructure — public cloud, private cloud, bare metal, network devices, edge nodes — into a single, coherent control plane. It extends orchestration beyond infrastructure into individual applications, binaries, and end-to-end workflows.

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

## Status

Early stage — architecture and design phase.

## License

TBD
