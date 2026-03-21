# Application-Layer Gap Responses

> **Date:** 2026-03-21
> **Scope:** Responses to all application-layer findings (AL-1 through AL-13) from [APPLICATION-LAYER-GAP-ANALYSIS.md](APPLICATION-LAYER-GAP-ANALYSIS.md)
> **Status:** Accepted decisions — ready for Phase 1A implementation

---

## 1. Adapter SDK / Developer Experience (AL-1, AL-2)

**Findings addressed:** No adapter SDK or developer experience story (AL-1, Critical). Plugin/adapter lifecycle undefined (AL-2, High).

### Adapter SDK

Full design is captured in [ADAPTER-SDK.md](ADAPTER-SDK.md) (being created separately). This section covers the decisions that intersect with the broader application layer.

### Plugin Lifecycle

**Binary versioning:** Every adapter binary follows semver independently from the LOOM hub version. The hub tracks which adapter versions are deployed and performs compatibility checks before assigning workflows to an adapter instance.

**Hot-reload is NOT supported.** Adapter updates require a process restart. Hot-reload is too risky for infrastructure tooling — a partially-loaded adapter executing a power-cycle operation could leave hardware in an undefined state. The operator must explicitly restart the adapter process or roll the deployment.

**Dependency isolation:** Adapters run as separate processes using the HashiCorp go-plugin subprocess model. Each adapter is its own binary communicating with the hub over gRPC. This provides:
- Process-level isolation (adapter crash does not crash the hub)
- Independent deployment (patch one adapter without redeploying LOOM)
- Resource containment (adapter memory/CPU usage is bounded by OS process limits)

**Adapter version matrix:** The hub maintains a registry of deployed adapter versions. Before dispatching a workflow to an adapter, the hub verifies:
1. The adapter version satisfies the minimum version required for the operation
2. The adapter's `MinHubVersion` is compatible with the running hub version
3. The adapter supports the required protocol and operation family

The hub supports the current adapter version plus N-1 (one prior minor version). Adapters older than N-1 are flagged for upgrade but not forcibly disconnected.

### Go Types

```go
// AdapterManifest is returned by every adapter on startup registration.
type AdapterManifest struct {
    Name               string           // e.g., "redfish", "ssh", "netconf-iosxe"
    Version            string           // semver, e.g., "1.3.0"
    SupportedProtocols []string         // e.g., ["redfish-v1", "redfish-v1.1"]
    OperationFamilies  []string         // e.g., ["power", "discovery", "firmware"]
    MinHubVersion      string           // minimum LOOM hub version, e.g., "0.4.0"
}
```

---

## 2. REST API Design (AL-3)

**Finding addressed:** REST API design completeness (AL-3, High). The API conventions document defines pagination, error format, and headers but never specifies the actual endpoints.

### Endpoint Inventory

All endpoints are under `/api/v1/` and follow the conventions in [API-CONVENTIONS.md](API-CONVENTIONS.md).

| Resource | Methods | Description |
|----------|---------|-------------|
| `/api/v1/devices` | GET, POST | List/search devices, create device |
| `/api/v1/devices/{id}` | GET, PATCH, DELETE | Get, update, delete device |
| `/api/v1/devices/{id}/endpoints` | GET, POST | List/create endpoints for device |
| `/api/v1/devices/{id}/endpoints/{eid}` | GET, PATCH, DELETE | Get, update, delete endpoint |
| `/api/v1/devices/{id}/facts` | GET | Read-only observed state (facts are written by adapters, not by API consumers) |
| `/api/v1/workflows` | GET, POST | List workflows, create (submit) workflow |
| `/api/v1/workflows/{id}` | GET | Get workflow status and history |
| `/api/v1/workflows/{id}/cancel` | POST | Cancel a running workflow |
| `/api/v1/workflows/{id}/approve` | POST | Approve a workflow awaiting human approval |
| `/api/v1/tenants` | GET, POST | Admin: list and create tenants |
| `/api/v1/tenants/{id}` | GET, PATCH, DELETE | Admin: get, update, decommission tenant |
| `/api/v1/credentials` | GET, POST | List credential metadata, create credential |
| `/api/v1/credentials/{id}` | DELETE | Delete credential (NO retrieve-value endpoint — credentials are write-only) |
| `/api/v1/discovery` | POST | Trigger a discovery scan |
| `/api/v1/topology` | GET | Read graph data (nodes + edges for visualization) |
| `/api/v1/cost` | GET | Cost profiles, estimates, and budget status |
| `/api/v1/cost/profiles` | GET, POST | CRUD for cost profiles |
| `/api/v1/cost/budgets` | GET, POST, PATCH | Per-tenant budget management |
| `/api/v1/health` | GET | System health (no auth required) |
| `/api/v1/webhooks` | GET, POST | List and register webhooks |
| `/api/v1/webhooks/{id}` | GET, PATCH, DELETE | Manage individual webhooks |

### Bulk Operations

```
POST /api/v1/devices/bulk
```

Request body:
```json
{
  "operations": [
    {"op": "create", "data": {"name": "switch-01", "type": "network_switch", ...}},
    {"op": "update", "ref": "device-uuid-1", "data": {"tags": {"rack": "A3"}}},
    {"op": "delete", "ref": "device-uuid-2"}
  ]
}
```

Constraints:
- Maximum 100 operations per batch
- Operations are executed sequentially within the batch (not atomically — partial failure is possible)
- Response includes per-operation status: `{results: [{op: "create", status: "ok", id: "..."}, {op: "update", status: "error", error: {...}}]}`

### Webhook Design

```
POST /api/v1/webhooks
```

Request body:
```json
{
  "url": "https://ops.example.com/hooks/loom",
  "events": ["device.discovered", "workflow.completed", "workflow.failed", "device.state_changed"],
  "secret": "hmac-secret-for-signature-verification"
}
```

Webhook delivery:
- Payloads are signed with HMAC-SHA256 using the registered secret (in `X-LOOM-Signature-256` header)
- Delivery attempts: 3 retries with exponential backoff (1s, 10s, 60s)
- Webhooks that fail 10 consecutive deliveries are automatically disabled with a notification event

### Cross-References

- Pagination, error envelope, and header conventions: [API-CONVENTIONS.md](API-CONVENTIONS.md)
- OpenAPI spec is generated from Go struct annotations via huma at build time

---

## 3. LLM Integration Practicalities (AL-5)

**Finding addressed:** LLM latency, cost, and context limits (AL-5, High).

### Latency Budget

The original activity timeouts in WORKFLOW-CONTRACT.md were designed for adapter calls, not LLM calls. LLM-specific activities use adjusted timeouts:

| Activity Type | Timeout | Heartbeat Interval |
|---------------|---------|-------------------|
| LLM classification | 120s | 15s |
| LLM placement decision | 120s | 15s |
| LLM config generation | 120s | 15s |
| LLM oversight review | 120s | 15s |

The 120-second timeout (not 30s) accommodates:
- Cloud API calls: 5-15s typical, up to 60s under load
- Local vLLM on CPU: 50-100s for 500-token response
- One retry within the timeout window for cloud providers

### Context Window Management

Hard limits on token budgets per LLM call:
- **Input budget:** Max 8,192 tokens total (system prompt + RAG context + user request)
- **Output budget:** Max 4,096 tokens

For large fleet operations (e.g., 1000-device placement), the context is constructed by:
1. Summarizing fleet state into aggregated groups (e.g., "47 Dell R640 in rack A, 12 free slots")
2. Including only the devices relevant to the decision (filtered by constraints)
3. Truncating with a token counter before submission — never exceeding the budget

### Cost Governance

Per-tenant monthly LLM token budgets are tracked in the cost model:

```go
type LLMBudget struct {
    TenantID         string
    MonthlyTokenLimit int64   // max tokens per calendar month
    TokensUsed       int64   // tokens consumed this month
    CostLimit        float64 // max dollar spend per month (provider-dependent pricing)
}
```

Enforcement:
- Token usage is metered at the API layer before dispatching LLM activities
- When a tenant exceeds 80% of their budget, a warning event is emitted
- When a tenant exceeds 100%, LLM activities are rejected and the fallback chain activates

### Fallback Chain

Each step in the fallback chain is an independent Temporal activity:

1. **Claude API** (or configured cloud provider) — primary
2. **Local vLLM** — used when cloud is unavailable or in air-gapped deployments
3. **Deterministic rule engine** — hardcoded decision tables for common operations (e.g., power-cycle always follows the same steps regardless of LLM)

Fallback triggers:
- LLM timeout (no response within 120s) → retry once with same provider → fall back to next provider
- LLM error (API 5xx, rate limit) → fall back to next provider immediately
- All LLM providers exhausted → deterministic rule engine
- Rule engine cannot handle the request → fail the activity with a human-readable explanation

### Error Handling

Every LLM activity follows this pattern:
1. Call primary LLM provider with timeout
2. On timeout: retry once (same provider)
3. On second failure: fall back to next provider in chain
4. On all providers exhausted: invoke deterministic rule engine
5. On rule engine miss: fail activity with `LLMExhaustedError` containing explanation of what was attempted

The workflow receives the error and can either escalate to human approval or abort with a clear message.

---

## 4. Multi-Tenancy Application Layer (AL-6)

**Finding addressed:** Multi-tenancy gaps above the data layer (AL-6, High).

### Quota Management

Per-tenant resource limits enforced at the API layer:

```go
type TenantQuota struct {
    TenantID            string
    MaxDevices          int    // max devices under management
    MaxWorkflowsPerHour int    // max workflow submissions per hour
    MaxAPICallsPerMin   int    // API rate limit (all operations)
    MaxStorageMB        int64  // max storage (facts, audit, config snapshots)
    MaxLLMTokensPerMonth int64 // LLM token budget (see LLMBudget)
}
```

Resource accounting is metered at the API middleware layer. Every mutating request checks the relevant quota before dispatching the operation. Quota violations return HTTP 429 with a `QuotaExceeded` error containing the specific limit that was hit.

### Tenant Lifecycle

```
create → active → suspended → decommissioned
```

```go
type TenantLifecycle string

const (
    TenantCreated        TenantLifecycle = "created"        // provisioning in progress
    TenantActive         TenantLifecycle = "active"         // fully operational
    TenantSuspended      TenantLifecycle = "suspended"      // no new operations, existing workflows drain
    TenantDecommissioned TenantLifecycle = "decommissioned" // data retained per policy, then purged
)
```

State transitions:
- **Created → Active:** Automatic after onboarding completes
- **Active → Suspended:** Admin action. Running workflows are allowed to complete; new submissions are rejected. Device monitoring continues (read-only).
- **Suspended → Active:** Admin action. Resumes full operation.
- **Active/Suspended → Decommissioned:** Admin action with confirmation. Workflows are cancelled. Data is retained for the configured retention period (default: 90 days), then purged.

### Tenant Configuration

```go
type TenantConfig struct {
    TenantID              string
    LLMModelPreference    string            // e.g., "claude-sonnet", "local-vllm"
    ApprovalPolicy        ApprovalPolicy    // per-operation-type approval requirements
    AutonomyBoundaries    []AutonomyPolicy  // graduated trust per operation type
    CostThresholds        CostThresholds    // alert and block thresholds
    DiscoverySchedule     string            // cron expression for periodic discovery
    NotificationEndpoints []string          // webhook URLs for tenant events
}
```

### Tenant Onboarding

`POST /api/v1/tenants` triggers a Temporal workflow that:

```go
type TenantOnboarding struct {
    TenantID      string
    AdminUserID   string
    AdminEmail    string
    InitialConfig TenantConfig
}
```

Onboarding workflow steps:
1. Create tenant record in PostgreSQL
2. Provision NATS account with tenant-scoped subject permissions
3. Create Temporal namespace for tenant workflows
4. Create initial admin user with `tenant_admin` role
5. Apply default quotas
6. Emit `tenant.created` event
7. (Optional) Run welcome workflow that validates connectivity

---

## 5. Testing Strategy Enhancement (AL-7)

**Finding addressed:** Testing strategy for 25+ protocol adapters (AL-7, High).

### Shared Mock Servers

Instead of each adapter building its own bespoke mock, LOOM maintains one mock server per protocol type. All adapters for a given protocol share that mock:

| Protocol | Mock Server | Used By |
|----------|-------------|---------|
| Redfish | `mock-redfish` | redfish-dell, redfish-hpe, redfish-supermicro |
| SSH | `mock-ssh` | ssh-linux, ssh-sonic, ssh-cumulus |
| SNMP | `mock-snmp` | snmp-v2c, snmp-v3 |
| NETCONF | `mock-netconf` | netconf-iosxe, netconf-iosxr, netconf-junos |
| IPMI | `mock-ipmi` | ipmi-v2 |
| gNMI | `mock-gnmi` | gnmi-sonic, gnmi-arista |

Mock servers are protocol-correct simulators that accept "device profiles" to emulate vendor-specific responses. A Dell iDRAC profile produces Dell-flavored Redfish JSON; an HPE iLO profile produces HPE-flavored responses. Both run on the same `mock-redfish` server.

### Mock Server Registry

```go
type MockRegistry struct {
    // Start creates a mock device for the given protocol and vendor profile.
    // Returns the mock's address for adapter connection.
    Start(protocol string, profile string) (addr string, cleanup func(), err error)

    // StopAll tears down all mock devices started by this registry instance.
    StopAll()
}
```

Test suites use `MockRegistry` to start/stop mock devices as needed. The registry manages port allocation and cleanup.

### Chaos Testing

Chaos scenarios tested in the nightly E2E suite:
- **Adapter connection drop mid-operation:** TCP connection is killed after the adapter sends a command but before receiving a response. Verifies idempotency and retry behavior.
- **NATS partition:** NATS is made unreachable for 30 seconds during a workflow. Verifies event buffering and eventual delivery.
- **Temporal server kill during workflow:** Temporal is restarted while workflows are mid-execution. Verifies workflow replay and recovery.
- **PostgreSQL restart during projection write:** DB is bounced while the projector is processing events. Verifies projector idempotency and catch-up behavior.

### Scale Testing

Nightly CI includes a scale test with 1,000 simulated devices:
- 500 Redfish (mixed Dell/HPE profiles)
- 200 SSH (mixed Linux/SONiC profiles)
- 150 SNMP
- 100 NETCONF
- 50 mixed (IPMI, gNMI, PiKVM)

The scale test verifies:
- Discovery completes within 10 minutes for 1,000 devices
- Concurrent workflows (50 simultaneous) execute without deadlock
- Event throughput sustains 500 events/second
- PostgreSQL projection lag stays under 5 seconds

### Conformance Suite Integration

Every adapter must pass the conformance test suite defined in [ADAPTER-SDK.md](ADAPTER-SDK.md) before it can be registered with the hub. The conformance suite covers:
- All five interface implementations (Connector, Discoverer, Executor, StateReader, Watcher)
- Error classification (all 4 error categories)
- Idempotency contract (replayed operations produce same result)
- Compensation metadata (rollback information is populated)
- Manifest validity (all required fields present)

---

## 6. Documentation and Onboarding (AL-12)

**Finding addressed:** Documentation is architecturally complete but developer-hostile (AL-12, High).

### Quickstart Guide

`docs/quickstart/` contains three tutorials, each designed to complete in under 15 minutes:

1. **"Discover your first device"**
   - Prerequisites: one Linux server or VM accessible via SSH
   - Steps: download LOOM binary, configure SSH credentials, run discovery, see device in inventory
   - Uses Mode 1 (single binary) deployment — no Docker, no dependencies
   - Target time: 5 minutes from download to discovered device

2. **"Run your first workflow"**
   - Prerequisites: completed tutorial 1
   - Steps: submit a power-cycle workflow, approve it, watch it execute, verify device state
   - Introduces the approval model and workflow lifecycle
   - Target time: 10 minutes

3. **"Add a custom adapter"**
   - Prerequisites: Go 1.22+, completed tutorial 1
   - Steps: scaffold adapter using SDK, implement Connector + Discoverer for a dummy HTTP device, run conformance tests, register with hub
   - Cross-references [ADAPTER-SDK.md](ADAPTER-SDK.md) for the full SDK documentation
   - Target time: 15 minutes to passing conformance tests

### Concept Guide

`docs/concepts/` explains LOOM's mental model for newcomers:

- **Devices and Endpoints:** A device is a physical or virtual thing. Endpoints are the management interfaces (SSH on port 22, Redfish on port 443). One device, many endpoints.
- **Adapters:** Protocol-specific plugins that know how to talk to endpoints. LOOM orchestrates; adapters translate.
- **Workflows:** Multi-step operations that may span multiple devices and adapters. Always verified — LOOM checks that what it did actually worked.
- **Verification:** Every operation includes a post-action state check. "I set the VLAN to 100" is not complete until "I confirmed the VLAN is 100."
- **Tenants:** Isolation boundary. Your devices, your workflows, your credentials, your budget.

### Operator Manual

`docs/operations/` covers production concerns:

- **Deployment:** Mode 1 through Mode 4 deployment guides with example configurations
- **Upgrades:** Version-to-version upgrade procedure, database migration steps, rollback procedure
- **Monitoring:** Which Grafana dashboards to watch, alert thresholds, capacity planning guidance
- **Troubleshooting:** Decision tree for common issues (see below)

### Troubleshooting Decision Trees

Common issues with diagnostic steps:

- **Device unreachable:** Check endpoint connectivity → verify credentials → check adapter logs → verify network path → check device management interface status
- **Workflow stuck:** Check workflow status in Temporal UI → check activity task queue depth → check adapter health → check approval queue → check for device lock contention
- **Credential expired:** Check credential health status → re-test credential → rotate credential → verify propagation to all endpoints using that credential
- **Discovery finds nothing:** Check network range configuration → verify adapter is registered for expected protocol → check firewall rules → run manual probe from LOOM host

---

## 7. Backward Compatibility (AL-13, AL-18)

**Finding addressed:** Backward compatibility and upgrade paths (AL-13, High). Temporal workflow versioning (AL-18, High).

### NATS Event Evolution

Rules for NATS event schema changes:
- **Additive-only fields within a minor version.** New fields may be added; existing fields are never removed or renamed.
- **Version in envelope.** Every event envelope includes a `schema_version` field (integer, starting at 1). Consumers must tolerate unknown fields.
- **Breaking changes = new subject.** If a field must be removed, renamed, or its type changed, publish on a new subject (e.g., `loom.{tenant}.device.created.v2`). The old subject continues publishing for the deprecation period (2 minor versions).

### Temporal Workflow Versioning

Mandatory `workflow.GetVersion()` on all activity calls. Cross-reference: [WIRE-PROTOCOL-VERSIONING.md](WIRE-PROTOCOL-VERSIONING.md).

Rules:
- Every workflow change that modifies activity sequence, activity parameters, or control flow **must** be guarded by a `GetVersion` call
- Workflow code reviews include a checklist item: "Does this change require a `GetVersion` guard?"
- CI includes a replay test: start workflow on version N, upgrade to version N+1, verify the in-flight workflow completes without non-determinism errors

```go
// Example: adding a new verification step to a workflow
v := workflow.GetVersion(ctx, "add-post-config-verify", workflow.DefaultVersion, 1)
if v == 1 {
    // New behavior: verify after config push
    err = workflow.ExecuteActivity(ctx, VerifyConfigActivity, params).Get(ctx, &result)
}
// v == DefaultVersion: original behavior (no verification step)
```

### Adapter Contract Versioning

- Adapter contracts follow semver
- The hub supports the current adapter API version plus N-1
- When the adapter gRPC interface changes, the hub maintains both the old and new gRPC service definitions for the compatibility window
- Adapters declare their API version in `AdapterManifest`; the hub routes accordingly

### Config Migration

- **Database migrations:** Managed by golang-migrate. Every `up` migration has a corresponding `down` migration. Both are tested in CI (the CI pipeline runs `up`, seeds data, runs `down`, verifies clean state).
- **Config files:** Versioned config files with a `config_version` field. On startup, LOOM checks the config version and runs automatic upgrade transformations if needed. Transformations are sequential (v1→v2→v3, not v1→v3).
- **Rollback safety:** Every database migration `down` script is tested in CI. A failed upgrade can be rolled back with `loom migrate down` to return to the previous schema version.

---

## 8. Remaining Medium Findings (AL-4, AL-8 through AL-11)

### Self-Debugging (AL-4)

**Finding addressed:** Observability documents describe outputs but not debugging workflows (AL-4, Medium).

`loom admin debug` subcommands provide operator-focused diagnostics:

| Command | Purpose |
|---------|---------|
| `loom admin debug health` | Component health summary (PostgreSQL, NATS, Temporal, Vault, adapters) |
| `loom admin debug queues` | NATS consumer lag, Temporal task queue depth, pending workflows |
| `loom admin debug pools` | Connection pool status (PostgreSQL, Valkey, adapter connections) |
| `loom admin debug workflow <id>` | Aggregated view: workflow status + activity history + logs + traces for a single workflow |
| `loom admin debug adapter <name>` | Adapter health, connected devices, recent errors, version info |
| `loom admin debug tenant <id>` | Tenant quota usage, active workflows, recent events |

These commands query live system state and format it for human consumption. They do not modify state.

### CQRS Projection Rebuild (AL-8)

**Finding addressed:** Projection rebuild is claimed but never specified (AL-8, Medium).

```
loom admin db rebuild-projections [--from-date DATE] [--tenant TENANT_ID]
```

Rebuild procedure:
1. Mark all projections as stale (prevents serving stale data during rebuild)
2. Replay Temporal workflow histories to regenerate domain events
3. Apply events to PostgreSQL projections in order
4. Validate projection checksums against Temporal source-of-truth counts
5. Mark projections as current

The rebuild is idempotent — projections are written with upsert semantics. A partial rebuild (interrupted mid-way) can be resumed from the last processed workflow.

Scope controls:
- `--from-date` limits replay to workflows started after the given date (for partial corruption)
- `--tenant` limits replay to a single tenant (for tenant-specific issues)

### UI Real-Time (AL-9)

**Finding addressed:** WebSocket contract undefined (AL-9, Medium).

WebSocket endpoint defined in [API-CONVENTIONS.md](API-CONVENTIONS.md):

```
ws://{host}/api/v1/ws
wss://{host}/api/v1/ws  (production)
```

**Authentication:** JWT token passed as `token` query parameter on connection upgrade. Token is validated on upgrade; connection is closed if token expires (client must reconnect with fresh token).

**Message types:**

Client → Server:
```json
{"type": "subscribe", "channel": "devices", "filter": {"tenant_id": "..."}}
{"type": "subscribe", "channel": "workflows", "filter": {"workflow_id": "..."}}
{"type": "unsubscribe", "channel": "devices"}
```

Server → Client:
```json
{"type": "event", "channel": "devices", "event": "device.state_changed", "data": {...}}
{"type": "error", "code": "invalid_channel", "message": "..."}
```

**Reconnection:** Clients use exponential backoff (1s, 2s, 4s, 8s, max 30s). The server includes a `last_event_id` in each event; clients send this on reconnection to resume from where they left off.

**Throttling:** The server aggregates high-frequency events (e.g., 1000 device state changes per second) into batched updates delivered at most once per second per channel.

### CI/CD Gaps (AL-10)

**Finding addressed:** Missing CI/CD capabilities (AL-10, Medium).

Additions to the CI/CD pipeline:

- **DB migration down testing:** CI runs `migrate up` → seed data → `migrate down` → verify clean state for every migration. This catches broken rollback scripts before they reach production.
- **Multi-architecture builds:** CI tests run on both `linux/amd64` and `linux/arm64` runners. GoReleaser produces binaries for both architectures. ARM-specific crypto and alignment issues are caught before release.
- **Nightly E2E and scale tests:** The full E2E suite (12 mock devices) plus the scale suite (1,000 simulated devices) run nightly. Failures create GitHub issues automatically. PR-level E2E is not run by default (too slow) but can be triggered with a `/test-e2e` comment.

### LLM Autonomy Model (AL-11)

**Finding addressed:** "LLM suggests only" boundary will face pressure from operators wanting auto-approval (AL-11, Medium).

Graduated trust model — every operation type starts at "always require human approval" and can be relaxed based on historical success rate:

```go
type AutonomyPolicy struct {
    OperationType       string  // e.g., "power_cycle", "vlan_config", "firmware_update"
    RequiredSuccessRate float64 // e.g., 0.95 = 95% historical success before relaxing
    MinHistoricalOps    int     // minimum operations before trust can be upgraded
    CurrentTier         int     // 0-4, see below
}
```

**Autonomy tiers:**

| Tier | Name | Behavior |
|------|------|----------|
| 0 | Deterministic only | LLM is not consulted; hardcoded rules only |
| 1 | Suggest + approve all | LLM suggests, human approves every operation |
| 2 | Suggest + auto-approve routine | LLM suggests, routine operations auto-approved, changes require approval |
| 3 | Decide routine | LLM decides routine operations autonomously, human approves infrastructure changes |
| 4 | Full auto with circuit breaker | LLM decides all operations; circuit breaker halts on failure spike |

**Tier promotion criteria:**
- Minimum `MinHistoricalOps` operations completed at current tier
- Success rate >= `RequiredSuccessRate` over those operations
- No critical failures (any operation that required manual recovery resets the counter)
- Operator explicit approval to promote (tier changes are never automatic)

**Tier demotion:**
- Any critical failure immediately demotes to Tier 1 for that operation type
- Success rate dropping below threshold triggers a warning; operator decides whether to demote

**Audit:** Every auto-approved operation is logged with the autonomy tier, confidence score, and the LLM's reasoning. Operators can review auto-approved decisions in the approval analytics dashboard and retroactively flag incorrect decisions for trust recalibration.

---

## Cross-Reference to Consolidated Gap Analysis

| This Document | Consolidated Gap | Application-Layer Gap |
|---------------|-----------------|----------------------|
| Section 1 | C10 | Gap 1 (Adapter SDK), Gap 2 (Plugin Lifecycle) |
| Section 2 | — | Gap 3 (REST API) |
| Section 3 | H11 (partial) | Gap 5 (LLM Practicalities) |
| Section 4 | — | Gap 6 (Multi-Tenancy) |
| Section 5 | — | Gap 7 (Testing Strategy) |
| Section 6 | — | Gap 12 (Documentation) |
| Section 7 | C2, H23 | Gap 13 (Backward Compat), Gap 18 (Temporal Versioning) |
| Section 8 | M22, M23, M24 | Gap 4, Gap 8-11 (Medium findings) |
