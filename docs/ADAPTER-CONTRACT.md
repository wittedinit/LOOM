# LOOM Adapter Contract

Adapters implement only the operation families they support. No pretending SNMP, SSH, Redfish, PiKVM, and gNMI all behave the same. Each protocol has different strengths; the adapter model reflects that.

---

## Operation Families (Go Interfaces)

### Connector (Required)

Every adapter must implement `Connector`. It manages the connection lifecycle.

```go
// Connector is required for ALL adapters. Manages connection lifecycle.
type Connector interface {
    Connect(ctx context.Context, endpoint EndpointInfo, cred CredentialRef) error
    Disconnect(ctx context.Context) error
    Ping(ctx context.Context) error
    Connected() bool
}
```

### Discoverer (Optional)

Discovers facts about a target — hardware inventory, network topology, storage layout.

```go
// Discoverer discovers facts about a target. Optional.
type Discoverer interface {
    Discover(ctx context.Context, scope DiscoveryScope) (*DiscoveryResult, error)
}
```

### Executor (Optional)

Executes typed operations against a target. Each operation has a typed request and typed response.

```go
// Executor executes typed operations. Optional.
type Executor interface {
    Execute(ctx context.Context, op TypedOperation) (*OperationResult, error)
    SupportedOperations() []OperationType
}
```

### StateReader (Optional)

Reads the current state of a resource. Point-in-time snapshot, not streaming.

```go
// StateReader reads current state of a resource. Optional.
type StateReader interface {
    ReadState(ctx context.Context, resourceRef ResourceRef) (*StateSnapshot, error)
}
```

### Watcher (Optional)

Streams state changes over time. Only for protocols that natively support streaming (gNMI telemetry, vSphere events, libvirt domain events).

```go
// Watcher streams state changes. Optional — only for protocols that support streaming.
type Watcher interface {
    Watch(ctx context.Context, resourceRef ResourceRef) (<-chan StateEvent, error)
    StopWatch(ctx context.Context) error
}
```

---

## Supporting Types

```go
// EndpointInfo identifies a target to connect to.
type EndpointInfo struct {
    Address   string            // hostname, IP, or URI
    Port      int               // protocol-specific port
    Transport string            // "tcp", "tls", "ssh", "https"
    Metadata  map[string]string // protocol-specific connection params
}

// CredentialRef is a reference to credentials stored in the credential vault.
// Adapters never hold raw secrets — they receive a ref and resolve it at connect time.
type CredentialRef struct {
    VaultPath string
    Type      string // "basic", "ssh_key", "token", "certificate", "snmp_v3"
}

// DiscoveryScope defines what to discover.
type DiscoveryScope struct {
    Categories []string // "hardware", "network", "storage", "firmware", "all"
    Depth      int      // how deep to recurse (e.g., chassis → blades → components)
    Filter     string   // optional filter expression
}

// DiscoveryResult holds everything discovered about a target.
type DiscoveryResult struct {
    Identities []ExternalIdentity // serial numbers, MAC addresses, asset tags
    Facts      []ObservedFact     // hardware model, firmware version, interface count
    Timestamp  time.Time
}

// ExternalIdentity is a unique identifier observed on a device.
type ExternalIdentity struct {
    Type  string // "serial_number", "mac_address", "asset_tag", "uuid"
    Value string
    Scope string // where this identity was found ("bmc", "chassis", "os")
}

// ObservedFact is a piece of information discovered about a device.
type ObservedFact struct {
    Category  string // "hardware", "network", "storage", "firmware"
    Key       string // "model", "cpu_count", "total_memory_gb", "firmware_version"
    Value     any
    Source    string // which adapter/protocol observed this
    Timestamp time.Time
}

// ResourceRef identifies a specific resource on a target.
type ResourceRef struct {
    DeviceID     string // LOOM internal device ID
    ResourceType string // "power", "interface", "sensor", "vlan", "vm", "container"
    ResourceID   string // protocol-specific identifier (e.g., "eth0", "Sensor/Temp/CPU0")
}

// StateSnapshot is a point-in-time state reading.
type StateSnapshot struct {
    ResourceRef ResourceRef
    State       map[string]any
    Timestamp   time.Time
    Source      string // adapter that produced this snapshot
}

// StateEvent is a streaming state change from a Watcher.
type StateEvent struct {
    ResourceRef ResourceRef
    EventType   string // "update", "delete", "create"
    OldValue    any
    NewValue    any
    Timestamp   time.Time
    Sequence    uint64 // monotonically increasing per watch session
}
```

---

## Adapter Registration

Adapters are registered at startup via a registry. The registry enables lookup by protocol, capability, or device type.

```go
// AdapterRegistration declares what an adapter provides.
type AdapterRegistration struct {
    Name         string
    Protocol     string
    Capabilities []Capability
    Factory      func(config AdapterConfig) Adapter
}

// AdapterConfig is passed to the factory at construction time.
type AdapterConfig struct {
    Protocol string
    Options  map[string]any
    Logger   *slog.Logger
}

// Adapter is the base type — every adapter is at least a Connector.
// It may also implement Discoverer, Executor, StateReader, Watcher.
type Adapter interface {
    Connector
    Registration() AdapterRegistration
}
```

### Registry Operations

```go
// Registry stores adapter registrations and supports lookup.
type Registry interface {
    Register(reg AdapterRegistration) error
    ByProtocol(protocol string) (AdapterRegistration, bool)
    ByCapability(capability string) []AdapterRegistration
    All() []AdapterRegistration
}
```

---

## Capability Declaration

Each adapter declares what it can do at registration time. Capabilities are concrete, not abstract.

```go
// Capability declares a single thing an adapter can do.
type Capability struct {
    Name     string // "power_control", "discover_hardware", "config_read", "config_write",
                    // "watch_interfaces", "boot_control", "console"
    Protocol string
    ReadOnly bool
}
```

Examples:
- SNMP declares `{"discover_hardware", "snmp", true}` and `{"config_read", "snmp", true}` — it is read-only.
- Redfish declares `{"power_control", "redfish", false}` and `{"boot_control", "redfish", false}` — it can mutate.
- gNMI declares `{"watch_interfaces", "gnmi", true}` — it can stream telemetry.

---

## Error Classification

All adapter errors must be classified. No raw `error` returns. Every error carries context for the orchestrator to decide what to do.

```go
// TransientError — retry with backoff.
type TransientError struct {
    Code          string
    Message       string
    Adapter       string
    Operation     string
    CorrelationID string
    RetryAfter    time.Duration
}

// PermanentError — do not retry. The operation fundamentally cannot succeed.
type PermanentError struct {
    Code          string
    Message       string
    Adapter       string
    Operation     string
    CorrelationID string
}

// PartialError — some operations succeeded, some failed. Compensation may be needed.
type PartialError struct {
    Code          string
    Message       string
    Adapter       string
    Operation     string
    CorrelationID string
    Succeeded     []string // operation IDs that completed
    Failed        []string // operation IDs that failed
}

// ValidationError — input rejected before execution. Never reaches the adapter.
type ValidationError struct {
    Code          string
    Message       string
    Adapter       string
    Operation     string
    CorrelationID string
    Field         string // which field failed validation
    Constraint    string // what the constraint was
}
```

Every error implements the `error` interface and carries: `Code`, `Message`, `Retryable() bool`, `Adapter`, `Operation`, `CorrelationID`.

---

## Idempotency Contract

Every operation carries an `IdempotencyKey` (UUID). Adapters must honor it.

**Rules:**

1. If the same `IdempotencyKey` is submitted and the operation already completed: return the previous result. Do not re-execute.
2. If the same `IdempotencyKey` is submitted and the operation is in progress: wait for completion and return the result.
3. If the `IdempotencyKey` has never been seen: execute normally.

Adapters maintain an idempotency cache with a configurable TTL. The orchestrator is responsible for generating unique keys and retrying with the same key on transient failures.

---

## Compensatability Contract

Every `Executor` must declare for each operation type whether it is compensatable, and if so, what the reverse operation is.

```go
// CompensationInfo describes how to reverse an operation.
type CompensationInfo struct {
    Reversible     bool
    CompensationOp OperationType // the reverse operation type, or "" if not reversible
}
```

**Examples:**

| Operation | Compensation |
|-----------|-------------|
| PowerOnOp | PowerOffOp |
| PowerOffOp | PowerOnOp |
| CreateVLANOp | DeleteVLANOp |
| DeleteVLANOp | CreateVLANOp (with original params) |
| ConfigurePortOp | ConfigurePortOp (with previous config) |
| ExecuteCommandOp | CompensationNone (commands are not generically reversible) |
| ReadSensorsOp | CompensationNone (read-only) |
| ReadInterfacesOp | CompensationNone (read-only) |

The orchestrator uses compensation info to build rollback plans when multi-step workflows fail partway through.

---

## Which Adapters Implement Which Families

| Adapter | Connector | Discoverer | Executor | StateReader | Watcher |
|---------|-----------|------------|----------|-------------|---------|
| SSH | Yes | Yes | Yes (commands) | No | No |
| Redfish | Yes | Yes | Yes (power, boot) | Yes | No |
| SNMP | Yes | Yes | No (read-only) | Yes | No |
| IPMI | Yes | No | Yes (power, boot) | Yes (sensors) | No |
| NETCONF | Yes | Yes | Yes (config) | Yes | No |
| gNMI | Yes | Yes | Yes (config) | Yes | Yes (telemetry) |
| eAPI | Yes | Yes | Yes (config) | Yes | No |
| NX-API | Yes | Yes | Yes (config) | Yes | No |
| AMT | Yes | No | Yes (power, boot, KVM) | Yes | No |
| PiKVM | Yes | No | Yes (power, HID) | Yes (screenshot) | No |
| vSphere | Yes | Yes | Yes (VM lifecycle) | Yes | Yes (events) |
| Proxmox | Yes | Yes | Yes (VM/CT lifecycle) | Yes | No |
| libvirt | Yes | Yes | Yes (domain lifecycle) | Yes | Yes (events) |

**Key observations:**
- Every adapter implements `Connector` — no exceptions.
- SNMP is read-only: it implements `Discoverer` and `StateReader` but not `Executor`.
- Only gNMI, vSphere, and libvirt implement `Watcher` — they have native streaming support.
- IPMI and AMT skip `Discoverer` — they are control-plane protocols, not inventory sources.
- PiKVM is a special case: it provides HID injection and screenshot capture, not traditional network management.
