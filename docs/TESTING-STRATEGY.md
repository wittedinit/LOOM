# Testing Strategy

> How LOOM is tested: from unit tests through end-to-end verification against simulated device fleets.

Every test category described here must pass before a PR is merged. No exceptions.

---

## 1. Unit Testing

### Approach

Go table-driven tests for all business logic. Every package has a `_test.go` file adjacent to the code it tests.

```go
func TestDeviceStatusTransition(t *testing.T) {
    tests := []struct {
        name     string
        from     DeviceStatus
        to       DeviceStatus
        valid    bool
    }{
        {"discovered to active", DeviceStatusDiscovered, DeviceStatusActive, true},
        {"active to degraded", DeviceStatusActive, DeviceStatusDegraded, true},
        {"decommissioned to active", DeviceStatusDecommissioned, DeviceStatusActive, false},
    }

    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            err := ValidateTransition(tt.from, tt.to)
            if tt.valid && err != nil {
                t.Errorf("expected valid transition, got error: %v", err)
            }
            if !tt.valid && err == nil {
                t.Error("expected invalid transition, got nil error")
            }
        })
    }
}
```

### Coverage Target

- **Global minimum: >80% line coverage.**
- Critical paths (adapter error classification, workflow state machines, credential handling, tenant isolation enforcement): **>90% coverage.**
- Coverage is measured by `go test -coverprofile` and enforced in CI. PRs that drop coverage below the threshold are blocked.

### What Gets Unit Tested

| Package | Focus |
|---------|-------|
| `domain/` | Model validation, enum transitions, type relationships |
| `adapter/` | Error classification, capability declaration, idempotency key handling |
| `workflow/` | Step ordering, compensation selection, timeout calculation |
| `llm/` | Prompt construction, structured output parsing, provider routing logic |
| `api/` | Request validation, pagination cursor encoding, error response formatting |
| `auth/` | JWT parsing, tenant extraction, RBAC policy evaluation |
| `cost/` | Budget calculations, rate aggregation, threshold checks |

---

## 2. Adapter Testing

Adapters talk to external devices over real protocols. Tests use mock servers that speak each protocol natively.

### Mock Device Infrastructure

Each protocol adapter has a dedicated mock server that simulates real device behavior:

| Protocol | Mock Implementation | What It Simulates |
|----------|-------------------|------------------|
| SNMP | `gosnmp/trap` listener + mock MIB responder | sysDescr, ifTable, entPhysicalTable responses |
| Redfish | `net/http/httptest` serving DMTF-compliant JSON | Chassis, Systems, Managers, Power, Thermal endpoints |
| SSH | `gliderlabs/ssh` test server | CLI command execution, config output, error responses |
| IPMI | UDP listener with IPMI message framing | Chassis status, sensor data, power control responses |
| NETCONF | TCP server with XML-framed NETCONF `<hello>` and `<rpc-reply>` | `<get-config>`, `<edit-config>`, capability exchange |
| gNMI | gRPC test server with `gnmi.proto` | Get, Set, Subscribe (telemetry streaming) |
| AMT | SOAP/WS-Management test server | Power state, boot options, KVM redirect |
| eAPI | JSON-RPC HTTP test server | `show` commands, config session responses |
| PiKVM | HTTP test server | ATX power control, HID events, screenshot capture |

### Test Fixture Types

```go
// TestDevice provides a pre-configured device for adapter tests.
type TestDevice struct {
    ID           uuid.UUID
    Name         string
    Type         DeviceType
    Vendor       string
    Model        string
    Endpoints    []TestEndpoint
    TenantID     uuid.UUID
}

// MockAdapter wraps a real adapter with a mock server backend.
// Start() launches the mock server; Stop() tears it down.
type MockAdapter struct {
    Adapter      Adapter
    MockServer   io.Closer       // the mock protocol server
    ListenAddr   string          // address the mock server is listening on
    Responses    map[string]any  // canned responses keyed by operation
    CallLog      []AdapterCall   // recorded calls for assertions
}

// TestTenant provides an isolated tenant context for tests.
type TestTenant struct {
    ID          uuid.UUID
    Name        string
    Credentials []CredentialRef
    Devices     []TestDevice
}
```

### Adapter Test Patterns

1. **Happy path**: Connect, discover, execute, read state, disconnect.
2. **Connection failure**: Mock server rejects connection. Verify `TransientError` with `RetryAfter`.
3. **Authentication failure**: Mock server returns 401/403. Verify `PermanentError`.
4. **Timeout**: Mock server delays response beyond activity timeout. Verify context cancellation propagates.
5. **Partial failure**: Mock server succeeds for some operations, fails for others. Verify `PartialError` with correct `Succeeded`/`Failed` lists.
6. **Idempotency**: Submit same `IdempotencyKey` twice. Verify second call returns cached result without re-execution.
7. **Protocol-specific edge cases**: Malformed SNMP responses, Redfish `@odata.nextLink` pagination, SSH session multiplexing limits.

---

## 3. Integration Testing

Integration tests verify that LOOM components work together with real infrastructure dependencies.

### Infrastructure Setup

All integration tests run against real services started via Docker Compose:

```yaml
# docker-compose.test.yml
services:
  postgres:
    image: timescale/timescaledb-ha:pg16
    # Includes TimescaleDB, AGE, pgvector extensions
    environment:
      POSTGRES_DB: loom_test
      POSTGRES_USER: loom_test
      POSTGRES_PASSWORD: test
    ports:
      - "5433:5432"

  nats:
    image: nats:2-alpine
    command: ["--jetstream", "--store_dir", "/data"]
    ports:
      - "4223:4222"
```

### Temporal Testing

Temporal integration tests use the Temporal test framework (`go.temporal.io/sdk/testsuite`), which provides an in-process test environment. No external Temporal server is needed.

```go
func TestProvisioningWorkflow(t *testing.T) {
    suite := testsuite.WorkflowTestSuite{}
    env := suite.NewTestWorkflowEnvironment()

    // Register activities with mock adapters
    env.RegisterActivity(allocateResources)
    env.RegisterActivity(configureNetwork)
    env.RegisterActivity(verifyProvisioning)

    // Override activities to use mock adapters
    env.OnActivity(configureNetwork, mock.Anything, mock.Anything).
        Return(&ActivityResult{Success: true}, nil)

    // Execute workflow
    env.ExecuteWorkflow(ProvisioningWorkflow, request)

    require.True(t, env.IsWorkflowCompleted())
    require.NoError(t, env.GetWorkflowError())
}
```

### Integration Test Categories

| Category | What It Tests | Dependencies |
|----------|-------------|--------------|
| Repository tests | CRUD operations, tenant isolation queries, cursor pagination | PostgreSQL |
| Event pipeline tests | Publish events, verify consumer receives, check NATS JetStream acknowledgment | NATS JetStream |
| Workflow integration | Full workflow execution with real DB writes and event emission | Temporal test env + PostgreSQL + NATS |
| Topology graph tests | Apache AGE Cypher queries for device relationships | PostgreSQL + AGE |
| Telemetry pipeline tests | TimescaleDB continuous aggregate creation, chunk compression | PostgreSQL + TimescaleDB |
| Credential lifecycle tests | Store, retrieve, rotate credentials with encryption | PostgreSQL |

### Tenant Isolation Verification

Every integration test that touches the database runs with two test tenants. After the test, an assertion verifies that Tenant A's operations never leaked data into Tenant B's namespace:

```go
func assertTenantIsolation(t *testing.T, db *pgxpool.Pool, tenantA, tenantB uuid.UUID) {
    // Query all tables for rows where tenant_id doesn't match the expected tenant
    // This catches missing WHERE tenant_id = ? clauses
    tables := []string{"devices", "endpoints", "observed_facts", "desired_states"}
    for _, table := range tables {
        var count int
        err := db.QueryRow(ctx,
            fmt.Sprintf("SELECT COUNT(*) FROM %s WHERE tenant_id NOT IN ($1, $2)", table),
            tenantA, tenantB,
        ).Scan(&count)
        require.NoError(t, err)
        require.Zero(t, count, "tenant isolation violated in table %s", table)
    }
}
```

---

## 4. End-to-End Testing

E2E tests exercise the full request-to-verification flow against a simulated device fleet.

### Simulated Fleet

A Docker Compose environment stands up a fleet of mock devices:

- 5 mock Redfish servers (simulating Dell iDRAC, HPE iLO)
- 3 mock SNMP agents (simulating Cisco, Arista switches)
- 2 mock SSH servers (simulating SONiC, Cumulus Linux)
- 1 mock gNMI server (simulating Nokia SR Linux)
- 1 mock PiKVM server

### E2E Test Flow

1. **Submit request**: POST a provisioning request to the LOOM API.
2. **Observe workflow**: Poll workflow status until completion or timeout.
3. **Verify execution**: Confirm that mock devices received the expected adapter calls in the correct order.
4. **Verify state**: Query the API for device state and confirm it matches `ExpectedOutcome`.
5. **Verify events**: Consume NATS events and confirm the expected event sequence was emitted.
6. **Verify audit trail**: Query the audit API and confirm all operations are recorded with correct actor, tenant, and correlation ID.
7. **Verify rollback**: For failure scenarios, confirm compensation activities executed and devices were restored to pre-operation state.

### E2E Scenarios

| Scenario | Description |
|----------|-------------|
| Full discovery | Discover all mock devices, verify correct type/vendor/model classification |
| Provisioning happy path | Provision a server: network config, OS install, verification |
| Provisioning with rollback | Fail mid-provisioning, verify saga compensation reverses all completed steps |
| Power cycle | Power off, verify off state, power on, verify on state |
| Teardown with approval | Submit teardown, approve via signal, verify complete decommission |
| Multi-tenant isolation | Run operations for two tenants simultaneously, verify zero cross-contamination |
| Edge agent sync | Disconnect edge agent, queue operations, reconnect, verify sync |

---

## 5. Security Testing

Security tests are derived from the attack surface documents in `docs/security/`. Every attack vector documented there has a corresponding regression test.

### Attack Surface Regression Tests

| Attack Surface Doc | Automated Tests |
|-------------------|-----------------|
| `ATTACK-SURFACE-API.md` | JWT forgery, tenant ID manipulation, injection attempts, rate limit bypass |
| `ATTACK-SURFACE-ADAPTERS.md` | Credential leak via adapter errors, command injection through device responses |
| `ATTACK-SURFACE-LLM.md` | Prompt injection via device data, output validation bypass, cost exhaustion |
| `ATTACK-SURFACE-NATS.md` | Cross-tenant event subscription, unauthorized publish, replay attacks |
| `ATTACK-SURFACE-POSTGRESQL.md` | SQL injection via JSONB, tenant isolation bypass, connection hijacking |
| `ATTACK-SURFACE-TEMPORAL.md` | Workflow impersonation, signal injection, activity spoofing |
| `ATTACK-SURFACE-VAULT.md` | Credential exfiltration, key rotation failure, envelope encryption bypass |
| `ATTACK-SURFACE-UI-SUPPLY-CHAIN.md` | XSS via device names, dependency vulnerability checks |

### Security Test Structure

```go
func TestAPITenantIsolation(t *testing.T) {
    tests := []struct {
        name        string
        attackerJWT string // JWT for Tenant A
        targetURL   string // URL targeting Tenant B's resource
        wantStatus  int
    }{
        {
            name:        "tenant A cannot read tenant B devices",
            attackerJWT: tenantAToken,
            targetURL:   "/api/v1/devices/" + tenantBDeviceID,
            wantStatus:  404, // 404 not 403 — do not leak existence
        },
        // ... more cases
    }
}
```

### Credential Handling Tests

- Verify `CredentialRef` structs never contain raw secrets in serialized output.
- Verify adapter error messages do not leak credential material.
- Verify log output is scrubbed of sensitive fields.
- Verify credentials are cleared from memory after adapter disconnect.

---

## 6. Performance Testing

### Benchmarks

Go benchmarks (`testing.B`) for performance-critical paths:

```go
func BenchmarkConcurrentDeviceConnections(b *testing.B) {
    // Measure time to establish N concurrent adapter connections
    // Target: 1000 connections in <5 seconds
}

func BenchmarkWorkflowThroughput(b *testing.B) {
    // Measure workflows completed per second
    // Target: >100 workflows/sec with Temporal test environment
}

func BenchmarkEventIngestion(b *testing.B) {
    // Measure events published and consumed per second via NATS
    // Target: >10,000 events/sec sustained
}

func BenchmarkTopologyQuery(b *testing.B) {
    // Measure Apache AGE graph traversal time
    // Target: 4-hop query on 10,000-node graph in <100ms
}
```

### Load Test Scenarios

| Scenario | Target | Tool |
|----------|--------|------|
| Concurrent device connections | 1,000 simultaneous adapter connections | Custom Go harness |
| Workflow throughput | 100 workflows/sec sustained for 10 minutes | Temporal load test framework |
| Event throughput | 10,000 events/sec via NATS JetStream | `nats bench` + custom publisher |
| API latency under load | P99 < 200ms at 500 req/sec | `vegeta` or `k6` |
| Discovery at scale | Discover 1,000 devices in <5 minutes | E2E test harness with mock fleet |

---

## 7. CI Integration

All tests run on every PR via GitHub Actions. See [CI-CD-PIPELINE.md](CI-CD-PIPELINE.md) for the full workflow structure.

### Test Execution Order

```
PR opened/updated
  └─ Lint + Vet (fast feedback)
      └─ Unit tests (parallel, ~2 min)
          └─ Integration tests (Docker Compose, ~5 min)
              └─ Security regression tests (~3 min)
                  └─ E2E tests (simulated fleet, ~10 min)
```

### Test Reporting

- JUnit XML output for GitHub Actions test summary.
- Coverage reports uploaded as PR artifacts.
- Flaky test detection: tests that fail intermittently are flagged and quarantined, not deleted.

### Build Tags

```go
//go:build integration
// +build integration

//go:build e2e
// +build e2e

//go:build security
// +build security
```

- Unit tests: no build tag (run by default).
- Integration tests: `go test -tags=integration` (requires Docker Compose services).
- E2E tests: `go test -tags=e2e` (requires simulated fleet).
- Security tests: `go test -tags=security` (may require specific setup).
- Performance benchmarks: `go test -bench=. -benchmem` (run on-demand, not on every PR).
