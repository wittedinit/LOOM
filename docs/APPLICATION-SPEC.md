# APPLICATION-SPEC.md

> Definitive application-layer specifications closing gaps 3-12 and 14-27 from [APPLICATION-LAYER-GAP-ANALYSIS.md](APPLICATION-LAYER-GAP-ANALYSIS.md).
> Every open question has a decision. Every decision has a specification. Gaps 1-2 (Adapter SDK, Plugin Lifecycle) are handled by the adapter-sdk agent.

---

## Table of Contents

1. [API Route Table (Gap 3)](#1-api-route-table-gap-3)
2. [Observability Self-Debugging (Gap 4)](#2-observability-self-debugging-gap-4)
3. [LLM Integration Practicalities (Gap 5)](#3-llm-integration-practicalities-gap-5)
4. [Multi-Tenancy Application Layer (Gap 6)](#4-multi-tenancy-application-layer-gap-6)
5. [Testing Strategy for 25+ Adapters (Gap 7)](#5-testing-strategy-for-25-adapters-gap-7)
6. [CQRS Projection Rebuild (Gap 8)](#6-cqrs-projection-rebuild-gap-8)
7. [UI Real-Time & Scale (Gap 9)](#7-ui-real-time--scale-gap-9)
8. [CI/CD & Upgrade Testing (Gap 10)](#8-cicd--upgrade-testing-gap-10)
9. [LLM Autonomy Framework (Gap 11)](#9-llm-autonomy-framework-gap-11)
10. [Documentation & Onboarding (Gap 12)](#10-documentation--onboarding-gap-12)
11. [Remaining Gaps (14-27)](#11-remaining-gaps-14-27)

---

## 1. API Route Table (Gap 3)

### Decision

The full REST API surface is defined here for MVP (Phases 1-5). This is the source of truth until code is written and the OpenAPI spec is generated from huma handler registrations. Any deviation from this table during implementation requires an ADR.

### Route Inventory

All routes are prefixed with `/api/v1`. All mutating endpoints require `Authorization: Bearer <JWT>` and `Idempotency-Key: <uuid>` headers. All responses include `X-Correlation-ID` and `X-Request-ID` headers.

#### Targets

| Method | Path | Description | Auth | Rate Limit Tier |
|--------|------|-------------|------|-----------------|
| `POST` | `/targets` | Create a discovery target | `operator` | write |
| `POST` | `/targets/batch` | Create up to 500 targets in one call | `operator` | bulk |
| `GET` | `/targets` | List targets (cursor-paginated) | `viewer` | read |
| `GET` | `/targets/{id}` | Get a single target | `viewer` | read |
| `PATCH` | `/targets/{id}` | Update target (protocol_hint, credentials) | `operator` | write |
| `DELETE` | `/targets/{id}` | Delete a target | `operator` | write |
| `DELETE` | `/targets/batch` | Batch delete targets | `operator` | bulk |

#### Devices

| Method | Path | Description | Auth | Rate Limit Tier |
|--------|------|-------------|------|-----------------|
| `GET` | `/devices` | List devices (cursor-paginated, filterable) | `viewer` | read |
| `GET` | `/devices/{id}` | Get a single device | `viewer` | read |
| `PATCH` | `/devices/{id}` | Update device metadata (name, tags, metadata) | `operator` | write |
| `DELETE` | `/devices/{id}` | Decommission a device | `admin` | write |
| `GET` | `/devices/{id}/endpoints` | List endpoints for a device | `viewer` | read |
| `GET` | `/devices/{id}/facts` | List observed facts for a device | `viewer` | read |
| `GET` | `/devices/{id}/relationships` | List relationships involving this device | `viewer` | read |
| `GET` | `/devices/{id}/history` | Event history for a device (cursor-paginated) | `viewer` | read |
| `POST` | `/devices/{id}/actions/power` | Power on/off/cycle a device | `operator` | write |
| `POST` | `/devices/{id}/actions/refresh` | Trigger immediate state refresh | `operator` | write |

#### Endpoints

| Method | Path | Description | Auth | Rate Limit Tier |
|--------|------|-------------|------|-----------------|
| `GET` | `/endpoints` | List all endpoints (cursor-paginated) | `viewer` | read |
| `GET` | `/endpoints/{id}` | Get a single endpoint | `viewer` | read |
| `PATCH` | `/endpoints/{id}` | Update endpoint (credential_ref_id, status) | `operator` | write |

#### Credentials

| Method | Path | Description | Auth | Rate Limit Tier |
|--------|------|-------------|------|-----------------|
| `POST` | `/credentials` | Store a new credential (envelope-encrypted) | `admin` | write |
| `GET` | `/credentials` | List credential refs (metadata only, never secrets) | `operator` | read |
| `GET` | `/credentials/{id}` | Get credential ref metadata | `operator` | read |
| `PATCH` | `/credentials/{id}` | Update credential (rotate secret material) | `admin` | write |
| `DELETE` | `/credentials/{id}` | Delete a credential ref | `admin` | write |
| `POST` | `/credentials/{id}/test` | Test credential against its endpoints | `operator` | write |

#### Discovery

| Method | Path | Description | Auth | Rate Limit Tier |
|--------|------|-------------|------|-----------------|
| `POST` | `/discovery/scans` | Submit a discovery scan (returns workflow ID) | `operator` | workflow |
| `GET` | `/discovery/scans` | List discovery scans (cursor-paginated) | `viewer` | read |
| `GET` | `/discovery/scans/{id}` | Get scan status and results | `viewer` | read |
| `POST` | `/discovery/scans/{id}/cancel` | Cancel a running scan | `operator` | write |

#### Workflows

| Method | Path | Description | Auth | Rate Limit Tier |
|--------|------|-------------|------|-----------------|
| `POST` | `/workflows` | Submit a workflow request | `operator` | workflow |
| `GET` | `/workflows` | List workflows (cursor-paginated, filterable) | `viewer` | read |
| `GET` | `/workflows/{id}` | Get workflow status, steps, and outcome | `viewer` | read |
| `POST` | `/workflows/{id}/approve` | Approve a workflow awaiting approval | `approver` | write |
| `POST` | `/workflows/{id}/reject` | Reject a workflow awaiting approval | `approver` | write |
| `POST` | `/workflows/{id}/cancel` | Cancel a running workflow | `operator` | write |
| `GET` | `/workflows/{id}/steps` | List step-level detail for a workflow | `viewer` | read |
| `GET` | `/workflows/{id}/events` | Event timeline for this workflow | `viewer` | read |

#### Cost

| Method | Path | Description | Auth | Rate Limit Tier |
|--------|------|-------------|------|-----------------|
| `GET` | `/cost/profiles` | List cost profiles for the tenant | `viewer` | read |
| `POST` | `/cost/profiles` | Create a cost profile | `admin` | write |
| `GET` | `/cost/profiles/{id}` | Get a cost profile | `viewer` | read |
| `PATCH` | `/cost/profiles/{id}` | Update a cost profile | `admin` | write |
| `DELETE` | `/cost/profiles/{id}` | Delete a cost profile | `admin` | write |
| `POST` | `/cost/profiles/import` | Bulk import cost profiles (CSV or JSON) | `admin` | bulk |
| `POST` | `/cost/estimate` | Estimate cost for a workload spec | `viewer` | read |
| `POST` | `/cost/compare` | Compare cost across substrates | `viewer` | read |
| `GET` | `/cost/budgets` | List budget policies | `viewer` | read |
| `POST` | `/cost/budgets` | Create a budget policy | `admin` | write |
| `GET` | `/cost/budgets/{id}` | Get a budget policy | `viewer` | read |
| `PATCH` | `/cost/budgets/{id}` | Update a budget policy | `admin` | write |
| `DELETE` | `/cost/budgets/{id}` | Delete a budget policy | `admin` | write |
| `GET` | `/cost/spend` | Get current spend summary for the tenant | `viewer` | read |

#### Tenants

| Method | Path | Description | Auth | Rate Limit Tier |
|--------|------|-------------|------|-----------------|
| `POST` | `/tenants` | Create a new tenant | `platform_admin` | write |
| `GET` | `/tenants` | List tenants (platform admin only) | `platform_admin` | read |
| `GET` | `/tenants/{id}` | Get tenant details | `admin` | read |
| `PATCH` | `/tenants/{id}` | Update tenant config (quotas, LLM settings) | `admin` | write |
| `POST` | `/tenants/{id}/suspend` | Suspend a tenant | `platform_admin` | write |
| `POST` | `/tenants/{id}/reactivate` | Reactivate a suspended tenant | `platform_admin` | write |
| `DELETE` | `/tenants/{id}` | Schedule tenant deletion (soft delete + 30-day retention) | `platform_admin` | write |

#### Audit

| Method | Path | Description | Auth | Rate Limit Tier |
|--------|------|-------------|------|-----------------|
| `GET` | `/audit/events` | Query audit trail (cursor-paginated, filterable) | `admin` | read |
| `GET` | `/audit/events/{id}` | Get a single audit event | `admin` | read |

#### Topology

| Method | Path | Description | Auth | Rate Limit Tier |
|--------|------|-------------|------|-----------------|
| `GET` | `/topology` | Get topology graph (nodes + edges, server-aggregated) | `viewer` | read |
| `GET` | `/topology/subgraph` | Get subgraph rooted at a specific device | `viewer` | read |
| `GET` | `/topology/blast-radius/{id}` | Get blast radius for a device (all affected resources) | `viewer` | read |
| `GET` | `/topology/path` | Shortest path between two devices | `viewer` | read |

#### System (no tenant scope)

| Method | Path | Description | Auth | Rate Limit Tier |
|--------|------|-------------|------|-----------------|
| `GET` | `/healthz` | Health check (all components) | none | none |
| `GET` | `/readyz` | Readiness check (can serve traffic) | none | none |
| `GET` | `/api/openapi.json` | OpenAPI 3.1 specification | none | none |

### Pagination

Cursor-based on all list endpoints. Cursor is an opaque base64-encoded token containing the sort key and row ID of the last returned item. Default page size: 50. Maximum: 200.

```go
// ListInput is embedded by all list endpoint inputs.
type ListInput struct {
    Cursor string `query:"cursor" doc:"Opaque pagination cursor from previous response"`
    Limit  int    `query:"limit" default:"50" minimum:"1" maximum:"200" doc:"Items per page"`
}

// ListOutput is embedded by all list endpoint outputs.
type ListOutput[T any] struct {
    Items      []T    `json:"items"`
    NextCursor string `json:"next_cursor,omitempty"`
    HasMore    bool   `json:"has_more"`
    Total      *int   `json:"total,omitempty"` // only included if ?include_total=true (expensive)
}
```

```bash
# Example: paginate through devices
curl -H "Authorization: Bearer $TOKEN" \
  "https://loom.example.com/api/v1/devices?limit=50"

# Follow pagination
curl -H "Authorization: Bearer $TOKEN" \
  "https://loom.example.com/api/v1/devices?cursor=eyJpZCI6IjU1MGU4ND...&limit=50"
```

### Filtering

Field-based query parameters with operator suffixes. Operators supported:

| Operator | Suffix | Example | SQL Equivalent |
|----------|--------|---------|----------------|
| Equals (default) | none | `?status=active` | `WHERE status = 'active'` |
| Not equals | `__ne` | `?status__ne=decommissioned` | `WHERE status != 'decommissioned'` |
| Greater than | `__gt` | `?created_at__gt=2026-01-01T00:00:00Z` | `WHERE created_at > '2026-01-01'` |
| Less than | `__lt` | `?created_at__lt=2026-03-01T00:00:00Z` | `WHERE created_at < '2026-03-01'` |
| In set | `__in` | `?type__in=server,switch` | `WHERE type IN ('server','switch')` |
| Contains | `__contains` | `?name__contains=prod` | `WHERE name ILIKE '%prod%'` |

```bash
# Find all active switches made by Cisco
curl -H "Authorization: Bearer $TOKEN" \
  "https://loom.example.com/api/v1/devices?status=active&type=switch&vendor__contains=cisco"
```

### Bulk Operations

Bulk endpoints accept arrays with a maximum of 500 items. Responses report per-item success/failure:

```go
// BulkCreateTargetsInput is the request body for POST /targets/batch.
type BulkCreateTargetsInput struct {
    Items []CreateTargetInput `json:"items" minItems:"1" maxItems:"500"`
}

// BulkResult reports per-item outcomes.
type BulkResult[T any] struct {
    Succeeded []BulkItemResult[T] `json:"succeeded"`
    Failed    []BulkItemError     `json:"failed"`
    Total     int                 `json:"total"`
}

// BulkItemResult wraps a successfully processed item with its index.
type BulkItemResult[T any] struct {
    Index int `json:"index"` // position in the original request array
    Item  T   `json:"item"`
}

// BulkItemError reports a failure for a specific item in a bulk request.
type BulkItemError struct {
    Index   int    `json:"index"`
    Code    string `json:"code"`
    Message string `json:"message"`
}
```

The HTTP status for a bulk response is `207 Multi-Status` when some items succeed and some fail. `201 Created` when all succeed. `400 Bad Request` when the request envelope is invalid (before per-item processing).

```bash
# Bulk create targets
curl -X POST -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -H "Idempotency-Key: $(uuidgen)" \
  "https://loom.example.com/api/v1/targets/batch" \
  -d '{
    "items": [
      {"address": "10.0.1.1", "protocol_hint": "redfish"},
      {"address": "10.0.1.2", "protocol_hint": "ssh"},
      {"address": "10.0.1.3", "protocol_hint": "snmp"}
    ]
  }'
```

### Error Responses

All errors use RFC 7807 Problem Details format:

```go
// ProblemDetail implements RFC 7807.
type ProblemDetail struct {
    Type           string         `json:"type"`            // URI reference identifying the problem type
    Title          string         `json:"title"`           // short human-readable summary
    Status         int            `json:"status"`          // HTTP status code
    Detail         string         `json:"detail"`          // human-readable explanation specific to this occurrence
    Instance       string         `json:"instance"`        // URI for this specific occurrence (correlation_id-based)
    CorrelationID  string         `json:"correlation_id"`  // X-Correlation-ID value
    Errors         []FieldError   `json:"errors,omitempty"` // field-level validation errors
}

// FieldError identifies a specific field that failed validation.
type FieldError struct {
    Field   string `json:"field"`   // JSON path to the field (e.g., "items[2].address")
    Code    string `json:"code"`    // machine-readable error code (e.g., "required", "invalid_format")
    Message string `json:"message"` // human-readable message
}
```

Problem type URIs follow the pattern `https://loom.dev/problems/{code}`:

| Type URI | Status | Title |
|----------|--------|-------|
| `https://loom.dev/problems/not-found` | 404 | Resource Not Found |
| `https://loom.dev/problems/validation-error` | 400 | Validation Error |
| `https://loom.dev/problems/unauthorized` | 401 | Authentication Required |
| `https://loom.dev/problems/forbidden` | 403 | Insufficient Permissions |
| `https://loom.dev/problems/conflict` | 409 | Resource Conflict |
| `https://loom.dev/problems/rate-limited` | 429 | Rate Limit Exceeded |
| `https://loom.dev/problems/quota-exceeded` | 409 | Tenant Quota Exceeded |
| `https://loom.dev/problems/budget-exceeded` | 409 | Budget Limit Exceeded |
| `https://loom.dev/problems/tenant-suspended` | 403 | Tenant Suspended |
| `https://loom.dev/problems/internal-error` | 500 | Internal Server Error |

```bash
# Example error response
HTTP/1.1 400 Bad Request
Content-Type: application/problem+json
X-Correlation-ID: req-abc-123

{
  "type": "https://loom.dev/problems/validation-error",
  "title": "Validation Error",
  "status": 400,
  "detail": "The request body contains 2 validation errors.",
  "instance": "urn:loom:error:req-abc-123",
  "correlation_id": "req-abc-123",
  "errors": [
    {"field": "address", "code": "required", "message": "address is required"},
    {"field": "port", "code": "out_of_range", "message": "port must be between 1 and 65535"}
  ]
}
```

### Rate Limiting

Per-tenant token bucket, backed by Valkey. Four tiers with distinct limits:

| Tier | Default Limit | Burst | Applies To |
|------|--------------|-------|------------|
| `read` | 600 req/min | 100 | GET endpoints |
| `write` | 120 req/min | 20 | POST/PATCH/DELETE (non-bulk, non-workflow) |
| `bulk` | 10 req/min | 5 | Batch create/delete endpoints |
| `workflow` | 30 req/min | 10 | Discovery scans, workflow submissions |

Rate limit tier is configurable per tenant via `TenantConfig.RateLimits`. Premium tenants can have higher limits.

Headers on every response:

```
X-RateLimit-Limit: 600
X-RateLimit-Remaining: 594
X-RateLimit-Reset: 1711929600
```

When rate-limited:

```
HTTP/1.1 429 Too Many Requests
Retry-After: 12
Content-Type: application/problem+json

{
  "type": "https://loom.dev/problems/rate-limited",
  "title": "Rate Limit Exceeded",
  "status": 429,
  "detail": "Rate limit for 'write' tier exceeded. Retry after 12 seconds."
}
```

### Resource Expansion

To avoid N+1 requests, list and get endpoints support the `?expand` parameter:

```bash
# Get a device with its endpoints and facts included
curl -H "Authorization: Bearer $TOKEN" \
  "https://loom.example.com/api/v1/devices/abc-123?expand=endpoints,facts"
```

Supported expansions per resource:

| Resource | Expandable Fields |
|----------|------------------|
| Device | `endpoints`, `facts`, `relationships` |
| Endpoint | `device`, `credential` (metadata only) |
| Workflow | `steps`, `events` |

### Field Selection

To reduce payload size on large inventories:

```bash
# Get only id, name, and status for devices
curl -H "Authorization: Bearer $TOKEN" \
  "https://loom.example.com/api/v1/devices?fields=id,name,status"
```

When `?fields` is provided, only the listed fields plus `id` (always included) are returned.

### Webhook Contract

External systems register for event notifications via webhooks:

```go
// Webhook defines a tenant's webhook subscription.
type Webhook struct {
    ID          string   `json:"id"`
    TenantID    string   `json:"tenant_id"`
    URL         string   `json:"url"`           // HTTPS endpoint that receives POST requests
    Events      []string `json:"events"`        // e.g., ["device.created", "workflow.completed", "workflow.failed"]
    Secret      string   `json:"secret"`         // HMAC-SHA256 signing key (write-only, never returned on GET)
    Active      bool     `json:"active"`
    CreatedAt   string   `json:"created_at"`
}
```

| Method | Path | Description |
|--------|------|-------------|
| `POST` | `/webhooks` | Register a webhook |
| `GET` | `/webhooks` | List webhooks |
| `GET` | `/webhooks/{id}` | Get a webhook |
| `PATCH` | `/webhooks/{id}` | Update a webhook |
| `DELETE` | `/webhooks/{id}` | Delete a webhook |
| `POST` | `/webhooks/{id}/test` | Send a test payload |

Webhook delivery: POST to the registered URL with the event envelope as the body. The `X-Loom-Signature` header contains `sha256=<HMAC-SHA256(body, secret)>`. Delivery retries: 3 attempts with exponential backoff (1s, 10s, 60s). After 3 failures, the webhook is marked `active: false` and a `loom.{tenant_id}.webhook.disabled` event is emitted. The tenant must re-enable it manually.

---

## 2. Observability Self-Debugging (Gap 4)

### Decision

Observability outputs (metrics, traces, logs) are necessary but not sufficient. LOOM ships with debugging runbooks, a CLI debug toolkit, and log aggregation guidance. Grafana + Loki is the expected log exploration stack; LOOM does not embed a log viewer.

### Structured Logging Specification

All logs use Go `slog` with JSON output. Every log line includes:

```go
// LogFields are automatically injected by middleware.
type LogFields struct {
    Time          string `json:"time"`           // RFC 3339 with milliseconds
    Level         string `json:"level"`          // DEBUG, INFO, WARN, ERROR
    Msg           string `json:"msg"`            // human-readable message
    CorrelationID string `json:"correlation_id"` // from X-Correlation-ID
    TenantID      string `json:"tenant_id"`      // from JWT
    Component     string `json:"component"`      // "api", "workflow", "adapter", "projector", "llm"
    TraceID       string `json:"trace_id"`       // OpenTelemetry trace ID
    SpanID        string `json:"span_id"`        // OpenTelemetry span ID
}
```

Adapter calls additionally log:

```json
{
  "adapter": "redfish",
  "device_id": "dev-456",
  "endpoint_id": "ep-789",
  "protocol": "redfish",
  "operation": "get_chassis",
  "duration_ms": 342,
  "http_status": 200,
  "http_method": "GET",
  "http_url": "https://10.0.1.5/redfish/v1/Chassis/1"
}
```

Raw HTTP request/response bodies are logged at `DEBUG` level only, with credential scrubbing enforced at the handler level. In production, `DEBUG` is disabled by default but can be enabled per-adapter via runtime config (`PUT /api/v1/admin/config/adapters/{name}/log-level`).

### Log Aggregation Strategy

LOOM writes structured JSON to stdout. The deployment environment is responsible for collection:

| Deployment Mode | Aggregation |
|----------------|-------------|
| Mode 1 (single binary) | stdout + optional file rotation via `--log-file` flag |
| Mode 2 (distributed) | stdout captured by systemd journal, forwarded to Loki via Promtail |
| Mode 3 (Kubernetes) | stdout captured by K8s log driver, forwarded to Loki via Grafana Agent |
| Mode 4 (hub-spoke) | Edge agents log locally + forward to hub via NATS subject `loom._system.log.{agent_id}` |

### Distributed Tracing Specification

Traces span from API request through Temporal workflow through adapter call to device. Adapter spans include:

| Span Attribute | Description |
|----------------|-------------|
| `loom.adapter.http.request_headers` | Redacted request headers (auth removed) |
| `loom.adapter.http.response_status` | HTTP status code |
| `loom.adapter.http.response_body_size` | Response body size in bytes |
| `loom.adapter.duration_ms` | Adapter call duration |
| `loom.adapter.retry_count` | Number of retries before success/failure |
| `loom.adapter.error_type` | `transient`, `permanent`, `partial`, `validation` |

When an adapter call fails, the trace includes the full raw HTTP response (or protocol-specific error) as a span event, enabling operators to see exactly what the device returned.

### Metrics: RED per Adapter, per Tenant, per Device Type

Every adapter reports RED metrics with three label dimensions:

```
loom_adapter_operation_duration_seconds{adapter="redfish", tenant_id="t-abc", device_type="server", operation="get_chassis"}
loom_adapter_errors_total{adapter="redfish", tenant_id="t-abc", device_type="server", error_type="transient"}
loom_adapter_requests_total{adapter="redfish", tenant_id="t-abc", device_type="server", operation="get_chassis", result="success"}
```

### Dashboard Specifications

Four pre-built Grafana dashboards ship as JSON in `deploy/grafana/dashboards/`:

**Dashboard 1: Operator Overview** -- see [OBSERVABILITY-STRATEGY.md](OBSERVABILITY-STRATEGY.md) Section 4 (already specified).

**Dashboard 2: Adapter Health** (new)

| Panel | Shows |
|-------|-------|
| Adapter Success Rate | Per-adapter success/failure rate over time |
| Adapter Latency Heatmap | P50/P95/P99 per adapter as heat map |
| Adapter Connection Pool | Active/idle/waiting connections per protocol |
| Error Classification | Pie chart: transient vs permanent vs partial vs validation |
| Top Failing Devices | Table: devices with highest adapter error rate |

**Dashboard 3: Tenant Dashboard** (new, tenant-scoped)

| Panel | Shows |
|-------|-------|
| My Devices | Total devices by status |
| My Workflows | Active/completed/failed workflows |
| My Spend | Current period spend vs budget |
| My API Usage | Request rate + rate limit utilization |
| My LLM Usage | LLM calls + cost this period |

**Dashboard 4: Debugging Dashboard** (new)

| Panel | Shows |
|-------|-------|
| Workflow Timeline | Gantt chart of workflow step durations |
| Correlation ID Search | Text input -> log lines + trace link |
| Event Stream | Live tail of NATS events for a selected tenant |
| NATS Consumer Lag | Per-consumer pending message count |
| Temporal Task Queue Depth | Per-queue depth and schedule-to-start latency |

### Alert Rules

Supplement the existing alerts in OBSERVABILITY-STRATEGY.md Section 5 with:

| Alert | Condition | Severity |
|-------|-----------|----------|
| `LOOMAdapterErrorRateHigh` | `rate(loom_adapter_errors_total{error_type!="transient"}[5m]) / rate(loom_adapter_requests_total[5m]) > 0.05` for 5m | Warning |
| `LOOMWorkflowStuck` | Workflow in `running` or `awaiting_approval` state for >3 hours | Warning |
| `LOOMDeviceUnreachableCluster` | >10% of devices for a tenant in `unreachable` status | Critical |
| `LOOMProjectionLag` | DB projection consumer lag >5000 messages for 10m | Warning |

### Debugging Runbooks

Five runbooks ship in `docs/runbooks/`:

**Runbook 1: Workflow Stuck**
1. Get workflow ID from dashboard or alert.
2. `loom admin workflow inspect <workflow-id>` -- shows current step, Temporal state, last activity attempt, error.
3. Check if blocked on approval: if `state=awaiting_approval`, check approver notification.
4. Check if blocked on adapter timeout: inspect the trace for the current activity span.
5. If Temporal replay error: check if workflow code was changed mid-execution (see Gap 18 for versioning).
6. Manual resolution: `loom admin workflow signal <workflow-id> --signal=force-complete` or `--signal=cancel`.

**Runbook 2: Adapter Timeout**
1. Identify the device and endpoint from the alert/trace.
2. `loom admin adapter test <endpoint-id>` -- attempts a connectivity check.
3. If device is unreachable: check network path, device power state, firewall rules.
4. If device responds but slowly: check device load, reduce concurrent connections via `TenantConfig.MaxConcurrentAdapterConnections`.
5. If adapter code is failing: enable DEBUG logging for that adapter, reproduce, file a bug with the trace.

**Runbook 3: LLM Confidence Too Low**
1. Check `loom_llm_decisions_total{result="fallback"}` -- is this a spike or steady state?
2. `loom admin llm inspect <correlation-id>` -- shows the LLM request context, response, confidence score, and fallback decision.
3. If context was too large (truncated): check `loom_llm_context_tokens` metric, adjust summarization depth.
4. If model quality issue: compare outputs across providers, consider upgrading model tier.
5. Deterministic fallback is always safe -- investigate at leisure, not under pager pressure.

**Runbook 4: Event Consumer Falling Behind**
1. Check NATS consumer lag: `nats consumer info LOOM_EVENTS db-projector`.
2. If lag is growing: check projector processing rate, check PostgreSQL write latency.
3. If PostgreSQL is the bottleneck: check connection pool saturation, check active queries, check disk I/O.
4. If sustained: scale projector workers (increase `MaxAckPending`), or investigate slow event processing (single expensive event type).
5. Consumer lag does not affect API writes or workflow execution -- only the DB projection falls behind.

**Runbook 5: Tenant Isolation Violation Suspected**
1. Identify the suspicious access from audit logs: `GET /api/v1/audit/events?actor_id=<user>&action=read`.
2. Verify JWT claims: was `tenant_id` in the token correct?
3. Check if the query path has a `WHERE tenant_id = ?` clause (all repository methods must).
4. Run the tenant isolation integration test: `go test -tags=integration -run TestTenantIsolation`.
5. If confirmed: incident response per SECURITY-POLICY.md. Rotate all credentials for the affected tenant.

### CLI Debug Toolkit

```bash
# Inspect a workflow's full state (Temporal + DB + NATS events)
loom admin workflow inspect <workflow-id>

# Test adapter connectivity to a specific endpoint
loom admin adapter test <endpoint-id>

# Replay an LLM decision with debug output
loom admin llm inspect <correlation-id>

# Tail events for a tenant
loom admin events tail --tenant <tenant-id> --filter "device.*"

# Check system health with component details
loom admin health --verbose
```

---

## 3. LLM Integration Practicalities (Gap 5)

### Decision

LLM calls are bounded by explicit latency budgets, token budgets, and cost budgets. Every LLM integration point has a deterministic fallback that activates automatically on timeout, cost cap, or low confidence. The LLM is never in the critical path for system availability.

### Latency Budget

| Task Type | LLM Timeout | Temporal Activity Timeout | Retry Budget |
|-----------|------------|--------------------------|--------------|
| Classification | 5s | 30s (short) | 2 retries (15s total) |
| Placement | 15s | 5m (medium) | 1 retry (30s total) |
| Config suggestion | 30s | 5m (medium) | 0 retries (fallback to template) |
| Oversight detection | 10s | 30s (short) | 1 retry (20s total) |
| Anomaly correlation | 10s | 30s (short) | 0 retries (fallback to threshold) |
| Cost optimization | 5s | 30s (short) | 1 retry (10s total) |

If the LLM call exceeds its timeout, the activity immediately falls back to the deterministic path. The LLM timeout is enforced via Go `context.WithTimeout`, independent of the Temporal activity timeout.

### Cost Budget

```go
// LLMCostConfig defines per-tenant LLM cost guardrails.
type LLMCostConfig struct {
    MonthlyCapUSD       float64 `json:"monthly_cap_usd"`        // default: $100/month
    DailyCapUSD         float64 `json:"daily_cap_usd"`          // default: $10/day (circuit breaker)
    PerRequestCapUSD    float64 `json:"per_request_cap_usd"`    // default: $1.00 (reject single request over this)
    WarningThresholdPct float64 `json:"warning_threshold_pct"`  // default: 0.8 (alert at 80% of monthly cap)
}
```

Cost tracking is per-tenant, per-model. Tracked via the `loom_llm_cost_dollars` counter with `{tenant_id, provider, model}` labels. When `daily_cap_usd` is exceeded, all LLM calls for that tenant fall back to deterministic for the remainder of the calendar day (UTC). This is a circuit breaker, not a hard reject -- operations continue with deterministic fallbacks.

```bash
# Check LLM spend for a tenant
curl -H "Authorization: Bearer $TOKEN" \
  "https://loom.example.com/api/v1/tenants/t-abc/llm-usage?period=current_month"

# Response
{
  "tenant_id": "t-abc",
  "period": "2026-03",
  "total_cost_usd": 47.23,
  "monthly_cap_usd": 100.00,
  "requests": 1247,
  "tokens_input": 2340000,
  "tokens_output": 580000,
  "cache_hit_rate": 0.34,
  "by_task_type": {
    "classification": {"cost_usd": 12.10, "requests": 800},
    "placement": {"cost_usd": 28.50, "requests": 310},
    "oversight": {"cost_usd": 6.63, "requests": 137}
  }
}
```

### Context Window Management

Maximum 4K tokens per LLM request. Device metadata is summarized, not sent raw.

```go
// LLMContext is the structured input provided to the LLM for any decision.
type LLMContext struct {
    TaskType        string         `json:"task_type"`
    TenantID        string         `json:"tenant_id"`
    DeviceSummary   []DeviceBrief  `json:"device_summary"`    // max 50 devices, summarized
    Constraints     []string       `json:"constraints"`        // natural language constraints
    CostContext     *CostContext   `json:"cost_context,omitempty"`
    RecentDecisions []DecisionRef  `json:"recent_decisions"`  // last 5 similar decisions for consistency
    MaxTokens       int            `json:"max_tokens"`         // output token limit
}

// DeviceBrief is a token-efficient device summary for LLM context.
type DeviceBrief struct {
    ID       string `json:"id"`
    Name     string `json:"name"`
    Type     string `json:"type"`
    Vendor   string `json:"vendor"`
    Status   string `json:"status"`
    Location string `json:"location"`
}
```

For placement decisions involving large fleets (>50 devices), a pre-filtering step selects the top 50 candidate devices using deterministic rules (capacity, location, affinity) before constructing the LLM context. The LLM ranks and refines; it does not search.

### Model Selection

```yaml
# loom.yaml -- LLM provider configuration
llm:
  default_provider: "claude"
  providers:
    claude:
      endpoint: "https://api.anthropic.com/v1"
      model: "claude-sonnet-4-20250514"
      api_key_env: "ANTHROPIC_API_KEY"
    ollama:
      endpoint: "http://localhost:11434/v1"
      model: "llama3.1:8b"
      # No API key needed for local
    vllm:
      endpoint: "http://gpu-server:8000/v1"
      model: "meta-llama/Llama-3.1-70B-Instruct"
  fallback_chain: ["claude", "ollama"]  # try claude, fall back to ollama
  air_gapped_mode: false                 # when true, only local providers are used
```

### Semantic Cache

Valkey-backed semantic cache using embedding similarity:

```go
// SemanticCacheConfig controls the LLM response cache.
type SemanticCacheConfig struct {
    Enabled            bool          `json:"enabled"`             // default: true
    SimilarityThreshold float64      `json:"similarity_threshold"` // default: 0.95
    TTL                time.Duration `json:"ttl"`                  // default: 1 hour
    MaxEntries         int           `json:"max_entries"`           // default: 10000 per tenant
    EmbeddingModel     string        `json:"embedding_model"`       // model used for embedding (e.g., "text-embedding-3-small")
}
```

Cache flow:
1. Hash the LLM request context (task type + constraints + device summary).
2. Generate embedding of the request context.
3. Search Valkey for cached responses with embedding similarity > 0.95.
4. If cache hit: return cached response, increment `loom_llm_cache_hits_total`.
5. If cache miss: call LLM, store response + embedding in Valkey with TTL.

### Fallback Chain

```
LLM Provider (within timeout)
  → Cached LLM response (semantic similarity > 0.95)
    → Deterministic rules engine (always available, always fast)
```

The deterministic fallback for each task type:
- **Classification**: sysObjectID lookup table + Redfish schema fingerprinting.
- **Placement**: bin-packing algorithm with capacity + affinity constraint solver.
- **Config suggestion**: Go template engine with vendor-specific templates.
- **Oversight**: checklist validation against a rule set (JSON-configurable).
- **Cost optimization**: simple threshold alerts (cheapest available substrate).

---

## 4. Multi-Tenancy Application Layer (Gap 6)

### Decision

Multi-tenancy goes beyond `WHERE tenant_id = ?`. LOOM implements quotas, lifecycle management, per-tenant configuration, and resource isolation at Temporal, NATS, and DB layers. Shared devices use an infrastructure tenant model.

### Tenant Configuration

```go
// TenantConfig captures all per-tenant settings.
type TenantConfig struct {
    ID                  string          `json:"id" db:"id"`
    TenantID            string          `json:"tenant_id" db:"tenant_id"`
    DisplayName         string          `json:"display_name" db:"display_name"`
    Status              TenantStatus    `json:"status" db:"status"` // "active", "suspended", "pending_deletion"

    // Quotas
    MaxDevices          int             `json:"max_devices" db:"max_devices"`           // default: 1000
    MaxWorkflowsPerDay  int             `json:"max_workflows_per_day" db:"max_workflows_per_day"` // default: 500
    MaxConcurrentWorkflows int          `json:"max_concurrent_workflows" db:"max_concurrent_workflows"` // default: 50
    MaxConcurrentAdapterConnections int `json:"max_concurrent_adapter_connections" db:"max_concurrent_adapter_connections"` // default: 100
    MaxStorageGB        int             `json:"max_storage_gb" db:"max_storage_gb"`     // default: 50
    MaxCredentials      int             `json:"max_credentials" db:"max_credentials"`   // default: 500

    // Rate Limits (override defaults from Section 1)
    RateLimits          *RateLimitOverrides `json:"rate_limits,omitempty" db:"-"`

    // LLM Settings
    LLMConfig           LLMCostConfig   `json:"llm_config" db:"-"`
    PreferredLLMProvider string         `json:"preferred_llm_provider" db:"preferred_llm_provider"` // override default provider

    // Cost
    BudgetPolicies      []string        `json:"budget_policy_ids" db:"-"` // references to BudgetPolicy IDs

    // Discovery
    DiscoveryScheduleCron string        `json:"discovery_schedule_cron" db:"discovery_schedule_cron"` // e.g., "0 */6 * * *"
    AutoDiscoveryEnabled  bool          `json:"auto_discovery_enabled" db:"auto_discovery_enabled"`

    // Approval Policy
    ApprovalPolicy      ApprovalPolicy  `json:"approval_policy" db:"-"`

    CreatedAt           time.Time       `json:"created_at" db:"created_at"`
    UpdatedAt           time.Time       `json:"updated_at" db:"updated_at"`
    SuspendedAt         *time.Time      `json:"suspended_at,omitempty" db:"suspended_at"`
    DeletionScheduledAt *time.Time      `json:"deletion_scheduled_at,omitempty" db:"deletion_scheduled_at"`
}

// TenantStatus tracks tenant lifecycle state.
type TenantStatus string

const (
    TenantStatusActive          TenantStatus = "active"
    TenantStatusSuspended       TenantStatus = "suspended"
    TenantStatusPendingDeletion TenantStatus = "pending_deletion"
)

// ApprovalPolicy defines when human approval is required.
type ApprovalPolicy struct {
    RequireApprovalForTypes []string `json:"require_approval_for_types"` // workflow types that need approval
    AutoApproveRiskLevel   string   `json:"auto_approve_risk_level"`    // "low", "medium", "high", "none"
}
```

### Quota Enforcement Middleware

Quota checks run as HTTP middleware before the handler executes:

```go
// QuotaMiddleware checks tenant quotas before allowing mutating operations.
func QuotaMiddleware(quotaService QuotaService) func(http.Handler) http.Handler {
    return func(next http.Handler) http.Handler {
        return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
            tenantID := auth.TenantIDFromContext(r.Context())
            resource := quotaResourceFromPath(r.URL.Path, r.Method)

            if resource != "" {
                if err := quotaService.Check(r.Context(), tenantID, resource); err != nil {
                    // Returns 409 with "quota-exceeded" problem type
                    writeProblemDetail(w, quotaExceededProblem(tenantID, resource, err))
                    return
                }
            }
            next.ServeHTTP(w, r)
        })
    }
}
```

Quota resource mapping:

| HTTP Path Pattern | Resource | Quota Field |
|-------------------|----------|-------------|
| `POST /targets` | `targets` | `MaxDevices` (targets eventually become devices) |
| `POST /credentials` | `credentials` | `MaxCredentials` |
| `POST /workflows` | `workflows` | `MaxWorkflowsPerDay` + `MaxConcurrentWorkflows` |
| `POST /discovery/scans` | `workflows` | `MaxWorkflowsPerDay` + `MaxConcurrentWorkflows` |

Quota counts are maintained as materialized counters in Valkey, refreshed from PostgreSQL every 60 seconds. This avoids full table scans for "how many devices does Tenant A have?"

### Tenant Lifecycle

| Operation | What Happens |
|-----------|-------------|
| **Create** | Insert `tenant_config` row. Create Temporal namespace `loom-{tenant_id}`. Create NATS account `tenant-{tenant_id}`. Emit `tenant.created` event. |
| **Suspend** | Set `status=suspended`, `suspended_at=now()`. All API calls except GET return `403 Tenant Suspended`. Running workflows are allowed to complete but no new workflows can start. No new adapter connections. |
| **Reactivate** | Set `status=active`, clear `suspended_at`. Normal operations resume. |
| **Delete (schedule)** | Set `status=pending_deletion`, `deletion_scheduled_at=now()+30d`. Behaves like suspended. Emit `tenant.deletion_scheduled` event. |
| **Delete (execute, 30 days later)** | Background job: cancel all running workflows. Delete all devices, endpoints, facts, relationships, credentials. Archive audit records to cold storage (retained for 7 years per compliance). Drop Temporal namespace. Delete NATS account. Remove `tenant_config` row. Emit `tenant.deleted` event. |
| **Delete (cancel)** | If within 30-day window: reactivate. If past 30 days: data is gone. |

### Resource Isolation

| Layer | Isolation Mechanism |
|-------|-------------------|
| PostgreSQL | Row-Level Security (RLS) policies on all tenant-scoped tables. Every query passes through RLS. |
| Temporal | Separate namespace per tenant (`loom-{tenant_id}`). Workflows cannot cross namespace boundaries. |
| NATS | Separate NATS account per tenant. Publish/subscribe restricted to `loom.{tenant_id}.>`. System consumers use a system account. |
| Valkey | Key prefix `tenant:{tenant_id}:` on all tenant-scoped keys. |
| LLM | Per-tenant cost tracking and budget enforcement. LLM context never includes data from other tenants. |

### Tenant Billing Hooks

LOOM emits usage tracking events for external billing integration:

```
loom.{tenant_id}.usage.devices_count      -- daily snapshot of device count
loom.{tenant_id}.usage.api_requests       -- hourly API request count by tier
loom.{tenant_id}.usage.workflow_executions -- daily workflow execution count
loom.{tenant_id}.usage.llm_spend          -- daily LLM spend in USD
loom.{tenant_id}.usage.storage_gb         -- daily storage consumption
```

These events contain the raw numbers. LOOM does not implement billing -- it provides the usage data for an external billing system to consume via NATS subscription or webhook.

### Shared Device Ownership

**Decision**: Use an infrastructure tenant model. Shared physical devices (switches, hypervisors, storage arrays) belong to a reserved `_infrastructure` tenant. Per-tenant virtual device projections are created for tenant visibility:

```go
// SharedDeviceMapping links a physical device (owned by _infrastructure tenant)
// to a tenant's virtual view of that device.
type SharedDeviceMapping struct {
    ID              string `json:"id"`
    PhysicalDeviceID string `json:"physical_device_id"` // device in _infrastructure tenant
    TenantDeviceID   string `json:"tenant_device_id"`   // virtual device in tenant's scope
    TenantID         string `json:"tenant_id"`
    AccessLevel      string `json:"access_level"`        // "read_only", "configure", "full"
}
```

Tenant A sees a virtual device record. Operations on the virtual device are translated to operations on the physical device with access level checks. Device locking operates on the physical device ID to prevent concurrent modifications across tenants.

---

## 5. Testing Strategy for 25+ Adapters (Gap 7)

### Decision

Adapters use a four-level test pyramid with shared infrastructure. Not every adapter runs at every level in CI. A conformance test suite validates contract compliance. The "device zoo" pattern captures real device responses for replay.

### Test Pyramid

| Level | Description | Runs In | Coverage |
|-------|-------------|---------|----------|
| L1: Unit | Mock protocol server, happy path + error classification | Every PR | All adapters |
| L2: Conformance | Adapter conformance suite validates all 7 contract patterns | Every PR | All adapters |
| L3: Integration | Containerized device simulators (SONiC-VS, DMTF Redfish Mockup Server) | Nightly | Adapters with available simulators |
| L4: E2E / Lab | Real device in a test lab, manual trigger | Pre-release | Priority adapters (Redfish, SSH, SNMP, NETCONF) |

### Conformance Test Suite

The conformance suite is a Go test package that adapter authors import:

```go
package conformance

// RunConformanceSuite runs all adapter contract tests against the provided adapter.
// The adapter must be wired to a mock or real device.
func RunConformanceSuite(t *testing.T, adapter Adapter, mockEndpoint string) {
    t.Run("happy_path", func(t *testing.T) { testHappyPath(t, adapter, mockEndpoint) })
    t.Run("connection_failure", func(t *testing.T) { testConnectionFailure(t, adapter) })
    t.Run("auth_failure", func(t *testing.T) { testAuthFailure(t, adapter, mockEndpoint) })
    t.Run("timeout", func(t *testing.T) { testTimeout(t, adapter, mockEndpoint) })
    t.Run("partial_failure", func(t *testing.T) { testPartialFailure(t, adapter, mockEndpoint) })
    t.Run("idempotency", func(t *testing.T) { testIdempotency(t, adapter, mockEndpoint) })
    t.Run("compensation", func(t *testing.T) { testCompensation(t, adapter, mockEndpoint) })
}
```

### Containerized Device Simulators for CI

| Adapter | Simulator | Docker Image | License |
|---------|-----------|-------------|---------|
| Redfish | DMTF Redfish Mockup Server | `dmtf/redfish-mockup-server` | BSD |
| SNMP | GoSNMP mock agent | Custom (in-repo) | MIT |
| SSH | gliderlabs/ssh test server | Custom (in-repo) | MIT |
| gNMI | Nokia SR Linux container | `ghcr.io/nokia/srlinux` | Free tier |
| NETCONF | Netopeer2 | `sysrepo/sysrepo-netopeer2` | BSD |
| eAPI | Arista cEOS-lab | `ceos:latest` | Licensed (Arista account required) |
| SONiC | SONiC-VS | `docker-sonic-vs` | Apache 2.0 |
| PiKVM | Custom HTTP mock | Custom (in-repo) | MIT |

Licensed simulators (cEOS) run in nightly CI only, not on every PR. The CI environment has the license pre-configured.

### Test Matrix

| Adapter | L1 (PR) | L2 (PR) | L3 (Nightly) | L4 (Pre-release) |
|---------|---------|---------|--------------|------------------|
| Redfish | Yes | Yes | Yes (DMTF Mockup) | Yes (Dell iDRAC, HPE iLO) |
| SNMP | Yes | Yes | Yes (GoSNMP mock) | Yes (Cisco, Arista) |
| SSH | Yes | Yes | Yes (gliderlabs) | Yes (SONiC, Cumulus) |
| NETCONF | Yes | Yes | Yes (Netopeer2) | Yes (Cisco IOS-XE) |
| gNMI | Yes | Yes | Yes (SR Linux) | Best effort |
| IPMI | Yes | Yes | No (no simulator) | Yes (any IPMI BMC) |
| eAPI | Yes | Yes | Yes (cEOS, nightly) | Best effort |
| AMT | Yes | Yes | No | Best effort |
| PiKVM | Yes | Yes | Yes (HTTP mock) | Yes (Raspberry Pi lab) |

### Performance Benchmarks

Per-adapter performance benchmarks run nightly and report p50/p95/p99 latency:

```go
func BenchmarkRedfishDiscover(b *testing.B) {
    // Target: p99 < 2s per device
    adapter := redfish.NewAdapter()
    mock := startRedfishMockServer(b)
    b.ResetTimer()
    for i := 0; i < b.N; i++ {
        adapter.Discover(ctx, mock.Endpoint())
    }
}
```

Benchmark results are compared against a baseline using `benchstat`. If p99 regresses by >20%, the nightly run emits a warning (not a blocker, since hardware variance can cause fluctuations).

### Device Zoo

Captured real-device responses stored in `testdata/device-zoo/`:

```
testdata/device-zoo/
├── redfish/
│   ├── dell-idrac9-r750/           # captured from real Dell R750
│   │   ├── chassis.json
│   │   ├── systems.json
│   │   ├── managers.json
│   │   └── metadata.yaml           # device info, capture date, firmware version
│   ├── hpe-ilo5-dl380/
│   └── supermicro-x12/
├── snmp/
│   ├── cisco-c9300-17.3/
│   │   ├── sysdescr.snmprec
│   │   └── iftable.snmprec
│   └── arista-7050-4.28/
└── ssh/
    ├── sonic-202305/
    │   ├── show-version.txt
    │   └── show-interfaces.txt
    └── cumulus-5.5/
```

Mock servers can replay these captured responses for realistic testing without real devices. Contract tests verify that mock servers and real devices produce identical responses for the same operation.

---

## 6. CQRS Projection Rebuild (Gap 8)

### Decision

Event replay uses NATS JetStream as the primary source, with Temporal workflow history as a secondary reconciliation source. Projections are versioned. Rebuild is an offline-safe operation that creates new tables before swapping.

### Event Schema Versioning

Add a `schema_version` field to the event envelope:

```go
type Event struct {
    // ... existing fields ...
    SchemaVersion int `json:"schema_version"` // starts at 1, incremented on breaking changes
}
```

Schema evolution rules:
- **Within a major version**: additive-only changes. New fields with defaults. Old consumers ignore unknown fields.
- **Breaking changes**: increment `SchemaVersion`. Old consumers log a warning and skip events with unknown schema versions.
- **No schema registry**: schema versions are documented in `EVENT-MODEL.md` and enforced in code review. A registry is premature for MVP.

### Event Replay Mechanism

```go
// ProjectionRebuilder replays NATS JetStream events to rebuild a projection.
type ProjectionRebuilder struct {
    Stream      string          // "LOOM_EVENTS"
    Consumer    string          // "rebuild-{projection}-{timestamp}"
    Projection  ProjectionHandler
    BatchSize   int             // events per transaction, default: 1000
}

// ProjectionHandler processes events into a projection table.
type ProjectionHandler interface {
    // TableName returns the projection table name (e.g., "devices_v2").
    TableName() string
    // SchemaVersion returns the current projection schema version.
    SchemaVersion() int
    // Apply processes a single event. Must be idempotent.
    Apply(ctx context.Context, tx pgx.Tx, event Event) error
}
```

### Rebuild Procedure

1. Create a new projection table with a versioned name (`devices_v2`).
2. Create a temporary NATS consumer from `DeliverAllPolicy` (replay all retained events).
3. Replay events in batches of 1000 within database transactions. Each batch is committed atomically.
4. Track replay progress: `{consumer_name, last_event_id, events_processed, started_at}`.
5. When replay reaches current stream position, pause writes to the old projection table.
6. Drain remaining events (small gap from step 5 to now).
7. Rename tables: `devices` -> `devices_v1_archived`, `devices_v2` -> `devices`.
8. Resume normal projection consumer.
9. Drop the old table after 24 hours (safety period for rollback).

```bash
# CLI command for rebuild
loom admin projection rebuild --projection devices --confirm

# Monitor rebuild progress
loom admin projection status
```

### Idempotent Projectors

All projectors must be idempotent. The same event applied twice produces the same state. This is enforced by:
- Using `ON CONFLICT (id) DO UPDATE` for upserts.
- Tracking the last applied event ID per projection row to skip already-processed events.
- The conformance test for projectors replays the same event sequence twice and asserts identical DB state.

### Rebuild Time Estimates

| Fleet Size | Events (90-day) | Estimated Rebuild Time |
|------------|-----------------|----------------------|
| 100 devices | ~500K events | ~2 minutes |
| 1,000 devices | ~5M events | ~15 minutes |
| 10,000 devices | ~50M events | ~2.5 hours |

Estimates assume 2000 events/second replay rate on a single-core projector with PostgreSQL on SSD. Parallelization by tenant ID can reduce this by ~5x.

### 90-Day Retention Gap

NATS retains events for 90 days. For rebuilds beyond 90 days, LOOM maintains a daily snapshot of each projection table in PostgreSQL (a separate schema `snapshots`). Rebuild procedure for >90-day recovery:
1. Restore from the most recent daily snapshot.
2. Replay NATS events from the snapshot timestamp forward.

Daily snapshot is a `pg_dump` of projection tables, stored locally and optionally in object storage.

---

## 7. UI Real-Time & Scale (Gap 9)

### Decision

SSE is the primary real-time transport for unidirectional server-to-client updates. WebSocket is used only for bidirectional terminal access (device console). The UI is a desktop-first operations console; mobile is best-effort responsive, not a first-class target.

### SSE Contract

**Endpoint**: `GET /api/v1/events/stream`

**Authentication**: JWT passed as query parameter (`?token=<jwt>`), since SSE via `EventSource` does not support custom headers. Token is validated on connection and periodically (every 60s) during the connection lifetime.

**Message format**:

```
event: device.status_changed
id: evt-2026-03-21-abc123
data: {"resource_type":"device","resource_id":"dev-456","tenant_id":"t-abc","data":{"status":"active","previous_status":"discovered"},"timestamp":"2026-03-21T10:00:00.123Z"}

event: workflow.step_completed
id: evt-2026-03-21-def456
data: {"resource_type":"workflow","resource_id":"wf-789","tenant_id":"t-abc","data":{"step":"configure_network","result":"success"},"timestamp":"2026-03-21T10:00:01.456Z"}
```

**Subscription filtering**: clients specify which event types they want via query parameters:

```
GET /api/v1/events/stream?token=<jwt>&events=device.*,workflow.*&tenant_id=t-abc
```

**Client-side throttling**: the server applies a maximum event rate of 10 events/second per SSE connection. Events above this rate are batched into a single `batch` event type:

```
event: batch
data: {"events": [...], "dropped": 5}
```

The `dropped` field indicates how many events were discarded during the batch window. The client can request full history via the REST API if needed.

**Reconnection**: SSE supports automatic reconnection via the `Last-Event-ID` header. The server buffers the last 1000 events per tenant in Valkey. On reconnection, missed events are replayed.

### WebSocket Contract (Terminal Access Only)

**Endpoint**: `GET /api/v1/devices/{id}/terminal`

**Authentication**: JWT in the `Sec-WebSocket-Protocol` header (subprotocol `loom-auth-<jwt>`). Validated before the WebSocket upgrade completes.

**Message format**: Binary frames. Client-to-server: terminal input bytes. Server-to-client: terminal output bytes. Control messages use JSON text frames:

```json
{"type": "resize", "cols": 120, "rows": 40}
{"type": "ping"}
{"type": "error", "message": "Device unreachable"}
```

**Reconnection**: exponential backoff starting at 1s, max 30s. No resumption -- terminal session is re-established from scratch.

**Concurrency**: maximum 5 concurrent WebSocket connections per device (prevents console session exhaustion on the target device). Maximum 50 concurrent WebSocket connections per tenant.

### Topology Visualization

**Library**: Cytoscape.js with the `cytoscape-cose-bilkent` layout for force-directed graph rendering.

**Progressive disclosure**:
1. **Cluster level** (default): groups of devices aggregated by site/datacenter. Each cluster node shows device count and health summary.
2. **Site level** (click to expand): devices within a site, grouped by rack.
3. **Rack level** (click to expand): individual devices with status indicators.
4. **Device level** (click on device): detail panel with endpoints, facts, and relationships.

**Scale limits**: maximum 500 nodes rendered simultaneously in the browser. When the total topology exceeds 500 nodes, the server returns aggregated cluster nodes. The `?detail_level` parameter controls aggregation:

```bash
# Get cluster-level topology (always <100 nodes)
GET /api/v1/topology?detail_level=cluster

# Get site-level for a specific cluster
GET /api/v1/topology?detail_level=site&cluster_id=dc-east-1

# Get rack-level for a specific site
GET /api/v1/topology?detail_level=rack&site_id=site-rack-a3
```

**Topology data format**:

```go
// TopologyResponse is the response for GET /topology.
type TopologyResponse struct {
    Nodes []TopologyNode `json:"nodes"`
    Edges []TopologyEdge `json:"edges"`
    Meta  TopologyMeta   `json:"meta"`
}

type TopologyNode struct {
    ID          string            `json:"id"`
    Label       string            `json:"label"`
    Type        string            `json:"type"`       // "device", "cluster", "site", "rack"
    Status      string            `json:"status"`     // aggregate health
    DeviceCount int               `json:"device_count,omitempty"` // for aggregated nodes
    Metadata    map[string]string `json:"metadata,omitempty"`
    Position    *Position         `json:"position,omitempty"` // saved layout position
}

type TopologyEdge struct {
    Source string `json:"source"`
    Target string `json:"target"`
    Type   string `json:"type"` // "connected_to", "member_of", "depends_on"
    Label  string `json:"label,omitempty"`
}
```

**Real-time updates**: device state changes are pushed via SSE. The UI client updates the corresponding Cytoscape node's status property and re-colors it. No full topology re-fetch on every state change.

### Mobile/Responsive

The UI is a desktop-first operations console. Layout is responsive via CSS grid with a minimum viewport of 1024px. Below 1024px, the topology view is hidden and a list view is shown instead. Touch targets are minimum 44x44px for tablet usability. No native mobile app is planned.

---

## 8. CI/CD & Upgrade Testing (Gap 10)

### Decision

E2E tests run on PRs that touch adapter or workflow code (not on every PR). Database migration rollbacks are tested in CI. Upgrade testing is automated. Performance regression detection uses `benchstat`.

### E2E Test Trigger

E2E tests run on every PR if any of these paths are modified:
- `adapter/**`
- `workflow/**`
- `api/**`
- `domain/**`
- `docker-compose*.yml`
- `Dockerfile`

Otherwise, E2E runs nightly only. This balances feedback speed (~10 minutes per E2E run) against CI cost.

### Database Migration Testing

Every PR that adds a migration triggers:

```yaml
test-migrations:
  needs: test-unit
  steps:
    # Test forward migration
    - run: loom migrate up
    # Seed test data
    - run: loom admin seed --fixtures=test
    # Test rollback
    - run: loom migrate down --steps=1
    # Verify rollback succeeded (schema matches pre-migration)
    - run: loom migrate status --verify
    # Re-apply migration (ensures idempotent re-application)
    - run: loom migrate up
```

### Upgrade Path Testing

A nightly CI job tests the upgrade path from `v(N-1)` to `v(N)`:

```yaml
test-upgrade:
  runs-on: ubuntu-latest
  steps:
    - name: Deploy v(N-1)
      run: |
        docker compose -f docker-compose.test.yml up -d
        docker run --network=host loom:previous-release serve --migrate
        loom admin seed --fixtures=upgrade-test

    - name: Upgrade to v(N)
      run: |
        docker run --network=host loom:current serve --migrate

    - name: Verify data integrity
      run: |
        loom admin verify-upgrade --expected-devices=100 --expected-workflows=50

    - name: Verify API compatibility
      run: |
        # Run the v(N-1) SDK test suite against the v(N) API
        go test -tags=compat ./test/compat/...
```

### Backward Compatibility Gate

CI blocks merge if the OpenAPI spec diff shows breaking changes (field removal, type change, endpoint removal) unless the PR title contains `[BREAKING]` and targets a new major version:

```yaml
api-compat-check:
  steps:
    - run: |
        npx @redocly/cli diff api/openapi.json main:api/openapi.json --format=json > diff.json
        if jq '.breaking | length > 0' diff.json; then
          if ! echo "$PR_TITLE" | grep -q '\[BREAKING\]'; then
            echo "::error::Breaking API change detected without [BREAKING] tag"
            exit 1
          fi
        fi
```

### Performance Regression Detection

Go benchmarks run nightly. Results are compared to the `main` branch baseline:

```yaml
benchmark:
  steps:
    - run: go test -bench=. -benchmem -count=5 ./... > new.txt
    - run: git stash && go test -bench=. -benchmem -count=5 ./... > old.txt && git stash pop
    - run: benchstat old.txt new.txt > benchstat.txt
    - uses: actions/upload-artifact@v4
      with:
        name: benchstat
        path: benchstat.txt
```

If any benchmark regresses by >15% (p-value < 0.05), the nightly run posts a GitHub issue.

### Release Artifacts

| Artifact | Format | Built By |
|----------|--------|----------|
| Single binary | `loom` (linux/amd64, linux/arm64, darwin/amd64, darwin/arm64) | GoReleaser |
| Docker image | `ghcr.io/org/loom:v0.3.0` | GoReleaser |
| Helm chart | `loom-0.3.0.tgz` | Helm package in release workflow |
| systemd unit | `loom.service` in release tarball | GoReleaser archive |
| Air-gapped bundle | `loom-airgap-v0.3.0.tar.gz` | Custom release step |

### Multi-Architecture CI

ARM64 tests run nightly using GitHub's `ubuntu-24.04-arm` runners:

```yaml
test-arm64:
  runs-on: ubuntu-24.04-arm
  steps:
    - run: go test -race ./...
    - run: go test -race -tags=integration ./...
```

---

## 9. LLM Autonomy Framework (Gap 11)

### Decision

LOOM implements a five-level autonomy model. Each tenant starts at Level 1. Promotion to higher levels requires explicit opt-in and is based on measured accuracy, not time. Every decision is audited with its autonomy level.

### Autonomy Levels

| Level | Name | Behavior | Promotion Criteria |
|-------|------|----------|-------------------|
| 0 | Deterministic | LLM disabled. All decisions use deterministic fallback. | N/A (operator override) |
| 1 | Suggest All | LLM suggests. Human approves everything. | Default for new tenants. |
| 2 | Auto-Routine | LLM auto-approves low-risk operations (classification, read-only). Human approves medium/high-risk. | LLM agreement rate >95% over 100+ decisions at Level 1. |
| 3 | Auto-Standard | LLM auto-approves low and medium-risk. Human approves high-risk only (production changes, teardowns). | LLM agreement rate >98% over 500+ decisions at Level 2. |
| 4 | Full Auto | LLM decides everything. Circuit breaker halts on error spike. | Explicit opt-in by platform admin. Not recommended. |

```go
// AutonomyPolicy defines the LLM autonomy level for a tenant.
type AutonomyPolicy struct {
    TenantID           string `json:"tenant_id"`
    Level              int    `json:"level"`                // 0-4
    PromotedAt         *time.Time `json:"promoted_at"`      // when last promoted
    TotalDecisions     int    `json:"total_decisions"`       // decisions at current level
    AgreementRate      float64 `json:"agreement_rate"`       // human-agrees-with-LLM rate
    CircuitBreakerOpen bool   `json:"circuit_breaker_open"`  // true = fallback to deterministic
}
```

### Risk Classification

| Risk Level | Examples | Auto-Approve at Level |
|------------|---------|----------------------|
| Low | Device classification, read-only queries, cost estimation | 2+ |
| Medium | VLAN creation, firewall rule addition, non-production provisioning | 3+ |
| High | Production config change, power cycle, teardown, credential rotation | 4 only (or never) |

### Approval Analytics

Every decision is recorded:

```go
// DecisionRecord captures an LLM decision and its outcome.
type DecisionRecord struct {
    ID              string    `json:"id"`
    TenantID        string    `json:"tenant_id"`
    CorrelationID   string    `json:"correlation_id"`
    TaskType        string    `json:"task_type"`
    RiskLevel       string    `json:"risk_level"`
    AutonomyLevel   int       `json:"autonomy_level"`
    LLMRecommendation any    `json:"llm_recommendation"`
    LLMConfidence   float64   `json:"llm_confidence"`
    HumanDecision   *string   `json:"human_decision"`     // "approved", "rejected", "modified", nil if auto-approved
    AutoApproved    bool      `json:"auto_approved"`
    FallbackUsed    bool      `json:"fallback_used"`
    Outcome         string    `json:"outcome"`             // "success", "failed", "rolled_back"
    CreatedAt       time.Time `json:"created_at"`
}
```

The approval analytics dashboard shows:
- Agreement rate over time (human agrees with LLM).
- Override rate by task type (where does the LLM underperform?).
- Auto-approved success rate (are auto-approvals safe?).
- Fallback frequency (is the LLM available?).

### Trust Calibration Loop

```
LLM decisions → human feedback → agreement rate → autonomy level adjustment
                                                         ↓
                                                 if error rate spikes → circuit breaker → drop to Level 0
                                                         ↓
                                                 after cool-down (1 hour) → restore to previous level
```

Circuit breaker triggers: if >3 LLM-decided operations fail within a 15-minute window, drop to Level 0 for the tenant. After a 1-hour cool-down, restore to the previous level. If it triggers again within 24 hours, stay at Level 0 until manually reset.

---

## 10. Documentation & Onboarding (Gap 12)

### Decision

The primary audience is operators who deploy and use LOOM. The secondary audience is developers who extend LOOM with adapters. Four new documents are required before Phase 1A coding begins: Quickstart Guide, Operator Manual, Adapter Development Guide, and Troubleshooting Guide.

### Quickstart Guide (`docs/QUICKSTART.md`)

Target: zero to discovered device in 5 minutes.

1. Prerequisites: PostgreSQL 16+, Go 1.22+ (or download binary).
2. Start LOOM: `./loom serve --db postgres://... --nats-embedded --temporal-embedded`.
3. Create a tenant: `curl -X POST .../api/v1/tenants`.
4. Add a target: `curl -X POST .../api/v1/targets -d '{"address":"10.0.1.5","protocol_hint":"redfish"}'`.
5. Run discovery: `curl -X POST .../api/v1/discovery/scans -d '{"target_ids":["..."]}'`.
6. See your device: `curl .../api/v1/devices`.

### Operator Manual (`docs/OPERATOR-MANUAL.md`)

Covers: deployment modes, configuration reference, backup/restore, upgrade procedure, monitoring setup, troubleshooting common issues, tenant management, credential management.

### Adapter Development Guide (`docs/ADAPTER-DEVELOPMENT.md`)

Covers: scaffold a new adapter, implement the five interfaces, run the conformance suite, test against a mock device, submit a PR. Includes a 200-line SNMP adapter walkthrough.

### Troubleshooting Guide (`docs/TROUBLESHOOTING.md`)

The five runbooks from Section 2 plus common operational issues (PostgreSQL connection exhaustion, NATS consumer lag, Temporal task queue backlog, LLM provider timeout).

### Concept Guide Navigation

Add a "reading order" section to the existing README.md:

| If You Want To... | Read |
|-------------------|------|
| Try LOOM in 5 minutes | [QUICKSTART.md](docs/QUICKSTART.md) |
| Understand the architecture | [ARCHITECTURE.md](docs/ARCHITECTURE.md) |
| Deploy in production | [OPERATOR-MANUAL.md](docs/OPERATOR-MANUAL.md) |
| Write an adapter | [ADAPTER-DEVELOPMENT.md](docs/ADAPTER-DEVELOPMENT.md) |
| Understand design decisions | [ADR Index](docs/adr/) |
| Debug a problem | [TROUBLESHOOTING.md](docs/TROUBLESHOOTING.md) |

---

## 11. Remaining Gaps (14-27)

### Gap 14: Telemetry Schema and Ingestion Pipeline

**Decision**: Device telemetry (high-frequency time-series metrics) is stored in TimescaleDB hypertables, separate from observed facts (low-frequency point-in-time observations). The dividing line: if it changes more than once per hour, it is telemetry; otherwise, it is a fact.

```sql
CREATE TABLE device_metrics (
    time        TIMESTAMPTZ NOT NULL,
    device_id   UUID NOT NULL,
    tenant_id   UUID NOT NULL,
    metric_name TEXT NOT NULL,         -- e.g., "cpu_utilization", "power_draw_watts", "temperature_celsius"
    value       DOUBLE PRECISION NOT NULL,
    tags        JSONB DEFAULT '{}'     -- e.g., {"sensor": "inlet", "unit": "celsius"}
);

SELECT create_hypertable('device_metrics', by_range('time'));
CREATE INDEX idx_device_metrics_device ON device_metrics (device_id, time DESC);
CREATE INDEX idx_device_metrics_tenant ON device_metrics (tenant_id, time DESC);
```

Retention: 90 days at full resolution. Continuous aggregates for hourly and daily rollups retained for 2 years. Storage estimate at 1-minute resolution with 20 metrics per device: 100 devices = ~5 GB/90 days; 1,000 devices = ~50 GB/90 days; 10,000 devices = ~500 GB/90 days.

### Gap 15: Apache AGE Graph Schema

**Decision**: AGE stores the topology graph with defined node and edge labels. Five critical graph queries are defined. If AGE performance is insufficient at 10K nodes, the fallback is recursive CTEs on the `relationships` SQL table -- not a migration to Neo4j.

Node labels: `Device`, `Switch`, `VLAN`, `Subnet`, `Site`, `Rack`. Edge labels: `connected_to` (physical), `member_of` (VLAN/subnet membership), `depends_on` (service dependency), `located_in` (physical location), `runs_on` (compute placement).

Critical queries:
1. **Blast radius**: `MATCH (d:Device {id: $id})-[*1..4]-(affected) RETURN affected` -- all resources within 4 hops.
2. **Dependency chain**: `MATCH p = (d:Device {id: $id})-[:depends_on*]->(dep) RETURN p` -- full dependency tree.
3. **Shortest path**: `MATCH p = shortestPath((a:Device {id: $a})-[*]-(b:Device {id: $b})) RETURN p`.
4. **Cluster identification**: `MATCH (d:Device)-[:located_in]->(r:Rack)-[:located_in]->(s:Site) WHERE s.id = $site RETURN d, r`.
5. **Topology diff**: Compare two point-in-time snapshots of edges to detect changes.

### Gap 16: Credential Rotation and Lifecycle

**Decision**: LOOM implements a `loom credential rotate` workflow. Credentials have a health check that runs on first use and optionally on a schedule. Edge agent credential caching has a 24-hour maximum TTL with refresh-on-reconnect.

Rotation workflow: update vault entry -> test connectivity on all endpoints using the new credential -> if all pass, commit -> if any fail, roll back to old credential and alert. Rotation emits `credential.rotated` or `credential.rotation_failed` events.

Health check: when an adapter connects using a credential, the result (success/failure) is recorded. If a credential fails authentication 3 times consecutively across any endpoint, the credential is flagged as `status: failed` and an alert is emitted. Periodic health check interval is configurable per credential (default: disabled; recommended: daily for production).

Edge agent credential cache: credentials cached in SQLite have a `cached_at` timestamp. If `now() - cached_at > 24h`, the agent must refresh from the hub before using the credential. On reconnection after a network partition, the agent always refreshes all cached credentials.

### Gap 17: Disaster Recovery Procedure

**Decision**: LOOM defines RTO/RPO targets per deployment mode and a concrete backup strategy. The DR procedure is documented in `docs/DISASTER-RECOVERY.md`.

| Deployment Mode | RTO | RPO | Backup Method |
|-----------------|-----|-----|---------------|
| Mode 1 (single binary) | 1 hour | 1 hour | PostgreSQL `pg_dump` daily + WAL archive hourly |
| Mode 2 (distributed) | 15 minutes | 0 (streaming replication) | PostgreSQL streaming replication + NATS JetStream replication |
| Mode 3 (Kubernetes) | 15 minutes | 0 | PersistentVolumeClaim snapshots + PostgreSQL HA (Patroni) |
| Mode 4 (hub-spoke) | 30 minutes (hub), edge survives independently | 5 minutes | Hub: same as Mode 2. Edge: local SQLite is self-contained. |

Backup targets: PostgreSQL (full daily + continuous WAL archiving), NATS JetStream (replicated storage, no separate backup needed in Mode 2+), Valkey (ephemeral cache, no backup needed), configuration files (`loom.yaml`), vault keyring (encrypted backup to object storage).

Recovery order: PostgreSQL -> Temporal (implicit, uses PostgreSQL) -> NATS JetStream -> Valkey (cold start from empty) -> LOOM API server -> verify with `loom admin health --verbose`.

### Gap 18: Temporal Workflow Versioning

**Decision**: All workflow code changes must use `workflow.GetVersion()`. This is a mandatory code review requirement enforced by a CI lint check.

```go
// Every workflow code change must be guarded by GetVersion.
v := workflow.GetVersion(ctx, "add-dns-verification-step", workflow.DefaultVersion, 1)
if v == 1 {
    // New behavior: includes DNS verification step
    err = workflow.ExecuteActivity(ctx, verifyDNS, input).Get(ctx, &result)
} else {
    // Old behavior: no DNS verification
}
```

CI lint rule: any PR that modifies files in `workflow/` must include a `GetVersion` call in the diff, or include `[NO-VERSION]` in the PR description with justification (e.g., "bugfix to activity that does not change workflow determinism").

Testing pattern: the upgrade test (Section 8) starts workflows on `v(N-1)`, upgrades to `v(N)`, and verifies that in-flight workflows complete successfully.

### Gap 19: Feature Flag Infrastructure

**Decision**: Use OpenFeature SDK with a built-in PostgreSQL provider. No third-party service dependency. Feature flags are per-tenant, audited, and have a mandatory cleanup process.

```sql
CREATE TABLE feature_flags (
    id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name        TEXT NOT NULL UNIQUE,          -- e.g., "llm-cost-prediction"
    description TEXT,
    enabled     BOOLEAN NOT NULL DEFAULT false, -- global default
    tenant_overrides JSONB DEFAULT '{}',       -- {"tenant-abc": true, "tenant-xyz": false}
    created_at  TIMESTAMPTZ NOT NULL DEFAULT now(),
    expires_at  TIMESTAMPTZ,                   -- flags must have an expiry (max 90 days)
    created_by  TEXT NOT NULL
);
```

Flags are evaluated per-request via `featureflag.Enabled("name", tenantID)`. Evaluation checks tenant override first, then global default. Flags with `expires_at` in the past are treated as globally disabled (stale flag cleanup). A weekly CI job lists flags past expiry and creates a GitHub issue for cleanup.

Temporal workflow interaction: flags are evaluated once at workflow start time and passed as a workflow parameter. They are not re-evaluated mid-workflow, avoiding non-determinism.

API endpoints:

| Method | Path | Auth |
|--------|------|------|
| `GET` | `/admin/feature-flags` | `platform_admin` |
| `POST` | `/admin/feature-flags` | `platform_admin` |
| `PATCH` | `/admin/feature-flags/{name}` | `platform_admin` |
| `DELETE` | `/admin/feature-flags/{name}` | `platform_admin` |

### Gap 20: Cost Model Data Source Bootstrapping

**Decision**: LOOM provides sensible defaults, a bulk import API, and integration hooks for DCIM tools. An empty cost model still functions -- it just cannot compare substrates.

Sensible defaults by device type:

| Device Type | Default Monthly Cost | Assumptions |
|------------|---------------------|-------------|
| Server (1U) | $350 | $12K amortized/60mo + 500W power + 1U rack space |
| Server (2U) | $500 | $18K amortized/60mo + 750W power + 2U rack space |
| Switch (1U) | $250 | $8K amortized/60mo + 200W power + 1U rack space |
| Storage (4U) | $1,200 | $30K amortized/48mo + 600W power + 4U rack space |

Defaults use PUE 1.5 and $0.10/kWh. Operators override these via `POST /api/v1/cost/profiles` or bulk import.

Bulk import format: CSV with columns `name, substrate_type, provider, location, dimension, unit_cost, unit, quantity, currency`. Example:

```csv
name,substrate_type,provider,location,dimension,unit_cost,unit,quantity,currency
us-east-rack-12,bare_metal,self,dc-east,compute,200.00,per_month,1,USD
us-east-rack-12,bare_metal,self,dc-east,power,36.50,per_month,500,USD
```

DCIM integration: LOOM can pull cost-relevant data from NetBox via its REST API. A `loom admin import netbox --url=<netbox-url> --token=<token>` command fetches devices, racks, and sites and creates corresponding cost profiles using default cost assumptions.

### Gap 21: NATS Subject Hierarchy and Tenant ID

**Decision**: Keep `loom.{tenant_id}.{domain}.{action}` as the subject hierarchy. Use short tenant aliases (not UUIDs) in subjects. NATS accounts provide tenant isolation.

Tenant ID in subjects uses a short alias (e.g., `acme`, `globex`) derived from the tenant's slug, not the UUID. The UUID remains in the event payload. The `_system` tenant ID is reserved for system events.

NATS authorization model: one NATS account per tenant, configured via NATS account resolver. Each tenant account has publish/subscribe permissions restricted to `loom.{tenant_alias}.>`. System consumers use a system account with `loom.>` access.

```
# NATS account configuration (generated by LOOM)
accounts {
  SYS {
    users: [{ user: "loom-system", password: "..." }]
  }
  ACME {
    users: [{ user: "tenant-acme", password: "..." }]
    exports: [{ stream: "loom.acme.>" }]
  }
  GLOBEX {
    users: [{ user: "tenant-globex", password: "..." }]
    exports: [{ stream: "loom.globex.>" }]
  }
}
```

### Gap 22: PostgreSQL Extension Maturity

**Decision**: Target PostgreSQL 16. Pin specific extension versions. Test extension coexistence in CI. Plan for PostgreSQL 17 migration in Phase 10+.

Extension version matrix:

| Extension | Version | Purpose |
|-----------|---------|---------|
| TimescaleDB | 2.14+ | Device telemetry hypertables |
| Apache AGE | 1.5+ | Topology graph queries |
| pgvector | 0.7+ | LLM semantic cache embeddings |

CI uses the `timescale/timescaledb-ha:pg16` image which bundles TimescaleDB. AGE and pgvector are installed via `CREATE EXTENSION` in the migration. A nightly CI job tests all three extensions under concurrent load to detect memory/WAL conflicts.

PostgreSQL major version upgrade procedure: test all extensions on the new version in CI -> upgrade staging -> run full E2E suite -> upgrade production. This is a manual, planned operation, not automated.

### Gap 23: WebSocket Contract

**Decision**: WebSocket is used only for device terminal access. SSE handles all other real-time updates. The WebSocket contract is defined in Section 7 above.

SSE is sufficient for MVP real-time updates because all real-time needs are unidirectional (server pushes state changes to client). WebSocket is reserved for terminal access (bidirectional byte stream) which is Phase 9+ functionality.

### Gap 24: Shared Device Ownership

**Decision**: Infrastructure tenant model. See Section 4 (Multi-Tenancy) above for the `SharedDeviceMapping` type and access control model.

### Gap 25: Chaos Engineering and Resilience Testing

**Decision**: LOOM uses Toxiproxy for network fault injection in CI. Game days are quarterly. Chaos tests run nightly, not on every PR.

Chaos test infrastructure:

```yaml
# docker-compose.chaos.yml
services:
  toxiproxy:
    image: ghcr.io/shopify/toxiproxy:latest
    ports:
      - "8474:8474"
```

Automated chaos scenarios (nightly):

| Scenario | Method | Verifies |
|----------|--------|----------|
| PostgreSQL disconnect mid-workflow | Toxiproxy: `downstream_close` on PostgreSQL port | Workflow pauses, resumes on reconnect |
| NATS partition (5 seconds) | Toxiproxy: `timeout` on NATS port | Events queued locally, delivered after partition heals |
| Adapter timeout (30 seconds) | Toxiproxy: `latency` on mock device port | Adapter reports `TransientError`, retry succeeds |
| Temporal restart mid-activity | `docker restart temporal` | Activity retried on new Temporal server |
| Valkey unavailable | `docker stop valkey` | API falls back to PostgreSQL reads |

Game day procedure (quarterly): SRE team manually introduces failures in a staging environment following a structured scenario. Results documented in a post-game-day report. Failure modes that are not handled correctly are filed as bugs.

### Gap 26: Concurrent Discovery Race Conditions

**Decision**: Database-level uniqueness constraints prevent duplicate device creation. The identity matcher uses a retry-and-merge loop on conflict. Discovery scans acquire a per-subnet advisory lock to prevent concurrent scans of the same address range.

```sql
-- Unique constraint on serial number per tenant (prevents duplicate devices)
CREATE UNIQUE INDEX idx_devices_serial_tenant
  ON devices (serial_number, tenant_id)
  WHERE serial_number IS NOT NULL AND serial_number != '';
```

When a discovery workflow attempts to insert a device and hits the unique constraint, it:
1. Fetches the existing device by serial number.
2. Runs the identity matcher merge logic (enrich existing device with new facts).
3. Emits a `device.enriched` event instead of `device.created`.

Per-subnet advisory lock:

```go
// AcquireDiscoveryLock acquires a PostgreSQL advisory lock for a subnet.
// Only one discovery scan can run per subnet per tenant at a time.
func AcquireDiscoveryLock(ctx context.Context, tx pgx.Tx, tenantID, subnet string) error {
    lockID := hash64(tenantID + ":" + subnet)
    _, err := tx.Exec(ctx, "SELECT pg_try_advisory_xact_lock($1)", lockID)
    return err
}
```

If the lock is already held, the second scan returns immediately with a `409 Conflict` response: "A discovery scan is already running for subnet 10.0.1.0/24."

### Gap 27: Rate Limiting Per Operation Type

**Decision**: Rate limits are tiered by operation type (read, write, bulk, workflow). Concurrency limits are separate from rate limits. Both are configurable per tenant.

Rate limit tiers are defined in Section 1 (API Route Table) above. Additionally, concurrency limits:

```go
// ConcurrencyLimits are enforced separately from rate limits.
type ConcurrencyLimits struct {
    MaxConcurrentWorkflows          int `json:"max_concurrent_workflows"`           // default: 50
    MaxConcurrentAdapterConnections int `json:"max_concurrent_adapter_connections"` // default: 100
    MaxConcurrentDiscoveryScans     int `json:"max_concurrent_discovery_scans"`     // default: 5
    MaxConcurrentWebSockets         int `json:"max_concurrent_websockets"`          // default: 50
}
```

Concurrency limits are enforced via Valkey atomic counters. When a workflow starts, increment `tenant:{id}:concurrent:workflows`. When it completes, decrement. If the counter exceeds the limit, return `429` with `Retry-After`.

Cost-weighted rate limiting is deferred to post-MVP. For MVP, the four rate limit tiers (read/write/bulk/workflow) provide sufficient differentiation. The `workflow` tier inherently captures expensive operations (discovery, provisioning) and has the lowest limit.

---

## Appendix: Gap-to-Section Cross-Reference

| Gap # | Title | Section |
|-------|-------|---------|
| 3 | REST API Design Completeness | Section 1 |
| 4 | Observability Self-Debugging | Section 2 |
| 5 | LLM Integration Practicalities | Section 3 |
| 6 | Multi-Tenancy Application Layer | Section 4 |
| 7 | Testing Strategy for 25+ Adapters | Section 5 |
| 8 | Event Sourcing / CQRS Implications | Section 6 |
| 9 | UI Architecture Gaps | Section 7 |
| 10 | CI/CD for LOOM Itself | Section 8 |
| 11 | LLM Boundary Pressure | Section 9 |
| 12 | Documentation and Onboarding | Section 10 |
| 13 | Backward Compatibility and Upgrade Paths | Sections 6 (schema versioning), 8 (upgrade testing, compat gate), 11 (Gap 18: Temporal versioning) |
| 14 | Telemetry Schema | Section 11 (Gap 14) |
| 15 | Apache AGE Usage | Section 11 (Gap 15) |
| 16 | Credential Rotation | Section 11 (Gap 16) |
| 17 | Disaster Recovery | Section 11 (Gap 17) |
| 18 | Temporal Versioning | Section 11 (Gap 18) |
| 19 | Feature Flags | Section 11 (Gap 19) |
| 20 | Cost Bootstrapping | Section 11 (Gap 20) |
| 21 | NATS Subject Hierarchy | Section 11 (Gap 21) |
| 22 | PG Extension Maturity | Section 11 (Gap 22) |
| 23 | WebSocket Contract | Section 11 (Gap 23) |
| 24 | Shared Device Ownership | Section 11 (Gap 24) |
| 25 | Chaos Testing | Section 11 (Gap 25) |
| 26 | Discovery Race Conditions | Section 11 (Gap 26) |
| 27 | Rate Limiting Per Operation | Section 11 (Gap 27) |

---

## Reference Documents

| Document | Relationship |
|----------|-------------|
| [APPLICATION-LAYER-GAP-ANALYSIS.md](APPLICATION-LAYER-GAP-ANALYSIS.md) | The gaps this spec closes |
| [API-CONVENTIONS.md](API-CONVENTIONS.md) | Pagination, error format, headers (this spec extends) |
| [API-DOCUMENTATION.md](API-DOCUMENTATION.md) | OpenAPI generation strategy (this spec provides the route table) |
| [OBSERVABILITY-STRATEGY.md](OBSERVABILITY-STRATEGY.md) | Metrics, tracing, logging (this spec adds runbooks and dashboards) |
| [TESTING-STRATEGY.md](TESTING-STRATEGY.md) | Test categories (this spec adds conformance suite and chaos tests) |
| [CI-CD-PIPELINE.md](CI-CD-PIPELINE.md) | CI structure (this spec adds migration and upgrade testing) |
| [EVENT-MODEL.md](EVENT-MODEL.md) | Event envelope (this spec adds schema versioning) |
| [COST-MODEL.md](COST-MODEL.md) | Cost types (this spec adds bootstrapping and defaults) |
| [LLM-BOUNDARIES.md](LLM-BOUNDARIES.md) | LLM rules (this spec adds latency/cost/context budgets) |
| [ARCHITECTURE.md](ARCHITECTURE.md) | System architecture (this spec adds operational specifics) |
| [DOMAIN-MODEL.md](DOMAIN-MODEL.md) | Resource types (this spec adds TenantConfig, SharedDeviceMapping) |
| [WORKFLOW-CONTRACT.md](WORKFLOW-CONTRACT.md) | Workflow rules (this spec adds GetVersion requirement) |
