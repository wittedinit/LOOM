# LOOM: Request-to-Device Flow

## The End-to-End Orchestration Path

This document describes the complete flow from a user request to physical device configuration — every layer LOOM must touch, every decision it must make, and every verification it must perform.

---

## The Challenge No Tool Solves Today

**User Request**: "Build me a workflow environment with 30 VMs, 5 applications, 3 network segments, proper firewall rules, across mixed infrastructure."

**What actually needs to happen**:

```
User Request
    │
    ▼
Intent Understanding (What do they actually want?)
    │
    ▼
Oversight Detection (What did they forget to specify?)
    │
    ▼
Resource Planning (Where should things go? What does it cost?)
    │
    ▼
Network Infrastructure (VLANs, subnets, routing, firewall rules)
    │
    ▼
Compute Infrastructure (Bare metal power-on, VM creation, OS install)
    │
    ▼
Application Deployment (Binaries, configs, services)
    │
    ▼
End-to-End Verification (Does EVERYTHING work?)
    │
    ▼
Continuous Monitoring (Is it STILL working?)
    │
    ▼
Lifecycle Management (When does it get torn down?)
```

**No existing tool handles more than 1-2 of these steps.** LOOM handles all of them.

---

## Phase 1: Intent Understanding & Decomposition

### Input Processing

The user submits a request. LOOM's LLM engine parses it into structured intent:

```yaml
# User says: "I need 30 VMs across 3 tenants with web, app, and DB tiers,
# proper segmentation, firewalls between segments, running on our mixed
# infrastructure (some bare metal, some VMware, some Proxmox)"

intent:
  type: workflow_environment
  tenants:
    - name: production
      vm_count: 15
      tiers: [web, app, db]
    - name: staging
      vm_count: 10
      tiers: [web, app, db]
    - name: dev
      vm_count: 5
      tiers: [web, app]

  applications:
    - name: frontend (web tier)
    - name: api-gateway (web tier)
    - name: business-logic (app tier)
    - name: data-processor (app tier)
    - name: postgres (db tier)

  requirements:
    network_segmentation: per-tenant AND per-tier
    firewall_rules: strict (web→app allowed, app→db allowed, web→db blocked)
    os_mix: [ubuntu-22.04, rhel-9, windows-server-2022]
    infrastructure: [bare-metal, vmware, proxmox]
```

### Oversight Detection

The LLM reviews the decomposed intent and flags oversights:

```
⚠️  OVERSIGHT DETECTED: No DNS specification
    → Recommendation: Deploy internal DNS per tenant with cross-tenant resolution
       for shared services only

⚠️  OVERSIGHT DETECTED: No backup/snapshot policy specified
    → Recommendation: Daily snapshots for DB tier, weekly for app tier

⚠️  OVERSIGHT DETECTED: DB tier has no persistence specification
    → Recommendation: Attach dedicated storage volumes to DB VMs

⚠️  OVERSIGHT DETECTED: No load balancer specified for web tier
    → Recommendation: Deploy HAProxy per tenant for web tier

⚠️  OVERSIGHT DETECTED: Windows Server 2022 requested but no licensing
       method specified
    → Question: Do you have KMS, MAK, or AVMA licensing? This affects
       provisioning flow.

⚠️  SECURITY CONCERN: No mention of encryption requirements
    → Recommendation: Enable TLS between all tiers, encrypt storage at rest
       for DB tier
```

**This is what no existing tool does.** Terraform applies what you wrote. Ansible runs what you wrote. LOOM *questions* what you wrote.

---

## Phase 2: Resource Planning & Scheduling

### Infrastructure Discovery

LOOM queries its inventory to understand available resources:

```
Available Infrastructure:
├── Bare Metal Servers (8 available)
│   ├── BM-01 to BM-04: Dell PowerEdge R750 (Redfish API, iDRAC 9)
│   ├── BM-05 to BM-06: Supermicro SYS-620U (IPMI, no Redfish)
│   └── BM-07 to BM-08: Intel NUC (Intel AMT, no IPMI, no Redfish)
│
├── VMware Cluster (vCenter API)
│   ├── ESXi-01: 128GB RAM, 64 cores, 40% utilized
│   ├── ESXi-02: 128GB RAM, 64 cores, 60% utilized
│   └── ESXi-03: 256GB RAM, 128 cores, 20% utilized
│
├── Proxmox Cluster (REST API)
│   ├── PVE-01: 64GB RAM, 32 cores, 30% utilized
│   └── PVE-02: 64GB RAM, 32 cores, 10% utilized
│
├── Network Infrastructure
│   ├── Core: Arista 7280R (EOS, eAPI)
│   ├── Spine: 2x Dell S5248F-ON (SONiC, gNMI)
│   ├── Leaf: 4x Edgecore AS7726-32X (SONiC, gNMI)
│   ├── Leaf: 2x Cisco Catalyst 9300 (IOS-XE, NETCONF)
│   ├── Firewall: OPNsense HA pair (REST API)
│   └── Edge: MikroTik RB4011 (RouterOS REST API)
│
├── Remote Management
│   ├── BM-01 to BM-04: iDRAC accessible via Redfish
│   ├── BM-05 to BM-06: IPMI accessible via ipmitool
│   ├── BM-07 to BM-08: Intel AMT via WS-Management
│   └── PiKVM-01: Managing BM-07 console (REST API fallback)
│
└── Storage
    ├── TrueNAS: 100TB available (NFS/iSCSI)
    └── Ceph: 50TB available (RBD/CephFS)
```

### LLM Placement Decision

The LLM reasons across all dimensions:

```
PLACEMENT DECISION:

Production Tenant (15 VMs, highest priority):
  DB Tier (3 VMs): Bare metal BM-01, BM-02, BM-03 (Redfish)
    Reason: DB workloads need dedicated I/O; bare metal eliminates
    hypervisor overhead. Dell iDRAC provides Redfish — reliable
    automated provisioning.

  App Tier (6 VMs): VMware cluster (ESXi-03, lowest utilization)
    Reason: App tier benefits from live migration and HA. ESXi-03
    has most headroom.

  Web Tier (6 VMs): Proxmox cluster (PVE-01 + PVE-02)
    Reason: Web tier is stateless — Proxmox is cost-effective.
    Spread across both nodes for HA.

Staging Tenant (10 VMs):
  All tiers: VMware cluster (ESXi-01 + ESXi-02)
    Reason: Staging doesn't need bare metal performance. VMware
    provides snapshot/rollback for testing workflows.

Dev Tenant (5 VMs):
  All tiers: Proxmox cluster (remaining capacity)
    Reason: Lowest priority, cost-optimized placement.

  NOTE: BM-07 and BM-08 (Intel AMT/NUC) excluded from VM hosting —
  insufficient resources. Available as management nodes or edge
  testing targets.

NETWORK PLACEMENT:
  Production VLANs → Leaf switches (SONiC on Edgecore) — best
    performance, gNMI telemetry
  Staging VLANs → Leaf switches (Cisco IOS-XE) — stable, NETCONF
    management
  Dev VLANs → Same Cisco switches — lower traffic, shared
    infrastructure acceptable
  Inter-tenant routing → Arista core (eAPI) with strict ACLs
  Firewall enforcement → OPNsense HA pair (REST API)

COST ESTIMATE:
  Infrastructure: $X/month (bare metal amortized + power + cooling)
  VMware licensing: $Y/month (per-socket)
  Network: $0 (owned infrastructure)
  Storage: $Z/month (TrueNAS power + drives amortized)
  Total: $TOTAL/month
```

---

## Phase 3: Network Infrastructure Provisioning

### The Multi-NOS Challenge

LOOM must configure 4 different network operating systems in a single deployment:

```
Step 3.1: VLAN Creation

  SONiC (Edgecore leaf switches) via gNMI:
    → Create VLAN 100 (prod-web), VLAN 101 (prod-app), VLAN 102 (prod-db)
    → Create VLAN 200 (staging-web), VLAN 201 (staging-app), VLAN 202 (staging-db)
    → Create VLAN 300 (dev-web), VLAN 301 (dev-app)
    → Assign ports to VLANs based on physical server connections
    → Configure trunk ports to spine switches

  Cisco IOS-XE (Catalyst 9300) via NETCONF:
    → Same VLANs, different syntax, different API
    → LOOM's adapter translates from universal model to IOS-XE YANG
    → Configure trunk ports to spine

  Arista EOS (core switch) via eAPI:
    → Configure inter-VLAN routing (SVIs)
    → BGP peering for route distribution
    → Different API format, same logical operation

  MikroTik RouterOS (edge) via REST API:
    → Configure NAT for external access
    → Different API paradigm entirely (path-based, not model-based)

Step 3.2: Firewall Rules

  OPNsense (firewall) via REST API:
    → Per-tenant zones
    → Inter-zone rules:
      - web → app: ALLOW ports 8080, 8443
      - app → db: ALLOW ports 5432 (Postgres), 3306 (MySQL)
      - web → db: DENY (blocked — must go through app tier)
      - tenant A → tenant B: DENY (full isolation)
      - tenant A → shared-services: ALLOW DNS, NTP, monitoring
      - shared-services → tenant A: ALLOW monitoring probes only
    → Logging enabled for all deny rules
    → Rate limiting on external-facing rules

Step 3.3: Subnet & DHCP Configuration

  Per-tenant, per-tier subnets:
    Production:  10.1.1.0/24 (web), 10.1.2.0/24 (app), 10.1.3.0/24 (db)
    Staging:     10.2.1.0/24 (web), 10.2.2.0/24 (app), 10.2.3.0/24 (db)
    Dev:         10.3.1.0/24 (web), 10.3.2.0/24 (app)
    Management:  10.255.0.0/24 (LOOM management plane — all devices)

  DHCP reservations for all VMs (static assignment via MAC)
  DNS entries created per VM hostname
```

### Adapter Translation Example

LOOM's universal model for "create VLAN 100 on a switch port":

```yaml
# LOOM Universal Model
resource:
  type: network.vlan
  id: vlan-100
  properties:
    vlan_id: 100
    name: "prod-web"
    ports:
      - interface: "Ethernet1/1"
        mode: access
```

**Translated by SONiC adapter (gNMI)**:
```json
{
  "openconfig-vlan:vlans": {
    "vlan": [{"vlan-id": 100, "config": {"vlan-id": 100, "name": "prod-web"}}]
  }
}
```

**Translated by Cisco IOS-XE adapter (NETCONF)**:
```xml
<config>
  <native xmlns="http://cisco.com/ns/yang/Cisco-IOS-XE-native">
    <vlan><vlan-list><id>100</id><name>prod-web</name></vlan-list></vlan>
    <interface><GigabitEthernet><name>1/0/1</name>
      <switchport><mode><access/></mode>
        <access><vlan><vlan>100</vlan></vlan></access>
      </switchport>
    </GigabitEthernet></interface>
  </native>
</config>
```

**Translated by Arista EOS adapter (eAPI)**:
```json
{
  "jsonrpc": "2.0",
  "method": "runCmds",
  "params": {
    "cmds": ["enable", "configure", "vlan 100", "name prod-web",
             "interface Ethernet1", "switchport access vlan 100"]
  }
}
```

**Translated by MikroTik adapter (REST API)**:
```
POST /rest/interface/bridge/vlan
{"bridge": "bridge1", "vlan-ids": "100", "tagged": "ether1",
 "comment": "prod-web"}
```

**Same intent. Four completely different implementations. LOOM handles all of them.**

---

## Phase 4: Compute Provisioning

### The Multi-Protocol Challenge

LOOM must provision servers using 4 different remote management protocols:

```
BM-01 to BM-04 (Dell with iDRAC/Redfish):
  1. Redfish API: Set next boot to PXE
  2. Redfish API: Power on
  3. PXE server: Serve OS installer (Ubuntu 22.04)
  4. Cloud-init: Post-install configuration
  5. Redfish API: Monitor sensor data during install

BM-05 to BM-06 (Supermicro with IPMI only):
  1. ipmitool: Set boot device to PXE
     $ ipmitool -I lanplus -H bm05-bmc -U admin chassis bootdev pxe
  2. ipmitool: Power on
     $ ipmitool -I lanplus -H bm05-bmc -U admin power on
  3. PXE server: Serve OS installer
  4. NOTE: No Redfish — limited monitoring during install
     → LOOM polls via IPMI sensors and SSH availability

BM-07 to BM-08 (Intel NUC with AMT):
  1. Intel AMT via WS-Management/SOAP:
     → Different protocol entirely (not Redfish, not IPMI)
     → AMT SDK or meshcommander for remote KVM
  2. If AMT provisioning fails → PiKVM fallback:
     → PiKVM-01 REST API: Send keyboard input for BIOS/boot menu
     → PiKVM-01 REST API: Take screenshot to verify boot progress
     → This is the "last resort" path for non-standard hardware
  3. PXE boot: Serve OS installer
  4. Post-install via SSH

VMware VMs (ESXi-01/02/03 via vCenter API):
  1. vSphere REST API: Create VM with spec
  2. vSphere REST API: Attach ISO / configure PXE
  3. vSphere REST API: Power on
  4. vSphere REST API: Monitor via guest tools
  5. OS install + cloud-init / unattended setup

Proxmox VMs (PVE-01/02 via REST API):
  1. Proxmox API: Create VM (qm create)
  2. Proxmox API: Attach cloud-init drive
  3. Proxmox API: Configure network (bridge + VLAN tag)
  4. Proxmox API: Start VM
  5. Proxmox API: Monitor via QEMU guest agent
```

### OS Diversity Handling

Different VMs may need different operating systems:

```
OS Provisioning Matrix:
├── Ubuntu 22.04 LTS
│   ├── Method: Cloud-init (autoinstall)
│   ├── Targets: Web tier, app tier
│   └── Post-install: apt update, deploy agent, configure firewall
│
├── RHEL 9
│   ├── Method: Kickstart
│   ├── Targets: DB tier (production)
│   └── Post-install: subscription-manager, deploy agent, SELinux config
│
├── Windows Server 2022
│   ├── Method: Unattended.xml (autounattend)
│   ├── Targets: Specific app tier VMs (legacy .NET application)
│   └── Post-install: WinRM enable, deploy agent, Windows Firewall rules
│   └── NOTE: Different firewall management (Windows Firewall API, not iptables)
│
└── LOOM handles ALL of these with OS-specific adapters
    └── Post-install agent provides uniform management regardless of OS
```

---

## Phase 5: Application Deployment

```
For each application, across each tenant:

1. Binary/Container Deployment
   ├── If containerized → K8s/Nomad deployment with tenant namespace
   ├── If binary → SCP to target, systemd unit creation
   └── If Windows service → WinRM deployment, service registration

2. Configuration Injection
   ├── Per-tenant database connection strings
   ├── Per-environment feature flags
   ├── Service discovery endpoints
   ├── TLS certificates (per-tenant CA)
   └── Logging/monitoring agent configuration

3. Service Registration
   ├── DNS record creation (per-tenant internal DNS)
   ├── Load balancer backend registration
   └── Monitoring target registration (Prometheus/Datadog)

4. Health Check Verification
   ├── HTTP health endpoint responds 200
   ├── Application logs show successful startup
   ├── Database connectivity confirmed
   └── Inter-service communication verified
```

---

## Phase 6: End-to-End Verification

**This is LOOM's most critical differentiator.** After every deployment, LOOM proves its work:

```
VERIFICATION REPORT — Request #2024-001
=========================================

NETWORK VERIFICATION ✅
├── ✅ VLAN 100 (prod-web) active on all leaf switches
│   ├── SONiC switches: verified via gNMI state query
│   └── Cisco switches: verified via NETCONF get
├── ✅ VLAN 101 (prod-app) active on all leaf switches
├── ✅ Inter-VLAN routing functional (Arista core)
│   └── Verified: traceroute from 10.1.1.0/24 → 10.1.2.0/24 = 2 hops
├── ✅ Trunk ports carrying all VLANs between leaf and spine
└── ✅ MikroTik NAT rules active for external access

FIREWALL VERIFICATION ✅
├── ✅ Tenant isolation enforced:
│   ├── Prod (10.1.x.x) → Staging (10.2.x.x): BLOCKED ✅
│   ├── Staging → Dev: BLOCKED ✅
│   └── Dev → Prod: BLOCKED ✅
├── ✅ Tier segmentation enforced:
│   ├── Web → App (port 8080): ALLOWED ✅
│   ├── App → DB (port 5432): ALLOWED ✅
│   ├── Web → DB (port 5432): BLOCKED ✅ (security requirement)
│   └── All → Shared Services (DNS/NTP): ALLOWED ✅
└── ✅ Logging active on all deny rules

COMPUTE VERIFICATION ✅
├── ✅ 30/30 VMs running and reachable via SSH/WinRM
├── ✅ OS verification:
│   ├── 22 VMs: Ubuntu 22.04 (kernel 5.15, cloud-init complete)
│   ├── 5 VMs: RHEL 9 (kernel 5.14, subscription active)
│   └── 3 VMs: Windows Server 2022 (build 20348, activated)
├── ✅ All VMs on correct VLANs (MAC/VLAN binding verified)
├── ✅ Storage volumes mounted:
│   ├── DB tier: NFS volumes from TrueNAS (/data mounted, read-write)
│   └── App tier: Local storage (sufficient capacity verified)
└── ✅ LOOM agent installed and reporting on all 30 VMs

APPLICATION VERIFICATION ✅
├── ✅ 5/5 applications deployed and healthy:
│   ├── frontend: HTTP 200 on :443, latency 12ms
│   ├── api-gateway: HTTP 200 on :8443, latency 8ms
│   ├── business-logic: gRPC health check SERVING
│   ├── data-processor: processing queue depth = 0 (idle, ready)
│   └── postgres: accepting connections, replication lag = 0ms
├── ✅ Service discovery: all services registered and resolvable
├── ✅ TLS: all inter-service communication encrypted (cert valid 365 days)
└── ✅ Monitoring: all 30 VMs and 5 apps visible in monitoring stack

COST ATTRIBUTION
├── Production tenant: $X/month (bare metal + VMware + storage)
├── Staging tenant: $Y/month (VMware only)
├── Dev tenant: $Z/month (Proxmox only)
├── Network infrastructure: amortized, shared
└── Total: $TOTAL/month

LIFECYCLE
├── Teardown schedule: None set (production)
├── Staging auto-teardown: Fridays at 18:00 (rebuild Monday 06:00)
└── Dev auto-teardown: Idle > 72 hours

OVERSIGHTS ADDRESSED
├── ✅ DNS: Internal DNS deployed per tenant (as recommended)
├── ✅ Backups: Daily snapshots configured for DB tier
├── ✅ Load balancer: HAProxy deployed per tenant for web tier
├── ⚠️ Windows licensing: KMS configured (user confirmed)
└── ✅ Encryption: TLS enabled between all tiers

CONFIDENCE: 100% — All 147 verification checks passed.
```

---

## The Feedback Loop

Verification data feeds back into the LLM Decision Engine:

```
LESSON LEARNED (stored for future decisions):
  - BM-05 (Supermicro/IPMI) took 45 minutes to provision vs 12 minutes
    for BM-01 (Dell/Redfish). Factor IPMI overhead into scheduling.
  - Cisco IOS-XE NETCONF took 8 seconds per VLAN vs 2 seconds for
    SONiC gNMI. Prefer SONiC switches for large-scale VLAN deployments.
  - Windows Server 2022 unattended install required 3 retries due to
    driver issue. Cache working driver pack for future deployments.
  - PiKVM fallback for Intel AMT worked but added 20 minutes. Prefer
    Redfish-capable hardware for automated provisioning.
```

---

## Summary: Why This Flow Is Unique

| Step | Existing Tools | LOOM |
|------|---------------|------|
| Intent parsing | Manual YAML/JSON | LLM-powered with natural language |
| Oversight detection | None | LLM questions missing requirements |
| Placement decisions | Static rules | LLM reasons across cost/availability/reachability |
| Network config | Per-vendor manual | Universal model → auto-translated per NOS |
| Bare metal provisioning | Per-protocol manual | Auto-detects Redfish/IPMI/AMT/PiKVM |
| OS diversity | Separate tools per OS | Unified provisioning across Linux/Windows |
| Multi-tenant isolation | Manual config | First-class tenant model with verification |
| Firewall rules | Manual per device | Declarative intent → device-specific rules |
| Application deployment | Separate tool | Integrated in orchestration flow |
| Verification | Manual / hope | Automated 147-point verification |
| Cost tracking | Separate tool | Inline, per-tenant, per-resource |
| Teardown | Manual / forgotten | Automated lifecycle with budget triggers |
| Learning | None | Feedback loop improves future decisions |

**LOOM is the first system that takes a complex infrastructure request from intent to verified reality across heterogeneous infrastructure — questioning oversights, adapting to device diversity, and proving every element was delivered correctly.**
