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

Adapters maintain an idempotency cache with a configurable TTL. Idempotency cache TTL MUST be at least 2x the maximum operation timeout. Default: idempotency keys are cached for 24 hours. If the TTL is shorter than the operation timeout, the same operation could execute twice.

The orchestrator is responsible for generating unique keys and retrying with the same key on transient failures.

### Retry Ownership Rule

**Adapters MUST NOT retry.** Retry logic is exclusively owned by the Temporal activity retry policy (defined in ERROR-MODEL.md). Adapters perform exactly ONE attempt per call and return errors immediately. A `TransientError` return signals the workflow layer to retry according to its retry policy.

This prevents retry amplification: if adapters retried 3 times internally and Temporal retried 5 times externally, a single failure could cause up to 15 actual attempts against the target device. Instead, Temporal controls the total retry count, backoff, and timeout budget for the entire operation.

---

## Compensatability Contract

Every `Executor` must declare for each operation type whether it is compensatable, and if so, what the reverse operation is.

```go
// CompensationReliability declares how reliably an adapter can undo a mutation.
// This is per-adapter (not per-operation) because reliability is a property of
// the protocol and device, not the logical operation.
type CompensationReliability string

const (
    // CompReliabilityTransactional — the device supports atomic rollback.
    // Examples: NETCONF candidate/running with confirmed-commit, Junos rollback,
    // vSphere snapshot-revert, Proxmox snapshot-revert.
    // Compensation is guaranteed to restore exact pre-op state.
    CompReliabilityTransactional CompensationReliability = "transactional"

    // CompReliabilitySnapshotRestore — the adapter takes a full config snapshot
    // before any mutation and can restore it on failure. Not atomic, but the
    // restore target is the exact pre-op config, not a generic reverse.
    // Examples: SSH adapter dumps running-config before change, restores on failure.
    CompReliabilitySnapshotRestore CompensationReliability = "snapshot_restore"

    // CompReliabilityBestEffort — the adapter can attempt a reverse operation,
    // but there is no guarantee it restores the exact pre-op state.
    // Examples: IPMI power-off to compensate power-on (but original state may
    // have been "on"), eAPI VLAN delete to compensate VLAN create.
    CompReliabilityBestEffort CompensationReliability = "best_effort"

    // CompReliabilityNone — no programmatic undo is possible. The device or
    // protocol does not support rollback, snapshot, or reliable reverse operations.
    // Examples: MikroTik REST (no transactions, server-assigned IDs),
    // SSH raw command execution, legacy Cisco IOS without archive/rollback.
    // Workflows targeting these adapters MUST flag this before execution and
    // require explicit human acknowledgment.
    CompReliabilityNone CompensationReliability = "none"
)

// CompensationInfo describes how to reverse an operation.
type CompensationInfo struct {
    Reversible     bool
    CompensationOp OperationType           // the reverse operation type, or "" if not reversible
    Reliability    CompensationReliability  // how reliably this compensation restores pre-op state
}
```

### Pre-Operation Snapshots

Adapters that support `CompReliabilitySnapshotRestore` or `CompReliabilityTransactional` must implement the `SnapshotCapable` interface. This is checked at workflow planning time.

```go
// SnapshotCapable is implemented by adapters that can capture device config
// state before a mutation. The orchestrator calls TakeSnapshot before any
// mutating operation and stores the result for potential restoration.
type SnapshotCapable interface {
    // TakeSnapshot captures the current config state of the target.
    // The returned ConfigSnapshot is opaque to the orchestrator — only the
    // originating adapter knows how to interpret and restore it.
    TakeSnapshot(ctx context.Context, scope SnapshotScope) (*ConfigSnapshot, error)

    // RestoreSnapshot restores the device to a previously captured state.
    // This is called during saga compensation when the adapter's
    // CompensationReliability is snapshot_restore or transactional.
    RestoreSnapshot(ctx context.Context, snapshot *ConfigSnapshot) error
}

// SnapshotScope defines what to capture.
type SnapshotScope struct {
    ResourceRefs []ResourceRef // specific resources, or empty for full config
    Format       string        // "running_config", "candidate_config", "full_state"
}

// ConfigSnapshot is the captured state. Opaque to the orchestrator.
type ConfigSnapshot struct {
    AdapterName string    // which adapter produced this snapshot
    DeviceID    string    // LOOM internal device ID
    Scope       SnapshotScope
    Data        []byte    // adapter-specific serialized state
    Format      string    // "text/plain", "application/json", "application/xml"
    Checksum    string    // SHA-256 of Data for integrity verification
    CapturedAt  time.Time
}
```

### Workflow Behavior for CompensationReliability = None

When a workflow targets a device whose adapter declares `CompReliabilityNone`:

1. **Pre-execution check**: The workflow planner inspects all target adapters. If any adapter has `CompReliabilityNone`, the workflow is flagged as `requires_acknowledgment`.
2. **Human acknowledgment required**: The workflow will not execute until the operator explicitly acknowledges: "I understand that rollback is not possible for device X. Manual intervention may be required if this workflow fails."
3. **Acknowledgment is a Temporal signal**, consistent with the approval gate pattern in WORKFLOW-CONTRACT.md.
4. **On failure**: The workflow enters `compensation_partial` state. Steps targeting adapters with reliable compensation are rolled back normally. Steps targeting `CompReliabilityNone` adapters are logged with full context (what was changed, what the pre-op state was if a snapshot was taken manually, and what the operator needs to do to restore it).

**Examples:**

| Operation | Compensation | Reliability |
|-----------|-------------|-------------|
| PowerOnOp | PowerOffOp | best_effort (original state unknown) |
| PowerOffOp | PowerOnOp | best_effort (original state unknown) |
| CreateVLANOp (NETCONF) | DeleteVLANOp | transactional (candidate commit) |
| CreateVLANOp (eAPI) | DeleteVLANOp | best_effort (no atomic rollback) |
| CreateVLANOp (MikroTik) | manual delete by `.id` | none (server-assigned IDs) |
| ConfigurePortOp (NETCONF) | RestoreSnapshot | snapshot_restore |
| ConfigurePortOp (SSH) | RestoreSnapshot | snapshot_restore (if config dump supported) |
| DeleteVLANOp | CreateVLANOp (with original params) | best_effort |
| ExecuteCommandOp (SSH) | CompensationNone | none |
| ReadSensorsOp | CompensationNone (read-only) | n/a |
| ReadInterfacesOp | CompensationNone (read-only) | n/a |

The orchestrator uses compensation info to build rollback plans when multi-step workflows fail partway through.

---

## Which Adapters Implement Which Families

| Adapter | Connector | Discoverer | Executor | StateReader | Watcher | Compensation Reliability | SnapshotCapable |
|---------|-----------|------------|----------|-------------|---------|--------------------------|-----------------|
| SSH | Yes | Yes | Yes (commands) | No | No | none (raw commands) / snapshot_restore (config backup) | Yes (config dump) |
| Redfish | Yes | Yes | Yes (power, boot) | Yes | No | best_effort | No |
| SNMP | Yes | Yes | No (read-only) | Yes | No | n/a (read-only) | No |
| IPMI | Yes | No | Yes (power, boot) | Yes (sensors) | No | best_effort | No |
| NETCONF | Yes | Yes | Yes (config) | Yes | No | transactional (candidate commit) | Yes |
| gNMI | Yes | Yes | Yes (config) | Yes | Yes (telemetry) | transactional (OpenConfig) | Yes |
| eAPI | Yes | Yes | Yes (config) | Yes | No | best_effort | Yes (show running) |
| NX-API | Yes | Yes | Yes (config) | Yes | No | best_effort | Yes (show running) |
| AMT | Yes | No | Yes (power, boot, KVM) | Yes | No | best_effort | No |
| PiKVM | Yes | No | Yes (power, HID) | Yes (screenshot) | No | none | No |
| vSphere | Yes | Yes | Yes (VM lifecycle) | Yes | Yes (events) | transactional (snapshots) | Yes |
| Proxmox | Yes | Yes | Yes (VM/CT lifecycle) | Yes | No | transactional (snapshots) | Yes |
| libvirt | Yes | Yes | Yes (domain lifecycle) | Yes | Yes (events) | transactional (snapshots) | Yes |

**Key observations:**
- Every adapter implements `Connector` — no exceptions.
- SNMP is read-only: it implements `Discoverer` and `StateReader` but not `Executor`. Compensation reliability is not applicable.
- Only gNMI, vSphere, and libvirt implement `Watcher` — they have native streaming support.
- IPMI and AMT skip `Discoverer` — they are control-plane protocols, not inventory sources.
- PiKVM is a special case: it provides HID injection and screenshot capture, not traditional network management.
- NETCONF and gNMI have `transactional` compensation — they support candidate configs and confirmed commits.
- SSH compensation depends on the device: if the device supports `show running-config` or equivalent, the SSH adapter can take a config snapshot. Otherwise, compensation is `none`.
- vSphere, Proxmox, and libvirt have `transactional` compensation via VM/container snapshots.
- Adapters with `best_effort` compensation (Redfish, IPMI, eAPI, NX-API, AMT) can attempt reverse operations but cannot guarantee exact pre-op state restoration.

---

## Config Translation Strategy

> Addresses ADVERSARIAL-REVIEW.md Finding 2.2 and 4.3: Multi-vendor config translation is a research problem, not an engineering task. LOOM uses vendor-native templates, not universal translation.

### Approach: Intent + Vendor-Native Templates

LOOM does NOT attempt live cross-vendor config translation (e.g., Cisco ACL to Junos firewall filter to Arista ACL). That is an unsolved research problem. Instead:

1. **LOOM stores intent in a canonical format.** Intent describes WHAT should happen, not HOW.
2. **Each adapter has vendor-specific templates** that render the intent into native device syntax.
3. **Templates are validated, tested, cached, and version-controlled.** They are not generated at runtime by the LLM.

```go
// ConfigIntent is a vendor-neutral description of a desired configuration change.
// It describes WHAT, not HOW. Each adapter's template engine renders it into
// native syntax for the target device.
type ConfigIntent struct {
    IntentType  string         `json:"intent_type"`  // "acl_rule", "vlan_create", "interface_config", "static_route"
    Parameters  map[string]any `json:"parameters"`   // intent-specific typed parameters
    Description string         `json:"description"`  // human-readable: "block TCP 80 from 10.0.0.0/8"
    Priority    int            `json:"priority"`     // ordering hint for order-dependent configs (ACLs)
}

// VendorTemplate renders a ConfigIntent into vendor-native syntax.
type VendorTemplate struct {
    ID           string    `json:"id"`            // "cisco_ios_acl_deny", "junos_firewall_filter_term"
    IntentType   string    `json:"intent_type"`   // which ConfigIntent.IntentType this handles
    Vendor       string    `json:"vendor"`        // "cisco_ios", "junos", "arista_eos", "sonic"
    TemplateBody string    `json:"template_body"` // Go template or Jinja2 template
    Version      string    `json:"version"`       // semver for the template
    Validated    bool      `json:"validated"`      // has this template been tested against a real device?
    ValidatedAt  time.Time `json:"validated_at"`
    Checksum     string    `json:"checksum"`       // SHA-256 of TemplateBody
}

// TemplateRegistry stores and retrieves vendor templates.
type TemplateRegistry interface {
    // Lookup finds the best template for a given intent type and vendor.
    Lookup(intentType string, vendor string) (*VendorTemplate, error)

    // Register adds or updates a template. Requires validation before production use.
    Register(template VendorTemplate) error

    // ListByVendor returns all templates for a vendor, grouped by intent type.
    ListByVendor(vendor string) ([]VendorTemplate, error)
}
```

### LLM Role in Template Generation

The LLM can assist with **generating new templates**, but templates are never used in production without validation:

1. Operator requests a new intent type for a vendor (e.g., "QoS policy for Arista EOS").
2. LLM generates a candidate template based on vendor documentation (RAG-assisted).
3. Template is syntax-checked against the vendor's config grammar (where available).
4. Template is tested against a lab device or config simulator.
5. Template is committed to the template registry with `Validated: true`.
6. Only validated templates are used in production workflows.

### MVP Path: Raw Config Push

For MVP (Phases 0-5), the primary config path is **raw config push**: the operator provides vendor-native config, and LOOM pushes it to the device via the appropriate adapter. No translation, no templates.

```go
// RawConfigPush is the MVP escape hatch. The operator provides vendor-native
// config and LOOM pushes it without translation.
type RawConfigPush struct {
    DeviceID     string `json:"device_id"`
    ConfigFormat string `json:"config_format"` // "cli_commands", "xml", "json", "set_commands"
    ConfigBody   string `json:"config_body"`   // the actual vendor-native config
    DryRun       bool   `json:"dry_run"`       // if true, validate but do not apply
}
```

### What Is NOT Supported

- No live Cisco-to-Junos config translation.
- No universal ACL model that covers stateful firewalls, object groups, NAT, QoS, or time-based rules. The `SetACLOp` in OPERATION-TYPES.md covers basic permit/deny only. Complex policies use `RawConfigPush`.
- No universal routing protocol model. BGP, OSPF, and IS-IS have vendor-specific knobs that matter in production. Use `RawConfigPush` for these.
- Template library grows over time via validated, tested, version-controlled contributions. Coverage is explicitly tracked per vendor per intent type.
