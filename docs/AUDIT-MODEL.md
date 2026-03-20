# Audit Model

> Audit trail for compliance, forensics, and operational traceability in LOOM.

## Core Principle

Every mutation in LOOM produces an audit record. Audit records are immutable. There is no UPDATE, no DELETE on the audit table. The audit trail is the single source of truth for "who did what, when, and why."

## Rules

1. **Every mutation produces an audit record.** No exception. If state changed, there is a record.
2. **Actor identity is always present.** Never empty, never null. If the actor cannot be determined, the operation must fail.
3. **System-initiated actions use a structured actor ID:** `system:{component}:{workflow_id}`. For example: `system:discovery-worker:wf-abc123`.
4. **Correlation ID links all audit records in one request chain.** A single API call that triggers a workflow that triggers five adapter calls produces six audit records, all sharing the same correlation ID.
5. **Causation ID links to the parent action.** The adapter call's audit record has a causation ID pointing to the workflow step's audit record, which points to the workflow's audit record, which points to the API call's audit record.
6. **Audit records are IMMUTABLE.** The audit table has no UPDATE or DELETE operations. The DB user for the audit writer should not have UPDATE or DELETE grants on the audit table.
7. **Before and after state are captured.** For updates, both the state before and after the change are recorded. For creates, `before_state` is nil. For deletes, `after_state` is nil.

## Audit Envelope

```go
package audit

import "time"

// AuditRecord is the immutable record of a single action in LOOM.
type AuditRecord struct {
    ID            string            `json:"id"`              // UUID v7
    Timestamp     time.Time         `json:"timestamp"`       // when the action occurred (UTC)
    TenantID      string            `json:"tenant_id"`       // tenant scope
    ActorType     string            `json:"actor_type"`      // "user", "system", "workflow"
    ActorID       string            `json:"actor_id"`        // JWT subject, system:{component}:{id}, workflow ID
    Action        string            `json:"action"`          // "device.created", "workflow.started", "power.cycled"
    ResourceType  string            `json:"resource_type"`   // "device", "endpoint", "vlan", "workflow"
    ResourceID    string            `json:"resource_id"`     // ID of the affected resource
    BeforeState   any               `json:"before_state"`    // state before the action (nil for creates)
    AfterState    any               `json:"after_state"`     // state after the action (nil for deletes)
    CorrelationID string            `json:"correlation_id"`  // links all records in one request chain
    CausationID   string            `json:"causation_id"`    // links to parent action
    Outcome       string            `json:"outcome"`         // "success", "failure", "partial"
    Metadata      map[string]string `json:"metadata"`        // additional context (IP address, user agent, etc.)
}
```

### Field Details

- **ID**: UUID v7. Time-sortable. Generated at record creation.
- **Timestamp**: UTC. When the action actually occurred, not when the record was written.
- **TenantID**: Every audit record is scoped to a tenant. System-internal actions use `_system`.
- **ActorType / ActorID**: Identifies who or what performed the action.
  - User: `ActorType="user"`, `ActorID="usr-abc123"` (from JWT `sub` claim)
  - System: `ActorType="system"`, `ActorID="system:discovery-worker:wf-abc123"`
  - Workflow: `ActorType="workflow"`, `ActorID="wf-abc123"`
- **Action**: Dot-notated action name. Format: `{domain}.{verb}`. Examples: `device.created`, `workflow.started`, `power.cycled`, `acl.pushed`, `approval.granted`.
- **ResourceType / ResourceID**: What was acted upon.
- **BeforeState / AfterState**: JSON snapshots of the resource. For creates, `BeforeState` is nil. For deletes, `AfterState` is nil. For updates, both are present. These are full snapshots, not diffs.
- **CorrelationID**: Assigned at the API boundary. Propagated through every downstream action.
- **CausationID**: For root actions (direct API call), this is the record's own ID. For derived actions, this is the ID of the audit record that triggered this one.
- **Outcome**: `"success"`, `"failure"`, or `"partial"`. Partial means some sub-operations succeeded and others failed.
- **Metadata**: Extra context. Common keys: `ip_address`, `user_agent`, `approval_reason`, `error_code`, `workflow_step`.

## What Gets Audited

### Always Audited (every occurrence)

| Category | Actions | Examples |
|----------|---------|----------|
| **Resource CRUD** | create, update, delete | Device created, VLAN modified, endpoint removed |
| **Workflow lifecycle** | start, step completion, completion, failure, cancellation | Workflow wf-123 started, step "configure_network" completed |
| **Approval decisions** | approve, reject, escalate | Approval granted by user-456 for workflow wf-123 |
| **Adapter calls** | call, success, failure | Redfish adapter called for power cycle on device-789 |
| **LLM decisions** | query, response, override | LLM recommended action X, confidence 0.87, user accepted |
| **Authentication** | login, logout, token refresh, auth failure | User usr-abc logged in from 10.0.1.5 |
| **Authorization** | permission grant, permission deny, role change | User usr-abc denied access to tenant-xyz resource |
| **Configuration changes** | setting change, policy update | Discovery interval changed from 1h to 30m |
| **Cost events** | budget set, threshold reached, overage | Budget for tenant-abc exceeded 80% threshold |
| **Compensation** | compensation started, compensation completed, compensation failed | Saga compensation: releasing IP 10.0.5.20 |

### Never Audited (excluded)

| Category | Reason |
|----------|--------|
| Health checks | Too frequent, no mutation, would overwhelm audit table |
| Read-only API queries | No state change. Use access logs for read tracking if needed. |
| Internal heartbeats | Temporal heartbeats are operational, not audit-worthy |
| Metrics collection | Prometheus scrapes are not mutations |
| Event bus internal messages | Events are their own record; auditing events creates infinite loops |

## Database Schema

```sql
CREATE TABLE audit_records (
    id              UUID PRIMARY KEY,
    timestamp       TIMESTAMPTZ NOT NULL,
    tenant_id       TEXT NOT NULL,
    actor_type      TEXT NOT NULL,
    actor_id        TEXT NOT NULL,
    action          TEXT NOT NULL,
    resource_type   TEXT NOT NULL,
    resource_id     TEXT NOT NULL,
    before_state    JSONB,
    after_state     JSONB,
    correlation_id  TEXT NOT NULL,
    causation_id    TEXT NOT NULL,
    outcome         TEXT NOT NULL CHECK (outcome IN ('success', 'failure', 'partial')),
    metadata        JSONB DEFAULT '{}'
);

-- Immutability: Revoke UPDATE and DELETE from the application user.
-- REVOKE UPDATE, DELETE ON audit_records FROM loom_app;

-- Indexes for common query patterns.
CREATE INDEX idx_audit_tenant_time ON audit_records (tenant_id, timestamp DESC);
CREATE INDEX idx_audit_correlation ON audit_records (correlation_id);
CREATE INDEX idx_audit_resource ON audit_records (resource_type, resource_id, timestamp DESC);
CREATE INDEX idx_audit_actor ON audit_records (actor_type, actor_id, timestamp DESC);
CREATE INDEX idx_audit_action ON audit_records (action, timestamp DESC);

-- Partition by month for retention management.
-- CREATE TABLE audit_records_2026_01 PARTITION OF audit_records
--     FOR VALUES FROM ('2026-01-01') TO ('2026-02-01');
```

## Audit Trail Queries (Common Patterns)

```sql
-- Everything that happened to a specific device
SELECT * FROM audit_records
WHERE resource_type = 'device' AND resource_id = 'dev-abc123'
ORDER BY timestamp DESC;

-- Full trace of a single request through the system
SELECT * FROM audit_records
WHERE correlation_id = 'corr-xyz789'
ORDER BY timestamp ASC;

-- All actions by a specific user in the last 24 hours
SELECT * FROM audit_records
WHERE actor_id = 'usr-abc123' AND timestamp > NOW() - INTERVAL '24 hours'
ORDER BY timestamp DESC;

-- All failed workflow steps in a tenant
SELECT * FROM audit_records
WHERE tenant_id = 'tenant-abc' AND action LIKE 'workflow.%' AND outcome = 'failure'
ORDER BY timestamp DESC;

-- All LLM decisions (for AI audit compliance)
SELECT * FROM audit_records
WHERE action LIKE 'llm.%'
ORDER BY timestamp DESC;
```

## Retention Policy

- **Hot storage (PostgreSQL):** 90 days. Partitioned by month. Old partitions detached and archived.
- **Warm storage (object storage):** 1 year. Parquet files exported from detached partitions. Queryable via tools like DuckDB or Athena.
- **Cold storage (compliance archive):** 7 years. Compressed, encrypted, immutable. For regulatory and legal requirements.
- Retention periods are per-tenant configurable (some tenants may require longer retention).

## Audit Writer Architecture

1. Audit records are created in-process by the component performing the action.
2. Records are published as events to the NATS audit stream (`loom.{tenant_id}.audit.created`).
3. The audit writer consumer reads from the stream and writes to PostgreSQL.
4. If the audit writer is down, records accumulate in NATS (retained for up to 90 days).
5. The audit writer is idempotent -- replaying the same event produces no duplicate records (upsert on ID).

## Anti-Patterns

- **Never skip audit for "internal" operations.** If it mutates state, it gets audited.
- **Never backfill audit records.** If you missed an audit record, that is a bug. Fix the code, do not fabricate history.
- **Never store secrets in audit records.** Before/after state snapshots must redact passwords, tokens, and keys.
- **Never use audit records as the primary data store.** Audit is a log, not a database. Query the main tables for operational data.
- **Never allow application code to UPDATE or DELETE audit records.** Enforce at the DB grant level.
