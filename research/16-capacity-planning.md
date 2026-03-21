# Capacity Planning and Demand Forecasting Platforms: Comprehensive Research Report

## 1. Densify

**Core Capabilities:**
Densify is a machine learning-powered cloud and container resource optimization platform. It analyzes workload resource utilization patterns (CPU, memory, network, storage IOPS) across cloud VMs, Kubernetes containers, and on-prem virtual machines to recommend optimal instance types, container resource requests/limits, and reserved capacity purchases. Densify's ML models consider utilization patterns, performance requirements, cost, and cloud provider instance families to recommend right-sizing, instance family migration, and commitment purchases (Reserved Instances, Savings Plans). Densify integrates with CI/CD and IaC pipelines to automatically apply optimizations.

**Forecasting Approach:**
Densify uses proprietary ML algorithms trained on workload telemetry (CPU, memory, network, disk) to predict future resource demand. It factors in historical patterns, seasonal trends, and growth rates. Forecasting drives both right-sizing recommendations (what instance type/size to use) and commitment planning (how much reserved capacity to purchase). Densify also models "what-if" scenarios: what happens if workload X grows 50%? What instance types will be needed?

**Platform Coverage:**
- **Cloud:** AWS, Azure, GCP (VM instance optimization, commitment management)
- **Kubernetes:** Container resource request/limit optimization
- **VMware:** On-prem virtual machine right-sizing
- **No bare metal, no network, no storage orchestration**

**Strengths:**
- Deep ML-based resource analysis -- recommendations are workload-specific, not generic
- "What-if" scenario modeling for capacity planning
- Integrates with Terraform and CI/CD for automated optimization
- Handles both cloud and on-prem VMware environments
- Commitment management (RI/SP) driven by actual workload data, not billing data alone
- Container optimization considers K8s scheduling behavior and QoS classes
- Proven at enterprise scale with major financial institutions and telecom companies

**Weaknesses:**
- Proprietary platform with enterprise pricing -- not accessible to small teams
- Cloud compute and K8s focus -- no storage, network, or bare metal optimization
- Recommendations are within-platform -- cannot evaluate cross-platform tradeoffs (cloud vs. colo vs. bare metal)
- No integration with infrastructure orchestration -- recommendations are advisory
- Forecasting is based on historical utilization, not business demand signals
- No cost modeling for infrastructure beyond cloud compute and K8s
- Vendor-proprietary ML -- no transparency into algorithm decisions
- No awareness of network topology, data gravity, or latency constraints

**Scope:** Cloud VM and K8s container optimization. RI/SP commitment planning. VMware on-prem. No bare metal, no network, no storage, no cross-domain.

---

## 2. Turbonomic (IBM)

**Core Capabilities:**
Turbonomic is an Application Resource Management (ARM) platform that models the entire application stack as a "supply chain" of resources: applications consume containers, containers consume VMs/pods, VMs consume hosts, hosts consume storage and network. Turbonomic's economic scheduling engine continuously analyzes supply and demand across this stack and generates (or automates) actions: scale up/down, resize, move, provision, suspend. Turbonomic supports cloud (AWS, Azure, GCP), Kubernetes, VMware, Hyper-V, and integrates with Instana for application performance data.

**Forecasting Approach:**
Turbonomic's approach is real-time demand-driven rather than forecasting-based. It continuously evaluates resource supply and demand, generating actions when demand exceeds supply (scale up) or supply exceeds demand (scale down). For capacity planning, Turbonomic provides "Plan" mode which simulates future scenarios: "what if we add 500 VMs?" or "what if we migrate to AWS?" Plans model the resource impact across the full stack (compute, storage, network).

**Platform Coverage:**
- **Cloud:** AWS, Azure, GCP (VM, container, serverless optimization)
- **Kubernetes:** Pod and cluster optimization
- **VMware/Hyper-V:** On-prem virtualization
- **Storage:** Some storage array integration (NetApp, Pure, Dell) for capacity awareness
- **Network:** Limited network awareness through application topology
- **No bare metal compute orchestration**

**Strengths:**
- Full-stack resource modeling (application -> container -> VM -> host -> storage -> network)
- Economic scheduling engine makes optimization decisions autonomously
- Automated actions (not just recommendations) -- can execute resizing, migration, and scaling
- Integration with Instana provides application performance awareness
- Plan mode enables sophisticated capacity planning scenarios
- Broadest platform coverage (cloud, K8s, VMware, Hyper-V, storage)
- IBM backing with Cloudability integration for unified cost + resource management

**Weaknesses:**
- Complex enterprise platform with steep learning curve
- IBM acquisition has introduced enterprise sales complexity
- Storage integration is awareness-level, not storage orchestration
- Network modeling is limited to topology awareness, not network device configuration
- No bare metal compute optimization
- Economic model can generate excessive actions without careful tuning
- Automated actions can be aggressive -- requires trust in the algorithm
- No integration with infrastructure provisioning (Terraform, Ansible)
- Forecasting is scenario-based (plans), not continuous predictive modeling
- Expensive enterprise licensing

**Scope:** Full-stack resource management. Cloud, K8s, VMware. Storage awareness. No bare metal, no network orchestration, no IaC integration.

---

## 3. Flexera One (Cloud Management)

**Core Capabilities:**
Flexera One provides IT asset management, cloud cost management, and cloud migration planning in a unified platform. For capacity planning, Flexera offers cloud migration assessment (discover on-prem workloads, model cloud target sizing, estimate costs), cloud cost optimization (right-sizing, commitment management, waste identification), and SaaS management. Flexera acquired Spot.io's technology from NetApp, integrating cloud compute optimization capabilities. The platform supports multi-cloud (AWS, Azure, GCP) and hybrid environments.

**Forecasting Approach:**
Flexera's forecasting is primarily cost-oriented: projecting cloud spend trends, commitment utilization forecasts, and budget vs. actual tracking. For migration planning, Flexera models target cloud sizing based on current on-prem workload characteristics (CPU, memory, storage, network). Machine learning identifies optimization opportunities from utilization patterns. Flexera does not provide workload demand forecasting or predictive scaling.

**Platform Coverage:**
- **Cloud:** AWS, Azure, GCP (cost management, right-sizing, commitments)
- **On-prem:** Discovery and assessment for migration planning
- **SaaS:** License and usage management
- **No Kubernetes-native, no bare metal, no network, no storage orchestration**

**Strengths:**
- Broadest IT asset management coverage (cloud, on-prem, SaaS)
- Cloud migration planning with detailed cost modeling
- Multi-cloud cost management with commitment optimization
- Spot.io integration brings compute optimization capabilities
- SaaS management addresses shadow IT and license waste
- Strong enterprise presence with Fortune 500 adoption

**Weaknesses:**
- IT asset management heritage -- not a modern infrastructure optimization platform
- No Kubernetes-native capacity planning
- No bare metal or network capacity planning
- Forecasting is cost-based, not demand-based
- No integration with infrastructure orchestration
- No automated optimization actions (advisory only for most capabilities)
- Complex enterprise platform with long implementation timelines
- No real-time capacity optimization -- periodic assessment and recommendation
- Limited AI/ML compared to Densify or Turbonomic
- Migration planning is point-in-time, not continuous

**Scope:** IT asset management and cloud cost optimization. Migration planning. No K8s, no bare metal, no network, no real-time optimization.

---

## 4. CloudPhysics (VMware)

**Core Capabilities:**
CloudPhysics (acquired by VMware/Broadcom) provides data center analytics and cloud migration assessment for VMware environments. It collects workload data from vCenter (CPU, memory, storage, network utilization) and provides: right-sizing recommendations for VMs, capacity forecasting for ESXi hosts and datastores, cloud migration readiness assessment with target sizing and cost estimation, and "what-if" scenario planning for infrastructure changes. CloudPhysics uses ML-based analytics to identify optimization opportunities and predict capacity exhaustion.

**Forecasting Approach:**
CloudPhysics predicts capacity exhaustion dates for compute (CPU, memory), storage (datastore capacity, IOPS), and network (bandwidth) based on historical trends and ML models. Growth projections factor in seasonal patterns and linear/exponential growth. Migration assessments model cloud target sizing and cost for AWS, Azure, and GCP.

**Platform Coverage:**
- **VMware:** ESXi hosts, VMs, datastores, vSAN
- **Cloud (target):** AWS, Azure, GCP for migration assessment
- **No Kubernetes, no bare metal, no network devices, no non-VMware**

**Strengths:**
- Deep VMware integration -- most detailed VMware capacity analytics
- Capacity exhaustion prediction with specific date estimates
- Cloud migration assessment with target sizing and cost comparison
- "What-if" scenario planning for infrastructure changes
- ML-based analytics trained on massive VMware telemetry dataset

**Weaknesses:**
- VMware/Broadcom acquisition creates product direction uncertainty
- VMware-only -- no bare metal, no Kubernetes, no non-VMware infrastructure
- Cloud assessment is migration planning, not ongoing cloud capacity management
- No integration with infrastructure orchestration or provisioning
- No real-time optimization -- periodic analysis and recommendations
- No network capacity planning beyond VMware vSwitch utilization
- No storage capacity planning beyond VMware datastores
- Vendor lock-in to VMware ecosystem
- Product future is unclear under Broadcom ownership

**Scope:** VMware data center analytics and migration assessment. No K8s, no bare metal, no cloud-native, no orchestration.

---

## 5. Spot.io Forecasting (NetApp/Flexera)

**Core Capabilities:**
Spot.io (now part of Flexera, originally NetApp) provides AI-driven cloud infrastructure optimization with predictive capabilities. For capacity planning, Spot Ocean (Kubernetes) provides predictive autoscaling that anticipates workload demand and pre-provisions nodes before scaling events. Spot Elastigroup provides similar predictive scaling for VM-based workloads. Spot Eco manages reserved capacity (RIs/Savings Plans) with ML-based utilization forecasting. The platform continuously analyzes workload patterns to predict optimal instance types, scaling timing, and capacity reservation levels.

**Forecasting Approach:**
Spot uses ML algorithms trained on workload metrics to predict: scaling events (when additional capacity is needed), optimal instance types (based on workload resource patterns), spot instance availability (predicting spot market interruptions), and commitment utilization (forecasting RI/SP usage for optimal purchasing). Predictive autoscaling provisions capacity before demand spikes rather than reacting to them.

**Platform Coverage:**
- **Cloud:** AWS, Azure, GCP (compute optimization)
- **Kubernetes:** Ocean for K8s-native optimization
- **No on-prem, no bare metal, no network, no storage**

**Strengths:**
- Predictive autoscaling is genuinely differentiated -- provisions before demand
- Spot instance management with interruption prediction
- Real-time actuation -- not just recommendations, but automated scaling and instance selection
- Ocean provides Kubernetes-native predictive scaling
- ML-trained on massive cloud workload dataset
- 73% average compute cost reduction through optimization
- Integrates scaling decisions with cost optimization

**Weaknesses:**
- Cloud compute only -- no bare metal, no network, no storage capacity planning
- Acquired by Flexera -- product direction and integration uncertainty
- Predictions are workload-utilization-based, not business-demand-based
- No cross-platform capacity planning (cannot evaluate cloud vs. colo vs. bare metal)
- No integration with broader infrastructure orchestration
- No data center or on-prem capacity planning
- Limited to instance-level decisions -- no holistic infrastructure capacity modeling
- No awareness of network topology, data gravity, or latency constraints
- AI/ML is compute-focused -- does not extend to storage or network forecasting

**Scope:** Cloud compute predictive optimization. Kubernetes via Ocean. No on-prem, no bare metal, no network, no storage, no cross-domain.

---

## 6. Karpenter (Predictive Scaling)

**Core Capabilities:**
Karpenter is an open-source Kubernetes node autoscaler (originally from AWS, now a CNCF project) designed to replace the Kubernetes Cluster Autoscaler. Karpenter directly provisions cloud instances (without node groups/managed node groups) based on pod scheduling constraints: instance type, architecture, availability zone, and capacity type (on-demand, spot). Karpenter evaluates pending pods, calculates optimal instance types and counts, and provisions nodes in seconds. It supports node consolidation (binpacking running pods onto fewer nodes) and node disruption (replacing nodes with better options).

**Forecasting Approach:**
Karpenter is primarily reactive (responds to pending pods) rather than predictive. However, its design enables faster response than Cluster Autoscaler by directly provisioning instances without node group intermediation. Node consolidation provides ongoing optimization by identifying underutilized nodes and migrating pods to achieve higher utilization. Karpenter does not forecast future demand -- it responds to current demand signals (pending pods) with minimal latency.

**Platform Coverage:**
- **Kubernetes:** AWS EKS (primary), Azure AKS and GCP GKE support emerging
- **Cloud instances:** Direct EC2 instance provisioning (bypassing node groups)
- **No on-prem, no bare metal, no non-K8s**

**Strengths:**
- Fastest Kubernetes node provisioning -- seconds rather than minutes
- Instance type selection is workload-driven (evaluates pod requirements to choose optimal instance)
- No node groups required -- provisions exactly what is needed
- Node consolidation reduces waste by binpacking workloads
- Open source (CNCF) with broad adoption
- Handles diverse instance requirements (GPU, ARM, spot, on-demand) natively
- Active development with growing multi-cloud support

**Weaknesses:**
- Reactive, not predictive -- provisions after pods are pending, not before demand arrives
- Kubernetes-only -- no VM, bare metal, or non-K8s capacity planning
- AWS-primary -- Azure and GCP support is less mature
- No demand forecasting or growth planning
- No cross-infrastructure capacity planning
- No integration with storage or network capacity
- Does not consider data gravity or inter-service latency in placement
- No cost modeling beyond instance type selection
- No reserved capacity management (RI/SP optimization)
- Cannot evaluate whether a workload should run on K8s vs. bare metal vs. VMs

**Scope:** Kubernetes node autoscaling. AWS-primary. Reactive provisioning. No forecasting, no cross-domain, no orchestration.

---

## 7. Custom ML Approaches

**Core Capabilities:**
Organizations building custom capacity planning solutions typically use time series forecasting models (ARIMA, Prophet, LSTM neural networks, Transformer models) trained on infrastructure metrics (CPU, memory, network, storage utilization) and business signals (user traffic, transaction volume, event calendars). These solutions ingest data from monitoring platforms (Prometheus, Datadog, CloudWatch), apply forecasting models, and generate capacity recommendations or trigger automated scaling actions.

**Common Approaches:**

**Time Series Forecasting:**
- Facebook/Meta Prophet: Decomposable time series model handling seasonality, holidays, and trend changes. Easy to use, good for forecasting infrastructure metrics with daily/weekly/annual patterns.
- ARIMA/SARIMA: Statistical models for stationary time series with seasonal components. Well-understood but requires careful parameter tuning.
- LSTM/GRU Neural Networks: Deep learning models that capture complex temporal dependencies. Better for non-linear patterns but require more data and compute.
- Transformer Models: Attention-based models (Temporal Fusion Transformer, Informer) that handle multiple input signals and variable-length predictions. State-of-the-art for multi-signal forecasting.

**Anomaly Detection:**
- Isolation Forest, DBSCAN, Autoencoders: Detect unusual resource utilization patterns that may indicate capacity issues, misconfigurations, or security incidents.

**Reinforcement Learning:**
- RL agents that learn optimal scaling policies through interaction with the environment. Can discover non-obvious scaling strategies but require careful reward function design and significant training time.

**Platform Coverage:**
Custom solutions can theoretically cover any infrastructure type, but in practice most organizations build for a single domain: cloud compute scaling, K8s pod/node autoscaling, or database capacity planning. Cross-domain solutions are rare due to engineering complexity.

**Strengths:**
- Can be tailored to specific organizational patterns and business signals
- Can incorporate domain knowledge (event calendars, product launches, seasonal patterns)
- Can combine infrastructure metrics with business metrics for demand-driven forecasting
- Modern ML models (Transformers) achieve high accuracy for complex patterns
- Can be integrated into custom orchestration pipelines
- No vendor lock-in -- built on open-source ML frameworks

**Weaknesses:**
- Requires significant ML engineering expertise to build and maintain
- Model training, evaluation, and monitoring are ongoing operational burdens
- Data quality issues (missing metrics, label inconsistencies) undermine model accuracy
- Most custom solutions are single-domain (cloud OR K8s OR database)
- No standardized approach -- every organization reinvents the wheel
- Forecasting accuracy degrades for novel patterns (no historical data for new workloads)
- Integration with infrastructure actuation (scaling, provisioning) requires custom engineering
- Model drift requires continuous retraining
- No cost modeling or compliance awareness -- purely resource-focused
- ROI is difficult to justify for smaller organizations

**Scope:** Customizable to any domain but typically single-domain in practice. High engineering cost. No standardization.

---

## Comparative Matrix

| Capability | Densify | Turbonomic | Flexera | CloudPhysics | Spot.io | Karpenter | Custom ML |
|---|---|---|---|---|---|---|---|
| **Cloud compute** | Yes (right-sizing) | Yes (full-stack) | Yes (cost) | Target only | Yes (predictive) | K8s nodes only | Depends |
| **Kubernetes** | Yes (containers) | Yes (pods) | No | No | Yes (Ocean) | Core | Depends |
| **VMware/on-prem** | Yes | Yes | Discovery | Core | No | No | Depends |
| **Bare metal** | No | No | No | No | No | No | Rarely |
| **Network capacity** | No | Limited (awareness) | No | VMware only | No | No | Rarely |
| **Storage capacity** | No | Limited (awareness) | No | VMware only | No | No | Rarely |
| **Demand forecasting** | ML-based | Real-time demand | Cost projection | ML trend | ML-based | Reactive only | Yes (custom) |
| **Predictive scaling** | No (advisory) | Yes (automated) | No | No | Yes (automated) | Reactive | Possible |
| **Cross-domain** | Compute + K8s | Compute + storage (limited) | Cloud cost | VMware only | Cloud compute | K8s only | Typically single |
| **Cost optimization** | Yes (RI/SP) | Yes (via Cloudability) | Yes (core) | Migration cost | Yes (spot/RI) | Instance selection | Rarely |
| **Automated actuation** | Via IaC integration | Yes (actions) | No | No | Yes (scaling) | Yes (provisioning) | Custom |
| **Open source** | No | No | No | No | No | Yes (CNCF) | Open-source ML frameworks |
| **Orchestration integration** | No | No | No | No | No | K8s only | Custom |

---

## Gaps Analysis: What ALL Capacity Planning Tools Collectively Fail to Address

### 1. Every Tool Is Single-Domain -- No Cross-Infrastructure Capacity Planning

This is the most fundamental gap. Densify optimizes cloud compute and K8s. Turbonomic covers cloud, K8s, and VMware. Spot.io optimizes cloud compute scaling. Karpenter handles K8s node autoscaling. CloudPhysics covers VMware. Custom ML solutions are typically built for one domain.

No tool provides unified capacity planning across cloud VMs, Kubernetes clusters, bare metal servers, network infrastructure (switch port capacity, bandwidth, cross-connects), storage systems (array capacity, IOPS headroom, replication bandwidth), and edge/colo infrastructure. An organization running workloads across AWS, on-prem K8s, and colo bare metal servers has no tool that can answer: "Do we have enough capacity across all our infrastructure to handle projected demand for the next quarter?"

**LOOM's Opportunity:** LOOM provides cross-domain capacity planning by modeling capacity across every infrastructure type: cloud compute (instances, quotas, commitment utilization), Kubernetes (cluster resources, node capacity, pod scheduling headroom), bare metal (server inventory, CPU/memory/disk capacity), network (switch port availability, bandwidth utilization, cross-connect capacity), and storage (array capacity, IOPS headroom, replication bandwidth). LOOM answers capacity questions across the full infrastructure, not within a single silo.

### 2. No Business-Demand-Driven Infrastructure Forecasting

Current tools forecast infrastructure capacity based on infrastructure metrics: CPU utilization trends, memory consumption patterns, pod count growth. But infrastructure demand is driven by business factors: user growth, product launches, marketing campaigns, seasonal events, geographic expansion, and market conditions. No tool ingests business demand signals and translates them into infrastructure capacity requirements across all infrastructure types.

**LOOM's Opportunity:** LOOM can ingest business demand signals (user growth projections, planned product launches, seasonal traffic models, geographic expansion plans) and translate them into cross-domain infrastructure requirements. "We expect 50% user growth in APAC next quarter" translates into: additional compute capacity in APAC cloud regions, expanded K8s clusters, additional bare metal servers in APAC colo, increased network bandwidth on transpacific links, additional storage capacity for APAC data, and new DNS/CDN configuration for APAC traffic.

### 3. No Capacity-Aware Orchestration

Current capacity planning tools generate recommendations or forecasts that are consumed by humans. Turbonomic and Spot.io can automate scaling actions within their domain, but no tool feeds capacity data into infrastructure orchestration decisions. When placing a workload, no orchestrator evaluates: "Does this datacenter have enough compute capacity? Is the network path saturated? Is the storage array near IOPS limits? Will this placement exceed commitment utilization targets?"

**LOOM's Opportunity:** LOOM uses real-time capacity data as a placement constraint. When placing a workload, LOOM evaluates available capacity across all infrastructure dimensions: compute (CPU, memory, GPU), storage (capacity, IOPS, throughput), network (bandwidth, latency, port availability), and commitment utilization (are we within committed capacity, or will this exceed commitments?). Placement decisions are capacity-aware by default.

### 4. No Network Capacity Planning

No tool in this landscape provides network capacity planning: predicting when switch port density will be exhausted, when uplink bandwidth will be saturated, when cross-connect capacity will need expansion, when BGP peering bandwidth will need upgrading, or when CDN capacity will need scaling. Network capacity is planned manually using spreadsheets, SNMP utilization reports, and educated guesses.

**LOOM's Opportunity:** LOOM models network capacity across the full network infrastructure: switch port utilization and density, uplink bandwidth consumption and trends, cross-connect capacity and planned expansion, BGP peering bandwidth and traffic growth, CDN capacity and cache hit ratios, and WAN link utilization and latency. LOOM forecasts network capacity exhaustion and factors network capacity into workload placement decisions -- a workload will not be placed in a datacenter where the uplink bandwidth is approaching saturation.

### 5. No Integrated Capacity + Cost Planning

Capacity planning and cost planning are performed by different tools (Densify for capacity, Cloudability for cost) or different teams (infrastructure for capacity, finance for cost). No tool provides integrated capacity-cost planning: "We need 30% more compute capacity next quarter. What is the cheapest way to provide it -- expanding cloud commitments, adding bare metal servers in colo, or upgrading existing VMware hosts?" This question requires understanding both capacity requirements and cost models across all infrastructure types.

**LOOM's Opportunity:** LOOM integrates capacity planning with cost modeling. When capacity expansion is needed, LOOM evaluates all options: purchasing cloud reserved instances vs. adding bare metal servers vs. expanding K8s clusters vs. upgrading VMware hosts. Each option is evaluated for cost (upfront + ongoing), performance, availability, and compliance. LOOM recommends the optimal capacity expansion strategy that meets demand at minimum cost within policy constraints.

### 6. No Capacity Planning for Supporting Infrastructure

Current tools plan capacity for compute and (to a limited extent) storage. But infrastructure has capacity needs beyond compute: IP address space exhaustion, DNS query volume limits, DHCP scope capacity, certificate authority throughput, secrets management performance, logging/monitoring pipeline capacity, and backup storage consumption. No tool plans capacity for these supporting infrastructure components.

**LOOM's Opportunity:** LOOM plans capacity across all infrastructure components: compute (CPU, memory, GPU), storage (capacity, IOPS), network (bandwidth, ports), IP address space (subnet utilization, DHCP scope capacity), DNS (query volume, zone count), certificate infrastructure (CA throughput, certificate inventory growth), secrets management (Vault performance, secret count growth), observability (log volume, metric cardinality, trace storage), and backup (backup storage growth, restore SLA capacity). This comprehensive capacity model prevents the common failure mode where compute capacity is available but a supporting service is exhausted.

### 7. No Predictive Maintenance Integration

Capacity planning should account for planned and predicted maintenance events: hardware refresh cycles, firmware upgrade windows, certificate renewal schedules, cloud provider maintenance windows, and predicted hardware failures (SMART disk data, memory error rates). No capacity planning tool integrates maintenance predictions into capacity models. A capacity plan that shows 95% compute utilization is actually at risk if 10% of servers are due for maintenance in the same quarter.

**LOOM's Opportunity:** LOOM integrates maintenance forecasting into capacity planning. Hardware refresh schedules, predicted failure rates (based on age, error rates, manufacturer data), planned cloud provider maintenance windows, certificate expiry schedules, and firmware upgrade requirements are all factored into capacity models. LOOM ensures that adequate capacity exists to maintain service levels during maintenance windows, not just during steady-state operations.

---

## The Fundamental Gap: Single-Domain Optimization vs. Cross-Domain Capacity Intelligence

The capacity planning industry is fragmented by infrastructure domain. Each tool optimizes within its silo:
- Densify optimizes cloud compute
- Turbonomic optimizes application resources across VMs and containers
- Spot.io optimizes cloud scaling
- Karpenter optimizes K8s node provisioning
- CloudPhysics optimizes VMware
- Custom ML solutions typically target one domain

No tool provides a unified capacity model that spans cloud, Kubernetes, bare metal, network, storage, DNS, certificates, secrets, and supporting infrastructure. No tool translates business demand signals into cross-domain infrastructure requirements. No tool integrates capacity data into orchestration decisions. No tool combines capacity planning with cost optimization across infrastructure types.

LOOM provides cross-domain capacity intelligence by:
1. **Modeling capacity across all infrastructure types** -- compute, storage, network, supporting services
2. **Translating business demand into infrastructure requirements** -- not just metric extrapolation
3. **Feeding capacity data into orchestration decisions** -- capacity is a placement constraint
4. **Integrating capacity with cost** -- evaluating expansion options across infrastructure types for optimal cost
5. **Planning capacity for the entire stack** -- including DNS, certificates, secrets, monitoring, and backup
6. **Incorporating maintenance and lifecycle** -- accounting for planned and predicted infrastructure events

This is the difference between domain-specific resource optimization (what exists today) and cross-domain infrastructure capacity intelligence (what LOOM provides). No existing tool crosses the domain boundaries that LOOM is designed to span.

---

*Sources consulted:*
- [Densify Documentation](https://www.densify.com/docs/)
- [Densify Cloud Optimization](https://www.densify.com/solutions/cloud-optimization/)
- [IBM Turbonomic](https://www.ibm.com/products/turbonomic)
- [Turbonomic ARM Platform](https://www.ibm.com/docs/en/tarm)
- [Flexera One Cloud Management](https://www.flexera.com/products/cloud-management-platform)
- [CloudPhysics](https://www.vmware.com/products/aria-migration.html)
- [Spot by NetApp / Flexera](https://spot.io/)
- [Spot Ocean Documentation](https://docs.spot.io/ocean/)
- [Karpenter Documentation](https://karpenter.sh/docs/)
- [Karpenter - CNCF](https://www.cncf.io/projects/karpenter/)
- [Facebook Prophet](https://facebook.github.io/prophet/)
- [Temporal Fusion Transformer](https://arxiv.org/abs/1912.09363)
- [Kubernetes Cluster Autoscaler](https://github.com/kubernetes/autoscaler)
- [ARIMA Time Series Forecasting](https://otexts.com/fpp3/arima.html)
