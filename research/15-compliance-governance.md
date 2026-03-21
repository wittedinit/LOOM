# Compliance, Governance, and Policy Enforcement Frameworks: Comprehensive Research Report

## 1. Open Policy Agent (OPA / Rego)

**Core Capabilities:**
Open Policy Agent (OPA) is a CNCF Graduated general-purpose policy engine that decouples policy decisions from policy enforcement. OPA evaluates policies written in Rego (a purpose-built declarative language) against structured data (JSON) and returns decisions. OPA is used across the infrastructure stack: Kubernetes admission control (Gatekeeper), API authorization, Terraform plan validation (Conftest), microservice authorization (Envoy external authorization), data filtering, and CI/CD pipeline policy. OPA runs as a sidecar, daemon, or library, making decisions at the speed of microseconds without external network calls.

**Policy Language:**
Rego is a declarative query language designed for policy authoring. It supports hierarchical data traversal, set operations, comprehensions, and built-in functions for string manipulation, JWT verification, networking (CIDR matching), and more. Rego policies are composable -- multiple policies can be combined with conflict resolution rules. Policies are versioned and testable with OPA's built-in test framework.

**Integration Points:**
- **Kubernetes:** Gatekeeper provides admission control with constraint templates (CRDs)
- **Terraform:** Conftest validates Terraform plans against OPA policies before apply
- **Envoy/Istio:** External authorization filter for API gateway and service mesh policy
- **CI/CD:** Policy checks in pipelines via conftest or OPA CLI
- **Custom applications:** OPA Go library, REST API, or WASM runtime for embedded policy evaluation

**Compliance Mapping:**
OPA policies can encode compliance requirements but require manual policy authoring. No built-in compliance frameworks or benchmarks. Organizations must translate CIS, PCI-DSS, HIPAA, SOC 2, and FedRAMP requirements into Rego policies themselves. Community policy libraries exist (Rego Policy Library, Styra DAS policy packs) but coverage is incomplete.

**Strengths:**
- CNCF Graduated -- the standard for policy-as-code across cloud-native infrastructure
- General-purpose -- policies can govern any domain (K8s, Terraform, APIs, data access)
- Microsecond decision latency with no external dependencies at runtime
- Rego is purpose-built for policy -- more expressive than YAML-based alternatives
- Gatekeeper provides native Kubernetes integration
- Conftest enables Terraform policy validation before deployment
- Active community with broad ecosystem adoption
- Styra DAS provides enterprise management, policy distribution, and compliance mapping

**Weaknesses:**
- Rego has a steep learning curve -- unfamiliar syntax for most engineers
- No built-in compliance frameworks -- policies must be authored from scratch
- Policy authoring requires deep understanding of the target system's data model
- Gatekeeper (K8s) is the most mature integration; other integrations require more effort
- No remediation -- OPA denies requests but does not fix violations
- No continuous monitoring -- OPA evaluates at decision time, not continuously against state
- Policy testing, while supported, requires discipline and dedicated effort
- No infrastructure awareness -- OPA evaluates data provided to it, not infrastructure state directly
- No cross-domain policy correlation (a K8s policy does not know about Terraform state)

**Scope:** General-purpose policy engine. Policy-as-code. Kubernetes, Terraform, APIs. No remediation, no continuous monitoring, no cross-domain correlation.

---

## 2. Kyverno

**Core Capabilities:**
Kyverno is a CNCF Incubating Kubernetes-native policy engine designed specifically for Kubernetes. Unlike OPA/Gatekeeper, Kyverno policies are written as Kubernetes resources (YAML) -- no new language to learn. Kyverno supports four policy types: validation (admit/deny), mutation (modify resources), generation (create resources based on triggers), and image verification (verify container image signatures and attestations). Kyverno also provides cleanup policies (automated resource deletion), policy exceptions, and background scanning of existing resources.

**Policy Language:**
Kyverno policies are YAML manifests using Kubernetes-native constructs: match/exclude selectors, CEL (Common Expression Language) and JMESPath expressions for conditions, and overlay patterns for mutations. This YAML-based approach has a significantly lower barrier to entry than Rego. Policy reports (PolicyReport CRD) provide compliance status for existing resources.

**Integration Points:**
- **Kubernetes:** Native admission controller (validating and mutating webhooks)
- **Background scanning:** Continuously evaluates existing resources against policies
- **Image verification:** Cosign, Notary v2, and Sigstore integration
- **Policy reports:** Compliance reporting via PolicyReport CRDs
- **Kyverno CLI:** Pre-deployment policy testing in CI/CD pipelines
- **Kyverno JSON:** Policy evaluation for non-Kubernetes JSON data (Terraform plans, API payloads)

**Compliance Mapping:**
Kyverno provides official policy libraries mapping to Pod Security Standards, CIS Kubernetes Benchmark, and best practices. Community policies cover common compliance scenarios. Kyverno does not natively map policies to regulatory frameworks (PCI-DSS, HIPAA, SOC 2) -- this mapping is left to operators.

**Strengths:**
- No new language -- policies are Kubernetes YAML
- Mutation and generation capabilities go beyond OPA/Gatekeeper's validation-only model
- Background scanning provides continuous compliance for existing resources
- Image verification supports software supply chain security
- Cleanup policies automate resource lifecycle management
- PolicyReport CRDs provide standardized compliance reporting
- Lower barrier to entry than OPA for Kubernetes-only environments
- Active CNCF community with growing policy library

**Weaknesses:**
- Kubernetes-only -- no support for Terraform, APIs, or non-K8s infrastructure
- Kyverno JSON extends to non-K8s data but is early-stage
- YAML policies can become complex and verbose for sophisticated logic
- No remediation beyond mutation at admission time -- cannot fix running resources
- Performance concerns at scale (many policies increase admission latency)
- No cross-cluster policy management without external tooling
- No integration with infrastructure provisioning outside Kubernetes
- No compliance mapping to regulatory frameworks (PCI-DSS, HIPAA, etc.)
- Cannot evaluate Terraform plans, network device configurations, or bare metal compliance

**Scope:** Kubernetes-native policy engine. Validation, mutation, generation, image verification. No cross-platform, no Terraform, no network, no bare metal.

---

## 3. Falco

**Core Capabilities:**
Falco is a CNCF Graduated runtime security tool that detects anomalous behavior in containers, hosts, Kubernetes, and cloud environments. Falco uses eBPF (or kernel module) to monitor system calls in real-time and evaluate them against rules. It detects: unexpected process execution, file access violations, network connections, privilege escalation, container escapes, and cloud API abuse (via CloudTrail plugin). Falco rules are written in YAML with a condition/output format. Falco Sidekick provides 60+ output integrations (Slack, PagerDuty, Elasticsearch, etc.).

**Policy Language:**
Falco rules use a condition-based language that matches system call events. Conditions reference system call arguments, process metadata, container information, and Kubernetes context. Macros and lists provide reusability. Rules are easier to write than Rego but less expressive for complex policy logic.

**Integration Points:**
- **Linux kernel:** eBPF-based system call monitoring (kernel module fallback)
- **Kubernetes:** Pod metadata enrichment for container context
- **Cloud:** CloudTrail, GCP Audit Logs, Azure Activity Logs via plugins
- **Alerting:** Falco Sidekick provides 60+ output integrations
- **Incident response:** Falco Talon provides automated response actions

**Compliance Mapping:**
Falco rules can be mapped to compliance requirements (CIS benchmarks for runtime controls, PCI-DSS requirements for file integrity monitoring, HIPAA for access logging), but this mapping is manual. The Falco rules repository provides community-maintained rules for common compliance scenarios.

**Strengths:**
- Runtime detection -- catches threats that admission-time policies miss
- eBPF-based monitoring is low-overhead and kernel-level comprehensive
- Cloud plugin system extends detection to cloud control plane (CloudTrail)
- CNCF Graduated with strong community and ecosystem
- Falco Sidekick provides comprehensive alerting integration
- Falco Talon enables automated response (kill container, isolate pod)
- Detects zero-day threats through behavioral analysis, not just known signatures
- Open-source with commercial offering (Sysdig Secure)

**Weaknesses:**
- Detection only (by default) -- Falco alerts but does not prevent violations
- Falco Talon (automated response) is early-stage
- No policy enforcement at deployment time -- runtime-only
- Rule authoring requires understanding of Linux system calls
- False positive tuning is an ongoing operational burden
- No integration with infrastructure provisioning or orchestration
- Cannot enforce compliance on network devices, storage arrays, or DNS/DHCP
- eBPF requires kernel 4.18+ -- incompatible with older operating systems
- High-volume alert generation can overwhelm SOC teams without tuning

**Scope:** Runtime security detection for containers, hosts, and cloud. No preventive enforcement, no infrastructure provisioning, no orchestration.

---

## 4. Prowler

**Core Capabilities:**
Prowler is an open-source security assessment tool for cloud environments (AWS, Azure, GCP, Kubernetes). It performs automated compliance checks against security benchmarks: CIS AWS/Azure/GCP/K8s Foundations, PCI-DSS, HIPAA, GDPR, NIST 800-53, SOC 2, FedRAMP, ENS, and more. Prowler executes 300+ checks covering IAM, networking, encryption, logging, monitoring, and compute configuration. Results are output in multiple formats (CSV, JSON, HTML, OCSF) and can be sent to AWS Security Hub, S3, or third-party SIEMs.

**Policy Language:**
Prowler checks are implemented in Python (AWS) and Go (Azure, GCP, K8s). Custom checks can be added by writing new check classes. No declarative policy language -- checks are code. Prowler v4 introduced a metadata-driven check system with compliance mapping baked into check definitions.

**Integration Points:**
- **AWS:** Direct API scanning with IAM role-based access
- **Azure:** Service principal-based scanning
- **GCP:** Service account-based scanning
- **Kubernetes:** kubeconfig-based scanning
- **CI/CD:** CLI execution in pipelines
- **AWS Security Hub:** Native integration for findings aggregation
- **OCSF format:** Standardized output for SIEM integration

**Compliance Mapping:**
Prowler's primary strength is built-in compliance mapping. Each check is tagged with the frameworks it satisfies (CIS, PCI-DSS, HIPAA, SOC 2, FedRAMP, NIST, GDPR, ENS). Compliance dashboards show pass/fail status per framework. This is the most comprehensive out-of-the-box compliance mapping of any open-source tool.

**Strengths:**
- Most comprehensive open-source cloud compliance scanner
- 300+ built-in checks with compliance framework mapping
- Multi-cloud support (AWS, Azure, GCP, Kubernetes)
- HTML dashboards provide executive-ready compliance reports
- OCSF output format for standardized SIEM integration
- AWS Security Hub integration for findings aggregation
- Active community with regular check updates
- Free and open source (Apache 2.0)

**Weaknesses:**
- Point-in-time scanning -- not continuous monitoring
- No enforcement or remediation -- reports violations but does not fix them
- Cloud-only -- no bare metal, no network devices, no on-prem infrastructure
- Checks are code (Python/Go), not declarative policies -- harder to customize
- No integration with infrastructure provisioning or orchestration
- Scanning large environments can take significant time
- No prevention -- runs after infrastructure is deployed
- Cannot validate Terraform plans or IaC before deployment
- No policy-as-code approach -- checks are hardcoded for known configurations

**Scope:** Cloud security assessment and compliance scanning. AWS, Azure, GCP, K8s. Point-in-time. No enforcement, no on-prem, no orchestration.

---

## 5. ScoutSuite

**Core Capabilities:**
ScoutSuite is an open-source multi-cloud security auditing tool that collects cloud configuration data via provider APIs and generates an interactive HTML report highlighting security risks. ScoutSuite supports AWS, Azure, GCP, Alibaba Cloud, and Oracle Cloud. It evaluates configurations against built-in security rules covering IAM, compute, storage, networking, databases, logging, and encryption.

**Policy Language:**
ScoutSuite rules are defined in JSON with conditions that match against collected cloud configuration data. Custom rules can be added by defining new JSON rule files. The rule format is simpler than Rego but less expressive.

**Integration Points:**
- **Multi-cloud:** AWS, Azure, GCP, Alibaba, Oracle
- **Output:** Interactive HTML report with filtering and drill-down
- **CI/CD:** CLI execution with JSON/CSV output for pipeline integration
- **No SIEM integration:** Results are file-based only

**Compliance Mapping:**
ScoutSuite rules are organized by service and risk level but do not map to specific compliance frameworks (CIS, PCI-DSS, HIPAA). Users must manually correlate findings to compliance requirements.

**Strengths:**
- Multi-cloud coverage including Alibaba and Oracle (broader than Prowler)
- Interactive HTML report is excellent for security review and presentations
- JSON-based rules are easy to understand and customize
- Lightweight -- no server component, runs as a Python CLI
- Offline analysis -- collects data once, analyzes locally

**Weaknesses:**
- No compliance framework mapping (must be done manually)
- No enforcement or remediation
- Point-in-time scanning only
- No continuous monitoring
- HTML report format is not easily integrated into automated workflows
- Development pace has slowed compared to Prowler and Checkov
- No Kubernetes support
- No bare metal or on-prem infrastructure support
- No integration with orchestration or provisioning
- No prevention -- post-deployment analysis only

**Scope:** Multi-cloud security auditing. Point-in-time. No compliance mapping, no enforcement, no K8s, no on-prem, no orchestration.

---

## 6. Checkov

**Core Capabilities:**
Checkov (by Prisma Cloud/Palo Alto Networks) is an open-source static analysis tool for Infrastructure as Code (IaC). It scans Terraform, CloudFormation, Kubernetes manifests, Helm charts, Dockerfiles, ARM templates, Bicep, Serverless framework, and more for security misconfigurations before deployment. Checkov includes 1,000+ built-in policies covering CIS benchmarks, NIST, PCI-DSS, HIPAA, and SOC 2. Custom policies can be written in Python or YAML. Checkov integrates with Prisma Cloud for enterprise policy management, drift detection, and supply chain security.

**Policy Language:**
Checkov supports custom policies in Python (full flexibility) and YAML (declarative, simpler). Built-in policies use Python. YAML policies use a simple attribute-matching syntax with conditional logic. Graph-based policies evaluate cross-resource relationships (e.g., "this S3 bucket is referenced by a CloudFront distribution with a WAF").

**Integration Points:**
- **IaC:** Terraform, CloudFormation, Kubernetes, Helm, Dockerfile, ARM, Bicep
- **CI/CD:** GitHub Actions, GitLab CI, Jenkins, Azure DevOps integrations
- **IDE:** VS Code extension for in-editor policy feedback
- **Prisma Cloud:** Enterprise platform for policy management and drift detection
- **SARIF output:** Code scanning alert integration with GitHub/GitLab

**Compliance Mapping:**
Checkov provides built-in compliance framework mapping: CIS AWS/Azure/GCP/K8s, NIST 800-53, PCI-DSS, HIPAA, SOC 2, and ISO 27001. Compliance reports show pass/fail status per framework with per-check detail. Graph-based policies can evaluate complex compliance requirements spanning multiple resources.

**Strengths:**
- Most comprehensive IaC static analysis tool
- 1,000+ built-in policies with compliance framework mapping
- Pre-deployment scanning prevents misconfigurations before they reach production
- Graph-based policies evaluate cross-resource relationships
- Multi-format support (Terraform, CloudFormation, K8s, Helm, Dockerfile)
- IDE integration provides immediate developer feedback
- YAML custom policies lower the barrier for policy authoring
- Open source with active community and Palo Alto Networks backing

**Weaknesses:**
- Pre-deployment only -- no runtime monitoring or enforcement
- IaC-focused -- cannot evaluate running infrastructure state
- No bare metal or network device configuration scanning
- No enforcement -- flags violations but does not prevent deployment (without CI/CD gate)
- Prisma Cloud integration for enterprise features creates vendor dependency
- Graph-based policies are powerful but complex to author
- Cannot evaluate ad-hoc infrastructure changes (manual console changes, CLI commands)
- No integration with orchestration or provisioning workflows
- False positives require ongoing policy tuning

**Scope:** IaC static analysis. Pre-deployment security scanning. No runtime, no bare metal, no network devices, no orchestration.

---

## 7. Terrascan

**Core Capabilities:**
Terrascan (by Tenable/Accurics) is an open-source static analysis tool for IaC supporting Terraform, Kubernetes, Helm, Kustomize, Dockerfiles, CloudFormation, ARM templates, and more. Terrascan includes 500+ built-in policies based on CIS benchmarks and industry best practices. Policies are written in Rego (OPA), enabling reuse of OPA policy expertise. Terrascan can run as a CLI, in CI/CD pipelines, as a Kubernetes admission controller, or as a server with REST API.

**Policy Language:**
Terrascan uses Rego (OPA's policy language) for all policy definitions. This means organizations already using OPA can reuse policy knowledge. Custom policies follow the same Rego conventions as OPA/Gatekeeper.

**Integration Points:**
- **IaC:** Terraform, K8s manifests, Helm, CloudFormation, ARM, Dockerfile
- **CI/CD:** CLI and server mode for pipeline integration
- **Kubernetes:** Admission controller mode for runtime validation
- **Git hooks:** Pre-commit integration for developer workflow
- **Atlantis:** Integration with Terraform PR automation

**Compliance Mapping:**
Terrascan maps policies to CIS benchmarks (AWS, Azure, GCP, K8s), NIST 800-53, PCI-DSS, HIPAA, and SOC 2. Compliance reports show pass/fail per framework.

**Strengths:**
- Uses Rego -- consistent policy language with OPA/Gatekeeper ecosystem
- Admission controller mode bridges IaC scanning with runtime enforcement
- 500+ built-in policies with compliance mapping
- Server mode with REST API enables centralized policy evaluation
- Good IaC format coverage
- Tenable backing provides enterprise support path

**Weaknesses:**
- Rego learning curve applies
- Fewer built-in policies than Checkov (500 vs. 1,000+)
- Less active community than Checkov or OPA
- No graph-based cross-resource policy evaluation
- No runtime monitoring beyond admission control
- No bare metal or network device support
- No integration with infrastructure orchestration
- No remediation capabilities
- Development pace has slowed compared to Checkov

**Scope:** IaC static analysis with Rego. Pre-deployment and K8s admission. No runtime monitoring, no bare metal, no orchestration.

---

## 8. Cloud Custodian

**Core Capabilities:**
Cloud Custodian is an open-source rules engine for cloud resource management, providing automated governance, compliance, and cost optimization. Custodian policies are YAML-based rules that filter cloud resources and apply actions: tag, notify, stop, terminate, snapshot, resize, and more. Custodian supports AWS, Azure, and GCP with 300+ resource types. It runs as Lambda functions (AWS), Azure Functions, or GCP Cloud Functions for real-time event-driven policy enforcement, or as scheduled batch scans.

**Policy Language:**
Custodian policies are YAML with a filter-action model: filters select resources matching criteria, and actions define what to do with matching resources. Filters support boolean logic, value comparisons, cross-resource relationships, and age-based conditions. The YAML format is intuitive and accessible to non-developers.

**Integration Points:**
- **AWS:** Lambda-based real-time enforcement, CloudWatch Events, CloudTrail
- **Azure:** Azure Functions, Event Grid
- **GCP:** Cloud Functions, Pub/Sub
- **Notifications:** Email, Slack, SNS, SQS, webhooks
- **Scheduled scanning:** Cron-based batch evaluation

**Compliance Mapping:**
Custodian policies can be mapped to compliance requirements, but there is no built-in compliance framework mapping. Organizations must create their own policy-to-framework mappings. Community policy collections exist for CIS benchmarks.

**Strengths:**
- Unique in this category: Cloud Custodian both detects AND remediates violations
- Real-time event-driven enforcement via serverless functions
- Accessible YAML policy language with filter-action model
- Actions include stop, terminate, tag, snapshot, resize -- actual infrastructure changes
- 300+ resource types across AWS, Azure, GCP
- Cost optimization policies (stop idle instances, delete old snapshots, tag untagged resources)
- Open source with active community (CNCF Incubating)
- Batch + real-time modes cover both continuous and periodic compliance

**Weaknesses:**
- Cloud-only -- no bare metal, no network devices, no on-prem
- No Kubernetes-native policies (focuses on cloud resources, not K8s objects)
- No IaC scanning -- evaluates running infrastructure, not pre-deployment code
- Action-based remediation can be dangerous without careful policy testing
- No cross-cloud policy correlation
- No integration with infrastructure orchestration
- Policy-to-compliance mapping is manual
- Serverless deployment model can create operational complexity across many functions
- No prevention -- policies evaluate after resources are created (event-driven catches quickly but not preemptively)

**Scope:** Cloud resource governance with automated remediation. AWS, Azure, GCP. Real-time and batch. No K8s, no bare metal, no IaC, no orchestration.

---

## 9. Compliance Frameworks: HIPAA, SOC 2, PCI-DSS, CIS, FedRAMP

**HIPAA (Health Insurance Portability and Accountability Act):**
HIPAA requires technical safeguards for Protected Health Information (PHI): access controls (unique user identification, emergency access, automatic logoff, encryption), audit controls (recording and examining access to PHI systems), integrity controls (authentication mechanisms for ePHI), transmission security (encryption, integrity controls for data in transit), and facility/device access controls. Infrastructure implications: encryption at rest and in transit, audit logging, access control, backup and disaster recovery, and BAA (Business Associate Agreement) requirements for cloud providers.

**SOC 2 (System and Organization Controls 2):**
SOC 2 evaluates service organizations against Trust Services Criteria: security (common criteria, required), availability, processing integrity, confidentiality, and privacy (optional). Infrastructure requirements: logical and physical access controls, change management, risk assessment, incident response, monitoring, encryption, network security, and vendor management. SOC 2 Type II requires evidence of controls operating effectively over a period (typically 6-12 months).

**PCI-DSS (Payment Card Industry Data Security Standard):**
PCI-DSS v4.0 includes 12 requirements across 6 categories: network security (firewalls, segmentation), access controls (unique IDs, MFA, least privilege), data protection (encryption, key management), vulnerability management (patching, AV, secure development), monitoring (logging, IDS, file integrity), and security policies. Infrastructure implications: network segmentation, encryption, access logging, vulnerability scanning, configuration management, and regular testing.

**CIS Benchmarks (Center for Internet Security):**
CIS provides prescriptive hardening guides for operating systems (Linux, Windows), cloud platforms (AWS, Azure, GCP), Kubernetes, databases, network devices, and more. CIS benchmarks define specific configuration settings with rationale, audit procedures, and remediation steps. CIS Controls (formerly Critical Security Controls) provide a prioritized framework of 18 control categories.

**FedRAMP (Federal Risk and Authorization Management Program):**
FedRAMP provides a standardized approach to security assessment and authorization for cloud products used by US federal agencies. It is based on NIST SP 800-53 controls with three impact levels: Low (125 controls), Moderate (325 controls), and High (421 controls). FedRAMP requires continuous monitoring, annual assessments, vulnerability scanning, and incident response. FedRAMP Authorization (ATO) is notoriously difficult and time-consuming to obtain.

**Infrastructure Implications Across Frameworks:**
All major compliance frameworks share common infrastructure requirements:
- Encryption at rest and in transit
- Access control with authentication, authorization, and audit logging
- Network segmentation and firewall rules
- Configuration management and hardening
- Vulnerability management and patching
- Incident detection and response
- Backup and disaster recovery
- Change management and change tracking
- Continuous monitoring and alerting

**The Compliance Gap:**
These frameworks define WHAT must be achieved but not HOW to achieve it in modern infrastructure. They were designed for traditional IT environments (physical servers, on-prem networks) and have been adapted -- often awkwardly -- to cloud-native, containerized, and hybrid environments. No framework provides native guidance for Kubernetes, GitOps, Infrastructure as Code, or multi-cloud architectures. Compliance is assessed after infrastructure is deployed, not embedded into the deployment process.

---

## Comparative Matrix

| Capability | OPA/Rego | Kyverno | Falco | Prowler | ScoutSuite | Checkov | Terrascan | Cloud Custodian |
|---|---|---|---|---|---|---|---|---|
| **Policy timing** | Admission/API | Admission + background | Runtime | Scan (point-in-time) | Scan (point-in-time) | Pre-deploy (IaC) | Pre-deploy + admission | Real-time + batch |
| **Enforcement** | Deny/Allow | Deny/Mutate/Generate | Alert (Talon for response) | Report only | Report only | Flag (CI/CD gate) | Flag + admission deny | Filter + Action (remediate) |
| **K8s native** | Gatekeeper | Core | eBPF + K8s metadata | Yes (scanning) | No | Yes (manifests) | Yes (scanning + admission) | No |
| **Cloud scanning** | Via data input | No | CloudTrail plugin | AWS, Azure, GCP | AWS, Azure, GCP, Ali, Oracle | Via IaC | Via IaC | AWS, Azure, GCP |
| **Terraform** | Conftest | No | No | No | No | Core | Core | No |
| **Bare metal** | Possible (data-driven) | No | Yes (host eBPF) | No | No | No | No | No |
| **Network devices** | Possible (data-driven) | No | No | No | No | No | No | No |
| **Compliance mapping** | Manual | Pod Security/CIS | Manual | CIS, PCI, HIPAA, SOC2, FedRAMP | Manual | CIS, PCI, HIPAA, SOC2, NIST | CIS, PCI, HIPAA, SOC2 | Manual |
| **Remediation** | No | Mutation only | Talon (early) | No | No | No | No | Yes (actions) |
| **Cross-domain** | General-purpose | K8s only | Runtime only | Cloud only | Cloud only | IaC only | IaC + K8s | Cloud only |
| **Open source** | Yes (CNCF) | Yes (CNCF) | Yes (CNCF) | Yes (Apache 2) | Yes (GPL) | Yes (Apache 2) | Yes (Apache 2) | Yes (CNCF) |
| **Orchestration** | No | No | No | No | No | No | No | No |

---

## Gaps Analysis: What ALL Compliance/Governance Tools Collectively Fail to Address

### 1. Compliance Is Assessed After Deployment, Not Embedded in Orchestration

The compliance tooling ecosystem follows a pattern: infrastructure is deployed, then compliance tools scan it, report violations, and -- at best -- remediate them. Even "shift-left" tools like Checkov and Terrascan evaluate IaC before deployment but do not participate in the deployment decision. No tool embeds compliance evaluation into the orchestration decision -- "should this workload be placed here, given these compliance constraints?"

OPA/Gatekeeper and Kyverno enforce policy at Kubernetes admission time, but this is reactive (deny after the request is made) rather than proactive (evaluate compliance as a placement constraint). Cloud Custodian remediates after creation. Prowler scans after deployment. The entire paradigm is reactive.

**LOOM's Opportunity:** LOOM embeds compliance as a first-class constraint in every orchestration decision. Before placing a workload, LOOM evaluates: "Does this target environment meet the compliance requirements of this workload?" A PCI-DSS workload will only be placed in a PCI-scoped network segment with encrypted storage, audit logging enabled, and approved firewall rules. A HIPAA workload will only be placed where BAA-covered infrastructure exists. Compliance is not checked after deployment -- it is a placement constraint that prevents non-compliant deployments from being attempted.

### 2. No Cross-Domain Compliance Enforcement

Current tools enforce compliance within their domain: OPA/Kyverno for Kubernetes, Checkov/Terrascan for IaC, Prowler for cloud configuration, Falco for runtime behavior, Cloud Custodian for cloud resources. No tool provides unified compliance enforcement across Kubernetes, cloud resources, bare metal servers, network devices, storage arrays, and DNS/DHCP configuration.

A PCI-DSS deployment requires compliant configuration across all these domains simultaneously: network segmentation on switches, firewall rules on routers, encryption on storage, access controls on compute, audit logging on all systems, and vulnerability management across all components. No tool evaluates PCI-DSS compliance across all these domains as a unified assessment.

**LOOM's Opportunity:** LOOM evaluates compliance across the entire infrastructure stack. When a PCI-DSS workload is deployed, LOOM verifies: network segmentation is configured on the physical switches (via NAPALM), firewall rules are in place (via network device APIs), storage encryption is enabled (via storage array APIs), compute hardening is applied (via Ansible), audit logging is configured (via observability APIs), and certificates are valid (via cert-manager/Vault). This cross-domain compliance evaluation is performed as part of the deployment decision, not as a separate scanning exercise.

### 3. No Compliance-Driven Infrastructure Provisioning

Current compliance tools tell you what is wrong (Prowler, ScoutSuite) or prevent what is wrong (OPA, Kyverno, Checkov). No tool proactively provisions compliant infrastructure. If a workload requires HIPAA compliance, no tool can automatically provision the required infrastructure: encrypted storage, audit logging, access controls, network segmentation, backup policies, and incident response configuration -- all configured to meet HIPAA requirements.

**LOOM's Opportunity:** LOOM can provision compliance-ready infrastructure from a single declaration. Specifying a workload as "HIPAA-scoped" triggers LOOM to automatically configure: encrypted volumes (via storage orchestration), TLS certificates (via cert-manager/Vault), network segmentation (via VLAN provisioning on switches), audit logging (via observability configuration), access controls (via RBAC and secrets management), backup policies (via storage replication), and monitoring (via alerting configuration). Compliance is provisioned, not assessed.

### 4. No Continuous Compliance Verification Across All Infrastructure

Prowler and ScoutSuite scan periodically. Falco monitors runtime continuously but only for system call behavior. Kyverno's background scanning continuously evaluates Kubernetes resources. But no tool continuously verifies compliance across all infrastructure types: cloud configurations, Kubernetes objects, bare metal server configurations, network device configurations, storage encryption status, certificate validity, and secret rotation compliance.

**LOOM's Opportunity:** LOOM continuously verifies compliance across all managed infrastructure. Every infrastructure resource is continuously evaluated against its applicable compliance policies. When drift is detected (a firewall rule is modified, encryption is disabled, a certificate expires, a secret rotation is overdue), LOOM can alert, remediate, or prevent dependent operations from proceeding until compliance is restored.

### 5. Compliance Evidence Generation Is Manual

Demonstrating compliance for audits (SOC 2 Type II, PCI-DSS, FedRAMP) requires collecting evidence: configuration screenshots, log exports, policy documentation, change records, and test results. This evidence collection is manual, time-consuming, and error-prone. No tool automatically generates audit-ready evidence from infrastructure state.

**LOOM's Opportunity:** LOOM generates compliance evidence automatically from its infrastructure state and operation history. Every deployment decision, configuration change, policy evaluation, and remediation action is recorded with full context. When an auditor requests evidence for a SOC 2 control, LOOM can produce the complete history of how that control was implemented, verified, and maintained across all infrastructure -- without manual evidence collection.

### 6. No Compliance-Aware Cost Optimization

Compliance requirements constrain infrastructure options: PCI-DSS workloads cannot use shared tenancy in some interpretations, FedRAMP requires GovCloud regions, HIPAA requires BAA-covered services. These constraints affect infrastructure cost but no compliance or cost optimization tool evaluates them together. A cost optimizer might recommend moving a workload to a cheaper region without knowing that the workload is PCI-scoped and must remain in a compliant environment.

**LOOM's Opportunity:** LOOM evaluates cost optimization within compliance constraints. When optimizing workload placement for cost, LOOM respects compliance boundaries: a PCI workload will not be moved to a non-PCI environment even if it is cheaper. LOOM finds the optimal cost within the compliant solution space, not outside it.

---

## The Fundamental Gap: Compliance as a Constraint vs. Compliance as an Assessment

The entire compliance and governance ecosystem operates on an assessment paradigm:
1. Infrastructure is provisioned
2. Compliance tools scan the infrastructure
3. Violations are reported
4. Operators remediate violations
5. Auditors verify compliance
6. Cycle repeats

This is fundamentally reactive. Even "shift-left" tools (Checkov, Terrascan, Conftest) move the assessment earlier in the pipeline but do not change the paradigm -- they still assess and report, requiring human action to remediate.

LOOM inverts this paradigm. Compliance is a constraint in the optimization function, not an assessment after the fact. Infrastructure cannot be provisioned in a non-compliant state because compliance requirements are evaluated before provisioning begins. The orchestrator will not place a HIPAA workload on non-encrypted storage, will not deploy a PCI service without network segmentation, and will not provision a FedRAMP workload outside of GovCloud regions.

This is the difference between compliance as audit (checking what was built) and compliance as infrastructure (building what is compliant). No existing tool provides the latter.

---

*Sources consulted:*
- [Open Policy Agent Documentation](https://www.openpolicyagent.org/docs/)
- [OPA Gatekeeper](https://open-policy-agent.github.io/gatekeeper/)
- [Kyverno Documentation](https://kyverno.io/docs/)
- [Falco Documentation](https://falco.org/docs/)
- [Prowler Documentation](https://docs.prowler.com/)
- [ScoutSuite](https://github.com/nccgroup/ScoutSuite)
- [Checkov Documentation](https://www.checkov.io/1.Welcome/Quick%20Start.html)
- [Terrascan Documentation](https://runterrascan.io/)
- [Cloud Custodian Documentation](https://cloudcustodian.io/docs/)
- [HIPAA Security Rule](https://www.hhs.gov/hipaa/for-professionals/security/)
- [PCI-DSS v4.0](https://www.pcisecuritystandards.org/)
- [SOC 2 Trust Services Criteria](https://www.aicpa.org/resources/landing/system-and-organization-controls-soc-suite-of-services)
- [CIS Benchmarks](https://www.cisecurity.org/cis-benchmarks)
- [FedRAMP](https://www.fedramp.gov/)
- [NIST SP 800-53](https://csrc.nist.gov/publications/detail/sp/800-53/rev-5/final)
