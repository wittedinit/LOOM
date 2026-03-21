# Domain Model Addendum

> **Status:** Definitive — all design decisions final
> **Closes:** GAP-ANALYSIS.md findings C3, C6, C8, C9, H2, H4, H7, H20
> **Supersedes:** Open questions in RESEARCH-domain-model-gaps.md (all now resolved)
> **Last updated:** 2026-03-21

This document provides implementation-ready type definitions and SQL schemas for eight domain model gaps identified in RESEARCH-domain-model-gaps.md. Every open question is answered with a definitive decision. No further design review is needed — these are ready for Phase 1B migration.

---

## Table of Contents

1. [Multi-Tenant Shared Device Model (C3)](#1-multi-tenant-shared-device-model-c3)
2. [IPAM Model (C6)](#2-ipam-model-c6)
3. [Physical Topology Model (C8)](#3-physical-topology-model-c8)
4. [Firmware Lifecycle Model (C9)](#4-firmware-lifecycle-model-c9)
5. [Identity Matcher Concurrency (H2)](#5-identity-matcher-concurrency-h2)
6. [DesiredState Comparison Specification (H4)](#6-desiredstate-comparison-specification-h4)
7. [DesiredState Conflict Resolution (H7)](#7-desiredstate-conflict-resolution-h7)
8. [Storage Model (H20)](#8-storage-model-h20)

---

## 1. Multi-Tenant Shared Device Model (C3)

### Decision

**Option B — Virtual Device Records.** The existing `Device` type is decomposed into two types: `PhysicalDevice` (hardware identity, tenant-agnostic) and `TenantDeviceView` (tenant-scoped projection). Each tenant interacts exclusively with their `TenantDeviceView`. The `PhysicalDevice` is an internal record that tenants never query directly.

### Resolved Open Questions

| Question | Decision |
|----------|----------|
| How does discovery match a new device to an existing PhysicalDevice? | Admin-only cross-tenant linking. Discovery always creates within the discovering tenant's scope. An administrator can subsequently link a TenantDeviceView to an existing PhysicalDevice if the serial number or BIOS UUID matches. Automated cross-tenant matching is forbidden — it would leak the existence of devices between tenants. |
| Who can create a TenantDeviceView for an existing PhysicalDevice? | Platform administrators only. Tenants can request shared access through a support workflow, but the linkage is performed by an admin who can see all PhysicalDevice records. |
| How are TenantDeviceViews notified of PhysicalDevice state changes? | Event fan-out. When an adapter updates PhysicalDevice state (power state, reachability, hardware health), the system emits one `device.physical_state_changed` event per linked TenantDeviceView. Each event is scoped to the tenant's NATS subject. |
| Do TenantDeviceViews have independent DesiredState? | Yes. Each tenant declares DesiredState against their TenantDeviceView. The physical device has no DesiredState of its own. Conflict resolution between tenants on shared hardware is handled by the device locking mechanism (DEVICE-LOCKING.md), not by the data model. |

### Go Types

```go
// PhysicalDevice represents the hardware-level identity of a piece of infrastructure.
// This is an internal record — tenants do not query this table directly.
// It exists to deduplicate physical hardware shared across tenants.
type PhysicalDevice struct {
    // ID is the stable hardware identity. Survives tenant changes.
    ID uuid.UUID `json:"id" db:"id"`

    // SerialNumber is the manufacturer's serial number.
    // Primary deduplication key for cross-tenant matching.
    SerialNumber string `json:"serial_number" db:"serial_number"`

    // BIOSUUID is the SMBIOS/DMI UUID, if available.
    BIOSUUID *string `json:"bios_uuid,omitempty" db:"bios_uuid"`

    // Vendor is the hardware manufacturer.
    Vendor string `json:"vendor" db:"vendor"`

    // Model is the hardware model identifier.
    Model string `json:"model" db:"model"`

    // DeviceType classifies the hardware (server, switch, router, etc.).
    DeviceType DeviceType `json:"device_type" db:"device_type"`

    // PowerState is the last-known physical power state.
    // Updated by any adapter that can observe it (Redfish, IPMI, AMT).
    PowerState string `json:"power_state" db:"power_state"`

    // LastSeenAt records the last time any adapter successfully contacted this device.
    LastSeenAt time.Time `json:"last_seen_at" db:"last_seen_at"`

    CreatedAt time.Time `json:"created_at" db:"created_at"`
    UpdatedAt time.Time `json:"updated_at" db:"updated_at"`
}

// TenantDeviceView is a tenant-scoped projection of a PhysicalDevice.
// This is what tenants see, query, attach DesiredState to, and run workflows against.
// Each tenant gets exactly one TenantDeviceView per PhysicalDevice they have access to.
type TenantDeviceView struct {
    // ID is the tenant-scoped device identity. This is the DeviceID used in
    // all tenant-facing APIs, relationships, facts, desired state, and workflows.
    ID uuid.UUID `json:"id" db:"id"`

    // TenantID scopes this view to a tenant.
    TenantID uuid.UUID `json:"tenant_id" db:"tenant_id"`

    // PhysicalDeviceID links to the underlying hardware record.
    PhysicalDeviceID uuid.UUID `json:"physical_device_id" db:"physical_device_id"`

    // Name is the tenant's chosen name for this device. Mutable.
    Name string `json:"name" db:"name"`

    // Status is the tenant-scoped lifecycle status.
    // May differ from other tenants' views of the same physical device
    // (e.g., one tenant decommissions their view while another keeps it active).
    Status DeviceStatus `json:"status" db:"status"`

    // Role describes the device's function in this tenant's infrastructure.
    // Examples: "core-switch", "leaf-switch", "compute-node", "storage-array".
    Role string `json:"role,omitempty" db:"role"`

    // AccessLevel defines what this tenant can do with the physical device.
    // "owner" = full control (power, config, firmware).
    // "operator" = read state + execute pre-approved operations.
    // "observer" = read-only state visibility.
    AccessLevel DeviceAccessLevel `json:"access_level" db:"access_level"`

    // Metadata holds tenant-specific non-core attributes.
    Metadata map[string]string `json:"metadata,omitempty" db:"-"`

    CreatedAt time.Time `json:"created_at" db:"created_at"`
    UpdatedAt time.Time `json:"updated_at" db:"updated_at"`
}

// DeviceAccessLevel defines the RBAC level for a tenant's view of a shared device.
type DeviceAccessLevel string

const (
    // DeviceAccessOwner grants full control: power, configuration, firmware, decommission.
    DeviceAccessOwner DeviceAccessLevel = "owner"

    // DeviceAccessOperator grants read access to state and execution of
    // pre-approved operations (e.g., create VLANs within assigned range).
    DeviceAccessOperator DeviceAccessLevel = "operator"

    // DeviceAccessObserver grants read-only visibility of device state and facts.
    DeviceAccessObserver DeviceAccessLevel = "observer"
)
```

### SQL Schema

```sql
-- Physical device: hardware-level identity, not tenant-scoped.
-- No RLS on this table — accessed only by internal services, never by tenant APIs.
CREATE TABLE physical_devices (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    serial_number   TEXT NOT NULL,
    bios_uuid       TEXT,
    vendor          TEXT NOT NULL DEFAULT '',
    model           TEXT NOT NULL DEFAULT '',
    device_type     TEXT NOT NULL DEFAULT 'unknown'
                    CHECK (device_type IN ('server','switch','router','firewall',
                           'hypervisor','kvm','storage','appliance','unknown')),
    power_state     TEXT NOT NULL DEFAULT 'unknown',
    last_seen_at    TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- Serial number uniqueness prevents physical device duplication.
-- BIOS UUID is a secondary dedup key when serial is missing.
CREATE UNIQUE INDEX idx_physical_devices_serial
    ON physical_devices (serial_number)
    WHERE serial_number != '';

CREATE UNIQUE INDEX idx_physical_devices_bios_uuid
    ON physical_devices (bios_uuid)
    WHERE bios_uuid IS NOT NULL AND bios_uuid != '';

-- Tenant device view: tenant-scoped projection of a physical device.
CREATE TABLE tenant_device_views (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id           UUID NOT NULL,
    physical_device_id  UUID NOT NULL REFERENCES physical_devices(id),
    name                TEXT NOT NULL,
    status              TEXT NOT NULL DEFAULT 'discovered'
                        CHECK (status IN ('discovered','active','degraded',
                               'unreachable','decommissioned')),
    role                TEXT NOT NULL DEFAULT '',
    access_level        TEXT NOT NULL DEFAULT 'observer'
                        CHECK (access_level IN ('owner','operator','observer')),
    metadata            JSONB NOT NULL DEFAULT '{}',
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),

    -- One view per tenant per physical device.
    CONSTRAINT uq_tenant_physical_device
        UNIQUE (tenant_id, physical_device_id)
);

CREATE INDEX idx_tdv_tenant ON tenant_device_views (tenant_id);
CREATE INDEX idx_tdv_physical ON tenant_device_views (physical_device_id);
CREATE INDEX idx_tdv_status ON tenant_device_views (tenant_id, status);

-- Row-Level Security: tenants see only their own views.
ALTER TABLE tenant_device_views ENABLE ROW LEVEL SECURITY;

CREATE POLICY tenant_isolation ON tenant_device_views
    USING (tenant_id = current_setting('app.current_tenant_id')::UUID);
```

### Cross-Tenant Discovery Flow

```
1. Discovery adapter produces identity attributes (serial, MAC, BMC IP, etc.)
2. Identity matcher searches within the discovering tenant's TenantDeviceViews
3. If match found → enrich existing TenantDeviceView (normal path)
4. If no match found:
   a. Create PhysicalDevice (or find existing by serial_number)
   b. Create TenantDeviceView linked to that PhysicalDevice
   c. The new TenantDeviceView has access_level = 'owner'
5. Cross-tenant linking (admin-only, never automated):
   a. Admin queries physical_devices by serial_number
   b. Admin creates a new TenantDeviceView for tenant B,
      referencing the same PhysicalDevice
   c. Admin sets access_level = 'operator' or 'observer'
   d. Tenant B now sees the device in their inventory
```

### Event Propagation

```
When PhysicalDevice state changes (power_state, last_seen_at):
  1. UPDATE physical_devices SET power_state = $1 WHERE id = $2
  2. SELECT tdv.id, tdv.tenant_id
     FROM tenant_device_views tdv
     WHERE tdv.physical_device_id = $2
  3. For each TenantDeviceView:
     PUBLISH to loom.{tenant_id}.device.state_changed {
       device_id:  tdv.id,    -- tenant-scoped ID
       tenant_id:  tdv.tenant_id,
       property:   "power_state",
       old_value:  "on",
       new_value:  "off",
       source:     "physical_device_sync"
     }
```

### RBAC Rules

| Access Level | Read State | Read Facts | Run Discovery | Execute Operations | Modify Config | Power Control | Firmware Update | Decommission |
|-------------|-----------|-----------|--------------|-------------------|--------------|--------------|----------------|-------------|
| owner       | Yes | Yes | Yes | Yes | Yes | Yes | Yes | Yes |
| operator    | Yes | Yes | No  | Pre-approved only | No | No | No | No |
| observer    | Yes | Yes | No  | No | No | No | No | No |

### Migration Notes

The existing `Device` struct in DOMAIN-MODEL.md is replaced. All existing references to `Device.ID` become references to `TenantDeviceView.ID` — the tenant-facing DeviceID does not change. The `Device` table migrates to `tenant_device_views`, and hardware-identity columns move to `physical_devices`. Existing single-tenant devices get a 1:1 PhysicalDevice-to-TenantDeviceView mapping with `access_level = 'owner'`.

---

## 2. IPAM Model (C6)

### Decision

**Hybrid — built-in minimal IPAM with optional external sync.** LOOM ships a self-contained IP address allocator that works without external dependencies. An optional adapter syncs state with NetBox, Infoblox, or phpIPAM (Phase 5+). IPv6 is supported from day one via `netip.Prefix` (handles both v4 and v6).

### Resolved Open Questions

| Question | Decision |
|----------|----------|
| Built-in or external IPAM? | Built-in minimal, external sync optional. LOOM must work standalone in air-gapped environments. |
| IPv6 from day one? | Yes. Go's `netip.Prefix` and `netip.Addr` handle both v4 and v6. PostgreSQL `inet`/`cidr` types handle both. No dual-stack afterthought. |
| How does IPAM interact with DHCP? | LOOM creates DHCPScope records. External DHCP servers (ISC DHCP, Kea, Infoblox) are configured by adapters that read DHCPScope and push config. LOOM does not run a DHCP server. |
| Subnet allocation: LLM or deterministic? | Deterministic allocator. The LLM may suggest a prefix role and size; the allocator finds the next available prefix of that size within the parent. No LLM in the allocation hot path. |

### Go Types

```go
// PrefixStatus tracks the lifecycle of an IP prefix.
type PrefixStatus string

const (
    PrefixStatusActive     PrefixStatus = "active"
    PrefixStatusReserved   PrefixStatus = "reserved"
    PrefixStatusDeprecated PrefixStatus = "deprecated"
    PrefixStatusContainer  PrefixStatus = "container" // parent-only, not directly assignable
)

// AddressStatus tracks the lifecycle of an individual IP address.
type AddressStatus string

const (
    AddressStatusActive    AddressStatus = "active"
    AddressStatusReserved  AddressStatus = "reserved"
    AddressStatusDHCP      AddressStatus = "dhcp"
    AddressStatusSLAAC     AddressStatus = "slaac"
    AddressStatusAvailable AddressStatus = "available"
    AddressStatusDeprecated AddressStatus = "deprecated"
)

// IPPrefix represents a network prefix (subnet) in the IPAM hierarchy.
// Prefixes form a tree: a /16 contains /24s, which contain /28s, etc.
type IPPrefix struct {
    // ID is the unique identifier for this prefix.
    ID uuid.UUID `json:"id" db:"id"`

    // TenantID scopes this prefix to a tenant.
    TenantID uuid.UUID `json:"tenant_id" db:"tenant_id"`

    // Prefix is the CIDR notation (e.g., "10.1.0.0/24", "2001:db8::/48").
    // Stored as Go's netip.Prefix; mapped to PostgreSQL cidr.
    Prefix netip.Prefix `json:"prefix" db:"prefix"`

    // ParentID links to the parent prefix in the hierarchy. Nil for top-level.
    ParentID *uuid.UUID `json:"parent_id,omitempty" db:"parent_id"`

    // Status is the lifecycle state of this prefix.
    Status PrefixStatus `json:"status" db:"status"`

    // Role describes the function of this prefix.
    // Examples: "management", "production", "storage", "oob", "loopback", "p2p".
    Role string `json:"role" db:"role"`

    // VLANID optionally associates this prefix with a VLAN.
    VLANID *int `json:"vlan_id,omitempty" db:"vlan_id"`

    // SiteID optionally scopes this prefix to a physical site.
    SiteID *uuid.UUID `json:"site_id,omitempty" db:"site_id"`

    // Description is a human-readable description.
    Description string `json:"description,omitempty" db:"description"`

    // IsPool marks this prefix as a pool from which individual addresses
    // can be allocated (as opposed to a container that only holds child prefixes).
    IsPool bool `json:"is_pool" db:"is_pool"`

    // Utilization is the cached percentage of addresses in use (0.0–1.0).
    // Updated on allocation/release. Not authoritative — recalculate for accuracy.
    Utilization float64 `json:"utilization" db:"utilization"`

    CreatedAt time.Time `json:"created_at" db:"created_at"`
    UpdatedAt time.Time `json:"updated_at" db:"updated_at"`
}

// IPAddress represents a single IP address within a prefix.
type IPAddress struct {
    // ID is the unique identifier for this address record.
    ID uuid.UUID `json:"id" db:"id"`

    // TenantID scopes this address to a tenant.
    TenantID uuid.UUID `json:"tenant_id" db:"tenant_id"`

    // Address is the IP address (e.g., "10.1.0.5", "2001:db8::5").
    // Stored as Go's netip.Addr; mapped to PostgreSQL inet.
    Address netip.Addr `json:"address" db:"address"`

    // PrefixID links to the parent prefix.
    PrefixID uuid.UUID `json:"prefix_id" db:"prefix_id"`

    // Status is the allocation state of this address.
    Status AddressStatus `json:"status" db:"status"`

    // DeviceID is the device this address is assigned to (nullable).
    DeviceID *uuid.UUID `json:"device_id,omitempty" db:"device_id"`

    // InterfaceID is the specific interface this address is bound to (nullable).
    InterfaceID *uuid.UUID `json:"interface_id,omitempty" db:"interface_id"`

    // DNSName is the forward DNS name for this address.
    DNSName string `json:"dns_name,omitempty" db:"dns_name"`

    // Description is a human-readable note.
    Description string `json:"description,omitempty" db:"description"`

    CreatedAt time.Time `json:"created_at" db:"created_at"`
    UpdatedAt time.Time `json:"updated_at" db:"updated_at"`
}

// DHCPScope defines a DHCP range within a prefix.
// LOOM does not serve DHCP — this record is consumed by DHCP server adapters
// (ISC Kea, Infoblox, etc.) to generate their configuration.
type DHCPScope struct {
    // ID is the unique identifier for this scope.
    ID uuid.UUID `json:"id" db:"id"`

    // TenantID scopes this DHCP scope to a tenant.
    TenantID uuid.UUID `json:"tenant_id" db:"tenant_id"`

    // PrefixID links to the parent prefix.
    PrefixID uuid.UUID `json:"prefix_id" db:"prefix_id"`

    // RangeStart is the first address in the DHCP range (inclusive).
    RangeStart netip.Addr `json:"range_start" db:"range_start"`

    // RangeEnd is the last address in the DHCP range (inclusive).
    RangeEnd netip.Addr `json:"range_end" db:"range_end"`

    // LeaseTimeSec is the default lease duration in seconds.
    LeaseTimeSec int `json:"lease_time_sec" db:"lease_time_sec"`

    // Options holds DHCP options as key-value pairs.
    // Standard keys: "router", "dns_servers", "ntp_servers", "domain_name".
    Options map[string]string `json:"options" db:"-"`

    // Enabled controls whether this scope is active.
    Enabled bool `json:"enabled" db:"enabled"`

    CreatedAt time.Time `json:"created_at" db:"created_at"`
    UpdatedAt time.Time `json:"updated_at" db:"updated_at"`
}

// IPReservation is a DHCP reservation (static mapping of MAC to IP).
type IPReservation struct {
    // ID is the unique identifier for this reservation.
    ID uuid.UUID `json:"id" db:"id"`

    // TenantID scopes this reservation to a tenant.
    TenantID uuid.UUID `json:"tenant_id" db:"tenant_id"`

    // ScopeID links to the parent DHCP scope.
    ScopeID uuid.UUID `json:"scope_id" db:"scope_id"`

    // MACAddress is the client's MAC address.
    MACAddress string `json:"mac_address" db:"mac_address"`

    // IPAddressID links to the reserved IP address.
    IPAddressID uuid.UUID `json:"ip_address_id" db:"ip_address_id"`

    // Hostname is the hostname to assign via DHCP option 12.
    Hostname string `json:"hostname,omitempty" db:"hostname"`

    // Description is a human-readable note.
    Description string `json:"description,omitempty" db:"description"`

    CreatedAt time.Time `json:"created_at" db:"created_at"`
    UpdatedAt time.Time `json:"updated_at" db:"updated_at"`
}
```

### SQL Schema

```sql
CREATE TABLE ip_prefixes (
    id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id   UUID NOT NULL,
    prefix      CIDR NOT NULL,
    parent_id   UUID REFERENCES ip_prefixes(id),
    status      TEXT NOT NULL DEFAULT 'active'
                CHECK (status IN ('active','reserved','deprecated','container')),
    role        TEXT NOT NULL DEFAULT '',
    vlan_id     INTEGER,
    site_id     UUID,
    description TEXT NOT NULL DEFAULT '',
    is_pool     BOOLEAN NOT NULL DEFAULT FALSE,
    utilization REAL NOT NULL DEFAULT 0.0,
    created_at  TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at  TIMESTAMPTZ NOT NULL DEFAULT NOW(),

    -- A prefix must be unique within a tenant.
    CONSTRAINT uq_tenant_prefix UNIQUE (tenant_id, prefix)
);

-- GiST index for CIDR containment queries (e.g., "find all prefixes within 10.0.0.0/8").
CREATE INDEX idx_ip_prefixes_cidr ON ip_prefixes USING gist (prefix inet_ops);
CREATE INDEX idx_ip_prefixes_tenant ON ip_prefixes (tenant_id);
CREATE INDEX idx_ip_prefixes_parent ON ip_prefixes (parent_id);
CREATE INDEX idx_ip_prefixes_site ON ip_prefixes (site_id) WHERE site_id IS NOT NULL;

-- Exclusion constraint: prevent overlapping prefixes within the same tenant and role.
-- Two prefixes in the same tenant with the same role must not overlap.
-- Uses the && operator on CIDR (inet range overlap).
CREATE EXTENSION IF NOT EXISTS btree_gist;

ALTER TABLE ip_prefixes ADD CONSTRAINT excl_prefix_overlap
    EXCLUDE USING gist (
        tenant_id WITH =,
        role WITH =,
        prefix inet_ops WITH &&
    ) WHERE (status != 'deprecated');

-- IP addresses
CREATE TABLE ip_addresses (
    id           UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id    UUID NOT NULL,
    address      INET NOT NULL,
    prefix_id    UUID NOT NULL REFERENCES ip_prefixes(id),
    status       TEXT NOT NULL DEFAULT 'available'
                 CHECK (status IN ('active','reserved','dhcp','slaac',
                        'available','deprecated')),
    device_id    UUID,
    interface_id UUID,
    dns_name     TEXT NOT NULL DEFAULT '',
    description  TEXT NOT NULL DEFAULT '',
    created_at   TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at   TIMESTAMPTZ NOT NULL DEFAULT NOW(),

    -- An address must be unique within a tenant.
    CONSTRAINT uq_tenant_address UNIQUE (tenant_id, address)
);

CREATE INDEX idx_ip_addresses_prefix ON ip_addresses (prefix_id);
CREATE INDEX idx_ip_addresses_device ON ip_addresses (device_id)
    WHERE device_id IS NOT NULL;
CREATE INDEX idx_ip_addresses_status ON ip_addresses (prefix_id, status);
CREATE INDEX idx_ip_addresses_available
    ON ip_addresses (prefix_id, address)
    WHERE status = 'available';

-- DHCP scopes
CREATE TABLE dhcp_scopes (
    id             UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id      UUID NOT NULL,
    prefix_id      UUID NOT NULL REFERENCES ip_prefixes(id),
    range_start    INET NOT NULL,
    range_end      INET NOT NULL,
    lease_time_sec INTEGER NOT NULL DEFAULT 3600,
    options        JSONB NOT NULL DEFAULT '{}',
    enabled        BOOLEAN NOT NULL DEFAULT TRUE,
    created_at     TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at     TIMESTAMPTZ NOT NULL DEFAULT NOW(),

    -- Range end must be >= range start.
    CONSTRAINT chk_range_order CHECK (range_end >= range_start)
);

CREATE INDEX idx_dhcp_scopes_prefix ON dhcp_scopes (prefix_id);
CREATE INDEX idx_dhcp_scopes_tenant ON dhcp_scopes (tenant_id);

-- IP reservations (static DHCP)
CREATE TABLE ip_reservations (
    id            UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id     UUID NOT NULL,
    scope_id      UUID NOT NULL REFERENCES dhcp_scopes(id),
    mac_address   MACADDR NOT NULL,
    ip_address_id UUID NOT NULL REFERENCES ip_addresses(id),
    hostname      TEXT NOT NULL DEFAULT '',
    description   TEXT NOT NULL DEFAULT '',
    created_at    TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at    TIMESTAMPTZ NOT NULL DEFAULT NOW(),

    -- One reservation per MAC per scope.
    CONSTRAINT uq_scope_mac UNIQUE (scope_id, mac_address)
);

CREATE INDEX idx_ip_reservations_scope ON ip_reservations (scope_id);

-- RLS on all IPAM tables.
ALTER TABLE ip_prefixes ENABLE ROW LEVEL SECURITY;
ALTER TABLE ip_addresses ENABLE ROW LEVEL SECURITY;
ALTER TABLE dhcp_scopes ENABLE ROW LEVEL SECURITY;
ALTER TABLE ip_reservations ENABLE ROW LEVEL SECURITY;

CREATE POLICY tenant_isolation ON ip_prefixes
    USING (tenant_id = current_setting('app.current_tenant_id')::UUID);
CREATE POLICY tenant_isolation ON ip_addresses
    USING (tenant_id = current_setting('app.current_tenant_id')::UUID);
CREATE POLICY tenant_isolation ON dhcp_scopes
    USING (tenant_id = current_setting('app.current_tenant_id')::UUID);
CREATE POLICY tenant_isolation ON ip_reservations
    USING (tenant_id = current_setting('app.current_tenant_id')::UUID);
```

### Atomic Allocation

```sql
-- Allocate the next available IP address from a prefix.
-- Uses SELECT FOR UPDATE SKIP LOCKED to handle concurrent allocations
-- without serialization failures.
--
-- This is the hot path. No LLM involvement. Deterministic.
BEGIN;

SELECT id, address
FROM ip_addresses
WHERE prefix_id = $1
  AND status = 'available'
  AND tenant_id = $2
ORDER BY address ASC
LIMIT 1
FOR UPDATE SKIP LOCKED;

-- If a row was returned:
UPDATE ip_addresses
SET status = 'active',
    device_id = $3,
    interface_id = $4,
    dns_name = $5,
    updated_at = NOW()
WHERE id = $6;

-- Update prefix utilization cache.
UPDATE ip_prefixes
SET utilization = (
    SELECT COUNT(*)::REAL / NULLIF(
        (SELECT COUNT(*) FROM ip_addresses WHERE prefix_id = $1), 0
    )
    FROM ip_addresses
    WHERE prefix_id = $1 AND status IN ('active','reserved','dhcp')
),
    updated_at = NOW()
WHERE id = $1;

COMMIT;
```

### NetBox Sync Adapter (Phase 5+)

The optional NetBox sync adapter is a bidirectional reconciler:

- **LOOM -> NetBox:** On IPPrefix/IPAddress create/update/delete, push changes to NetBox via REST API.
- **NetBox -> LOOM:** Periodic poll (every 5 minutes) fetches NetBox prefixes/IPs and reconciles with LOOM's state. Conflicts are resolved by a configurable policy: `loom_authoritative` (default), `netbox_authoritative`, or `manual_review`.

This adapter is out of scope until Phase 5. The IPAM schema is designed to work standalone until then.

### Migration Notes

These are new tables. No existing DOMAIN-MODEL.md types are modified. The `IPPrefix.SiteID` foreign key references the `sites` table defined in Section 3 below. The `IPAddress.DeviceID` references `tenant_device_views.id` (not `physical_devices.id`).

---

## 3. Physical Topology Model (C8)

### Decision

**Optional physical topology.** LOOM works without it. When topology data is present, the placement engine uses it for failure domain awareness. When absent, LOOM falls back to device tags/metadata or treats all devices as being in a single failure domain.

### Resolved Open Questions

| Question | Decision |
|----------|----------|
| Required or optional for placement? | Optional. When topology is absent, the placement engine uses device metadata tags (`failure_domain`, `availability_zone`) if present, otherwise assumes a single domain. |
| What does placement use when topology is missing? | Cloud deployments map to Site (region) + Rack (AZ). On-prem without topology data: all devices in one implicit site. Operators can add topology incrementally. |
| Auto-discover cables via LLDP/CDP? | Yes, Phase 3. LLDP/CDP discovery populates CableConnection records. Manual entry available from Phase 2 via API. |

### Go Types

```go
// Site represents a physical location (data center, colo, cloud region).
type Site struct {
    // ID is the unique identifier for this site.
    ID uuid.UUID `json:"id" db:"id"`

    // TenantID scopes this site to a tenant.
    TenantID uuid.UUID `json:"tenant_id" db:"tenant_id"`

    // Name is a human-readable label (e.g., "us-east-1", "london-dc2", "ams-equinix-am5").
    Name string `json:"name" db:"name"`

    // Slug is a URL-safe short name, unique within tenant.
    Slug string `json:"slug" db:"slug"`

    // Region groups sites for failure domain calculations.
    // Cloud: AWS region. On-prem: metro area or geographic region.
    Region string `json:"region,omitempty" db:"region"`

    // Facility is the data center facility name (e.g., "Equinix AM5", "CyrusOne DFW3").
    Facility string `json:"facility,omitempty" db:"facility"`

    // Address is the physical street address.
    Address string `json:"address,omitempty" db:"address"`

    // Latitude for geographic-aware placement (nullable).
    Latitude *float64 `json:"latitude,omitempty" db:"latitude"`

    // Longitude for geographic-aware placement (nullable).
    Longitude *float64 `json:"longitude,omitempty" db:"longitude"`

    // Status tracks the operational state of the site.
    Status string `json:"status" db:"status"` // "active", "planned", "decommissioned"

    CreatedAt time.Time `json:"created_at" db:"created_at"`
    UpdatedAt time.Time `json:"updated_at" db:"updated_at"`
}

// Rack represents a physical equipment rack within a site.
type Rack struct {
    // ID is the unique identifier for this rack.
    ID uuid.UUID `json:"id" db:"id"`

    // SiteID links this rack to a site.
    SiteID uuid.UUID `json:"site_id" db:"site_id"`

    // TenantID scopes this rack to a tenant.
    TenantID uuid.UUID `json:"tenant_id" db:"tenant_id"`

    // Name is the rack label (e.g., "A3", "Row-B-Rack-14").
    Name string `json:"name" db:"name"`

    // TotalUnits is the total rack units (e.g., 42, 48).
    TotalUnits int `json:"total_units" db:"total_units"`

    // CoolingZone groups racks that share cooling infrastructure.
    CoolingZone string `json:"cooling_zone,omitempty" db:"cooling_zone"`

    // Weight capacity in kg (nullable).
    MaxWeightKg *float64 `json:"max_weight_kg,omitempty" db:"max_weight_kg"`

    // Power capacity in watts (nullable).
    MaxPowerW *int `json:"max_power_w,omitempty" db:"max_power_w"`

    CreatedAt time.Time `json:"created_at" db:"created_at"`
    UpdatedAt time.Time `json:"updated_at" db:"updated_at"`
}

// RackMount records where a device is physically installed in a rack.
type RackMount struct {
    // ID is the unique identifier for this mount record.
    ID uuid.UUID `json:"id" db:"id"`

    // RackID links to the rack.
    RackID uuid.UUID `json:"rack_id" db:"rack_id"`

    // DeviceID links to the device (PhysicalDevice, not TenantDeviceView).
    DeviceID uuid.UUID `json:"device_id" db:"device_id"`

    // StartUnit is the lowest rack unit occupied (1-indexed from bottom).
    StartUnit int `json:"start_unit" db:"start_unit"`

    // Height is the number of rack units consumed (e.g., 1, 2, 4).
    Height int `json:"height" db:"height"`

    // Position is the face of the rack (front or rear).
    Position string `json:"position" db:"position"` // "front", "rear"

    CreatedAt time.Time `json:"created_at" db:"created_at"`
    UpdatedAt time.Time `json:"updated_at" db:"updated_at"`
}

// PowerCircuit represents a power feed to a rack.
type PowerCircuit struct {
    // ID is the unique identifier for this circuit.
    ID uuid.UUID `json:"id" db:"id"`

    // SiteID links this circuit to a site.
    SiteID uuid.UUID `json:"site_id" db:"site_id"`

    // TenantID scopes this circuit to a tenant.
    TenantID uuid.UUID `json:"tenant_id" db:"tenant_id"`

    // Name is the circuit label (e.g., "PDU-A3-L1", "UPS-B-Feed-2").
    Name string `json:"name" db:"name"`

    // Feed identifies the redundancy group ("A", "B" for dual-feed racks).
    Feed string `json:"feed" db:"feed"`

    // MaxAmps is the circuit breaker rating.
    MaxAmps float64 `json:"max_amps" db:"max_amps"`

    // Voltage is the nominal voltage (120, 208, 240, etc.).
    Voltage int `json:"voltage" db:"voltage"`

    // Phase is the power phase count (1 or 3).
    Phase int `json:"phase" db:"phase"`

    CreatedAt time.Time `json:"created_at" db:"created_at"`
    UpdatedAt time.Time `json:"updated_at" db:"updated_at"`
}

// RackPowerFeed links a rack to the power circuits that feed it.
type RackPowerFeed struct {
    ID        uuid.UUID `json:"id" db:"id"`
    RackID    uuid.UUID `json:"rack_id" db:"rack_id"`
    CircuitID uuid.UUID `json:"circuit_id" db:"circuit_id"`
    CreatedAt time.Time `json:"created_at" db:"created_at"`
}

// CoolingZone represents a cooling domain within a site.
type CoolingZone struct {
    // ID is the unique identifier for this cooling zone.
    ID uuid.UUID `json:"id" db:"id"`

    // SiteID links this zone to a site.
    SiteID uuid.UUID `json:"site_id" db:"site_id"`

    // TenantID scopes this zone to a tenant.
    TenantID uuid.UUID `json:"tenant_id" db:"tenant_id"`

    // Name is the zone label (e.g., "hot-aisle-A", "CRAH-zone-2").
    Name string `json:"name" db:"name"`

    // CoolingType classifies the cooling method.
    CoolingType string `json:"cooling_type" db:"cooling_type"` // "air", "liquid", "immersion"

    // CapacityKW is the cooling capacity in kilowatts (nullable).
    CapacityKW *float64 `json:"capacity_kw,omitempty" db:"capacity_kw"`

    CreatedAt time.Time `json:"created_at" db:"created_at"`
    UpdatedAt time.Time `json:"updated_at" db:"updated_at"`
}

// CableConnection represents a physical cable between two device ports.
// Auto-populated from LLDP/CDP discovery (Phase 3) or manual entry (Phase 2).
type CableConnection struct {
    // ID is the unique identifier for this cable record.
    ID uuid.UUID `json:"id" db:"id"`

    // TenantID scopes this cable to a tenant.
    TenantID uuid.UUID `json:"tenant_id" db:"tenant_id"`

    // SourceDeviceID is the device on one end of the cable.
    SourceDeviceID uuid.UUID `json:"source_device_id" db:"source_device_id"`

    // SourcePort is the port name on the source device (e.g., "Ethernet1/1", "Gi1/0/1").
    SourcePort string `json:"source_port" db:"source_port"`

    // DestDeviceID is the device on the other end of the cable.
    DestDeviceID uuid.UUID `json:"dest_device_id" db:"dest_device_id"`

    // DestPort is the port name on the destination device.
    DestPort string `json:"dest_port" db:"dest_port"`

    // MediaType classifies the cable medium.
    MediaType string `json:"media_type" db:"media_type"`
    // Known values: "fiber-sm", "fiber-mm", "cat5e", "cat6", "cat6a", "dac", "aoc"

    // CableID is the physical cable label (printed on the cable or in the cable tray).
    CableID string `json:"cable_id,omitempty" db:"cable_id"`

    // Speed is the negotiated link speed (e.g., "10G", "25G", "100G", "1G").
    Speed string `json:"speed,omitempty" db:"speed"`

    // DiscoverySource indicates how this connection was discovered.
    // "lldp", "cdp", "manual"
    DiscoverySource string `json:"discovery_source" db:"discovery_source"`

    // LastVerifiedAt is the last time this connection was confirmed by discovery.
    LastVerifiedAt time.Time `json:"last_verified_at" db:"last_verified_at"`

    CreatedAt time.Time `json:"created_at" db:"created_at"`
    UpdatedAt time.Time `json:"updated_at" db:"updated_at"`
}
```

### SQL Schema

```sql
CREATE TABLE sites (
    id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id   UUID NOT NULL,
    name        TEXT NOT NULL,
    slug        TEXT NOT NULL,
    region      TEXT NOT NULL DEFAULT '',
    facility    TEXT NOT NULL DEFAULT '',
    address     TEXT NOT NULL DEFAULT '',
    latitude    DOUBLE PRECISION,
    longitude   DOUBLE PRECISION,
    status      TEXT NOT NULL DEFAULT 'active'
                CHECK (status IN ('active','planned','decommissioned')),
    created_at  TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at  TIMESTAMPTZ NOT NULL DEFAULT NOW(),

    CONSTRAINT uq_tenant_site_slug UNIQUE (tenant_id, slug)
);

CREATE INDEX idx_sites_tenant ON sites (tenant_id);
CREATE INDEX idx_sites_region ON sites (tenant_id, region);

CREATE TABLE racks (
    id           UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    site_id      UUID NOT NULL REFERENCES sites(id) ON DELETE CASCADE,
    tenant_id    UUID NOT NULL,
    name         TEXT NOT NULL,
    total_units  INTEGER NOT NULL DEFAULT 42
                 CHECK (total_units > 0 AND total_units <= 60),
    cooling_zone TEXT NOT NULL DEFAULT '',
    max_weight_kg DOUBLE PRECISION,
    max_power_w  INTEGER,
    created_at   TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at   TIMESTAMPTZ NOT NULL DEFAULT NOW(),

    CONSTRAINT uq_site_rack_name UNIQUE (site_id, name)
);

CREATE INDEX idx_racks_site ON racks (site_id);
CREATE INDEX idx_racks_tenant ON racks (tenant_id);

CREATE TABLE rack_mounts (
    id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    rack_id     UUID NOT NULL REFERENCES racks(id) ON DELETE CASCADE,
    device_id   UUID NOT NULL REFERENCES physical_devices(id),
    start_unit  INTEGER NOT NULL CHECK (start_unit >= 1),
    height      INTEGER NOT NULL DEFAULT 1 CHECK (height >= 1),
    position    TEXT NOT NULL DEFAULT 'front'
                CHECK (position IN ('front','rear')),
    created_at  TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at  TIMESTAMPTZ NOT NULL DEFAULT NOW(),

    -- A device can be mounted in only one rack at a time.
    CONSTRAINT uq_device_mount UNIQUE (device_id),

    -- Rack units must not overlap within the same rack and position.
    -- Enforced via exclusion constraint on the integer range [start_unit, start_unit+height).
    CONSTRAINT chk_unit_range CHECK (start_unit + height - 1 <= 60)
);

-- Exclusion constraint: no two mounts in the same rack+position can overlap unit ranges.
ALTER TABLE rack_mounts ADD CONSTRAINT excl_rack_units
    EXCLUDE USING gist (
        rack_id WITH =,
        position WITH =,
        int4range(start_unit, start_unit + height) WITH &&
    );

CREATE INDEX idx_rack_mounts_rack ON rack_mounts (rack_id);
CREATE INDEX idx_rack_mounts_device ON rack_mounts (device_id);

CREATE TABLE power_circuits (
    id         UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    site_id    UUID NOT NULL REFERENCES sites(id) ON DELETE CASCADE,
    tenant_id  UUID NOT NULL,
    name       TEXT NOT NULL,
    feed       TEXT NOT NULL DEFAULT 'A' CHECK (feed IN ('A','B','C')),
    max_amps   REAL NOT NULL CHECK (max_amps > 0),
    voltage    INTEGER NOT NULL CHECK (voltage > 0),
    phase      INTEGER NOT NULL DEFAULT 1 CHECK (phase IN (1, 3)),
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),

    CONSTRAINT uq_site_circuit_name UNIQUE (site_id, name)
);

CREATE INDEX idx_power_circuits_site ON power_circuits (site_id);

CREATE TABLE rack_power_feeds (
    id         UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    rack_id    UUID NOT NULL REFERENCES racks(id) ON DELETE CASCADE,
    circuit_id UUID NOT NULL REFERENCES power_circuits(id) ON DELETE CASCADE,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),

    CONSTRAINT uq_rack_circuit UNIQUE (rack_id, circuit_id)
);

CREATE TABLE cooling_zones (
    id           UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    site_id      UUID NOT NULL REFERENCES sites(id) ON DELETE CASCADE,
    tenant_id    UUID NOT NULL,
    name         TEXT NOT NULL,
    cooling_type TEXT NOT NULL DEFAULT 'air'
                 CHECK (cooling_type IN ('air','liquid','immersion')),
    capacity_kw  DOUBLE PRECISION,
    created_at   TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at   TIMESTAMPTZ NOT NULL DEFAULT NOW(),

    CONSTRAINT uq_site_cooling_name UNIQUE (site_id, name)
);

CREATE TABLE cable_connections (
    id               UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id        UUID NOT NULL,
    source_device_id UUID NOT NULL REFERENCES physical_devices(id),
    source_port      TEXT NOT NULL,
    dest_device_id   UUID NOT NULL REFERENCES physical_devices(id),
    dest_port        TEXT NOT NULL,
    media_type       TEXT NOT NULL DEFAULT 'cat6'
                     CHECK (media_type IN ('fiber-sm','fiber-mm','cat5e',
                            'cat6','cat6a','dac','aoc')),
    cable_id         TEXT NOT NULL DEFAULT '',
    speed            TEXT NOT NULL DEFAULT '',
    discovery_source TEXT NOT NULL DEFAULT 'manual'
                     CHECK (discovery_source IN ('lldp','cdp','manual')),
    last_verified_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    created_at       TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at       TIMESTAMPTZ NOT NULL DEFAULT NOW(),

    -- A port can have only one cable connection.
    CONSTRAINT uq_source_port UNIQUE (source_device_id, source_port),
    CONSTRAINT uq_dest_port UNIQUE (dest_device_id, dest_port),

    -- No self-loops.
    CONSTRAINT chk_no_self_loop CHECK (source_device_id != dest_device_id)
);

CREATE INDEX idx_cables_source ON cable_connections (source_device_id);
CREATE INDEX idx_cables_dest ON cable_connections (dest_device_id);
CREATE INDEX idx_cables_tenant ON cable_connections (tenant_id);

-- RLS on tenant-scoped topology tables.
ALTER TABLE sites ENABLE ROW LEVEL SECURITY;
ALTER TABLE racks ENABLE ROW LEVEL SECURITY;
ALTER TABLE power_circuits ENABLE ROW LEVEL SECURITY;
ALTER TABLE cooling_zones ENABLE ROW LEVEL SECURITY;
ALTER TABLE cable_connections ENABLE ROW LEVEL SECURITY;

CREATE POLICY tenant_isolation ON sites
    USING (tenant_id = current_setting('app.current_tenant_id')::UUID);
CREATE POLICY tenant_isolation ON racks
    USING (tenant_id = current_setting('app.current_tenant_id')::UUID);
CREATE POLICY tenant_isolation ON power_circuits
    USING (tenant_id = current_setting('app.current_tenant_id')::UUID);
CREATE POLICY tenant_isolation ON cooling_zones
    USING (tenant_id = current_setting('app.current_tenant_id')::UUID);
CREATE POLICY tenant_isolation ON cable_connections
    USING (tenant_id = current_setting('app.current_tenant_id')::UUID);
```

### Failure Domain Queries

```sql
-- Q1: Which servers share a power circuit with device X?
-- Used by placement engine to avoid co-locating replicas on the same power feed.
SELECT DISTINCT rm2.device_id AS co_powered_device
FROM rack_mounts rm1
JOIN rack_power_feeds rpf1 ON rpf1.rack_id = rm1.rack_id
JOIN rack_power_feeds rpf2 ON rpf2.circuit_id = rpf1.circuit_id
JOIN rack_mounts rm2 ON rm2.rack_id = rpf2.rack_id
WHERE rm1.device_id = $1      -- device X
  AND rm2.device_id != $1;

-- Q2: What is the blast radius of a TOR switch failure?
-- All devices with a cable connection to switch S.
SELECT cc.source_device_id AS affected_device, cc.source_port
FROM cable_connections cc
WHERE cc.dest_device_id = $1   -- switch S
UNION
SELECT cc.dest_device_id AS affected_device, cc.dest_port
FROM cable_connections cc
WHERE cc.source_device_id = $1;

-- Q3: Place N replicas in distinct failure domains.
-- Returns one device per rack (no two replicas share a rack).
SELECT DISTINCT ON (rm.rack_id)
    rm.device_id, rm.rack_id, r.name AS rack_name, s.name AS site_name
FROM rack_mounts rm
JOIN racks r ON r.id = rm.rack_id
JOIN sites s ON s.id = r.site_id
JOIN physical_devices pd ON pd.id = rm.device_id
WHERE pd.device_type = 'server'
  AND pd.power_state = 'on'
ORDER BY rm.rack_id, random()
LIMIT $1;  -- N replicas

-- Q4: Devices sharing a cooling zone (for thermal-aware placement).
SELECT rm.device_id
FROM rack_mounts rm
JOIN racks r ON r.id = rm.rack_id
WHERE r.cooling_zone = (
    SELECT r2.cooling_zone FROM racks r2
    JOIN rack_mounts rm2 ON rm2.rack_id = r2.id
    WHERE rm2.device_id = $1
);
```

### Cloud Deployment Mapping

For cloud environments without physical racks:

| Physical Concept | Cloud Mapping |
|-----------------|---------------|
| Site | Cloud region (us-east-1, eu-west-2) |
| Rack | Availability zone (us-east-1a) |
| RackMount | Not applicable (no physical units) |
| PowerCircuit | Not applicable |
| CableConnection | Not applicable (virtual networking) |
| CoolingZone | Not applicable |

Cloud tenants create Site records for regions and Rack records for AZs. The placement engine treats these as failure domains identically to physical racks.

### Migration Notes

All new tables. No existing types modified. `rack_mounts.device_id` references `physical_devices.id` (Section 1). `cable_connections` reference `physical_devices.id` because cables connect physical hardware, not tenant views.

---

## 4. Firmware Lifecycle Model (C9)

### Decision

**Built-in firmware inventory + compliance policies. Update execution via Temporal workflow.** LOOM tracks firmware versions as discoverable facts, maintains compliance policies, and orchestrates updates through Temporal workflows with vendor-specific adapter activities. Air-gapped environments use a local artifact mirror.

### Resolved Open Questions

| Question | Decision |
|----------|----------|
| Built-in compliance scanning or external? | Built-in. Compliance scanning is a periodic Temporal workflow that compares FirmwareInventory against FirmwarePolicy. No external dependency. |
| Where are firmware packages stored? | A configurable artifact store (S3-compatible, HTTP server, or local filesystem). LOOM stores the URL/path, not the binary. Air-gapped: local HTTP mirror on the control plane or a LAN server. |
| Firmware compatibility matrices? | Manual entry via API. Vendor API integration (Dell DSET, HPE SUM) is Phase 5+. No community database dependency. |
| Temporal workflows for firmware updates? | Yes. Each firmware update is a Temporal workflow with saga compensation: pre-check -> stage -> apply -> reboot (if needed) -> post-check. Failure at any step triggers rollback to the pre-update version. |
| Air-gapped firmware update? | Local artifact mirror. The FirmwarePackage.PackageURL points to an internal HTTP server. The administrator uploads firmware binaries to the mirror out-of-band. |

### Go Types

```go
// ComplianceStatus tracks whether a firmware component meets policy requirements.
type ComplianceStatus string

const (
    ComplianceCompliant      ComplianceStatus = "compliant"
    ComplianceUpdateAvail    ComplianceStatus = "update_available"
    ComplianceCriticalCVE    ComplianceStatus = "critical_cve"
    ComplianceEOL            ComplianceStatus = "end_of_life"
    ComplianceUnknown        ComplianceStatus = "unknown"
)

// FirmwareComponentType classifies which part of the device the firmware applies to.
type FirmwareComponentType string

const (
    FirmwareComponentBIOS    FirmwareComponentType = "bios"
    FirmwareComponentBMC     FirmwareComponentType = "bmc"
    FirmwareComponentNIC     FirmwareComponentType = "nic"
    FirmwareComponentDisk    FirmwareComponentType = "disk"
    FirmwareComponentGPU     FirmwareComponentType = "gpu"
    FirmwareComponentCPLD    FirmwareComponentType = "cpld"
    FirmwareComponentPSU     FirmwareComponentType = "psu"
    FirmwareComponentOS      FirmwareComponentType = "os"
    FirmwareComponentSwitch  FirmwareComponentType = "switch_os" // NX-OS, EOS, JunOS, etc.
)

// FirmwareUpdateStatus tracks the lifecycle of a firmware update operation.
type FirmwareUpdateStatus string

const (
    FirmwareUpdatePending    FirmwareUpdateStatus = "pending"
    FirmwareUpdateStaging    FirmwareUpdateStatus = "staging"
    FirmwareUpdateApplying   FirmwareUpdateStatus = "applying"
    FirmwareUpdateRebooting  FirmwareUpdateStatus = "rebooting"
    FirmwareUpdateVerifying  FirmwareUpdateStatus = "verifying"
    FirmwareUpdateCompleted  FirmwareUpdateStatus = "completed"
    FirmwareUpdateFailed     FirmwareUpdateStatus = "failed"
    FirmwareUpdateRolledBack FirmwareUpdateStatus = "rolled_back"
)

// FirmwareInventory records the current firmware version of a specific component
// on a specific device. Populated by discovery adapters (Redfish, SNMP, SSH).
type FirmwareInventory struct {
    // ID is the unique identifier for this inventory record.
    ID uuid.UUID `json:"id" db:"id"`

    // DeviceID links to the physical device.
    DeviceID uuid.UUID `json:"device_id" db:"device_id"`

    // TenantID scopes this record to the tenant that discovered it.
    TenantID uuid.UUID `json:"tenant_id" db:"tenant_id"`

    // Component identifies which firmware component this is.
    Component FirmwareComponentType `json:"component" db:"component"`

    // ComponentName is the vendor-specific component name (e.g., "iDRAC", "iLO 5",
    // "Broadcom BCM57416", "Intel S2600WF BIOS").
    ComponentName string `json:"component_name" db:"component_name"`

    // CurrentVersion is the currently installed firmware version string.
    CurrentVersion string `json:"current_version" db:"current_version"`

    // AvailableVersion is the latest known version from policy/repository (nullable).
    AvailableVersion *string `json:"available_version,omitempty" db:"available_version"`

    // Compliance is the current compliance status against applicable policies.
    Compliance ComplianceStatus `json:"compliance" db:"compliance"`

    // LastChecked is the last time this component's firmware was verified.
    LastChecked time.Time `json:"last_checked" db:"last_checked"`

    // LastUpdated is the last time the firmware was actually changed.
    LastUpdated *time.Time `json:"last_updated,omitempty" db:"last_updated"`

    // Source is the adapter that reported this version.
    Source string `json:"source" db:"source"`

    CreatedAt time.Time `json:"created_at" db:"created_at"`
    UpdatedAt time.Time `json:"updated_at" db:"updated_at"`
}

// FirmwarePolicy defines compliance requirements for firmware versions.
// Policies are evaluated against FirmwareInventory during compliance scans.
type FirmwarePolicy struct {
    // ID is the unique identifier for this policy.
    ID uuid.UUID `json:"id" db:"id"`

    // TenantID scopes this policy to a tenant.
    TenantID uuid.UUID `json:"tenant_id" db:"tenant_id"`

    // Name is a human-readable policy name (e.g., "Dell iDRAC minimum version").
    Name string `json:"name" db:"name"`

    // Description explains the policy rationale.
    Description string `json:"description,omitempty" db:"description"`

    // DeviceSelector matches devices by vendor, model, type, site, or tag.
    // JSON-encoded selector with fields: vendor, model, device_type, site_id, tags.
    // All fields are optional; a device must match ALL specified fields.
    DeviceSelector map[string]any `json:"device_selector" db:"-"`

    // Component is the firmware component this policy applies to.
    Component FirmwareComponentType `json:"component" db:"component"`

    // MinVersion is the minimum acceptable firmware version (semver or vendor string).
    MinVersion string `json:"min_version" db:"min_version"`

    // MaxVersion is an optional ceiling to avoid known-bad versions (nullable).
    MaxVersion *string `json:"max_version,omitempty" db:"max_version"`

    // BlockedVersions lists specific versions that must not be running.
    BlockedVersions []string `json:"blocked_versions,omitempty" db:"-"`

    // AutoStage determines whether matching firmware packages are automatically
    // staged to the device (downloaded but not applied).
    AutoStage bool `json:"auto_stage" db:"auto_stage"`

    // AutoApply determines whether staged firmware is automatically applied
    // during the next maintenance window.
    AutoApply bool `json:"auto_apply" db:"auto_apply"`

    // MaintenanceWindow is a cron expression defining when updates may be applied.
    // Nil means updates can be applied at any time (manual trigger still required if
    // AutoApply is false).
    MaintenanceWindow *string `json:"maintenance_window,omitempty" db:"maintenance_window"`

    // Severity classifies how critical non-compliance is.
    // "critical" = blocks workflows and generates alerts.
    // "warning" = generates alerts but does not block.
    // "info" = logged only.
    Severity string `json:"severity" db:"severity"` // "critical", "warning", "info"

    // Enabled controls whether this policy is actively evaluated.
    Enabled bool `json:"enabled" db:"enabled"`

    CreatedAt time.Time `json:"created_at" db:"created_at"`
    UpdatedAt time.Time `json:"updated_at" db:"updated_at"`
}

// FirmwarePackage represents a firmware binary available for installation.
// Stored in the artifact repository, not in the database.
type FirmwarePackage struct {
    // ID is the unique identifier for this package.
    ID uuid.UUID `json:"id" db:"id"`

    // TenantID scopes this package to a tenant.
    TenantID uuid.UUID `json:"tenant_id" db:"tenant_id"`

    // Vendor is the hardware vendor.
    Vendor string `json:"vendor" db:"vendor"`

    // Model is the hardware model this package applies to. Wildcard "*" means all models.
    Model string `json:"model" db:"model"`

    // Component is the firmware component type.
    Component FirmwareComponentType `json:"component" db:"component"`

    // Version is the firmware version this package installs.
    Version string `json:"version" db:"version"`

    // PackageURL is the location of the firmware binary.
    // Supports: "s3://bucket/path", "https://mirror.local/path", "file:///local/path".
    PackageURL string `json:"package_url" db:"package_url"`

    // Checksum is the SHA-256 hash of the firmware binary for integrity verification.
    Checksum string `json:"checksum" db:"checksum"`

    // ChecksumAlgo is the hash algorithm used (always "sha256" for now).
    ChecksumAlgo string `json:"checksum_algo" db:"checksum_algo"`

    // SizeBytes is the firmware binary size.
    SizeBytes int64 `json:"size_bytes" db:"size_bytes"`

    // RebootRequired indicates whether applying this firmware requires a reboot.
    RebootRequired bool `json:"reboot_required" db:"reboot_required"`

    // ReleaseNotes is a URL to the vendor's release notes.
    ReleaseNotes string `json:"release_notes,omitempty" db:"release_notes"`

    // UploadedBy records who added this package.
    UploadedBy string `json:"uploaded_by" db:"uploaded_by"`

    CreatedAt time.Time `json:"created_at" db:"created_at"`
}

// FirmwareUpdateOp records a firmware update operation executed as a Temporal workflow.
type FirmwareUpdateOp struct {
    // ID is the unique identifier for this update operation.
    ID uuid.UUID `json:"id" db:"id"`

    // TenantID scopes this operation to a tenant.
    TenantID uuid.UUID `json:"tenant_id" db:"tenant_id"`

    // DeviceID is the device being updated.
    DeviceID uuid.UUID `json:"device_id" db:"device_id"`

    // Component is the firmware component being updated.
    Component FirmwareComponentType `json:"component" db:"component"`

    // FromVersion is the version before the update.
    FromVersion string `json:"from_version" db:"from_version"`

    // TargetVersion is the version being installed.
    TargetVersion string `json:"target_version" db:"target_version"`

    // PackageID links to the firmware package used.
    PackageID uuid.UUID `json:"package_id" db:"package_id"`

    // PolicyID links to the policy that triggered this update (nullable for manual updates).
    PolicyID *uuid.UUID `json:"policy_id,omitempty" db:"policy_id"`

    // WorkflowID is the Temporal workflow ID executing this update.
    WorkflowID string `json:"workflow_id" db:"workflow_id"`

    // Status tracks the current phase of the update.
    Status FirmwareUpdateStatus `json:"status" db:"status"`

    // RollbackVersion is the version to revert to on failure.
    RollbackVersion string `json:"rollback_version" db:"rollback_version"`

    // Error captures the failure reason if the update failed.
    Error *string `json:"error,omitempty" db:"error"`

    // StartedAt is when the update workflow began.
    StartedAt time.Time `json:"started_at" db:"started_at"`

    // CompletedAt is when the update reached a terminal state (nullable).
    CompletedAt *time.Time `json:"completed_at,omitempty" db:"completed_at"`

    CreatedAt time.Time `json:"created_at" db:"created_at"`
    UpdatedAt time.Time `json:"updated_at" db:"updated_at"`
}
```

### SQL Schema

```sql
CREATE TABLE firmware_inventory (
    id                UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    device_id         UUID NOT NULL REFERENCES physical_devices(id),
    tenant_id         UUID NOT NULL,
    component         TEXT NOT NULL CHECK (component IN ('bios','bmc','nic','disk',
                      'gpu','cpld','psu','os','switch_os')),
    component_name    TEXT NOT NULL DEFAULT '',
    current_version   TEXT NOT NULL,
    available_version TEXT,
    compliance        TEXT NOT NULL DEFAULT 'unknown'
                      CHECK (compliance IN ('compliant','update_available',
                             'critical_cve','end_of_life','unknown')),
    last_checked      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    last_updated      TIMESTAMPTZ,
    source            TEXT NOT NULL DEFAULT '',
    created_at        TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at        TIMESTAMPTZ NOT NULL DEFAULT NOW(),

    -- One inventory record per component per device.
    CONSTRAINT uq_device_component UNIQUE (device_id, component, component_name)
);

CREATE INDEX idx_firmware_inv_device ON firmware_inventory (device_id);
CREATE INDEX idx_firmware_inv_compliance ON firmware_inventory (tenant_id, compliance);
CREATE INDEX idx_firmware_inv_component ON firmware_inventory (tenant_id, component);

CREATE TABLE firmware_policies (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id           UUID NOT NULL,
    name                TEXT NOT NULL,
    description         TEXT NOT NULL DEFAULT '',
    device_selector     JSONB NOT NULL DEFAULT '{}',
    component           TEXT NOT NULL CHECK (component IN ('bios','bmc','nic','disk',
                        'gpu','cpld','psu','os','switch_os')),
    min_version         TEXT NOT NULL,
    max_version         TEXT,
    blocked_versions    JSONB NOT NULL DEFAULT '[]',
    auto_stage          BOOLEAN NOT NULL DEFAULT FALSE,
    auto_apply          BOOLEAN NOT NULL DEFAULT FALSE,
    maintenance_window  TEXT,
    severity            TEXT NOT NULL DEFAULT 'warning'
                        CHECK (severity IN ('critical','warning','info')),
    enabled             BOOLEAN NOT NULL DEFAULT TRUE,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),

    CONSTRAINT uq_tenant_policy_name UNIQUE (tenant_id, name)
);

CREATE INDEX idx_firmware_policies_tenant ON firmware_policies (tenant_id);
CREATE INDEX idx_firmware_policies_component ON firmware_policies (tenant_id, component);

CREATE TABLE firmware_packages (
    id             UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id      UUID NOT NULL,
    vendor         TEXT NOT NULL,
    model          TEXT NOT NULL DEFAULT '*',
    component      TEXT NOT NULL CHECK (component IN ('bios','bmc','nic','disk',
                   'gpu','cpld','psu','os','switch_os')),
    version        TEXT NOT NULL,
    package_url    TEXT NOT NULL,
    checksum       TEXT NOT NULL,
    checksum_algo  TEXT NOT NULL DEFAULT 'sha256',
    size_bytes     BIGINT NOT NULL DEFAULT 0,
    reboot_required BOOLEAN NOT NULL DEFAULT FALSE,
    release_notes  TEXT NOT NULL DEFAULT '',
    uploaded_by    TEXT NOT NULL DEFAULT '',
    created_at     TIMESTAMPTZ NOT NULL DEFAULT NOW(),

    -- Unique package per vendor + model + component + version.
    CONSTRAINT uq_firmware_package
        UNIQUE (tenant_id, vendor, model, component, version)
);

CREATE INDEX idx_firmware_packages_lookup
    ON firmware_packages (tenant_id, vendor, component);

CREATE TABLE firmware_update_ops (
    id               UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id        UUID NOT NULL,
    device_id        UUID NOT NULL REFERENCES physical_devices(id),
    component        TEXT NOT NULL,
    from_version     TEXT NOT NULL,
    target_version   TEXT NOT NULL,
    package_id       UUID NOT NULL REFERENCES firmware_packages(id),
    policy_id        UUID REFERENCES firmware_policies(id),
    workflow_id      TEXT NOT NULL DEFAULT '',
    status           TEXT NOT NULL DEFAULT 'pending'
                     CHECK (status IN ('pending','staging','applying','rebooting',
                            'verifying','completed','failed','rolled_back')),
    rollback_version TEXT NOT NULL DEFAULT '',
    error            TEXT,
    started_at       TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    completed_at     TIMESTAMPTZ,
    created_at       TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at       TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_fw_update_device ON firmware_update_ops (device_id);
CREATE INDEX idx_fw_update_status ON firmware_update_ops (tenant_id, status);
CREATE INDEX idx_fw_update_workflow ON firmware_update_ops (workflow_id);

-- RLS on all firmware tables.
ALTER TABLE firmware_inventory ENABLE ROW LEVEL SECURITY;
ALTER TABLE firmware_policies ENABLE ROW LEVEL SECURITY;
ALTER TABLE firmware_packages ENABLE ROW LEVEL SECURITY;
ALTER TABLE firmware_update_ops ENABLE ROW LEVEL SECURITY;

CREATE POLICY tenant_isolation ON firmware_inventory
    USING (tenant_id = current_setting('app.current_tenant_id')::UUID);
CREATE POLICY tenant_isolation ON firmware_policies
    USING (tenant_id = current_setting('app.current_tenant_id')::UUID);
CREATE POLICY tenant_isolation ON firmware_packages
    USING (tenant_id = current_setting('app.current_tenant_id')::UUID);
CREATE POLICY tenant_isolation ON firmware_update_ops
    USING (tenant_id = current_setting('app.current_tenant_id')::UUID);
```

### Compliance Scan Workflow

```
Temporal Workflow: FirmwareComplianceScan
Schedule: Every 6 hours (configurable per tenant)

Steps:
  1. Activity: ListDevicesWithInventory(tenant_id)
     → Returns all (device_id, component, current_version) tuples

  2. Activity: ListActivePolicies(tenant_id)
     → Returns all enabled FirmwarePolicy records

  3. For each device inventory record:
     a. Find matching policies (by device_selector + component)
     b. Compare current_version against min_version, max_version, blocked_versions
     c. Determine ComplianceStatus:
        - current >= min AND (max is nil OR current <= max)
          AND current NOT IN blocked → "compliant"
        - current < min → "update_available"
        - current IN blocked → "critical_cve"
        - version is EOL per vendor data → "end_of_life"
     d. Activity: UpdateComplianceStatus(device_id, component, status)
     e. If status != "compliant" AND policy.severity == "critical":
        Emit event: loom.{tenant}.firmware.compliance_violation

  4. Activity: GenerateComplianceReport(tenant_id, scan_results)
     → Stores summary for dashboard/export

  5. If policy.auto_stage AND status == "update_available":
     Start child workflow: FirmwareUpdate(device_id, component, target_version)
```

### Vendor-Specific Update Adapters

| Vendor | Adapter | Update Mechanism | Notes |
|--------|---------|-----------------|-------|
| Dell | `redfish-dell` | Redfish UpdateService + DUP via `SimpleUpdate` action | iDRAC stages firmware, optional immediate install or deferred to maintenance window |
| HPE | `redfish-hpe` | Redfish UpdateService + `.fwpkg` upload | iLO auto-rollback on failure supported |
| Supermicro | `redfish-supermicro` | Vendor IPMI commands (non-standard Redfish in older models) | Some models require BMC reboot after update |
| Network switches | `switch-firmware` | Vendor CLI: `install add`, `software install`, `request system software add` | ISSU when supported; full reload otherwise |

Air-gapped firmware staging:
1. Administrator uploads firmware binary to local HTTP mirror (e.g., `https://firmware.internal/`)
2. Administrator creates FirmwarePackage record with `package_url = "https://firmware.internal/dell/idrac-6.10.00.00.bin"`
3. FirmwareUpdate workflow downloads from the internal URL

### Migration Notes

All new tables. `firmware_inventory.device_id` references `physical_devices.id`. The existing `Device.FirmwareVersion` field in DOMAIN-MODEL.md becomes a convenience accessor that reads the "os" or "bios" component from `firmware_inventory`. The column remains on the device table for backward compatibility but is no longer the authoritative source.

---

## 5. Identity Matcher Concurrency (H2)

### Decision

**Two-phase matching: optimistic fast path with PostgreSQL advisory locks + background reconciliation sweep.** Auto-merge above 0.9 confidence, human review queue for 0.7-0.9, automatic create below 0.7. Cross-tenant isolation is absolute — the same serial number in two tenants produces two separate PhysicalDevice records; they are never automatically merged across tenants.

### Resolved Open Questions

| Question | Decision |
|----------|----------|
| Acceptable duplicate rate for optimistic path? | Target: < 0.1%. Advisory locks on identity values make the optimistic path nearly collision-free. The reconciliation sweep exists as a safety net, not the primary deduplication mechanism. |
| Auto-merge or human approval? | Confidence >= 0.9: auto-merge. Confidence 0.7-0.9: merge but flag for review (reviewable in UI, auto-expires after 7 days if not rejected). Confidence < 0.7: no match, create new device. |
| Cross-tenant identity isolation? | Absolute. Advisory locks are scoped to `(tenant_id, identity_value)`. The same serial number in Tenant A and Tenant B produces two separate PhysicalDevice records. Platform admins can manually link them via the shared device model (Section 1) but this is never automatic. |

### Advisory Lock Pattern

```sql
-- Phase 1: Optimistic matching with advisory locks.
-- Called by the identity matcher for every discovery result.
--
-- Advisory locks are held for the duration of the transaction (pg_advisory_xact_lock).
-- They do NOT show up in pg_locks as regular row locks, so they have minimal
-- overhead and no risk of table-level lock escalation.

-- Step 1: Acquire advisory locks on all identity values in the discovery result.
-- The lock key is hash(tenant_id || identity_type || identity_value).
-- This serializes concurrent discoveries of the same device within a tenant.
SELECT pg_advisory_xact_lock(
    hashtext($1::TEXT || ':' || identity_value)
)
FROM unnest($2::TEXT[]) AS identity_value;
-- $1 = tenant_id (UUID cast to TEXT)
-- $2 = array of normalized identity strings, e.g.:
--   ARRAY['serial:ABC123', 'mac:00:11:22:33:44:55', 'bmc_ip:10.0.0.115']

-- Step 2: Search for existing devices matching any of these identities.
-- Now safe from concurrent inserts because we hold the advisory locks.
SELECT d.id AS device_id,
       ei.identity_type,
       ei.identity_value,
       ei.confidence
FROM external_identities ei
JOIN tenant_device_views tdv ON tdv.id = ei.device_id
WHERE ei.tenant_id = $1
  AND ei.superseded_at IS NULL
  AND (ei.identity_type, ei.identity_value) IN (
      SELECT * FROM unnest($3::TEXT[], $4::TEXT[])
  )
ORDER BY ei.confidence DESC;
-- $3 = identity types array
-- $4 = identity values array

-- Step 3: Based on match result:
--   (a) Unique match found, confidence >= 0.7 → MERGE (enrich existing device)
--   (b) No match found → INSERT new PhysicalDevice + TenantDeviceView + ExternalIdentities
--   (c) Multiple matches → Queue for Phase 2 reconciliation
--   (d) Match found but confidence < 0.7 → INSERT new device (no merge)

-- Step 4: Transaction commits, advisory locks released automatically.
```

### Phase 2: Background Reconciliation Sweep

```go
// ReconciliationSweep runs every 5 minutes (configurable).
// It detects duplicates that slipped through Phase 1 (rare, < 0.1% of cases).
type ReconciliationSweep struct {
    // IntervalMinutes is how often the sweep runs.
    IntervalMinutes int `json:"interval_minutes"`

    // LookbackMinutes is how far back to check for recently created devices.
    LookbackMinutes int `json:"lookback_minutes"`

    // AutoMergeThreshold is the minimum confidence for automatic merging.
    // Devices above this threshold are merged without human review.
    AutoMergeThreshold float64 `json:"auto_merge_threshold"` // 0.9

    // ReviewThreshold is the minimum confidence for flagging for review.
    // Devices between ReviewThreshold and AutoMergeThreshold are flagged.
    ReviewThreshold float64 `json:"review_threshold"` // 0.7

    // ReviewExpiryDays is how long a review flag stays open before auto-expiring.
    // After expiry, the devices remain separate (conservative default).
    ReviewExpiryDays int `json:"review_expiry_days"` // 7
}
```

```sql
-- Reconciliation sweep: find potential duplicates created in the last interval.
-- This query finds pairs of devices that share a high-confidence identity value
-- but have different DeviceIDs (i.e., they were created as separate devices
-- during a Phase 1 race condition).

SELECT
    ei1.device_id AS device_a,
    ei2.device_id AS device_b,
    ei1.identity_type,
    ei1.identity_value,
    GREATEST(ei1.confidence, ei2.confidence) AS match_confidence
FROM external_identities ei1
JOIN external_identities ei2
    ON ei1.identity_type = ei2.identity_type
   AND ei1.identity_value = ei2.identity_value
   AND ei1.tenant_id = ei2.tenant_id
   AND ei1.device_id < ei2.device_id  -- avoid self-join and duplicate pairs
WHERE ei1.tenant_id = $1
  AND ei1.superseded_at IS NULL
  AND ei2.superseded_at IS NULL
  AND ei1.confidence >= 0.7
  AND (
      ei1.created_at >= NOW() - INTERVAL '10 minutes'
      OR ei2.created_at >= NOW() - INTERVAL '10 minutes'
  )
ORDER BY match_confidence DESC;

-- For each pair found:
--   confidence >= 0.9 → auto-merge (keep older DeviceID, per IDENTITY-MODEL.md Section 3)
--   confidence 0.7-0.9 → insert into merge_review_queue
--   confidence < 0.7 → ignore (not a duplicate)
```

### Merge Review Queue

```sql
CREATE TABLE merge_review_queue (
    id           UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id    UUID NOT NULL,
    device_a_id  UUID NOT NULL,
    device_b_id  UUID NOT NULL,
    identity_type TEXT NOT NULL,
    identity_value TEXT NOT NULL,
    confidence   REAL NOT NULL,
    status       TEXT NOT NULL DEFAULT 'pending'
                 CHECK (status IN ('pending','approved','rejected','expired')),
    reviewed_by  UUID,         -- user ID who reviewed (nullable)
    reviewed_at  TIMESTAMPTZ,
    expires_at   TIMESTAMPTZ NOT NULL,
    created_at   TIMESTAMPTZ NOT NULL DEFAULT NOW(),

    CONSTRAINT uq_review_pair UNIQUE (device_a_id, device_b_id)
);

CREATE INDEX idx_merge_review_tenant ON merge_review_queue (tenant_id, status);
CREATE INDEX idx_merge_review_expires ON merge_review_queue (expires_at)
    WHERE status = 'pending';

ALTER TABLE merge_review_queue ENABLE ROW LEVEL SECURITY;
CREATE POLICY tenant_isolation ON merge_review_queue
    USING (tenant_id = current_setting('app.current_tenant_id')::UUID);
```

### Cross-Tenant Isolation Enforcement

```go
// IdentityMatcherConfig enforces tenant boundaries.
// The matcher NEVER searches across tenants.
type IdentityMatcherConfig struct {
    // CrossTenantMatchingEnabled is ALWAYS false.
    // This field exists only to make the design decision explicit and auditable.
    // If someone sets it to true, the matcher panics at startup.
    CrossTenantMatchingEnabled bool `json:"cross_tenant_matching_enabled"`
    // Must be false. Validated at startup:
    //   if config.CrossTenantMatchingEnabled { panic("cross-tenant matching is forbidden") }
}
```

The advisory lock key includes the tenant_id, so `hashtext('tenant-a-uuid:serial:ABC123')` and `hashtext('tenant-b-uuid:serial:ABC123')` produce different locks. Two tenants discovering the same physical device simultaneously do not interfere with each other and produce two independent PhysicalDevice records.

### Migration Notes

Adds the `merge_review_queue` table. The `external_identities` table must exist (derived from IDENTITY-MODEL.md's `ExternalIdentity` type). The advisory lock pattern is application-level logic, not a schema change.

---

## 6. DesiredState Comparison Specification (H4)

### Decision

**Typed property comparison with a code-defined mapping registry.** Comparison rules are defined in Go code (not stored in the database) to keep the comparison logic testable, version-controlled, and free from runtime schema dependencies. `DesiredState.Properties` remains `map[string]string` — the comparison layer handles type coercion. We do not change Properties to `map[string]any` because string encoding keeps the storage model simple and the comparison logic handles all type conversions.

### Resolved Open Questions

| Question | Decision |
|----------|----------|
| Comparison rules in DB or code? | Code-defined. The mapping registry is a Go map compiled into the binary. New property types are added by PR, not by API call. This ensures comparison semantics are version-controlled and unit-testable. |
| Should Properties become `map[string]any`? | No. `map[string]string` is sufficient. The comparison layer parses strings into typed values as needed. This keeps serialization/deserialization trivial and avoids JSON schema validation complexity. |
| Vendor-specific property name normalization? | The mapping registry includes a `Normalizer` function per property. Vendor-specific names are normalized by the adapter at ObservedFact creation time (e.g., Cisco "interface mtu" and Juniper "mtu" both produce `FactType = "interface.mtu"`). The comparison layer does not handle vendor normalization — that is the adapter's responsibility per ADAPTER-CONTRACT.md. |

### Go Types

```go
// CompareMode defines how a desired property value is compared against
// an observed fact value.
type CompareMode int

const (
    // CompareExact performs string equality after normalization.
    // Used for: power_state, hostname, os_type, vlan_name.
    CompareExact CompareMode = iota

    // CompareNumericTolerance parses both values as float64 and compares
    // within the specified tolerance.
    // Used for: temperature, voltage, fan_speed, utilization percentages.
    CompareNumericTolerance

    // CompareContains checks whether the observed value contains the desired
    // value as a substring (after normalization).
    // Used for: firmware version prefixes, interface descriptions.
    CompareContains

    // CompareRegex treats the desired value as a regex pattern and matches
    // it against the observed value.
    // Used for: banner messages, version patterns, flexible matching.
    CompareRegex

    // CompareJSONPath extracts a value from a JSON-encoded observed fact
    // using a JSONPath expression, then compares the extracted value.
    // Used for: structured facts like interface lists, VLAN membership.
    CompareJSONPath

    // CompareSetEquality parses both values as JSON arrays and compares
    // as unordered sets.
    // Used for: VLAN member lists, NTP server lists, DNS server lists.
    CompareSetEquality

    // CompareVersionGTE parses both values as version strings and checks
    // that observed >= desired.
    // Used for: minimum firmware version, minimum OS version.
    CompareVersionGTE
)

// PropertyComparison defines how to compare a single property.
type PropertyComparison struct {
    // Mode is the comparison algorithm.
    Mode CompareMode `json:"mode"`

    // Tolerance is the acceptable numeric difference (only for CompareNumericTolerance).
    // For example, Tolerance=2.0 means observed=23.5 matches desired=25.0.
    Tolerance float64 `json:"tolerance,omitempty"`

    // JSONPath is the path expression for CompareJSONPath mode.
    // Uses RFC 9535 JSONPath syntax (e.g., "$.vlan_ids", "$.interfaces[0].mtu").
    JSONPath string `json:"json_path,omitempty"`

    // CaseInsensitive normalizes both values to lowercase before comparison.
    CaseInsensitive bool `json:"case_insensitive"`

    // TrimWhitespace strips leading/trailing whitespace before comparison.
    TrimWhitespace bool `json:"trim_whitespace"`
}

// PropertyMapping links a DesiredState property name to the ObservedFact
// type it should be compared against, along with the comparison rules.
type PropertyMapping struct {
    // FactType is the ObservedFact.FactType to query for comparison.
    FactType string `json:"fact_type"`

    // Comparison defines how to compare the desired value against the fact value.
    Comparison PropertyComparison `json:"comparison"`

    // Description documents what this property represents.
    Description string `json:"description"`
}

// PropertyMappingRegistry is the authoritative mapping of DesiredState
// property names to ObservedFact types and comparison rules.
// This is a code-defined constant, not a database table.
// Add new entries by submitting a pull request.
var PropertyMappingRegistry = map[string]PropertyMapping{
    "power_state": {
        FactType:    "power.state",
        Comparison:  PropertyComparison{Mode: CompareExact, CaseInsensitive: true, TrimWhitespace: true},
        Description: "Server power state: on, off, standby",
    },
    "hostname": {
        FactType:    "system.hostname",
        Comparison:  PropertyComparison{Mode: CompareExact, CaseInsensitive: true, TrimWhitespace: true},
        Description: "Device hostname",
    },
    "mtu": {
        FactType:    "interface.mtu",
        Comparison:  PropertyComparison{Mode: CompareNumericTolerance, Tolerance: 0},
        Description: "Interface MTU (exact match, no tolerance)",
    },
    "cpu_temp": {
        FactType:    "sensor.cpu_temperature",
        Comparison:  PropertyComparison{Mode: CompareNumericTolerance, Tolerance: 5.0},
        Description: "CPU temperature in Celsius (within 5C tolerance)",
    },
    "inlet_temp": {
        FactType:    "sensor.inlet_temperature",
        Comparison:  PropertyComparison{Mode: CompareNumericTolerance, Tolerance: 3.0},
        Description: "Inlet air temperature in Celsius (within 3C tolerance)",
    },
    "vlan_membership": {
        FactType:    "interface.vlans",
        Comparison:  PropertyComparison{Mode: CompareSetEquality, JSONPath: "$.vlan_ids"},
        Description: "VLAN membership list (unordered set comparison)",
    },
    "firmware_version": {
        FactType:    "firmware.version",
        Comparison:  PropertyComparison{Mode: CompareVersionGTE},
        Description: "Minimum firmware version (observed must be >= desired)",
    },
    "os_version": {
        FactType:    "system.os_version",
        Comparison:  PropertyComparison{Mode: CompareContains, CaseInsensitive: true},
        Description: "OS version string (desired is a prefix/substring of observed)",
    },
    "ntp_servers": {
        FactType:    "system.ntp_servers",
        Comparison:  PropertyComparison{Mode: CompareSetEquality},
        Description: "NTP server list (unordered set comparison)",
    },
    "dns_servers": {
        FactType:    "system.dns_servers",
        Comparison:  PropertyComparison{Mode: CompareSetEquality},
        Description: "DNS resolver list (unordered set comparison)",
    },
    "interface_state": {
        FactType:    "interface.admin_state",
        Comparison:  PropertyComparison{Mode: CompareExact, CaseInsensitive: true},
        Description: "Interface admin state: up, down",
    },
    "interface_description": {
        FactType:    "interface.description",
        Comparison:  PropertyComparison{Mode: CompareContains, CaseInsensitive: true, TrimWhitespace: true},
        Description: "Interface description (substring match)",
    },
    "boot_order": {
        FactType:    "bios.boot_order",
        Comparison:  PropertyComparison{Mode: CompareExact},
        Description: "BIOS boot order (JSON-encoded ordered list, exact match)",
    },
}
```

### Comparison Execution

```go
// CompareProperty executes a single property comparison between a desired value
// and an observed fact value, using the rules defined in the mapping registry.
//
// Returns (passed bool, details string).
// The details string contains human-readable comparison information for the
// VerificationResult.Evidence field.
func CompareProperty(propertyName string, desiredValue string, observedFact ObservedFact) (bool, string) {
    mapping, ok := PropertyMappingRegistry[propertyName]
    if !ok {
        return false, fmt.Sprintf("unknown property %q: no mapping in registry", propertyName)
    }

    desired := desiredValue
    observed := observedFact.Value

    // Apply normalization.
    if mapping.Comparison.TrimWhitespace {
        desired = strings.TrimSpace(desired)
        observed = strings.TrimSpace(observed)
    }
    if mapping.Comparison.CaseInsensitive {
        desired = strings.ToLower(desired)
        observed = strings.ToLower(observed)
    }

    switch mapping.Comparison.Mode {
    case CompareExact:
        passed := desired == observed
        return passed, fmt.Sprintf("exact: desired=%q observed=%q", desired, observed)

    case CompareNumericTolerance:
        dv, _ := strconv.ParseFloat(desired, 64)
        ov, _ := strconv.ParseFloat(observed, 64)
        diff := math.Abs(dv - ov)
        passed := diff <= mapping.Comparison.Tolerance
        return passed, fmt.Sprintf("numeric: desired=%.2f observed=%.2f diff=%.2f tolerance=%.2f",
            dv, ov, diff, mapping.Comparison.Tolerance)

    case CompareContains:
        passed := strings.Contains(observed, desired)
        return passed, fmt.Sprintf("contains: desired=%q in observed=%q → %v", desired, observed, passed)

    case CompareRegex:
        re, err := regexp.Compile(desired)
        if err != nil {
            return false, fmt.Sprintf("invalid regex %q: %v", desired, err)
        }
        passed := re.MatchString(observed)
        return passed, fmt.Sprintf("regex: pattern=%q observed=%q → %v", desired, observed, passed)

    case CompareJSONPath:
        // Extract value from observed JSON using JSONPath, then compare as exact.
        extracted := jsonpath.Extract(observed, mapping.Comparison.JSONPath)
        passed := desired == extracted
        return passed, fmt.Sprintf("jsonpath: path=%q extracted=%q desired=%q",
            mapping.Comparison.JSONPath, extracted, desired)

    case CompareSetEquality:
        // Parse both as JSON arrays, compare as unordered sets.
        dSet := parseStringSet(desired)
        oSet := parseStringSet(observed)
        passed := dSet.Equal(oSet)
        return passed, fmt.Sprintf("set: desired=%v observed=%v", dSet, oSet)

    case CompareVersionGTE:
        cmp := semver.Compare(observed, desired) // >= 0 means observed >= desired
        passed := cmp >= 0
        return passed, fmt.Sprintf("version: observed=%q >= desired=%q → %v", observed, desired, passed)

    default:
        return false, fmt.Sprintf("unknown compare mode %d", mapping.Comparison.Mode)
    }
}
```

### Normalization Rules (Definitive)

| Rule | Applied When | Example |
|------|-------------|---------|
| Lowercase | `CaseInsensitive = true` | `"On"` -> `"on"`, `"PoweredOff"` -> `"poweredoff"` |
| Trim whitespace | `TrimWhitespace = true` | `" 9000 "` -> `"9000"` |
| Numeric parsing | `CompareNumericTolerance` | `"9000"` -> `9000.0`, `"9.0k"` -> parse error (strict) |
| Set parsing | `CompareSetEquality` | `"[1,2,3]"` -> `{1,2,3}`, `"1,2,3"` -> `{1,2,3}` |
| Version parsing | `CompareVersionGTE` | `"6.10.00.00"` handled by semver-compatible parser |

Numeric values must be parseable by Go's `strconv.ParseFloat`. No unit conversion (e.g., "9k" is not interpreted as 9000). Adapters are responsible for reporting values in canonical numeric form.

### Migration Notes

No schema changes. The `PropertyMappingRegistry` is code-only. The existing `DesiredState.Properties` type (`map[string]string`) is unchanged. The `VerificationResult.Evidence` field gains richer comparison details from the `CompareProperty` function.

---

## 7. DesiredState Conflict Resolution (H7)

### Decision

**Priority + last-write-wins with auto-expiry.** DesiredState declarations carry a priority derived from their source type, a monotonically increasing version, and an optional expiry time. The drift detector evaluates only the winning DesiredState per property. Conflicts are logged as events.

### Resolved Open Questions

| Question | Decision |
|----------|----------|
| Should conflicting DesiredState prevent the second workflow from starting? | No. Both declarations are stored. The higher-priority one wins. The lower-priority one is marked as `shadowed` but remains in the database. If the higher-priority one expires or is retired, the lower-priority one automatically takes effect. This avoids workflow submission failures that would be confusing for operators. |
| Should the drift detector consider all active DesiredState or only the winning one? | Only the winning one. The drift detector resolves conflicts at query time by selecting the highest-priority, latest-version DesiredState per (resource_id, property). This is a single SQL query, not a multi-pass algorithm. |
| How are policy-based DesiredState distinguished from workflow-based? | By `SourceType`. Policies produce DesiredState with `SourceType = "policy"` and `Priority = 300`. Workflows produce `SourceType = "workflow"` with `Priority = 200`. Manual overrides produce `SourceType = "manual"` with `Priority = 100`. The priority values are conventions, not hardcoded — operators can override per-declaration. |

### Updated Go Types

```go
// DesiredStateSourceType classifies the origin of a desired state declaration.
type DesiredStateSourceType string

const (
    DesiredStateSourcePolicy   DesiredStateSourceType = "policy"
    DesiredStateSourceWorkflow DesiredStateSourceType = "workflow"
    DesiredStateSourceManual   DesiredStateSourceType = "manual"
)

// DesiredStateStatus tracks whether this declaration is active.
type DesiredStateStatus string

const (
    DesiredStateActive   DesiredStateStatus = "active"
    DesiredStateRetired  DesiredStateStatus = "retired"
    DesiredStateShadowed DesiredStateStatus = "shadowed"
    DesiredStateExpired  DesiredStateStatus = "expired"
)

// DesiredState declares the intended state of a resource.
// Updated from DOMAIN-MODEL.md to include priority, source, expiry, and versioning.
type DesiredState struct {
    // ID is the unique identifier for this desired state declaration.
    ID uuid.UUID `json:"id" db:"id"`

    // TenantID scopes this desired state to a tenant.
    TenantID uuid.UUID `json:"tenant_id" db:"tenant_id"`

    // ResourceType is the kind of resource ("device", "vlan", "interface", "acl").
    ResourceType string `json:"resource_type" db:"resource_type"`

    // ResourceID is the UUID of the resource this desired state applies to.
    ResourceID uuid.UUID `json:"resource_id" db:"resource_id"`

    // Properties maps property names to their desired values.
    Properties map[string]string `json:"properties" db:"-"`

    // Priority determines which declaration wins on conflict.
    // Higher value = higher priority. Default priorities by source:
    //   policy = 300, workflow = 200, manual = 100.
    // Operators can override these defaults per declaration.
    Priority int `json:"priority" db:"priority"`

    // SourceType classifies the origin of this declaration.
    SourceType DesiredStateSourceType `json:"source_type" db:"source_type"`

    // SourceID identifies the specific source (workflow ID, policy ID, or user ID).
    SourceID uuid.UUID `json:"source_id" db:"source_id"`

    // SourceName is a human-readable label for the source
    // (e.g., "ntp-compliance-policy", "provisioning-workflow-42").
    SourceName string `json:"source_name" db:"source_name"`

    // Version is a monotonically increasing counter per (resource_id, source_id).
    // Used for last-write-wins within the same priority level.
    Version int `json:"version" db:"version"`

    // Status tracks whether this declaration is currently active.
    Status DesiredStateStatus `json:"status" db:"status"`

    // ActiveUntil is the optional expiry time. Nil means permanent.
    // After expiry, the drift detector ignores this declaration and the
    // next-highest-priority declaration (if any) takes effect.
    ActiveUntil *time.Time `json:"active_until,omitempty" db:"active_until"`

    // DeclaredBy is the workflow or user that created this declaration.
    // Kept for backward compatibility with DOMAIN-MODEL.md.
    DeclaredBy string `json:"declared_by" db:"declared_by"`

    // DeclaredAt is the time this desired state was declared.
    DeclaredAt time.Time `json:"declared_at" db:"declared_at"`

    // RetiredAt is the time this declaration was retired (nullable).
    RetiredAt *time.Time `json:"retired_at,omitempty" db:"retired_at"`

    CreatedAt time.Time `json:"created_at" db:"created_at"`
    UpdatedAt time.Time `json:"updated_at" db:"updated_at"`
}
```

### SQL Schema Changes

```sql
-- Updated desired_states table (replaces the schema implied by DOMAIN-MODEL.md).
CREATE TABLE desired_states (
    id            UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id     UUID NOT NULL,
    resource_type TEXT NOT NULL,
    resource_id   UUID NOT NULL,
    properties    JSONB NOT NULL DEFAULT '{}',
    priority      INTEGER NOT NULL DEFAULT 200,
    source_type   TEXT NOT NULL DEFAULT 'manual'
                  CHECK (source_type IN ('policy','workflow','manual')),
    source_id     UUID NOT NULL,
    source_name   TEXT NOT NULL DEFAULT '',
    version       INTEGER NOT NULL DEFAULT 1,
    status        TEXT NOT NULL DEFAULT 'active'
                  CHECK (status IN ('active','retired','shadowed','expired')),
    active_until  TIMESTAMPTZ,
    declared_by   TEXT NOT NULL DEFAULT '',
    declared_at   TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    retired_at    TIMESTAMPTZ,
    created_at    TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at    TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- Primary lookup: active desired states for a resource.
CREATE INDEX idx_desired_states_resource
    ON desired_states (tenant_id, resource_type, resource_id)
    WHERE status = 'active';

-- Source lookup: find all declarations by a workflow or policy.
CREATE INDEX idx_desired_states_source
    ON desired_states (tenant_id, source_type, source_id);

-- Expiry sweep: find expired declarations to mark as 'expired'.
CREATE INDEX idx_desired_states_expiry
    ON desired_states (active_until)
    WHERE status = 'active' AND active_until IS NOT NULL;

-- Version ordering for conflict resolution.
CREATE INDEX idx_desired_states_priority
    ON desired_states (resource_id, priority DESC, version DESC)
    WHERE status = 'active';

ALTER TABLE desired_states ENABLE ROW LEVEL SECURITY;
CREATE POLICY tenant_isolation ON desired_states
    USING (tenant_id = current_setting('app.current_tenant_id')::UUID);
```

### Conflict Resolution Query

```sql
-- Resolve the winning DesiredState for each property on a resource.
-- This is the query the drift detector runs to determine what state to verify.
--
-- Returns: one row per property, with the highest-priority, latest-version value.
WITH active_states AS (
    SELECT
        id,
        properties,
        priority,
        version,
        source_type,
        source_name
    FROM desired_states
    WHERE tenant_id = $1
      AND resource_type = $2
      AND resource_id = $3
      AND status = 'active'
      AND (active_until IS NULL OR active_until > NOW())
),
-- Flatten properties: one row per (desired_state_id, property_key, property_value).
flattened AS (
    SELECT
        a.id AS desired_state_id,
        a.priority,
        a.version,
        a.source_type,
        a.source_name,
        kv.key AS property_name,
        kv.value AS property_value
    FROM active_states a,
    LATERAL jsonb_each_text(a.properties) AS kv(key, value)
),
-- Rank by priority DESC, version DESC per property.
ranked AS (
    SELECT *,
        ROW_NUMBER() OVER (
            PARTITION BY property_name
            ORDER BY priority DESC, version DESC
        ) AS rn
    FROM flattened
)
SELECT
    property_name,
    property_value,
    desired_state_id,
    priority,
    version,
    source_type,
    source_name
FROM ranked
WHERE rn = 1;
```

### Auto-Retirement Rules

```go
// DesiredState auto-retirement is handled by a periodic Temporal workflow
// running every 60 seconds.

// Rule 1: Expired declarations.
// UPDATE desired_states SET status = 'expired', retired_at = NOW()
// WHERE status = 'active' AND active_until IS NOT NULL AND active_until < NOW();

// Rule 2: Completed/cancelled workflow declarations.
// When a workflow reaches terminal state (completed, failed, cancelled),
// the workflow engine calls RetireDesiredState(workflowID).
// UPDATE desired_states SET status = 'retired', retired_at = NOW()
// WHERE source_type = 'workflow' AND source_id = $1 AND status = 'active';

// Rule 3: Shadow detection.
// After any DesiredState insert or status change, re-evaluate all active
// declarations for the same (resource_id, property). Lower-priority
// declarations are marked 'shadowed'. If the higher-priority one is retired
// or expires, shadowed declarations are re-evaluated and may become 'active'.
```

### Conflict Event

```go
// Emitted when a new DesiredState shadows an existing one.
type DesiredStateConflictEvent struct {
    TenantID       uuid.UUID `json:"tenant_id"`
    ResourceType   string    `json:"resource_type"`
    ResourceID     uuid.UUID `json:"resource_id"`
    PropertyName   string    `json:"property_name"`
    WinnerID       uuid.UUID `json:"winner_id"`       // DesiredState that won
    WinnerValue    string    `json:"winner_value"`
    WinnerSource   string    `json:"winner_source"`
    WinnerPriority int       `json:"winner_priority"`
    ShadowedID     uuid.UUID `json:"shadowed_id"`     // DesiredState that lost
    ShadowedValue  string    `json:"shadowed_value"`
    ShadowedSource string    `json:"shadowed_source"`
    Timestamp      time.Time `json:"timestamp"`
}
// Published to: loom.{tenant_id}.desired_state.conflict
```

### Migration Notes

The `desired_states` table schema changes from DOMAIN-MODEL.md. New columns: `priority`, `source_type`, `source_id`, `source_name`, `version`, `status`, `active_until`, `retired_at`. The existing `declared_by` column is retained for backward compatibility. Existing rows get `priority = 200`, `source_type = 'manual'`, `version = 1`, `status = 'active'`.

---

## 8. Storage Model (H20)

### Decision

**Schema-only types in Phase 6, no storage adapters until Phase 8+.** LOOM observes storage provisioned by external tools (CSI, Trident, NetApp, Pure, Ceph) but does not directly manage storage arrays until Phase 8+. The schema is defined now to prevent breaking changes later.

### Resolved Open Questions

| Question | Decision |
|----------|----------|
| Storage orchestration in scope for MVP? | No. Deferred to Phase 8+. The schema is defined now but no adapters, workflows, or API endpoints are implemented until Phase 6 (schema) and Phase 8 (adapters). |
| Should LOOM manage storage directly or observe/verify? | Phase 6-7: observe only. LOOM discovers storage pools and volumes via adapters and records them as ObservedFacts. Phase 8+: LOOM can provision volumes through CSI or vendor APIs. |
| How do storage operations differ in compensation? | Volume delete = data loss, irreversible. Storage operations use a `RequireConfirmation` flag that forces human approval before destructive operations. No automatic compensation for volume deletes — the compensation is "alert operator, do not recreate." |

### Go Types

```go
// StorageType classifies the storage access model.
type StorageType string

const (
    StorageTypeBlock  StorageType = "block"
    StorageTypeFile   StorageType = "file"
    StorageTypeObject StorageType = "object"
)

// StoragePoolStatus tracks the operational state of a storage pool.
type StoragePoolStatus string

const (
    StoragePoolOnline    StoragePoolStatus = "online"
    StoragePoolDegraded  StoragePoolStatus = "degraded"
    StoragePoolOffline   StoragePoolStatus = "offline"
    StoragePoolUnknown   StoragePoolStatus = "unknown"
)

// StorageTier classifies the performance tier of storage media.
type StorageTier string

const (
    StorageTierNVMe   StorageTier = "nvme"
    StorageTierSSD    StorageTier = "ssd"
    StorageTierHDD    StorageTier = "hdd"
    StorageTierHybrid StorageTier = "hybrid"
)

// VolumeStatus tracks the state of a volume.
type VolumeStatus string

const (
    VolumeStatusAvailable VolumeStatus = "available"
    VolumeStatusAttached  VolumeStatus = "attached"
    VolumeStatusDetaching VolumeStatus = "detaching"
    VolumeStatusDeleting  VolumeStatus = "deleting"
    VolumeStatusError     VolumeStatus = "error"
)

// StoragePool represents a storage pool on a storage controller or array.
// Discovered by adapters (Redfish, NetApp API, CSI) and stored as inventory.
type StoragePool struct {
    // ID is the unique identifier for this pool.
    ID uuid.UUID `json:"id" db:"id"`

    // TenantID scopes this pool to a tenant.
    TenantID uuid.UUID `json:"tenant_id" db:"tenant_id"`

    // DeviceID links to the storage controller or array (PhysicalDevice).
    DeviceID uuid.UUID `json:"device_id" db:"device_id"`

    // Name is the pool name as reported by the storage system
    // (e.g., "aggr1", "tank", "default-storage-pool").
    Name string `json:"name" db:"name"`

    // Type classifies the storage access model.
    Type StorageType `json:"type" db:"type"`

    // Tier classifies the performance tier.
    Tier StorageTier `json:"tier" db:"tier"`

    // TotalBytes is the total capacity in bytes.
    TotalBytes int64 `json:"total_bytes" db:"total_bytes"`

    // UsedBytes is the currently consumed capacity in bytes.
    UsedBytes int64 `json:"used_bytes" db:"used_bytes"`

    // AvailableBytes is the remaining allocatable capacity.
    // May differ from TotalBytes - UsedBytes due to reservations, overhead, etc.
    AvailableBytes int64 `json:"available_bytes" db:"available_bytes"`

    // Status is the operational state of the pool.
    Status StoragePoolStatus `json:"status" db:"status"`

    // RAID is the RAID level or data protection scheme (e.g., "raid-dp", "raid6",
    // "mirrored", "erasure-coded", "none").
    RAID string `json:"raid,omitempty" db:"raid"`

    // Deduplication indicates whether dedup is enabled.
    Deduplication bool `json:"deduplication" db:"deduplication"`

    // Compression indicates whether compression is enabled.
    Compression bool `json:"compression" db:"compression"`

    // Source is the adapter that discovered this pool.
    Source string `json:"source" db:"source"`

    CreatedAt time.Time `json:"created_at" db:"created_at"`
    UpdatedAt time.Time `json:"updated_at" db:"updated_at"`
}

// Volume represents a logical volume, LUN, or share provisioned from a StoragePool.
type Volume struct {
    // ID is the unique identifier for this volume.
    ID uuid.UUID `json:"id" db:"id"`

    // TenantID scopes this volume to a tenant.
    TenantID uuid.UUID `json:"tenant_id" db:"tenant_id"`

    // PoolID links to the parent storage pool.
    PoolID uuid.UUID `json:"pool_id" db:"pool_id"`

    // Name is the volume name (e.g., "vol_data_01", "pvc-abc123").
    Name string `json:"name" db:"name"`

    // SizeBytes is the provisioned size in bytes.
    SizeBytes int64 `json:"size_bytes" db:"size_bytes"`

    // UsedBytes is the actual space consumed (for thin-provisioned volumes).
    UsedBytes int64 `json:"used_bytes" db:"used_bytes"`

    // Protocol is the access protocol.
    Protocol string `json:"protocol" db:"protocol"`
    // Known values: "iscsi", "fc", "nfs", "smb", "s3", "nvme-of", "csi"

    // Status is the volume lifecycle state.
    Status VolumeStatus `json:"status" db:"status"`

    // ThinProvisioned indicates whether the volume is thin-provisioned.
    ThinProvisioned bool `json:"thin_provisioned" db:"thin_provisioned"`

    // MountedOnDeviceID is the device this volume is attached/mounted to (nullable).
    MountedOnDeviceID *uuid.UUID `json:"mounted_on_device_id,omitempty" db:"mounted_on_device_id"`

    // MountPath is the filesystem mount point (for NFS/SMB volumes) (nullable).
    MountPath *string `json:"mount_path,omitempty" db:"mount_path"`

    // IQN is the iSCSI Qualified Name (for iSCSI volumes) (nullable).
    IQN *string `json:"iqn,omitempty" db:"iqn"`

    // WWN is the World Wide Name (for FC volumes) (nullable).
    WWN *string `json:"wwn,omitempty" db:"wwn"`

    // ExportPath is the NFS export path (for NFS volumes) (nullable).
    ExportPath *string `json:"export_path,omitempty" db:"export_path"`

    // SnapshotPolicyID links to a snapshot policy (nullable, Phase 8+).
    SnapshotPolicyID *uuid.UUID `json:"snapshot_policy_id,omitempty" db:"snapshot_policy_id"`

    // RPOSeconds is the recovery point objective in seconds (nullable).
    // Used by compliance checks to verify backup frequency meets requirements.
    RPOSeconds *int `json:"rpo_seconds,omitempty" db:"rpo_seconds"`

    // RTOSeconds is the recovery time objective in seconds (nullable).
    RTOSeconds *int `json:"rto_seconds,omitempty" db:"rto_seconds"`

    // Source is the adapter that discovered or provisioned this volume.
    Source string `json:"source" db:"source"`

    // RequireConfirmation forces human approval before delete operations.
    // Always true for volumes containing data. Cannot be set to false via API
    // without explicit admin override.
    RequireConfirmation bool `json:"require_confirmation" db:"require_confirmation"`

    CreatedAt time.Time `json:"created_at" db:"created_at"`
    UpdatedAt time.Time `json:"updated_at" db:"updated_at"`
}
```

### SQL Schema

```sql
CREATE TABLE storage_pools (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL,
    device_id       UUID NOT NULL REFERENCES physical_devices(id),
    name            TEXT NOT NULL,
    type            TEXT NOT NULL CHECK (type IN ('block','file','object')),
    tier            TEXT NOT NULL DEFAULT 'hdd'
                    CHECK (tier IN ('nvme','ssd','hdd','hybrid')),
    total_bytes     BIGINT NOT NULL DEFAULT 0 CHECK (total_bytes >= 0),
    used_bytes      BIGINT NOT NULL DEFAULT 0 CHECK (used_bytes >= 0),
    available_bytes BIGINT NOT NULL DEFAULT 0 CHECK (available_bytes >= 0),
    status          TEXT NOT NULL DEFAULT 'unknown'
                    CHECK (status IN ('online','degraded','offline','unknown')),
    raid            TEXT NOT NULL DEFAULT '',
    deduplication   BOOLEAN NOT NULL DEFAULT FALSE,
    compression     BOOLEAN NOT NULL DEFAULT FALSE,
    source          TEXT NOT NULL DEFAULT '',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),

    -- Pool name is unique per device.
    CONSTRAINT uq_device_pool_name UNIQUE (device_id, name)
);

CREATE INDEX idx_storage_pools_tenant ON storage_pools (tenant_id);
CREATE INDEX idx_storage_pools_device ON storage_pools (device_id);
CREATE INDEX idx_storage_pools_type ON storage_pools (tenant_id, type);

CREATE TABLE volumes (
    id                    UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id             UUID NOT NULL,
    pool_id               UUID NOT NULL REFERENCES storage_pools(id),
    name                  TEXT NOT NULL,
    size_bytes            BIGINT NOT NULL DEFAULT 0 CHECK (size_bytes >= 0),
    used_bytes            BIGINT NOT NULL DEFAULT 0 CHECK (used_bytes >= 0),
    protocol              TEXT NOT NULL DEFAULT 'iscsi'
                          CHECK (protocol IN ('iscsi','fc','nfs','smb','s3',
                                 'nvme-of','csi')),
    status                TEXT NOT NULL DEFAULT 'available'
                          CHECK (status IN ('available','attached','detaching',
                                 'deleting','error')),
    thin_provisioned      BOOLEAN NOT NULL DEFAULT FALSE,
    mounted_on_device_id  UUID,
    mount_path            TEXT,
    iqn                   TEXT,
    wwn                   TEXT,
    export_path           TEXT,
    snapshot_policy_id    UUID,
    rpo_seconds           INTEGER,
    rto_seconds           INTEGER,
    source                TEXT NOT NULL DEFAULT '',
    require_confirmation  BOOLEAN NOT NULL DEFAULT TRUE,
    created_at            TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at            TIMESTAMPTZ NOT NULL DEFAULT NOW(),

    -- Volume name is unique per pool.
    CONSTRAINT uq_pool_volume_name UNIQUE (pool_id, name)
);

CREATE INDEX idx_volumes_pool ON volumes (pool_id);
CREATE INDEX idx_volumes_tenant ON volumes (tenant_id);
CREATE INDEX idx_volumes_mounted ON volumes (mounted_on_device_id)
    WHERE mounted_on_device_id IS NOT NULL;
CREATE INDEX idx_volumes_status ON volumes (tenant_id, status);

-- RLS on storage tables.
ALTER TABLE storage_pools ENABLE ROW LEVEL SECURITY;
ALTER TABLE volumes ENABLE ROW LEVEL SECURITY;

CREATE POLICY tenant_isolation ON storage_pools
    USING (tenant_id = current_setting('app.current_tenant_id')::UUID);
CREATE POLICY tenant_isolation ON volumes
    USING (tenant_id = current_setting('app.current_tenant_id')::UUID);
```

### Example Queries

```sql
-- Q1: Storage utilization by tier for a tenant.
SELECT
    tier,
    COUNT(*) AS pool_count,
    SUM(total_bytes) AS total_capacity,
    SUM(used_bytes) AS used_capacity,
    ROUND(SUM(used_bytes)::NUMERIC / NULLIF(SUM(total_bytes), 0) * 100, 1)
        AS utilization_pct
FROM storage_pools
WHERE tenant_id = $1
  AND status != 'offline'
GROUP BY tier
ORDER BY tier;

-- Q2: Volumes attached to a specific device.
SELECT v.name, v.size_bytes, v.protocol, v.mount_path, sp.name AS pool_name
FROM volumes v
JOIN storage_pools sp ON sp.id = v.pool_id
WHERE v.mounted_on_device_id = $1
  AND v.status = 'attached'
ORDER BY v.name;

-- Q3: Thin-provisioned volumes with high overcommit risk.
-- Finds volumes where used_bytes > 80% of size_bytes.
SELECT v.name, v.size_bytes, v.used_bytes,
       ROUND(v.used_bytes::NUMERIC / NULLIF(v.size_bytes, 0) * 100, 1) AS used_pct,
       sp.name AS pool_name, sp.available_bytes AS pool_remaining
FROM volumes v
JOIN storage_pools sp ON sp.id = v.pool_id
WHERE v.tenant_id = $1
  AND v.thin_provisioned = TRUE
  AND v.used_bytes::NUMERIC / NULLIF(v.size_bytes, 0) > 0.8
ORDER BY used_pct DESC;

-- Q4: RPO compliance check — volumes with RPO > 0 that haven't been
-- snapshotted within their RPO window. (Phase 8+, when snapshot tracking exists.)
-- Placeholder for future use.
```

### Scope Timeline

| Phase | Storage Capability |
|-------|--------------------|
| 1-5 | No storage model in database. Storage is out of scope. |
| 6 | Schema deployed (tables created). No adapters, no API endpoints. Manual data import only. |
| 7 | Read-only storage adapters: discover pools and volumes from NetApp, Pure, Ceph, CSI. |
| 8+ | Read-write storage adapters: provision/resize/delete volumes. Snapshot policies. Replication. |

### Migration Notes

All new tables. `storage_pools.device_id` references `physical_devices.id`. `volumes.mounted_on_device_id` references `physical_devices.id`. These tables are created in Phase 6 but remain empty until Phase 7 adapters populate them.

---

## Cross-Cutting Concerns

### Foreign Key Reference Summary

All cross-references between the new types and existing DOMAIN-MODEL.md types:

| New Table | Column | References | Table |
|-----------|--------|------------|-------|
| tenant_device_views | physical_device_id | id | physical_devices |
| tenant_device_views | tenant_id | (app-enforced) | tenants |
| ip_prefixes | site_id | id | sites |
| ip_addresses | device_id | id | tenant_device_views |
| rack_mounts | device_id | id | physical_devices |
| cable_connections | source_device_id, dest_device_id | id | physical_devices |
| firmware_inventory | device_id | id | physical_devices |
| firmware_update_ops | device_id | id | physical_devices |
| storage_pools | device_id | id | physical_devices |
| volumes | mounted_on_device_id | id | physical_devices |

### RLS Policy Pattern

Every tenant-scoped table follows the same RLS pattern:

```sql
ALTER TABLE {table_name} ENABLE ROW LEVEL SECURITY;
CREATE POLICY tenant_isolation ON {table_name}
    USING (tenant_id = current_setting('app.current_tenant_id')::UUID);
```

The application sets `app.current_tenant_id` via `SET LOCAL` at the beginning of each database transaction. This is enforced in the repository layer and cannot be bypassed by API callers.

Tables without `tenant_id` (currently only `physical_devices` and `rack_mounts`) are accessed by internal services only and do not have RLS policies. API access to physical device data always goes through `tenant_device_views`.

### Migration Order

These schemas must be migrated in dependency order:

1. `physical_devices` (no dependencies)
2. `sites` (no dependencies)
3. `tenant_device_views` (depends on physical_devices)
4. `racks` (depends on sites)
5. `power_circuits` (depends on sites)
6. `cooling_zones` (depends on sites)
7. `rack_mounts` (depends on racks, physical_devices)
8. `rack_power_feeds` (depends on racks, power_circuits)
9. `cable_connections` (depends on physical_devices)
10. `ip_prefixes` (depends on sites)
11. `ip_addresses` (depends on ip_prefixes)
12. `dhcp_scopes` (depends on ip_prefixes)
13. `ip_reservations` (depends on dhcp_scopes, ip_addresses)
14. `firmware_inventory` (depends on physical_devices)
15. `firmware_policies` (no FK dependencies)
16. `firmware_packages` (no FK dependencies)
17. `firmware_update_ops` (depends on physical_devices, firmware_packages, firmware_policies)
18. `desired_states` (updated schema, migration alters existing table)
19. `merge_review_queue` (depends on tenant_device_views)
20. `storage_pools` (depends on physical_devices) — Phase 6
21. `volumes` (depends on storage_pools) — Phase 6
