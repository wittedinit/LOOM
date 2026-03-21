# LOOM Adapter SDK

> **Status:** Approved -- resolves adapter-related gaps from GAP-ANALYSIS.md
> **Addresses:** C7, C10, H8, H15, H16, H19, H21, PAR-1 through PAR-10
> **Sources:** RESEARCH-protocol-adapter-realism.md, GAP-ANALYSIS.md, ADAPTER-CONTRACT.md
> **Last updated:** 2026-03-21

---

## 1. Adapter SDK Design (C10, PAR-1, AL-1)

### 1.1 Scaffolding CLI

The `loom adapter init` command generates a working adapter project with all required boilerplate, conformance test imports, and mock device setup.

```bash
$ loom adapter init my-snmp-adapter snmp

Created adapters/my-snmp-adapter/:
  adapter.go          # Adapter struct implementing Connector + optional interface stubs
  adapter_test.go     # Conformance test suite import + protocol-specific test cases
  config.go           # AdapterConfig with protocol-specific defaults
  operations.go       # Operation type stubs for Executor (if applicable)
  mock_device.go      # MockSNMPDevice with pre-loaded MIB responses
  README.md           # Getting started guide, conformance checklist, review requirements
  testdata/           # Sample device responses for mock testing
```

The CLI accepts any protocol supported by the mock device framework: `ssh`, `snmp`, `redfish`, `netconf`, `gnmi`, `ipmi`, `amt`, `pikvm`, `eapi`, `nxapi`, `http`.

### 1.2 Generated Project Structure

Every scaffolded adapter project contains:

| File | Purpose |
|------|---------|
| `adapter.go` | Struct implementing `Connector` (required) plus stubs for `Discoverer`, `Executor`, `StateReader`, `Watcher` marked with `// TODO: implement or remove` |
| `adapter_test.go` | Imports `github.com/wittedinit/loom/sdk/conformance` and runs the full conformance suite against a mock device |
| `config.go` | `AdapterConfig` struct with protocol-specific defaults (ports, timeouts, retry behavior) |
| `operations.go` | Typed operation stubs with `CompensationInfo` declarations |
| `mock_device.go` | Protocol-specific mock device initialized with realistic responses |
| `README.md` | Step-by-step guide: build, test, register, submit |
| `testdata/` | Canned device responses (SNMP walks, Redfish JSON payloads, CLI outputs) |

### 1.3 Conformance Test Suite

Every adapter MUST pass the conformance suite before merge. The suite contains **47 tests** organized into seven categories:

```go
import "github.com/wittedinit/loom/sdk/conformance"

func TestMyAdapter(t *testing.T) {
    adapter := NewMyAdapter(testConfig)
    conformance.RunAll(t, adapter, conformance.Options{
        Connector:   true,  // required for all adapters
        Discoverer:  true,
        Executor:    true,
        StateReader: true,
        Watcher:     false, // only if protocol supports streaming
    })
}
```

**Conformance test categories:**

| Category | Test Count | What It Validates |
|----------|-----------|-------------------|
| Connect/Disconnect | 6 | Connection lifecycle, reconnect, ping, context cancellation |
| Discovery | 7 | Scope handling, identity extraction, fact categorization, empty results |
| Execute | 10 | Operation dispatch, typed results, idempotency key honor, compensation info |
| Verify | 5 | State read accuracy, snapshot integrity, checksum validation |
| Compensate | 8 | Reverse operations, snapshot restore, CompensationReliability declaration accuracy |
| Error handling | 7 | Error classification (transient/permanent/partial/validation), correlation ID propagation |
| Idempotency | 4 | Same key returns same result, concurrent same-key handling, TTL >= 2x operation timeout |

**Hard rules enforced by conformance:**

1. Adapters MUST NOT retry internally (retry ownership belongs to Temporal).
2. All errors MUST be classified -- raw `error` returns fail the suite.
3. Every mutating operation MUST declare `CompensationInfo`.
4. Context cancellation MUST be respected within 1 second.
5. `IdempotencyKey` replay MUST return the cached result, not re-execute.

### 1.4 Mock Device Framework

Protocol-specific mock servers simulate real device behavior for testing without lab hardware.

```go
import "github.com/wittedinit/loom/sdk/mockdevice"

// SNMP mock with standard MIBs
snmpDevice := mockdevice.NewSNMP(mockdevice.SNMPConfig{
    Community:  "public",
    MIBs:       mockdevice.StandardMIBs,
    FailRate:   0.0, // deterministic for conformance tests
})

// Redfish mock with Dell OEM extensions
redfishDevice := mockdevice.NewRedfish(mockdevice.RedfishConfig{
    Vendor:       "Dell",
    Model:        "PowerEdge R750",
    InitialState: mockdevice.PoweredOn,
    OEMData:      mockdevice.DellIDRACDefaults,
})

// SSH mock with command/response pairs
sshDevice := mockdevice.NewSSH(mockdevice.SSHConfig{
    OS:       "ios-xe",
    Hostname: "switch01",
    Responses: map[string]string{
        "show version":        testdata.IOSXE_ShowVersion,
        "show running-config": testdata.IOSXE_RunningConfig,
    },
})

// NETCONF mock with YANG datastore
netconfDevice := mockdevice.NewNETCONF(mockdevice.NETCONFConfig{
    Vendor:       "junos",
    Capabilities: mockdevice.JunosCapabilities,
    Datastore:    mockdevice.JunosSampleConfig,
})
```

**`MockDevice` interface per protocol family:**

| Mock Type | Simulates | Key Behaviors |
|-----------|-----------|---------------|
| `MockSNMPDevice` | SNMP v2c/v3 agent | GET, GETNEXT, GETBULK, WALK; MIB tree navigation; community/USM auth |
| `MockRedfishDevice` | Redfish BMC | REST endpoints, OEM extensions, async tasks, session auth |
| `MockSSHDevice` | SSH CLI device | Command/response matching, prompt detection, enable mode |
| `MockNETCONFDevice` | NETCONF server | Capability exchange, edit-config, commit, confirmed-commit, lock/unlock |
| `MockGNMIDevice` | gNMI target | Get, Set, Subscribe (on-change + sample); OpenConfig paths |
| `MockIPMIDevice` | IPMI BMC | Chassis control, sensor data records, SOL session |

Mock devices support configurable failure injection (`FailRate`, `LatencyRange`, `ErrorAfterN`) for resilience testing beyond conformance.

### 1.5 Example Walkthrough: SSH Adapter from Zero to Passing Conformance

A complete, documented example walks a contributor through building a simple SSH adapter in under one hour:

1. **Scaffold**: `loom adapter init my-ssh-adapter ssh`
2. **Implement Connector**: Fill in `Connect()` (SSH dial + auth), `Disconnect()` (close session), `Ping()` (exec `echo ok`), `Connected()` (check session state)
3. **Implement Discoverer**: Parse `show version`, `show inventory`, extract `ExternalIdentity` (serial number) and `ObservedFact` (model, firmware version, interface count)
4. **Implement Executor**: Handle `ExecuteCommandOp` -- dispatch a CLI command, parse output, return typed `OperationResult` with `CompensationInfo{Reliability: CompReliabilityNone}`
5. **Implement SnapshotCapable**: `TakeSnapshot()` runs `show running-config`, stores output as `ConfigSnapshot.Data`; `RestoreSnapshot()` replays config line by line
6. **Run conformance**: `go test -v -run TestConformance` -- fix any failures
7. **Register**: Add `AdapterRegistration` to the adapter factory in `cmd/loom/adapters.go`

The example is ~200 lines of implementation code. The walkthrough document explains every decision (why `CompReliabilityNone` for raw commands, why `snapshot_restore` for config-capable devices, how idempotency cache works).

### 1.6 Contributor Guide

**How to add a new adapter:**

1. Run `loom adapter init <name> <protocol>`
2. Implement `Connector` (required) and whichever optional interfaces the protocol supports
3. Declare `CompensationReliability` honestly -- do not overclaim
4. Pass the conformance suite at 100%
5. Add integration test against mock device (required) and real device (encouraged)
6. Submit PR with: adapter code, passing conformance, mock device test, README

**Review checklist (enforced by CI):**

- [ ] All 47 conformance tests pass
- [ ] `CompensationReliability` matches protocol capabilities (reviewer verifies)
- [ ] No internal retries (grep for `time.Sleep`, `retry`, backoff patterns)
- [ ] All errors are classified (no raw `error` returns)
- [ ] Credentials accessed via `CredentialRef` only (no hardcoded secrets)
- [ ] `IdempotencyKey` TTL >= 2x max operation timeout
- [ ] Mock device covers all supported operations
- [ ] README documents supported operations, known limitations, and lab validation status

**CI requirements:**

- Conformance suite runs on every PR
- Linting: `golangci-lint` with LOOM-specific rules (no `map[string]any` in adapter types)
- Race detector: `go test -race`
- Coverage: minimum 80% on adapter code

---

## 2. NETCONF Vendor Fragmentation (C7, PAR-2, PAR-3)

### 2.1 Decision: 5 Separate NETCONF Adapters

A single universal NETCONF adapter is not achievable. YANG model families, candidate datastore semantics, confirmed-commit support, and error reporting differ radically across vendors. LOOM implements **five vendor-specific NETCONF adapters** sharing a common transport layer.

```
adapters/
  netconf-base/       # Shared package: SSH transport, NETCONF framing, capability exchange
  netconf-iosxe/      # Cisco IOS-XE
  netconf-iosxr/      # Cisco IOS-XR
  netconf-junos/      # Juniper Junos
  netconf-nokia/      # Nokia SR OS
  netconf-generic/    # Fallback for unrecognized YANG models (read-only)
```

### 2.2 Adapter-per-Vendor-Family Breakdown

| Adapter | YANG Model Families | Candidate Datastore | Confirmed Commit | Compensation Reliability |
|---------|--------------------|--------------------|-----------------|--------------------------|
| `netconf-iosxe` | Cisco-IOS-XE-native, OpenConfig subset | Yes (IOS-XE 16.x+) | Buggy on some versions | transactional (with version guards) |
| `netconf-iosxr` | Cisco-IOS-XR-*, OpenConfig | Yes | Yes | transactional |
| `netconf-junos` | junos-conf-*, OpenConfig | Yes (always) | Yes (robust) | transactional |
| `netconf-nokia` | nokia-*, OpenConfig (limited) | Yes | Yes | transactional |
| `netconf-generic` | Auto-detected via capability exchange | Probed at connect time | Probed at connect time | best_effort (no vendor-specific knowledge) |

### 2.3 Shared `netconf-base` Package

All NETCONF adapters share a common base package that handles protocol mechanics:

```go
package netconfbase

// Transport handles the SSH + NETCONF framing layer.
type Transport interface {
    Dial(ctx context.Context, endpoint EndpointInfo, cred CredentialRef) error
    Close() error
    SendRPC(ctx context.Context, rpc []byte) ([]byte, error)
}

// Session manages NETCONF session state.
type Session struct {
    transport    Transport
    capabilities []string  // server-advertised capabilities
    sessionID    uint32
    // Lock/unlock, commit, discard-changes
}

// Shared operations implemented once:
// - SSH transport with NETCONF subsystem
// - NETCONF 1.0 and 1.1 framing (EOM and chunked)
// - Capability exchange (<hello>)
// - Lock/unlock datastore
// - Commit / discard-changes
// - Confirmed commit (where supported)
// - Get / get-config
```

### 2.4 Vendor-Specific Responsibilities

Each vendor adapter handles:

- **YANG model parsing**: Map vendor-specific YANG paths to LOOM operation types (e.g., Cisco `Cisco-IOS-XE-native:native/vlan` vs Juniper `junos-conf-root:configuration/vlans`)
- **Confirmed-commit semantics**: IOS-XE has partial/buggy support on certain versions; Junos is robust; Nokia is consistent. Each adapter encodes vendor-specific commit behavior.
- **Error mapping**: Vendor-specific NETCONF `<rpc-error>` codes mapped to LOOM error types
- **Candidate config behavior**: Some vendors require explicit lock; others auto-lock. Each adapter handles its vendor's semantics.

### 2.5 `netconf-generic` Fallback Adapter

For devices with unknown YANG models, the generic adapter provides:

- **Read-only operation**: `get-config` on running datastore, `get` for operational state
- **Capability discovery**: Parse `<hello>` capabilities to identify supported YANG modules
- **No mutations**: Generic adapter does not attempt `edit-config` without vendor knowledge
- **Migration path**: When a device is identified, the operator upgrades to a vendor-specific adapter

### 2.6 Lab Validation Requirements

No NETCONF adapter ships without lab testing against real or virtual devices.

| Vendor | Lab Target | Virtual Option | Containerlab Support |
|--------|-----------|----------------|---------------------|
| Cisco IOS-XE | Catalyst 8000v, CSR1000v | Cisco CML/VIRL | No (requires full VM) |
| Cisco IOS-XR | XRv 9000 | Cisco CML/VIRL | No (requires full VM) |
| Juniper Junos | vSRX, vQFX | Juniper vLabs | Yes (cRPD, vSRX containerlab) |
| Nokia SR OS | SR Linux | Free tier | Yes (containerlab native) |
| SONiC | SONiC-VS | GNS3, KVM | Yes (containerlab) |

**Mandatory lab tests per vendor adapter:**

1. Create VLAN via `edit-config` with vendor-specific YANG model
2. Confirmed commit with timeout, then rollback
3. Read running config via `get-config`
4. Subscribe to interface state changes (where supported)
5. Error handling: invalid config, lock contention, session timeout

---

## 3. Redfish OEM Extension Handling (H19, PAR-4)

### 3.1 Layered Architecture

The Redfish adapter uses a layered design: a base adapter handles DMTF-standard resources, and pluggable OEM extension layers handle vendor-specific operations.

```go
// Base Redfish adapter handles DMTF-standard resources.
type RedfishAdapter struct {
    client    *gofish.APIClient
    vendor    OEMExtension      // pluggable OEM layer, selected at connect time
    limiter   *DeviceRateLimiter
}

// OEMExtension interface for vendor-specific Redfish operations.
type OEMExtension interface {
    // Detect returns true if this extension handles the given Redfish service.
    Detect(ctx context.Context, service *gofish.Service) (bool, error)

    // Enrich adds vendor-specific facts to a discovery result.
    Enrich(ctx context.Context, system *redfish.ComputerSystem, result *DiscoveryResult) error

    // Execute handles vendor-specific operations not covered by base Redfish.
    Execute(ctx context.Context, op TypedOperation) (*OperationResult, error)

    // SupportedOEMOperations returns operations this extension adds.
    SupportedOEMOperations() []OperationType
}
```

### 3.2 Standard vs. OEM Coverage

The base Redfish adapter (no OEM layer) supports:

| Operation | Standard Redfish | Notes |
|-----------|-----------------|-------|
| Power on/off/cycle | Yes | `Actions/ComputerSystem.Reset` |
| Read sensors | Yes | `/Chassis/{id}/Thermal`, `/Chassis/{id}/Power` |
| Set boot device | Yes | `Boot` property on ComputerSystem |
| Virtual media mount | Yes | `VirtualMedia` resource |
| Read hardware inventory | Yes | `Systems`, `Chassis`, `Processors`, `Memory` |

Operations requiring OEM extensions:

| Operation | Dell (iDRAC) | HPE (iLO) | Supermicro |
|-----------|-------------|-----------|------------|
| BIOS configuration | Lifecycle Controller API | Bios/Settings endpoint | Different attribute names |
| Firmware update | DUP package format | iLO firmware format | Different async behavior |
| Storage config (RAID) | PERC/BOSS controller API | Smart Array/Storage API | Limited |
| Full network config | iDRAC NIC settings | iLO NIC settings | Quirks |

### 3.3 OEM Extension Implementations

```go
// Dell iDRAC extension
type DellExtension struct{}

func (d *DellExtension) Detect(ctx context.Context, svc *gofish.Service) (bool, error) {
    // Check /redfish/v1/ for Vendor == "Dell" or Oem.Dell presence
}

func (d *DellExtension) Enrich(ctx context.Context, sys *redfish.ComputerSystem, result *DiscoveryResult) error {
    // Query Lifecycle Controller for: BIOS settings, storage backplane, warranty info
}

func (d *DellExtension) Execute(ctx context.Context, op TypedOperation) (*OperationResult, error) {
    // Handle Dell-specific: BIOSConfigOp, FirmwareUpdateOp (DUP), RAIDConfigOp
}

// HPE iLO extension
type HPEExtension struct{}
// Handles: Active Health System, Federation groups, SmartArray config, iLO firmware

// Supermicro BMC extension
type SMCExtension struct{}
// Handles: IPMI-to-Redfish mapping quirks, non-standard sensor paths, firmware update differences

// Fallback for unrecognized vendors
type GenericExtension struct{}
// Operates with base Redfish only. Logs warning on first connect.
```

### 3.4 OEM Vendor Detection

At connect time, the adapter queries `/redfish/v1/` and inspects the response:

```go
func detectVendor(ctx context.Context, client *gofish.APIClient) (OEMExtension, error) {
    service := client.Service

    // Check Vendor field
    switch {
    case strings.Contains(service.Vendor, "Dell"):
        return &DellExtension{}, nil
    case strings.Contains(service.Vendor, "HPE") || strings.Contains(service.Vendor, "iLO"):
        return &HPEExtension{}, nil
    case strings.Contains(service.Vendor, "Supermicro"):
        return &SMCExtension{}, nil
    default:
        log.Warn("unknown Redfish vendor, using base-only mode", "vendor", service.Vendor)
        return &GenericExtension{}, nil
    }
}
```

OEM extensions are registered at startup and selected dynamically. Unknown vendors operate with base Redfish only -- no operations fail, but vendor-specific features are unavailable.

---

## 4. Device Rate Limiting (H8, PAR-5)

### 4.1 Problem

Without rate limiting, a burst of concurrent workflows targeting the same device can exhaust its management plane. BMCs are especially fragile: 4 concurrent Redfish sessions can exhaust a Dell iDRAC, causing it to drop all management connections including human operators.

### 4.2 Design: Valkey-Backed Token Bucket

```go
// DeviceRateLimiter enforces per-device concurrency and throughput limits.
// State is stored in Valkey for coordination across all LOOM workers.
type DeviceRateLimiter struct {
    valkey       *valkey.Client
    deviceID     string
    profile      RateLimitProfile
}

// Acquire blocks until a rate limit slot is available, or returns error after timeout.
func (r *DeviceRateLimiter) Acquire(ctx context.Context, deviceID string, opType string) (*RateLimitToken, error)

// Release returns a rate limit slot to the pool. Called after operation completes.
func (r *DeviceRateLimiter) Release(ctx context.Context, token *RateLimitToken) error

// RateLimitToken represents a held slot. Includes a lease timeout for crash safety.
type RateLimitToken struct {
    DeviceID  string
    TokenID   string
    AcquiredAt time.Time
    LeaseExpiry time.Time  // auto-released if worker crashes
}
```

### 4.3 Default Rate Limit Profiles

| Device Type | Max Concurrent | Max Per Minute | Rationale |
|------------|---------------|----------------|-----------|
| Network switch (SSH/NETCONF) | 5 | 30 | Management plane shares CPU with control plane |
| Server BMC (Redfish) | 3 | 20 | ARM-class BMC processors are slow |
| Server BMC (IPMI) | 1 | 10 | IPMI is single-threaded on most BMCs |
| Cloud API endpoint | 50 | 1000 | API rate limits are higher but still finite |
| PiKVM | 1 | 10 | Single-user device |

### 4.4 Per-Device Override

Operators can override defaults for specific devices:

```yaml
# loom-config.yaml
rate_limits:
  overrides:
    - device_id: "switch-spine-01"
      max_concurrent: 10
      max_per_minute: 60
      comment: "High-end Arista 7800 with dedicated management CPU"
    - device_id: "bmc-old-server-42"
      max_concurrent: 1
      max_per_minute: 5
      comment: "Ancient Supermicro BMC, extremely fragile"
```

### 4.5 Queue Behavior

1. Worker calls `Acquire(deviceID, opType)` before invoking any adapter operation.
2. If a slot is available, the token is granted immediately.
3. If no slot is available, the request queues in Valkey with FIFO ordering.
4. Queue wait timeout: **30 seconds** (configurable). After 30s, `Acquire` returns a `RateLimitError`.
5. `RateLimitError` is classified as `TransientError` with `RetryAfter` set to the estimated wait time. Temporal retries accordingly.
6. If a worker crashes while holding a token, the lease expires (default: 60s) and the slot is automatically reclaimed.

### 4.6 Interaction with Temporal Activity Timeouts

Temporal activity `ScheduleToClose` timeout must account for potential rate limit wait time. Recommendation: set activity timeout to `max_rate_limit_wait + operation_timeout + buffer`. For BMC operations: `30s wait + 120s operation + 30s buffer = 180s activity timeout`.

---

## 5. RawConfigPush Mitigations (H15, PAR-6)

### 5.1 Problem

`RawConfigPush.ConfigBody` is a freeform string sent directly to a device. Without controls, an operator can push backdoor accounts, disable authentication, modify routing tables, or brick a device.

### 5.2 Mitigation Layers

#### 5.2.1 Content Analysis

Scan config body for dangerous patterns before execution:

```go
// RawConfigPushPolicy defines safety rules for raw config pushes.
type RawConfigPushPolicy struct {
    // DangerousPatterns are regex patterns that flag configs for extra review.
    DangerousPatterns []DangerousPattern

    // RequiresApproval is ALWAYS true for RawConfigPush. Not configurable.
    RequiresApproval bool

    // RequiresBackup forces a ConfigSnapshot before push. Always true.
    RequiresBackup bool
}

type DangerousPattern struct {
    Pattern     string // regex
    Severity    string // "block" or "warn"
    Description string // human-readable explanation
}
```

**Default dangerous patterns:**

| Pattern | Severity | Description |
|---------|----------|-------------|
| `(?i)shutdown` (on interface context) | warn | May disable critical interfaces |
| `(?i)no\s+ip\s+routing` | block | Disables IP routing on the device |
| `(?i)erase\s+startup` | block | Erases startup configuration |
| `(?i)reload` | block | Reboots the device |
| `(?i)format\s+` | block | Formats storage |
| `(?i)no\s+enable\s+(secret\|password)` | block | Removes enable authentication |
| `(?i)username\s+.*privilege\s+15` | warn | Creates privileged user account |
| `(?i)permit\s+ip\s+any\s+any` | warn | Creates permissive ACL rule |
| `(?i)no\s+service\s+password-encryption` | warn | Disables password encryption |

Patterns with severity `block` halt execution. Patterns with severity `warn` flag the push for explicit acknowledgment.

#### 5.2.2 Mandatory Approval Gate

RawConfigPush ALWAYS requires human approval. There is no auto-execute tier. This is enforced at the workflow layer:

1. Workflow planner detects `RawConfigPush` operation type.
2. Workflow pauses and emits approval request via Temporal signal.
3. Approver sees: device target, config body, content analysis results, dangerous pattern matches.
4. Approval or rejection is recorded in the audit trail.

#### 5.2.3 Pre-Push Config Backup

Before any `RawConfigPush` execution:

1. Adapter calls `TakeSnapshot()` on the target device (if `SnapshotCapable`).
2. Snapshot is stored with the workflow execution context.
3. If snapshot fails (device does not support it), the push proceeds but the operator is warned: "No rollback snapshot available."

#### 5.2.4 Post-Push Diff

After successful push:

1. Adapter reads the new running config.
2. LOOM diffs pre-push snapshot against post-push config.
3. Diff is emitted to the audit trail as an `AuditEvent` with type `config_change`.
4. If diff reveals unexpected changes (changes beyond what was in `ConfigBody`), an alert is raised.

#### 5.2.5 Rollback on Verification Failure

If post-push verification detects the device is in an unhealthy state:

1. Workflow triggers `RestoreSnapshot()` with the pre-push snapshot.
2. Restore result is verified.
3. If restore fails, workflow enters `compensation_failed` state and alerts the operator.

---

## 6. Adapter Plugin Security (H16, PAR-7)

### 6.1 Trust Model

Third-party adapters loaded as go-plugin binaries operate as separate processes communicating over local gRPC. Without controls, a malicious plugin could fabricate discovery results, exfiltrate credentials, connect to arbitrary endpoints, or consume excessive resources.

### 6.2 Plugin Binary Signing

All plugin binaries MUST be signed before loading:

```go
// PluginPolicy defines security requirements for loading a plugin adapter.
type PluginPolicy struct {
    // RequireSigning enforces cosign verification before loading.
    RequireSigning bool // default: true

    // TrustedKeys are the cosign public keys accepted for verification.
    TrustedKeys []string

    // AllowUnsigned permits loading unsigned plugins (development only).
    // Requires explicit flag: --allow-unsigned-plugins
    AllowUnsigned bool
}
```

- **Signing tool**: cosign (Sigstore project), same infrastructure used for container image signing.
- **Verification**: Before `plugin.Open()`, LOOM verifies the binary signature against the trusted key set.
- **Unsigned plugins**: Rejected by default. Development mode (`--allow-unsigned-plugins`) permits unsigned plugins with a prominent log warning.

### 6.3 Credential Scoping

Plugins receive a `CredentialRef` and a scoped vault client. The scoped client can ONLY access credentials for devices assigned to that plugin instance:

```go
// PluginCredentialScope restricts what credentials a plugin can access.
type PluginCredentialScope struct {
    // AllowedDeviceIDs are the devices this plugin instance may manage.
    AllowedDeviceIDs []string

    // AllowedVaultPaths are the vault paths this plugin can read.
    // Derived from AllowedDeviceIDs at plugin load time.
    AllowedVaultPaths []string

    // MaxCredentialHoldTime is the maximum duration a plugin may hold
    // a decrypted credential in memory. Exceeding this triggers an alert.
    MaxCredentialHoldTime time.Duration // default: 5 minutes
}
```

### 6.4 Network Restrictions

Plugin processes run with restricted network access:

- **Allowed**: Management VLAN (where target devices live), localhost (gRPC to host)
- **Blocked**: Internet access, other VLANs, internal services (except LOOM hub API)
- **Implementation**: Network namespace (Linux) or pf rules (macOS) applied at plugin process startup
- **Phase note**: Full network namespace isolation is Phase 5+. Phase 2-4 relies on firewall rules.

### 6.5 Resource Limits

```go
// PluginResourceLimits caps what a plugin process can consume.
type PluginResourceLimits struct {
    MaxMemoryMB      int // default: 512
    MaxGoroutines    int // default: 1000
    MaxFileDescriptors int // default: 256
    MaxCPUPercent    int // default: 50 (of one core)
}
```

- **Enforcement**: cgroups v2 (Linux) for memory and CPU. Goroutine and FD limits monitored by the host process.
- **Violation behavior**: First violation triggers a warning. Second violation within 5 minutes triggers plugin termination and restart.
- **Monitoring**: Hub tracks plugin resource usage metrics, exposed via Prometheus.

### 6.6 Runaway Plugin Detection

The hub monitors plugin behavior:

- **Discovery anomaly**: If a plugin suddenly discovers 100x more devices than its historical baseline, an alert fires.
- **Error rate spike**: If a plugin's error rate exceeds 50% over 5 minutes, it is quarantined (no new operations dispatched).
- **Credential access pattern**: If a plugin requests credentials for devices outside its assignment, the request is denied and logged.

---

## 7. MVP Scale (H21, PAR-8)

### 7.1 Updated MVP Target

The original 100-device MVP target is insufficient for architectural validation. Updated target:

| Tier | Device Count | Type | Purpose |
|------|-------------|------|---------|
| Real devices | 100 | Physical or virtual lab equipment | Validate adapter correctness, real protocol behavior |
| Simulated devices | 1,000 | Mock SNMP/Redfish/SSH responders | Validate connection pooling, DB performance, rate limiting, identity matching |

### 7.2 Simulated Device Fleet

The device simulator provides synthetic load using the mock device framework from section 1.4:

```go
simulator := devicesim.New(devicesim.Config{
    DeviceCount:    1000,
    Protocols:      []string{"ssh", "redfish", "snmp"},
    FailureRate:    0.05,  // 5% of operations fail randomly
    LatencyProfile: devicesim.WAN,  // realistic WAN latencies (10-200ms)
    DuplicateRate:  0.10,  // 10% of devices share identity attributes (tests merge logic)
})
```

Simulated devices run as lightweight goroutines (SNMP, Redfish) or processes (SSH) on the test infrastructure. They respond to real protocol requests with canned but varied data.

### 7.3 Scale Test Acceptance Criteria

| Metric | Target | Measured Against |
|--------|--------|-----------------|
| p99 discovery latency | < 30 seconds | 100 real devices, full hardware discovery |
| p99 workflow execution | < 60 seconds | Single-device config push workflow |
| p99 bulk discovery | < 5 minutes | 1,000 simulated devices, concurrent |
| Zero data loss | 0 events dropped | Under sustained 100 ops/sec for 1 hour |
| Identity merge accuracy | > 99% | 1,000 devices with 10% duplicate rate |

### 7.4 Scale Test Execution

- Scale tests run **nightly in CI** against the simulated fleet.
- Real device tests run **weekly** (lab equipment availability).
- Results are tracked as time-series data for regression detection.
- Any regression beyond 10% on p99 latency fails the CI pipeline.

---

## 8. MikroTik and gNMI Realism (PAR-9, PAR-10)

### 8.1 MikroTik Adapter

The MikroTik REST API does not support transactions, rollback, or atomic multi-operation execution. Server-assigned resource IDs (`.id` field) make compensation unreliable -- the ID of a recreated resource differs from the original.

```go
// MikroTik adapter registration
AdapterRegistration{
    Name:     "mikrotik-rest",
    Protocol: "mikrotik-rest",
    // ...
}

// CompensationReliability: BestEffort
// Rationale: No transactional rollback. Reverse operations are possible
// (delete a created VLAN, re-add a deleted route) but server-assigned IDs
// mean the restored state is not identical to the pre-operation state.
```

**Documented limitations:**

- No atomic multi-operation support. Each API call is independent.
- Server-assigned `.id` values are not predictable. Compensation creates new resources with new IDs.
- No candidate configuration or confirmed commit.
- Error responses are not structured -- adapter must parse error strings.
- Firmware version differences change API behavior without warning.

### 8.2 gNMI Adapter

The gNMI adapter (targeting Arista EOS, SONiC, and other OpenConfig-first vendors) cannot guarantee atomic rollback on mixed success/failure across multiple Set operations.

```go
// gNMI adapter registration
AdapterRegistration{
    Name:     "gnmi-openconfig",
    Protocol: "gnmi",
    // ...
}

// CompensationReliability: BestEffort
// Rationale: gNMI Set operations are applied sequentially. If a Set
// containing multiple updates partially fails, the successful updates
// are already applied. There is no confirmed-commit or candidate
// datastore in the gNMI protocol itself (vendor implementations vary).
```

**Documented limitations:**

- gNMI `Set` with multiple updates is NOT atomic on all implementations. Arista applies updates sequentially.
- No protocol-level confirmed commit (unlike NETCONF). Rollback requires issuing reverse `Set` operations.
- Subscribe `ON_CHANGE` behavior varies by vendor -- some vendors silently fall back to `SAMPLE`.
- OpenConfig model coverage varies -- some paths are read-only on certain vendors.
- SONiC gNMI support is evolving and may have gaps in OpenConfig path coverage.

### 8.3 Compensation Reliability Comparison

| Adapter | Compensation Reliability | Atomic Rollback | Confirmed Commit | Notes |
|---------|------------------------|----------------|-----------------|-------|
| netconf-junos | transactional | Yes | Yes (robust) | Gold standard for network config |
| netconf-iosxr | transactional | Yes | Yes | Reliable on modern versions |
| netconf-iosxe | transactional | Yes | Partial (version-dependent) | Test on target version |
| netconf-nokia | transactional | Yes | Yes | Consistent behavior |
| gnmi-openconfig | best_effort | No | No (protocol limitation) | Weaker than NETCONF |
| mikrotik-rest | best_effort | No | No | Weakest guarantees |
| eapi (Arista) | best_effort | No | No | Config session provides some protection |
| ssh (with config backup) | snapshot_restore | No | No | Depends on device CLI support |

Workflows targeting devices with `best_effort` or weaker compensation MUST account for this in their rollback planning. The workflow planner flags these automatically and may require operator acknowledgment for multi-step mutations.

---

## 9. Implementation Priority

| Item | Phase | Effort | Dependencies |
|------|-------|--------|--------------|
| Adapter SDK (scaffolding CLI + conformance suite) | 1B | 3-4 weeks | Adapter contract finalized |
| Mock device framework (SNMP + Redfish + SSH) | 1B | 2-3 weeks | Adapter interfaces |
| Device simulator (1,000 synthetic devices) | 1B-2 | 2-3 weeks | Mock device framework |
| Device rate limiter (Valkey-backed token bucket) | 2 | 2-3 weeks | Valkey deployment |
| NETCONF vendor lab testing (IOS-XE, Junos, Nokia) | 2 | 2-3 weeks | Lab equipment |
| Redfish layered adapter (standard + Dell OEM) | 2 | 3-4 weeks | Lab equipment |
| RawConfigPush approval gate + content analysis | 3 | 1-2 weeks | RBAC implementation |
| Plugin binary signing + credential scoping | 3 | 2-3 weeks | Signing infrastructure |
| Plugin network restrictions + resource limits | 5 | 2-3 weeks | Container/namespace infrastructure |
| gNMI vendor capability matrix | 7 | 2-3 weeks | Lab equipment |

---

## 10. Open Questions Tracker

| # | Question | Status | Decision Target |
|---|----------|--------|----------------|
| OQ-1 | Should the SDK be a separate Go module (`github.com/wittedinit/loom-sdk`)? | Open | Phase 1B |
| OQ-2 | Should Nokia SR OS be deferred to Phase 7+ given limited market share outside telco? | Open | Phase 2 planning |
| OQ-3 | Can containerized NOS images serve as CI integration test targets? | Open | Phase 2 |
| OQ-4 | Should rate limits be auto-discovered from device capabilities? | Open | Phase 3 |
| OQ-5 | Should RawConfigPush be disabled by default (explicit opt-in per tenant)? | Open | Phase 3 |
| OQ-6 | Is plugin isolation worth the complexity for Phases 1-5, or compile all adapters in? | Open | Phase 2 |
| OQ-7 | Should LOOM target OpenConfig-only for network adapters? | Open | Phase 2 |
| OQ-8 | What percentage of real-world Redfish operations need OEM extensions? | Open | Phase 2 |
| OQ-9 | Can the device simulator double as the mock device framework for adapter testing? | Open | Phase 1B |
| OQ-10 | Should scale testing be automated in CI or run as periodic benchmarks? | Resolved | Nightly CI (section 7.4) |

---

## 11. References

- [ADAPTER-CONTRACT.md](ADAPTER-CONTRACT.md) -- Adapter interface definitions
- [OPERATION-TYPES.md](OPERATION-TYPES.md) -- Operation type taxonomy
- [RESEARCH-protocol-adapter-realism.md](RESEARCH-protocol-adapter-realism.md) -- Full research findings
- [GAP-ANALYSIS.md](GAP-ANALYSIS.md) -- Consolidated gap analysis
- [ERROR-MODEL.md](ERROR-MODEL.md) -- Error classification and retry ownership
- [TESTING-STRATEGY.md](TESTING-STRATEGY.md) -- Testing approach
- ADR-006 -- Adapter families decision
- ADR-012 -- Plugin architecture
