# ADR-008: Temporal as Execution Truth, Database as Projection

## Status

Accepted

## Context

LOOM has two stateful systems: Temporal (workflow execution state) and PostgreSQL (queryable device/inventory state). When both systems claim to be the source of truth, dual-write problems emerge: a workflow completes but the DB update fails, or the DB is updated but the workflow crashes before recording completion. Resolving conflicts between two sources of truth is a well-known distributed systems problem with no clean solution.

## Decision

Temporal is the source of truth for execution state. The database is a read-optimized projection of that truth.

Architecture:

- **Temporal owns execution state**: Whether a workflow is running, completed, failed, or waiting for approval. What step it is on. What has been rolled back.
- **Database owns queryable projections**: Device inventory, topology, configuration history, telemetry. Optimized for API queries, dashboards, and reporting.
- **Reconciliation is one-directional**: Temporal -> Database. Activities within workflows update the database as a side effect of execution. If the database falls behind, it can be reconciled from Temporal's event history.
- **No database-to-Temporal writes**: The database never triggers or modifies Temporal workflows. API requests that start workflows go through Temporal directly.

Flow:
```
User Request → API → Temporal Workflow Started
                        ↓
              Activities Execute (device operations)
                        ↓
              Activities Update DB (projection)
                        ↓
              Workflow Completes in Temporal
                        ↓
              DB reflects final state
```

## Consequences

- No dual-write coordination problems — Temporal is always authoritative for execution
- Database inconsistency is temporary and self-healing (activities retry, reconciliation catches gaps)
- API reads come from the database (fast, indexed, SQL-queryable)
- Workflow status queries go to Temporal (authoritative, real-time)
- Reconciliation worker periodically verifies DB state matches Temporal completion records
- Trade-off: slight delay between workflow completion and DB projection update (eventually consistent, typically < 1 second)
- Disaster recovery is simpler: restore Temporal, replay to rebuild DB projections
