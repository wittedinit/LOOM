# ADR-002: Temporal for Workflow Orchestration

## Status

Accepted

## Context

LOOM orchestrates multi-step infrastructure operations: discover devices, classify them, generate configs, execute changes, verify state. These workflows must survive process restarts, handle partial failures with rollback, support human approval gates, and provide full audit history. A workflow engine is needed rather than ad-hoc state machines.

## Decision

Temporal is the workflow orchestration engine for all LOOM operations.

Key factors:

- **Durable execution with saga rollback**: Temporal workflows survive process crashes and restarts. If a config push fails midway, the saga pattern rolls back completed steps automatically. No manual compensation logic needed.
- **Go SDK is the original**: Temporal was built in Go (evolved from Uber's Cadence). The Go SDK is the most mature, best documented, and first to receive new features.
- **Signals for human approval**: Temporal signals allow workflows to pause and wait for external input (human approval, manual verification) without polling or external state. The workflow resumes exactly where it left off.
- **Event replay for restart survival**: Temporal replays event history to reconstruct workflow state after restarts. No checkpoint management, no state serialization bugs. The workflow code itself is the state machine.
- **Visibility and searchability**: Temporal's visibility features allow querying workflow state, filtering by tenant/device/status, and building operational dashboards without a separate reporting layer.

### Rejected Alternatives

- **Argo Workflows**: Requires Kubernetes. LOOM must run at edge sites that may not have K8s. Container-per-step model adds overhead for lightweight operations like SNMP polls.
- **Apache Airflow**: Batch-oriented DAG scheduler. Not designed for event-driven infrastructure operations with human-in-the-loop approval. Python-only task definitions.
- **Prefect**: Python-only. Same deployment concerns as Python in ADR-001. No Go SDK.
- **Custom workflow engine**: Estimated 12-18 months to build durable execution, replay, signals, and visibility from scratch. Temporal provides all of this immediately and is battle-tested at scale.

## Consequences

- All multi-step operations are Temporal workflows (discovery, provisioning, remediation, decommissioning)
- Temporal server is a required infrastructure dependency (runs alongside LOOM services)
- Workflow definitions serve as living documentation of operational processes
- Temporal is the source of truth for execution state; the database is a projection (see ADR-008)
- Team members need to understand Temporal's programming model (deterministic workflows, activities for side effects)
- Human approval workflows are built using Temporal signals rather than external ticketing integration
