# Workflow Execution Contract

> LOOM workflow rules for Temporal integration.

## Foundational Principles

1. **Temporal is the execution source of truth.** PostgreSQL is a searchable projection and reporting layer. If Temporal and the DB disagree, Temporal wins.
2. **One workflow step = one meaningful business action.** A step is not a micro-activity. "Provision server" is a step. "Set hostname" is an activity within that step.
3. **One activity = one retryable side effect.** An activity wraps exactly one adapter call plus its verification. Nothing else.
4. **Verification is mandatory.** Every side-effecting activity has a corresponding verification activity. Verification is never optional, never skippable, never folded into the main activity.
5. **Compensation is explicit.** Where rollback is needed, compensation is its own activity with its own timeout and retry policy.
6. **Every workflow declares `ExpectedOutcome` at submission.** Verification activities use this to determine pass/fail. No expected outcome, no workflow execution.
7. **Approval gates use Temporal signals.** Never polling. A workflow blocks on a signal; the approval service sends the signal when a human or policy engine decides.
8. **Status projection to DB is async.** Workflow execution never blocks waiting for the DB projection to catch up.
9. **`continue-as-new` is mandatory** when workflow history approaches ~10,000 events. Workflows must checkpoint state and continue in a new execution.
10. **Heartbeat is required** for any activity running longer than 60 seconds.

## Activity Timeout Defaults

| Category | Timeout | Examples |
|----------|---------|----------|
| Short | 30s | Read operations, lookups, DNS queries |
| Medium | 5min | Provisioning calls, VLAN creation, ACL push |
| Long | 30min | OS install, firmware upgrade, full discovery |

Activities may override defaults with justification. Heartbeat interval should be no more than 1/3 of the activity timeout.

## Core Types

```go
package workflow

import "time"

// WorkflowRequest is submitted by a user or system to start a workflow.
type WorkflowRequest struct {
    ID              string            `json:"id"`               // UUID v7, assigned at submission
    TenantID        string            `json:"tenant_id"`
    WorkflowType    string            `json:"workflow_type"`    // "discovery", "provisioning", "power_cycle", "teardown"
    TargetResources []ResourceRef     `json:"target_resources"` // what this workflow operates on
    Parameters      map[string]any    `json:"parameters"`       // workflow-type-specific inputs
    ExpectedOutcome ExpectedOutcome   `json:"expected_outcome"` // what success looks like
    RequestedBy     ActorRef          `json:"requested_by"`     // who submitted this
    Priority        int               `json:"priority"`         // 0 = normal, higher = more urgent
    DryRun          bool              `json:"dry_run"`          // plan but do not execute
    CorrelationID   string            `json:"correlation_id"`
    SubmittedAt     time.Time         `json:"submitted_at"`
}

// ResourceRef identifies a target resource for a workflow.
type ResourceRef struct {
    Type string `json:"type"` // "device", "endpoint", "vlan", "subnet"
    ID   string `json:"id"`
}

// ActorRef identifies who or what initiated an action.
type ActorRef struct {
    Type string `json:"type"` // "user", "system", "workflow", "adapter"
    ID   string `json:"id"`
}

// WorkflowStatus is the projected state of a workflow in the reporting DB.
type WorkflowStatus struct {
    WorkflowID      string          `json:"workflow_id"`
    RunID           string          `json:"run_id"`          // Temporal run ID
    TenantID        string          `json:"tenant_id"`
    WorkflowType    string          `json:"workflow_type"`
    State           string          `json:"state"`           // "pending", "running", "awaiting_approval", "completed", "failed", "compensating", "cancelled"
    CurrentStep     string          `json:"current_step"`
    Steps           []StepStatus    `json:"steps"`
    ExpectedOutcome ExpectedOutcome `json:"expected_outcome"`
    ActualOutcome   *ActualOutcome  `json:"actual_outcome"`  // nil until workflow completes
    StartedAt       *time.Time      `json:"started_at"`
    CompletedAt     *time.Time      `json:"completed_at"`
    Error           *string         `json:"error"`
    CorrelationID   string          `json:"correlation_id"`
}

// StepStatus tracks an individual step within a workflow.
type StepStatus struct {
    Name        string     `json:"name"`
    State       string     `json:"state"`        // "pending", "running", "completed", "failed", "skipped", "compensated"
    StartedAt   *time.Time `json:"started_at"`
    CompletedAt *time.Time `json:"completed_at"`
    Error       *string    `json:"error"`
}

// ExpectedOutcome declares what success looks like for a workflow.
// Verification activities compare observed state against these expectations.
type ExpectedOutcome struct {
    Description string                  `json:"description"`      // human-readable summary
    Checks      []ExpectedCheck         `json:"checks"`           // each check that must pass
}

// ExpectedCheck is a single verifiable condition.
type ExpectedCheck struct {
    CheckType    string `json:"check_type"`    // "reachability", "state_match", "port_open", etc.
    ResourceRef  ResourceRef `json:"resource_ref"`
    Property     string `json:"property"`      // what property to check
    ExpectedValue any   `json:"expected_value"` // what it should be
}

// ActualOutcome is the result of verification after workflow execution.
type ActualOutcome struct {
    Converged bool                `json:"converged"` // true only if ALL checks passed
    Results   []VerificationResult `json:"results"`
}

// VerificationResult is the outcome of a single verification check.
type VerificationResult struct {
    CheckName  string    `json:"check_name"`
    Expected   any       `json:"expected"`
    Observed   any       `json:"observed"`
    Passed     bool      `json:"passed"`
    Evidence   string    `json:"evidence"`    // raw adapter output as proof
    CheckedAt  time.Time `json:"checked_at"`
}

// ActivityResult is the standard return type for every Temporal activity.
type ActivityResult struct {
    Success       bool              `json:"success"`
    ResourceRef   ResourceRef       `json:"resource_ref"`
    Action        string            `json:"action"`         // what was done
    Evidence      string            `json:"evidence"`       // raw output proving it was done
    Error         *string           `json:"error"`
    Retryable     bool              `json:"retryable"`
    Metadata      map[string]string `json:"metadata"`
}

// ApprovalRequest is sent when a workflow reaches an approval gate.
type ApprovalRequest struct {
    WorkflowID    string            `json:"workflow_id"`
    TenantID      string            `json:"tenant_id"`
    StepName      string            `json:"step_name"`
    Description   string            `json:"description"`    // what is being approved
    Risk          string            `json:"risk"`           // "low", "medium", "high", "critical"
    RequestedBy   ActorRef          `json:"requested_by"`
    ApproverRoles []string          `json:"approver_roles"` // roles allowed to approve
    Context       map[string]any    `json:"context"`        // data the approver needs to decide
    ExpiresAt     time.Time         `json:"expires_at"`     // auto-reject after this time
    CorrelationID string            `json:"correlation_id"`
}

// ApprovalDecision is the signal sent back to the workflow.
type ApprovalDecision struct {
    Approved    bool      `json:"approved"`
    DecidedBy   ActorRef  `json:"decided_by"`
    Reason      string    `json:"reason"`
    DecidedAt   time.Time `json:"decided_at"`
}
```

## Workflow Patterns

### 1. Discovery Workflow

**Purpose:** Scan infrastructure, collect facts, reconcile with known state.

**Steps:**
1. **Scan** -- Adapter calls to discover devices/endpoints (activity: `ScanSubnet`, `ScanSwitchPorts`, etc.)
2. **Verify reachability** -- Confirm discovered devices respond (activity: `VerifyReachability`)
3. **Enrich** -- Collect detailed facts from each device (activity: `CollectFacts`)
4. **Reconcile** -- Compare discovered facts against DB, emit `conflict_detected` events for mismatches (activity: `ReconcileFacts`)
5. **Verify convergence** -- Confirm DB now reflects discovered state (activity: `VerifyDiscoveryOutcome`)

**Compensation:** None. Discovery is read-only with no side effects to undo.

### 2. Provisioning Workflow (Saga Pattern)

**Purpose:** Bring a device or service from unprovisioned to fully operational.

**Steps (with compensation):**
1. **Allocate resources** -- Reserve IPs, VLANs, ports (activity: `AllocateResources` / compensate: `ReleaseResources`)
2. **Approval gate** -- Signal-based wait for human or policy approval
3. **Configure network** -- Push VLAN, ACL, routing config (activity: `ConfigureNetwork` / compensate: `RemoveNetworkConfig`)
4. **Install OS** -- Trigger PXE/iPXE boot, OS deployment (activity: `InstallOS` / compensate: `WipeOS`)
5. **Configure host** -- Post-install config: users, packages, agents (activity: `ConfigureHost` / compensate: `RevertHostConfig`)
6. **Verify provisioning** -- Full verification against `ExpectedOutcome` (activity: `VerifyProvisioningOutcome`)
7. **Register in CMDB** -- Update DB and emit events (activity: `RegisterAsset` / compensate: `DeregisterAsset`)

**Saga semantics:** On failure at any step, run compensation for all previously completed steps in reverse order. Compensation failures are logged and alerted but do not block other compensations.

### 3. Power-Cycle Workflow

**Purpose:** Power off, wait, power on a device.

**Steps:**
1. **Pre-check** -- Verify device is reachable and current power state (activity: `CheckPowerState`)
2. **Power off** -- IPMI/Redfish/iLO power off command (activity: `PowerOff`)
3. **Verify off** -- Confirm device is unreachable / power state is off (activity: `VerifyPowerOff`)
4. **Wait** -- Configurable delay (Temporal timer, not an activity)
5. **Power on** -- Power on command (activity: `PowerOn`)
6. **Verify on** -- Confirm device is reachable and booted (activity: `VerifyPowerOn`)

**Compensation:** If power-on fails, leave device off and alert. Do not retry indefinitely.

### 4. Teardown Workflow

**Purpose:** Decommission a device or service. Reverse of provisioning.

**Steps:**
1. **Pre-check** -- Verify no active workloads depend on this resource (activity: `CheckDependencies`)
2. **Approval gate** -- Signal-based wait (teardown always requires approval)
3. **Drain workloads** -- Migrate or stop workloads (activity: `DrainWorkloads`)
4. **Remove host config** -- Undo host-level configuration (activity: `RemoveHostConfig`)
5. **Remove network config** -- Remove VLANs, ACLs, routes (activity: `RemoveNetworkConfig`)
6. **Release resources** -- Free IPs, ports, VLANs (activity: `ReleaseResources`)
7. **Verify teardown** -- Confirm device is fully decommissioned (activity: `VerifyTeardownOutcome`)
8. **Deregister** -- Mark as decommissioned in DB and emit events (activity: `DeregisterAsset`)

**Compensation:** Teardown compensation re-registers and re-configures. This is the most complex compensation path and must be tested thoroughly.

## Rules for Workflow Authors

- Never call an adapter directly from workflow code. Always go through an activity.
- Never skip verification. If you cannot verify, the workflow cannot succeed.
- Never use `time.Sleep` in workflow code. Use Temporal timers.
- Never store secrets in workflow input or state. Reference them by vault path.
- Always include `CorrelationID` in every activity call for tracing.
- Always set appropriate timeouts per the table above.
- Always implement `continue-as-new` for long-running or looping workflows.
- Always emit events at workflow start, step completion, and workflow completion.
