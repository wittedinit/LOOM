The complete research document was already returned in my previous response. Here it is again in full:

---

# Cloud-Native Container & Workload Orchestrators: Comprehensive Research Report

## Purpose

This document provides a thorough investigation of the major cloud-native container and workload orchestrators, analyzing their capabilities, strengths, weaknesses, and the collective gaps that remain unaddressed -- gaps that a universal orchestrator like LOOM is positioned to fill.

---

## 1. Kubernetes

### What It Does

Kubernetes is the dominant open-source container orchestration platform, originally developed by Google and now maintained by the CNCF. It organizes responsibilities through two layers: a **control plane** (storing cluster state and scheduling decisions) and **worker nodes** (executing containers and reporting status). It continuously compares desired state with actual state and auto-corrects drift through its controller reconciliation loop.

### Key Strengths

- **Ecosystem dominance**: The largest community, tooling ecosystem, and cloud provider support of any orchestrator. Every major cloud offers managed K8s (EKS, GKE, AKS).
- **Declarative model**: Self-correcting reconciliation loops ensure workloads converge to desired state automatically.
- **Multi-cloud abstraction**: Serves as the portability layer across cloud providers, reducing vendor lock-in.
- **AI/ML momentum**: 66% of organizations hosting generative AI models use Kubernetes for inference workloads. Projects like llm-d, KServe, Volcano, and Kueue are purpose-built for GPU-accelerated and distributed AI workloads.
- **Extensibility**: CRDs, operators, and a rich API allow nearly infinite customization.
- **Security improvements**: K8s v1.32+ introduces user namespaces, pod certificates for mTLS, and integrated policy enforcement moving toward zero-trust by default.

### Weaknesses and Gaps

- **Operational complexity**: Setting up, configuring, and managing Kubernetes remains time-consuming and resource-intensive. 25% of developers report Kubernetes has actually made cost management *worse*.
- **No native cost awareness**: Kubernetes has zero built-in understanding of cloud costs, instance pricing, or resource economics. Cost management requires third-party tools (Kubecost, Cast AI, Finout, Vantage).
- **No automated teardown semantics**: There is no native concept of "this workload should be torn down when idle for X minutes" or "this environment expires at date Y." TTL controllers exist for finished Jobs, but not for general workloads.
- **No network device orchestration**: Kubernetes orchestrates containers, not switches, routers, or firewalls. Bare-metal networking (MetalLB, Calico, BGP peering) requires deep manual configuration of physical network gear. K8s has no model for managing network infrastructure as a schedulable resource.
- **Bare metal is hard**: Running K8s on bare metal requires solving load balancing (MetalLB), storage provisioning, and physical network integration manually -- problems that cloud providers abstract away.
- **Scheduling limitations for AI scale**: The pod-centric scheduling model struggles with gang scheduling, topology-aware placement, and multi-node GPU jobs without add-on schedulers.
- **No unified workflow orchestration**: K8s orchestrates infrastructure, not business workflows. Tools like Argo Workflows or Temporal are needed for application-level orchestration.

### Multi-Cloud/Hybrid Support

Strong. K8s is the de facto multi-cloud abstraction layer, but each cluster is an island -- cross-cluster scheduling, failover, and cost arbitrage require additional tooling (KubeFed, Liqo, Admiralty).

### LLM/AI Integration

Rapidly evolving. KubeIntellect demonstrates LLM-powered Kubernetes management via natural language. llm-d v0.5 provides disaggregated inference with cache-aware routing and scale-to-zero. However, these are bolt-on projects, not native capabilities.

---

## 2. HashiCorp Nomad

### What It Does

Nomad is a flexible, lightweight workload orchestrator that runs as a single binary. It schedules containers (Docker, Podman), standalone executables, Java applications, batch jobs, and virtual machines (QEMU) across heterogeneous infrastructure.

### Key Strengths

- **Simplicity**: Single binary, no external dependencies (no etcd, no separate database). Drastically lower operational overhead than Kubernetes.
- **Multi-workload support**: Natively handles containers, VMs, raw executables, and batch jobs -- not just containers.
- **Bare metal native**: Runs seamlessly across bare metal, OpenStack, VMware, and public clouds without adaptation.
- **Efficient bin-packing**: Advanced bin-packing and anti-affinity algorithms maximize resource utilization.
- **HashiCorp ecosystem**: Deep integration with Vault (secrets), Consul (service discovery/mesh), and Terraform (provisioning).
- **GPU support**: Built-in GPU workload scheduling for ML/AI use cases.
- **Scale**: Capable of scheduling thousands of tasks per second.

### Weaknesses and Gaps

- **Smaller ecosystem**: Significantly fewer third-party integrations, operators, and community plugins compared to Kubernetes.
- **Limited networking**: No native service mesh, ingress controller, or advanced network policies. Relies on Consul for service discovery.
- **No cost awareness**: No built-in understanding of cloud pricing, spot instances, or resource economics.
- **No network device orchestration**: Like Kubernetes, Nomad orchestrates workloads, not network infrastructure.
- **Weaker monitoring/UI**: Built-in observability and dashboard capabilities are considered less mature.
- **No automated teardown**: No native concept of environment expiration or cost-driven cleanup.
- **Uncertain future post-IBM**: HashiCorp's acquisition by IBM and the BSL license change have introduced uncertainty about Nomad's open-source trajectory.
- **No LLM/AI-specific features**: No built-in inference serving, model management, or AI-aware scheduling beyond basic GPU allocation.

### Multi-Cloud/Hybrid Support

Good. Nomad federates across regions and datacenters natively. However, it lacks the ecosystem depth for cross-cloud service mesh, DNS, and load balancing that Kubernetes has.

---

## 3. Docker Swarm

### What It Does

Docker Swarm is Docker's native clustering and orchestration solution, built directly into the Docker Engine. It turns a pool of Docker hosts into a single virtual host with declarative service definitions, overlay networking, and mutual TLS between nodes.

### Key Strengths

- **Extreme simplicity**: Can go from zero to production orchestration in hours. Uses standard Docker CLI -- no new tools to learn.
- **Built-in security**: Mutual TLS encryption and certificate rotation between nodes by default.
- **Low barrier to entry**: Ideal for small teams, startups, and moderate-scale deployments.
- **Native Docker integration**: No translation layer between development (docker-compose) and production (swarm services).

### Weaknesses and Gaps

- **Effectively stalled**: Active feature development has largely stopped. Docker Inc. has shifted focus to Docker Desktop and Kubernetes integration.
- **Scalability ceiling**: Not suited for large clusters (hundreds of nodes) or complex workloads.
- **No advanced networking**: No network policies, no service mesh, no sophisticated traffic management.
- **No multi-cloud/hybrid**: Designed for single-cluster operation with no federation or cross-cloud capabilities.
- **No cost awareness**: Zero cost management capabilities.
- **No bare metal sophistication**: Basic bare-metal support but no integration with physical network infrastructure.
- **No network device orchestration**: Cannot manage switches, routers, or any network hardware.
- **No AI/LLM support**: No GPU scheduling, no inference serving, no AI-aware features.
- **No automated teardown**: No TTL, expiration, or idle-based cleanup semantics.
- **Declining community**: Shrinking user base and ecosystem as the industry has consolidated around Kubernetes.

### Multi-Cloud/Hybrid Support

Effectively none. Single-cluster only.

---

## 4. Apache Mesos / Marathon

### What It Does

Mesos was a datacenter-level resource manager that abstracted CPU, memory, storage, and other compute resources away from individual machines, enabling elastic and fault-tolerant distributed systems. Marathon was its container orchestration framework.

### Current Status

**Fully deprecated.** Mesos was retired in August 2025 and moved to the Apache Attic in October 2025. Marathon's GitHub repository is archived under d2iq-archive. All development has ceased.

### Historical Strengths

- **Datacenter-scale resource abstraction**: Could manage tens of thousands of nodes as a single resource pool.
- **Multi-framework**: Supported multiple scheduling frameworks simultaneously (Marathon for long-running services, Chronos for batch, Spark for analytics).
- **Two-level scheduling**: Unique architecture where Mesos offered resources to frameworks that could accept or reject them.
- **Proven at scale**: Used by Twitter, Airbnb, Apple, and Uber at massive scale.

### Why It Failed

- **Complexity**: Two-level scheduling was powerful but difficult to operate and reason about.
- **Kubernetes momentum**: K8s ecosystem growth made Mesos increasingly irrelevant.
- **Mesosphere/D2iQ pivot**: The primary commercial backer pivoted away from Mesos to Kubernetes.
- **Community collapse**: Contributor base dwindled as organizations migrated to K8s.

### Relevance to LOOM

Mesos' vision of datacenter-as-a-computer -- abstracting heterogeneous resources into a unified pool -- remains architecturally relevant. Its failure was one of ecosystem and timing, not concept. LOOM can learn from both Mesos' ambition and its failure modes.

---

## 5. Red Hat OpenShift

### What It Does

OpenShift is Red Hat's enterprise Kubernetes platform, adding developer tooling, CI/CD pipelines, a container registry, service mesh (Istio-based), and hardened security defaults on top of upstream Kubernetes. Now owned by IBM.

### Key Strengths

- **Enterprise security**: Security Context Constraints (SCCs), integrated image scanning, and RBAC out of the box. Hardened beyond upstream K8s defaults.
- **Developer experience**: Built-in CI/CD (Tekton pipelines), source-to-image (S2I) builds, and a developer console.
- **Hybrid cloud**: OpenShift runs on AWS, Azure, GCP, bare metal, and VMware with a consistent experience.
- **Red Hat support**: Enterprise-grade 24/7 support, certified operator catalog, and regular security patches.
- **VM integration**: OpenShift Virtualization (KubeVirt-based) allows running VMs alongside containers.
- **AI focus**: OpenShift 4.20 introduces capabilities specifically for accelerating AI workloads.

### Weaknesses and Gaps

- **Cost**: Premium licensing is expensive. Infrastructure requirements are heavy (minimum 3 control plane + 2 worker nodes).
- **Complexity**: Far from base Kubernetes -- steep learning curve and deep Red Hat ecosystem knowledge required.
- **Vendor lock-in**: Heavy reliance on Red Hat-specific tooling and patterns.
- **Limited autoscaling intelligence**: Autoscaling is limited to CPU and memory metrics; no cost-aware or business-metric-driven scaling.
- **No network device orchestration**: Cannot manage physical network infrastructure.
- **No cost awareness**: No built-in FinOps or cost optimization. Requires external tools.
- **No automated teardown**: No native environment expiration or idle-based cleanup.
- **Multi-cluster management gaps**: Advanced multi-cluster management (ACM) exists but is an additional paid product with its own complexity.

### Multi-Cloud/Hybrid Support

Strong but expensive. Available on all major clouds and bare metal, but each deployment requires significant infrastructure investment.

---

## 6. Rancher (SUSE)

### What It Does

Rancher is a multi-cluster Kubernetes management platform that provides a centralized control plane for provisioning, managing, and monitoring Kubernetes clusters across any infrastructure -- cloud, on-prem, bare metal, or edge.

### Key Strengths

- **Multi-cluster management at scale**: Fleet can manage up to 1 million clusters. Unified RBAC, monitoring, and policy across all clusters.
- **Cloud-agnostic**: Manages EKS, GKE, AKS, RKE, K3s, and bare-metal clusters from a single pane of glass.
- **Open source**: Fully open-source core (no licensing cost). Commercial support available from SUSE.
- **GitOps native**: Fleet provides GitOps-based deployment and configuration management at scale.
- **Edge support**: K3s (lightweight K8s) + Rancher enables edge Kubernetes at scale.
- **Low vendor lock-in**: Works with any conformant K8s distribution.

### Weaknesses and Gaps

- **Management plane overhead**: Production deployments require a dedicated 3-node HA cluster just for Rancher itself. Beyond 15-20 downstream clusters, external database offloading is recommended.
- **Not an application platform**: Rancher manages infrastructure, not application workflows. It does not solve deployment practices or distributed system design.
- **Upgrade fragility**: Rancher upgrades can be fragile and require dedicated operational expertise.
- **No cost awareness**: No built-in cost tracking, optimization, or cross-cluster cost comparison.
- **No network device orchestration**: Manages Kubernetes clusters, not network infrastructure.
- **No automated teardown**: No native concept of environment expiration, idle detection, or cost-driven cleanup.
- **No AI/LLM-specific features**: No inference serving, model management, or AI-aware scheduling.
- **OS management limitations**: Rancher struggles with managing the underlying operating system of nodes.

### Multi-Cloud/Hybrid Support

Excellent -- this is Rancher's primary value proposition. However, it provides visibility and management, not intelligent cross-cloud scheduling or cost arbitrage.

---

## 7. VMware Tanzu

### What It Does

VMware Tanzu (now under Broadcom) is an enterprise Kubernetes platform that integrates with VMware's vSphere infrastructure, providing Kubernetes clusters as a native vSphere capability. Includes Tanzu Kubernetes Grid (TKG), Tanzu Mission Control, and application platform services.

### Key Strengths

- **vSphere integration**: For VMware-native shops, Tanzu provides seamless Kubernetes on existing vSphere infrastructure.
- **Enterprise lifecycle management**: Automated cluster provisioning, patching, and upgrades within the VMware ecosystem.
- **Multi-cloud Kubernetes**: TKG runs on vSphere, AWS, and Azure with a consistent Kubernetes footprint.
- **Application platform**: Built-in application deployment, service binding, and observability.

### Weaknesses and Gaps

- **Broadcom acquisition chaos**: Since Broadcom's November 2023 acquisition of VMware, Tanzu has been caught in massive pricing upheaval. Reported cost increases of 150% to over 1,000%. Perpetual licenses eliminated. Mandatory bundling forces customers to buy Tanzu even if they do not need it.
- **Minimum core requirements**: Starting April 2025, a minimum 72-core license subscription is enforced (up from 16 cores), disproportionately harming SMBs and edge deployments.
- **SaaS EOL**: Tanzu Mission Control SaaS was end-of-lifed in November 2025, forcing migration to self-managed.
- **VMware lock-in**: Deeply coupled to the VMware ecosystem. Limited value outside vSphere environments.
- **Customer exodus**: Major customers (including Tesco, which sued Broadcom) are actively migrating away.
- **No cost awareness**: No built-in FinOps capabilities. Ironic given the platform's own cost controversy.
- **No network device orchestration**: Manages VMs and containers, not network hardware.
- **No automated teardown**: No native environment expiration or idle-based cleanup.
- **No AI/LLM-specific features**: Limited AI workload support compared to cloud-native Kubernetes distributions.
- **Uncertain roadmap**: Broadcom's aggressive monetization strategy raises questions about long-term investment in the platform.

### Multi-Cloud/Hybrid Support

Moderate. Works on vSphere, AWS, and Azure, but the value proposition is heavily weighted toward VMware-centric environments. The Broadcom pricing changes have undermined the platform's competitiveness.

---

## Comparative Matrix

| Capability | K8s | Nomad | Swarm | Mesos | OpenShift | Rancher | Tanzu |
|---|---|---|---|---|---|---|---|
| **Container orchestration** | Excellent | Good | Basic | Deprecated | Excellent | Via K8s | Via K8s |
| **Multi-workload (VMs, exec, batch)** | Limited | Excellent | No | Yes | Good (VMs) | No | Limited |
| **Bare metal support** | Manual/hard | Native | Basic | Yes | Yes | Via K8s | Via vSphere |
| **Multi-cloud/hybrid** | Good | Good | No | N/A | Good | Excellent | Moderate |
| **Cost awareness** | None | None | None | N/A | None | None | None |
| **Network device orchestration** | None | None | None | None | None | None | None |
| **Automated teardown/TTL** | Jobs only | No | No | N/A | No | No | No |
| **LLM/AI integration** | Emerging | Basic GPU | None | N/A | Emerging | None | Limited |
| **Operational simplicity** | Low | High | Very High | Low | Low | Medium | Medium |
| **Single pane of glass** | No | No | No | N/A | Partial | Yes (K8s only) | Partial |
| **Workflow orchestration** | No | Basic | No | N/A | CI/CD only | No | Limited |
| **Community/ecosystem** | Massive | Small | Declining | Dead | Large | Medium | Shrinking |

---

## Gaps Analysis: What ALL These Tools Collectively Fail to Address

Despite the maturity and breadth of the existing orchestration landscape, several fundamental gaps persist across every tool examined. These gaps define the opportunity space for LOOM's universal orchestration vision.

### 1. No Universal Resource Model

Every orchestrator defines its own narrow resource universe. Kubernetes sees pods and nodes. Nomad sees tasks and allocations. None of them can model a switch, a router, a bare-metal server, a cloud VM, a serverless function, a database instance, and a container workload within a single, unified resource graph. Organizations are forced to maintain separate control planes for compute, network, storage, and platform concerns -- with no cross-domain scheduling intelligence.

**LOOM opportunity**: A universal resource model where any manageable entity -- container, VM, network device, cloud service, bare-metal host -- is a first-class schedulable object with uniform lifecycle semantics.

### 2. Zero Native Cost Awareness

Not a single orchestrator examined has built-in understanding of what resources cost. Kubernetes does not know that an m5.xlarge costs $0.192/hr while a spot instance costs $0.06/hr. Nomad's bin-packing optimizes utilization, not spend. Cost management is universally delegated to third-party tools (Kubecost, Cast AI, Finout) that observe but do not *control* scheduling decisions.

**LOOM opportunity**: Cost as a first-class scheduling dimension. The orchestrator should natively understand pricing across providers, factor cost into placement decisions, and continuously rebalance workloads toward cheaper capacity without sacrificing availability or performance.

### 3. No Automated Teardown or Lifecycle Semantics

None of these tools have a native concept of "this environment should be destroyed after 72 hours," "tear down idle dev clusters nightly," or "this workload's budget is $50/month -- stop it when exhausted." Kubernetes has TTL for finished Jobs, but nothing for running workloads, namespaces, or environments. Cleanup is universally manual or requires custom scripting and external tools.

**LOOM opportunity**: First-class lifecycle policies -- TTL, budget caps, idle detection, scheduled teardown, and capacity reservations with automatic release -- built into the orchestration primitive itself, not bolted on.

### 4. No Network Infrastructure Orchestration

This is the most glaring and universal gap. Every tool examined orchestrates compute workloads exclusively. None can provision a VLAN on a switch, configure a route on a router, deploy an ACL on a firewall, or manage a load balancer as part of a unified deployment. Network automation exists (Ansible, Itential, Cisco NSO) but lives in an entirely separate silo from workload orchestration. Gartner's 2025 Market Guide explicitly calls out this "orchestration gap" where automated pieces live in silos.

**LOOM opportunity**: Treat network devices as orchestrable resources alongside compute. A deployment that needs "3 containers + a VLAN + a firewall rule + a BGP peer" should be expressible as a single declarative intent, with the orchestrator handling provisioning across both compute and network domains.

### 5. No LLM-Driven Heuristic Intelligence

While Kubernetes is adding AI workload *support* (running LLMs), no orchestrator uses AI/LLM intelligence for its own *decision-making*. Scheduling remains rule-based: affinity/anti-affinity, resource requests, topology constraints. No orchestrator can reason about "this workload pattern looks like a batch job that will finish in ~2 hours, so place it on preemptible capacity" or "based on historical patterns, this service needs 3x resources on Mondays." KubeIntellect and similar projects are research prototypes, not production schedulers.

**LOOM opportunity**: LLM-driven heuristics for placement, scaling, cost optimization, and anomaly detection. The orchestrator should learn from operational history and make intelligent decisions that go beyond static rules -- understanding workload behavior patterns, predicting demand, and proactively optimizing.

### 6. No True Single Pane of Glass Across Resource Types

Rancher provides a single pane for Kubernetes clusters. OpenShift provides a single pane for its own platform. But no tool provides unified visibility and control across containers, VMs, bare-metal servers, network devices, cloud services, and costs in a single interface. Organizations typically operate 3-5 separate management planes for different infrastructure domains.

**LOOM opportunity**: A genuinely unified control plane where an operator can see and manage every resource -- from a container in AWS to a switch port in a colo to a bare-metal GPU server -- through a single interface with consistent semantics.

### 7. No Availability/Reachability-Aware Scheduling

Orchestrators schedule based on resource capacity (CPU, memory, GPU) but have no native understanding of network reachability, latency, or availability zone economics. They cannot express "place this workload where it has sub-5ms latency to the database" or "avoid regions where the network path traverses a congested peering point." Cloud-specific features (topology spread constraints) are primitive approximations.

**LOOM opportunity**: Reachability and latency as first-class scheduling constraints. The orchestrator should understand network topology, measure real-time path quality, and factor connectivity into placement decisions alongside compute capacity and cost.

### 8. No Capacity Scheduling with Budget-Aware Preemption

Kubernetes has priority-based preemption and resource quotas, but these are static and cost-unaware. No orchestrator can express "this team has a $10K/month budget -- schedule their workloads within that envelope, preempting lower-priority work if needed, and alert when 80% consumed." Capacity planning and budget enforcement are entirely separate concerns managed by different teams with different tools.

**LOOM opportunity**: Unified capacity scheduling that binds compute capacity to budgets, enforces spending limits as a scheduling constraint, and provides automated preemption and teardown when budgets are exhausted.

---

## Conclusion

The container orchestration landscape is mature for its original problem domain -- scheduling containers across clusters -- but remains remarkably primitive when measured against the full scope of modern infrastructure orchestration needs. Every tool examined operates within a narrow resource model (containers, sometimes VMs), ignores cost as a scheduling dimension, has no lifecycle automation, cannot touch network infrastructure, and relies on static rules rather than intelligent heuristics.

LOOM's vision of universal orchestration -- encompassing compute, network, and platform resources with LLM-driven intelligence, native cost awareness, automated lifecycle management, and reachability-aware scheduling -- addresses a gap that no existing tool, alone or in combination, currently fills. The opportunity is not to replace Kubernetes (which will remain the container scheduling substrate) but to provide the *meta-orchestration layer* that unifies all resource types, all providers, and all operational concerns into a single, intelligent control plane.

---

### Sources

- [Kubernetes 2025 Review & 2026 Forecast (Arcfra)](https://www.arcfra.com/blog/kubernetes_2025_review_2026_forecast)
- [2026 Kubernetes Playbook (Fairwinds)](https://www.fairwinds.com/blog/2026-kubernetes-playbook-ai-self-healing-clusters-growth)
- [4 Trends Transforming Kubernetes in 2026 (InformationWeek)](https://www.informationweek.com/it-infrastructure/4-trends-that-will-transform-kubernetes-in-2026)
- [Nomad: The Workload Orchestrator You Might Be Overlooking (Medium)](https://medium.com/@bayounm95.eng/nomad-by-hashicorp-the-simple-powerful-workload-orchestrator-you-might-be-overlooking-ba568710d2d1)
- [Kubernetes vs Nomad (Overcast)](https://overcast.blog/kubernetes-vs-hashicorp-nomad-a-practical-comparison-d8308ef7c952)
- [Docker Swarm in 2025 (Medium)](https://medium.com/@niksa.makitan/docker-swarm-in-2025-0d2f2bc5d929)
- [Docker Swarm Limitations (LinkedIn)](https://www.linkedin.com/pulse/docker-swarm-limitations-nikhil-k-menon-pmp-csm-itil)
- [Apache Mesos Attic](https://attic.apache.org/projects/mesos.html)
- [Mesos Narrowly Avoids Attic (The New Stack)](https://thenewstack.io/apache-mesos-narrowly-avoids-a-move-to-the-attic-for-now/)
- [Red Hat OpenShift Pros and Cons 2026 (PeerSpot)](https://www.peerspot.com/products/red-hat-openshift-pros-and-cons)
- [OpenShift 4.20 Announcement (BusinessWire)](https://www.businesswire.com/news/home/20251111227520/en/Red-Hat-OpenShift-4.20-Enhances-Security-of-the-Modern-Application-Platform-to-Unite-Enterprise-IT-from-Virtual-Machines-to-AI)
- [Rancher Enterprise Kubernetes Management 2025 (BayTech)](https://www.baytechconsulting.com/blog/rancher-enterprise-kubernetes-management-2025)
- [Rancher in Practice: Managing 50+ Clusters](https://timderzhavets.com/blog/rancher-in-practice-managing-50-kubernetes-clusters/)
- [VMware Tanzu Platform (VMware)](https://www.vmware.com/products/app-platform/tanzu)
- [Broadcom VMware Acquisition Impact (NinjaOne)](https://www.ninjaone.com/blog/broadcom-vmware-acquisition/)
- [Post-Broadcom Reality for VMware Customers 2026 (Medium)](https://medium.com/@NickHystax/the-post-broadcom-reality-vmware-customers-face-in-2026-163ce4582d8f)
- [The Great Migration: AI Converging on Kubernetes (CNCF)](https://www.cncf.io/blog/2026/03/05/the-great-migration-why-every-ai-platform-is-converging-on-kubernetes/)
- [Running LLM Inference on Kubernetes: What Breaks First](https://dasroot.net/posts/2026/02/running-llm-inference-on-kubernetes-what-breaks-first/)
- [llm-d Architecture](https://llm-d.ai/docs/architecture)
- [Gartner 2025 Market Guide for Network Automation (Itential)](https://www.itential.com/blog/company/hybrid-cloud-automation/what-gartners-2025-market-guide-means-for-network-leaders-why-its-time-to-rethink-your-automation-strategy/)
- [FinOps Tools for Kubernetes Cost Management 2026 (Amnic)](https://amnic.com/blogs/finops-tools-for-kubernetes-cost-management)
- [Rancher vs OpenShift vs Tanzu Comparison (TechTarget)](https://www.techtarget.com/searchitoperations/post/Compare-Rancher-vs-OpenShift-vs-Tanzu-for-Kubernetes)
- [MetalLB: Bare Metal Load Balancer for Kubernetes](https://metallb.io/)