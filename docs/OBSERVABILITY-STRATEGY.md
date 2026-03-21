# Observability Strategy

> Metrics, tracing, logging, dashboards, alerting, and self-monitoring for LOOM.

---

## 1. Metrics

### Exposition Format

LOOM exposes metrics in Prometheus exposition format at `/metrics` on the API server port. All metrics use the `loom_` prefix.

### Key Metrics

#### Device Management

| Metric | Type | Description |
|--------|------|-------------|
| `loom_devices_total` | Gauge | Total devices by `{tenant_id, type, status}` |
| `loom_device_discovery_duration_seconds` | Histogram | Time to discover a single device by `{protocol}` |
| `loom_device_connections_active` | Gauge | Currently open adapter connections by `{protocol}` |
| `loom_device_connections_total` | Counter | Total adapter connections attempted by `{protocol, result}` |
| `loom_device_state_changes_total` | Counter | Device state transitions by `{from_status, to_status}` |

#### Adapter Performance

| Metric | Type | Description |
|--------|------|-------------|
| `loom_adapter_operation_duration_seconds` | Histogram | Adapter operation latency by `{adapter, operation}` |
| `loom_adapter_errors_total` | Counter | Adapter errors by `{adapter, error_type}` (`transient`, `permanent`, `partial`, `validation`) |
| `loom_adapter_retries_total` | Counter | Retry attempts by `{adapter, operation}` |
| `loom_adapter_idempotency_cache_hits_total` | Counter | Idempotency cache hits by `{adapter}` |

#### Workflow Execution

| Metric | Type | Description |
|--------|------|-------------|
| `loom_workflow_started_total` | Counter | Workflows started by `{workflow_type, tenant_id}` |
| `loom_workflow_completed_total` | Counter | Workflows completed by `{workflow_type, result}` (`success`, `failed`, `cancelled`) |
| `loom_workflow_duration_seconds` | Histogram | End-to-end workflow duration by `{workflow_type}` |
| `loom_workflow_active` | Gauge | Currently running workflows by `{workflow_type}` |
| `loom_workflow_approval_wait_seconds` | Histogram | Time spent waiting for human approval by `{risk_level}` |
| `loom_workflow_compensation_total` | Counter | Saga compensations triggered by `{workflow_type, step}` |

#### LLM Decision Engine

| Metric | Type | Description |
|--------|------|-------------|
| `loom_llm_decision_duration_seconds` | Histogram | LLM decision latency by `{provider, model, task_type}` |
| `loom_llm_decisions_total` | Counter | LLM decisions by `{provider, model, task_type, result}` |
| `loom_llm_tokens_total` | Counter | Tokens consumed by `{provider, model, direction}` (`input`, `output`) |
| `loom_llm_cost_dollars` | Counter | Estimated LLM cost in dollars by `{provider, model}` |
| `loom_llm_cache_hits_total` | Counter | Semantic cache hits (decisions reused without LLM call) |
| `loom_llm_fallback_total` | Counter | Provider fallbacks by `{from_provider, to_provider, reason}` |
| `loom_llm_validation_failures_total` | Counter | LLM output validation failures by `{task_type, failure_reason}` |

#### Verification

| Metric | Type | Description |
|--------|------|-------------|
| `loom_verification_checks_total` | Counter | Verification checks by `{check_type, result}` (`passed`, `failed`) |
| `loom_verification_duration_seconds` | Histogram | Verification check duration by `{check_type}` |
| `loom_drift_detections_total` | Counter | Configuration drift events by `{resource_type, tenant_id}` |

#### API

| Metric | Type | Description |
|--------|------|-------------|
| `loom_http_requests_total` | Counter | HTTP requests by `{method, path, status_code}` |
| `loom_http_request_duration_seconds` | Histogram | HTTP request latency by `{method, path}` |
| `loom_http_request_size_bytes` | Histogram | Request body size |
| `loom_http_response_size_bytes` | Histogram | Response body size |
| `loom_ratelimit_rejected_total` | Counter | Rate-limited requests by `{tenant_id}` |

#### Security

| Metric | Type | Description |
|--------|------|-------------|
| `loom_auth_failures_total` | Counter | Authentication failures by `{source, reason, tenant_id}` |
| `loom_authz_denials_total` | Counter | Authorization denials by `{tenant_id, resource, action}` |
| `loom_credential_access_total` | Counter | Credential store access by `{tenant_id, credential_ref, actor}` |
| `loom_tenant_boundary_violations_total` | Counter | Cross-tenant access attempts by `{source_tenant, target_tenant, resource}` |
| `loom_adapter_anomaly_score` | Gauge | Behavioral anomaly score by `{adapter_name, device_id}` |
| `loom_break_glass_used_total` | Counter | Break-glass escalations by `{tenant_id, approver_count}` |
| `loom_certificate_expiry_days` | Gauge | Days until certificate expiry by `{component, serial}` |
| `loom_lock_contention_total` | Counter | Lock contention events by `{device_id, tenant_id}` |

---

## 2. Distributed Tracing

### OpenTelemetry Integration

LOOM uses OpenTelemetry for distributed tracing. Traces follow requests across all system boundaries:

```
API Request
  └─ HTTP Handler (span)
      └─ Temporal Workflow Start (span)
          └─ Activity: AllocateResources (span)
          └─ Activity: ConfigureNetwork (span)
              └─ Adapter: NETCONF Push (span)
                  └─ Device: Switch 10.0.1.1 (span)
          └─ Activity: VerifyProvisioning (span)
              └─ Adapter: SNMP Poll (span)
              └─ Adapter: SSH Command (span)
```

### Trace Propagation

| Boundary | Propagation Method |
|----------|-------------------|
| HTTP API → Temporal | `traceparent` header in workflow input metadata |
| Temporal → Activity | Temporal's built-in context propagation |
| Activity → Adapter | Go `context.Context` carries span context |
| LOOM Hub → Edge Agent | `traceparent` in NATS message headers |
| LOOM → LLM Provider | **No propagation.** `traceparent` is stripped from outbound LLM API calls. A new span is created at the LLM boundary carrying only `request_type` and `model` attributes. This prevents leaking internal trace topology to third-party LLM providers. |

### Span Attributes

All spans include:

| Attribute | Description |
|-----------|-------------|
| `loom.tenant_id` | Tenant that owns the request |
| `loom.correlation_id` | Cross-system correlation ID (from `X-Correlation-ID` header) |
| `loom.workflow_id` | Temporal workflow ID (when applicable) |
| `loom.device_id` | Target device ID (when applicable) |
| `loom.adapter` | Adapter name (when applicable) |
| `loom.protocol` | Protocol used (when applicable) |

### Trace Export

Traces are exported via OTLP (OpenTelemetry Protocol) to any compatible backend:

```bash
# Configure trace export
OTEL_EXPORTER_OTLP_ENDPOINT=http://localhost:4317   # Jaeger, Tempo, etc.
OTEL_SERVICE_NAME=loom
OTEL_RESOURCE_ATTRIBUTES=deployment.environment=production
```

Supported backends: Jaeger, Grafana Tempo, Zipkin, any OTLP-compatible collector.

---

## 3. Logging

### Structured JSON Logging

LOOM uses Go's `slog` package for structured logging. All logs are JSON-formatted in production:

```json
{
  "time": "2026-03-20T10:00:00.123Z",
  "level": "INFO",
  "msg": "device discovered",
  "correlation_id": "req-abc-123",
  "tenant_id": "tenant-xyz",
  "device_id": "dev-456",
  "adapter": "redfish",
  "protocol": "redfish",
  "vendor": "Dell",
  "model": "PowerEdge R750",
  "duration_ms": 342
}
```

### Log Levels

| Level | Usage |
|-------|-------|
| `DEBUG` | Adapter protocol details, SQL queries, NATS message payloads. Never in production by default. |
| `INFO` | Workflow start/complete, device state changes, adapter connect/disconnect. |
| `WARN` | Transient errors (will be retried), approaching rate limits, replication lag, deprecated config keys. |
| `ERROR` | Permanent failures, unrecoverable adapter errors, verification failures, security violations. |

### Correlation IDs

Every log entry includes `correlation_id`. This is the same ID that appears in:
- The HTTP request header (`X-Correlation-ID`).
- The Temporal workflow metadata.
- The NATS event envelope.
- The adapter error struct (`CorrelationID` field).

This allows operators to trace a single request across all log sources with one query:

```bash
# Find all logs for a specific request
grep '"correlation_id":"req-abc-123"' /var/log/loom/*.log
```

### Tenant-Scoped Log Filtering

All logs include `tenant_id`. In multi-tenant deployments, operators can filter logs per tenant:

```bash
# Show only Tenant A's activity
jq 'select(.tenant_id == "tenant-a")' /var/log/loom/loom.log
```

For deployments where tenants must not see each other's log data, LOOM supports per-tenant log routing via slog handlers that split output to tenant-specific log files or streams.

### Sensitive Data Scrubbing — Allowlist Model

Log field scrubbing uses an **allowlist**, not a blocklist. Only explicitly permitted fields are emitted. Any field not on the allowlist is logged as `[FIELD:redacted]`. This ensures that new fields added to structs are safe by default — they must be explicitly approved for logging.

**Allowed log fields:**

| Field | Justification |
|-------|---------------|
| `time` | Timestamp — no sensitive data |
| `level` | Log level — no sensitive data |
| `msg` | Free-text message — reviewed per call site |
| `correlation_id` | Opaque request ID — no sensitive data |
| `tenant_id` | Tenant identifier — required for filtering |
| `device_id` | Device identifier — required for debugging |
| `adapter` | Adapter name — no sensitive data |
| `protocol` | Protocol name — no sensitive data |
| `duration_ms` | Latency — no sensitive data |
| `status_code` | HTTP status — no sensitive data |
| `error` | Error message — scrubbed separately to strip credential fragments |
| `workflow_id` | Temporal workflow ID — required for debugging |
| `method` | HTTP method — no sensitive data |
| `path` | Request path — no sensitive data |
| `component` | Health check component name — no sensitive data |

**Explicitly blocked (never logged regardless of allowlist):**
- Credential values (passwords, tokens, private keys, SNMP community strings).
- JWT token bodies (only the `sub` and `tenant_id` claims are logged).
- Device configuration secrets.
- LLM prompt content that includes credentials.
- Request/response bodies.

Scrubbing is enforced at the `slog.Handler` level, not at individual call sites. The handler wraps every `slog.Record` and drops or redacts any attribute whose key is not in the allowlist.

---

## 4. Dashboards

### Grafana Dashboards

LOOM ships pre-built Grafana dashboard JSON files in `deploy/grafana/dashboards/`:

#### Dashboard: LOOM Overview

Target audience: LOOM operators (SRE, infrastructure team).

| Panel | Data Source | Shows |
|-------|------------|-------|
| Device Fleet Health | Prometheus | Devices by status (active/degraded/unreachable), stacked by type |
| Workflow Status | Prometheus | Active/completed/failed workflows over time |
| Adapter Error Rate | Prometheus | Errors per adapter per minute, grouped by error type |
| API Latency | Prometheus | P50/P95/P99 HTTP response times |
| LLM Decision Rate | Prometheus | Decisions per minute by provider, with cost overlay |

#### Dashboard: Workflow Deep Dive

| Panel | Data Source | Shows |
|-------|------------|-------|
| Workflow Duration Distribution | Prometheus | Histogram of workflow completion times by type |
| Approval Wait Times | Prometheus | Time spent in approval gates by risk level |
| Compensation Events | Prometheus | Saga compensations over time |
| Step-Level Latency | Prometheus | Per-step duration breakdown for each workflow type |
| Active Workflow List | Prometheus + API | Table of currently running workflows with status |

#### Dashboard: Cost Analytics

| Panel | Data Source | Shows |
|-------|------------|-------|
| LLM Cost Trend | Prometheus | Daily/weekly LLM spend by provider |
| Cost per Decision | Prometheus | Average cost per LLM decision by task type |
| Cache Savings | Prometheus | Estimated cost saved by semantic caching |
| Provider Distribution | Prometheus | Pie chart of decisions by provider (local vs cloud) |
| Budget Utilization | Prometheus + API | Per-tenant budget usage vs allocation |

#### Dashboard: Infrastructure Health

| Panel | Data Source | Shows |
|-------|------------|-------|
| PostgreSQL Connections | Prometheus (pg_exporter) | Active connections, pool utilization, replication lag |
| NATS JetStream | Prometheus (nats_exporter) | Consumer lag, message rates, storage usage |
| Temporal Queue Depth | Prometheus (temporal metrics) | Task queue depth, schedule-to-start latency |
| Valkey Hit Rate | Prometheus (valkey_exporter) | Cache hit/miss ratio, memory usage |

---

## 5. Alerting

### Alert Rules

Alert rules are defined as Prometheus alerting rules. Each alert maps to a failure mode documented in [FAILURE-MODES.md](FAILURE-MODES.md).

#### Critical Alerts (Page Immediately)

| Alert | Condition | FAILURE-MODES.md Reference |
|-------|-----------|---------------------------|
| `LOOMDatabaseDown` | `up{job="postgresql"} == 0` for 1m | Section 1.1: Primary Down |
| `LOOMTemporalDown` | `up{job="temporal"} == 0` for 1m | Section 2.1: All Temporal Services Down |
| `LOOMNATSDown` | `up{job="nats"} == 0` for 1m | Section 3.1: NATS Server Down |
| `LOOMHighErrorRate` | `rate(loom_http_requests_total{status_code=~"5.."}[5m]) > 0.05` | API failure rate exceeds 5% |
| `LOOMWorkflowCompensationSpike` | `rate(loom_workflow_compensation_total[5m]) > 0.1` | Unusual saga compensation rate |
| `LOOMDiskFull` | `node_filesystem_avail_bytes{mountpoint="/var/lib/postgresql"} / node_filesystem_size_bytes < 0.1` | Section 1.4: Disk Full |

#### Warning Alerts (Investigate During Business Hours)

| Alert | Condition | FAILURE-MODES.md Reference |
|-------|-----------|---------------------------|
| `LOOMReplicationLag` | `pg_replication_lag_seconds > 10` for 5m | Section 1.3: Replication Lag |
| `LOOMPoolSaturation` | `loom_db_pool_waiting > 0` for 5m | Section 1.5: Pool Exhaustion |
| `LOOMNATSConsumerLag` | `nats_consumer_num_pending > 10000` for 10m | Section 3.3: Consumer Lag |
| `LOOMLLMLatencyHigh` | `histogram_quantile(0.95, loom_llm_decision_duration_seconds) > 10` for 5m | LLM provider degradation |
| `LOOMLLMCostExceeded` | `increase(loom_llm_cost_dollars[24h]) > daily_budget` | Cost exceeds daily budget |
| `LOOMDeviceUnreachable` | `loom_devices_total{status="unreachable"} > 0` for 15m | Device connectivity loss |
| `LOOMTemporalQueueDepth` | `temporal_task_queue_depth > 1000` for 5m | Section 2.4: Task Queue Backlog |

#### Info Alerts (Dashboard Annotation Only)

| Alert | Condition |
|-------|-----------|
| `LOOMDeploymentStarted` | Detected via version label change in `loom_build_info` |
| `LOOMLLMProviderFallback` | `rate(loom_llm_fallback_total[5m]) > 0` |
| `LOOMEdgeAgentDisconnected` | Edge agent NATS heartbeat missed for 5m |

#### Security Alerts

| Alert | Condition | Severity |
|-------|-----------|----------|
| `LOOMAuthFailureSpike` | `rate(loom_auth_failures_total[5m]) > 10` | Critical — possible brute force or credential stuffing |
| `LOOMAuthzDenialSpike` | `rate(loom_authz_denials_total[5m]) > 5` | Warning — possible privilege escalation attempt |
| `LOOMTenantBoundaryViolation` | `increase(loom_tenant_boundary_violations_total[1m]) > 0` | Critical — immediate investigation required |
| `LOOMCredentialAccessAnomaly` | `rate(loom_credential_access_total[5m]) > 3 * avg_over_time(rate(loom_credential_access_total[5m])[1h:])` | Warning — unusual credential access pattern |
| `LOOMAdapterAnomalyHigh` | `loom_adapter_anomaly_score > 0.8` | Warning — adapter behavioral anomaly detected |
| `LOOMBreakGlassUsed` | `increase(loom_break_glass_used_total[1m]) > 0` | Critical — break-glass event requires post-incident review |
| `LOOMCertificateExpiringSoon` | `loom_certificate_expiry_days < 30` | Warning — certificate renewal required |
| `LOOMCertificateExpiryCritical` | `loom_certificate_expiry_days < 7` | Critical — certificate expires within 7 days |
| `LOOMLockContentionHigh` | `rate(loom_lock_contention_total[5m]) > 1` | Warning — possible resource contention or DoS |

---

## 6. Self-Monitoring

LOOM monitors its own infrastructure components. This is distinct from monitoring managed devices.

### Health Check Endpoints

Health is split into two endpoints to prevent leaking component topology to unauthenticated callers.

**Unauthenticated — for load balancer probes:**

```
GET /healthz

Response (200 or 503):
{
  "status": "ok"               // "ok" or "degraded" — no component details
}
```

This endpoint returns only aggregate status. It does **not** expose component names, versions, or uptime. Load balancers and external monitors use this endpoint.

**Authenticated — for operators (requires valid JWT):**

```
GET /api/v1/health
Authorization: Bearer <token>

Response:
{
  "status": "healthy",          // "healthy", "degraded", "unhealthy"
  "components": {
    "database": "healthy",
    "nats": "healthy",
    "temporal": "healthy",
    "valkey": "healthy",        // "skipped" if Valkey is not configured
    "llm": "healthy"            // checks default LLM provider reachability
  },
  "version": "v0.3.0",
  "uptime_seconds": 86400
}
```

### Component Health Checks

| Component | Check | Interval |
|-----------|-------|----------|
| PostgreSQL | `SELECT 1` on primary connection | 5s |
| PostgreSQL replication | `pg_stat_replication.replay_lag` | 10s |
| NATS JetStream | Publish/subscribe round-trip on `loom._system.healthcheck` | 5s |
| NATS consumer lag | `nats_consumer_num_pending` for all LOOM consumers | 10s |
| Temporal | `DescribeNamespace` API call | 10s |
| Temporal task queues | `DescribeTaskQueue` for all LOOM task queues | 30s |
| Valkey | `PING` command | 5s |
| LLM provider | Lightweight prompt completion ("respond with OK") | 60s |
| Edge agents | NATS heartbeat presence check | 30s |

### Degraded Mode Behavior

When a component becomes unhealthy, LOOM does not crash. It degrades gracefully as documented in [FAILURE-MODES.md](FAILURE-MODES.md):

| Component Down | LOOM Behavior |
|---------------|--------------|
| PostgreSQL | Rejects new requests (503). Temporal workflows pause. Cached reads from Valkey continue. |
| NATS | Events queued locally. Temporal workflows continue (they use PostgreSQL, not NATS). |
| Temporal | No new workflows. Existing workflows suspend. API reads continue from PostgreSQL. |
| Valkey | Falls back to PostgreSQL for all reads. Higher latency but functional. |
| LLM provider | Falls back to next provider in chain (cloud → local, or vice versa). Deterministic fallback for critical decisions. |

### Internal Event Subjects

Self-monitoring events are published to NATS under the `loom._system` namespace:

```
loom._system.database.connection_lost
loom._system.database.connection_restored
loom._system.database.replication_lag_high
loom._system.database.pool_exhausted
loom._system.nats.consumer_lag_high
loom._system.temporal.queue_depth_high
loom._system.llm.provider_fallback
loom._system.llm.cost_threshold_exceeded
loom._system.edge.agent_disconnected
loom._system.edge.agent_reconnected
```

These events drive the alerting rules above and are also consumed by the UI for real-time operator notifications.
