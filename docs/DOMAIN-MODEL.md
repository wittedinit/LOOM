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

    CreatedAt time.Time `json:"created_at" db:"created_at"`
    UpdatedAt time.Time `json:"updated_at" db:"updated_at"`
}
```

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

## DesiredState

What LOOM wants a resource to look like. Declared by workflows and used by the verification pipeline to detect drift.

```go
// DesiredState declares the intended state of a resource.
// Workflows declare desired state; the verification pipeline checks it against observed facts.
type DesiredState struct {
    // ID is the unique identifier for this desired state declaration.
    ID uuid.UUID `json:"id" db:"id"`

    // ResourceType is the kind of resource ("device", "vlan", "interface", "acl").
    ResourceType string `json:"resource_type" db:"resource_type"`

    // ResourceID is the UUID of the resource this desired state applies to.
    ResourceID uuid.UUID `json:"resource_id" db:"resource_id"`

    // Properties maps property names to their desired values.
    // Examples: "power_state": "on", "vlan_id": "100", "mtu": "9000".
    Properties map[string]string `json:"properties" db:"-"`

    // DeclaredBy is the workflow ID that declared this desired state.
    DeclaredBy string `json:"declared_by" db:"declared_by"`

    // DeclaredAt is the time this desired state was declared.
    DeclaredAt time.Time `json:"declared_at" db:"declared_at"`

    // TenantID scopes this desired state to a tenant.
    TenantID uuid.UUID `json:"tenant_id" db:"tenant_id"`
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
  |-- has many --> Endpoint
  |                  |
  |                  |-- has optional --> CredentialRef
  |                  |-- declares --> Capability (embedded, not a table)
  |                  |-- produces --> ObservedFact
  |
  |-- has many --> ObservedFact
  |-- has many --> StateSnapshot (aggregates ObservedFacts)
  |-- has many --> DesiredState
  |-- has many --> VerificationResult (via workflow)
```

### Cardinalities

| Parent | Child | Cardinality | Notes |
|--------|-------|-------------|-------|
| Target | Device | 1:0..1 | A target may or may not produce a device after discovery. |
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

### Lifecycle Flow

1. **Operator registers a Target** (address + optional protocol hint).
2. **Discovery workflow probes the Target**, creating a **Device** and one or more **Endpoints**.
3. Each Endpoint declares its **Capabilities** and is optionally linked to a **CredentialRef**.
4. Adapters interact with Endpoints, producing **ObservedFacts**.
5. Facts are aggregated into **StateSnapshots** for point-in-time auditing.
6. Workflows declare **DesiredState** for resources.
7. The verification pipeline compares ObservedFacts against DesiredState, producing **VerificationResults**.
8. **Relationships** are created as topology is discovered (switch-to-server links, hypervisor-to-VM, etc.).
