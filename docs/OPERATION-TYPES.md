# LOOM Operation Types

Every executor operation has a TYPED request struct and TYPED response struct. No `map[string]any`. No generic payloads. The type system enforces correctness at compile time and validation enforces it at dispatch time.

---

## Base Operation Type

```go
// TypedOperation is the envelope for every operation dispatched to an adapter.
type TypedOperation struct {
    Type           OperationType
    TargetRef      ResourceRef
    IdempotencyKey string        // UUID — see idempotency contract in ADAPTER-CONTRACT.md
    RequestedBy    string        // actor identity (user, service account, workflow engine)
    TenantID       string
    Timeout        time.Duration
    DryRun         bool
    // Params is one of the typed operation structs below.
    // MUST be one of the registered typed param structs — validated at dispatch.
    Params         any
}

// OperationResult is the typed response from every operation.
type OperationResult struct {
    OperationType  OperationType
    Status         OperationStatus // succeeded, failed, partial, dry_run_ok
    IdempotencyKey string
    StartedAt      time.Time
    CompletedAt    time.Time
    Output         any             // typed response matching the operation
    Error          error
    CompensationOp *TypedOperation // the reverse op, populated on success for compensatable operations
}

// OperationStatus enumerates the possible outcomes.
type OperationStatus string

const (
    StatusSucceeded OperationStatus = "succeeded"
    StatusFailed    OperationStatus = "failed"
    StatusPartial   OperationStatus = "partial"
    StatusDryRunOK  OperationStatus = "dry_run_ok"
)
```

---

## Typed Operation Definitions

### PowerCycleOp

Power cycle a device: on, off, cycle, or reset.

```go
type PowerCycleParams struct {
    Action           string // "on", "off", "cycle", "reset"
    GracefulShutdown bool   // attempt graceful OS shutdown before hard power action
    WaitSeconds      int    // seconds to wait between off and on during "cycle"
}

type PowerCycleResult struct {
    PreviousState string // "on", "off", "unknown"
    CurrentState  string // "on", "off", "unknown"
}
```

- **Compensation:** `PowerCycleOp` with reverse action (`"on"` compensates `"off"`, `"off"` compensates `"on"`).
- **Adapters:** Redfish, IPMI, AMT, PiKVM, vSphere, Proxmox, libvirt.

---

### SetBootDeviceOp

Set the next boot device for a machine.

```go
type SetBootDeviceParams struct {
    Device     string // "pxe", "disk", "cdrom", "bios"
    Persistent bool   // survive reboot or one-time only
}

type SetBootDeviceResult struct {
    PreviousDevice string
    CurrentDevice  string
}
```

- **Compensation:** `SetBootDeviceOp` with the previous device value.
- **Adapters:** Redfish, IPMI, AMT.

---

### ExecuteCommandOp

Run a command via SSH or CLI session.

```go
type ExecuteCommandParams struct {
    Command      string        // the command to execute
    Timeout      time.Duration // per-command timeout (distinct from operation-level timeout)
    ExpectPrompt string        // regex to match when command is "done" (for interactive sessions)
}

type ExecuteCommandResult struct {
    ExitCode int
    Stdout   string
    Stderr   string
}
```

- **Compensation:** `CompensationNone` — commands are not reversible generically.
- **Adapters:** SSH.

---

### ReadInterfacesOp

Read network interface status from a device.

```go
type ReadInterfacesParams struct {
    InterfaceFilter string // regex or glob to filter interface names (e.g., "eth*", "Ethernet1/[0-9]+")
}

type ReadInterfacesResult struct {
    Interfaces []InterfaceInfo
}
```

- **Compensation:** `CompensationNone` — read-only operation.
- **Adapters:** SNMP, NETCONF, gNMI, eAPI, NX-API.

---

### CreateVLANOp

Create a VLAN on a network device.

```go
type CreateVLANParams struct {
    VLANID int
    Name   string
    Ports  []PortAssignment
}

type CreateVLANResult struct {
    Created bool
    VLANID  int
}
```

- **Compensation:** `DeleteVLANOp` with the same VLAN ID.
- **Adapters:** NETCONF, gNMI, eAPI, NX-API.

---

### DeleteVLANOp

Delete a VLAN from a network device.

```go
type DeleteVLANParams struct {
    VLANID int
}

type DeleteVLANResult struct {
    Deleted bool
}
```

- **Compensation:** `CreateVLANOp` with the original params (VLAN ID, name, port assignments).
- **Adapters:** NETCONF, gNMI, eAPI, NX-API.

---

### ConfigurePortOp

Configure a switch port: access mode, trunk mode, VLAN assignment, description.

```go
type ConfigurePortParams struct {
    Interface   string // "Ethernet1/1", "GigabitEthernet0/1", etc.
    Mode        string // "access", "trunk"
    AccessVLAN  int    // VLAN ID for access mode
    TrunkVLANs  []int  // allowed VLANs for trunk mode
    Description string
}

type ConfigurePortResult struct {
    Interface string
    Applied   bool
}
```

- **Compensation:** `ConfigurePortOp` with the previous configuration snapshot.
- **Adapters:** NETCONF, gNMI, eAPI, NX-API.

---

### SetACLOp

Apply an access control list to an interface.

```go
type SetACLParams struct {
    Name      string    // ACL name
    Rules     []ACLRule
    Interface string    // interface to bind the ACL to
    Direction string    // "in", "out"
}

type SetACLResult struct {
    Applied    bool
    RulesCount int
}
```

- **Compensation:** `RemoveACLOp` with the same ACL name and interface.
- **Adapters:** NETCONF, gNMI, eAPI, NX-API.

---

### ReadSensorsOp

Read hardware sensors: temperature, voltage, fan speed.

```go
type ReadSensorsParams struct {
    SensorTypes []string // "temperature", "voltage", "fan", "power", or empty for all
}

type ReadSensorsResult struct {
    Sensors []SensorReading
}
```

- **Compensation:** `CompensationNone` — read-only operation.
- **Adapters:** IPMI, Redfish, SNMP.

---

## Supporting Types

```go
// ACLRule defines a single rule in an access control list.
type ACLRule struct {
    Action      string // "permit", "deny"
    Protocol    string // "tcp", "udp", "icmp", "ip", "any"
    Source      string // CIDR, "any", or named object
    Destination string // CIDR, "any", or named object
    Port        string // port number, range ("80-443"), or "any"
}

// PortAssignment maps a port to a VLAN membership mode.
type PortAssignment struct {
    Interface string // "Ethernet1/1", etc.
    Mode      string // "tagged", "untagged"
}

// InterfaceInfo holds the observed state of a network interface.
type InterfaceInfo struct {
    Name        string
    Status      string // "up", "down", "admin_down"
    Speed       string // "1G", "10G", "25G", "100G"
    MTU         int
    MacAddress  string
    IPAddresses []string // ["192.168.1.1/24", "fe80::1/64"]
}

// SensorReading holds a single sensor measurement.
type SensorReading struct {
    Name   string  // "CPU0 Temp", "PSU1 Voltage", "Fan3 Speed"
    Type   string  // "temperature", "voltage", "fan", "power"
    Value  float64
    Unit   string  // "celsius", "volts", "rpm", "watts"
    Status string  // "ok", "warning", "critical", "unknown"
}
```

---

## Validation Rules

Validation occurs BEFORE execution dispatch. Always. An invalid operation never reaches an adapter.

```go
// Validatable is implemented by every Params struct.
type Validatable interface {
    Validate() error
}
```

### What validation checks:

- **Required fields** — no zero values where a value is mandatory.
- **Valid ranges** — VLAN IDs between 1-4094, port numbers between 1-65535, wait seconds non-negative.
- **Valid enum values** — power action must be one of `"on"`, `"off"`, `"cycle"`, `"reset"`. Boot device must be one of `"pxe"`, `"disk"`, `"cdrom"`, `"bios"`. Port mode must be `"access"` or `"trunk"`. ACL direction must be `"in"` or `"out"`.
- **Structural consistency** — access mode requires `AccessVLAN`, trunk mode requires `TrunkVLANs`. ACL rules must have valid protocol values.

### Validation example:

```go
func (p PowerCycleParams) Validate() error {
    validActions := map[string]bool{"on": true, "off": true, "cycle": true, "reset": true}
    if !validActions[p.Action] {
        return &ValidationError{
            Code:       "INVALID_ACTION",
            Message:    fmt.Sprintf("action must be one of on/off/cycle/reset, got %q", p.Action),
            Field:      "Action",
            Constraint: "enum:on,off,cycle,reset",
        }
    }
    if p.Action == "cycle" && p.WaitSeconds < 0 {
        return &ValidationError{
            Code:       "INVALID_WAIT",
            Message:    "WaitSeconds must be non-negative for cycle action",
            Field:      "WaitSeconds",
            Constraint: "min:0",
        }
    }
    return nil
}
```

If validation fails, a `ValidationError` is returned and the operation never reaches the adapter.

---

## OperationType Registry

The registry maps operation type strings to their typed metadata. This enables runtime dispatch with compile-time type safety for the params and output structs.

```go
// OperationTypeInfo describes a registered operation type.
type OperationTypeInfo struct {
    Name           OperationType
    ParamsType     reflect.Type
    OutputType     reflect.Type
    Compensation   CompensationInfo
    Description    string
    ReadOnly       bool
}

// OperationTypes is the global registry of all known operation types.
var OperationTypes = map[OperationType]OperationTypeInfo{
    "power_cycle": {
        Name:       "power_cycle",
        ParamsType: reflect.TypeOf(PowerCycleParams{}),
        OutputType: reflect.TypeOf(PowerCycleResult{}),
        Compensation: CompensationInfo{Reversible: true, CompensationOp: "power_cycle"},
        Description: "Power cycle a device: on, off, cycle, or reset",
        ReadOnly:    false,
    },
    "set_boot_device": {
        Name:       "set_boot_device",
        ParamsType: reflect.TypeOf(SetBootDeviceParams{}),
        OutputType: reflect.TypeOf(SetBootDeviceResult{}),
        Compensation: CompensationInfo{Reversible: true, CompensationOp: "set_boot_device"},
        Description: "Set the next boot device",
        ReadOnly:    false,
    },
    "execute_command": {
        Name:       "execute_command",
        ParamsType: reflect.TypeOf(ExecuteCommandParams{}),
        OutputType: reflect.TypeOf(ExecuteCommandResult{}),
        Compensation: CompensationInfo{Reversible: false},
        Description: "Run a command via SSH or CLI",
        ReadOnly:    false,
    },
    "read_interfaces": {
        Name:       "read_interfaces",
        ParamsType: reflect.TypeOf(ReadInterfacesParams{}),
        OutputType: reflect.TypeOf(ReadInterfacesResult{}),
        Compensation: CompensationInfo{Reversible: false},
        Description: "Read network interface status",
        ReadOnly:    true,
    },
    "create_vlan": {
        Name:       "create_vlan",
        ParamsType: reflect.TypeOf(CreateVLANParams{}),
        OutputType: reflect.TypeOf(CreateVLANResult{}),
        Compensation: CompensationInfo{Reversible: true, CompensationOp: "delete_vlan"},
        Description: "Create a VLAN on a network device",
        ReadOnly:    false,
    },
    "delete_vlan": {
        Name:       "delete_vlan",
        ParamsType: reflect.TypeOf(DeleteVLANParams{}),
        OutputType: reflect.TypeOf(DeleteVLANResult{}),
        Compensation: CompensationInfo{Reversible: true, CompensationOp: "create_vlan"},
        Description: "Delete a VLAN from a network device",
        ReadOnly:    false,
    },
    "configure_port": {
        Name:       "configure_port",
        ParamsType: reflect.TypeOf(ConfigurePortParams{}),
        OutputType: reflect.TypeOf(ConfigurePortResult{}),
        Compensation: CompensationInfo{Reversible: true, CompensationOp: "configure_port"},
        Description: "Configure a switch port",
        ReadOnly:    false,
    },
    "set_acl": {
        Name:       "set_acl",
        ParamsType: reflect.TypeOf(SetACLParams{}),
        OutputType: reflect.TypeOf(SetACLResult{}),
        Compensation: CompensationInfo{Reversible: true, CompensationOp: "remove_acl"},
        Description: "Apply an access control list to an interface",
        ReadOnly:    false,
    },
    "read_sensors": {
        Name:       "read_sensors",
        ParamsType: reflect.TypeOf(ReadSensorsParams{}),
        OutputType: reflect.TypeOf(ReadSensorsResult{}),
        Compensation: CompensationInfo{Reversible: false},
        Description: "Read hardware sensors",
        ReadOnly:    true,
    },
}
```

### Dispatch Validation

At dispatch time, the operation type is looked up in the registry. The `Params` field is type-asserted against `ParamsType`. If it does not match, a `ValidationError` is returned before the operation ever reaches an adapter.

```go
func ValidateOperation(op TypedOperation) error {
    info, ok := OperationTypes[op.Type]
    if !ok {
        return &ValidationError{
            Code:    "UNKNOWN_OPERATION",
            Message: fmt.Sprintf("unknown operation type: %s", op.Type),
        }
    }

    // Type-check the params
    paramsType := reflect.TypeOf(op.Params)
    if paramsType != info.ParamsType {
        return &ValidationError{
            Code:    "PARAMS_TYPE_MISMATCH",
            Message: fmt.Sprintf("expected %s, got %s", info.ParamsType, paramsType),
            Field:   "Params",
        }
    }

    // Run the params' own validation
    if v, ok := op.Params.(Validatable); ok {
        return v.Validate()
    }

    return nil
}
```
