Now I have comprehensive research data. Let me compile this into the complete document.

---

# Comprehensive Survey of Network Operating Systems, Switch Platforms, and Management Abstractions

## The Landscape LOOM Must Orchestrate

---

## Part I: Open / Disaggregated Network Operating Systems

---

### 1. SONiC (Software for Open Networking in the Cloud)

**Architecture and Design Philosophy:**
SONiC is an open-source NOS originated by Microsoft, now governed by the Linux Foundation's SONiC Foundation. It runs on unmodified Debian Linux with a containerized microservices architecture. Each networking function (BGP via FRR, LLDP, LACP, NAT, etc.) runs in its own Docker container. A Redis database serves as the central information bus, sharing configuration and state between all containers. The critical innovation is the **Switch Abstraction Interface (SAI)** -- a vendor-independent C-language API that abstracts the underlying switching ASIC, enabling SONiC to run on hardware from Broadcom, Mellanox/NVIDIA, Barefoot/Intel, Marvell, and others without modification.

**Management APIs:**
- CLI (click-based, Klish-based in newer versions)
- REST API (via SONiC Management Framework)
- gNMI (OpenConfig and native YANG models)
- SNMP (legacy support)
- Redis DB direct access for internal state

**Programmability:** Fully open source; Python-based CLI extensions; containerized architecture allows custom application containers; SAI enables custom ASIC programming.

**Strengths:**
- True hardware/software disaggregation via SAI
- Backed by hyperscalers (Microsoft Azure, Meta, Alibaba)
- Rapidly growing ecosystem; 650 Group projects revenue growing from ~$2B to ~$4.5B
- Carrier-grade deployments (Orange France deploying across hundreds of PoPs on Edgecore hardware)
- Container-native architecture enables independent service upgrades

**Weaknesses:**
- Steep learning curve; documentation historically sparse
- Enterprise feature gaps compared to mature vendor NOS (though closing rapidly)
- Multiple SONiC distributions (community, Edgecore Enterprise, Dell, Cisco) create sub-fragmentation
- Limited campus/branch networking story

**Vendor Lock-in:** Minimal -- SAI ensures portability across ASIC vendors. However, vendor-specific SONiC distributions can introduce proprietary extensions.

**Container/VM Integration:** Native Docker architecture; runs containers natively.

**Telemetry:** gNMI streaming telemetry; Redis-based state export; What Just Happened (WJH) on NVIDIA platforms.

**Cost Model:** Open source (Apache 2.0). Enterprise distributions from Edgecore, Dell, and Cisco carry support fees.

---

### 2. Cumulus Linux (NVIDIA)

**Architecture and Design Philosophy:**
Cumulus Linux treats the network switch as a Linux server. It runs on unmodified Debian/Ubuntu Linux, meaning standard Linux tools (apt, systemctl, ip, tcpdump) work natively. Networking is provided by FRR for routing, ifupdown2 for interface management, and switchd for ASIC programming. The key management evolution: NCLU (Network Command Line Utility) was an add-on CLI layer, but **Cumulus Linux 5.0+ replaced it entirely with NVUE** -- a declarative, object-oriented configuration engine with its own data model and REST API based on OpenAPI specification.

**Management APIs:**
- NVUE CLI (declarative, tree-structured)
- NVUE REST API (OpenAPI/Swagger)
- Standard Linux tools and APIs
- SNMP
- Ansible, Puppet, Chef, SaltStack integration
- NAPALM driver available

**Programmability:** Full Linux environment; Python, Bash, any Linux-compatible language; NVUE's OpenAPI spec enables programmatic config generation.

**Strengths:**
- True Linux experience -- network engineers can use familiar Linux tooling
- NVUE provides authoritative config source with atomic commits
- Strong NVIDIA Spectrum switch integration with Cumulus NetQ for telemetry
- Air virtual lab environment for testing

**Weaknesses:**
- Increasingly tied to NVIDIA hardware ecosystem
- NVUE data model described as "incomplete" by some analysts
- Migration from NCLU to NVUE was disruptive
- Market position uncertain after NVIDIA acquisition

**Vendor Lock-in:** Moderate -- while Linux-based, NVUE and switchd are NVIDIA-specific. Moving off NVIDIA hardware requires NOS change.

**Container/VM Integration:** Native Linux enables Docker, LXC, KVM workloads on switches.

**Telemetry:** NetQ provides streaming telemetry, network validation, and What Just Happened (WJH).

**Cost Model:** Included with NVIDIA Spectrum switch purchase; community edition discontinued.

---

### 3. OpenSwitch (OPX)

**Architecture and Design Philosophy:**
OpenSwitch OPX is Dell's contribution to open networking, hosted under the Linux Foundation. It runs on unmodified Linux with a Control Plane Services (CPS) API providing a publish/subscribe model for applications, and leverages SAI for ASIC abstraction (same as SONiC).

**Management APIs:**
- Linux CLI
- CPS API (publish/subscribe, Python bindings)
- SAI for ASIC programming
- Standard Linux networking tools

**Programmability:** Full Linux; CPS API in Python and C.

**Strengths:**
- True disaggregation with SAI
- Fully open source
- Dell hardware support

**Weaknesses:**
- Project activity has significantly declined
- Community and ecosystem much smaller than SONiC
- Limited enterprise features
- Effectively superseded by SONiC and Dell OS10

**Vendor Lock-in:** Low, but limited hardware platform support.

**Cost Model:** Open source (Apache 2.0).

---

### 4. Arista EOS (Extensible Operating System)

**Architecture and Design Philosophy:**
Arista EOS runs on an unmodified Linux kernel (Fedora-based) with a single-image software architecture. Every EOS process runs in Linux user space, and the system maintains a central state database called **Sysdb** (System Database). All subsystems read/write state through Sysdb, enabling extraordinary programmability. The CLI itself is just another Sysdb client.

**Management APIs:**
- **eAPI** (JSON-RPC 2.0 over HTTP/HTTPS) -- the primary programmatic interface; can execute any CLI command and returns structured JSON
- **EOS SDK** -- native C++ SDK for on-box applications
- NETCONF/YANG
- gNMI/OpenConfig
- SNMP
- REST (via eAPI)

**Programmability:** eAPI is language-agnostic (any HTTP client works); on-box Python shell; EOS SDK for native extensions; Arista Validated Designs (AVD) Ansible collection is industry-leading.

**Strengths:**
- Single binary image across all platforms
- eAPI provides 100% feature parity with CLI
- Sysdb architecture enables consistent state across all interfaces
- CloudVision provides centralized multi-device management
- Strong OpenConfig/gNMI adoption
- Dominant in large-scale data center leaf-spine fabrics

**Weaknesses:**
- Proprietary NOS (despite Linux foundation)
- Premium pricing
- Limited service provider / carrier features compared to IOS-XR or Junos
- CloudVision adds significant cost for management plane

**Vendor Lock-in:** High -- EOS only runs on Arista hardware. eAPI is proprietary (though widely supported by automation tools).

**Container/VM Integration:** EOS supports running Docker containers on switches; CloudVision as a service.

**Telemetry:** Streaming telemetry via gNMI; OpenConfig models; CloudVision provides analytics.

**Cost Model:** Proprietary, perpetual or subscription licensing. Premium positioning.

---

### 5. VyOS

**Architecture and Design Philosophy:**
VyOS is a fork of the discontinued Vyatta Core, providing an open-source router/firewall platform. It uses a layered architecture where all networking components (firewall, VPN, routing) are built on top of a unified configuration management framework. Command definitions are written in XML (validated via RelaxNG schema), and backend config generators can be written in Python 3, Perl, or shell.

**Management APIs:**
- CLI (Junos-inspired commit/rollback model)
- HTTP API (REST)
- OpenAPI for extensions
- Cloud-init support
- NETCONF support is limited/in development

**Programmability:** Python-based backend; Ansible, Terraform, SaltStack, NAPALM, Netmiko support.

**Strengths:**
- Free and open source for rolling release
- Runs on bare metal, VMs, and all major clouds (AWS, Azure, GCP)
- Junos-like CLI familiar to network engineers
- Strong VPN/firewall feature set
- Cloud-init integration for automated deployment

**Weaknesses:**
- Small development team
- Stable releases require paid subscription
- Limited YANG/NETCONF maturity
- No gNMI support
- Primarily router/firewall, not a switch OS

**Vendor Lock-in:** None -- runs on any x86 hardware or VM.

**Cost Model:** Rolling release is free; stable/LTS releases require subscription.

---

### 6. OPNsense / pfSense

**Architecture and Design Philosophy:**
Both are FreeBSD-based open-source firewall/router platforms. OPNsense forked from pfSense in 2015 and adopted a modern MVC architecture with HardenedBSD as its base. pfSense uses standard FreeBSD with a PHP-based web interface. OPNsense emphasizes API-first design and plugin extensibility.

**Management APIs:**
- **OPNsense**: Comprehensive REST API, CLI, plugin system, Ansible support
- **pfSense**: Primarily GUI-driven; limited API (pfSense Plus has improved API); XML-RPC for HA sync

**Programmability:** OPNsense is significantly more automation-friendly with its API; pfSense is more GUI-centric.

**Strengths:**
- Mature firewall/router platforms with large communities
- OPNsense: weekly security updates, LibreSSL, modern MVC architecture
- pfSense: larger install base, Netgate hardware integration
- Both: excellent VPN, IDS/IPS, traffic shaping

**Weaknesses:**
- Not designed for switch fabric or data center use
- Limited scalability compared to enterprise firewalls
- pfSense licensing controversy (pfSense CE vs pfSense Plus)
- Neither supports gNMI, NETCONF, or YANG

**Vendor Lock-in:** OPNsense: none. pfSense: Netgate increasingly pushing pfSense Plus (proprietary).

**Cost Model:** OPNsense: fully open source (BSD license). pfSense CE: open source; pfSense Plus: proprietary (free for home use).

---

### 7. OpenWrt

**Architecture and Design Philosophy:**
OpenWrt is a Linux-based embedded OS for network devices (primarily consumer/prosumer routers). It uses a minimal Linux system (musl libc, BusyBox) optimized for constrained hardware. Configuration is managed through UCI (Unified Configuration Interface), and the web interface is LuCI (Lua/ucode-based MVC framework communicating via JSON-RPC).

**Management APIs:**
- LuCI web interface (JSON-RPC backend)
- UCI CLI for configuration
- ubus (micro bus) IPC system
- SSH/CLI
- SNMP (optional package)

**Programmability:** Shell scripting; Lua/ucode for LuCI extensions; ubus provides IPC API; extensive package system (opkg).

**Strengths:**
- Runs on thousands of hardware devices
- Massive community and package ecosystem
- Extremely lightweight and customizable
- Foundation for many commercial products

**Weaknesses:**
- No enterprise-grade management APIs
- No NETCONF, gNMI, or YANG support
- Limited to consumer/SOHO hardware
- Fragmented device support quality
- No structured API for automation at scale

**Vendor Lock-in:** None -- runs on any supported hardware.

**Cost Model:** Fully open source (GPL).

---

## Part II: Vendor Network Operating Systems

---

### 8. Cisco IOS / IOS-XE / IOS-XR / NX-OS / ACI

Cisco operates **five distinct NOS families**, each with different architectures, CLIs, and APIs. This alone represents the fragmentation problem in microcosm.

#### IOS (Classic)
- Monolithic, single-process architecture
- Legacy CLI (the "original" network CLI that defined the industry)
- Limited programmability; SNMP-centric management
- End-of-life on new platforms; persists on legacy hardware

#### IOS-XE
- Modernized IOS on a Linux kernel; modular architecture
- Powers Catalyst 9000, ASR routers, CSR1000V
- **APIs**: NETCONF, RESTCONF, gNMI, YANG models, SNMP, CLI
- Model-driven telemetry; Day-2 programmability via Python on-box
- Guest shell for running containers

#### IOS-XR
- Carrier-grade; originally QNX microkernel, migrated to 64-bit Linux (6.x+)
- Powers NCS, ASR 9000, Cisco 8000 series
- **APIs**: NETCONF, gRPC, gNMI, YANG, service-layer APIs (SL-API for direct RIB/FIB manipulation)
- Segment Routing native support
- Model-driven streaming telemetry at scale

#### NX-OS
- Data center focused; powers Nexus switches
- Multiprocess, SMP architecture (not shared memory like IOS)
- **APIs**: NX-API (REST/JSON-RPC), NETCONF, gRPC, gNMI, YANG, OpenConfig
- On-box Python, Bash; container hosting (Docker on Nexus 9000)
- PowerOn Auto Provisioning (POAP)

#### ACI (Application Centric Infrastructure)
- Entirely different paradigm: policy-driven fabric with centralized APIC controller
- **APIs**: APIC REST API (JSON/XML over HTTP); object-model driven
- The entire fabric state is represented as a Management Information Tree (MIT)
- Same REST API backs CLI, GUI, and SDK
- Ansible cisco.aci collection for automation

**Strengths:**
- Broadest feature set in the industry
- Massive install base and ecosystem
- Strong programmability on modern platforms (IOS-XE, IOS-XR, NX-OS)
- ACI provides true intent-based fabric management

**Weaknesses:**
- **Five different NOS families with different CLIs, APIs, and data models** -- this is the quintessential fragmentation problem
- Licensing complexity (Smart Licensing, DNA licenses, ACI licenses)
- ACI's object model has an extremely steep learning curve
- Migration between NOS families is non-trivial
- Vendor lock-in is very high, especially with ACI

**Telemetry:** Model-driven telemetry across IOS-XE, IOS-XR, NX-OS; APIC provides fabric-wide analytics.

**Cost Model:** Proprietary; subscription licensing (Cisco DNA, Smart Licensing); ACI requires APIC + Nexus fabric.

---

### 9. Juniper Junos

**Architecture and Design Philosophy:**
Junos is built on a **single modular codebase** across all Juniper platforms (routers, switches, firewalls). Strong separation between control plane, forwarding plane, and management processes. The **commit model** (candidate config -> commit -> running config, with rollback) is a defining feature. Junos Evolved is the newer variant running on native Linux.

**Management APIs:**
- CLI (the gold standard for operational CLIs)
- NETCONF (Juniper was the primary NETCONF RFC contributor; native support)
- gNMI (OpenConfig and Juniper native YANG models)
- gRPC (Juniper Telemetry Interface / JTI)
- REST API (RESTful API service)
- XML-RPC
- PyEZ (Python library built on NETCONF)

**Programmability:** Junos automation includes: on-box Python, SLAX scripting, commit scripts, op scripts, event scripts. PyEZ 3.1 (2025) introduces async RPCs for sub-10ms latencies in 400G environments. Junos Evolved introduces genstate YANG data models (24.2R1+).

**Strengths:**
- Single OS across all platforms -- strongest single-codebase story in the industry
- Commit/rollback model prevents configuration drift
- NETCONF pioneer; deepest NETCONF/YANG implementation
- Excellent CLI consistency
- Strong service provider features

**Weaknesses:**
- Proprietary and premium priced
- Junos vs Junos Evolved creates a new fragmentation axis
- Market share declining relative to Arista in data center
- Mist AI cloud management adds another management paradigm

**Vendor Lock-in:** High -- Junos only runs on Juniper hardware.

**Telemetry:** JTI (gRPC streaming), gNMI, SNMP. OpenConfig telemetry models supported.

**Cost Model:** Proprietary; perpetual and subscription licensing; AppSecure and advanced features require additional licenses.

---

### 10. Nokia SR OS / SR Linux

Nokia operates **two distinct NOS platforms**:

#### SR OS (Service Router Operating System)
- Powers 7750 SR, 7250 IXR (carrier/service provider)
- 64-bit SMP architecture; 15+ years of carrier-grade deployment
- **APIs**: MD-CLI (model-driven CLI), NETCONF, gRPC/gNMI -- all share the same underlying YANG models
- SDN support via BGP-LS, PCEP, Segment Routing

#### SR Linux (Service Router Linux)
- The modern, cloud-native NOS for data center fabrics
- **Container-based microservices** on unmodified Linux kernel
- Each application has its own YANG model defining config and state
- **APIs**: gNMI (primary), JSON-RPC, CLI -- all natively model-driven
- Nokia Impart Database (IDB) for pub/sub state sharing via protobufs/gRPC
- OpenConfig YANG models supported alongside native models
- **Containerlab** integration for lab/testing environments

**Strengths:**
- SR Linux is arguably the most modern NOS architecture (fully model-driven, container-native)
- gNMI-first design on SR Linux
- Unified YANG model foundation across all management interfaces
- Strong Segment Routing and EVPN implementation
- Containerlab enables DevOps-style network testing

**Weaknesses:**
- Two NOS platforms (SR OS and SR Linux) with different architectures
- Smaller ecosystem than Cisco/Juniper/Arista
- Limited community tooling compared to SONiC or Arista AVD
- Nokia's enterprise market presence is limited

**Vendor Lock-in:** High for SR OS; SR Linux is more open but Nokia-hardware only.

**Telemetry:** gNMI streaming telemetry (native, no translation layers on SR Linux); 100% of state is YANG-modeled.

**Cost Model:** Proprietary; bundled with hardware.

---

### 11. Huawei VRP (Versatile Routing Platform)

**Architecture and Design Philosophy:**
VRP is Huawei's unified NOS across routers, switches, and firewalls. It uses a distributed, multi-process, component-based architecture with a layered design: hardware drivers -> RTOS and task scheduling -> IP/ATM forwarding -> routing policy management -> service layer.

**Management APIs:**
- CLI (Cisco-influenced syntax)
- NETCONF/YANG
- SNMP
- OPS (Open Programmability System) -- Python-based on-box scripting
- RESTful API on newer platforms (CloudEngine)

**Programmability:** OPS provides Python scripting; NETCONF/YANG for model-driven management; Ansible support via community modules.

**Strengths:**
- Runs across Huawei's full product line
- Strong carrier/SP feature set
- Competitive pricing, especially in APAC/EMEA/Africa markets
- CloudEngine series has modern programmability

**Weaknesses:**
- Geopolitical restrictions (banned in some markets -- US, UK, Australia for 5G)
- Documentation quality in English can be inconsistent
- Smaller automation community/ecosystem
- Proprietary and closed source

**Vendor Lock-in:** Very high -- both software and geopolitical dependency.

**Cost Model:** Proprietary; competitive pricing.

---

### 12. HPE/Aruba AOS-CX

**Architecture and Design Philosophy:**
AOS-CX is designed as a "cloud-native" NOS with a **microservices architecture** and a central state database. The defining feature is **100% REST API coverage** -- every configuration and state element accessible via the CLI is also available through the REST API. Built on the premise that the network switch should be as programmable as a cloud server.

**Management APIs:**
- CLI
- REST API (100% coverage, OpenAPI/Swagger documented)
- SNMP
- NETCONF (limited)
- Aruba Central (cloud management platform)
- Python on-box interpreter
- NAE (Network Analytics Engine) scripts

**Programmability:** Full REST API with Swagger documentation; Python on-box; NAE for programmable analytics; NAPALM driver available; Ansible modules.

**Strengths:**
- 100% REST API parity with CLI is unique in the industry
- Modern microservices architecture
- Strong campus networking story (Aruba wireless + wired integration)
- Network Analytics Engine for on-box programmable monitoring

**Weaknesses:**
- Data center story less mature than Arista or SONiC
- Limited gNMI/OpenConfig support
- Aruba Central adds another management layer/cost
- Smaller automation community

**Vendor Lock-in:** Moderate -- REST API is well-documented but AOS-CX only runs on HPE/Aruba hardware.

**Telemetry:** NAE for on-box analytics; SNMP; limited streaming telemetry compared to competitors.

**Cost Model:** Proprietary; included with hardware; Aruba Central requires subscription.

---

### 13. MikroTik RouterOS

**Architecture and Design Philosophy:**
RouterOS is MikroTik's proprietary NOS running on MikroTik hardware (RouterBOARD) and x86 platforms. It provides an extremely feature-rich networking stack at aggressive price points. The architecture is monolithic but highly configurable.

**Management APIs:**
- CLI (terminal)
- WinBox (proprietary GUI, uses a binary API protocol)
- **RouterOS API** (custom binary protocol on TCP:8728/8729)
- **REST API** (added in RouterOS v7.1, JSON wrapper over console API)
- SNMP
- SSH

**Programmability:** RouterOS scripting language (built-in); REST API (v7+); Python libraries (RouterOS-api, Netmiko); Ansible limited support.

**Strengths:**
- Extraordinary price/performance ratio
- Feature-rich (MPLS, BGP, OSPF, firewall, VPN, QoS, wireless)
- Huge install base, especially in ISPs and developing markets
- RouterOS v7 REST API is a significant programmability improvement

**Weaknesses:**
- Proprietary binary API protocol (pre-v7) is non-standard
- No NETCONF, gNMI, or YANG support
- No OpenConfig
- Limited scalability for large deployments
- Security track record has been mixed
- WinBox binary protocol is opaque

**Vendor Lock-in:** Moderate -- RouterOS runs on MikroTik and x86, but no standard management interfaces.

**Telemetry:** SNMP; The Dude monitoring tool; limited streaming telemetry.

**Cost Model:** Extremely affordable; license included with hardware; x86 licenses from $45.

---

### 14. Ubiquiti EdgeOS / UniFi OS

**Architecture and Design Philosophy:**
Ubiquiti operates two product lines with different management paradigms:
- **EdgeOS** (EdgeRouter/EdgeSwitch) -- Vyatta/VyOS fork with CLI-driven management
- **UniFi OS** (UniFi gateways, switches, APs) -- centralized controller-based management

UniFi uses a hybrid cloud architecture with a local control plane and optional cloud management via unifi.ui.com.

**Management APIs:**
- **EdgeOS**: CLI (Vyatta-derived), limited HTTP API
- **UniFi**: Controller REST API (JSON), Site Manager API, SNMP
- No NETCONF, gNMI, or YANG on either platform

**Programmability:** UniFi API is undocumented/unofficial (community-reverse-engineered for years; official API recently released). EdgeOS has limited scripting. Ansible community modules exist.

**Strengths:**
- Extremely competitive pricing
- UniFi provides integrated wireless + wired + security management
- Simple deployment for SMB/prosumer
- Growing enterprise adoption

**Weaknesses:**
- UniFi API was historically undocumented
- No standard network management protocols (NETCONF, gNMI)
- EdgeOS/EdgeMAX product line appears deprioritized
- Limited scalability and feature depth for enterprise
- Controller dependency for UniFi

**Vendor Lock-in:** High -- proprietary management, no standard APIs.

**Cost Model:** Very affordable hardware; UniFi controller is free (self-hosted) or cloud.

---

## Part III: Bare-Metal / White-Box Switch Hardware

---

### 15. Edgecore Networks

**Role:** ODM (Original Design Manufacturer) and leading white-box switch vendor. Subsidiary of Accton Technology.

**Hardware Portfolio:** 1G to 400G switches covering leaf, spine, and super-spine roles. Key platforms include AS7726-32X (100G), AS9516-32D (400G), and the AS9726-32DB for AI/HPC fabrics.

**NOS Support:** SONiC (Edgecore Enterprise SONiC distribution), Cumulus Linux, IP Infusion OcNOS, DENT, and other ONIE-compatible NOS.

**Key Differentiator:** Edgecore provides "tested and hardened" enterprise SONiC distribution with TAC support. Orange France's large-scale SONiC deployment runs on Edgecore hardware.

**Cost Model:** Hardware-only purchase; NOS choice is separate. Significant CapEx savings vs. integrated vendor switches.

---

### 16. Dell OS10 (SmartFabric OS10)

**Architecture:** Linux-based NOS with SAI and CPS (Control Plane Services) abstraction layers. Enterprise Edition provides full L2/L3 switching with a traditional CLI. Open Edition (based on OPX) provides bare-metal Linux networking.

**Management APIs:** CLI, REST API, YANG/NETCONF, Python/C/C++ programmability.

**NOS Flexibility:** Dell PowerSwitch hardware supports OS10 Enterprise Edition, SONiC, Cumulus Linux, and other ONIE-compatible NOS.

**Cost Model:** OS10 Enterprise included with Dell PowerSwitch; open edition is free.

---

### 17. NVIDIA/Mellanox Spectrum + Onyx/Cumulus

**Hardware:** NVIDIA Spectrum (1/2/3/4) ASICs power SN-series switches optimized for HPC, AI/ML, and storage fabrics.

**NOS Options:**
- **Onyx** (legacy Mellanox NOS) -- traditional CLI/SNMP management with VXLAN/EVPN
- **Cumulus Linux** (preferred path forward) -- Linux-native NOS
- **SONiC** -- also supported on Spectrum platforms

**Key Differentiator:** WJH (What Just Happened) provides real-time packet-level telemetry by reading discard reasons directly from the ASIC. FlexFlow packet processing technology. Spectrum SDK for direct data-plane programming.

**Cost Model:** Onyx included with hardware; Cumulus bundled with Spectrum switches; SONiC is open source.

---

### 18-19. Celestica / Accton (Edgecore Parent)

**Celestica:** Carrier-grade white-box hardware (DS series) trusted by telecom and hyperscale operators. DS3000 features modular COM-E control plane and BMC management plane. Focus on AI/ML and ultra-high-throughput environments.

**Accton Technology:** Parent company of Edgecore; one of the world's largest ODMs for networking equipment. Manufactures for both its Edgecore brand and other OEMs.

Both support ONIE for NOS installation, SAI-compatible ASICs, and work with SONiC, Cumulus, and other open NOS platforms.

---

## Part IV: Management Abstractions and Automation Frameworks

---

### 20. NAPALM (Network Automation and Programmability Abstraction Layer with Multivendor support)

**Architecture:** Python library providing a unified API across multiple vendor platforms. Each vendor has a "driver" that translates unified NAPALM calls into vendor-specific operations. Core drivers use different underlying protocols: NX-API for Nexus, eAPI for Arista, SSH for IOS, NETCONF for Junos.

**Supported Platforms (Core):** Arista EOS, Cisco IOS/IOS-XR/NX-OS, Juniper Junos. Community drivers extend to VyOS, RouterOS, AOS-CX, FortiOS, and others.

**Key Operations:** get_facts, get_interfaces, get_bgp_neighbors, load_merge_candidate, load_replace_candidate, commit_config, rollback, compare_config.

**Strengths:**
- Elegant abstraction for common operations
- Integration with Salt, Ansible, Nautobot/NetBox
- Community driver ecosystem

**Weaknesses:**
- Lowest-common-denominator problem: unified API can only expose features common to all platforms
- Core driver coverage is limited (5-6 vendors)
- No real-time telemetry support
- Read operations return inconsistent data structures across drivers
- Some drivers are poorly maintained

---

### 21. Nornir

**Architecture:** Pure Python automation framework (not DSL-based like Ansible). Provides inventory management, concurrent task execution, and a plugin architecture. Since v3.0, ships with no built-in plugins -- everything is installed via pip (nornir_napalm, nornir_netmiko, nornir_scrapli, nornir_jinja2).

**Strengths:**
- Full Python control; no DSL limitations
- Threaded execution for parallelism
- Plugin architecture keeps core lightweight
- Can leverage any Python library
- Excellent for complex logic that Ansible playbooks struggle with

**Weaknesses:**
- Requires Python development skills
- Smaller community than Ansible
- No built-in device support (requires plugins)
- Less "turnkey" than Ansible for simple tasks

---

### 22. Netmiko

**Architecture:** Python SSH library built on Paramiko, providing simplified SSH connections to network devices. Handles login, privilege escalation, command sending, and output parsing for 100+ device types.

**Supported Platforms:** ~100 device types including Cisco IOS/IOS-XE/IOS-XR/NX-OS/ASA, Arista, Juniper, HP ProCurve, MikroTik, Huawei, and many more.

**Strengths:**
- Broadest device support of any SSH library
- Simple API for CLI interaction
- Handles vendor-specific SSH quirks (paging, prompts, timing)
- Well-maintained by Kirk Byers

**Weaknesses:**
- Screen-scraping paradigm (unstructured text output)
- No native structured data; requires TextFSM/TTP for parsing
- Slower than Scrapli for multi-host operations
- Synchronous only

---

### 23. Scrapli

**Architecture:** Modern Python SSH/NETCONF library designed as a faster, more flexible alternative to Netmiko. Uses the system's native SSH binary by default (vs. Paramiko). Built-in async support via asyncssh.

**Supported Platforms:** Focused on 5 core platforms (Cisco IOS-XE, IOS-XR, NX-OS, Arista EOS, Juniper Junos) with a generic driver for others.

**Strengths:**
- Significantly faster than Netmiko for multi-host operations
- Native async support (asyncssh transport)
- Clean API with privilege level handling
- Built-in TextFSM, Genie, TTP parsing
- scrapli_netconf for NETCONF operations

**Weaknesses:**
- Narrower core device support (5 platforms vs. Netmiko's 100+)
- Smaller community
- Less battle-tested in production at scale

---

### 24. Ansible Network Modules

**Architecture:** Ansible provides network automation through platform-specific collections (cisco.ios, cisco.nxos, cisco.iosxr, arista.eos, junipernetworks.junos, etc.). Uses the "network resource module" pattern where each module manages a specific resource (interfaces, VLANs, BGP, ACLs) with a declarative state model (merged, replaced, overridden, deleted).

**Supported Platforms:** Extensive -- Cisco (IOS, IOS-XE, IOS-XR, NX-OS, ACI, Meraki), Juniper, Arista, Nokia, F5, Palo Alto, Fortinet, Aruba, VyOS, and many more. Red Hat Certified content ecosystem saw "exponential growth in 2025."

**Strengths:**
- Largest automation ecosystem in networking
- Declarative resource modules abstract vendor differences
- Event-Driven Ansible for reactive automation
- Red Hat support and certification
- Agentless (SSH/NETCONF/API-based)

**Weaknesses:**
- YAML/DSL limitations for complex logic
- Performance issues at scale (sequential by default; mitigation via async/forks)
- Ivan Pepelnjak (ipSpace.net) argues Red Hat "abandoned network automation" in late 2025
- Module quality varies significantly across vendors
- Debugging playbook failures can be challenging

**Cost Model:** Ansible Community: free. Ansible Automation Platform: Red Hat subscription.

---

### 25. Batfish

**Architecture:** A network configuration analysis and validation tool. Ingests configuration snapshots (offline -- no device access required) and builds vendor-agnostic data models including routing tables, ACLs, and topology. Uses BDD (Binary Decision Diagrams) for efficient analysis. Runs as a Docker container with pybatfish Python client.

**Supported Platforms:** Parses configurations from Cisco (IOS, IOS-XR, NX-OS, ASA), Juniper (Junos), Arista (EOS), Palo Alto, F5, Cumulus, SONiC, and others.

**Strengths:**
- Pre-deployment validation -- find bugs before pushing config
- No device access needed; works on config files
- Vendor-agnostic analysis model
- Recognized with SIGCOMM Networking Systems Award
- Scales to thousands of nodes

**Weaknesses:**
- Offline analysis only -- no real-time state awareness
- Parser fidelity varies by vendor/feature
- Does not generate or push configurations
- Requires config file collection infrastructure

---

## Part V: Comprehensive Gaps Analysis

---

### Gap 1: The NOS Fragmentation Problem

The research reveals a staggering diversity of management interfaces:

**CLI Dialects (at least 15+ distinct syntaxes):**
| Family | CLI Style | Example |
|--------|----------|---------|
| Cisco IOS/IOS-XE | `configure terminal` / `interface gi0/0` | Hierarchical, mode-based |
| Cisco NX-OS | IOS-like but different keywords | `feature bgp` paradigm |
| Cisco ACI | Object-model CLI or APIC GUI | Completely different from IOS |
| Juniper Junos | `set`/`delete` with hierarchical commit model | `set interfaces ge-0/0/0 unit 0` |
| Arista EOS | IOS-like (intentionally compatible) | Subtle differences from IOS |
| Nokia SR OS | MD-CLI with `/configure` tree | Own syntax family |
| Nokia SR Linux | Flat, model-driven CLI | Yet another Nokia syntax |
| Huawei VRP | Cisco-influenced but distinct | `system-view` instead of `conf t` |
| MikroTik | `/ip address add` path-style | Unique slash-path syntax |
| Cumulus/NVUE | `nv set` declarative tree | OpenAPI-derived |
| VyOS | `set`/`delete` Junos-like | Vyatta heritage |
| HPE AOS-CX | Cisco-influenced with differences | Modern resource-oriented |
| OPNsense | Web GUI primary; shell secondary | FreeBSD-based |
| OpenWrt | UCI (`uci set`) + ash shell | Embedded Linux style |
| SONiC | Click-based / Klish-based | Still evolving |

**API Protocol Matrix:**
| Platform | REST | NETCONF | gNMI | SNMP | gRPC | Custom |
|----------|------|---------|------|------|------|--------|
| Cisco IOS-XE | RESTCONF | Yes | Yes | Yes | -- | -- |
| Cisco NX-OS | NX-API | Yes | Yes | Yes | -- | -- |
| Cisco ACI | APIC REST | -- | -- | Yes | -- | Object Model |
| Juniper Junos | REST | Yes (native) | Yes | Yes | JTI | XML-RPC |
| Arista EOS | eAPI | Yes | Yes | Yes | -- | -- |
| Nokia SR Linux | -- | Yes | Yes (primary) | Yes | gRPC | JSON-RPC |
| Nokia SR OS | -- | Yes | Yes | Yes | gRPC | MD-CLI |
| Huawei VRP | REST (newer) | Yes | Limited | Yes | -- | OPS |
| HPE AOS-CX | REST (100%) | Limited | No | Yes | -- | -- |
| MikroTik | REST (v7+) | No | No | Yes | -- | Binary API |
| VyOS | HTTP API | Limited | No | Yes | -- | -- |
| SONiC | REST | -- | Yes | Yes | -- | Redis |
| Cumulus | NVUE REST | -- | -- | Yes | -- | Linux native |
| OPNsense | REST | No | No | -- | -- | -- |
| OpenWrt | -- | No | No | SNMP | -- | JSON-RPC/ubus |
| Ubiquiti | REST | No | No | Yes | -- | -- |

**This means any universal orchestrator faces 15+ CLI dialects, 6+ API protocols, and dozens of data model variants.**

---

### Gap 2: The Abstraction Challenge -- How Tools Try But Fall Short

**NAPALM** provides the cleanest abstraction but suffers from the **lowest-common-denominator problem**. Its unified API can only expose operations that work across all supported platforms. Platform-specific features (Cisco ACI policies, Arista CloudVision integration, Nokia SR Linux's gNMI-native telemetry) are inaccessible. Core driver support covers only 5-6 vendors; community drivers are inconsistently maintained.

**Nornir** avoids the LCD problem by being a framework, not an abstraction -- but this means every task must be written with vendor awareness. It pushes complexity to the developer.

**Netmiko** provides the broadest device coverage (100+ types) but is fundamentally a screen-scraping tool. Parsing unstructured CLI output is fragile and breaks with software updates.

**Scrapli** is faster and more modern but covers only 5 core platforms, leaving the long tail of vendors unaddressed.

**Ansible** has the broadest module ecosystem but suffers from inconsistent module quality across vendors, YAML's limitations for complex logic, and potential organizational deprioritization of network automation.

**Batfish** provides excellent offline validation but cannot generate configurations or interact with live devices.

**The fundamental gap:** No existing tool provides both broad vendor coverage AND deep, vendor-specific feature access AND real-time state awareness AND configuration generation. Each tool makes a different tradeoff on this spectrum.

---

### Gap 3: The Disaggregated vs. Proprietary Divide

The industry is split along a deep architectural fault line:

**Disaggregated / Open:**
- SONiC, Cumulus, OPX on white-box hardware (Edgecore, Celestica, Dell)
- SAI provides ASIC abstraction; ONIE provides NOS portability
- Customer chooses hardware independently from software
- Rapidly growing (driven by hyperscaler demand and AI infrastructure)

**Vertically Integrated / Proprietary:**
- Cisco (IOS-XE/NX-OS/ACI), Juniper (Junos), Arista (EOS), Nokia (SR OS)
- Hardware and software tightly coupled
- Deep feature integration and single-vendor support
- Still dominant in enterprise and service provider

**The Gap:** Management tools tend to target one side or the other. SONiC has its own management ecosystem; Cisco ACI has APIC; Juniper has Junos Space. An orchestrator serving both worlds must bridge fundamentally different operational models:
- Disaggregated: Linux-native, container-based, open APIs, operator-assembled
- Proprietary: Vendor-managed, feature-rich, single-throat-to-choke, closed ecosystem

---

### Gap 4: How LOOM's Adapter Layer Could Provide Unified Network Orchestration

LOOM's adapter architecture could address the fragmentation problem through a multi-layered approach:

**Layer 1 -- Protocol Adapters:**
Abstract the transport layer. A single adapter interface that speaks:
- SSH/CLI (via Netmiko/Scrapli underneath)
- NETCONF/YANG (via ncclient/scrapli_netconf)
- gNMI/gRPC (via gNMIc or similar)
- REST/HTTP (native HTTP clients)
- Vendor-specific protocols (NX-API, eAPI, APIC REST, MikroTik API, RouterOS REST)

**Layer 2 -- Data Model Normalization:**
Translate vendor-specific data models into a LOOM-internal canonical model, inspired by but going beyond OpenConfig:
- Interface model that normalizes Cisco `GigabitEthernet0/0`, Juniper `ge-0/0/0`, Arista `Ethernet1`, Nokia `ethernet-1/1`, etc.
- Routing model that normalizes BGP/OSPF configuration across all vendors
- Security policy model that spans ACLs, firewall rules, and ACI contracts
- Telemetry model that normalizes streaming data across gNMI, JTI, NX-API telemetry

**Layer 3 -- Capability Registry:**
Not all platforms support all features. LOOM must maintain a dynamic capability matrix:
- "Can this device do VXLAN/EVPN?" (Yes: Arista, NX-OS, Junos, SR Linux; No: MikroTik, OpenWrt)
- "Does this device support gNMI streaming telemetry?" (Yes: EOS, Junos, NX-OS, SR Linux; No: MikroTik, pfSense)
- "Can I do atomic config replace?" (Yes: Junos, EOS, IOS-XE; Partial: NX-OS; No: classic IOS)

**Layer 4 -- Intent Translation:**
Convert high-level intents ("create a VXLAN fabric between these leaf switches") into vendor-specific configurations, handling:
- Syntax differences (same BGP config is expressed completely differently on IOS-XE vs. Junos vs. EOS)
- Feature gaps (if a device doesn't support a feature, what's the workaround?)
- Validation (pre-deployment checks via Batfish-like analysis)

---

### Gap 5: Where LLM-Driven Heuristics Could Help

Research in 2025-2026 demonstrates concrete opportunities:

**1. Intent-to-Configuration Translation:**
LLMs can translate natural language intents into vendor-specific configurations. The INTA framework (arxiv 2501.08760) uses LLM agents with "configuration intent" as an intermediate representation, achieving translation between NOS dialects. A system integrated with ETSI TeraFlowSDN achieved 93% factual accuracy in intent-based operations.

**2. Cross-Vendor Config Translation:**
An LLM trained on Cisco IOS, Junos, EOS, NX-OS, and SR Linux configurations can translate between dialects. Example: "Convert this Junos BGP configuration to equivalent Cisco IOS-XE configuration, accounting for syntax differences in address-family, route-policy vs. route-map, and community handling."

**3. Anomaly Detection and Root Cause Analysis:**
MeshAgent (SIGMETRICS 2026) demonstrates LLM-based reliable network management. LLMs can correlate telemetry streams, syslog messages, and configuration state across heterogeneous devices to identify issues that span vendor boundaries.

**4. Configuration Validation and Compliance:**
LLMs can review generated configurations against best practices, security policies, and vendor-specific gotchas. This goes beyond static analysis (Batfish) to include contextual understanding: "This BGP config will work, but the timer values are suboptimal for this topology."

**5. Documentation and Knowledge Synthesis:**
Network engineers must constantly reference vendor-specific documentation. An LLM can serve as a universal knowledge interface: "How do I configure EVPN Type-5 routes on Nokia SR Linux vs. Arista EOS?"

**Key Limitations:**
- Hallucination risk is critical in network config (one wrong subnet mask = outage)
- LLMs must be grounded via RAG with vendor documentation
- Human-in-the-loop validation is essential for production configs
- Fine-tuning on network-specific corpora significantly outperforms general-purpose models

---

### Gap 6: From "Managing Switches" to "Orchestrating Network Capacity"

The deepest gap is conceptual. Current tools -- NAPALM, Netmiko, Ansible, even vendor controllers -- operate at the **device configuration level**: "configure interface X on switch Y." LOOM's opportunity is to operate at the **capacity orchestration level**:

**Current State (Device-Centric):**
- "Configure VLAN 100 on switch A, port 5"
- "Set BGP neighbor 10.0.0.1 on router B"
- "Create ACL rule on firewall C"
- Each device is managed individually or in batches

**LOOM's Vision (System-Centric):**
- "Provide 100Gbps of low-latency capacity between compute cluster A and storage array B"
- "Ensure workload X has isolated network paths with 99.99% availability"
- "Scale network capacity for training job Y across 1000 GPUs, using whatever switches and paths are available"
- The network is a pool of capacity, not a collection of devices

**What's Missing to Bridge This Gap:**
1. **Topology awareness**: Understanding the physical and logical topology across heterogeneous devices (LLDP/CDP discovery + config parsing + cable management integration)
2. **Capacity modeling**: Knowing available bandwidth, latency characteristics, and failure domains across the entire fabric
3. **Workload-to-network mapping**: Translating compute/storage workload requirements into specific network configurations across potentially dozens of different NOS platforms
4. **Cross-domain orchestration**: A single GPU training job might traverse SONiC leaf switches, Arista spine switches, a Cisco ACI fabric, and a Juniper WAN router -- each requiring different config in different syntax via different APIs
5. **Feedback loops**: Real-time telemetry from heterogeneous sources (gNMI from some, SNMP from others, REST polls from others) feeding back into capacity decisions
6. **Failure domain awareness**: Understanding that a Broadcom Memory Corruption ASIC bug affects SONiC + Cumulus + OPX switches on that ASIC, while Spectrum-based switches are unaffected

**LOOM's unique position:** By integrating compute, storage, and network orchestration in a single system, LOOM can close the loop between "what the workload needs" and "what the network provides" -- something no network-only tool can do. The adapter layer must be broad enough to speak to MikroTik and Cisco ACI alike, while the orchestration layer must be intelligent enough to treat them as pooled capacity rather than individual devices.

---

## Summary Statistics

| Category | Count | Examples |
|----------|-------|---------|
| Distinct NOS Platforms | 20+ | IOS-XE, Junos, EOS, NX-OS, SONiC, SR Linux, RouterOS, VRP... |
| Distinct CLI Syntaxes | 15+ | Cisco-style, Junos-style, MikroTik path-style, NVUE declarative... |
| API Protocols in Use | 8+ | REST, NETCONF, gNMI, SNMP, gRPC, SSH/CLI, JSON-RPC, custom binary |
| YANG Model Variants | 5+ | OpenConfig, vendor-native (per vendor), IETF standard |
| Automation Frameworks | 6+ | NAPALM, Nornir, Netmiko, Scrapli, Ansible, Salt |
| White-Box HW Vendors | 5+ | Edgecore, Celestica, Dell, UfiSpace, Accton |
| Vendor Controller Platforms | 5+ | APIC, CloudVision, Junos Space, Aruba Central, UniFi Controller |

**The conclusion is inescapable: network infrastructure is profoundly heterogeneous. Any system claiming to be a universal orchestrator must either support this diversity natively or accept that it only serves a subset of the market. LOOM's adapter layer, augmented by LLM-driven heuristics for cross-vendor translation and intent decomposition, represents arguably the most promising architectural approach to this unsolved problem.**

---

Sources:
- [SONiC Architecture - Sonic Foundation](https://sonicfoundation.dev/deep-dive-into-sonic-architecture-design/)
- [SONiC Architecture - Cisco DevNet](https://developer.cisco.com/docs/sonic/sonic-architecture/)
- [NVIDIA Cumulus Linux](https://www.nvidia.com/en-us/networking/ethernet-switching/cumulus-linux/)
- [Cumulus Linux NVUE Documentation](https://docs.nvidia.com/networking-ethernet-software/cumulus-linux-44/System-Configuration/NVIDIA-User-Experience-NVUE/)
- [Arista EOS Product Page](https://www.arista.com/en/products/eos)
- [Arista eAPI Whitepaper](https://www.arista.com/assets/data/pdf/Whitepapers/Arista_eAPI_FINAL.pdf)
- [Cisco IOS XE vs IOS XR Comparison](https://network-switch.com/blogs/networking/cisco-ios-xe-vs-ios-xr-2025-edition)
- [Comparing Cisco IOS, NX-OS, and IOS-XR](https://www.kwtrain.com/blog/comparing-cisco-ios-nx-os-and-ios-xr)
- [Cisco APIC REST API Guide](https://developer.cisco.com/docs/aci/)
- [Juniper Junos NETCONF Developer Guide](https://www.juniper.net/documentation/us/en/software/junos/netconf/index.html)
- [Nokia SR OS](https://www.nokia.com/ip-networks/service-router-operating-system-nos/)
- [Nokia SR Linux Overview](https://documentation.nokia.com/srlinux/25-3/books/product-overview/about-sr-linux.html)
- [Huawei VRP NETCONF Architecture](https://info.support.huawei.com/hedex/api/pages/EDOC1100277644/AEM10221/04/resources/software/nev8r10_vrpv8r16/user/galaxy/vrp_netconf_cfg1_0004.html)
- [HPE Aruba AOS-CX Developer Portal](https://developer.arubanetworks.com/aoscx/docs/about)
- [MikroTik RouterOS REST API](https://help.mikrotik.com/docs/spaces/ROS/pages/47579162/REST+API)
- [VyOS Datasheet 2025](https://vyos.io/files/vyos-datasheet-v1.4.2.pdf)
- [OPNsense vs pfSense Comparison](https://datazone.de/en/aktuelles/opnsense-vs-pfsense/)
- [OpenWrt LuCI Framework](https://deepwiki.com/openwrt/luci)
- [Ubiquiti UniFi API](https://help.ui.com/hc/en-us/articles/30076656117655-Getting-Started-with-the-Official-UniFi-API)
- [NAPALM GitHub](https://github.com/napalm-automation/napalm)
- [Nornir Documentation](https://nornir.readthedocs.io/)
- [Scrapli vs Netmiko Comparison](https://www.packetcoders.io/scrapli-vs-netmiko/)
- [Ansible Network Automation](https://docs.ansible.com/ansible/latest/network/index.html)
- [Batfish GitHub](https://github.com/batfish/batfish)
- [Edgecore SONiC](https://www.edge-core.com/productsList.php?cls=509&cls2=510)
- [Dell OS10](https://www.dell.com/en-us/shop/ipovw/open-platform-software)
- [Celestica Networking](https://www.celestica.com/our-expertise/hardware-platform-solutions/networking)
- [NVIDIA Ethernet Switching](https://www.nvidia.com/en-us/networking/ethernet-switching/)
- [OpenConfig gNMI Specification](https://openconfig.net/docs/gnmi/gnmi-specification/)
- [gNMI/OpenConfig Telemetry Interoperability - Cisco](https://community.cisco.com/t5/crosswork-automation-hub-blogs/gnmi-openconfig-telemetry-interoperability-testing/ba-p/4891395)
- [INTA: Intent-Based Translation for Network Configuration with LLM Agents](https://arxiv.org/abs/2501.08760)
- [LLM-Based Network Management Survey](https://onlinelibrary.wiley.com/doi/abs/10.1002/nem.70029)
- [Ansible Abandoned Network Automation? - ipSpace.net](https://blog.ipspace.net/2025/12/ansible-abandoned-network-automation/)
- [OpenSwitch OPX Architecture](https://github.com/open-switch/opx-docs/wiki/Architecture)