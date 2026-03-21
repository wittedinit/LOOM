# Research: Protocol Adapter Realism

> **Status:** Open — requires lab validation and design decisions before Phase 2
> **Addresses:** GAP-ANALYSIS.md findings C7, C10, H8, H15, H16, H19, H21
> **Owner:** TBD
> **Last updated:** 2026-03-21

---

## 1. Problem Statement

LOOM's value scales linearly with adapter count. The current contracts present protocol adapters as cleaner abstractions than reality permits. Six areas require research:

1. **NETCONF vendor fragmentation** — Single adapter assumed for radically different YANG models
2. **Redfish OEM divergence** — Single adapter for Dell/HPE/Supermicro/Lenovo with vendor-specific extensions
3. **Adapter SDK and developer experience** — No scaffolding, no examples, no conformance tests
4. **Device rate limiting** — No protection against overloading device management planes
5. **RawConfigPush security** — Freeform config push bypasses all validation
6. **Plugin trust boundary** — go-plugin gRPC adapters have no sandbox or behavioral monitoring
7. **MVP validation scale** — 100 devices is insufficient for architectural validation

---

## 2. NETCONF Vendor Fragmentation

### 2.1 The Reality

The adapter contract lists NETCONF as a single adapter supporting "Cisco IOS-XE/XR, Junos, Nokia, Arista" with "transactional" compensation reliability. This is not achievable with a single adapter.

**YANG model divergence:**

| Capability | Cisco IOS-XE | Cisco IOS-XR | Juniper Junos | Nokia SR OS | Arista EOS |
|-----------|-------------|-------------|---------------|-------------|------------|
| YANG model family | Cisco-native + partial OpenConfig | Cisco-native (different from XE) + OpenConfig | Juniper-native + partial OpenConfig | Nokia-native + limited OpenConfig | OpenConfig-first + Arista extensions |
| Candidate datastore | Yes (IOS-XE 16.x+) | Yes | Yes (always) | Yes | No (direct apply) |
| Confirmed commit | Buggy on some versions | Yes | Yes (robust) | Yes | No |
| `<edit-config>` merge behavior | Replace semantics vary by model | Consistent | Well-defined | Consistent | N/A (gNMI preferred) |
| Error reporting | Varies by subsystem | Structured | Structured | Structured | gNMI preferred |
| Subscription support | On-change limited | Yes (dial-in + dial-out) | Limited | Yes | gNMI preferred |

### 2.2 Recommended Approach: Adapter-per-Vendor-Family

Instead of one NETCONF adapter, implement:

```
adapters/
  netconf-iosxe/      # Cisco IOS-XE (uses Cisco-native YANG + OpenConfig)
  netconf-iosxr/      # Cisco IOS-XR (different Cisco YANG models)
  netconf-junos/      # Juniper (Juniper-native YANG)
  netconf-nokia/      # Nokia SR OS
  gnmi-openconfig/    # Arista EOS, SONiC (OpenConfig-first vendors)
```

Each adapter:
- Implements the same LOOM adapter interfaces (Connector, Executor, StateReader, etc.)
- Uses vendor-specific YANG model mappings
- Declares its actual compensation reliability (not assumed transactional)
- Ships with vendor-specific integration tests against real or containerized devices

### 2.3 Lab Validation Required

Before committing to the adapter architecture, lab-test these operations on at least 3 vendors:

| Operation | Test on IOS-XE | Test on Junos | Test on SONiC |
|-----------|---------------|---------------|---------------|
| Create VLAN | NETCONF `<edit-config>` with Cisco VLAN model | NETCONF `<edit-config>` with Juniper VLAN model | gNMI Set with OpenConfig VLAN model |
| Verified commit + rollback | `<confirmed-commit>` with timeout | `<commit confirmed>` with rollback | N/A (no candidate) — test gNMI replace |
| Read running config | `<get-config>` on running datastore | `<get-config>` on running | gNMI Get on `/` |
| Subscribe to interface state | NETCONF notification stream | NETCONF notification stream | gNMI Subscribe on-change |

**Minimum lab equipment:**
- 1x Cisco IOS-XE device (or CML/VIRL virtual)
- 1x Juniper vMX or vSRX (or Containerized Routing Engine)
- 1x SONiC virtual switch (GNS3 or KVM)

### 2.4 Open Questions

- [ ] Should LOOM target OpenConfig-only for network adapters, accepting reduced feature coverage on vendors with partial OpenConfig support?
- [ ] Can containerized network OS images (Cisco CML, Juniper vLabs, SONiC-VS) serve as integration test targets in CI?
- [ ] What is the effort to build one vendor-specific NETCONF adapter vs. the original plan of one universal adapter?
- [ ] Should Nokia SR OS be deferred to Phase 7+ given its limited market share outside telco?

---

## 3. Redfish OEM Extension Divergence

### 3.1 Standard vs. OEM Coverage

Redfish (DMTF standard) provides a common schema for server management, but real-world operations depend heavily on OEM extensions:

| Operation | Standard Redfish | Dell OEM Required? | HPE OEM Required? | Supermicro OEM? |
|-----------|-----------------|-------------------|-------------------|-----------------|
| Power on/off | Yes (`Actions/ComputerSystem.Reset`) | No | No | No |
| Read sensors | Yes (`/Chassis/Thermal`, `/Power`) | No | No | Quirks |
| Set boot device | Yes (`Boot` property) | No | No | Vendor-specific paths |
| BIOS config | Partial (`/Systems/{id}/Bios`) | Yes (Lifecycle Controller) | Yes (Bios/Settings) | Different attribute names |
| Firmware update | Yes (`UpdateService`) | Yes (DUP packages) | Yes (iLO-specific firmware format) | Different async behavior |
| Virtual media | Yes (`VirtualMedia`) | Minor OEM | Minor OEM | Buggy on some firmware |
| Storage config | Partial (`Storage`) | Yes (BOSS cards, PERC RAID) | Yes (Smart Array, Smart Storage) | Limited |
| Network config | Partial (`EthernetInterfaces`) | Yes (iDRAC network) | Yes (iLO network) | Quirks |

### 3.2 Recommended Approach: Layered Adapter

```go
// Core Redfish adapter (standard schema only)
type RedfishAdapter struct {
    client  *gofish.APIClient
    vendor  VendorExtension  // pluggable OEM layer
}

// Vendor extension interface
type VendorExtension interface {
    BIOSConfig(ctx context.Context) (map[string]string, error)
    FirmwareUpdate(ctx context.Context, pkg FirmwarePackage) (TaskID, error)
    StorageConfig(ctx context.Context) ([]StorageController, error)
    // ... vendor-specific operations
}

// Implementations
type DellExtension struct { /* iDRAC Lifecycle Controller */ }
type HPEExtension struct  { /* iLO-specific APIs */ }
type SMCExtension struct   { /* Supermicro quirks */ }
type GenericExtension struct { /* standard-only fallback */ }
```

### 3.3 Vendor Detection

Auto-detect vendor from Redfish root (`/redfish/v1/`):

```json
{
  "Vendor": "Dell",
  "Product": "Integrated Dell Remote Access Controller",
  "RedfishVersion": "1.17.0"
}
```

The adapter factory selects the appropriate `VendorExtension` based on the `Vendor` field.

### 3.4 Open Questions

- [ ] What percentage of real-world Redfish operations use OEM extensions? (Requires survey of target user base)
- [ ] Should LOOM ship with Dell + HPE OEM support at MVP, adding Supermicro/Lenovo later?
- [ ] Lab test: how many LOOM operations work with standard-only Redfish? (Power, sensors, boot — likely sufficient for MVP)

---

## 4. Adapter SDK and Developer Experience

### 4.1 Current Gap

Writing a LOOM adapter currently requires reading 6+ contract documents, understanding idempotency cache TTL rules, correctly classifying errors into 4 categories, implementing compensation metadata, and building mock servers from scratch. There is no scaffolding tool, no example walkthrough, and no conformance test suite.

### 4.2 Recommended SDK Components

**4.2.1 Scaffolding CLI**

```bash
$ loom adapter scaffold --protocol snmp --name my-snmp-adapter

Created:
  adapters/my-snmp-adapter/
    adapter.go          # Adapter struct with interface stubs
    adapter_test.go     # Test file with conformance suite import
    config.go           # AdapterConfig with protocol-specific defaults
    operations.go       # Operation type stubs
    README.md           # "Getting Started" guide
    testdata/           # Sample device responses for mock testing
```

**4.2.2 Conformance Test Suite**

```go
import "github.com/wittedinit/loom/sdk/conformance"

func TestMyAdapter(t *testing.T) {
    adapter := NewMyAdapter(testConfig)
    conformance.RunAll(t, adapter, conformance.Options{
        // Which interface families to test
        Connector:  true,
        Discoverer: true,
        Executor:   true,
        StateReader: true,
        Watcher:    false, // not implemented yet
    })
}
```

The conformance suite validates:
- Error types are correctly classified (transient vs permanent vs auth vs validation)
- Idempotency keys are honored (same key = same result)
- Compensation operations are declared for all mutating operations
- Context cancellation is respected
- Timeouts are honored
- Resource cleanup on disconnect

**4.2.3 Mock Device Framework**

```go
import "github.com/wittedinit/loom/sdk/mockdevice"

// Protocol-specific mock servers
snmpDevice := mockdevice.NewSNMP(mockdevice.SNMPConfig{
    Community: "public",
    MIBs:      mockdevice.StandardMIBs,
})

redfishDevice := mockdevice.NewRedfish(mockdevice.RedfishConfig{
    Vendor: "Dell",
    Model:  "PowerEdge R750",
    InitialState: mockdevice.PoweredOn,
})
```

**4.2.4 Example Adapter Walkthrough**

A documented, 200-line example adapter (e.g., a simple HTTP health-check adapter) that walks through:
1. Implementing Connector (connect, disconnect, ping)
2. Implementing Discoverer (probe, classify)
3. Implementing Executor (single operation with compensation)
4. Using the conformance suite
5. Registering in the adapter factory

### 4.3 Open Questions

- [ ] Should the SDK be a separate Go module (`github.com/wittedinit/loom-sdk`) for dependency isolation?
- [ ] Should adapter conformance tests run against mock devices only, or also support real device targets?
- [ ] What is the minimum viable SDK for Phase 2? (Proposal: scaffolding + conformance + 1 mock server)

---

## 5. Device Rate Limiting

### 5.1 The Problem

No rate limiting is defined at the adapter level. With 10 workers running 50 concurrent activities each, all targeting the same switch, that device receives 500 concurrent management API calls. Many network devices have limited management plane CPU:

| Device Type | Management Plane Limit | Typical Concurrent Sessions |
|-------------|----------------------|---------------------------|
| Cisco IOS-XE (Catalyst 9K) | Single core | ~50-100 SSH, ~100 RESTCONF |
| Arista EOS | Shared with control plane | ~100 eAPI |
| Juniper Junos | RE CPU shared | ~75 NETCONF |
| Dell iDRAC 9 | ARM Cortex-A9 | ~4-8 concurrent Redfish |
| HPE iLO 6 | Limited | ~8-10 concurrent |
| Supermicro BMC | Very limited | ~2-4 concurrent |

BMCs are particularly fragile — 4 concurrent Redfish sessions can exhaust a Dell iDRAC, causing it to drop all management connections including human operators.

### 5.2 Recommended Design

```go
// Per-device rate limiter, coordinated across all workers
type DeviceRateLimiter struct {
    // Configured per device type (discovered via adapter)
    MaxConcurrent   int           // e.g., 4 for iDRAC, 50 for IOS-XE
    MaxPerSecond    float64       // e.g., 2/sec for BMC, 20/sec for switch
    BurstSize       int           // e.g., 2 for BMC, 10 for switch
    BackoffOnError  time.Duration // e.g., 5s on timeout, 30s on rate-limit response
}

// Rate limit profiles (built-in defaults, overridable per device)
var DefaultProfiles = map[string]DeviceRateLimiter{
    "bmc-dell-idrac":     {MaxConcurrent: 4, MaxPerSecond: 2, BurstSize: 2},
    "bmc-hpe-ilo":        {MaxConcurrent: 8, MaxPerSecond: 4, BurstSize: 4},
    "bmc-supermicro":     {MaxConcurrent: 2, MaxPerSecond: 1, BurstSize: 1},
    "switch-cisco-iosxe": {MaxConcurrent: 50, MaxPerSecond: 20, BurstSize: 10},
    "switch-arista-eos":  {MaxConcurrent: 50, MaxPerSecond: 20, BurstSize: 10},
    "switch-juniper":     {MaxConcurrent: 40, MaxPerSecond: 15, BurstSize: 8},
}
```

**Coordination across workers:**
- Rate limiter state stored in Valkey (distributed token bucket per device)
- Workers acquire a token before calling an adapter operation
- Token includes a lease timeout (if worker crashes, token is auto-released)
- Adapter operations return `RATE_LIMITED` error when device responds with 429 or equivalent

### 5.3 Open Questions

- [ ] Should rate limits be auto-discovered from device capabilities, or always manually configured?
- [ ] Should LOOM have a "safe mode" that halves all rate limits during incidents?
- [ ] How does rate limiting interact with Temporal activity timeouts? (Activity may time out waiting for a rate limit token)

---

## 6. RawConfigPush Security

### 6.1 The Problem

`RawConfigPush.ConfigBody` is a freeform string sent directly to a device with no validation, sanitization, or content analysis. An operator can push any config: backdoor accounts, open management ports, disable authentication, modify routing.

### 6.2 Recommended Mitigations

| Mitigation | Effort | Effectiveness |
|-----------|--------|---------------|
| **Content analysis rules** — Regex patterns for dangerous config (e.g., `permit ip any any`, `no enable secret`, `username.*privilege 15`) | Low | Medium — bypassable with obfuscation |
| **Mandatory approval gate** — RawConfigPush requires `admin` role and a second approver | Low | High — procedural control |
| **Pre/post diff** — Capture running config before push, diff after, store for audit | Medium | High — visibility into actual changes |
| **LLM review** — Optional: send config to LLM for security anti-pattern detection before execution | Medium | Medium — advisory, not blocking |
| **Restricted mode** — Allow RawConfigPush only for devices where no structured adapter exists | Low | High — incentivizes adapter development |

### 6.3 Open Questions

- [ ] Should RawConfigPush be disabled by default and require explicit opt-in per tenant?
- [ ] Should the pre/post diff be automatically reviewed by the verification pipeline?
- [ ] Is LLM review of raw config practical given latency constraints?

---

## 7. Adapter Plugin Trust Boundary

### 7.1 The Problem

go-plugin adapters are separate processes communicating over local gRPC. A malicious plugin could: fabricate discovery results, exfiltrate credentials, connect to arbitrary network endpoints, or consume excessive resources.

### 7.2 Recommended Mitigations

| Mitigation | Effort | Phase |
|-----------|--------|-------|
| **Plugin signing** — Only load plugins with valid signatures (Ed25519, same key infrastructure as agent updates) | Medium | 2 |
| **Credential use monitoring** — Track how long a plugin holds a decrypted credential; alert if > TTL | Low | 2 |
| **Network policy** — Plugins run in a network namespace with only declared endpoints reachable | High | 5+ |
| **Resource limits** — cgroups v2 limits per plugin process (CPU, memory, open FDs, goroutines) | Medium | 3 |
| **Behavioral baseline** — Monitor operation patterns; alert on anomalies (e.g., plugin suddenly discovers 1000x more devices) | High | 5+ |

### 7.3 Open Questions

- [ ] Is plugin isolation worth the operational complexity for Phase 1-5, or should all adapters be compiled in?
- [ ] Should plugins have a "capability declaration" that the host verifies at load time?
- [ ] Can gVisor or Firecracker provide stronger isolation without excessive overhead?

---

## 8. MVP Validation Scale

### 8.1 The Problem

100 devices does not exercise architectural boundaries:
- 100 concurrent SSH sessions is trivial
- Database query performance with 100 devices is unmeasurable
- Identity merge/split with realistic duplicate rates needs 500+ devices
- Network device management at scale needs 50+ switches with 100+ VLANs each

### 8.2 Recommended Approach: Device Simulator

Build a device simulator early (Phase 1B) that provides synthetic load:

```go
// Simulates N devices with configurable protocol responses
simulator := devicesim.New(devicesim.Config{
    DeviceCount:    1000,
    Protocols:      []string{"ssh", "redfish", "snmp"},
    FailureRate:    0.05,  // 5% of operations fail randomly
    LatencyProfile: devicesim.WAN, // realistic WAN latencies
    DuplicateRate:  0.10,  // 10% of devices share identity attributes
})
```

### 8.3 Recommended Scale Targets

| Phase | Target | Validates |
|-------|--------|-----------|
| MVP (Phase 5) | 100 real + 1,000 simulated | Connection pooling, DB performance, identity matching |
| Production (Phase 8) | 500 real + 5,000 simulated | Multi-tenant isolation, NATS throughput, rate limiting |
| Scale (Phase 10) | 1,000 real + 10,000 simulated | HA failover, edge agent sync, cost engine |

### 8.4 Open Questions

- [ ] Can the device simulator double as the mock device framework for adapter testing?
- [ ] What p99 latency targets should be defined for discovery, state read, and config push at each scale tier?
- [ ] Should scale testing be automated in CI or run as periodic benchmarks?

---

## 9. Implementation Priority

| Item | Phase | Effort | Dependencies |
|------|-------|--------|--------------|
| Adapter SDK (scaffolding + conformance) | 1B | 3-4 weeks | Adapter contract finalized |
| Device rate limiter (Valkey-backed) | 2 | 2-3 weeks | Valkey deployment |
| NETCONF vendor lab testing | 2 | 2-3 weeks | Lab equipment |
| Redfish layered adapter (standard + Dell OEM) | 2 | 3-4 weeks | Lab equipment |
| Device simulator | 1B-2 | 2-3 weeks | Adapter interfaces |
| RawConfigPush approval gate | 3 | 1 week | RBAC implementation |
| Plugin signing and loading | 3+ | 2-3 weeks | Signing infrastructure |
| gNMI vendor capability matrix | 7 | 2-3 weeks | Lab equipment |

---

## 10. References

- ADAPTER-CONTRACT.md — Adapter interface definitions
- OPERATION-TYPES.md — Operation type taxonomy
- TESTING-STRATEGY.md — Current testing approach
- research/09-remote-management-protocols.md — Protocol research
- research/10-network-os-switch-platforms.md — NOS research
- ADR-006 — Adapter families decision
- ADR-012 — Plugin architecture (mentioned, not specified)
