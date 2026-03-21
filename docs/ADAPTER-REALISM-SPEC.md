# Adapter Realism Specification

> **Status:** Accepted -- closes gaps C7, H8, H15, H16, H19 from GAP-ANALYSIS.md
> **Supersedes:** Open questions in RESEARCH-protocol-adapter-realism.md sections 2, 3, 5, 6, 7
> **Owner:** Protocol Engineering
> **Last updated:** 2026-03-21

---

## 1. Scope and Decisions Summary

This document makes binding design decisions for five open areas identified in the protocol adapter realism research. Each section states the decision, the rationale, and provides Go type definitions precise enough to implement from.

| Gap | Decision | Section |
|-----|----------|---------|
| C7 (NETCONF vendor fragmentation) | Adapter-per-vendor-family. Four shipping adapters, not one universal NETCONF adapter. | [2](#2-netconf-vendor-strategy) |
| H19 (Redfish OEM divergence) | Layered adapter with pluggable `VendorExtension` interface. Dell + HPE at MVP. | [3](#3-redfish-layered-architecture) |
| H8 (Device rate limiting) | Valkey-backed distributed token bucket per device with safe mode. | [4](#4-device-rate-limiting) |
| H15 (RawConfigPush security) | Disabled by default, opt-in per tenant, admin + second approver required. | [5](#5-rawconfigpush-security) |
| H16 (Plugin trust boundary) | Compiled-in through Phase 5; go-plugin with Ed25519 signing and cgroups for Phase 6+. | [6](#6-plugin-trust-boundary) |

---

## 2. NETCONF Vendor Strategy

### 2.1 Decision: Adapter-per-Vendor-Family

LOOM ships one adapter per vendor family, not a universal NETCONF adapter. A shared `netconfcore` library provides session management, XML encoding, and datastore primitives. Each vendor adapter handles YANG model mapping, compensation semantics, and vendor-specific RPC quirks.

### 2.2 Adapter Breakdown

```
internal/adapter/
  netconfcore/          # Shared library: session, transport, XML, datastore ops
  netconf-iosxe/        # Cisco IOS-XE (Catalyst 9K, ISR 4K, ASR 1K)
  netconf-iosxr/        # Cisco IOS-XR (NCS 5500, ASR 9K, XRd)
  netconf-junos/        # Juniper Junos (MX, QFX, SRX, vMX, cRPD)
  gnmi-arista/          # Arista EOS (gNMI-first, OpenConfig models)
```

Nokia SR OS is **deferred to Phase 8+**. The Nokia customer base is predominantly telco (not enterprise datacenter), and the YANG model divergence is significant. If a Nokia deployment arises before Phase 8, use `RawConfigPush` via the SSH adapter with snapshot_restore compensation.

Arista EOS uses gNMI as its primary management protocol. While Arista does support NETCONF, their OpenConfig implementation and tooling investment is gNMI-first. The `gnmi-arista` adapter uses gNMI Set/Get/Subscribe with Arista-specific OpenConfig augmentations. A future generic `gnmi-openconfig` adapter (Phase 7+) will target SONiC and other pure-OpenConfig platforms, but Arista ships first because it is the most common datacenter switching vendor with strong gNMI support.

### 2.3 Shared Library: netconfcore

```go
package netconfcore

import (
    "context"
    "encoding/xml"
    "time"
)

// Session manages a single NETCONF session over SSH subsystem.
type Session struct {
    transport   Transport       // SSH subsystem connection
    sessionID   uint32          // server-assigned session ID
    capabilities []string       // server-advertised capabilities
    msgID       uint32          // monotonically increasing message-id
}

// Transport abstracts the SSH subsystem connection.
type Transport interface {
    Send(ctx context.Context, msg []byte) error
    Receive(ctx context.Context) ([]byte, error)
    Close() error
}

// Datastore identifies a NETCONF datastore.
type Datastore string

const (
    Running   Datastore = "running"
    Candidate Datastore = "candidate"
    Startup   Datastore = "startup"
)

// EditOperation wraps an <edit-config> payload.
type EditOperation struct {
    Target          Datastore       // which datastore to edit
    DefaultOp       string          // "merge", "replace", "none"
    TestOption      string          // "test-then-set", "set", "test-only"
    ErrorOption     string          // "stop-on-error", "continue-on-error", "rollback-on-error"
    Config          xml.RawMessage  // the <config> subtree
}

// CommitOptions controls commit behavior.
type CommitOptions struct {
    Confirmed       bool            // use confirmed commit
    ConfirmTimeout  time.Duration   // auto-revert if not confirmed within this window
    Persist         string          // persist-id for confirmed commit across sessions
}

// RPCReply represents a parsed NETCONF <rpc-reply>.
type RPCReply struct {
    MessageID string          `xml:"message-id,attr"`
    OK        bool            // true if <ok/> present
    Data      xml.RawMessage  // <data> content for get/get-config
    Errors    []RPCError      // <rpc-error> elements
}

// RPCError represents a single <rpc-error> from the device.
type RPCError struct {
    Type     string `xml:"error-type"`      // "transport", "rpc", "protocol", "application"
    Tag      string `xml:"error-tag"`       // "in-use", "invalid-value", "operation-failed", etc.
    Severity string `xml:"error-severity"`  // "error", "warning"
    AppTag   string `xml:"error-app-tag"`
    Path     string `xml:"error-path"`
    Message  string `xml:"error-message"`
    Info     string `xml:"error-info"`
}

// SessionOps provides the core NETCONF operations available to vendor adapters.
type SessionOps interface {
    // Hello exchanges capabilities with the server.
    Hello(ctx context.Context) ([]string, error)

    // GetConfig retrieves configuration from a datastore.
    GetConfig(ctx context.Context, source Datastore, filter xml.RawMessage) (*RPCReply, error)

    // Get retrieves running configuration and state data.
    Get(ctx context.Context, filter xml.RawMessage) (*RPCReply, error)

    // EditConfig applies a configuration change.
    EditConfig(ctx context.Context, edit EditOperation) (*RPCReply, error)

    // Commit commits the candidate datastore to running.
    Commit(ctx context.Context, opts CommitOptions) (*RPCReply, error)

    // ConfirmCommit confirms a previously issued confirmed commit.
    ConfirmCommit(ctx context.Context, persistID string) (*RPCReply, error)

    // Lock acquires a datastore lock.
    Lock(ctx context.Context, target Datastore) (*RPCReply, error)

    // Unlock releases a datastore lock.
    Unlock(ctx context.Context, target Datastore) (*RPCReply, error)

    // DiscardChanges reverts the candidate datastore to running.
    DiscardChanges(ctx context.Context) (*RPCReply, error)

    // CustomRPC sends a vendor-specific RPC.
    CustomRPC(ctx context.Context, rpc xml.RawMessage) (*RPCReply, error)

    // ServerCapabilities returns the capabilities exchanged during hello.
    ServerCapabilities() []string

    // HasCapability checks whether the server advertised a specific capability URI.
    HasCapability(uri string) bool
}
```

### 2.4 Per-Vendor Capability Matrix

| Capability | netconf-iosxe | netconf-iosxr | netconf-junos | gnmi-arista |
|-----------|--------------|--------------|---------------|-------------|
| **Protocol** | NETCONF 1.1 | NETCONF 1.1 | NETCONF 1.0/1.1 | gNMI 0.8+ |
| **YANG models** | Cisco-native + partial OC | Cisco-native + OC | Juniper-native + partial OC | OpenConfig + Arista augmentations |
| **Candidate datastore** | Yes (16.x+) | Yes | Yes (always) | N/A (gNMI Set is atomic) |
| **Confirmed commit** | Buggy (session-tied, no cross-session persist on some versions) | Yes (robust) | Yes (robust, cross-session persist) | N/A (gNMI Set is atomic) |
| **Rollback support** | `rollback-on-error` in edit-config only | `rollback-on-error` + commit rollback | Up to 50 rollback slots, `rollback 0` restores previous | Replace operation on full subtree |
| **Lock support** | Yes (but lock contention with CLI users is common) | Yes | Yes (configure exclusive) | N/A |
| **Subscription** | Limited (NETCONF notifications, on-change unreliable) | dial-in + dial-out telemetry | Limited NETCONF notifications | gNMI Subscribe (on-change + sample) |
| **Error reporting** | Varies by IOS subsystem; some return generic `operation-failed` | Structured, consistent | Structured, hierarchical | gRPC status codes + gNMI error details |

### 2.5 Compensation Reliability Per Vendor

The ADAPTER-CONTRACT.md table currently claims `transactional` for all NETCONF. This is corrected:

| Adapter | Compensation Reliability | Rationale |
|---------|-------------------------|-----------|
| netconf-junos | `transactional` | Confirmed commit is robust. Rollback slots provide point-in-time recovery. `commit confirmed` with timeout auto-reverts reliably. Cross-session persist works. |
| netconf-iosxr | `transactional` | Confirmed commit is reliable on IOS-XR 7.x+. Candidate datastore semantics are consistent. |
| netconf-iosxe | `snapshot_restore` | Confirmed commit has known bugs: session-tied on some versions (commit lost if SSH drops), `persist` not supported on all platforms, and some subsystems silently accept invalid config in candidate then fail on commit. Compensation falls back to snapshot-restore: capture `running` before mutation, restore on failure. |
| gnmi-arista | `best_effort` | gNMI Set is atomic per-request but there is no confirmed-commit equivalent. A successful Set cannot be auto-reverted by the device. Compensation is a reverse Set with the pre-operation state, which is best-effort because concurrent changes may have occurred. |

### 2.6 Vendor Adapter Interface

Each vendor adapter implements the standard LOOM adapter interfaces plus a vendor-specific YANG mapper:

```go
package adapter

// VendorYANGMapper translates between LOOM canonical intent types and
// vendor-specific YANG paths and XML/JSON payloads.
type VendorYANGMapper interface {
    // IntentToNative converts a LOOM ConfigIntent into a vendor-native
    // edit-config payload or gNMI SetRequest.
    IntentToNative(intent ConfigIntent) (NativePayload, error)

    // NativeToFacts converts a vendor-native get-config or gNMI GetResponse
    // into LOOM ObservedFacts.
    NativeToFacts(data []byte, format string) ([]ObservedFact, error)

    // SupportedIntents returns the list of ConfigIntent types this mapper
    // handles. Intents not in this list fall through to RawConfigPush.
    SupportedIntents() []string

    // VendorName returns the vendor identifier for this mapper.
    VendorName() string

    // MinimumVersion returns the minimum NOS version this mapper supports.
    // The adapter must verify the device version during Connect.
    MinimumVersion() string
}

// NativePayload is the vendor-specific representation of a config change.
type NativePayload struct {
    // For NETCONF adapters: XML edit-config content.
    XMLConfig []byte

    // For gNMI adapters: gNMI SetRequest paths and values.
    GNMIUpdates []GNMIUpdate

    // For both: the YANG path being modified (for logging and audit).
    YANGPath string
}

// GNMIUpdate represents a single gNMI path-value pair for a Set operation.
type GNMIUpdate struct {
    Path     string // gNMI path (e.g., "/interfaces/interface[name=Ethernet1]/config/description")
    Value    []byte // JSON-encoded value
    Encoding string // "json_ietf" or "json"
}
```

### 2.7 Lab Validation Requirements

Each vendor adapter must pass the following validation matrix before shipping. Lab validation runs against real or vendor-provided virtual devices, not mocks.

| Test | netconf-iosxe | netconf-iosxr | netconf-junos | gnmi-arista |
|------|--------------|--------------|---------------|-------------|
| Connect + capability exchange | CML (IOS-XE 17.x) | XRd container | vMX or cRPD | cEOS container |
| Create VLAN via structured intent | Required | Required | Required | Required |
| Delete VLAN via structured intent | Required | Required | Required | Required |
| Configure port (access + trunk) | Required | Required | Required | Required |
| Snapshot running config | Required | Required | Required | Required (gNMI Get `/`) |
| Restore snapshot after failure | Required | Required | Required | Required (gNMI Replace) |
| Confirmed commit + auto-revert | Required (document bugs hit) | Required | Required | N/A |
| Concurrent session lock contention | Required (known issue with CLI) | Required | Required | N/A |
| Error handling (invalid config) | Required | Required | Required | Required |
| Read interface state | Required | Required | Required | Required |

**Minimum lab equipment per adapter:**
- netconf-iosxe: 1x Cisco CML instance with Catalyst 9Kv or IOSv-L2 (IOS-XE 17.6+)
- netconf-iosxr: 1x XRd container (IOS-XR 7.7+) or CML IOS-XRv 9000
- netconf-junos: 1x Juniper vMX or vSRX (Junos 21.4+) or containerized RE (cRPD)
- gnmi-arista: 1x cEOS container (EOS 4.28+) or vEOS-lab

**CI integration:** Containerized NOS images (XRd, cRPD, cEOS) run as ephemeral test fixtures in CI. CML-based tests (IOS-XE) run as nightly integration tests against a persistent CML instance. The conformance test suite (from RESEARCH-protocol-adapter-realism.md section 4.2.2) validates all LOOM adapter interface contracts. Vendor-specific tests validate YANG mapping correctness.

### 2.8 Open Question Resolutions

| Question (from section 2.4) | Decision |
|------|----------|
| Should LOOM target OpenConfig-only? | **No.** OpenConfig coverage is too uneven across vendors. Use vendor-native YANG models as the primary source, with OpenConfig as a bonus where available. The `VendorYANGMapper` abstraction isolates LOOM from this choice. |
| Can containerized NOS images serve as CI test targets? | **Yes, for XRd, cRPD, and cEOS.** These are officially supported by their vendors for testing. IOS-XE requires CML, which runs as a persistent VM, not an ephemeral container. IOS-XE CI tests are nightly, not per-commit. |
| Effort: one vendor adapter vs. one universal adapter? | **Higher initial effort, dramatically lower maintenance cost.** One universal adapter requires N vendor-specific code paths behind conditionals, which is the same code but harder to test, harder to reason about, and harder to debug. Separate adapters with a shared core library are more code upfront but each adapter is independently testable and deployable. |
| Should Nokia SR OS be deferred? | **Yes, to Phase 8+.** Nokia's market is predominantly service provider, not enterprise datacenter. The YANG model (Nokia-native) is substantially different from both Cisco and Juniper, requiring a full adapter build. Defer until a concrete customer need arises. |

---

## 3. Redfish Layered Architecture

### 3.1 Decision: Layered Adapter with Pluggable VendorExtension

A single Redfish adapter handles standard DMTF Redfish operations. Vendor-specific OEM extensions are handled by pluggable `VendorExtension` implementations, selected automatically at connect time based on the Redfish service root.

### 3.2 VendorExtension Interface

```go
package redfish

import (
    "context"
    "time"
)

// VendorExtension provides vendor-specific operations that go beyond
// the standard DMTF Redfish schema. The Redfish adapter delegates to
// the appropriate VendorExtension after auto-detection.
type VendorExtension interface {
    // VendorID returns the vendor identifier (e.g., "Dell", "HPE", "Supermicro").
    VendorID() string

    // --- BIOS Configuration ---

    // GetBIOSAttributes reads all BIOS attributes.
    // Standard Redfish provides /Systems/{id}/Bios but attribute names
    // and value formats differ across vendors.
    GetBIOSAttributes(ctx context.Context) (map[string]BIOSAttribute, error)

    // SetBIOSAttributes applies BIOS attribute changes.
    // Returns a task ID for asynchronous operations (BIOS changes
    // typically require a reboot to take effect).
    SetBIOSAttributes(ctx context.Context, attrs map[string]string) (TaskRef, error)

    // --- Firmware Update ---

    // GetFirmwareInventory returns all firmware components and their versions.
    // Standard Redfish provides /UpdateService/FirmwareInventory but
    // component naming and update methods differ per vendor.
    GetFirmwareInventory(ctx context.Context) ([]FirmwareComponent, error)

    // UpdateFirmware pushes a firmware package to the device.
    // The package format is vendor-specific (Dell DUP, HPE SPP component, etc.).
    UpdateFirmware(ctx context.Context, component string, pkg FirmwarePackage) (TaskRef, error)

    // GetFirmwareUpdateStatus checks the status of an async firmware update.
    GetFirmwareUpdateStatus(ctx context.Context, taskRef TaskRef) (FirmwareUpdateStatus, error)

    // --- Storage Configuration ---

    // GetStorageControllers returns detailed storage controller info
    // including RAID configuration, disk groups, and virtual disks.
    // Standard Redfish /Storage is insufficient for RAID management.
    GetStorageControllers(ctx context.Context) ([]StorageControllerDetail, error)

    // CreateVirtualDisk creates a RAID virtual disk.
    CreateVirtualDisk(ctx context.Context, params VirtualDiskParams) (TaskRef, error)

    // DeleteVirtualDisk deletes a RAID virtual disk.
    DeleteVirtualDisk(ctx context.Context, controllerID string, vdiskID string) (TaskRef, error)

    // --- Lifecycle / Maintenance ---

    // GetLifecycleLog retrieves vendor-specific lifecycle or event logs.
    GetLifecycleLog(ctx context.Context, since time.Time, maxEntries int) ([]LifecycleEvent, error)

    // ClearLifecycleLog clears the lifecycle log.
    ClearLifecycleLog(ctx context.Context) error

    // --- Network Configuration ---

    // GetManagementNetworkConfig reads the BMC management network configuration.
    // This is the BMC's own network, not the host OS network.
    GetManagementNetworkConfig(ctx context.Context) (*ManagementNetwork, error)

    // SetManagementNetworkConfig updates the BMC management network.
    SetManagementNetworkConfig(ctx context.Context, config ManagementNetwork) error
}

// BIOSAttribute represents a single BIOS setting.
type BIOSAttribute struct {
    Name         string   `json:"name"`
    CurrentValue string   `json:"current_value"`
    PendingValue string   `json:"pending_value,omitempty"` // set if change pending reboot
    Type         string   `json:"type"`                    // "string", "integer", "enumeration", "boolean"
    AllowedValues []string `json:"allowed_values,omitempty"`
    ReadOnly     bool     `json:"read_only"`
}

// FirmwareComponent represents an installed firmware component.
type FirmwareComponent struct {
    ID            string `json:"id"`
    Name          string `json:"name"`
    Version       string `json:"version"`
    Updateable    bool   `json:"updateable"`
    ComponentType string `json:"component_type"` // "bmc", "bios", "cpld", "nic", "storage_controller", "psu"
}

// FirmwarePackage describes a firmware update payload.
type FirmwarePackage struct {
    FilePath    string `json:"file_path"`    // local path to firmware file
    Checksum    string `json:"checksum"`     // SHA-256 of the file
    ForceUpdate bool   `json:"force_update"` // update even if same version installed
}

// TaskRef is a reference to an asynchronous BMC task.
type TaskRef struct {
    TaskID   string `json:"task_id"`
    TaskURI  string `json:"task_uri"`  // Redfish task URI (e.g., "/redfish/v1/TaskService/Tasks/42")
}

// FirmwareUpdateStatus reports the state of an async firmware update.
type FirmwareUpdateStatus struct {
    TaskRef     TaskRef `json:"task_ref"`
    State       string  `json:"state"`    // "pending", "running", "completed", "failed"
    PercentDone int     `json:"percent_done"`
    Message     string  `json:"message"`
    Error       string  `json:"error,omitempty"`
}

// StorageControllerDetail extends standard Redfish storage info.
type StorageControllerDetail struct {
    ID            string        `json:"id"`
    Model         string        `json:"model"`
    FirmwareVer   string        `json:"firmware_version"`
    RAIDLevels    []string      `json:"supported_raid_levels"` // "RAID0", "RAID1", "RAID5", "RAID6", "RAID10"
    PhysicalDisks []PhysicalDisk `json:"physical_disks"`
    VirtualDisks  []VirtualDisk  `json:"virtual_disks"`
}

// PhysicalDisk represents a physical drive.
type PhysicalDisk struct {
    ID       string `json:"id"`
    Model    string `json:"model"`
    Serial   string `json:"serial"`
    SizeGB   int    `json:"size_gb"`
    Type     string `json:"type"`     // "HDD", "SSD", "NVMe"
    Protocol string `json:"protocol"` // "SATA", "SAS", "NVMe"
    Status   string `json:"status"`   // "online", "offline", "failed", "rebuilding"
}

// VirtualDisk represents a RAID virtual disk.
type VirtualDisk struct {
    ID        string   `json:"id"`
    Name      string   `json:"name"`
    RAIDLevel string   `json:"raid_level"`
    SizeGB    int      `json:"size_gb"`
    Status    string   `json:"status"` // "online", "offline", "degraded", "rebuilding"
    DiskIDs   []string `json:"disk_ids"`
}

// VirtualDiskParams defines parameters for creating a virtual disk.
type VirtualDiskParams struct {
    ControllerID string   `json:"controller_id"`
    Name         string   `json:"name"`
    RAIDLevel    string   `json:"raid_level"`
    DiskIDs      []string `json:"disk_ids"`
    StripeSizeKB int      `json:"stripe_size_kb,omitempty"` // 0 = controller default
}

// LifecycleEvent represents a vendor lifecycle log entry.
type LifecycleEvent struct {
    Timestamp time.Time `json:"timestamp"`
    Severity  string    `json:"severity"` // "info", "warning", "critical"
    MessageID string    `json:"message_id"`
    Message   string    `json:"message"`
    Component string    `json:"component"`
}

// ManagementNetwork describes the BMC's own network configuration.
type ManagementNetwork struct {
    IPv4Address string `json:"ipv4_address"`
    SubnetMask  string `json:"subnet_mask"`
    Gateway     string `json:"gateway"`
    DHCPEnabled bool   `json:"dhcp_enabled"`
    VLANID      int    `json:"vlan_id,omitempty"`
    Hostname    string `json:"hostname"`
}
```

### 3.3 Vendor Auto-Detection

The Redfish adapter auto-detects the vendor from the service root response at connect time.

```go
package redfish

import (
    "context"
    "strings"
)

// VendorDetector determines which VendorExtension to use based on the
// Redfish service root (/redfish/v1/).
type VendorDetector struct {
    extensions map[string]func() VendorExtension
}

// DetectionResult contains the vendor identification from the service root.
type DetectionResult struct {
    Vendor        string // "Dell", "HPE", "Supermicro", "Lenovo", "unknown"
    Product       string // "Integrated Dell Remote Access Controller", "iLO 6", etc.
    RedfishVersion string // "1.17.0"
    ODataID       string // service root @odata.id
}

// Detect reads the Redfish service root and identifies the vendor.
// Detection keys (checked in order):
//   1. "Vendor" field in /redfish/v1/ response (Dell populates this)
//   2. "Product" field containing vendor identifiers ("iLO" = HPE, "iDRAC" = Dell)
//   3. Oem field keys in /redfish/v1/ ("Dell", "Hpe", "Supermicro")
//   4. /redfish/v1/Managers/{id} "Model" field
// Falls back to "unknown" if no vendor matched.
func (d *VendorDetector) Detect(ctx context.Context, client RedfishClient) (*DetectionResult, error) {
    root, err := client.GetServiceRoot(ctx)
    if err != nil {
        return nil, err
    }

    result := &DetectionResult{
        RedfishVersion: root.RedfishVersion,
    }

    // Priority 1: Explicit Vendor field
    if root.Vendor != "" {
        result.Vendor = normalizeVendor(root.Vendor)
        result.Product = root.Product
        return result, nil
    }

    // Priority 2: Product field contains vendor identifier
    if root.Product != "" {
        result.Product = root.Product
        switch {
        case strings.Contains(root.Product, "iDRAC"):
            result.Vendor = "Dell"
        case strings.Contains(root.Product, "iLO"):
            result.Vendor = "HPE"
        case strings.Contains(root.Product, "IPMI"):
            result.Vendor = "Supermicro" // Supermicro BMC often reports as "IPMI"
        default:
            result.Vendor = "unknown"
        }
        return result, nil
    }

    // Priority 3: OEM keys in service root
    for key := range root.Oem {
        switch strings.ToLower(key) {
        case "dell":
            result.Vendor = "Dell"
            return result, nil
        case "hpe", "hp":
            result.Vendor = "HPE"
            return result, nil
        case "supermicro":
            result.Vendor = "Supermicro"
            return result, nil
        case "lenovo":
            result.Vendor = "Lenovo"
            return result, nil
        }
    }

    // Priority 4: Manager model
    result.Vendor = "unknown"
    return result, nil
}

// RedfishClient is the minimal interface needed for vendor detection.
type RedfishClient interface {
    GetServiceRoot(ctx context.Context) (*ServiceRoot, error)
}

// ServiceRoot represents the parsed /redfish/v1/ response.
type ServiceRoot struct {
    Vendor         string                 `json:"Vendor"`
    Product        string                 `json:"Product"`
    RedfishVersion string                 `json:"RedfishVersion"`
    Oem            map[string]interface{} `json:"Oem"`
}

func normalizeVendor(v string) string {
    switch strings.ToLower(strings.TrimSpace(v)) {
    case "dell", "dell inc.", "dell inc":
        return "Dell"
    case "hpe", "hp", "hewlett packard enterprise":
        return "HPE"
    case "supermicro", "super micro":
        return "Supermicro"
    case "lenovo":
        return "Lenovo"
    default:
        return v
    }
}
```

### 3.4 Vendor Extension Implementations and Shipping Schedule

| Extension | Ships | Phase | Notes |
|-----------|-------|-------|-------|
| `DellExtension` | MVP | 2 | iDRAC 8/9. Uses Dell OEM Lifecycle Controller APIs for BIOS, firmware (DUP packages), RAID (PERC/BOSS). Most common datacenter BMC. |
| `HPEExtension` | MVP | 2 | iLO 5/6. Uses HPE OEM APIs for BIOS (iLO REST API), firmware (SPP component format), storage (Smart Array). Second most common BMC. |
| `GenericExtension` | MVP | 2 | Standard-only fallback. Returns `PermanentError` with code `ADAPTER_UNSUPPORTED_OP` for operations that require OEM extensions. |
| `SupermicroExtension` | Phase 5 | 5 | Supermicro BMC. Lower priority due to firmware version fragmentation and limited Redfish conformance on older BMC revisions. |
| `LenovoExtension` | Phase 7 | 7 | Lenovo XClarity Controller. Deferred unless customer demand arises earlier. |

### 3.5 GenericExtension Fallback Behavior

When the vendor is unrecognized or a vendor extension is not installed, the `GenericExtension` handles all calls. Its behavior:

```go
package redfish

// GenericExtension provides standard-only Redfish operations.
// Used when the vendor is unrecognized or a specific vendor extension
// is not installed.
type GenericExtension struct{}

func (g *GenericExtension) VendorID() string { return "generic" }
```

**Operations that work with GenericExtension (standard Redfish):**
- Power on/off/cycle/reset via `ComputerSystem.Reset`
- Read sensors via `/Chassis/{id}/Thermal`, `/Chassis/{id}/Power`
- Set next boot device via `Boot` property
- Read basic system info (model, serial, firmware version)
- Virtual media mount/unmount via `/Managers/{id}/VirtualMedia`

**Operations that return `ADAPTER_UNSUPPORTED_OP` with GenericExtension:**
- BIOS attribute configuration (vendor-specific attribute names)
- Firmware update (vendor-specific package formats)
- RAID virtual disk creation/deletion (vendor-specific storage APIs)
- Lifecycle log access (vendor-specific log formats)

This means a device with an unrecognized BMC vendor can still be power-managed, monitored, and boot-configured through LOOM. Only advanced provisioning operations require a vendor-specific extension.

---

## 4. Device Rate Limiting

### 4.1 Decision: Valkey-Backed Distributed Token Bucket

Every device managed by LOOM has a rate limiter that restricts the number of concurrent management sessions and operations per second. Rate limiter state is stored in Valkey so that all LOOM workers (potentially running across multiple pods) coordinate against the same device limits.

### 4.2 Rate Limiter Interface

```go
package ratelimit

import (
    "context"
    "time"
)

// DeviceRateLimiter controls access to a single device's management plane.
// All methods are safe for concurrent use across multiple worker processes
// via Valkey-backed coordination.
type DeviceRateLimiter interface {
    // Acquire blocks until a rate limit token is available or ctx expires.
    // Returns a Token that MUST be released when the operation completes.
    // If ctx expires before a token is available, returns a TransientError
    // with code RATE_LIMITED.
    Acquire(ctx context.Context, deviceID string, opWeight int) (*Token, error)

    // TryAcquire attempts to acquire a token without blocking.
    // Returns (nil, false) if no token is available immediately.
    TryAcquire(deviceID string, opWeight int) (*Token, bool)

    // Release returns a token to the pool. MUST be called when the
    // operation completes (success or failure). Failing to release
    // causes the token to auto-expire after Token.LeaseTimeout.
    Release(token *Token) error

    // Status returns the current rate limit state for a device.
    Status(deviceID string) (*RateLimitStatus, error)

    // SetProfile overrides the rate limit profile for a device.
    // Used for manual tuning or auto-adjustment.
    SetProfile(deviceID string, profile RateLimitProfile) error
}

// Token represents a rate limit grant for a single operation.
type Token struct {
    ID           string        `json:"id"`            // unique token ID (UUID v7)
    DeviceID     string        `json:"device_id"`
    AcquiredAt   time.Time     `json:"acquired_at"`
    LeaseTimeout time.Duration `json:"lease_timeout"` // auto-release after this duration
    OpWeight     int           `json:"op_weight"`     // how many concurrency slots this token consumes
}

// RateLimitProfile defines the limits for a device type.
type RateLimitProfile struct {
    // MaxConcurrent is the maximum number of simultaneous operations
    // allowed against this device. Each Acquire consumes OpWeight slots.
    MaxConcurrent int `json:"max_concurrent"`

    // MaxPerSecond is the maximum operations per second (token bucket refill rate).
    MaxPerSecond float64 `json:"max_per_second"`

    // BurstSize is the maximum burst above the sustained rate.
    BurstSize int `json:"burst_size"`

    // LeaseTimeout is how long a token lives before auto-release.
    // Must be longer than the longest expected operation duration.
    // Default: 5 minutes.
    LeaseTimeout time.Duration `json:"lease_timeout"`

    // BackoffOnError is the mandatory delay after a device returns a
    // rate-limit or overload error (HTTP 429, NETCONF resource-denied).
    BackoffOnError time.Duration `json:"backoff_on_error"`

    // BackoffOnTimeout is the mandatory delay after a device times out.
    BackoffOnTimeout time.Duration `json:"backoff_on_timeout"`
}

// RateLimitStatus is the current utilization of a device's rate limit.
type RateLimitStatus struct {
    DeviceID          string           `json:"device_id"`
    Profile           RateLimitProfile `json:"profile"`
    ActiveTokens      int              `json:"active_tokens"`
    AvailableSlots    int              `json:"available_slots"`
    TokensPerSecond   float64          `json:"tokens_per_second"`  // current refill rate
    SafeMode          bool             `json:"safe_mode"`          // true if safe mode is active
    LastBackoffUntil  *time.Time       `json:"last_backoff_until"` // nil if no backoff active
}
```

### 4.3 Default Profiles

```go
package ratelimit

import "time"

// DefaultProfiles contains built-in rate limit profiles keyed by device
// type string. These are applied when a device is first discovered.
// Operators can override per-device via SetProfile.
var DefaultProfiles = map[string]RateLimitProfile{
    // --- BMC / Out-of-Band Management ---
    // BMCs have extremely limited management plane CPUs.
    "bmc-dell-idrac": {
        MaxConcurrent:    4,
        MaxPerSecond:     2,
        BurstSize:        2,
        LeaseTimeout:     5 * time.Minute,
        BackoffOnError:   10 * time.Second,
        BackoffOnTimeout: 15 * time.Second,
    },
    "bmc-hpe-ilo": {
        MaxConcurrent:    8,
        MaxPerSecond:     4,
        BurstSize:        4,
        LeaseTimeout:     5 * time.Minute,
        BackoffOnError:   10 * time.Second,
        BackoffOnTimeout: 15 * time.Second,
    },
    "bmc-supermicro": {
        MaxConcurrent:    2,
        MaxPerSecond:     1,
        BurstSize:        1,
        LeaseTimeout:     5 * time.Minute,
        BackoffOnError:   15 * time.Second,
        BackoffOnTimeout: 30 * time.Second,
    },
    "bmc-generic": {
        MaxConcurrent:    3,
        MaxPerSecond:     1.5,
        BurstSize:        2,
        LeaseTimeout:     5 * time.Minute,
        BackoffOnError:   10 * time.Second,
        BackoffOnTimeout: 15 * time.Second,
    },

    // --- Network Switches ---
    // Management plane shares CPU with control plane on most platforms.
    "switch-cisco-iosxe": {
        MaxConcurrent:    50,
        MaxPerSecond:     20,
        BurstSize:        10,
        LeaseTimeout:     3 * time.Minute,
        BackoffOnError:   5 * time.Second,
        BackoffOnTimeout: 10 * time.Second,
    },
    "switch-cisco-iosxr": {
        MaxConcurrent:    40,
        MaxPerSecond:     15,
        BurstSize:        8,
        LeaseTimeout:     3 * time.Minute,
        BackoffOnError:   5 * time.Second,
        BackoffOnTimeout: 10 * time.Second,
    },
    "switch-arista-eos": {
        MaxConcurrent:    50,
        MaxPerSecond:     20,
        BurstSize:        10,
        LeaseTimeout:     3 * time.Minute,
        BackoffOnError:   5 * time.Second,
        BackoffOnTimeout: 10 * time.Second,
    },
    "switch-juniper": {
        MaxConcurrent:    40,
        MaxPerSecond:     15,
        BurstSize:        8,
        LeaseTimeout:     3 * time.Minute,
        BackoffOnError:   5 * time.Second,
        BackoffOnTimeout: 10 * time.Second,
    },

    // --- Hypervisors ---
    // vSphere, Proxmox, and libvirt have robust management APIs.
    "hypervisor-vsphere": {
        MaxConcurrent:    100,
        MaxPerSecond:     50,
        BurstSize:        20,
        LeaseTimeout:     10 * time.Minute,
        BackoffOnError:   3 * time.Second,
        BackoffOnTimeout: 5 * time.Second,
    },
    "hypervisor-proxmox": {
        MaxConcurrent:    30,
        MaxPerSecond:     15,
        BurstSize:        10,
        LeaseTimeout:     10 * time.Minute,
        BackoffOnError:   3 * time.Second,
        BackoffOnTimeout: 5 * time.Second,
    },

    // --- KVM / Out-of-Band ---
    "kvm-pikvm": {
        MaxConcurrent:    2,
        MaxPerSecond:     1,
        BurstSize:        1,
        LeaseTimeout:     5 * time.Minute,
        BackoffOnError:   10 * time.Second,
        BackoffOnTimeout: 15 * time.Second,
    },
}
```

### 4.4 Valkey Key Structure

Rate limiter state is stored in Valkey with the following key patterns:

```
loom:ratelimit:{device_id}:tokens        -- Sorted set of active token IDs, scored by expiry timestamp
loom:ratelimit:{device_id}:bucket        -- Token bucket counter (remaining tokens for per-second rate)
loom:ratelimit:{device_id}:bucket_ts     -- Last refill timestamp (for token bucket algorithm)
loom:ratelimit:{device_id}:profile       -- JSON-serialized RateLimitProfile override (if set)
loom:ratelimit:{device_id}:backoff       -- Backoff expiry timestamp (operations blocked until this time)
loom:ratelimit:safe_mode                 -- Boolean flag for global safe mode
```

Token acquisition and release are implemented as Lua scripts executed atomically in Valkey to prevent race conditions between workers.

### 4.5 Temporal Activity Timeout Interaction

Rate limiting interacts with Temporal activity timeouts as follows:

**Problem:** A Temporal activity has a `ScheduleToClose` timeout (e.g., 5 minutes for a VLAN creation). If the rate limiter blocks for 4 minutes waiting for a token, the activity has only 1 minute left for the actual device operation, which may not be enough.

**Solution:** The `Acquire` call accepts a `context.Context` derived from the Temporal activity context. This context carries both the activity deadline and the heartbeat requirement. The rate limiter respects the context deadline. If the token cannot be acquired before the context expires, `Acquire` returns a `TransientError` with code `RATE_LIMITED` and a `RetryAfter` hint. Temporal then retries the activity per its retry policy, and the next attempt starts a fresh rate limit acquisition.

```go
package ratelimit

// AcquireWithBudget is the recommended entry point for Temporal activities.
// It checks the remaining time budget and refuses to wait if insufficient
// time remains for both the rate limit wait and the actual operation.
//
// minOperationTime is the minimum duration needed for the device operation
// itself (exclusive of rate limit wait time).
func AcquireWithBudget(
    ctx context.Context,
    limiter DeviceRateLimiter,
    deviceID string,
    opWeight int,
    minOperationTime time.Duration,
) (*Token, error) {
    deadline, ok := ctx.Deadline()
    if ok {
        remaining := time.Until(deadline)
        if remaining < minOperationTime {
            // Not enough time for the operation even without rate limit wait.
            return nil, NewTransientError(
                "RATE_LIMITED",
                "insufficient time budget for operation",
                "", // correlation ID added by caller
                minOperationTime - remaining, // retry after this much time
                nil,
            )
        }
        // Create a sub-context that expires early enough to leave
        // minOperationTime for the actual operation.
        waitBudget := remaining - minOperationTime
        var cancel context.CancelFunc
        ctx, cancel = context.WithTimeout(ctx, waitBudget)
        defer cancel()
    }

    return limiter.Acquire(ctx, deviceID, opWeight)
}
```

**Heartbeat integration:** While waiting for a rate limit token, the Temporal activity must continue sending heartbeats. The `Acquire` implementation periodically checks the context for cancellation (which Temporal sends if the worker restarts or the workflow is cancelled) and returns early.

### 4.6 Safe Mode

Safe mode halves all rate limit profiles globally. It is activated during incidents to reduce pressure on management planes across the fleet.

```go
package ratelimit

// SafeMode controls the global safe mode flag.
type SafeMode struct {
    Active    bool      `json:"active"`
    Reason    string    `json:"reason"`
    ActivatedBy string `json:"activated_by"` // actor who activated safe mode
    ActivatedAt time.Time `json:"activated_at"`
    ExpiresAt   *time.Time `json:"expires_at,omitempty"` // nil = manual deactivation required
}
```

**Safe mode effects:**
- `MaxConcurrent` is halved (minimum 1)
- `MaxPerSecond` is halved (minimum 0.5)
- `BurstSize` is halved (minimum 1)
- `BackoffOnError` is doubled
- `BackoffOnTimeout` is doubled
- All active tokens remain valid but new acquisitions use the reduced limits

**Safe mode activation:**
- Manual: API call by operator with `admin` role
- Automatic: When more than 30% of managed devices report `ADAPTER_TIMEOUT` or `ADAPTER_UNREACHABLE` within a 5-minute window (measured by the rate limiter's error callback). Auto-activation emits a NATS event on `loom.system.ratelimit.safe_mode_activated`.
- Safe mode auto-expires after 30 minutes by default (configurable). Operators can extend or cancel early.

### 4.7 Profile Auto-Discovery

When a device is first discovered, the adapter reports the device type as part of the `DiscoveryResult.DeviceType` field. The rate limiter looks up the default profile using this device type. If no exact match is found, it falls back to the most conservative profile for the device class (BMC, switch, hypervisor).

Adapters can also report rate limit hints as `ObservedFact` entries during discovery:

| Fact Key | Example Value | Meaning |
|----------|---------------|---------|
| `mgmt_plane_max_sessions` | `8` | Maximum concurrent management sessions |
| `mgmt_plane_rate_limit` | `10` | Vendor-documented operations per second |
| `mgmt_plane_model` | `iDRAC9 Enterprise` | Specific management controller model |

When these facts are present, the rate limiter adjusts the default profile accordingly, subject to a floor (never above the default profile's limits unless the operator explicitly overrides).

---

## 5. RawConfigPush Security

### 5.1 Decision: Disabled by Default, Opt-In Per Tenant

`RawConfigPush` is a privileged escape hatch. It is disabled by default for all tenants. Enabling it requires a tenant-level configuration change by a LOOM platform administrator. Once enabled for a tenant, individual uses still require elevated authorization.

### 5.2 Tenant Opt-In Configuration

```go
package tenant

// RawConfigPushPolicy controls whether and how RawConfigPush is allowed
// for a tenant. Stored in the tenant configuration.
type RawConfigPushPolicy struct {
    // Enabled controls whether RawConfigPush is available at all.
    // Default: false. Requires platform admin to change.
    Enabled bool `json:"enabled"`

    // RequireApproval controls whether a second approver must authorize
    // each RawConfigPush before execution. Default: true (when enabled).
    RequireApproval bool `json:"require_approval"`

    // AllowedDeviceTypes limits which device types can receive raw config.
    // Empty list = all device types allowed (when enabled).
    // Use this to restrict raw config to device types without structured adapters.
    AllowedDeviceTypes []string `json:"allowed_device_types,omitempty"`

    // ContentAnalysis controls whether content analysis rules are enforced.
    // Default: true. When true, config bodies are scanned for dangerous
    // patterns before execution. Matches block execution (not advisory).
    ContentAnalysis bool `json:"content_analysis"`

    // MaxConfigSizeBytes limits the size of a single RawConfigPush body.
    // Default: 65536 (64KB). Prevents accidental or malicious mega-configs.
    MaxConfigSizeBytes int `json:"max_config_size_bytes"`

    // AuditRetentionDays controls how long pre/post config diffs are retained.
    // Default: 365 days. Set to 0 for indefinite retention.
    AuditRetentionDays int `json:"audit_retention_days"`
}
```

### 5.3 Authorization Requirements

Every `RawConfigPush` execution requires:

1. **Tenant-level opt-in:** `RawConfigPushPolicy.Enabled` must be `true`.
2. **Admin role:** The requesting user must have the `admin` or `raw_config_push` role within the tenant.
3. **Second approver (when `RequireApproval` is true):** A different user with the `admin` or `raw_config_push_approver` role must approve the operation. This uses the same Temporal signal-based approval gate defined in WORKFLOW-CONTRACT.md.
4. **Content analysis pass:** The config body must pass all content analysis rules (unless `ContentAnalysis` is disabled).

### 5.4 Content Analysis Rules

Content analysis scans the `ConfigBody` for patterns known to be dangerous. Matches block execution and return a `ValidationError`.

```go
package rawconfig

import "regexp"

// ContentRule defines a single dangerous-config detection pattern.
type ContentRule struct {
    ID          string         `json:"id"`
    Name        string         `json:"name"`
    Description string         `json:"description"`
    Pattern     *regexp.Regexp `json:"-"`            // compiled regex
    PatternStr  string         `json:"pattern"`      // serializable form
    Severity    string         `json:"severity"`     // "block" or "warn"
    DeviceTypes []string       `json:"device_types"` // empty = all types
}

// DefaultContentRules are the built-in content analysis patterns.
// Operators can add custom rules per tenant. These cannot be disabled
// individually -- only ContentAnalysis as a whole can be toggled.
var DefaultContentRules = []ContentRule{
    // --- Authentication / Access Control ---
    {
        ID:          "RAW001",
        Name:        "Privilege 15 user creation",
        Description: "Creating a user with full administrative privileges",
        PatternStr:  `(?i)username\s+\S+\s+privilege\s+15`,
        Severity:    "block",
    },
    {
        ID:          "RAW002",
        Name:        "Enable secret removal",
        Description: "Removing or disabling the enable secret",
        PatternStr:  `(?i)no\s+enable\s+(secret|password)`,
        Severity:    "block",
    },
    {
        ID:          "RAW003",
        Name:        "AAA disabled",
        Description: "Disabling AAA authentication",
        PatternStr:  `(?i)no\s+aaa\s+(new-model|authentication)`,
        Severity:    "block",
    },
    {
        ID:          "RAW004",
        Name:        "SSH disabled",
        Description: "Disabling SSH server",
        PatternStr:  `(?i)no\s+(ip\s+)?ssh\s+server`,
        Severity:    "block",
    },

    // --- ACL / Firewall ---
    {
        ID:          "RAW005",
        Name:        "Permit any any",
        Description: "ACL rule permitting all traffic (potential backdoor)",
        PatternStr:  `(?i)permit\s+(ip|tcp|udp)\s+any\s+any`,
        Severity:    "block",
    },
    {
        ID:          "RAW006",
        Name:        "Management ACL removal",
        Description: "Removing management access control lists",
        PatternStr:  `(?i)no\s+(ip\s+)?access-(list|group)\s+\S*(mgmt|manage|admin)\S*`,
        Severity:    "block",
    },

    // --- Routing / Control Plane ---
    {
        ID:          "RAW007",
        Name:        "BGP peer with unrestricted prefix",
        Description: "BGP neighbor with no prefix limit (route leak risk)",
        PatternStr:  `(?i)no\s+neighbor\s+\S+\s+maximum-prefix`,
        Severity:    "warn",
    },

    // --- Management Plane ---
    {
        ID:          "RAW008",
        Name:        "SNMP community string change",
        Description: "Modifying SNMP community strings (credential change)",
        PatternStr:  `(?i)snmp-server\s+community\s+`,
        Severity:    "warn",
    },
    {
        ID:          "RAW009",
        Name:        "Logging disabled",
        Description: "Disabling syslog or logging",
        PatternStr:  `(?i)no\s+logging\s+(on|host|buffered|console)`,
        Severity:    "block",
    },

    // --- Junos-specific ---
    {
        ID:          "RAW010",
        Name:        "Junos superuser class",
        Description: "Creating a Junos user with super-user login class",
        PatternStr:  `(?i)set\s+system\s+login\s+user\s+\S+\s+class\s+super-user`,
        Severity:    "block",
    },
    {
        ID:          "RAW011",
        Name:        "Junos firewall deactivate",
        Description: "Deactivating Junos firewall filters",
        PatternStr:  `(?i)deactivate\s+firewall`,
        Severity:    "block",
    },

    // --- Generic dangerous patterns ---
    {
        ID:          "RAW012",
        Name:        "Plaintext password",
        Description: "Configuration containing a plaintext password",
        PatternStr:  `(?i)(password|secret)\s+0\s+\S+`,
        Severity:    "block",
    },
    {
        ID:          "RAW013",
        Name:        "Console/VTY access widened",
        Description: "Changing console or VTY access restrictions",
        PatternStr:  `(?i)line\s+(con|vty)\s+\d+.*\n.*no\s+(login|password)`,
        Severity:    "block",
    },
}
```

### 5.5 Mandatory Pre/Post Config Diff

Every `RawConfigPush` execution, regardless of content analysis or approval settings, captures the device's running configuration before and after the push. This is non-negotiable and cannot be disabled.

```go
package rawconfig

import (
    "time"
)

// RawConfigAuditRecord is stored for every RawConfigPush execution.
// Stored in PostgreSQL in the audit schema, linked to the workflow execution.
type RawConfigAuditRecord struct {
    ID              string    `json:"id"`               // UUID v7
    TenantID        string    `json:"tenant_id"`
    DeviceID        string    `json:"device_id"`
    WorkflowID      string    `json:"workflow_id"`      // Temporal workflow ID
    RequestedBy     string    `json:"requested_by"`     // who submitted the push
    ApprovedBy      string    `json:"approved_by"`      // who approved (empty if no approval required)
    ApprovedAt      *time.Time `json:"approved_at"`

    // Config body as submitted (before any normalization).
    ConfigBody      string    `json:"config_body"`
    ConfigFormat    string    `json:"config_format"`    // "cli_commands", "xml", "json", "set_commands"

    // Content analysis results.
    ContentRulesRun    bool      `json:"content_rules_run"`
    ContentWarnings    []string  `json:"content_warnings"`    // rule IDs that matched with severity "warn"
    ContentBlocked     bool      `json:"content_blocked"`     // true if any "block" rule matched

    // Pre/post config snapshots.
    PreConfig       string    `json:"pre_config"`       // running config before push
    PreConfigHash   string    `json:"pre_config_hash"`  // SHA-256 of PreConfig
    PostConfig      string    `json:"post_config"`      // running config after push
    PostConfigHash  string    `json:"post_config_hash"` // SHA-256 of PostConfig
    ConfigDiff      string    `json:"config_diff"`      // unified diff of PreConfig vs PostConfig

    // Execution result.
    Status          string    `json:"status"`           // "succeeded", "failed", "blocked", "rolled_back"
    Error           string    `json:"error,omitempty"`
    ExecutedAt      time.Time `json:"executed_at"`
    DurationMs      int64     `json:"duration_ms"`
}
```

### 5.6 Approval Gate Workflow

The `RawConfigPush` approval workflow integrates with the standard Temporal approval gate pattern:

```
1. Operator submits RawConfigPush request
2. LOOM validates:
   a. Tenant has RawConfigPushPolicy.Enabled = true
   b. Operator has admin or raw_config_push role
   c. Content analysis passes (no "block" matches)
   d. Config size within MaxConfigSizeBytes
3. If RequireApproval = true:
   a. Workflow enters "awaiting_approval" state
   b. Notification sent to users with raw_config_push_approver role
   c. Workflow blocks on Temporal signal
   d. Approver reviews: sees config body, content warnings, target device
   e. Approver sends approve/reject signal
   f. If rejected: workflow fails with APPROVAL_REJECTED
   g. Approval timeout: 24 hours, then auto-reject
4. Pre-config snapshot captured
5. Config pushed to device via adapter
6. Post-config snapshot captured
7. Diff computed and stored
8. Audit record written
```

The approver cannot be the same user who submitted the request. This is enforced at the signal handler level by comparing `RequestedBy` with the signal sender's identity.

### 5.7 Open Question Resolutions

| Question (from section 6.3) | Decision |
|------|----------|
| Should RawConfigPush be disabled by default? | **Yes.** Disabled per tenant. Platform admin enables it. Even when enabled, admin role + content analysis + optional approval gate protect against misuse. |
| Should pre/post diff be automatically reviewed? | **No automatic LLM review.** The diff is stored for human audit. Automatic review adds latency and is advisory at best. Instead, content analysis rules provide deterministic pre-execution blocking, and the post-diff is available for incident investigation. |
| Is LLM review of raw config practical? | **Not at execution time.** LLM latency (2-10s) is acceptable for review but the LLM cannot reliably distinguish "intentional network change" from "security misconfiguration" without understanding the operator's intent. Deferred to Phase 7+ as an optional pre-submission advisory tool, never a blocking gate. |

---

## 6. Plugin Trust Boundary

### 6.1 Decision: Compiled-In Through Phase 5, go-plugin with Signing for Phase 6+

All adapters are compiled into the LOOM binary for Phases 1-5. This eliminates the plugin trust boundary problem entirely during the period when the adapter count is small (expected 8-12 adapters) and the development team controls all adapter code.

Starting Phase 6, third-party and customer-developed adapters are supported via HashiCorp go-plugin (gRPC). These external adapters require cryptographic signing, resource limits, and capability declarations.

### 6.2 Phase 1-5: Compiled-In Adapters

```go
package adapter

// CompiledAdapterRegistry is used for Phases 1-5.
// All adapters are imported and registered at compile time.
// No plugin loading, no signing, no sandbox.
func RegisterCompiledAdapters(registry Registry) {
    // Phase 2 adapters
    registry.Register(ssh.Registration())
    registry.Register(redfish.Registration())
    registry.Register(snmp.Registration())
    registry.Register(ipmi.Registration())

    // Phase 3 adapters
    registry.Register(netconfiosxe.Registration())
    registry.Register(netconfiosxr.Registration())
    registry.Register(netconfjunos.Registration())
    registry.Register(gnmiarista.Registration())
    registry.Register(eapi.Registration())

    // Phase 4 adapters
    registry.Register(vsphere.Registration())
    registry.Register(proxmox.Registration())
    registry.Register(libvirt.Registration())

    // Phase 5 adapters
    registry.Register(pikvm.Registration())
    registry.Register(amt.Registration())
}
```

**Rationale for deferring plugins:** Plugin isolation adds operational complexity (signing infrastructure, sandbox configuration, resource monitoring, crash isolation) that is not justified when all adapter code is first-party. The 12-15 adapters planned through Phase 5 are manageable as compiled-in code. Plugin support is justified when customers or partners need to add adapters for proprietary hardware without forking LOOM.

### 6.3 Phase 6+: go-plugin with Signing and Resource Limits

#### 6.3.1 Plugin Signing

Every external adapter plugin must be signed with an Ed25519 key. The signing infrastructure reuses the same key management used for LOOM agent binary updates (per ADR-012).

```go
package plugin

import (
    "crypto/ed25519"
    "time"
)

// PluginSignature accompanies every external adapter plugin binary.
type PluginSignature struct {
    // PluginName is the adapter name declared by the plugin.
    PluginName string `json:"plugin_name"`

    // PluginVersion is the semver version of the plugin.
    PluginVersion string `json:"plugin_version"`

    // BinaryHash is the SHA-256 hash of the plugin binary.
    BinaryHash string `json:"binary_hash"`

    // SignerKeyID identifies which signing key was used.
    // Multiple signing keys may be active (key rotation).
    SignerKeyID string `json:"signer_key_id"`

    // Signature is the Ed25519 signature over the canonical form of
    // (PluginName + PluginVersion + BinaryHash + SignerKeyID + SignedAt).
    Signature []byte `json:"signature"`

    // SignedAt is when the signature was created.
    SignedAt time.Time `json:"signed_at"`

    // ExpiresAt is when this signature expires. Plugins with expired
    // signatures are not loaded. Default validity: 365 days.
    ExpiresAt time.Time `json:"expires_at"`
}

// SignatureVerifier validates plugin signatures before loading.
type SignatureVerifier interface {
    // Verify checks the plugin binary against its signature.
    // Returns nil if the signature is valid and not expired.
    // Returns a PermanentError if verification fails.
    Verify(binaryPath string, sig PluginSignature) error

    // TrustedKeys returns the set of currently trusted signing key IDs.
    TrustedKeys() []string
}

// PluginSignaturePayload is the canonical form that is signed.
// Fields are concatenated in this exact order with null byte separators.
type PluginSignaturePayload struct {
    PluginName    string
    PluginVersion string
    BinaryHash    string
    SignerKeyID   string
    SignedAt      time.Time
}

// Canonical returns the byte sequence that is signed/verified.
func (p PluginSignaturePayload) Canonical() []byte {
    // Format: name\x00version\x00hash\x00keyid\x00timestamp_rfc3339
    return []byte(
        p.PluginName + "\x00" +
        p.PluginVersion + "\x00" +
        p.BinaryHash + "\x00" +
        p.SignerKeyID + "\x00" +
        p.SignedAt.UTC().Format(time.RFC3339),
    )
}
```

**Key management rules:**
- Signing keys are Ed25519. Minimum 2 active keys at all times (for rotation).
- Key generation and storage follows the same process as agent update signing keys (see ADR-012).
- A key compromise triggers immediate revocation: the compromised key ID is added to a deny list in the LOOM control plane. All plugins signed by that key are unloaded within 5 minutes (next health check cycle).
- Customer-developed plugins can be signed with a customer-managed key. The customer's public key is registered as a trusted key in the LOOM control plane for that tenant only. Customer keys cannot sign plugins for other tenants.

#### 6.3.2 Resource Limits

External plugins run as separate processes (via go-plugin gRPC). Resource limits are enforced via cgroups v2 on Linux.

```go
package plugin

import "time"

// PluginResourceLimits defines the resource constraints for an external
// adapter plugin process. Enforced via cgroups v2.
type PluginResourceLimits struct {
    // MaxMemoryBytes is the hard memory limit. Plugin is killed if exceeded.
    // Default: 256MB. Most adapters need < 64MB.
    MaxMemoryBytes int64 `json:"max_memory_bytes"`

    // CPUQuotaPercent is the CPU time limit as a percentage of one core.
    // 100 = one full core, 50 = half a core.
    // Default: 50 (half a core).
    CPUQuotaPercent int `json:"cpu_quota_percent"`

    // MaxOpenFiles is the open file descriptor limit.
    // Default: 256. Adapters that manage connection pools may need more.
    MaxOpenFiles int `json:"max_open_files"`

    // MaxProcesses limits the number of threads/goroutines (via PID limit).
    // Default: 64.
    MaxProcesses int `json:"max_processes"`

    // NetworkPolicy controls what network endpoints the plugin can reach.
    // Empty = no network restrictions (for Phase 6).
    // Phase 8+ adds network namespace isolation with declared endpoints.
    NetworkPolicy NetworkPolicy `json:"network_policy"`

    // StartupTimeout is the maximum time for the plugin to respond to
    // the initial handshake. Default: 10 seconds.
    StartupTimeout time.Duration `json:"startup_timeout"`

    // HealthCheckInterval is how often the host pings the plugin.
    // Default: 30 seconds. Three consecutive missed health checks = plugin killed.
    HealthCheckInterval time.Duration `json:"health_check_interval"`
}

// NetworkPolicy defines the network access rules for a plugin.
// Phase 6-7: advisory only (logged but not enforced).
// Phase 8+: enforced via network namespace.
type NetworkPolicy struct {
    // AllowedEndpoints lists the host:port pairs the plugin is allowed to
    // connect to. These are the management endpoints of the devices the
    // plugin manages. Populated dynamically from the device registry.
    AllowedEndpoints []string `json:"allowed_endpoints"`

    // AllowDNS controls whether the plugin can make DNS queries.
    // Default: true (needed for hostname-based device endpoints).
    AllowDNS bool `json:"allow_dns"`

    // DenyOutbound, when true, blocks all outbound connections except
    // to AllowedEndpoints. Default: false (Phase 6-7), true (Phase 8+).
    DenyOutbound bool `json:"deny_outbound"`
}

// DefaultPluginLimits returns the default resource limits for plugins.
func DefaultPluginLimits() PluginResourceLimits {
    return PluginResourceLimits{
        MaxMemoryBytes:      256 * 1024 * 1024, // 256MB
        CPUQuotaPercent:     50,
        MaxOpenFiles:        256,
        MaxProcesses:        64,
        StartupTimeout:      10 * time.Second,
        HealthCheckInterval: 30 * time.Second,
        NetworkPolicy: NetworkPolicy{
            AllowDNS:     true,
            DenyOutbound: false, // advisory in Phase 6-7
        },
    }
}
```

#### 6.3.3 Capability Declaration

Every external plugin must provide a capability declaration at load time. The LOOM host verifies this declaration against the plugin's actual behavior. Declaring a capability the plugin does not implement results in the plugin being unloaded.

```go
package plugin

// PluginCapabilityDeclaration is provided by the plugin during the
// go-plugin handshake. The host verifies this against the adapter
// registration and refuses to load plugins with mismatches.
type PluginCapabilityDeclaration struct {
    // PluginName must match the signed PluginSignature.PluginName.
    PluginName string `json:"plugin_name"`

    // PluginVersion must match the signed PluginSignature.PluginVersion.
    PluginVersion string `json:"plugin_version"`

    // ProtocolVersion is the LOOM adapter protocol version this plugin
    // was built against. The host checks compatibility.
    // Format: "1.0", "1.1", etc. Minor version bumps are backward compatible.
    ProtocolVersion string `json:"protocol_version"`

    // Interfaces declares which adapter interfaces this plugin implements.
    Interfaces PluginInterfaces `json:"interfaces"`

    // SupportedOperations lists the operation types the Executor supports
    // (if Executor is true in Interfaces). Used for operation routing.
    SupportedOperations []string `json:"supported_operations,omitempty"`

    // CompensationReliability declares the plugin's compensation level.
    CompensationReliability string `json:"compensation_reliability"`

    // TargetDeviceTypes lists the device types this adapter handles.
    // Used for adapter selection during operation dispatch.
    TargetDeviceTypes []string `json:"target_device_types"`

    // TargetProtocol is the protocol this adapter uses (e.g., "snmp", "redfish").
    TargetProtocol string `json:"target_protocol"`

    // RequestedResources declares the resources this plugin needs.
    // The host uses this to validate against PluginResourceLimits.
    RequestedResources RequestedResources `json:"requested_resources"`
}

// PluginInterfaces declares which LOOM adapter interfaces the plugin implements.
type PluginInterfaces struct {
    Connector   bool `json:"connector"`    // always true (required)
    Discoverer  bool `json:"discoverer"`
    Executor    bool `json:"executor"`
    StateReader bool `json:"state_reader"`
    Watcher     bool `json:"watcher"`
}

// RequestedResources lets the plugin declare its expected resource usage.
// The host compares this against the configured PluginResourceLimits.
type RequestedResources struct {
    // ExpectedMemoryMB is the plugin's expected steady-state memory usage.
    ExpectedMemoryMB int `json:"expected_memory_mb"`

    // ExpectedConnections is the number of concurrent device connections
    // the plugin expects to maintain.
    ExpectedConnections int `json:"expected_connections"`

    // NeedsFileSystem indicates whether the plugin reads/writes local files
    // (e.g., for config template caching).
    NeedsFileSystem bool `json:"needs_file_system"`
}
```

#### 6.3.4 Credential Use Monitoring

External plugins receive credentials via the go-plugin gRPC channel. The host tracks credential access patterns and alerts on anomalies.

```go
package plugin

import "time"

// CredentialAccessRecord tracks a single credential access by a plugin.
type CredentialAccessRecord struct {
    PluginName    string    `json:"plugin_name"`
    CredentialRef string    `json:"credential_ref"` // vault path (never the actual secret)
    DeviceID      string    `json:"device_id"`
    AccessedAt    time.Time `json:"accessed_at"`
    ReleasedAt    *time.Time `json:"released_at"`   // nil if still held
    Purpose       string    `json:"purpose"`        // "connect", "execute", "discover"
}

// CredentialMonitor tracks and alerts on plugin credential usage.
type CredentialMonitor interface {
    // RecordAccess logs that a plugin accessed a credential.
    RecordAccess(record CredentialAccessRecord)

    // RecordRelease logs that a plugin released a credential.
    RecordRelease(pluginName string, credentialRef string)

    // CheckAlerts evaluates alert conditions.
    // Returns alerts for:
    //   - Credential held longer than TTL (default: 5 minutes)
    //   - Plugin accessing credentials for devices not in its target list
    //   - Sudden spike in credential access frequency (>10x normal rate)
    CheckAlerts() []CredentialAlert
}

// CredentialAlert is raised when plugin credential use is anomalous.
type CredentialAlert struct {
    PluginName    string    `json:"plugin_name"`
    AlertType     string    `json:"alert_type"`     // "hold_exceeded", "unauthorized_device", "access_spike"
    CredentialRef string    `json:"credential_ref"`
    Message       string    `json:"message"`
    Severity      string    `json:"severity"`       // "warning", "critical"
    Timestamp     time.Time `json:"timestamp"`
}
```

**Alert responses:**
- `hold_exceeded` (warning): Log and emit NATS event. The plugin may be legitimately maintaining a long-lived session.
- `unauthorized_device` (critical): Log, emit NATS event, and optionally kill the plugin process. A plugin accessing credentials for devices outside its declared target device types is a strong signal of compromise or misconfiguration.
- `access_spike` (warning): Log and emit NATS event. May indicate a legitimate batch operation or a credential-harvesting attempt.

### 6.4 Plugin Loading Sequence

```
Phase 6+ Plugin Loading:

1. Host discovers plugin binary at configured path
2. Host reads companion .sig file (PluginSignature JSON)
3. SignatureVerifier.Verify() checks:
   a. BinaryHash matches SHA-256 of the binary file
   b. Signature is valid Ed25519 over canonical payload
   c. SignerKeyID is in the trusted key set
   d. ExpiresAt is in the future
   e. SignerKeyID is not in the revocation list
4. If verification fails: refuse to load, log PermanentError, emit NATS alert
5. Host launches plugin process with cgroups v2 limits
6. go-plugin gRPC handshake
7. Plugin sends PluginCapabilityDeclaration
8. Host validates declaration:
   a. PluginName matches signature
   b. PluginVersion matches signature
   c. ProtocolVersion is compatible with host version
   d. Connector interface is declared (mandatory)
   e. RequestedResources fit within PluginResourceLimits
9. If validation fails: kill plugin process, log error
10. Host registers adapter in the Registry
11. Periodic health checks begin (HealthCheckInterval)
12. Credential access monitoring begins
```

### 6.5 Open Question Resolutions

| Question (from section 7.3) | Decision |
|------|----------|
| Is plugin isolation worth the complexity for Phases 1-5? | **No.** All adapters compiled-in for Phases 1-5. The adapter count is small and all code is first-party. Plugin isolation is deferred to Phase 6 when third-party adapters become relevant. |
| Should plugins have a capability declaration? | **Yes.** The `PluginCapabilityDeclaration` is verified at load time. Mismatches between declared and actual capabilities result in plugin rejection. |
| Can gVisor or Firecracker provide stronger isolation? | **Deferred to Phase 9+.** gVisor adds significant operational complexity (custom kernel syscall layer). Firecracker requires a full VM per plugin, which is excessive for most adapters. cgroups v2 + network namespaces (Phase 8+) provide sufficient isolation for the threat model. gVisor is reconsidered only if a plugin compromise occurs in the field. |

---

## 7. Cross-Cutting: Updates to Existing Contracts

This section lists the specific changes that must be made to existing LOOM contract documents to reflect the decisions in this specification.

### 7.1 ADAPTER-CONTRACT.md Updates

The compensation reliability table (section "Which Adapters Implement Which Families") must be updated:

| Adapter | Old Compensation Reliability | New Compensation Reliability |
|---------|------------------------------|------------------------------|
| NETCONF | `transactional` (single row) | Split into: netconf-junos = `transactional`, netconf-iosxr = `transactional`, netconf-iosxe = `snapshot_restore` |
| gNMI | `transactional` | `best_effort` (no confirmed-commit equivalent) |

Add to the adapter registration table:

| Adapter | Connector | Discoverer | Executor | StateReader | Watcher | Compensation Reliability | SnapshotCapable |
|---------|-----------|------------|----------|-------------|---------|--------------------------|-----------------|
| netconf-iosxe | Yes | Yes | Yes | Yes | No | snapshot_restore | Yes |
| netconf-iosxr | Yes | Yes | Yes | Yes | No | transactional | Yes |
| netconf-junos | Yes | Yes | Yes | Yes | No | transactional | Yes |
| gnmi-arista | Yes | Yes | Yes | Yes | Yes | best_effort | Yes |

### 7.2 ERROR-MODEL.md Updates

Add the following error code:

| Code | Type | Description |
|------|------|-------------|
| `RATE_LIMITED` | Transient | The operation was blocked by the device rate limiter. RetryAfter indicates when to try again. |
| `RAW_CONFIG_BLOCKED` | Permanent | The RawConfigPush was blocked by content analysis rules. |
| `RAW_CONFIG_DISABLED` | Permanent | RawConfigPush is disabled for this tenant. |

Add retry policy:

```go
// RateLimitedRetryPolicy: wait for the rate limiter to free up.
RateLimitedRetryPolicy = RetryPolicy{
    MaxAttempts:     10,
    InitialInterval: 2 * time.Second,
    MaxInterval:     60 * time.Second,
    BackoffFactor:   1.5,
}
```

### 7.3 DISCOVERY-CONTRACT.md Updates

Adapters should emit rate-limit-relevant `ObservedFact` entries during discovery (see section 4.7). Add to the `ObservedFact` examples:

```go
// Management plane capacity facts (used by rate limiter auto-configuration)
ObservedFact{Category: "management", Key: "mgmt_plane_max_sessions", Value: 8, Source: "redfish"}
ObservedFact{Category: "management", Key: "mgmt_plane_rate_limit", Value: 10, Source: "redfish"}
ObservedFact{Category: "management", Key: "mgmt_plane_model", Value: "iDRAC9 Enterprise", Source: "redfish"}
```

---

## 8. Implementation Priority

| Item | Phase | Effort | Dependencies | Closes |
|------|-------|--------|--------------|--------|
| netconfcore shared library | 2 | 2 weeks | None | C7 |
| netconf-iosxe adapter + lab validation | 2 | 3 weeks | netconfcore, CML lab | C7 |
| netconf-junos adapter + lab validation | 2 | 3 weeks | netconfcore, vMX lab | C7 |
| gnmi-arista adapter + lab validation | 3 | 3 weeks | cEOS lab | C7 |
| netconf-iosxr adapter + lab validation | 3 | 2 weeks | netconfcore, XRd lab | C7 |
| Redfish layered adapter (standard + Dell) | 2 | 3 weeks | Dell iDRAC lab | H19 |
| Redfish HPE extension | 2 | 2 weeks | HPE iLO lab | H19 |
| Valkey rate limiter implementation | 2 | 2 weeks | Valkey deployment | H8 |
| Rate limiter safe mode + auto-activation | 3 | 1 week | Rate limiter core | H8 |
| RawConfigPush content analysis + audit | 3 | 2 weeks | RBAC implementation | H15 |
| RawConfigPush approval gate | 3 | 1 week | Temporal approval signals | H15 |
| Plugin signing infrastructure | 6 | 2 weeks | Key management | H16 |
| Plugin resource limits (cgroups v2) | 6 | 2 weeks | Linux deployment | H16 |
| Plugin capability declaration + verification | 6 | 1 week | go-plugin integration | H16 |
| Credential use monitoring | 6 | 1 week | Plugin framework | H16 |

---

## 9. References

- [ADAPTER-CONTRACT.md](ADAPTER-CONTRACT.md) -- Adapter interface definitions (requires updates per section 7.1)
- [OPERATION-TYPES.md](OPERATION-TYPES.md) -- Operation type taxonomy
- [DISCOVERY-CONTRACT.md](DISCOVERY-CONTRACT.md) -- Discovery behavioral contract (requires updates per section 7.3)
- [ERROR-MODEL.md](ERROR-MODEL.md) -- Error taxonomy (requires updates per section 7.2)
- [WORKFLOW-CONTRACT.md](WORKFLOW-CONTRACT.md) -- Approval gate pattern used by RawConfigPush
- [GAP-ANALYSIS.md](GAP-ANALYSIS.md) -- Source gaps: C7, H8, H15, H16, H19
- [RESEARCH-protocol-adapter-realism.md](RESEARCH-protocol-adapter-realism.md) -- Research document this spec closes
- [ADR-006](adr/ADR-006-adapter-families.md) -- Operation-family adapter model
- [ADR-012](adr/ADR-012-binary-ip-protection.md) -- Binary IP protection (plugin signing key infrastructure)
