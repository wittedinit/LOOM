# LOOM Research: Competitive Landscape Analysis

Comprehensive research across 100+ tools spanning 10 domains, investigating what exists today, where they fall short, and how LOOM addresses the gaps.

## Research Documents

| # | Document | Scope | Tools Analyzed |
|---|----------|-------|----------------|
| 01 | [Cloud-Native Orchestrators](01-cloud-native-orchestrators.md) | Container/workload scheduling | Kubernetes, Nomad, Docker Swarm, Mesos, OpenShift, Rancher, Tanzu |
| 02 | [IaC & Provisioning Tools](02-iac-provisioning-tools.md) | Infrastructure as Code | Terraform, Pulumi, Crossplane, CloudFormation, Ansible, SaltStack, Puppet/Chef |
| 03 | [Workflow Orchestrators](03-workflow-orchestrators.md) | Pipeline/workflow engines | Airflow, Temporal, Prefect, Argo, Dagster, Conductor, NiFi, Step Functions |
| 04 | [Network & Bare-Metal Orchestrators](04-network-bare-metal-orchestrators.md) | Physical infra management | Cisco NSO, MAAS, Tinkerbell, Ironic, NetBox, Nautobot, Apstra, CloudVision |
| 05 | [Meta-Orchestrators](05-meta-orchestrators.md) | Manager-of-managers platforms | Backstage, Kratix, Humanitec, Port, Upbound, Spacelift, Massdriver |
| 06 | [AI/LLM-Driven Ops](06-ai-llm-driven-ops.md) | AIOps and AI orchestration | Kubiya, Shoreline, Dynatrace Davis, Cast.ai, Sedai, Harness, Karpenter |
| 07 | [FinOps & Cost Platforms](07-finops-cost-platforms.md) | Cloud cost management | CloudHealth, Kubecost, Infracost, Vantage, OpenCost, Spot.io |
| 08 | [Observability Platforms](08-observability-platforms.md) | Monitoring & single pane of glass | Datadog, Grafana, New Relic, Splunk, Dynatrace, Zabbix, ServiceNow |
| 09 | [Remote Management Protocols](09-remote-management-protocols.md) | Out-of-band & BMC protocols | IPMI, Redfish, Intel AMT, PiKVM, iDRAC, iLO, SNMP, NETCONF, gNMI |
| 10 | [Network OS & Switch Platforms](10-network-os-switch-platforms.md) | NOS diversity & automation | SONiC, Cumulus, Cisco IOS/NX-OS/ACI, Junos, Arista EOS, MikroTik, VyOS |

## Synthesis Documents

| Document | Purpose |
|----------|---------|
| [Competitive Analysis](../docs/COMPETITIVE-ANALYSIS.md) | Cross-domain gap analysis and LOOM positioning |
| [Architecture](../docs/ARCHITECTURE.md) | LOOM system architecture with adapter layer design |
| [Request-to-Device Flow](../docs/REQUEST-TO-DEVICE-FLOW.md) | End-to-end orchestration path from intent to verified delivery |

## Key Finding

**No single tool — or combination of tools — provides universal infrastructure orchestration.** The market is fragmented into 15-30 separate management tools per enterprise with zero coordination between them. LOOM is the horizontal integration layer.
