# Glossary

> Definitions of every LOOM-specific term used across the documentation. Alphabetically ordered. Each entry links to the document where the term is formally specified.

---

**Adapter**
A protocol-specific module that implements one or more operation families (Connector, Discoverer, Executor, StateReader, Watcher) to communicate with a target device. Adapters never hold raw secrets and are registered at startup via a registry.
See: [ADAPTER-CONTRACT.md](ADAPTER-CONTRACT.md)

**Capability**
A declared ability of an adapter, expressed as a concrete struct (`Name`, `Protocol`, `ReadOnly`). Capabilities are registered at startup and used by the orchestrator to select the right adapter for a given operation.
See: [ADAPTER-CONTRACT.md](ADAPTER-CONTRACT.md)

**Compensation**
The reverse operation that undoes a side-effecting action during saga rollback. Each compensatable operation declares its reverse operation type. Compensation is its own activity with its own timeout and retry policy.
See: [ADAPTER-CONTRACT.md](ADAPTER-CONTRACT.md), [WORKFLOW-CONTRACT.md](WORKFLOW-CONTRACT.md)

**Confidence Score**
A float value from 0.0 to 1.0 expressing certainty. Used in identity matching (how certain we are that two observations refer to the same device) and in LLM recommendations (how certain the model is in its suggestion). Below 0.7 triggers human review; below 0.4 uses deterministic fallback.
See: [IDENTITY-MODEL.md](IDENTITY-MODEL.md), [LLM-BOUNDARIES.md](LLM-BOUNDARIES.md)

**Convergence**
The state where observed reality matches desired state. Verification proves convergence by comparing expected outcomes against actual observations. A workflow has converged only when all verification checks pass.
See: [VERIFICATION-MODEL.md](VERIFICATION-MODEL.md)

**Cost Profile**
A structured description of the cost characteristics of a resource or substrate, including hardware amortization, power, rack space, cloud pricing, and network costs. Used to estimate and compare costs across substrates.
See: [COST-MODEL.md](COST-MODEL.md)

**Credential Ref**
A reference to credentials stored in the vault. Adapters never hold raw secrets -- they receive a `CredentialRef` (vault path + type) and resolve it at connect time. All credential access produces an audit record.
See: [ADAPTER-CONTRACT.md](ADAPTER-CONTRACT.md), [VAULT-ARCHITECTURE.md](VAULT-ARCHITECTURE.md)

**DEK (Data Encryption Key)**
A unique AES-256-GCM key generated for each individual credential. The DEK encrypts the credential plaintext. DEKs are never stored in plaintext -- they are wrapped by a KEK.
See: [VAULT-ARCHITECTURE.md](VAULT-ARCHITECTURE.md)

**Desired State**
The target configuration declared by a workflow's `ExpectedOutcome`. Desired state is what the system should look like after a workflow completes successfully. Verification compares observed state against desired state.
See: [VERIFICATION-MODEL.md](VERIFICATION-MODEL.md), [WORKFLOW-CONTRACT.md](WORKFLOW-CONTRACT.md)

**Device**
A discovered piece of infrastructure (server, switch, router, firewall, hypervisor, KVM, storage, appliance) identified by a stable `DeviceID` (UUID v7). A device has one or more endpoints and a set of external identities.
See: [DOMAIN-MODEL.md](DOMAIN-MODEL.md)

**DryRun**
A mode where a workflow is planned and validated but not executed. The orchestrator returns the execution plan, cost estimate, and expected outcomes without making any changes to target devices. Set via `DryRun: true` on `WorkflowRequest` or `TypedOperation`.
See: [WORKFLOW-CONTRACT.md](WORKFLOW-CONTRACT.md), [OPERATION-TYPES.md](OPERATION-TYPES.md)

**Endpoint**
A management interface on a device -- an addressable combination of IP/hostname, port, protocol, and transport. A single device may have multiple endpoints (e.g., Redfish on port 443, IPMI on port 623, SSH on port 22).
See: [DOMAIN-MODEL.md](DOMAIN-MODEL.md)

**Envelope Encryption**
The three-tier encryption scheme used by the credential vault: MEK wraps KEKs, KEKs wrap DEKs, DEKs wrap credential plaintext. Compromise of any single layer does not expose plaintext.
See: [VAULT-ARCHITECTURE.md](VAULT-ARCHITECTURE.md)

**External Identity**
An identifier observed on a device from the outside world -- serial number, MAC address, BMC IP, hostname, FQDN, asset tag, or BIOS UUID. External identities are append-only and carry a confidence score. They are the inputs to the identity matching algorithm.
See: [IDENTITY-MODEL.md](IDENTITY-MODEL.md)

**IdempotencyKey**
A UUID carried on every operation dispatched to an adapter. If the same key is submitted twice, the adapter returns the previous result without re-executing. Ensures safe retries on transient failures.
See: [ADAPTER-CONTRACT.md](ADAPTER-CONTRACT.md), [OPERATION-TYPES.md](OPERATION-TYPES.md)

**KEK (Key Encryption Key)**
A per-tenant key that wraps (encrypts) the DEKs for that tenant's credentials. KEK compromise exposes only one tenant's credentials. KEKs are themselves wrapped by the MEK.
See: [VAULT-ARCHITECTURE.md](VAULT-ARCHITECTURE.md)

**Leaf Node**
In the deployment topology, an edge agent running at a remote site with intermittent connectivity. Leaf nodes operate autonomously using local SQLite and sync state with the control plane when connectivity is available.
See: [DEPLOYMENT-TOPOLOGY.md](DEPLOYMENT-TOPOLOGY.md)

**MEK (Master Encryption Key)**
The root key in the encryption hierarchy, ideally backed by an HSM or TPM. The MEK wraps all tenant KEKs. Compromise of the MEK compromises the entire vault.
See: [VAULT-ARCHITECTURE.md](VAULT-ARCHITECTURE.md)

**Observed Fact**
A piece of information discovered about a device -- hardware model, firmware version, CPU count, interface count, etc. Facts carry a category, key, value, source adapter, and timestamp. Facts are append-only; new observations do not delete old ones.
See: [ADAPTER-CONTRACT.md](ADAPTER-CONTRACT.md), [DOMAIN-MODEL.md](DOMAIN-MODEL.md)

**Operation Family**
A category of adapter functionality: Connector (connection lifecycle), Discoverer (fact gathering), Executor (mutations), StateReader (point-in-time state), Watcher (streaming state). Each adapter implements only the families its protocol supports.
See: [ADAPTER-CONTRACT.md](ADAPTER-CONTRACT.md)

**Oversight Checking**
The process where the LLM (or deterministic rules) detects what a user forgot to specify in a request. For example, a request to create VMs without specifying firewall rules triggers an oversight check that flags the missing security configuration.
See: [LLM-BOUNDARIES.md](LLM-BOUNDARIES.md), [REQUEST-TO-DEVICE-FLOW.md](REQUEST-TO-DEVICE-FLOW.md)

**Probe**
A single attempt to reach a target using one protocol during discovery. Each probe has a timeout, records reachability, and captures any identities or facts available via that protocol.
See: [DISCOVERY-CONTRACT.md](DISCOVERY-CONTRACT.md)

**Saga**
A multi-step workflow pattern where each step has a corresponding compensation action. If step N fails, steps N-1 through 1 are compensated in reverse order. Implemented via Temporal workflows with explicit compensation activities.
See: [WORKFLOW-CONTRACT.md](WORKFLOW-CONTRACT.md)

**SecureBytes**
A memory-safe wrapper for credential plaintext. SecureBytes live on `mlock`'d pages that cannot be swapped to disk. Contents are zeroed immediately after use via constant-time wipe. SecureBytes refuse to serialize to JSON, appear in logs, or expose their contents via `String()`.
See: [VAULT-ARCHITECTURE.md](VAULT-ARCHITECTURE.md)

**State Snapshot**
A point-in-time reading of a resource's current state, produced by a `StateReader` adapter. Contains the resource reference, a map of state key-value pairs, a timestamp, and the source adapter.
See: [ADAPTER-CONTRACT.md](ADAPTER-CONTRACT.md)

**Tenant**
An isolation boundary in LOOM. Every resource, device, workflow, credential, event, and cost record belongs to exactly one tenant. Cross-tenant access is prevented at the type level, not just authorization checks. Multi-tenancy is enforced by `TenantID` on every query and every NATS subject.
See: [DOMAIN-MODEL.md](DOMAIN-MODEL.md), [SECURITY-MODEL.md](SECURITY-MODEL.md)

**Tombstoning**
The practice of marking a record as superseded rather than deleting it. External identities are tombstoned with a `SupersededAt` timestamp -- never removed. Audit records are immutable and never deleted. This preserves history for forensics and compliance.
See: [IDENTITY-MODEL.md](IDENTITY-MODEL.md), [AUDIT-MODEL.md](AUDIT-MODEL.md)

**Verification**
The mandatory process of comparing observed device state against expected outcomes after a workflow step executes. Verification is a separate Temporal activity with its own timeout, retry policy, and heartbeat. Failed verification means failed workflow, regardless of whether the execution step reported success.
See: [VERIFICATION-MODEL.md](VERIFICATION-MODEL.md)

**Workflow**
A multi-step orchestration plan executed by Temporal. Workflows declare expected outcomes at submission, execute steps as Temporal activities, verify each step, and compensate on failure. Temporal is the execution source of truth; PostgreSQL is a searchable projection.
See: [WORKFLOW-CONTRACT.md](WORKFLOW-CONTRACT.md)
