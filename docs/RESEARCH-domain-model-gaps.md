# Research: Domain Model Gaps

> **Status:** Open — requires design decisions before Phase 1B (schema migration)
> **Addresses:** GAP-ANALYSIS.md findings C3, C6, C8, C9, H2, H4, H7, H20
> **Owner:** TBD
> **Last updated:** 2026-03-21

---

## 1. Problem Statement

The domain model (DOMAIN-MODEL.md) defines the core types for LOOM's data layer. Eight gaps in the model will cause breaking schema changes if not addressed before Phase 1B (first Alembic migration):

1. **Shared device model** — `Device` has single `TenantID`; shared physical devices break multi-tenancy
2. **IPAM** — No IP address allocation, subnet tracking, or DHCP integration
3. **Physical topology** — No rack, PDU, cooling zone, or cable model
4. **Firmware lifecycle** — `FirmwareVersion` recorded but no update model
5. **Identity matcher concurrency** — Race condition RC-06 acknowledged but unsolved
6. **DesiredState comparison** — `map[string]string` vs heterogeneous ObservedFacts
7. **DesiredState conflicts** — No resolution for overlapping declarations
8. **Storage model** — No volume, LUN, pool, or filesystem types

---

## 2. Multi-Tenant Shared Device Model

### 2.1 The Problem

`Device` has a single `TenantID`. In production, every core switch, spine router, and shared storage array serves multiple tenants. Two options were proposed in EDGE-CASES.md RC-04 but no commitment was made.

### 2.2 Analysis of Options

**Option A: Authorized Tenant List**

```go
type Device struct {
    ID              uuid.UUID
    OwnerTenantID   uuid.UUID         // tenant that discovered/owns the device
    AuthorizedTenants []uuid.UUID     // tenants allowed to interact
    // ... existing fields
}
```

| Pro | Con |
|-----|-----|
| Simple schema change | Every query must filter by `OwnerTenantID OR ID IN AuthorizedTenants` — complex SQL |
| Single device record per physical device | RBAC becomes complex (what can non-owner tenants do?) |
| No data duplication | Audit trail for shared operations has cross-tenant visibility implications |

**Option B: Virtual Device Records**

```go
type PhysicalDevice struct {
    ID              uuid.UUID
    // hardware identity, endpoints, capabilities
}

type TenantDeviceView struct {
    ID              uuid.UUID
    TenantID        uuid.UUID
    PhysicalDeviceID uuid.UUID
    // tenant-scoped metadata, desired state, relationships
}
```

| Pro | Con |
|-----|-----|
| Clean tenant isolation — each tenant sees only their view | N records per physical device (query overhead) |
| RBAC is straightforward — tenant owns their view | Discovery must decide: create new physical + view, or link to existing physical |
| Audit trail is tenant-scoped | State changes on the physical device must propagate to all views |

### 2.3 Industry Precedent

| Tool | Approach |
|------|----------|
| NetBox | Single device, multi-tenant via tenant assignment (Option A style) |
| Cisco NSO | Per-tenant device configuration partitions (Option B style) |
| Kubernetes | Namespaced resources with RBAC (Option B style) |

### 2.4 Recommendation

**Option B (Virtual Device Records)** — It maintains clean tenant isolation without complicating every SQL query with multi-tenant OR clauses. The overhead of N records per physical device is acceptable for shared infrastructure (typically <5% of total devices).

### 2.5 Open Questions

- [ ] How does discovery determine if a newly found device matches an existing PhysicalDevice? (Cross-tenant identity matching — security sensitive)
- [ ] Who can create a TenantDeviceView for an existing PhysicalDevice? (Admin only? Self-service with approval?)
- [ ] When PhysicalDevice state changes, how are TenantDeviceViews notified? (Event? Trigger? Polling?)
- [ ] Should TenantDeviceViews have independent DesiredState, or inherit from PhysicalDevice?

---

## 3. IPAM (IP Address Management)

### 3.1 The Problem

LOOM's decomposition examples assign subnets and VLANs but the domain model has no types for IP addresses, subnets, prefixes, DHCP scopes, or address pools. Without IPAM, LOOM cannot prevent IP collisions between tenants or track address utilization.

### 3.2 Scope Decision: Build vs Integrate

| Approach | Effort | Benefit | Risk |
|----------|--------|---------|------|
| **Built-in IPAM** — LOOM manages IP space directly | High (8-12 weeks) | Single source of truth, no external dependency | Reinventing NetBox/Infoblox; scope creep |
| **External IPAM integration** — Adapter for NetBox, Infoblox, phpIPAM | Medium (3-4 weeks per integration) | Leverages existing tools, respects operator workflows | Dependency on external system; sync lag |
| **Hybrid** — Minimal built-in allocator + optional external sync | Medium (4-6 weeks) | Works standalone, integrates when available | Two code paths to maintain |

### 3.3 Minimum Viable IPAM Model

If LOOM builds a minimal IPAM (recommended for standalone deployments):

```go
type IPPrefix struct {
    ID          uuid.UUID
    TenantID    uuid.UUID
    Prefix      netip.Prefix   // e.g., 10.1.0.0/24
    ParentID    *uuid.UUID     // hierarchical prefix tree
    Status      PrefixStatus   // active, reserved, deprecated
    Role        string         // management, production, storage, oob
    VLANID      *int           // optional VLAN association
    SiteID      *uuid.UUID     // optional site/location scoping
}

type IPAddress struct {
    ID          uuid.UUID
    TenantID    uuid.UUID
    Address     netip.Addr     // e.g., 10.1.0.5
    PrefixID    uuid.UUID      // parent prefix
    Status      AddressStatus  // active, reserved, dhcp, slaac
    DeviceID    *uuid.UUID     // assigned device (nullable)
    InterfaceID *uuid.UUID     // assigned interface (nullable)
    DNSName     string         // forward DNS name
}

type DHCPScope struct {
    ID          uuid.UUID
    TenantID    uuid.UUID
    PrefixID    uuid.UUID
    RangeStart  netip.Addr
    RangeEnd    netip.Addr
    LeaseTime   time.Duration
    Options     map[string]string // DHCP options (gateway, DNS, NTP)
}
```

### 3.4 Concurrent Allocation Safety

Two concurrent workflows allocating IPs from the same prefix must not collide:

```go
// Atomic allocation using SELECT FOR UPDATE
func (s *IPAMService) AllocateAddress(ctx context.Context, prefixID uuid.UUID) (netip.Addr, error) {
    tx := s.db.BeginTx(ctx, &sql.TxOptions{Isolation: sql.LevelSerializable})
    // SELECT next available address FROM ip_addresses WHERE prefix_id = ? AND status = 'available' FOR UPDATE SKIP LOCKED
    // UPDATE ip_addresses SET status = 'active', device_id = ? WHERE id = ?
    return addr, tx.Commit()
}
```

### 3.5 Open Questions

- [ ] Should LOOM include a built-in IPAM or require an external one?
- [ ] If built-in, should it support IPv6 from day one?
- [ ] How does IPAM interact with DHCP? (LOOM creates scope → external DHCP server reads it? Or LOOM manages DHCP directly?)
- [ ] Should subnet allocation be part of the LLM decision engine or a deterministic allocator?

---

## 4. Physical Topology Model

### 4.1 The Problem

LOOM claims to make placement decisions factoring in "failure domains" and "availability," but has no model for physical infrastructure topology. Without it, LOOM cannot determine which devices share power, cooling, or network failure domains.

### 4.2 Minimum Viable Physical Topology

```go
type Site struct {
    ID          uuid.UUID
    TenantID    uuid.UUID
    Name        string         // e.g., "us-east-1", "london-dc2"
    Address     string
    Metadata    map[string]string
}

type Rack struct {
    ID          uuid.UUID
    SiteID      uuid.UUID
    Name        string         // e.g., "A3"
    TotalUnits  int            // e.g., 42
    PowerCircuits []PowerCircuitRef // which PDUs feed this rack
    CoolingZone string         // e.g., "zone-a-hot-aisle"
}

type RackMount struct {
    ID          uuid.UUID
    RackID      uuid.UUID
    DeviceID    uuid.UUID
    StartUnit   int            // e.g., 12 (U12)
    Height      int            // e.g., 2 (2U server)
    Position    string         // front, rear
}

type PowerCircuit struct {
    ID          uuid.UUID
    SiteID      uuid.UUID
    Name        string         // e.g., "PDU-A3-L1"
    Feed        string         // A, B (redundant feeds)
    MaxAmps     float64
    Voltage     int            // 120, 208, 240
}

type CableConnection struct {
    ID          uuid.UUID
    SourceDeviceID    uuid.UUID
    SourcePort        string    // e.g., "Ethernet1/1"
    DestDeviceID      uuid.UUID
    DestPort          string    // e.g., "Gi1/0/1"
    MediaType         string    // fiber-sm, fiber-mm, cat6, dac
    CableID           string    // physical label
}
```

### 4.3 Failure Domain Queries

With this model, LOOM can answer:
- "Which servers share a power circuit with server X?" → `JOIN rack_mounts ON rack → JOIN power_circuits`
- "Which servers are on the same TOR switch?" → `JOIN cable_connections WHERE dest_device is TOR`
- "Place replicas in different failure domains" → ensure no shared rack, power circuit, or TOR switch
- "What is the blast radius of TOR switch failure?" → all devices with cable_connections to that switch

### 4.4 Integration vs Built-In

| Source | Data Available | Integration Effort |
|--------|---------------|-------------------|
| NetBox API | Site, rack, device, cable, power | Medium (REST API adapter) |
| Device42 API | Similar to NetBox | Medium |
| LLDP/CDP discovery | Cable connections (auto-discovered) | Low (adapter capability) |
| IPMI/Redfish | Power state, sensors (not physical location) | Already planned |
| Manual input | Rack position, power circuit | Low (API + UI) |

### 4.5 Recommendation

- Phase 1-2: Add Site + Rack + RackMount to domain model (manual input via API)
- Phase 3: Add CableConnection (auto-populated from LLDP/CDP discovery)
- Phase 5+: Optional NetBox integration adapter for full DCIM sync
- Physical topology should be optional — LOOM must work without it (cloud-only deployments have no racks)

### 4.6 Open Questions

- [ ] Should physical topology be required or optional for placement decisions?
- [ ] If optional, what does the placement engine use when topology is missing? (Cloud provider AZ? Nothing?)
- [ ] Should LOOM auto-discover cable topology via LLDP/CDP, or is manual input sufficient?

---

## 5. Firmware Lifecycle Management

### 5.1 The Problem

`Device.FirmwareVersion` is recorded as a discovered fact, but no operations, workflows, or compliance policies address firmware updates.

### 5.2 Minimum Viable Firmware Model

```go
type FirmwareInventory struct {
    DeviceID        uuid.UUID
    Component       string         // bios, bmc, nic, disk, os
    CurrentVersion  string
    AvailableVersion *string       // nil if up-to-date
    Compliance      ComplianceStatus // compliant, update_available, critical_cve, eol
    LastChecked     time.Time
}

type FirmwarePolicy struct {
    ID              uuid.UUID
    TenantID        uuid.UUID
    Name            string
    DeviceSelector  DeviceSelector  // match by vendor, model, site, tag
    Component       string          // bios, bmc, nic, etc.
    MinVersion      string          // minimum acceptable version
    MaxVersion      *string         // optional ceiling (avoid known-bad versions)
    AutoApply       bool            // auto-stage updates
    MaintenanceWindow *CronExpr    // when updates can be applied
}

type FirmwareUpdateOp struct {
    DeviceID        uuid.UUID
    Component       string
    TargetVersion   string
    PackageURL      string          // firmware binary location
    PreChecks       []VerificationCheck  // pre-update health checks
    PostChecks      []VerificationCheck  // post-update verification
    RollbackVersion string          // version to revert to on failure
}
```

### 5.3 Firmware Update Workflow

```
1. Compliance scan (periodic): compare FirmwareInventory against FirmwarePolicy
2. Stage: download firmware package, verify signature
3. Pre-check: verify device health, take config snapshot
4. Apply: execute firmware update via Redfish UpdateService (or vendor-specific)
5. Reboot: if required (BIOS, BMC) — monitor power cycle
6. Post-check: verify new version, verify device health, verify config intact
7. Report: update FirmwareInventory, generate compliance report
```

### 5.4 Vendor-Specific Concerns

| Vendor | Update Mechanism | Reboot Required? | Rollback Possible? |
|--------|-----------------|-------------------|-------------------|
| Dell iDRAC | Redfish UpdateService + DUP | BMC: no, BIOS: yes | BMC: yes (rollback image), BIOS: vendor-dependent |
| HPE iLO | Redfish UpdateService + .bin/.fwpkg | BMC: no, BIOS: yes | Yes (auto-rollback on failure) |
| Supermicro | Vendor-specific (not standard Redfish) | BMC: yes (some models), BIOS: yes | Limited |
| Network switches | Vendor CLI / API | Depends (ISSU vs full reload) | Dual-image boot (most vendors) |

### 5.5 Open Questions

- [ ] Should firmware compliance scanning be a built-in capability or a separate tool integration?
- [ ] Where are firmware packages stored? (Local artifact server? Vendor download portals?)
- [ ] How are firmware compatibility matrices maintained? (Manual? Vendor API? Community database?)
- [ ] Should firmware updates be handled by Temporal workflows with saga rollback?
- [ ] How does air-gapped firmware update work? (USB delivery? Local mirror?)

---

## 6. Identity Matcher Concurrency

### 6.1 The Problem

EDGE-CASES.md RC-06 documents that concurrent discovery can create duplicate device records. The recommended fix (distributed lock on identity values) is difficult because two discoveries of the same device may produce disjoint identity sets.

### 6.2 Proposed Solution: Two-Phase Matching

```
Phase 1: Optimistic matching (fast path)
  - Discovery result arrives with identity attributes
  - Query existing devices for matching attributes (tenant-scoped)
  - If unique match found: merge (append facts, update confidence)
  - If no match found: create new device (with INSERT ... ON CONFLICT)
  - If ambiguous match: queue for Phase 2

Phase 2: Reconciliation sweep (background, periodic)
  - Run every 5 minutes (configurable)
  - Query all devices created in last interval
  - Cross-reference identity attributes for potential duplicates
  - Generate merge candidates with confidence scores
  - Auto-merge above threshold (e.g., confidence > 0.9)
  - Flag for human review below threshold
```

### 6.3 PostgreSQL Serialization

```sql
-- Phase 1: Use advisory locks on normalized identity values
SELECT pg_advisory_xact_lock(hashtext(identity_value))
FROM unnest(ARRAY['serial:ABC123', 'mac:00:11:22:33:44:55']) AS identity_value;

-- Then SELECT FOR UPDATE on candidate devices
-- This serializes within a tenant without a global lock
```

### 6.4 Open Questions

- [ ] What is the acceptable duplicate rate for the optimistic path? (1%? 5%?)
- [ ] Should the reconciliation sweep auto-merge or always require human approval?
- [ ] How does cross-tenant identity isolation work? (Same serial number in two tenants = two devices, never merged)

---

## 7. DesiredState Comparison Semantics

### 7.1 The Problem

`DesiredState.Properties` is `map[string]string`. `ObservedFact.Value` is a JSON-encoded string. The verification pipeline must compare them, but no specification defines how.

### 7.2 Proposed Property Comparison Specification

```go
type PropertyComparison struct {
    // How to compare this property
    Mode        CompareMode    // exact, numeric_tolerance, contains, regex, json_path

    // For numeric_tolerance mode
    Tolerance   float64        // e.g., 2.0 (degrees, percent, etc.)

    // For json_path mode
    JSONPath    string         // path into ObservedFact JSON value

    // Normalization
    CaseInsensitive bool
    TrimWhitespace  bool
}

type CompareMode int
const (
    CompareExact            CompareMode = iota // string equality after normalization
    CompareNumericTolerance                     // parse as float64, compare within tolerance
    CompareContains                             // observed value contains desired value
    CompareRegex                                // desired value is a regex pattern
    CompareJSONPath                             // extract value from JSON using path, then compare
)
```

### 7.3 Mapping Registry

```go
// Maps DesiredState property names to ObservedFact types and comparison rules
var PropertyMappings = map[string]PropertyMapping{
    "power_state": {
        FactType:    "power.state",
        Comparison:  PropertyComparison{Mode: CompareExact, CaseInsensitive: true},
    },
    "mtu": {
        FactType:    "interface.mtu",
        Comparison:  PropertyComparison{Mode: CompareNumericTolerance, Tolerance: 0},
    },
    "cpu_temp": {
        FactType:    "sensor.cpu_temperature",
        Comparison:  PropertyComparison{Mode: CompareNumericTolerance, Tolerance: 5.0},
    },
    "vlan_membership": {
        FactType:    "interface.vlans",
        Comparison:  PropertyComparison{Mode: CompareJSONPath, JSONPath: "$.vlan_ids"},
    },
}
```

### 7.4 Open Questions

- [ ] Should comparison rules be part of the domain model (stored in DB) or code-defined?
- [ ] Should `DesiredState.Properties` become `map[string]any` with a schema per resource type?
- [ ] How are vendor-specific property names normalized? (e.g., Cisco "mtu" vs Juniper "mtu" vs SNMP ifMtu)

---

## 8. DesiredState Conflict Resolution

### 8.1 The Problem

Two workflows can declare conflicting DesiredState for the same device property. No resolution mechanism exists.

### 8.2 Proposed Resolution: Priority + Last-Write-Wins

```go
type DesiredState struct {
    ID          uuid.UUID
    TenantID    uuid.UUID
    DeviceID    uuid.UUID
    Properties  map[string]string
    Priority    int            // higher priority wins on conflict
    SourceType  string         // workflow, policy, manual
    SourceID    uuid.UUID      // workflow ID, policy ID, or user ID
    ActiveUntil *time.Time     // auto-expire (nil = permanent)
    Version     int            // monotonically increasing
    CreatedAt   time.Time
}
```

**Resolution rules:**
1. Higher priority wins (policy > workflow > manual)
2. Within same priority: latest version wins
3. Expired DesiredState (`ActiveUntil < now()`) is automatically ignored
4. Completed/cancelled workflow DesiredState is retired (marked inactive)
5. Conflicts are logged as events for visibility

### 8.3 Open Questions

- [ ] Should conflicting DesiredState declarations prevent the second workflow from starting?
- [ ] Should the drift detector consider all active DesiredState or only the "winning" one?
- [ ] How are policy-based DesiredState (e.g., "all servers must have NTP configured") distinguished from workflow-based?

---

## 9. Storage Model

### 9.1 The Problem

Five storage adapters are listed but no domain model types exist for storage resources.

### 9.2 Minimum Viable Storage Model

```go
type StoragePool struct {
    ID          uuid.UUID
    TenantID    uuid.UUID
    DeviceID    uuid.UUID       // storage controller/array
    Name        string
    Type        StorageType     // block, file, object
    TotalBytes  int64
    UsedBytes   int64
    Tier        string          // ssd, nvme, hdd, hybrid
}

type Volume struct {
    ID          uuid.UUID
    TenantID    uuid.UUID
    PoolID      uuid.UUID
    Name        string
    SizeBytes   int64
    Protocol    string          // iscsi, fc, nfs, smb, s3
    MountedOn   *uuid.UUID      // device ID where mounted
    MountPath   string          // /mnt/data
    RPO         *time.Duration  // recovery point objective
    RTO         *time.Duration  // recovery time objective
}
```

### 9.3 Scope Recommendation

- Phase 1-5: Do not build storage adapters. Focus on compute + network.
- Phase 6+: Add storage model types to domain model.
- Integration strategy: delegate to CSI/Trident for Kubernetes storage, observe via adapter.
- Direct storage array management (NetApp, Pure, Ceph) deferred to Phase 8+.

### 9.4 Open Questions

- [ ] Is storage orchestration in scope for MVP, or should it be deferred entirely?
- [ ] Should LOOM manage storage directly or observe/verify what external tools provision?
- [ ] How do storage operations differ from compute in terms of compensation? (Volume delete = data loss, no rollback)

---

## 10. Implementation Priority

| Item | Phase | Effort | Dependencies |
|------|-------|--------|--------------|
| Shared device model (Option B: virtual records) | 1A | 2-3 weeks | Domain model finalized |
| DesiredState comparison specification | 1B | 2 weeks | Verification model |
| DesiredState conflict resolution | 1B | 1-2 weeks | DesiredState model |
| Identity matcher two-phase approach | 1B | 2-3 weeks | PostgreSQL advisory locks |
| IPAM minimum viable model | 1B | 3-4 weeks | Network adapter requirements |
| Physical topology (Site + Rack) | 2 | 2-3 weeks | UI for manual input |
| Firmware lifecycle model | 3 | 3-4 weeks | Redfish adapter |
| Storage model types (schema only) | 6 | 1-2 weeks | Storage scope decision |

---

## 11. References

- DOMAIN-MODEL.md — Current domain model types
- EDGE-CASES.md — RC-04 (shared devices), RC-06 (identity concurrency)
- VERIFICATION-MODEL.md — StateMatchCheck semantics
- IDENTITY-MODEL.md — Merge/split/enrichment rules
- research/11-storage-orchestration.md — Storage landscape
- research/12-dns-dhcp-automation.md — IPAM gap in competitive tools
