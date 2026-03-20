# ADR-010: Verification Is Mandatory

## Status

Accepted

## Context

Infrastructure automation systems commonly treat "execution succeeded" as equivalent to "the desired state was achieved." This is wrong. A CLI command can return exit code 0 while the config was not applied. A REST API can return 200 while the device queued the change for later. A firmware update can report success while the device is still running the old version. Without explicit verification, automation creates a false sense of convergence that erodes trust over time.

## Decision

Verification is a mandatory step in every LOOM workflow that modifies device state. A workflow that executes successfully but fails verification is a failed workflow.

Rules:

- **Execution success is not verification**: Receiving a 200 OK or exit code 0 from a device is recorded but does not constitute verification.
- **Verification reads back actual state**: After execution, a separate StateReader operation queries the device to confirm the desired state matches the observed state.
- **Failed verification = failed workflow**: If observed state does not match desired state after execution, the workflow enters a failed state with a verification mismatch report, even if execution reported success.
- **Verification has its own timeout and retry**: Verification may need to wait for device convergence (e.g., BGP session establishment after config push). Configurable per operation type.
- **Verification is auditable**: Both desired state and observed state are recorded in the workflow history, creating an auditable convergence record.

Verification flow:
```
Execute Change → Record Execution Result → Wait for Convergence Window → Read Actual State → Compare Desired vs Actual → Pass/Fail
```

## Consequences

- Higher confidence in automation outcomes — "verified" means the device is actually in the desired state
- Catches silent failures that would otherwise go undetected until an incident
- Workflows take slightly longer (verification adds a read-back step with convergence window)
- Adapter implementations must support StateReader for any operation that has an Executor (you cannot change what you cannot verify)
- Verification mismatches generate alerts and can trigger remediation workflows
- Audit records include both "what we asked for" and "what we observed," providing compliance evidence
- Team members cannot skip verification — it is not an optional workflow step
