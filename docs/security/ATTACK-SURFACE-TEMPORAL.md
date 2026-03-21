# Temporal Attack Surface Analysis for LOOM

> Security assessment of Temporal as used by LOOM -- a multi-tenant infrastructure orchestrator.
> All exploit and defense code is in `/tmp/loom-security-temporal/`.

**Assessment Date:** 2026-03-21
**Scope:** Temporal gRPC API, workflow signals, activity execution, namespace isolation, resource exhaustion, compensation integrity, credential lifecycle
**Risk Context:** LOOM holds credentials for every managed device (BMC/IPMI, network switches, hypervisors, cloud APIs). A Temporal compromise grants the attacker workflow execution authority over all managed infrastructure.

---

## Summary of Findings

| # | Attack Vector | Severity | Default Temporal Config | With Recommended Defenses |
|---|--------------|----------|------------------------|--------------------------|
| 1 | Signal Injection (Approval Bypass) | **Critical** | Vulnerable | Mitigated |
| 2 | Activity Result Poisoning | **Critical** | Vulnerable | Mitigated |
| 3 | Workflow History Info Disclosure | **High** | Vulnerable | Mitigated |
| 4 | Namespace Escape (Cross-Tenant) | **Critical** | Vulnerable | Mitigated |
| 5 | Workflow Bombing (DoS) | **High** | Partially Vulnerable | Mitigated |
| 6 | gRPC API Exploitation | **Critical** | Vulnerable | Mitigated |
| 7 | Compensation Manipulation | **High** | Vulnerable | Mitigated |
| 8 | Stale Credential in Long Workflows | **High** | Vulnerable | Mitigated |

**Bottom line:** Temporal's default configuration provides **no authentication or authorization**. Out of the box, any process that can reach port 7233 has full control over all workflows in all namespaces. Every attack in this document succeeds against a default Temporal deployment. All defenses require explicit configuration and application-level code.

---

## Attack 1: Workflow Tampering via Signal Injection

**Files:** `attack1-signal-injection/exploit.go`, `attack1-signal-injection/defense.go`

### Threat Model

LOOM's provisioning workflows use Temporal signals for human approval gates (see `WORKFLOW-CONTRACT.md`, principle 7). A workflow blocks on `workflow.GetSignalChannel("approval-decision")` until an approver sends an `ApprovalDecision` signal.

**The problem:** Temporal does not authenticate signal senders. Any client connected to the namespace can send a signal to any workflow.

### Attack Description

1. The attacker connects to Temporal without authentication (no mTLS configured).
2. The attacker targets a different tenant's namespace (e.g., `tenant-acme`).
3. Using `ListWorkflow`, the attacker enumerates running provisioning workflows.
4. The attacker crafts an `ApprovalDecision{Approved: true}` with a spoofed `DecidedBy` identity.
5. The attacker sends the signal via `client.SignalWorkflow()`.
6. The workflow unblocks and proceeds with provisioning -- without legitimate approval.

### Impact

- **Approval bypass:** Infrastructure changes execute without human oversight.
- **Cross-tenant attack:** Attacker in `tenant-evil` approves workflows in `tenant-acme`.
- **Impersonation:** The `DecidedBy` field is self-asserted; the attacker impersonates any user.

### Defense: Signed Signal Envelope

The defense wraps every signal in a `SignedSignal` envelope with:

1. **Ed25519 signature** over the payload, proving the signal came from LOOM's approval service.
2. **Tenant ID validation** -- signal's tenant must match the workflow's tenant.
3. **Nonce** -- prevents replay attacks. Stored in a distributed cache with TTL.
4. **Expiration** -- signals older than a configurable maximum age are rejected.
5. **Signer identity verification** -- only approval service instances with registered public keys are accepted.
6. **Authorization check** -- the approver's role is validated against the workflow's `ApproverRoles`.

The workflow loops on signal receipt: invalid signals are silently discarded (logged and audited) and the workflow continues waiting for a valid signal. The attacker receives no feedback about why their signal was rejected.

### Key Code Pattern

```go
// VULNERABLE (default):
var decision ApprovalDecision
signalCh.Receive(ctx, &decision) // Trusts ANY signal sender

// DEFENDED:
var signal SignedSignal
signalCh.Receive(ctx, &signal)
decision, err := validator.Validate(signal, workflow.Now(ctx))
if err != nil {
    // Log, audit, continue waiting
    continue
}
```

---

## Attack 2: Activity Result Poisoning

**Files:** `attack2-result-poisoning/exploit.go`, `attack2-result-poisoning/defense.go`

### Threat Model

Temporal dispatches activities to workers via task queues. Workers register for a task queue name and Temporal sends activities to any registered worker. There is no built-in worker authentication.

### Attack Description

1. A rogue process connects to Temporal and calls `worker.New(c, "loom-provisioning", ...)`.
2. The rogue worker registers malicious implementations for LOOM's activities: `VerifyProvisioningOutcome`, `ConfigureNetwork`, `InstallOS`.
3. Temporal dispatches some activities to the rogue worker (round-robin with legitimate workers).
4. The rogue worker returns fabricated results:
   - `VerifyProvisioningOutcome` returns `{Converged: true}` with fake evidence strings.
   - `ConfigureNetwork` returns `{Success: true}` without touching the device.
   - `InstallOS` returns `{Success: true}` without installing anything.
5. The provisioning workflow trusts these results and marks the device as operational.

### Impact

- **Verification bypass:** A device is marked as fully provisioned when it is not.
- **Security control gaps:** ACLs, firewall rules, and host hardening may not be applied.
- **Silent failure:** The workflow succeeds, so no alerts fire. The device is in an unknown state.

### Defense: mTLS + Activity Result Signing

**Layer 1: mTLS** -- Only workers with client certificates signed by LOOM's internal CA can connect to Temporal. A rogue process without a valid certificate cannot register as a worker.

**Layer 2: Activity Result Signing** -- Each activity result is wrapped in a `SignedActivityResult` with:
- Worker identity and Ed25519 signature.
- Activity ID and workflow ID for scope binding.
- The workflow verifies the signature against a registry of trusted worker public keys.

Even if an attacker obtains a valid client certificate (compromised worker), the result signature prevents cross-worker impersonation because each worker has a unique signing key.

---

## Attack 3: Workflow History Information Disclosure

**Files:** `attack3-history-disclosure/exploit.go`, `attack3-history-disclosure/defense.go`

### Threat Model

Temporal stores the complete event history for every workflow execution: all activity inputs, outputs, signals, and timer data. This history is queryable via the SDK client and persists for the namespace's retention period.

### Attack Description

1. The attacker connects to a tenant's namespace.
2. Using `ListWorkflow`, the attacker enumerates all workflow executions (running, completed, failed).
3. For each workflow, the attacker reads the full event history via `GetWorkflowHistory`.
4. The attacker extracts:
   - **Device IPs and BMC addresses** from activity inputs.
   - **VLAN configurations** from network provisioning activities.
   - **Operation parameters** revealing infrastructure topology.
   - **Credential material** if improperly placed in workflow inputs (e.g., passwords passed as activity arguments).

### What Could Leak

| Data Type | Where It Appears | Risk |
|-----------|-----------------|------|
| Device IPs | `WorkflowExecutionStarted` input | Infrastructure mapping |
| BMC/IPMI addresses | Activity inputs for power operations | Direct device access |
| VLAN/subnet config | `ConfigureNetwork` activity input/output | Network topology |
| SSH passwords | Activity inputs (if developer makes this mistake) | Full device compromise |
| API tokens | Activity inputs (if developer makes this mistake) | Cloud account access |
| Approval decisions | Signal data | Social engineering intel |

### Defense: Credential Indirection + Input Sanitization

1. **CredentialRef pattern:** Credentials NEVER appear in workflow inputs/outputs. Only `CredentialRef{ID: "uuid", Type: "ssh_password"}` appears. The credential store resolves UUIDs to plaintext at the adapter boundary, inside the activity execution, and the plaintext is never returned as an activity result.

2. **Input validation:** `SanitizeWorkflowInput()` scans all workflow and activity inputs for forbidden field names (`password`, `secret`, `api_key`, `token`, `private_key`, `ssh_key`, `credential`, `auth_token`, `passphrase`). If found, the operation is rejected before the workflow starts.

3. **History retention:** Namespace retention set to 7 days. Sensitive workflow types (credential rotation, security operations) use 24-hour retention.

4. **Custom DataConverter:** A `SanitizingDataConverter` wraps Temporal's default converter and regex-redacts any sensitive patterns that slip through validation.

### Key Architectural Rule

> Credentials are resolved at the adapter boundary. The flow is:
> `Workflow` -> `Activity(CredentialRef)` -> `CredentialService.Retrieve(uuid)` -> `Adapter.Execute(plaintext)` -> zero plaintext -> `Activity returns result (no credential)` -> `Workflow receives result`
>
> The plaintext credential exists only in the adapter's memory for the duration of the device call.

---

## Attack 4: Namespace Escape (Cross-Tenant Access)

**Files:** `attack4-namespace-escape/exploit.go`, `attack4-namespace-escape/defense.go`

### Threat Model

LOOM uses namespace-per-tenant isolation. Each tenant's workflows, activities, and history live in a separate Temporal namespace (e.g., `tenant-acme`, `tenant-bank`).

### Attack Description

1. The attacker has legitimate access to `tenant-evil`.
2. Without authorization controls, the attacker creates a new Temporal client targeting `tenant-acme`.
3. Temporal accepts the connection because **there is no authorization by default**.
4. The attacker can:
   - List all workflows in `tenant-acme`
   - Read workflow histories (combines with Attack 3)
   - Send signals to workflows (combines with Attack 1)
   - Cancel or terminate workflows
   - Register rogue workers (combines with Attack 2)
5. The attacker repeats this for every namespace, including `temporal-system` (Temporal's internal namespace).

### Temporal's Namespace Isolation Guarantees

**Without authorization plugins:** Namespaces provide **logical isolation only**. Any authenticated client (or any client at all without mTLS) can access any namespace. Namespaces are NOT a security boundary in default configuration.

**With Authorizer plugin:** Temporal supports a `ClaimMapper` + `Authorizer` plugin architecture that can enforce access control per namespace.

### Defense: ClaimMapper + Authorizer

1. **LoomClaimMapper:** Extracts tenant identity from the mTLS client certificate's `Subject.Organization` field. Maps it to Temporal claims scoped to `tenant-{org}` namespace.

2. **LoomAuthorizer:** Enforces that each request's claims match the target namespace. Tenant `acme` can only access namespace `tenant-acme`. Admin operations require the `loom-admin` role.

3. **Certificate structure:**
   - `Subject.Organization` = tenant ID
   - `Subject.OrganizationalUnit` = roles (`admin`, `writer`, `reader`)
   - `Subject.CommonName` = service identity (e.g., `worker.acme.loom.internal`)

4. **Namespace-specific worker SANs:** Each namespace's configuration includes `AllowedWorkerCertSANs` -- only workers with matching Subject Alternative Names can connect.

---

## Attack 5: Denial of Service via Workflow Bombing

**Files:** `attack5-workflow-bombing/exploit.go`, `attack5-workflow-bombing/defense.go`

### Threat Model

An attacker (or compromised tenant) creates workflows at high volume to exhaust Temporal's resources and block legitimate workflows.

### Attack Vectors

| Vector | Mechanism | Impact |
|--------|-----------|--------|
| **Workflow flood** | Create 10,000+ workflows in rapid succession | Exhaust persistence storage, overwhelm frontend |
| **Task queue saturation** | Fill task queues faster than workers can drain | Legitimate activities queue behind attacker's |
| **History bomb** | Send 100,000+ signals to a single workflow | Inflate event history, exhaust storage per-workflow |
| **Large payload** | Use 1KB+ payloads per workflow | Amplify storage impact |

### Defense: Multi-Layer Rate Limiting

**Layer 1: Application-level rate limiter (LOOM API)**
- `MaxWorkflowsPerMinute: 10` per tenant
- `MaxActiveWorkflows: 100` per tenant
- `MaxSignalsPerMinute: 50` per tenant per workflow
- All limits enforced in the `SecureWorkflowStarter` wrapper before reaching Temporal

**Layer 2: Temporal server-side rate limiting**
- `frontend.namespaceRPS: 200` -- per-namespace request rate limit
- `frontend.rps: 2400` -- global frontend rate limit
- `history.rps: 3000` / `matching.rps: 1200` -- internal service limits

**Layer 3: Workflow execution guards**
- `MaxWorkflowExecutionTimeout` capped at 24 hours
- `continue-as-new` forced at 10,000 events (per LOOM's workflow contract)
- Default timeout applied if none specified

**Layer 4: Monitoring**
- `QueueDepthMonitor` checks task queue depths every 30 seconds
- Alerts on zero pollers (no workers = complete DoS)
- Prometheus metrics on queue depth, workflow creation rate, rejection rate

---

## Attack 6: Temporal Server API Exploitation

**Files:** `attack6-api-exploitation/exploit.go`, `attack6-api-exploitation/defense.go`

### Threat Model

Temporal exposes a gRPC API with two primary services:
1. **WorkflowService** (port 7233): Workflow CRUD, namespace management, visibility queries
2. **OperatorService** (port 7233): Admin operations, search attributes, cluster management

### Attack Surface (Default Configuration)

| API | Operation | Auth Required (Default) | Impact |
|-----|-----------|------------------------|--------|
| `ListNamespaces` | Enumerate all tenants | **None** | Tenant discovery |
| `GetClusterInfo` | Temporal version, persistence config | **None** | Recon for targeted exploits |
| `RegisterNamespace` | Create new namespace | **None** | Attacker gets own execution environment |
| `ListWorkflowExecutions` | Enumerate workflows in any namespace | **None** | Target identification |
| `TerminateWorkflowExecution` | Force-kill any workflow | **None** | Mass disruption |
| `SignalWorkflowExecution` | Send signals to any workflow | **None** | Approval bypass (Attack 1) |
| `ListSearchAttributes` | Reveal indexed metadata schema | **None** | Intel on what LOOM indexes |

### Key Finding

> **Without mTLS and Authorizer, Temporal's default security posture is equivalent to an unauthenticated database with no access control.** Any process on the network can perform any operation, including creating namespaces, terminating workflows, and reading complete execution histories.

### Defense: mTLS + Network Segmentation

**mTLS Configuration** (in `temporal-server.yaml`):
- TLS 1.3 minimum for all connections
- `requireClientAuth: true` on frontend -- no client certificate = connection refused
- Separate certificates for internode communication
- All certificates signed by LOOM's internal CA

**Network Segmentation:**
- Temporal frontend (7233) accessible ONLY from `loom-api` and `loom-workers` pods
- Temporal internode traffic restricted to `temporal-*` pods
- Default deny all other traffic
- Implemented via Kubernetes NetworkPolicy or cloud security groups

**Admin API Restriction:**
- `RegisterNamespace`, `DeleteNamespace`, `UpdateNamespace` restricted to `loom-admin` role
- `GetClusterInfo` restricted (reveals version information useful for exploit targeting)
- All admin API calls logged to immutable audit trail

---

## Attack 7: Compensation/Rollback Manipulation

**Files:** `attack7-compensation-manipulation/exploit.go`, `attack7-compensation-manipulation/defense.go`

### Threat Model

LOOM uses the saga pattern for provisioning workflows. When a step fails, compensations execute in reverse order to restore consistent state. An attacker interrupts this process to leave infrastructure half-configured.

### Attack Sequence

1. Provisioning workflow fails at step 5 (`ConfigureHost`).
2. Workflow begins compensating:
   - `RevertHostConfig` (runs)
   - `WipeOS` (runs)
   - `RemoveNetworkConfig` (attacker cancels HERE)
   - `ReleaseResources` (NEVER RUNS)
3. Result: Network configuration removed but IPs, VLANs, and ports remain allocated. Resource leak.

**Worse scenario:** Attacker cancels between `WipeOS` and `RemoveNetworkConfig`:
- OS is wiped but network config remains
- VLAN and ACL still point to a device that no longer has an OS
- Potential for the VLAN/port to be reallocated to another device, inheriting the old ACL rules

### Defense: Disconnected Context + Compensation Tracking

**Critical Temporal API:** `workflow.NewDisconnectedContext(ctx)` creates a context that is NOT cancelled when the parent workflow is cancelled. Compensation activities run in this disconnected context.

```go
func (t *CompensationTracker) Compensate(ctx workflow.Context) error {
    disconnectedCtx, cancel := workflow.NewDisconnectedContext(ctx)
    defer cancel()
    // All compensation activities use disconnectedCtx
    // CancelWorkflow does NOT stop them
}
```

**Additional defenses:**
1. **Compensation plan persistence:** Before starting compensations, the plan is persisted to durable storage. If the workflow crashes entirely, a background reconciler picks up orphaned plans.
2. **Progress tracking:** After each compensation step, progress is persisted. On recovery, only remaining steps execute.
3. **Aggressive retry:** Compensation activities use `MaximumAttempts: 10` with exponential backoff. Compensations try very hard to succeed.
4. **Failure alerting:** If any compensation step exhausts retries, an alert fires for manual intervention.
5. **Non-short-circuit:** A failed compensation does NOT abort remaining compensations. All steps are attempted.

---

## Attack 8: Stale Credential in Long-Running Workflows

**Files:** `attack8-stale-credential/exploit.go`, `attack8-stale-credential/defense.go`

### Threat Model

LOOM workflows can run for hours (approval gates, OS installations). If a credential is rotated or revoked during execution, the workflow may continue using the old credential reference.

### Attack Scenario

1. Attacker compromises the SSH password for `switch-core-01`.
2. Security team detects the compromise and rotates the credential: UUID-123 (compromised) is revoked, UUID-456 (new) is stored.
3. A provisioning workflow started 2 hours ago still holds `credential_ref = UUID-123`.
4. When the workflow reaches `ConfigureNetwork`, it resolves UUID-123.

**Three outcomes depending on credential store behavior:**

| Scenario | Store Behavior | Impact |
|----------|---------------|--------|
| A (worst) | Soft delete: old credential still retrievable | Workflow uses compromised credential. Attacker's access window extends hours. |
| B | Hard delete: old credential gone | Workflow fails, enters compensation. Service disruption. |
| C (subtle) | Adapter caches credential in memory | Works even after rotation. No audit trail of stale usage. |

### Defense: Pre-Activity Validation + Rotation Signals

1. **Pre-activity credential check:** Before every activity that uses credentials, a `ValidateCredential` activity runs. This is a fast DB query (no decryption) that checks:
   - Does the credential still exist?
   - Is the version the workflow knows still current?
   - Is it expired or revoked?

2. **Credential rotation signal:** When LOOM's credential service rotates a credential, it queries Temporal's visibility API for all running workflows referencing that credential, then sends a `credential-rotated` signal to each one. The workflow updates its local `CredentialRef` to the new version.

3. **No adapter-side caching:** The `CredentialResolver` has no cache. Every activity execution resolves the credential ref fresh through the credential service. This eliminates scenario C.

4. **Version tracking:** `CredentialRef` includes a monotonic `Version` field. The `ValidateCredential` check compares the workflow's version against the current version. A mismatch means the credential was rotated and the workflow must re-resolve.

---

## Consolidated Defense Requirements

### Must-Have (Day 1)

| Defense | Attacks Mitigated | Effort |
|---------|------------------|--------|
| **mTLS on all Temporal connections** | 1, 2, 4, 6 | Medium (cert infrastructure) |
| **Authorizer + ClaimMapper plugin** | 1, 4, 6 | Medium (custom Temporal plugin) |
| **CredentialRef pattern (no plaintext in workflows)** | 3, 8 | Low (design discipline) |
| **Disconnected context for compensations** | 7 | Low (one API call) |
| **Pre-activity credential validation** | 8 | Low (one extra activity per step) |
| **Network segmentation** | 6 | Medium (infrastructure) |

### Should-Have (Day 30)

| Defense | Attacks Mitigated | Effort |
|---------|------------------|--------|
| **Signal signing (Ed25519 envelope)** | 1 | Medium (key management) |
| **Activity result signing** | 2 | Medium (key management) |
| **Per-tenant rate limiting** | 5 | Medium (application code) |
| **Credential rotation signals** | 8 | Medium (event plumbing) |

### Nice-to-Have (Day 90)

| Defense | Attacks Mitigated | Effort |
|---------|------------------|--------|
| **Custom DataConverter with redaction** | 3 | Low |
| **Queue depth monitoring** | 5 | Low |
| **Compensation plan persistence** | 7 | Medium |
| **Input sanitization (forbidden field scanner)** | 3 | Low |
| **History retention tuning** | 3 | Low (configuration) |

---

## Temporal-Specific Configuration Checklist

```yaml
# temporal-server.yaml security hardening for LOOM

tls:
  internode:
    server:
      requireClientAuth: true          # Internode mTLS
    rootCaFiles: [/etc/loom/certs/ca-cert.pem]
  frontend:
    server:
      requireClientAuth: true          # Client mTLS
    rootCaFiles: [/etc/loom/certs/ca-cert.pem]

authorization:
  authorizer: loom                     # Custom authorizer
  claimMapper: loom                    # Custom claim mapper

# Dynamic configuration (per-namespace rate limits)
frontend.namespaceRPS: 200             # Per-namespace RPS cap
frontend.rps: 2400                     # Global frontend RPS
history.persistenceMaxQPS: 9000
matching.rps: 1200
```

---

## References

- LOOM Workflow Contract: `docs/WORKFLOW-CONTRACT.md`
- LOOM Security Model: `docs/SECURITY-MODEL.md`
- Temporal Security Documentation: https://docs.temporal.io/security
- Temporal mTLS Configuration: https://docs.temporal.io/references/configuration#tls
- Temporal Authorization: https://docs.temporal.io/security#authorization
- Exploit/Defense Code: `/tmp/loom-security-temporal/`
