# LOOM Edge Case Catalog

> Systematic analysis of race conditions, undefined behavior, boundary cases, and contract contradictions across all LOOM behavioral contracts.

---

## 1. Race Conditions

### RC-01: Concurrent Workflows on Same Device

- **Finding**: Two workflows attempt to configure the same device simultaneously (e.g., two provisioning workflows targeting the same switch, or a provisioning workflow and a teardown workflow on the same server).
- **Affected Contracts**: WORKFLOW-CONTRACT.md, ADAPTER-CONTRACT.md, VERIFICATION-MODEL.md
- **Scenario**:
  1. Workflow A starts provisioning device D, reaches step "Configure Network" and pushes VLAN 100.
  2. Workflow B starts teardown on device D, reaches step "Remove Network Config" and removes VLAN 100.
  3. Workflow A's verification activity runs and finds VLAN 100 is missing.
  4. Workflow A fails verification. Compensation begins, but there is nothing to compensate because VLAN 100 was already removed by Workflow B.
  5. Workflow B's verification may also produce confusing results if Workflow A's compensation re-creates the VLAN.
- **Current Behavior**: No contract defines device-level locking or mutual exclusion. `WorkflowRequest` has `TargetResources` but nothing prevents overlapping targets across concurrent workflows. The idempotency contract in ADAPTER-CONTRACT.md is per-operation (same `IdempotencyKey`), not per-device.
- **Recommended Fix**: Add a **device lock** mechanism to WORKFLOW-CONTRACT.md. Before a workflow's first mutating step, it must acquire an exclusive lease on each target device (via Temporal side effect or external lock service). If the lock is held, the workflow queues with a configurable wait timeout. Add `DeviceLockPolicy` to `WorkflowRequest`:
  ```go
  type DeviceLockPolicy struct {
      WaitTimeout time.Duration // how long to wait for lock
      Scope       string        // "exclusive" or "shared_read"
  }
  ```

---

### RC-02: Discovery During Decommission

- **Finding**: A discovery workflow scans a device that is simultaneously being decommissioned by a teardown workflow. Discovery enriches a device that teardown is about to mark as `decommissioned`.
- **Affected Contracts**: WORKFLOW-CONTRACT.md, IDENTITY-MODEL.md, DOMAIN-MODEL.md
- **Scenario**:
  1. Teardown workflow for device D reaches step "Deregister" and sets `status = decommissioned`.
  2. Concurrently, a discovery workflow scans the same subnet and finds device D still responding.
  3. Discovery's identity matcher finds existing device D, sees it is `decommissioned`, and must decide: enrich (bringing it back to life) or create a new device (duplicate).
  4. IDENTITY-MODEL.md's merge rules say "keep the older DeviceID" and enrich, but the device is decommissioned.
- **Current Behavior**: IDENTITY-MODEL.md does not account for device status during merge decisions. The merge procedure blindly unions endpoints and publishes `device.enriched`. DOMAIN-MODEL.md defines `DeviceStatusDecommissioned` but no contract says "do not merge into decommissioned devices."
- **Recommended Fix**: Add a status check to IDENTITY-MODEL.md Section 3 (Merge Rules): "If the matched device has `status = decommissioned`, do NOT merge. Instead, create a new device and flag for review with reason `rediscovered_decommissioned`. Publish `device.conflict_detected` with details." Add to Section 5 (Enrichment Rules): "Enrichment is only permitted for devices with status `discovered`, `active`, or `degraded`."

---

### RC-03: Credential Rotation During Active Workflow

- **Finding**: A credential is rotated (updated in the vault) while a long-running workflow is actively using it. The adapter holds a `CredentialRef` (vault path), and mid-workflow the secret at that path changes.
- **Affected Contracts**: ADAPTER-CONTRACT.md, WORKFLOW-CONTRACT.md, DOMAIN-MODEL.md
- **Scenario**:
  1. Workflow starts with step "Configure Network" using credential ref `vault:prod-switch-admin`.
  2. Adapter A connects using credential ref, resolves it to password "OldPass123", connects successfully.
  3. Security team rotates the credential: `vault:prod-switch-admin` now resolves to "NewPass456".
  4. Workflow proceeds to step "Install OS" which requires a new adapter connection.
  5. If the adapter cached the resolved credential from step 1, it may try "OldPass123" and fail with `ADAPTER_AUTH_FAILED` (permanent, no retry).
  6. If the adapter re-resolves the credential ref, it succeeds with "NewPass456", but now the connection in step 1 and step 4 used different credentials, complicating audit.
- **Current Behavior**: ADAPTER-CONTRACT.md says "Adapters never hold raw secrets -- they receive a ref and resolve it at connect time." This implies resolution happens per `Connect()` call, but it does not address whether a long-lived connection should re-resolve on reconnect. There is no guidance on credential versioning or pinning within a workflow execution.
- **Recommended Fix**: Add to ADAPTER-CONTRACT.md a **credential resolution policy**: "Adapters resolve `CredentialRef` at each `Connect()` call. If a workflow step fails with `ADAPTER_AUTH_FAILED` after a credential rotation, the workflow engine should retry the step once with a fresh credential resolution before treating it as permanent." Add a `CredentialVersion` field to audit records so the audit trail shows which version of a credential was used at each step.

---

### RC-04: Multi-Tenant Shared Physical Device

- **Finding**: Two tenants share a physical switch (e.g., tenant A owns VLANs 100-199, tenant B owns VLANs 200-299 on the same switch). Both tenants may issue concurrent workflows against the same physical device.
- **Affected Contracts**: DOMAIN-MODEL.md, WORKFLOW-CONTRACT.md, ADAPTER-CONTRACT.md
- **Scenario**:
  1. Tenant A submits a workflow to create VLAN 150 on switch S.
  2. Tenant B submits a workflow to create VLAN 250 on the same switch S.
  3. Both workflows run concurrently. At the adapter level, both target the same physical endpoint.
  4. The switch may serialize these commands, or one may fail with a resource conflict.
  5. Tenant B's audit records reference a device that "belongs to" tenant A (or vice versa), violating tenant isolation.
- **Current Behavior**: DOMAIN-MODEL.md says "Cross-tenant access is forbidden" and "every model carries `TenantID`." But a physical switch serving multiple tenants must exist in both tenants' views. There is no concept of a shared device or a device existing in multiple tenant scopes. The `Device` struct has a single `TenantID`.
- **Recommended Fix**: Add a **shared device model** to DOMAIN-MODEL.md. Options: (a) A device can have a `OwnerTenantID` and a list of `AuthorizedTenantIDs`, or (b) each tenant gets a virtual device record that maps to a shared physical device via a `PhysicalDeviceID` relationship. Option (b) is cleaner for tenant isolation. Add to WORKFLOW-CONTRACT.md: "When multiple tenants share a physical device, the workflow engine must acquire a physical-device-level lock (not tenant-device-level) to serialize conflicting operations."

---

### RC-05: Out-of-Order NATS Event Delivery

- **Finding**: NATS JetStream guarantees ordering per subject, but cross-subject ordering is not guaranteed. A `device.created` event on one subject and a `workflow.started` event on another subject may arrive at a consumer in the wrong order.
- **Affected Contracts**: EVENT-MODEL.md, AUDIT-MODEL.md
- **Scenario**:
  1. Discovery workflow creates device D and publishes `loom.tenant-a.device.created`.
  2. Immediately after, a provisioning workflow starts targeting device D and publishes `loom.tenant-a.workflow.started`.
  3. The DB Projector consumer (subscribed to `loom.>`) receives `workflow.started` before `device.created`.
  4. The projector tries to project a workflow referencing device D, but device D does not exist in the DB yet.
  5. The projector either crashes, drops the event, or creates an orphan record.
- **Current Behavior**: EVENT-MODEL.md states "Event ordering is guaranteed per subject. Cross-subject ordering is not guaranteed." The DB Projector subscribes to `loom.>` (all subjects). No guidance is given for handling out-of-order cross-subject events.
- **Recommended Fix**: Add to EVENT-MODEL.md a **consumer ordering strategy** section: "Consumers that subscribe to multiple subjects must handle out-of-order delivery. Strategies: (1) Idempotent upserts with foreign key deferral. (2) Retry-with-backoff for missing references (re-queue the event with a short delay). (3) Eventual consistency windows where projections may be temporarily inconsistent. The DB Projector must implement strategy (2): if a referenced resource does not exist, NAK the message and retry after 1 second, up to 5 times, then dead-letter."

---

### RC-06: Duplicate Device Discovery from Multiple Edge Agents

- **Finding**: The same device is discovered simultaneously by two different edge agents (e.g., one scanning via SNMP, another via Redfish), each reporting to the central LOOM instance.
- **Affected Contracts**: IDENTITY-MODEL.md, ADAPTER-CONTRACT.md
- **Scenario**:
  1. Edge agent 1 discovers device via SNMP: hostname `sw-01`, management IP `10.0.0.5`.
  2. Edge agent 2 discovers device via Redfish: serial `XYZ789`, BMC IP `10.0.0.105`.
  3. Both discovery results arrive at the identity matcher near-simultaneously.
  4. First result processes: no match found, creates new Device A.
  5. Second result processes: no match on serial (Device A has no serial yet from SNMP), no match on BMC IP, creates new Device B.
  6. Now the same physical device exists as two LOOM devices.
  7. Later, when SSH discovery runs and finds serial `XYZ789` on management IP `10.0.0.5`, the matcher finds Device A (by IP) and Device B (by serial) -- a split scenario that should never have occurred.
- **Current Behavior**: IDENTITY-MODEL.md's flow (Section 9) processes discovery results sequentially but does not account for race conditions between concurrent discovery submissions. The identity matcher has no mutual exclusion.
- **Recommended Fix**: Add to IDENTITY-MODEL.md: "The identity matcher must serialize processing of discovery results. Implementations must use a distributed lock keyed on the union of all candidate identity values being matched. This prevents two concurrent discoveries of the same device from creating duplicates. If serialization is impractical at scale, implement a periodic deduplication reconciliation job that merges devices sharing high-confidence identities."

---

### RC-07: Idempotency Cache Race on Concurrent Retries

- **Finding**: Two Temporal activity retries for the same operation arrive at the adapter nearly simultaneously. The idempotency cache may not yet reflect the first execution's result when the second arrives.
- **Affected Contracts**: ADAPTER-CONTRACT.md
- **Scenario**:
  1. Activity dispatches operation with `IdempotencyKey = "abc-123"`.
  2. Adapter begins execution, inserts "in progress" into idempotency cache.
  3. Temporal times out waiting for the activity heartbeat and schedules a retry.
  4. Retry arrives at the adapter. The idempotency contract says "if in progress, wait for completion and return the result."
  5. But: how long does the retry wait? What if the original execution is genuinely stuck? There is no timeout on the wait.
  6. Both the original execution and the retry could end up blocked or both could execute.
- **Current Behavior**: ADAPTER-CONTRACT.md rule 2 says "If the same IdempotencyKey is submitted and the operation is in progress: wait for completion and return the result." No timeout on the wait is specified. No guidance on what happens if the original execution never completes (e.g., adapter crashes mid-operation).
- **Recommended Fix**: Add to ADAPTER-CONTRACT.md: "The idempotency wait has a maximum duration equal to the operation's `Timeout` field. If the original execution does not complete within this window, the wait returns a `TransientError` with code `IDEMPOTENCY_TIMEOUT`, allowing the caller to retry with a new idempotency key if appropriate. Idempotency cache entries for in-progress operations must include a TTL equal to 2x the operation timeout, after which they are evicted and a new execution is permitted."

---

## 2. Undefined Behavior

### UB-01: Adapter Returns Success but Device State Unchanged

- **Finding**: An adapter's `Execute()` returns `StatusSucceeded` but the device did not actually change state (e.g., Redfish returned HTTP 200 for a power-on request, but the server's BMC accepted the command and silently dropped it).
- **Affected Contracts**: ADAPTER-CONTRACT.md, VERIFICATION-MODEL.md, WORKFLOW-CONTRACT.md
- **Scenario**:
  1. Workflow step "Power On" calls Redfish adapter.
  2. Redfish returns HTTP 200 (task accepted). Adapter returns `StatusSucceeded`.
  3. Verification activity runs `StateMatchCheck` for `power_state = "on"`.
  4. Verification observes `power_state = "off"`. Verification fails.
  5. Workflow fails. Compensation begins. But: the "reverse" of "power on" is "power off" -- the device is already off. Compensation is a no-op but is still attempted.
- **Current Behavior**: VERIFICATION-MODEL.md correctly handles this: "Failed verification = failed workflow. Even if every adapter call returned HTTP 200." But the compensation path is awkward -- compensating a no-op power-on with a power-off when the device is already off.
- **Recommended Fix**: Add to ADAPTER-CONTRACT.md: "Adapters should implement a **post-execution verification** within `Execute()` for protocols that accept commands asynchronously (Redfish task model, vSphere tasks). The adapter should poll the task status until completion or timeout, and only return `StatusSucceeded` when the task has actually completed on the device." Add to WORKFLOW-CONTRACT.md: "Compensation activities must check current state before executing. If the device is already in the desired compensation state, the compensation activity returns success without taking action (idempotent compensation)."

---

### UB-02: Silent Device Reversion After Verification

- **Finding**: Verification passes at time T, but the device silently reverts its configuration at T+30s (e.g., a switch that does not persist running config to startup config, and reboots due to a watchdog timer).
- **Affected Contracts**: VERIFICATION-MODEL.md, WORKFLOW-CONTRACT.md
- **Scenario**:
  1. Workflow pushes VLAN config to switch.
  2. Config is applied to running config but NOT saved to startup config.
  3. Verification at T+5s: VLAN exists. Verification passes. Workflow succeeds.
  4. Switch reboots at T+30s (power glitch, watchdog, scheduled reload).
  5. VLAN is gone. No LOOM component detects this.
- **Current Behavior**: VERIFICATION-MODEL.md explicitly says "Do not use verification as monitoring. Verification runs once per workflow, not continuously." This is correct design, but there is no contract covering post-verification drift detection.
- **Recommended Fix**: Add a new section to VERIFICATION-MODEL.md: "**Post-Verification Drift Detection** is outside the scope of the verification pipeline. Drift detection is handled by periodic discovery/reconciliation workflows and by Watcher adapters (for protocols that support streaming). Workflow authors implementing configuration changes MUST include a `config_save` step (e.g., `write memory`, `copy running startup`) as a separate activity after the configuration activity and before verification. The verification check should include a `config_persistence` check type that validates the startup config matches running config."

---

### UB-03: LLM Suggests Unknown Operation Type

- **Finding**: The LLM integration suggests an operation type that does not exist in the OPERATION-TYPES.md registry (e.g., `"firmware_upgrade"` or `"reboot_bmc"`).
- **Affected Contracts**: OPERATION-TYPES.md, ERROR-MODEL.md
- **Scenario**:
  1. User asks LOOM to "update the firmware on server rack-3."
  2. LLM interprets this and suggests operation type `firmware_upgrade`.
  3. `ValidateOperation()` looks up `firmware_upgrade` in `OperationTypes` registry.
  4. Not found. Returns `ValidationError` with code `UNKNOWN_OPERATION`.
  5. The user gets a cryptic error. There is no feedback loop to the LLM to suggest an alternative.
- **Current Behavior**: OPERATION-TYPES.md defines `ValidateOperation()` which returns `UNKNOWN_OPERATION` for unregistered types. This is correct for direct API calls. But the LLM integration path is not addressed -- there is no contract for what happens when the LLM generates an invalid operation.
- **Recommended Fix**: Add to OPERATION-TYPES.md: "When an LLM-suggested operation type is not found in the registry, the system must: (1) Return the list of valid operation types to the LLM for re-evaluation. (2) Log the invalid suggestion as an `llm.invalid_suggestion` audit record. (3) If the LLM's second attempt also fails, escalate to the human user with the error and the list of available operations. Never silently retry with the LLM more than twice." Add `firmware_upgrade` and `reboot_bmc` to the operation types registry if these are legitimate operations, or document them as explicitly out of scope.

---

### UB-04: Multi-Protocol Device Ownership Ambiguity

- **Finding**: A device supports Redfish for power control and SSH for configuration. Two adapters both declare capabilities for the same device. When the workflow engine needs to execute a power cycle, it is clear (Redfish). But when it needs to discover hardware, both SSH (`dmidecode`) and Redfish (`/redfish/v1/Systems`) can do it. Which adapter's result is authoritative?
- **Affected Contracts**: ADAPTER-CONTRACT.md, DOMAIN-MODEL.md, IDENTITY-MODEL.md
- **Scenario**:
  1. Redfish adapter discovers CPU count = 2 (from BMC data, confidence 0.9).
  2. SSH adapter discovers CPU count = 4 (from `lscpu`, counting hyperthreaded cores, confidence 0.8).
  3. Both are stored as `ObservedFact` with different sources and confidences.
  4. A workflow declares `ExpectedOutcome` with `cpu_count = 4`.
  5. Verification queries facts and finds two conflicting values. Which one does it use?
- **Current Behavior**: DOMAIN-MODEL.md's `ObservedFact` stores both values with their source and confidence. But there is no contract for how verification resolves conflicting facts from different adapters. VERIFICATION-MODEL.md's `StateMatchCheck` compares a single `Expected` value against a single `Observed` value -- no provision for multiple observed values.
- **Recommended Fix**: Add to VERIFICATION-MODEL.md: "When multiple ObservedFacts exist for the same property from different sources, verification uses the fact with the highest confidence. If confidences are equal, the most recent observation wins. If the highest-confidence fact does not match the expected value but a lower-confidence fact does, verification fails (do not cherry-pick favorable results)." Add to DOMAIN-MODEL.md: "A `FactResolutionPolicy` should be defined per fact type to specify how conflicting observations are resolved."

---

### UB-05: Compensation Action Failure (Double Fault)

- **Finding**: A saga compensation step itself fails. The workflow is already in a failure state (that triggered compensation), and now the cleanup is also failing.
- **Affected Contracts**: WORKFLOW-CONTRACT.md, ERROR-MODEL.md, AUDIT-MODEL.md
- **Scenario**:
  1. Provisioning workflow step 3 ("Configure Network") succeeds -- VLAN 100 created.
  2. Step 4 ("Install OS") fails.
  3. Saga compensation begins. Step 3 compensation: "Delete VLAN 100."
  4. The switch is now unreachable (network outage). Compensation fails with `ADAPTER_UNREACHABLE`.
  5. Compensation retries exhaust. VLAN 100 is orphaned on the switch.
  6. Resources from step 1 ("Allocate Resources") also need compensation, but IPs were allocated from an external IPAM. That compensation also fails because IPAM is behind the same network outage.
- **Current Behavior**: WORKFLOW-CONTRACT.md says "Compensation failures are logged and alerted but do not block other compensations." This is good -- it means step 1 compensation still runs even if step 3 compensation failed. ERROR-MODEL.md says "Compensation errors do not mask the original error." But there is no contract for: (a) how orphaned resources are tracked, (b) when/how failed compensations are retried, (c) what the final workflow state is when compensation partially succeeds.
- **Recommended Fix**: Add a **compensation failure tracking** section to WORKFLOW-CONTRACT.md: "When a compensation activity fails after all retries, the workflow records a `CompensationFailure` in its state, including the step name, the compensation operation that failed, and the resources left in an inconsistent state. The workflow's final state is `compensation_partial` (new state, add to the `State` enum). A separate `CompensationRecovery` workflow can be triggered (manually or by policy) to retry failed compensations when the underlying issue is resolved. Failed compensations emit `loom.{tenant}.workflow.compensation_failed` events." Add `compensation_partial` to the `WorkflowStatus.State` enum.

---

### UB-06: Adapter `Connect()` Succeeds but `Ping()` Fails Immediately

- **Finding**: An adapter's `Connect()` succeeds (TCP handshake completes, TLS negotiates) but `Ping()` fails immediately after (e.g., the device accepted the connection but the management service is crashing/restarting).
- **Affected Contracts**: ADAPTER-CONTRACT.md
- **Scenario**:
  1. Adapter calls `Connect()` -- success.
  2. Adapter calls `Ping()` -- fails.
  3. `Connected()` returns true (TCP connection is established).
  4. Activity calls `Execute()` -- fails because the management service is down.
  5. Is this a TransientError or PermanentError? The connection exists but the device is not functional.
- **Current Behavior**: ADAPTER-CONTRACT.md defines `Connector` with `Connect()`, `Disconnect()`, `Ping()`, and `Connected()` but does not specify the expected relationship between `Connect()` success and `Ping()` success. There is no guidance on whether `Connected()` should return `true` if `Ping()` fails.
- **Recommended Fix**: Add to ADAPTER-CONTRACT.md: "`Connected()` returns true only if the most recent `Ping()` succeeded. If `Ping()` fails after a successful `Connect()`, the adapter must call `Disconnect()` and return a `TransientError`. Activities should call `Ping()` before `Execute()` and treat `Ping()` failure as `ADAPTER_UNREACHABLE` (transient, retryable)."

---

### UB-07: DryRun with Side Effects

- **Finding**: A `TypedOperation` with `DryRun = true` is dispatched, but the adapter has a bug or the protocol does not support dry-run natively, resulting in actual state changes on the device.
- **Affected Contracts**: OPERATION-TYPES.md, ADAPTER-CONTRACT.md, AUDIT-MODEL.md
- **Scenario**:
  1. User submits workflow with `DryRun = true`.
  2. Workflow dispatches `CreateVLANOp` with `DryRun = true`.
  3. Adapter receives the operation. The underlying protocol (e.g., NX-API) does not have a native dry-run mode.
  4. Adapter mistakenly executes the command for real. VLAN is created.
  5. Operation returns `StatusDryRunOK`, but the VLAN now exists on the device.
  6. No compensation is triggered because it was supposed to be a dry run.
- **Current Behavior**: OPERATION-TYPES.md defines `DryRun bool` on `TypedOperation` and `StatusDryRunOK` as a result status. But there is no contract requiring adapters to actually implement dry-run semantics, nor is there a fallback for protocols that do not support it.
- **Recommended Fix**: Add to ADAPTER-CONTRACT.md: "Adapters that do not support native dry-run for a given operation must return a `PermanentError` with code `DRY_RUN_UNSUPPORTED` rather than executing the operation. The workflow engine should handle this by simulating the dry run at the workflow level (validating inputs and checking preconditions without dispatching to the adapter). Add `DRY_RUN_UNSUPPORTED` to the error code table in ERROR-MODEL.md."

---

## 3. Boundary Cases

### BC-01: Zero Devices in Inventory

- **Finding**: A new tenant with no devices registered. Various subsystems may have implicit assumptions about non-empty device sets.
- **Affected Contracts**: WORKFLOW-CONTRACT.md, DOMAIN-MODEL.md, EVENT-MODEL.md
- **Scenario**:
  1. New tenant created. Zero devices, zero endpoints, zero facts.
  2. User submits a discovery workflow targeting subnet `10.0.0.0/24`.
  3. Discovery finds zero responsive hosts.
  4. Workflow has zero steps to execute (no targets).
  5. What is the `ExpectedOutcome`? What does verification check? Does the workflow succeed or fail?
  6. The DB Projector has no events to project. Dashboard shows nothing.
- **Current Behavior**: WORKFLOW-CONTRACT.md requires `ExpectedOutcome` and `TargetResources` at submission. For a subnet discovery, `TargetResources` might be empty (we don't know what's there yet). The contract says "No expected outcome, no workflow execution," which would block discovery workflows that are exploratory by nature.
- **Recommended Fix**: Add to WORKFLOW-CONTRACT.md: "Discovery workflows are exempt from the `TargetResources` requirement. They may specify a `DiscoveryScope` (subnet, IP range) instead of explicit resource refs. `ExpectedOutcome` for discovery workflows may use a special check type `discovery_found_devices` with `minimum_count` (which may be 0 for exploratory scans). A discovery that finds zero devices is a valid success if `minimum_count = 0`."

---

### BC-02: One Million Devices -- Scale Limits

- **Finding**: At extreme scale (1M+ devices), several contract-defined patterns break down.
- **Affected Contracts**: EVENT-MODEL.md, AUDIT-MODEL.md, IDENTITY-MODEL.md, DOMAIN-MODEL.md
- **Scenario**:
  1. Tenant has 1,000,000 devices, each with ~5 endpoints = 5M endpoints.
  2. Daily discovery: 1M `device.enriched` events + 5M `ObservedFact` rows = 5M events per day.
  3. NATS JetStream: 5M events/day at ~1KB each = ~5GB/day. 90-day retention = ~450GB in the stream. Stream config says 50GB max. **Events will be discarded after ~10 days**, not 90.
  4. Identity matcher: for each discovery result, searches existing devices by each external identity. With 1M devices, this is a linear scan unless indexed. Even with indexes, matching 1M devices against 5M candidate identities per discovery cycle is expensive.
  5. Audit table: 1M devices * 5 endpoints * daily discovery = 5M audit records/day. At 90 days hot storage = 450M rows. Queries without proper partitioning will be very slow.
  6. `continue-as-new` at 10K events: a discovery workflow scanning 1M devices will hit this limit rapidly.
- **Current Behavior**: EVENT-MODEL.md configures `MaxBytes: 50GB` and `MaxAge: 90 days`. These are contradictory at scale -- at high event volume, `MaxBytes` will trigger `DiscardOld` well before 90 days. WORKFLOW-CONTRACT.md mandates `continue-as-new` at ~10K events but does not address how large-scale discovery workflows should be structured.
- **Recommended Fix**: Add to EVENT-MODEL.md: "The `MaxBytes` and `MaxAge` settings must be tuned per deployment. At scale, `MaxBytes` is the binding constraint. Operators must monitor the effective retention period and adjust `MaxBytes` or add storage. Add a `loom._system.stream.retention_warning` event emitted when effective retention drops below 30 days." Add to WORKFLOW-CONTRACT.md: "Large-scale discovery must be partitioned into sub-workflows (e.g., per-subnet or per-rack) that each target a manageable number of devices. The parent workflow uses child workflows and `continue-as-new` to stay within the 10K event limit. Maximum recommended batch size: 500 devices per child workflow."

---

### BC-03: Device with No Working Management Protocol

- **Finding**: A device is added as a Target, but no adapter can successfully connect to it. Every protocol probe fails.
- **Affected Contracts**: ADAPTER-CONTRACT.md, DOMAIN-MODEL.md, WORKFLOW-CONTRACT.md
- **Scenario**:
  1. Operator adds Target `10.0.0.50` with no protocol hint.
  2. Discovery workflow tries SSH, Redfish, SNMP, IPMI -- all fail.
  3. No Device is created (discovery produced nothing).
  4. Target remains in `targets` table with no associated device.
  5. User sees the target in the UI but cannot do anything with it.
  6. Next discovery cycle tries again, fails again. Repeated failures generate noise.
- **Current Behavior**: DOMAIN-MODEL.md shows `Target -> Device` as `1:0..1` cardinality -- a target may not produce a device. But there is no contract for managing failed targets, suppressing repeated probe attempts, or surfacing actionable information to the user.
- **Recommended Fix**: Add to DOMAIN-MODEL.md: Add a `TargetStatus` enum: `pending`, `probing`, `unreachable`, `discovered`. Failed targets should be marked `unreachable` with a `LastProbeAt` timestamp and a `ProbeFailureCount`. Add to WORKFLOW-CONTRACT.md: "Discovery workflows must implement exponential backoff for unreachable targets: skip targets where `ProbeFailureCount > 5` unless `LastProbeAt` was more than 24 hours ago. Emit a `loom.{tenant}.discovery.target_unreachable` event on first failure and on every 10th consecutive failure thereafter."

---

### BC-04: Workflow with 1000 Steps -- Temporal History Overflow

- **Finding**: A complex workflow with many steps (e.g., configuring 200 switch ports, each requiring a configure + verify activity pair) generates Temporal history that exceeds the recommended limit.
- **Affected Contracts**: WORKFLOW-CONTRACT.md
- **Scenario**:
  1. Workflow: configure 200 switch ports with ACLs.
  2. Each port: 1 configure activity + 1 verify activity = 2 activities.
  3. 200 ports * 2 activities = 400 activities.
  4. Each activity generates ~25 Temporal history events (scheduled, started, completed, etc.).
  5. Total: 400 * 25 = ~10,000 history events -- right at the `continue-as-new` threshold.
  6. Adding retries, heartbeats, or compensations pushes it well over.
  7. If `continue-as-new` is triggered mid-compensation, the saga state (which steps completed, which need compensation) must be correctly serialized and restored.
- **Current Behavior**: WORKFLOW-CONTRACT.md mandates `continue-as-new` at ~10K events but provides no guidance on checkpointing saga state across continuation boundaries.
- **Recommended Fix**: Add to WORKFLOW-CONTRACT.md: "Workflows that may exceed 5,000 history events must proactively batch work into child workflows, each handling a subset of targets. The parent workflow tracks completion of child workflows and handles compensation at the batch level. Saga state for `continue-as-new` must include: `completedSteps []StepCheckpoint`, `pendingCompensations []CompensationCheckpoint`. Add a `ContinueAsNewCheckpoint` struct:
  ```go
  type ContinueAsNewCheckpoint struct {
      CompletedSteps        []StepCheckpoint
      PendingCompensations  []CompensationCheckpoint
      CurrentBatchIndex     int
      TotalBatches          int
  }
  ```
  "

---

### BC-05: Event Burst -- 100K Events in 1 Second

- **Finding**: A mass discovery or bulk operation generates a burst of events that may exceed NATS and consumer capacity.
- **Affected Contracts**: EVENT-MODEL.md, AUDIT-MODEL.md
- **Scenario**:
  1. Large-scale discovery completes, publishing 100K `device.enriched` events.
  2. NATS receives 100K messages in ~1 second.
  3. DB Projector consumer has `MaxAckPending: 1000`. It can only process 1000 events at a time.
  4. NATS queues the remaining 99K events. Consumer falls behind.
  5. `AckWait: 30s` -- if the consumer cannot process 1000 events in 30 seconds, NATS redelivers.
  6. Redelivered events pile up. Consumer processes duplicates.
  7. Audit writer has the same problem -- 100K audit records to write, each requiring a DB insert.
  8. PostgreSQL write throughput may bottleneck at ~5K inserts/second without batching.
- **Current Behavior**: EVENT-MODEL.md configures consumer with `MaxAckPending: 1000` and `AckWait: 30s`. `MaxDeliver: 5` means each event is redelivered up to 5 times. But no guidance on batch processing, back-pressure, or what happens when the consumer cannot keep up.
- **Recommended Fix**: Add to EVENT-MODEL.md: "Consumers processing high-volume event streams must implement batch acknowledgment. The DB Projector should batch inserts (100-500 events per transaction) and ack each batch atomically. `MaxAckPending` should be tuned to the consumer's throughput capacity. Add a `loom._system.consumer.lag` metric and alert when consumer lag exceeds 60 seconds. Publishers must implement client-side rate limiting during bulk operations (max 10K events/second per publisher)."

---

### BC-06: Tenant with Zero Budget

- **Finding**: COST-MODEL.md does not exist, so cost/budget enforcement is referenced in ERROR-MODEL.md (`BUDGET_EXCEEDED` error code) but has no backing contract.
- **Affected Contracts**: ERROR-MODEL.md, WORKFLOW-CONTRACT.md
- **Scenario**:
  1. Tenant has a budget of $0 (or no budget configured).
  2. User submits a read-only discovery workflow.
  3. Does `BUDGET_EXCEEDED` apply to read-only operations?
  4. If budget checking is mandatory, a zero-budget tenant cannot perform any operations.
  5. If budget checking is optional, the `BUDGET_EXCEEDED` error code is dead code for tenants without budgets.
- **Current Behavior**: ERROR-MODEL.md defines `BUDGET_EXCEEDED` as a permanent error. WORKFLOW-CONTRACT.md does not reference budget checks. There is no COST-MODEL.md to define budget enforcement rules.
- **Recommended Fix**: Create COST-MODEL.md defining: "Budget enforcement is optional per tenant. If no budget is configured, cost checks are skipped. Read-only operations (where `OperationTypeInfo.ReadOnly == true`) are exempt from budget enforcement. When a budget exists, cost estimation occurs at workflow submission (before execution). The estimated cost is compared against remaining budget. If the estimate exceeds the remaining budget, return `BUDGET_EXCEEDED` at submission time, not mid-execution."

---

### BC-07: Idempotency Cache TTL vs Long-Running Operations

- **Finding**: The idempotency cache has a "configurable TTL" but no guidance on minimum TTL relative to operation timeout.
- **Affected Contracts**: ADAPTER-CONTRACT.md, WORKFLOW-CONTRACT.md
- **Scenario**:
  1. Operation: firmware upgrade. Timeout: 30 minutes (Long category).
  2. Idempotency cache TTL: 5 minutes (misconfigured).
  3. Operation starts at T=0. At T=5m, idempotency key is evicted.
  4. Temporal retries the activity at T=6m (heartbeat timeout). New idempotency key lookup: not found.
  5. Adapter starts a SECOND firmware upgrade on the device. Device is now running two concurrent firmware upgrades.
  6. Device is bricked.
- **Current Behavior**: ADAPTER-CONTRACT.md says "Adapters maintain an idempotency cache with a configurable TTL" but provides no minimum or guidance for setting the TTL relative to operation timeouts.
- **Recommended Fix**: Add to ADAPTER-CONTRACT.md: "Idempotency cache TTL must be at least 2x the maximum operation timeout for that adapter. For Long-category operations (30 min timeout), the idempotency cache TTL must be at least 60 minutes. The cache must survive adapter restarts (persistent storage or shared cache). If the cache is lost (crash, restart), the adapter must check device state before re-executing a potentially duplicate operation."

---

## 4. Contract Contradictions

### CC-01: Retry Policy vs Idempotency Guarantee

- **Finding**: ERROR-MODEL.md's `TransientRetryPolicy` specifies 5 attempts with exponential backoff. ADAPTER-CONTRACT.md's idempotency contract says "if the same IdempotencyKey is submitted and the operation already completed: return the previous result." These are compatible in theory, but the interaction creates a subtle issue: Temporal's retry policy and the adapter's retry policy may overlap.
- **Affected Contracts**: ERROR-MODEL.md, ADAPTER-CONTRACT.md, WORKFLOW-CONTRACT.md
- **Scenario**:
  1. Adapter call times out at the activity level (Temporal marks activity as failed).
  2. Temporal's retry policy (from ERROR-MODEL.md) schedules a retry with the same IdempotencyKey.
  3. Meanwhile, the adapter's own error handling also retries internally (TransientRetryPolicy: 5 attempts).
  4. The adapter is now retrying internally AND Temporal is retrying externally.
  5. With 5 Temporal retries * 5 internal retries = up to 25 actual attempts against the device.
  6. Idempotency key prevents duplicate execution, but 25 connection attempts to an already-strained device makes the problem worse.
- **Current Behavior**: ERROR-MODEL.md defines retry policies. ADAPTER-CONTRACT.md defines idempotency. WORKFLOW-CONTRACT.md says "Temporal applies retry policy based on error.Retryable()." There is no contract defining which layer owns retry responsibility.
- **Recommended Fix**: Add to ERROR-MODEL.md: "Retry responsibility is assigned to exactly ONE layer. Adapters do NOT retry internally -- they return errors immediately with classification. Temporal activities are the retry boundary. The `RetryPolicy` in ERROR-MODEL.md applies to Temporal activity retries, not to adapter-internal retries. Adapters must be fail-fast: attempt once, classify the error, return." Add to ADAPTER-CONTRACT.md: "Adapters must not implement internal retry loops. All retry logic is handled by the Temporal activity layer using the idempotency key."

---

### CC-02: Saga Compensation vs Mandatory Verification

- **Finding**: WORKFLOW-CONTRACT.md says compensation runs for all completed steps in reverse on failure. VERIFICATION-MODEL.md says every side-effecting activity has a mandatory verification activity. When compensation runs, should compensation activities also be verified?
- **Affected Contracts**: WORKFLOW-CONTRACT.md, VERIFICATION-MODEL.md
- **Scenario**:
  1. Provisioning workflow: steps 1-4 completed, step 5 fails.
  2. Compensation begins for steps 4, 3, 2, 1.
  3. Compensation step 4: "Remove network config" executes.
  4. Per VERIFICATION-MODEL.md rule 4: "Every side-effecting activity has a corresponding verification activity."
  5. Should compensation step 4 be verified (check that network config is actually removed)?
  6. If yes: verification of compensation doubles the compensation time and adds failure modes.
  7. If no: this violates the "verification is mandatory" rule.
  8. If compensation verification fails, do we compensate the compensation? (Infinite recursion.)
- **Current Behavior**: VERIFICATION-MODEL.md says "Every side-effecting activity has a corresponding verification activity. Verification is never optional, never skippable." WORKFLOW-CONTRACT.md says "Compensation is its own activity with its own timeout and retry policy" but does not mention verification of compensation.
- **Recommended Fix**: Add to VERIFICATION-MODEL.md: "Compensation activities are exempt from mandatory verification. Compensation is best-effort by design (see WORKFLOW-CONTRACT.md). Verifying compensation outcomes is recommended but not required. If a compensation verification is included, its failure should be logged and alerted but must NOT trigger recursive compensation. Add a `CompensationVerification` check type that is advisory (does not fail the compensation)." Update WORKFLOW-CONTRACT.md to explicitly state: "Compensation activities do not require verification activities. Compensation failures are logged, alerted, and tracked for manual resolution."

---

### CC-03: Identity Merge vs Audit Immutability

- **Finding**: IDENTITY-MODEL.md's merge procedure changes relationships and device state. AUDIT-MODEL.md requires before/after state snapshots for every mutation. When two devices are merged, the audit trail for the merged-away device becomes confusing.
- **Affected Contracts**: IDENTITY-MODEL.md, AUDIT-MODEL.md
- **Scenario**:
  1. Device A (DeviceID = `aaa`) was created via SNMP discovery. It has 50 audit records.
  2. Device B (DeviceID = `bbb`) was created via Redfish discovery. It has 20 audit records.
  3. Identity matcher determines A and B are the same physical device (serial numbers match).
  4. Merge: keep `aaa` (older), attach B's endpoints to A, union identities.
  5. But device B (`bbb`) had 20 audit records. Those records reference `resource_id = bbb`.
  6. Device B is effectively gone (not decommissioned -- it never existed as a separate device; it was a discovery artifact).
  7. Audit query for device `aaa` misses the 20 records from device `bbb`.
  8. We cannot UPDATE audit records to change `resource_id` from `bbb` to `aaa` because audit records are immutable.
- **Current Behavior**: AUDIT-MODEL.md rule 6: "Audit records are IMMUTABLE. No UPDATE or DELETE." IDENTITY-MODEL.md merge procedure: "Keep the older DeviceID." No contract addresses audit record linkage after a merge.
- **Recommended Fix**: Add to IDENTITY-MODEL.md: "When devices are merged, create a `device.merged` audit record on the surviving device with `metadata.merged_device_id = {absorbed_device_id}`. Create a `device.absorbed` audit record on the absorbed device with `metadata.surviving_device_id = {surviving_device_id}`. Add a `MergeHistory` table:
  ```sql
  CREATE TABLE merge_history (
      id UUID PRIMARY KEY,
      surviving_device_id UUID NOT NULL,
      absorbed_device_id UUID NOT NULL,
      merged_at TIMESTAMPTZ NOT NULL,
      tenant_id UUID NOT NULL
  );
  ```
  Audit queries for a device must also query `merge_history` and include audit records for any absorbed devices. This preserves immutability while maintaining queryability."

---

### CC-04: Event CausationID Semantics Conflict

- **Finding**: EVENT-MODEL.md says "For root events (triggered directly by user action), CausationID equals the event's own ID." AUDIT-MODEL.md says "For root actions (direct API call), CausationID is the record's own ID." But these are independent implementations -- the event's ID and the audit record's ID are different UUIDs for the same action. Cross-referencing between events and audit records by CausationID will fail.
- **Affected Contracts**: EVENT-MODEL.md, AUDIT-MODEL.md
- **Scenario**:
  1. User creates a device via API. This generates:
     - Audit record with `ID = audit-001`, `CausationID = audit-001`.
     - Event with `ID = event-001`, `CausationID = event-001`.
  2. A workflow triggered by this event creates audit records with `CausationID = audit-001` and events with `CausationID = event-001`.
  3. Querying "all events caused by the original action" returns events with `CausationID = event-001`.
  4. Querying "all audit records caused by the original action" returns records with `CausationID = audit-001`.
  5. There is no linkage between `event-001` and `audit-001` -- they describe the same action but have different IDs and independent causation chains.
- **Current Behavior**: Both contracts define CausationID independently. CorrelationID links them (both use the same CorrelationID from the API boundary), but CausationID chains are disconnected.
- **Recommended Fix**: Add to both EVENT-MODEL.md and AUDIT-MODEL.md: "Events and audit records share the same `CorrelationID` for request-level tracing. `CausationID` is scoped to its own system (event CausationIDs reference event IDs; audit CausationIDs reference audit record IDs). To cross-reference between events and audit records for the same action, use `CorrelationID` + `Timestamp` + `Action` as a composite key. Alternatively, add an optional `AuditRecordID` field to the Event envelope to directly link events to their corresponding audit records."

---

### CC-05: Error Wrapping Default Contradicts Idempotency Safety

- **Finding**: ERROR-MODEL.md says "If the adapter returns a raw `error`, the activity wraps it as a `TransientError` with code `ADAPTER_TIMEOUT` (conservative default)." This means any unclassified error is treated as retryable. But retrying an unclassified error with the same idempotency key may be dangerous if the operation actually succeeded but the error occurred in response parsing.
- **Affected Contracts**: ERROR-MODEL.md, ADAPTER-CONTRACT.md
- **Scenario**:
  1. Adapter executes `CreateVLANOp`. Device creates the VLAN.
  2. Response parsing fails (malformed JSON from device). Adapter returns raw `error`.
  3. Activity wraps it as `TransientError` (retryable).
  4. Temporal retries with the same IdempotencyKey.
  5. Idempotency cache says "completed, result was..." -- but the cached result may also have the parsing error.
  6. Or if the idempotency cache did not store the result (because the error happened before caching), the adapter might try to create the VLAN again. The device returns "VLAN already exists" -- which might be a PermanentError, or might be handled by idempotency.
- **Current Behavior**: The conservative default of wrapping unknown errors as transient assumes retry is safe. Idempotency is supposed to make retry safe. But the idempotency cache may not have captured the result if the error occurred during result processing.
- **Recommended Fix**: Add to ADAPTER-CONTRACT.md: "The idempotency cache must record the operation as 'completed' BEFORE returning the result to the caller. If result serialization fails after the device operation succeeded, the cache must still contain a 'completed' entry (even without a result), preventing re-execution. On retry with a cached 'completed-no-result' entry, the adapter should re-read the current state from the device and construct the result, rather than re-executing the operation." Add to ERROR-MODEL.md: "The conservative TransientError wrapping for raw errors should be changed to `ADAPTER_UNKNOWN` with `Retryable() = true` but a maximum of 2 retries (not 5), to limit blast radius of misclassified errors."

---

### CC-06: Approval Gate Timeout vs Workflow Timeout

- **Finding**: WORKFLOW-CONTRACT.md defines approval gates that block on a Temporal signal. The `ApprovalRequest` has an `ExpiresAt` field. But there is no contract defining the relationship between the approval timeout and the workflow's overall timeout, or what happens if the approval signal arrives after `ExpiresAt`.
- **Affected Contracts**: WORKFLOW-CONTRACT.md
- **Scenario**:
  1. Provisioning workflow reaches approval gate at T=0.
  2. `ApprovalRequest.ExpiresAt = T+24h`.
  3. Workflow is blocked on the signal.
  4. At T+23h, the approver sends an "approved" signal.
  5. Workflow resumes. But the device's state may have changed in 23 hours. Pre-checks from step 1 are stale.
  6. Alternatively: at T+25h, no signal received. What happens? The contract does not specify auto-reject behavior.
- **Current Behavior**: WORKFLOW-CONTRACT.md defines `ExpiresAt` on `ApprovalRequest` and says "Approval gates use Temporal signals. Never polling." But no timer is defined to auto-reject after `ExpiresAt`. No requirement to re-run pre-checks after a long approval wait.
- **Recommended Fix**: Add to WORKFLOW-CONTRACT.md: "Approval gates must implement a Temporal timer alongside the signal wait. If the timer fires before a signal is received, the workflow auto-rejects with reason `approval_timeout` and transitions to compensation or cancelled state. When an approval signal is received after a wait exceeding a configurable `StaleCheckThreshold` (default: 1 hour), the workflow must re-run its pre-check step before proceeding. If pre-checks fail after re-run, the workflow fails with `PRECONDITION_STALE`." Add `PRECONDITION_STALE` to ERROR-MODEL.md.

---

### CC-07: PartialError Retryability Contradiction

- **Finding**: ERROR-MODEL.md defines `PartialError.Retryable()` as returning `false`. It says "partial errors need compensation, not retry." But ADAPTER-CONTRACT.md defines `PartialError` with `Succeeded` and `Failed` lists. For the failed portion, retry might be appropriate (e.g., 3 of 5 devices configured, 2 timed out -- those 2 could be retried individually).
- **Affected Contracts**: ERROR-MODEL.md, ADAPTER-CONTRACT.md
- **Scenario**:
  1. Multi-target activity configures 5 devices. 3 succeed, 2 fail with `ADAPTER_TIMEOUT`.
  2. Activity returns `PartialError` listing 3 succeeded, 2 failed.
  3. `PartialError.Retryable() = false`. Temporal does not retry.
  4. Workflow enters compensation. Compensates the 3 that succeeded.
  5. But the 2 that failed with a transient error could have been retried individually.
  6. The user's intent (configure 5 devices) is entirely rolled back because 2 of 5 had a transient failure.
- **Current Behavior**: `PartialError` is always non-retryable. The only path is compensation.
- **Recommended Fix**: Add to ERROR-MODEL.md: "When a `PartialError` is returned from a multi-target activity, the workflow should inspect the individual failure types. If all failures are transient (`ADAPTER_TIMEOUT`, `ADAPTER_UNREACHABLE`), the workflow may schedule a retry activity targeting only the failed resources, using the same idempotency keys. If any failure is permanent, the workflow should proceed to compensation. Add a `FailureDetails` field to `PartialError`:
  ```go
  type FailureDetail struct {
      OperationID string
      ErrorCode   string
      Retryable   bool
  }
  ```
  This allows the workflow engine to make per-failure retry/compensate decisions."

---

## 5. Missing Contract Coverage

### MC-01: No Device Lock/Lease Mechanism

- **Finding**: No contract defines how to prevent concurrent mutations on the same device.
- **Affected Contracts**: All
- **Recommended Fix**: Create a CONCURRENCY-MODEL.md defining device leases, lock scopes (exclusive for writes, shared for reads), lease TTL, and integration with Temporal workflow state.

---

### MC-02: No Health/Liveness Contract for Adapters

- **Finding**: ADAPTER-CONTRACT.md defines `Ping()` but no contract for adapter health checks, readiness probes, or graceful shutdown. If an adapter process crashes, open connections are abandoned.
- **Affected Contracts**: ADAPTER-CONTRACT.md
- **Recommended Fix**: Add a `Health` interface to ADAPTER-CONTRACT.md:
  ```go
  type Health interface {
      Ready() bool
      Shutdown(ctx context.Context) error
  }
  ```
  Define that `Shutdown()` must disconnect all active connections and drain the idempotency cache to persistent storage.

---

### MC-03: No Cross-Tenant Audit for Shared Resources

- **Finding**: AUDIT-MODEL.md scopes all audit records to a single `TenantID`. If a shared physical device exists (see RC-04), audit records from different tenants' operations on the same physical device cannot be correlated without cross-tenant queries, which are forbidden.
- **Affected Contracts**: AUDIT-MODEL.md, DOMAIN-MODEL.md
- **Recommended Fix**: If shared devices are supported, add a `PhysicalResourceID` field to audit records that allows platform operators (not tenants) to query all operations on a physical device across tenants, using a privileged audit query role.

---

### MC-04: No DISCOVERY-CONTRACT.md

- **Finding**: Discovery is referenced in WORKFLOW-CONTRACT.md (discovery workflow pattern), IDENTITY-MODEL.md (discovery flow), and ADAPTER-CONTRACT.md (Discoverer interface), but there is no standalone discovery contract defining scan strategies, subnet sweep behavior, protocol auto-detection order, or discovery frequency management.
- **Affected Contracts**: WORKFLOW-CONTRACT.md, IDENTITY-MODEL.md, ADAPTER-CONTRACT.md
- **Recommended Fix**: Create DISCOVERY-CONTRACT.md covering: (1) Protocol probe order (Redfish first, then SSH, SNMP, IPMI). (2) Subnet scan concurrency limits. (3) Auto-detection heuristics. (4) Discovery frequency per target (avoid hammering devices). (5) Integration with IDENTITY-MODEL.md's merge/split rules.

---

### MC-05: No `RemoveACLOp` Definition

- **Finding**: OPERATION-TYPES.md references `remove_acl` as the compensation for `set_acl`, but `RemoveACLOp` is never defined with typed params and result structs. It is referenced in the registry (`CompensationOp: "remove_acl"`) but missing from the operation type definitions and the `OperationTypes` map.
- **Affected Contracts**: OPERATION-TYPES.md
- **Recommended Fix**: Add `RemoveACLOp` with typed params and result:
  ```go
  type RemoveACLParams struct {
      Name      string
      Interface string
      Direction string
  }
  type RemoveACLResult struct {
      Removed bool
  }
  ```
  Register it in the `OperationTypes` map.

---

### MC-06: No Timeout on Approval Gate Signals

- **Finding**: WORKFLOW-CONTRACT.md defines `ApprovalRequest.ExpiresAt` but never defines the Temporal timer that enforces it. Without a timer, the workflow blocks on the signal indefinitely.
- **Affected Contracts**: WORKFLOW-CONTRACT.md
- **Recommended Fix**: See CC-06 above. The approval gate pattern must include a `workflow.NewTimer(expiresAt)` alongside `workflow.GetSignalChannel("approval")`, with a `workflow.NewSelector()` that handles whichever fires first.

---

### MC-07: StateEvent Sequence Gaps Not Addressed

- **Finding**: ADAPTER-CONTRACT.md's `StateEvent` includes a `Sequence uint64` field described as "monotonically increasing per watch session." But no contract defines what consumers should do when sequence gaps are detected (indicating lost events).
- **Affected Contracts**: ADAPTER-CONTRACT.md, EVENT-MODEL.md
- **Recommended Fix**: Add to ADAPTER-CONTRACT.md: "Watcher consumers must track the last-seen sequence number. If a gap is detected (received sequence N+2 without N+1), the consumer must: (1) Log a warning with the gap range. (2) Request a full state re-read via `StateReader.ReadState()` to reconcile. (3) Reset the sequence tracking to the latest received value. Watcher implementations must NOT skip sequence numbers except in error conditions."

---

## Summary

| Category | Count | Critical | Needs New Contract |
|----------|-------|----------|-------------------|
| Race Conditions | 7 | RC-01, RC-06 | RC-04 (CONCURRENCY-MODEL.md) |
| Undefined Behavior | 7 | UB-01, UB-05 | UB-03 (LLM integration contract) |
| Boundary Cases | 7 | BC-02, BC-07 | BC-06 (COST-MODEL.md) |
| Contract Contradictions | 7 | CC-01, CC-02, CC-05 | -- |
| Missing Coverage | 7 | MC-01, MC-04, MC-05 | MC-01 (CONCURRENCY-MODEL.md), MC-04 (DISCOVERY-CONTRACT.md) |

**Total findings: 35**

### Priority Order for Resolution

1. **RC-01** (device locking) and **MC-01** (concurrency model) -- most likely to cause production data corruption.
2. **CC-01** (retry ownership) -- retry storms will degrade infrastructure under load.
3. **UB-05** (double fault) -- compensation failures with no recovery path leave infrastructure in unknown state.
4. **CC-02** (compensation verification) -- contradictory rules will confuse implementors.
5. **RC-06** (duplicate discovery) -- will create data quality issues at scale.
6. **MC-05** (missing RemoveACLOp) -- will cause a panic/crash at runtime when compensation is attempted.
7. **BC-02** (scale limits) -- NATS stream config will silently discard events at scale.
8. **CC-05** (error wrapping default) -- misclassified errors lead to unsafe retries.
9. **MC-04** (missing discovery contract) -- discovery behavior is scattered across 3 documents with no single source of truth.
10. Everything else.
