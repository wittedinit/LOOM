# LOOM Domain Model

> Canonical resource model for LOOM — the universal infrastructure orchestrator.
> Every adapter, workflow, repository, and API references these types.

All IDs are `uuid.UUID`. All timestamps are `time.Time`. All models are tenant-scoped.

---

## Enums

```go
// DeviceType classifies a discovered piece of infrastructure.
type DeviceType string

const (
    DeviceTypeServer     DeviceType = "server"
    DeviceTypeSwitch     DeviceType = "switch"
    DeviceTypeRouter     DeviceType = "router"
    DeviceTypeFirewall   DeviceType = "firewall"
    DeviceTypeHypervisor DeviceType = "hypervisor"
    DeviceTypeKVM        DeviceType = "kvm"
    DeviceTypeStorage    DeviceType = "storage"
    DeviceTypeAppliance  DeviceType = "appliance"
    DeviceTypeUnknown    DeviceType = "unknown"
)

// DeviceStatus tracks the lifecycle state of a device within LOOM.
type DeviceStatus string

const (
    DeviceStatusDiscovered      DeviceStatus = "discovered"
    DeviceStatusActive          DeviceStatus = "active"
    DeviceStatusDegraded        DeviceStatus = "degraded"
    DeviceStatusUnreachable     DeviceStatus = "unreachable"
    DeviceStatusDecommissioned  DeviceStatus = "decommissioned"
)

// EndpointStatus tracks the reachability of a management endpoint.
type EndpointStatus string

const (
    EndpointStatusActive      EndpointStatus = "active"
    EndpointStatusUnreachable EndpointStatus = "unreachable"
    EndpointStatusDisabled    EndpointStatus = "disabled"
)

// CredentialType identifies the authentication mechanism a credential provides.
type CredentialType string

const (
    CredentialTypeSSHPassword   CredentialType = "ssh_password"
    CredentialTypeSSHKey        CredentialType = "ssh_key"
    CredentialTypeSNMPCommunity CredentialType = "snmp_community"
    CredentialTypeSNMPV3        CredentialType = "snmp_v3"
    CredentialTypeHTTPBasic     CredentialType = "http_basic"
    CredentialTypeHTTPBearer    CredentialType = "http_bearer"
    CredentialTypeCertificate   CredentialType = "certificate"
    CredentialTypeAPIKey        CredentialType = "api_key"
    CredentialTypeAMTDigest     CredentialType = "amt_digest"
)
```

---

## Target

An addressable endpoint that LOOM can attempt to reach. Before discovery, we know only how to reach it — not what it is.

Targets are the input to the discovery pipeline. A successful discovery produces a Device with one or more Endpoints.

```go
// Target represents an addressable endpoint that LOOM can attempt to reach.
// It exists before discovery — we know the address but not yet what lives there.
type Target struct {
    // ID is the unique identifier for this target.
    ID uuid.UUID `json:"id" db:"id"`

    // Address is the IP, hostname, or URL used to reach this target.
    Address string `json:"address" db:"address"`

    // Port is the optional port number. Zero means unspecified (adapter will use protocol default).
    Port int `json:"port,omitempty" db:"port"`

    // ProtocolHint is an optional hint about the expected protocol ("ssh", "redfish", "snmp", etc.).
    // When empty, LOOM will attempt auto-detection during discovery.
    ProtocolHint string `json:"protocol_hint,omitempty" db:"protocol_hint"`

    // TenantID scopes this target to a tenant.
    TenantID uuid.UUID `json:"tenant_id" db:"tenant_id"`

    CreatedAt time.Time `json:"created_at" db:"created_at"`
    UpdatedAt time.Time `json:"updated_at" db:"updated_at"`
}
```

---

## Device

A discovered, identified piece of infrastructure. Created after successful discovery of a Target.

The Device is the central node in LOOM's resource graph. Everything else — endpoints, facts, state, relationships — hangs off a Device.

```go
// Device represents a discovered, identified piece of infrastructure.
// Created when a Target is successfully probed during discovery.
type Device struct {
    // ID is the immutable LOOM-internal identity for this device.
    ID uuid.UUID `json:"id" db:"id"`

    // Name is a human-readable label. Mutable — operators can rename devices.
    Name string `json:"name" db:"name"`

    // Type classifies the device (server, switch, router, etc.).
    Type DeviceType `json:"type" db:"type"`

    // Vendor is the hardware manufacturer ("Dell", "Cisco", "Arista", etc.).
    Vendor string `json:"vendor" db:"vendor"`

    // Model is the hardware model identifier.
    Model string `json:"model" db:"model"`

    // SerialNumber is the manufacturer's serial number, if available.
    SerialNumber string `json:"serial_number,omitempty" db:"serial_number"`

    // FirmwareVersion is the currently reported firmware or OS version, if available.
    FirmwareVersion string `json:"firmware_version,omitempty" db:"firmware_version"`

    // Status tracks the device lifecycle within LOOM.
    Status DeviceStatus `json:"status" db:"status"`

    // TenantID scopes this device to a tenant.
    TenantID uuid.UUID `json:"tenant_id" db:"tenant_id"`

    // Metadata holds non-core attributes that do not warrant dedicated columns.
    // Examples: "rack_location": "A3-U12", "asset_tag": "SVR-00412".
    Metadata map[string]string `json:"metadata,omitempty" db:"-"`

    // OwnershipID is an optional foreign key to the DeviceOwnership record.
    // When set, multi-tenant access rules for this device are defined there.
    // When nil, the device is exclusively owned by TenantID.
    OwnershipID *uuid.UUID `json:"ownership_id,omitempty" db:"ownership_id"`

    CreatedAt time.Time `json:"created_at" db:"created_at"`
    UpdatedAt time.Time `json:"updated_at" db:"updated_at"`
}
```

---

## DeviceOwnership (C3 — Shared Device Model)

A physical switch, spine router, or shared storage array is typically owned by one tenant but may grant scoped access to others. `DeviceOwnership` captures this relationship without duplicating device records or complicating every SQL query with multi-tenant OR clauses.

> **Design rule:** A physical device can be owned by one tenant but grant scoped access to others. The `AuthorizedTenants` list defines who may interact with the device and what they may do.

```go
// AccessLevel defines what an authorized tenant may do with a shared device.
type AccessLevel string

const (
    AccessLevelFull      AccessLevel = "full"       // full read/write — same as owner
    AccessLevelReadOnly  AccessLevel = "read_only"  // observe only — no mutations
    AccessLevelPortScoped AccessLevel = "port_scoped" // access restricted to specific endpoints
)

// TenantAccess grants a specific tenant a defined level of access to a shared device.
type TenantAccess struct {
    // TenantID is the tenant being granted access.
    TenantID uuid.UUID `json:"tenant_id" db:"tenant_id"`

    // AccessLevel defines what this tenant may do (full, read_only, port_scoped).
    AccessLevel AccessLevel `json:"access_level" db:"access_level"`

    // AllowedEndpoints restricts access to specific endpoints when AccessLevel is port_scoped.
    // Empty means all endpoints are accessible (only meaningful for full/read_only).
    AllowedEndpoints []uuid.UUID `json:"allowed_endpoints,omitempty" db:"-"`

    GrantedAt time.Time `json:"granted_at" db:"granted_at"`
}

// DeviceOwnership defines the multi-tenant ownership and access model for a device.
// Every shared device (core switch, spine router, shared storage) should have one.
type DeviceOwnership struct {
    // ID is the unique identifier for this ownership record.
    ID uuid.UUID `json:"id" db:"id"`

    // DeviceID is a foreign key to the Device this ownership record governs.
    DeviceID uuid.UUID `json:"device_id" db:"device_id"`

    // OwnerTenantID is the tenant that physically owns or originally discovered this device.
    // The owner has full access and is the only tenant that can modify the ownership record.
    OwnerTenantID uuid.UUID `json:"owner_tenant_id" db:"owner_tenant_id"`

    // AuthorizedTenants lists other tenants that have been granted access to this device.
    // Each entry specifies the access level and optionally restricts to specific endpoints.
    AuthorizedTenants []TenantAccess `json:"authorized_tenants" db:"-"`

    CreatedAt time.Time `json:"created_at" db:"created_at"`
    UpdatedAt time.Time `json:"updated_at" db:"updated_at"`
}
```

> **Query pattern:** Repository implementations query shared devices with `WHERE device.tenant_id = ? OR device.id IN (SELECT device_id FROM device_ownership_tenants WHERE tenant_id = ?)`. The `DeviceOwnership` table is only consulted for devices that have `OwnershipID` set — single-tenant devices bypass this entirely.

---

## Endpoint

A management interface on a Device. A single device may expose multiple endpoints — SSH on the host OS, Redfish on the BMC, SNMP on a management VLAN, and so on.

Endpoints are the concrete thing an adapter talks to. Every adapter call targets a specific Endpoint.

```go
// Endpoint represents a management interface on a Device.
// A device may have many endpoints (SSH, Redfish, SNMP, etc.).
type Endpoint struct {
    // ID is the unique identifier for this endpoint.
    ID uuid.UUID `json:"id" db:"id"`

    // DeviceID is a foreign key to the parent Device.
    DeviceID uuid.UUID `json:"device_id" db:"device_id"`

    // Protocol identifies the management protocol.
    // Known values: "ssh", "redfish", "ipmi", "snmp", "netconf", "gnmi",
    // "eapi", "nxapi", "amt", "pikvm", "vsphere", "proxmox", "libvirt".
    Protocol string `json:"protocol" db:"protocol"`

    // Address is the IP or hostname for this specific endpoint.
    // May differ from the parent device's primary address (e.g., a BMC on a separate NIC).
    Address string `json:"address" db:"address"`

    // Port is the TCP/UDP port for this endpoint.
    Port int `json:"port" db:"port"`

    // CredentialRefID is an optional foreign key to the credential used for authentication.
    CredentialRefID *uuid.UUID `json:"credential_ref_id,omitempty" db:"credential_ref_id"`

    // Capabilities declares what this endpoint can do (power_control, discover, etc.).
    Capabilities []Capability `json:"capabilities" db:"-"`

    // Status tracks whether this endpoint is currently reachable.
    Status EndpointStatus `json:"status" db:"status"`

    // LastSeenAt records the last time this endpoint responded successfully.
    LastSeenAt time.Time `json:"last_seen_at" db:"last_seen_at"`

    // TenantID scopes this endpoint to a tenant.
    TenantID uuid.UUID `json:"tenant_id" db:"tenant_id"`

    CreatedAt time.Time `json:"created_at" db:"created_at"`
    UpdatedAt time.Time `json:"updated_at" db:"updated_at"`
}
```

---

## CredentialRef

A reference to encrypted credentials. The actual secret material is **never** loaded into this struct — only the reference metadata lives here.

The encrypted credential data is stored in a separate `credential_secrets` table, accessed only by the credential service at the moment of use.

```go
// CredentialRef is a reference to encrypted credentials.
// The actual credential value is NEVER in this struct — only the reference.
type CredentialRef struct {
    // ID is the unique identifier for this credential reference.
    ID uuid.UUID `json:"id" db:"id"`

    // Name is a human-readable label ("prod-bmc-admin", "switch-snmpv3", etc.).
    Name string `json:"name" db:"name"`

    // Type identifies the authentication mechanism.
    Type CredentialType `json:"type" db:"type"`

    // TenantID scopes this credential to a tenant.
    TenantID uuid.UUID `json:"tenant_id" db:"tenant_id"`

    CreatedAt time.Time `json:"created_at" db:"created_at"`
    UpdatedAt time.Time `json:"updated_at" db:"updated_at"`
}
```

> **Design rule:** The `credential_secrets` table stores the encrypted payload keyed by `CredentialRef.ID`. That table is never mapped to a Go struct that leaves the credential service boundary.

---

## IP Address Management (C6 — IPAM)

LOOM must prevent IP collisions between tenants and track address utilization. The strategy is **hybrid**: a minimal built-in pool allocator for standalone deployments, with an adapter interface for external IPAM systems (Infoblox, NetBox, phpIPAM).

> **Integration strategy:** External IPAM systems are the source of truth when connected. The built-in pool serves as a fallback for standalone or air-gapped deployments. The `IPAMAdapter` interface abstracts the backend.

```go
// IPPool represents a block of IP addresses available for allocation.
type IPPool struct {
    // ID is the unique identifier for this pool.
    ID uuid.UUID `json:"id" db:"id"`

    // TenantID scopes this pool to a tenant.
    TenantID uuid.UUID `json:"tenant_id" db:"tenant_id"`

    // CIDR is the network prefix for this pool (e.g., "10.1.0.0/24").
    CIDR string `json:"cidr" db:"cidr"`

    // Gateway is the default gateway address within this pool.
    Gateway string `json:"gateway" db:"gateway"`

    // Available is the count of unallocated addresses in this pool.
    Available int `json:"available" db:"available"`

    // Allocated is the count of addresses currently in use.
    Allocated int `json:"allocated" db:"allocated"`

    CreatedAt time.Time `json:"created_at" db:"created_at"`
    UpdatedAt time.Time `json:"updated_at" db:"updated_at"`
}

// IPAllocation records a single IP address assignment.
type IPAllocation struct {
    // ID is the unique identifier for this allocation.
    ID uuid.UUID `json:"id" db:"id"`

    // Address is the allocated IP address (e.g., "10.1.0.5").
    Address string `json:"address" db:"address"`

    // Prefix is the CIDR prefix this address belongs to (e.g., "10.1.0.0/24").
    Prefix string `json:"prefix" db:"prefix"`

    // TenantID scopes this allocation to a tenant.
    TenantID uuid.UUID `json:"tenant_id" db:"tenant_id"`

    // DeviceID is the device this address is assigned to (nullable).
    DeviceID *uuid.UUID `json:"device_id,omitempty" db:"device_id"`

    // EndpointID is the specific endpoint this address is assigned to (nullable).
    EndpointID *uuid.UUID `json:"endpoint_id,omitempty" db:"endpoint_id"`

    // Pool is the foreign key to the IPPool this allocation came from.
    Pool uuid.UUID `json:"pool" db:"pool_id"`

    // AllocatedAt is the time this address was assigned.
    AllocatedAt time.Time `json:"allocated_at" db:"allocated_at"`

    // ReleasedAt is the time this address was returned to the pool (nil if still active).
    ReleasedAt *time.Time `json:"released_at,omitempty" db:"released_at"`
}
```

> **Concurrent allocation safety:** Two concurrent workflows allocating from the same pool must not collide. The built-in allocator uses `SELECT ... FOR UPDATE` on pool rows to serialize allocations within a transaction. External IPAM adapters delegate concurrency control to the external system.

---

## Capability

What an adapter endpoint can do. Declared by adapters at registration time and attached to Endpoints.

```go
// Capability declares a single thing an adapter endpoint can do.
type Capability struct {
    // Name identifies the capability.
    // Known values: "power_control", "discover", "config_read", "config_write",
    // "watch", "boot_control", "console", "firmware_update".
    Name string `json:"name"`

    // Protocol is the protocol this capability operates over.
    Protocol string `json:"protocol"`

    // ReadOnly indicates whether this capability only reads state (true)
    // or can also mutate it (false).
    ReadOnly bool `json:"read_only"`
}
```

---

## Physical Topology (C8)

Physical topology types model the datacenter infrastructure that devices live in. These types enable failure domain analysis — determining which devices share power circuits, cooling zones, or top-of-rack switches.

> **Design rule:** Physical topology is optional. LOOM must function without it (cloud-only deployments have no racks). When present, it enables smarter placement decisions and blast-radius analysis.

```go
// Site represents a physical location (datacenter, colo, edge site).
type Site struct {
    // ID is the unique identifier for this site.
    ID uuid.UUID `json:"id" db:"id"`

    // TenantID scopes this site to a tenant.
    TenantID uuid.UUID `json:"tenant_id" db:"tenant_id"`

    // Name is a human-readable label (e.g., "us-east-1", "london-dc2").
    Name string `json:"name" db:"name"`

    // Location is the physical address or coordinates.
    Location string `json:"location" db:"location"`

    // Timezone is the IANA timezone for this site (e.g., "America/New_York").
    Timezone string `json:"timezone" db:"timezone"`

    CreatedAt time.Time `json:"created_at" db:"created_at"`
    UpdatedAt time.Time `json:"updated_at" db:"updated_at"`
}

// Rack represents a physical equipment rack within a site.
type Rack struct {
    // ID is the unique identifier for this rack.
    ID uuid.UUID `json:"id" db:"id"`

    // SiteID is a foreign key to the Site containing this rack.
    SiteID uuid.UUID `json:"site_id" db:"site_id"`

    // Name is the rack label (e.g., "A3", "Row2-Rack5").
    Name string `json:"name" db:"name"`

    // TotalUnits is the total rack units available (e.g., 42).
    TotalUnits int `json:"total_units" db:"total_units"`

    // PowerCircuits lists the power feeds serving this rack.
    PowerCircuits []uuid.UUID `json:"power_circuits" db:"-"`

    // TenantID scopes this rack to a tenant.
    TenantID uuid.UUID `json:"tenant_id" db:"tenant_id"`

    CreatedAt time.Time `json:"created_at" db:"created_at"`
    UpdatedAt time.Time `json:"updated_at" db:"updated_at"`
}

// RackMount records where a device is physically installed in a rack.
type RackMount struct {
    // ID is the unique identifier for this mount record.
    ID uuid.UUID `json:"id" db:"id"`

    // RackID is a foreign key to the Rack.
    RackID uuid.UUID `json:"rack_id" db:"rack_id"`

    // DeviceID is a foreign key to the Device installed at this position.
    DeviceID uuid.UUID `json:"device_id" db:"device_id"`

    // StartUnit is the lowest rack unit occupied (e.g., 12 for U12).
    StartUnit int `json:"start_unit" db:"start_unit"`

    // HeightUnits is the number of rack units the device occupies (e.g., 2 for a 2U server).
    HeightUnits int `json:"height_units" db:"height_units"`

    // Orientation is the device facing (front or rear).
    Orientation string `json:"orientation" db:"orientation"`

    // TenantID scopes this mount to a tenant.
    TenantID uuid.UUID `json:"tenant_id" db:"tenant_id"`
}

// PowerCircuit represents a power feed to a rack or group of racks.
type PowerCircuit struct {
    // ID is the unique identifier for this circuit.
    ID uuid.UUID `json:"id" db:"id"`

    // SiteID is a foreign key to the Site containing this circuit.
    SiteID uuid.UUID `json:"site_id" db:"site_id"`

    // RackID is the rack this circuit feeds (nullable — a circuit may feed multiple racks via PDU).
    RackID *uuid.UUID `json:"rack_id,omitempty" db:"rack_id"`

    // CircuitID is the operator's label for this circuit (e.g., "PDU-A3-L1").
    CircuitID string `json:"circuit_id" db:"circuit_id"`

    // CapacityWatts is the maximum power draw for this circuit.
    CapacityWatts int `json:"capacity_watts" db:"capacity_watts"`

    // RedundancyGroup groups circuits that share a failure domain (e.g., "feed-A", "feed-B").
    RedundancyGroup string `json:"redundancy_group" db:"redundancy_group"`

    // TenantID scopes this circuit to a tenant.
    TenantID uuid.UUID `json:"tenant_id" db:"tenant_id"`
}

// CableConnection records a physical cable between two device ports.
// Auto-populated from LLDP/CDP discovery or entered manually.
type CableConnection struct {
    // ID is the unique identifier for this cable record.
    ID uuid.UUID `json:"id" db:"id"`

    // SourceDeviceID is the device at one end of the cable.
    SourceDeviceID uuid.UUID `json:"source_device_id" db:"source_device_id"`

    // SourcePort is the port name on the source device (e.g., "Ethernet1/1").
    SourcePort string `json:"source_port" db:"source_port"`

    // DestDeviceID is the device at the other end of the cable.
    DestDeviceID uuid.UUID `json:"dest_device_id" db:"dest_device_id"`

    // DestPort is the port name on the destination device (e.g., "Gi1/0/1").
    DestPort string `json:"dest_port" db:"dest_port"`

    // CableType is the physical media type (fiber-sm, fiber-mm, cat6, dac, etc.).
    CableType string `json:"cable_type" db:"cable_type"`

    // Length is the cable length with unit (e.g., "3m", "10ft").
    Length string `json:"length" db:"length"`

    // TenantID scopes this cable to a tenant.
    TenantID uuid.UUID `json:"tenant_id" db:"tenant_id"`

    CreatedAt time.Time `json:"created_at" db:"created_at"`
    UpdatedAt time.Time `json:"updated_at" db:"updated_at"`
}
```

---

## ObservedFact

A timestamped piece of information discovered about a device from an adapter. ObservedFacts are **immutable after creation** — they represent what was true at a specific moment in time.

New observations do not update old facts; they create new rows. This gives LOOM a full audit trail of everything ever observed.

```go
// ObservedFact is an immutable, timestamped piece of information about a device.
// Created by adapters during discovery, polling, or event-driven updates.
type ObservedFact struct {
    // ID is the unique identifier for this fact.
    ID uuid.UUID `json:"id" db:"id"`

    // DeviceID is a foreign key to the Device this fact describes.
    DeviceID uuid.UUID `json:"device_id" db:"device_id"`

    // EndpointID identifies which endpoint reported this fact.
    EndpointID uuid.UUID `json:"endpoint_id" db:"endpoint_id"`

    // FactType categorizes the observation.
    // Examples: "os_type", "hostname", "cpu_count", "memory_gb",
    // "interface_list", "power_state", "bmc_version", "sys_descr".
    FactType string `json:"fact_type" db:"fact_type"`

    // Value is the observed value, always string-encoded.
    // Structured data (e.g., interface lists) is JSON-encoded into this field.
    Value string `json:"value" db:"value"`

    // Confidence indicates how reliable this fact is, from 0.0 (no confidence) to 1.0 (certain).
    // An SNMP sysDescr match might be 0.7; a Redfish serial number read is 1.0.
    Confidence float64 `json:"confidence" db:"confidence"`

    // Source is the adapter name that produced this fact (e.g., "redfish-adapter", "snmp-adapter").
    Source string `json:"source" db:"source"`

    // ObservedAt is the time this fact was observed.
    ObservedAt time.Time `json:"observed_at" db:"observed_at"`

    // TenantID scopes this fact to a tenant.
    TenantID uuid.UUID `json:"tenant_id" db:"tenant_id"`
}
```

---

## StateSnapshot

Full observed state of a device at a point in time. An aggregation of ObservedFacts captured together, used for diffing, auditing, and verification.

```go
// StateSnapshot is a point-in-time collection of ObservedFacts for a device.
type StateSnapshot struct {
    // ID is the unique identifier for this snapshot.
    ID uuid.UUID `json:"id" db:"id"`

    // DeviceID is a foreign key to the Device this snapshot describes.
    DeviceID uuid.UUID `json:"device_id" db:"device_id"`

    // Facts is the set of observations included in this snapshot.
    Facts []ObservedFact `json:"facts" db:"-"`

    // CapturedAt is the time this snapshot was taken.
    CapturedAt time.Time `json:"captured_at" db:"captured_at"`

    // CapturedBy identifies the adapter or workflow that triggered this capture.
    CapturedBy string `json:"captured_by" db:"captured_by"`

    // TenantID scopes this snapshot to a tenant.
    TenantID uuid.UUID `json:"tenant_id" db:"tenant_id"`
}
```

---

## Firmware Lifecycle (C9)

`Device.FirmwareVersion` records what is currently running, but the domain model must also track update targets, compatibility constraints, and the update workflow itself.

> **Firmware update workflow:** stage --> validate --> apply --> verify --> rollback-on-failure. Each step is a Temporal activity with compensation. A failed post-update verification triggers automatic rollback to the previous version.

```go
// FirmwareStatus tracks where a firmware version stands in the update lifecycle.
type FirmwareStatus string

const (
    FirmwareStatusCurrent  FirmwareStatus = "current"  // actively running on the device
    FirmwareStatusStaging  FirmwareStatus = "staging"   // downloaded, pending apply
    FirmwareStatusRollback FirmwareStatus = "rollback"  // previous version retained for rollback
)

// FirmwareRecord tracks the firmware state of a specific device.
type FirmwareRecord struct {
    // ID is the unique identifier for this record.
    ID uuid.UUID `json:"id" db:"id"`

    // DeviceID is a foreign key to the Device.
    DeviceID uuid.UUID `json:"device_id" db:"device_id"`

    // CurrentVersion is the firmware version currently running.
    CurrentVersion string `json:"current_version" db:"current_version"`

    // TargetVersion is the firmware version being staged or applied (empty if up-to-date).
    TargetVersion string `json:"target_version,omitempty" db:"target_version"`

    // Status tracks the lifecycle position (current, staging, rollback).
    Status FirmwareStatus `json:"status" db:"status"`

    // Component identifies what this firmware applies to (bios, bmc, nic, os).
    Component string `json:"component" db:"component"`

    // TenantID scopes this record to a tenant.
    TenantID uuid.UUID `json:"tenant_id" db:"tenant_id"`

    LastCheckedAt time.Time `json:"last_checked_at" db:"last_checked_at"`
    UpdatedAt     time.Time `json:"updated_at" db:"updated_at"`
}

// FirmwareCompatibility defines version constraints for a device model.
// Used to validate that a target version is safe to apply before staging.
type FirmwareCompatibility struct {
    // ID is the unique identifier for this compatibility rule.
    ID uuid.UUID `json:"id" db:"id"`

    // DeviceModel is the hardware model this rule applies to (e.g., "PowerEdge R750").
    DeviceModel string `json:"device_model" db:"device_model"`

    // Component identifies what firmware component this applies to.
    Component string `json:"component" db:"component"`

    // MinVersion is the minimum acceptable firmware version.
    MinVersion string `json:"min_version" db:"min_version"`

    // MaxVersion is the maximum acceptable firmware version (empty if no ceiling).
    MaxVersion string `json:"max_version,omitempty" db:"max_version"`

    // Dependencies lists other firmware components that must be at specific versions first.
    // Example: BIOS update may require BMC >= 5.0.
    Dependencies map[string]string `json:"dependencies,omitempty" db:"-"`

    // TenantID scopes this rule to a tenant (or global if zero-value).
    TenantID uuid.UUID `json:"tenant_id" db:"tenant_id"`
}
```

> **Operation type:** Firmware updates are executed as `FirmwareUpdateOp` operations within Temporal workflows. The operation references a `FirmwareRecord` for the device, validates against `FirmwareCompatibility` rules, and follows the stage/validate/apply/verify/rollback lifecycle.

---

## DesiredState

What LOOM wants a resource to look like. Declared by workflows, policies, or the LLM engine and used by the verification pipeline to detect drift.

### DesiredProperty (H4 — Typed Properties)

Properties are no longer `map[string]string`. Each property carries its type, enabling type-aware comparison with `ObservedFact` values.

```go
// PropertyType defines the data type of a desired property value.
type PropertyType string

const (
    PropertyTypeString PropertyType = "string"
    PropertyTypeInt    PropertyType = "int"
    PropertyTypeBool   PropertyType = "bool"
    PropertyTypeJSON   PropertyType = "json"
)

// PropertySource identifies who declared this property value.
type PropertySource string

const (
    PropertySourceUser    PropertySource = "user"    // operator set this explicitly
    PropertySourcePolicy  PropertySource = "policy"  // compliance policy requires this
    PropertySourceLLM     PropertySource = "llm"     // LLM decision engine recommended this
    PropertySourceDefault PropertySource = "default" // system default
)

// DesiredProperty is a single typed property within a DesiredState declaration.
type DesiredProperty struct {
    // Key is the property name (e.g., "power_state", "mtu", "vlan_id").
    Key string `json:"key" db:"key"`

    // Value is the desired value. The actual Go type depends on Type.
    Value any `json:"value" db:"value"`

    // Type declares the data type for type-aware comparison.
    Type PropertyType `json:"type" db:"type"`

    // Source identifies who declared this property (user, policy, llm, default).
    Source PropertySource `json:"source" db:"source"`
}
```

> **Comparison rules per type:**
> - `string`: exact match after case-normalization and whitespace trim.
> - `int`: parse both sides as int64, compare numerically (tolerates string-encoded integers in ObservedFact).
> - `bool`: parse both sides as boolean (accepts "true"/"false", "1"/"0", "on"/"off").
> - `json`: deep-equal after JSON normalization (key ordering, whitespace ignored).

```go
// DesiredState declares the intended state of a resource.
// Workflows, policies, and the LLM engine declare desired state;
// the verification pipeline checks it against observed facts.
type DesiredState struct {
    // ID is the unique identifier for this desired state declaration.
    ID uuid.UUID `json:"id" db:"id"`

    // ResourceType is the kind of resource ("device", "vlan", "interface", "acl").
    ResourceType string `json:"resource_type" db:"resource_type"`

    // ResourceID is the UUID of the resource this desired state applies to.
    ResourceID uuid.UUID `json:"resource_id" db:"resource_id"`

    // Properties is the set of typed desired property values.
    Properties []DesiredProperty `json:"properties" db:"-"`

    // Priority determines conflict resolution order (higher wins).
    // Convention: user=100, policy=80, llm=60, default=40.
    Priority int `json:"priority" db:"priority"`

    // SourceType identifies the origin of this declaration (user, policy, llm, default).
    SourceType PropertySource `json:"source_type" db:"source_type"`

    // DeclaredBy is the workflow ID, policy ID, or user ID that declared this desired state.
    DeclaredBy string `json:"declared_by" db:"declared_by"`

    // DeclaredAt is the time this desired state was declared.
    DeclaredAt time.Time `json:"declared_at" db:"declared_at"`

    // Version is a monotonically increasing counter for last-write-wins within the same priority.
    Version int `json:"version" db:"version"`

    // ActiveUntil is an optional expiration time. Nil means permanent.
    // Expired DesiredState is automatically ignored by the verification pipeline.
    ActiveUntil *time.Time `json:"active_until,omitempty" db:"active_until"`

    // TenantID scopes this desired state to a tenant.
    TenantID uuid.UUID `json:"tenant_id" db:"tenant_id"`
}
```

### DesiredState Conflict Resolution (H7)

When multiple DesiredState declarations target the same resource property, conflicts are resolved deterministically:

1. **Higher priority wins.** Priority order: `user` (100) > `policy` (80) > `llm` (60) > `default` (40).
2. **Last-write-wins within same priority level.** The declaration with the higher `Version` prevails.
3. **Expired declarations are ignored.** Any DesiredState with `ActiveUntil < now()` is excluded before conflict resolution.

When a conflict is detected, a `DesiredStateConflict` event is emitted to NATS for observability.

```go
// DesiredStateConflict is emitted as a NATS event when two DesiredState declarations
// target the same resource property with different values.
type DesiredStateConflict struct {
    // ResourceType is the kind of resource in conflict.
    ResourceType string `json:"resource_type"`

    // ResourceID is the UUID of the resource in conflict.
    ResourceID uuid.UUID `json:"resource_id"`

    // PropertyKey is the property name that has conflicting values.
    PropertyKey string `json:"property_key"`

    // WinnerID is the DesiredState ID that prevailed.
    WinnerID uuid.UUID `json:"winner_id"`

    // LoserID is the DesiredState ID that was overridden.
    LoserID uuid.UUID `json:"loser_id"`

    // Reason describes why the winner won (e.g., "higher_priority", "later_version").
    Reason string `json:"reason"`

    // DetectedAt is the time the conflict was detected.
    DetectedAt time.Time `json:"detected_at"`

    // TenantID scopes this event to a tenant.
    TenantID uuid.UUID `json:"tenant_id"`
}
```

---

## VerificationResult

Result of checking whether observed state matches desired state. Produced by the verification pipeline and consumed by workflows to decide pass/fail.

```go
// VerificationResult records the outcome of a single state check.
type VerificationResult struct {
    // ID is the unique identifier for this result.
    ID uuid.UUID `json:"id" db:"id"`

    // WorkflowID is the Temporal workflow ID that requested this verification.
    WorkflowID string `json:"workflow_id" db:"workflow_id"`

    // CheckName describes what was checked ("power_state_check", "vlan_exists", "ssh_reachable").
    CheckName string `json:"check_name" db:"check_name"`

    // Expected is the desired value that was checked against.
    Expected string `json:"expected" db:"expected"`

    // Observed is the actual value found during verification.
    Observed string `json:"observed" db:"observed"`

    // Passed indicates whether the check succeeded (observed matches expected).
    Passed bool `json:"passed" db:"passed"`

    // Evidence is the raw output or response that proves the result.
    // May contain CLI output, API response bodies, or SNMP walk results.
    Evidence string `json:"evidence" db:"evidence"`

    // CheckedAt is the time the verification was performed.
    CheckedAt time.Time `json:"checked_at" db:"checked_at"`

    // TenantID scopes this result to a tenant.
    TenantID uuid.UUID `json:"tenant_id" db:"tenant_id"`
}
```

---

## Relationship

A typed edge between two resources. Used to build and query the topology graph.

```go
// Relationship is a typed, directed edge between two resources in the topology graph.
type Relationship struct {
    // ID is the unique identifier for this relationship.
    ID uuid.UUID `json:"id" db:"id"`

    // SourceType is the kind of the source resource ("device", "endpoint", "vlan", "subnet").
    SourceType string `json:"source_type" db:"source_type"`

    // SourceID is the UUID of the source resource.
    SourceID uuid.UUID `json:"source_id" db:"source_id"`

    // TargetType is the kind of the target resource.
    TargetType string `json:"target_type" db:"target_type"`

    // TargetID is the UUID of the target resource.
    TargetID uuid.UUID `json:"target_id" db:"target_id"`

    // RelationType classifies the edge.
    // Known values: "connected_to", "runs_on", "member_of", "depends_on", "manages", "peers_with".
    RelationType string `json:"relation_type" db:"relation_type"`

    // Properties holds edge-specific metadata.
    // Examples: "port": "Ethernet1/1", "vlan": "100", "bandwidth": "10G".
    Properties map[string]string `json:"properties,omitempty" db:"-"`

    // TenantID scopes this relationship to a tenant.
    TenantID uuid.UUID `json:"tenant_id" db:"tenant_id"`

    CreatedAt time.Time `json:"created_at" db:"created_at"`
    UpdatedAt time.Time `json:"updated_at" db:"updated_at"`
}
```

---

## Storage Model (H20 — Phase 6+ Scope)

Storage types are defined now for schema planning but will not be implemented until Phase 6+. LOOM will observe storage state via adapters (CSI, Trident, NetApp, Pure, Ceph) rather than managing storage directly.

> **Scope:** These types are skeletal. Direct storage array management is deferred to Phase 8+. Phase 6 adds the types to the schema and read-only observation via adapters.

```go
// StorageProtocol identifies the access protocol for a storage pool.
type StorageProtocol string

const (
    StorageProtocolISCSI StorageProtocol = "iscsi"
    StorageProtocolNFS   StorageProtocol = "nfs"
    StorageProtocolFC    StorageProtocol = "fc"
    StorageProtocolS3    StorageProtocol = "s3"
)

// VolumeType classifies the storage access pattern.
type VolumeType string

const (
    VolumeTypeBlock  VolumeType = "block"
    VolumeTypeFile   VolumeType = "file"
    VolumeTypeObject VolumeType = "object"
)

// StoragePool represents a pool of storage capacity on a device or array.
type StoragePool struct {
    // ID is the unique identifier for this pool.
    ID uuid.UUID `json:"id" db:"id"`

    // TenantID scopes this pool to a tenant.
    TenantID uuid.UUID `json:"tenant_id" db:"tenant_id"`

    // Name is a human-readable label for this pool.
    Name string `json:"name" db:"name"`

    // TotalGB is the total capacity of this pool in gigabytes.
    TotalGB int64 `json:"total_gb" db:"total_gb"`

    // AvailableGB is the remaining capacity in gigabytes.
    AvailableGB int64 `json:"available_gb" db:"available_gb"`

    // Protocol is the access protocol (iSCSI, NFS, FC, S3).
    Protocol StorageProtocol `json:"protocol" db:"protocol"`

    CreatedAt time.Time `json:"created_at" db:"created_at"`
    UpdatedAt time.Time `json:"updated_at" db:"updated_at"`
}

// Volume represents a logical storage volume carved from a StoragePool.
type Volume struct {
    // ID is the unique identifier for this volume.
    ID uuid.UUID `json:"id" db:"id"`

    // TenantID scopes this volume to a tenant.
    TenantID uuid.UUID `json:"tenant_id" db:"tenant_id"`

    // DeviceID is the device this volume is attached to (nullable).
    DeviceID *uuid.UUID `json:"device_id,omitempty" db:"device_id"`

    // Name is a human-readable label.
    Name string `json:"name" db:"name"`

    // SizeGB is the provisioned size in gigabytes.
    SizeGB int64 `json:"size_gb" db:"size_gb"`

    // Type classifies the access pattern (block, file, object).
    Type VolumeType `json:"type" db:"type"`

    // StoragePoolID is a foreign key to the parent StoragePool.
    StoragePoolID uuid.UUID `json:"storage_pool_id" db:"storage_pool_id"`

    CreatedAt time.Time `json:"created_at" db:"created_at"`
    UpdatedAt time.Time `json:"updated_at" db:"updated_at"`
}
```

---

## Tenant Scoping

Tenant isolation is not a separate type — it is a **design rule** that pervades every layer of LOOM.

| Rule | Detail |
|------|--------|
| **Every model carries `TenantID`** | Every struct above includes a `TenantID uuid.UUID` field. |
| **All queries filter by tenant** | Repository implementations must add `WHERE tenant_id = ?` to every query. No exceptions. |
| **Events carry tenant context** | Every published event (NATS subject, Temporal workflow input) includes `TenantID`. |
| **Workflows execute within tenant scope** | A workflow receives `TenantID` at creation and passes it to every activity. |
| **Cross-tenant access is forbidden** | No API, adapter, or workflow may read or write another tenant's resources. |

---

## Type Relationships

The following diagram describes how the domain types relate to each other.

```
Target
  |
  | (discovery produces)
  v
Device ──────────────────────────────── Relationship
  |                                       (connects any two resources)
  |-- has optional --> DeviceOwnership (C3: multi-tenant shared access)
  |-- has many --> Endpoint
  |                  |
  |                  |-- has optional --> CredentialRef
  |                  |-- declares --> Capability (embedded, not a table)
  |                  |-- produces --> ObservedFact
  |
  |-- has many --> ObservedFact
  |-- has many --> StateSnapshot (aggregates ObservedFacts)
  |-- has many --> DesiredState (typed properties, conflict resolution)
  |-- has many --> VerificationResult (via workflow)
  |-- has many --> FirmwareRecord (C9: firmware lifecycle)
  |-- has optional --> RackMount --> Rack --> Site (C8: physical topology)
  |-- has many --> IPAllocation (C6: IPAM)
  |-- has many --> Volume (H20: storage, Phase 6+)

IPPool -- has many --> IPAllocation
StoragePool -- has many --> Volume
Site -- has many --> Rack
Rack -- has many --> PowerCircuit
CableConnection -- connects --> Device (source) to Device (dest)
```

### Cardinalities

| Parent | Child | Cardinality | Notes |
|--------|-------|-------------|-------|
| Target | Device | 1:0..1 | A target may or may not produce a device after discovery. |
| Device | DeviceOwnership | 1:0..1 | Only shared devices have an ownership record. Single-tenant devices have none. |
| DeviceOwnership | TenantAccess | 1:N | Each authorized tenant gets a separate access entry. |
| Device | Endpoint | 1:N | A device has at least one endpoint after discovery. |
| Endpoint | CredentialRef | N:0..1 | Many endpoints may share one credential ref. An endpoint may have no credential. |
| Endpoint | Capability | 1:N | Capabilities are embedded on the endpoint, not a separate table. |
| Endpoint | ObservedFact | 1:N | Each fact records which endpoint reported it. |
| Device | ObservedFact | 1:N | Facts also reference the parent device for direct queries. |
| Device | StateSnapshot | 1:N | Snapshots accumulate over time. |
| StateSnapshot | ObservedFact | 1:N | A snapshot aggregates facts (join table: `snapshot_facts`). |
| Device | DesiredState | 1:N | Multiple desired states may apply to a device (one per property group). |
| DesiredState | VerificationResult | 1:N | Each verification run produces results checked against desired state. |
| Relationship | Device/Endpoint/etc. | N:N | Relationships are a generic edge table connecting any two resources. |
| Device | FirmwareRecord | 1:N | One record per firmware component (BIOS, BMC, NIC, etc.). |
| Device | RackMount | 1:0..1 | A device is in at most one rack position. |
| Rack | RackMount | 1:N | A rack holds many devices. |
| Site | Rack | 1:N | A site contains many racks. |
| Site | PowerCircuit | 1:N | A site has many power circuits. |
| IPPool | IPAllocation | 1:N | A pool contains many allocations. |
| Device | IPAllocation | 1:N | A device may have many IP allocations. |
| StoragePool | Volume | 1:N | A pool contains many volumes. |
| Device | Volume | 1:N | A device may have many attached volumes. |
| Device | CableConnection | N:N | Cable connections link two devices (source and dest). |

### Lifecycle Flow

1. **Operator registers a Target** (address + optional protocol hint).
2. **Discovery workflow probes the Target**, creating a **Device** and one or more **Endpoints**.
3. Each Endpoint declares its **Capabilities** and is optionally linked to a **CredentialRef**.
4. Adapters interact with Endpoints, producing **ObservedFacts**.
5. Facts are aggregated into **StateSnapshots** for point-in-time auditing.
6. Workflows declare **DesiredState** for resources.
7. The verification pipeline compares ObservedFacts against DesiredState, producing **VerificationResults**.
8. **Relationships** are created as topology is discovered (switch-to-server links, hypervisor-to-VM, etc.).
