# LOOM Architecture: Universal Infrastructure Orchestrator

## Design Philosophy

**LOOM is the orchestrator of orchestrators.** It does not replace Kubernetes, Terraform, Ansible, Cisco NSO, or any tool that already does its job well. There is no point reworking things that already work. Instead, LOOM:

1. **Orchestrates existing orchestrators** — Tells Terraform to provision, Ansible to configure, Kubernetes to schedule, NSO to configure network devices — as coordinated steps in a single plan.
2. **Fills the gaps between them** — No tool handles the handoff from "Terraform created the VM" to "Ansible configured it" to "the switch port is in the right VLAN" to "the firewall rule allows the traffic." LOOM owns these gaps.
3. **Makes decisions no individual tool can** — Kubernetes doesn't know the cost. Terraform doesn't know the network latency. Ansible doesn't know the capacity. LOOM sees all dimensions and makes holistic decisions.
4. **Verifies end-to-end** — Individual tools report success for their slice. LOOM verifies the entire request was delivered correctly across all tools and layers.

---

## Core Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                        LOOM CONTROL PLANE                          │
│                                                                     │
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │                    REQUEST GATEWAY                            │   │
│  │  Intent Parser → Decomposer → Validator → Oversight Checker  │   │
│  └──────────────────────┬───────────────────────────────────────┘   │
│                         │                                           │
│  ┌──────────────────────▼───────────────────────────────────────┐   │
│  │                   LLM DECISION ENGINE                         │   │
│  │                                                               │   │
│  │  ┌─────────────┐ ┌──────────────┐ ┌───────────────────────┐  │   │
│  │  │ Cost Engine  │ │ Availability │ │ Reachability Engine   │  │   │
│  │  │ (FinOps)     │ │ Engine       │ │ (Network Topology)    │  │   │
│  │  └─────────────┘ └──────────────┘ └───────────────────────┘  │   │
│  │  ┌─────────────┐ ┌──────────────┐ ┌───────────────────────┐  │   │
│  │  │ Capacity    │ │ Compliance   │ │ Performance           │  │   │
│  │  │ Planner     │ │ Engine       │ │ Predictor             │  │   │
│  │  └─────────────┘ └──────────────┘ └───────────────────────┘  │   │
│  └──────────────────────┬───────────────────────────────────────┘   │
│                         │                                           │
│  ┌──────────────────────▼───────────────────────────────────────┐   │
│  │                    EXECUTION PLANNER                          │   │
│  │                                                               │   │
│  │  Dependency Graph → Execution Order → Parallel/Sequential    │   │
│  │  Rollback Plan → Verification Steps → Teardown Schedule      │   │
│  └──────────────────────┬───────────────────────────────────────┘   │
│                         │                                           │
│  ┌──────────────────────▼───────────────────────────────────────┐   │
│  │                 UNIVERSAL ADAPTER LAYER                       │   │
│  │                                                               │   │
│  │  ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐           │   │
│  │  │ Compute │ │ Network │ │ Storage │ │  App    │           │   │
│  │  │ Adapters│ │ Adapters│ │ Adapters│ │ Adapters│           │   │
│  │  └────┬────┘ └────┬────┘ └────┬────┘ └────┬────┘           │   │
│  └───────┼──────────┼──────────┼──────────┼────────────────────┘   │
│          │          │          │          │                         │
├──────────┼──────────┼──────────┼──────────┼─────────────────────────┤
│          │          │          │          │                         │
│  ┌───────▼──────────▼──────────▼──────────▼────────────────────┐   │
│  │              VERIFICATION & OBSERVABILITY                    │   │
│  │                                                               │   │
│  │  Health Monitor → Cost Tracker → Performance Analyzer        │   │
│  │  Compliance Checker → Drift Detector → Anomaly Detector      │   │
│  └──────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Layer 1: Request Gateway

The entry point for all orchestration requests. This is where LOOM's intelligence begins.

### Intent Parser
Accepts requests in multiple forms:
- Declarative YAML/JSON specifications
- Natural language (via LLM interpretation)
- API calls from CI/CD pipelines
- Event-driven triggers (alerts, schedules, webhooks)

### Request Decomposer
Breaks a complex request into a dependency graph of atomic operations:

**Example Request**: "Deploy a 30-VM workflow with 5 applications across 3 tenants"

**Decomposition**:
```
1. Tenant Isolation Layer
   ├── Tenant A: VLAN 100, Subnet 10.1.0.0/24, Firewall Zone A
   ├── Tenant B: VLAN 200, Subnet 10.2.0.0/24, Firewall Zone B
   └── Tenant C: VLAN 300, Subnet 10.3.0.0/24, Firewall Zone C

2. Network Infrastructure
   ├── Switch Configuration (SONiC/Cumulus/IOS — auto-detected per device)
   │   ├── VLAN creation on leaf switches
   │   ├── Trunk port configuration on spine switches
   │   └── Inter-VLAN routing (if required)
   ├── Firewall Rules
   │   ├── ACLs between tenant segments
   │   ├── Ingress/egress policies per application tier
   │   └── NAT rules for external access
   └── Load Balancer Configuration
       └── VIP assignments per application

3. Compute Provisioning
   ├── Bare Metal (via IPMI/Redfish/AMT/PiKVM — auto-detected)
   │   ├── Power-on sequence
   │   ├── PXE boot / OS installation
   │   └── Post-install configuration
   ├── Virtual Machines (via vSphere/Proxmox/libvirt/Hyper-V)
   │   ├── VM creation with tenant-specific networking
   │   ├── OS installation (Linux/Windows — as specified)
   │   └── Network interface binding to correct VLAN
   └── Container Workloads (via K8s/Nomad)
       └── Namespace per tenant with network policies

4. Application Deployment
   ├── Binary deployment (SCP/Ansible/direct)
   ├── Configuration injection (per-tenant, per-environment)
   ├── Service discovery registration
   └── Health check endpoint verification

5. Verification
   ├── Network connectivity tests (ping, traceroute, port scan)
   ├── Application health checks (HTTP, TCP, custom)
   ├── Firewall rule validation (can access what should, blocked from what shouldn't)
   ├── Tenant isolation verification (cross-tenant traffic blocked)
   ├── Performance baseline capture
   └── Cost attribution per tenant
```

### Oversight Checker (LLM-Powered)
Before execution, the LLM reviews the request for:
- **Missing dependencies**: "You requested 30 VMs but didn't specify DNS — shall I configure DNS?"
- **Security oversights**: "These VMs are in different tenants but share a subnet — is that intentional?"
- **Resource conflicts**: "The requested firewall rules would block the application's health check endpoint"
- **Cost implications**: "This configuration will cost approximately $X/month — 40% could be saved by using spot instances for the batch workload VMs"

---

## Layer 2: LLM Decision Engine

The core intelligence of LOOM. Unlike every existing orchestrator that uses static rules, LOOM uses LLMs to make decisions that require reasoning across multiple dimensions simultaneously.

### Decision Dimensions

| Dimension | What It Considers | Example Decision |
|-----------|------------------|-----------------|
| **Cost** | Spot vs on-demand, cloud vs bare metal, reserved vs pay-as-you-go | "Place batch workloads on spot instances, databases on reserved" |
| **Availability** | AZ distribution, failure domains, redundancy requirements | "Spread across 3 AZs, avoid single-rack failure domain" |
| **Reachability** | Network latency, hop count, bandwidth, path quality | "Place app server within 2ms of database, not across WAN" |
| **Capacity** | Current utilization, reservation windows, burst headroom | "GPU cluster 80% utilized — schedule training job for 2AM" |
| **Compliance** | Data sovereignty, regulatory requirements, tenant isolation | "EU data stays on EU bare metal, not US cloud" |
| **Performance** | Historical patterns, predicted load, resource contention | "This workload peaks on Mondays — pre-warm capacity" |

### LLM Advantages Over Static Rules

Static rules: `if cpu > 80% then scale_up()`

LLM reasoning: "CPU is at 75% but based on the last 4 Mondays, this service will hit 95% in 2 hours. The cheapest option is to add 2 spot instances now rather than waiting for the autoscaler to react. However, the spot market for m5.xlarge in us-east-1 is volatile — use us-west-2 where spot prices have been stable for 72 hours."

---

## Layer 3: Universal Adapter Layer

The adapter layer is LOOM's bridge to the physical world. Each adapter normalizes a specific management protocol into LOOM's universal resource model.

### Compute Adapters

| Adapter | Protocol | Targets |
|---------|----------|---------|
| Redfish | REST (DMTF standard) | Dell iDRAC, HPE iLO, Supermicro, Lenovo XClarity |
| IPMI | Binary/UDP | Legacy servers, any IPMI-capable BMC |
| Intel AMT | SOAP/WS-Management | Intel vPro desktops/servers (non-Redfish) |
| PiKVM/BliKVM | REST API | Raspberry Pi KVM-over-IP, DIY remote management |
| TinyPilot | REST API | Alternative Pi-based KVM |
| vSphere | REST/SOAP | VMware ESXi hosts and vCenter |
| Proxmox | REST API | Proxmox VE hypervisors |
| libvirt | API/virsh | KVM/QEMU on Linux hosts |
| Hyper-V | WMI/PowerShell | Windows Server hypervisors |
| Nutanix | REST API | Nutanix AHV clusters |
| Cloud | Provider SDKs | AWS EC2, GCP GCE, Azure VMs |

### Network Adapters

| Adapter | Protocol | Targets |
|---------|----------|---------|
| NETCONF/YANG | NETCONF | Cisco IOS-XE/XR, Junos, Nokia, Arista (standard) |
| gNMI | gRPC | SONiC, Nokia SR Linux, Arista, Cisco (modern) |
| NX-API | REST/JSON-RPC | Cisco NX-OS (Nexus switches) |
| eAPI | JSON-RPC/HTTP | Arista EOS |
| APIC REST | REST | Cisco ACI fabrics |
| NVUE | REST/OpenAPI | Cumulus Linux / NVIDIA switches |
| RouterOS | REST/Binary | MikroTik devices |
| SSH/CLI | SSH | Legacy devices, OpenWrt, VyOS, pfSense |
| SNMP | SNMP v2c/v3 | Monitoring/polling across all devices |
| OPNsense | REST API | OPNsense firewalls |
| UniFi | REST API | Ubiquiti UniFi infrastructure |

### Storage Adapters

| Adapter | Protocol | Targets |
|---------|----------|---------|
| S3/Object | REST | AWS S3, MinIO, Ceph RGW |
| NFS/SMB | Native | NAS devices, file servers |
| CSI | gRPC | Kubernetes storage providers |
| ZFS | CLI/API | TrueNAS, FreeBSD ZFS |
| Ceph | REST/CLI | Ceph clusters |

### Application Adapters

| Adapter | Protocol | Targets |
|---------|----------|---------|
| K8s | API | Kubernetes workloads |
| Nomad | REST API | Nomad jobs |
| Systemd | D-Bus/CLI | Linux service management |
| Docker | REST API | Container lifecycle |
| Binary | SSH/SCP | Direct binary deployment |
| Helm | CLI/API | Kubernetes package management |

---

## Layer 4: Verification & Observability

This is what separates LOOM from every existing tool. LOOM doesn't just deploy — it **verifies**.

### Verification Pipeline

After every deployment, LOOM runs a verification pipeline:

```
1. Infrastructure Verification
   ├── Can the VM/container reach its gateway? (L3 connectivity)
   ├── Are the correct VLANs configured on the switch? (L2 verification)
   ├── Is the firewall allowing the expected traffic? (ACL verification)
   ├── Is DNS resolving correctly? (Name resolution)
   └── Are storage volumes mounted and writable? (Storage verification)

2. Application Verification
   ├── Is the process running? (Process check)
   ├── Is the health endpoint responding? (HTTP/TCP check)
   ├── Can it reach its dependencies? (Dependency check)
   ├── Is it registered in service discovery? (Registration check)
   └── Is it processing requests correctly? (Functional check)

3. Tenant Isolation Verification
   ├── Can Tenant A reach Tenant B? (Should be NO)
   ├── Can Tenant A reach its own services? (Should be YES)
   ├── Are firewall rules enforced bidirectionally? (ACL audit)
   └── Is network segmentation intact at every hop? (Path verification)

4. Performance Baseline
   ├── Latency between tiers (app → db, app → cache)
   ├── Throughput capacity (network bandwidth utilization)
   ├── Resource utilization (CPU, memory, disk, GPU)
   └── Cost attribution per component

5. Compliance Verification
   ├── Data residency requirements met?
   ├── Encryption at rest and in transit?
   ├── Access controls in place?
   └── Audit logging enabled?
```

### Continuous Observability

LOOM doesn't stop at deployment verification. It continuously monitors:

- **Health**: Is everything still working?
- **Cost**: Is spending within budget? Is there waste?
- **Performance**: Is latency increasing? Is throughput degrading?
- **Drift**: Has configuration changed from the declared state?
- **Security**: Are new vulnerabilities exposed? Are ACLs intact?

When issues are detected, LOOM doesn't just alert — it **acts**:
- Remediates known issues automatically
- Escalates unknown issues with full context
- Proposes optimization opportunities
- Executes approved teardowns when resources are idle

---

## Multi-Tenancy Model

LOOM is tenant-aware at every layer:

```
LOOM Tenant Model
├── Organization (top-level boundary)
│   ├── Tenant A
│   │   ├── Network Segment: VLAN 100, 10.1.0.0/24
│   │   ├── Firewall Zone: zone-tenant-a
│   │   ├── Compute Resources: [VM1, VM2, ... VM10]
│   │   ├── Applications: [App1, App2]
│   │   ├── Cost Budget: $5,000/month
│   │   └── Compliance: SOC2, data-us-only
│   ├── Tenant B
│   │   ├── Network Segment: VLAN 200, 10.2.0.0/24
│   │   ├── Firewall Zone: zone-tenant-b
│   │   ├── Compute Resources: [VM11, VM12, ... VM20]
│   │   ├── Applications: [App3, App4, App5]
│   │   ├── Cost Budget: $8,000/month
│   │   └── Compliance: HIPAA, data-encrypted
│   └── Shared Services
│       ├── Network Segment: VLAN 999, 10.255.0.0/24
│       ├── DNS, NTP, Monitoring
│       └── Accessible by all tenants (read-only)
```

The LLM Decision Engine understands tenancy:
- "Tenant A's VMs must not share physical hosts with Tenant B (compliance requirement)"
- "Shared services are reachable by all tenants but tenant-to-tenant traffic is blocked"
- "Tenant B is approaching its budget — recommend scaling down non-critical workloads"

---

## Lifecycle Management

Every resource provisioned by LOOM has lifecycle semantics:

```yaml
lifecycle:
  created_by: "request-2024-001"
  created_at: "2024-03-20T10:00:00Z"

  # Automatic teardown triggers
  teardown:
    on_idle: 24h          # Tear down if idle for 24 hours
    on_date: "2024-04-20"  # Tear down on this date
    on_budget: $5000       # Tear down when budget exhausted
    on_completion: true     # Tear down when parent workflow completes

  # Capacity reservation
  reservation:
    gpu_a100: 4            # Reserve 4 A100 GPUs
    window: "02:00-06:00"  # During this time window
    release_after: true     # Release reservation after window

  # Dependencies (tear down in order)
  depends_on:
    - application-tier      # Tear down apps first
    - compute-tier          # Then VMs
    - network-tier          # Then network config
    - firewall-rules        # Then firewall rules (last)
```

---

## Technology Decisions

### Why LLM-Driven (Not Rule-Based)

| Challenge | Rule-Based Approach | LLM Approach |
|-----------|-------------------|--------------|
| Place 30 VMs optimally | Bin-packing algorithm | Considers cost, latency, failure domains, tenant isolation, compliance, AND current spot market conditions |
| Handle unknown device | Fail / require manual config | Identify device type, select appropriate adapter, generate config |
| Review request for oversights | Checklist validation | Contextual reasoning about security, dependencies, and edge cases |
| Cross-vendor network config | Per-vendor templates | Generate vendor-specific config from universal intent |
| Cost optimization | Threshold-based alerts | Continuous reasoning about tradeoffs across all dimensions |

### Why Adapter-Based (Not Monolithic)

- **Extensibility**: New device types added by writing an adapter, not modifying core
- **Isolation**: A buggy MikroTik adapter doesn't crash the Cisco adapter
- **Community**: Third parties can contribute adapters for niche devices
- **Testing**: Each adapter testable independently against device simulators

### Why Verification-First

- **Trust**: Operators trust LOOM because it proves its work
- **Compliance**: Auditors get machine-verified deployment reports
- **Reliability**: Issues caught at deployment time, not at 3AM
- **Intelligence**: Verification data feeds back into the LLM for better future decisions
