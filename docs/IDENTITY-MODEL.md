# LOOM Identity Model

> **This is the most dangerous seam in the project.** If identity is wrong, discovery creates duplicates, audit history becomes messy, and merge logic becomes guesswork. Every decision in this document exists to prevent silent data corruption.

---

## 1. Core Identity Concepts

### DeviceID — LOOM's Internal Identity

- UUID v7 (time-sortable)
- Immutable once created
- Never changes even if hostname, IP, or serial number changes
- This is the canonical reference for all relationships, facts, workflows, and audit

DeviceID is the only stable anchor. Everything else is an observation that may change over time.

### ExternalIdentity — Identifiers from the Outside World

```go
type IdentityType string

const (
	IdentitySerialNumber  IdentityType = "serial_number"
	IdentityMACAddress    IdentityType = "mac_address"
	IdentityBMCIP         IdentityType = "bmc_ip"
	IdentityManagementIP  IdentityType = "management_ip"
	IdentityHostname      IdentityType = "hostname"
	IdentityFQDN          IdentityType = "fqdn"
	IdentityAssetTag      IdentityType = "asset_tag"
	IdentityBIOSUUID      IdentityType = "uuid_bios"
)

type ExternalIdentity struct {
	Type        IdentityType // serial_number, mac_address, bmc_ip, management_ip, hostname, fqdn, asset_tag, uuid_bios
	Value       string
	Source      string  // which adapter/discovery reported this
	Confidence  float64 // 0.0 to 1.0
	FirstSeenAt time.Time
	LastSeenAt  time.Time
	SupersededAt *time.Time // non-nil if this identity has been replaced; never deleted
}
```

A device has a **set** of external identities. Multiple identities can point to the same device. Identities are append-only — superseded entries are timestamped, never removed.

---

## 2. Identity Type Priority Order

Precedence for identity matching, highest to lowest confidence:

| Priority | Identity Type              | Default Confidence | Rationale                                      |
|----------|----------------------------|--------------------|-------------------------------------------------|
| 1        | Serial Number              | 0.95               | Most stable, rarely changes                     |
| 2        | BIOS UUID                  | 0.90               | Embedded in hardware, stable                    |
| 3        | MAC Address (BMC/mgmt)     | 0.85               | Stable but can be spoofed                       |
| 4        | BMC IP                     | 0.70               | Relatively stable but can change with DHCP      |
| 5        | Asset Tag                  | 0.65               | Manually assigned, may be wrong                 |
| 6        | Management IP              | 0.50               | Changes with DHCP, readdressing                 |
| 7        | Hostname                   | 0.40               | Frequently changes                              |
| 8        | FQDN                       | 0.40               | Changes with DNS updates                        |

The matcher walks this list top-to-bottom. A match on a higher-priority identity dominates any conflict on a lower-priority one.

---

## 3. Merge Rules

When two discoveries match on a high-confidence identity:

- **Serial numbers match** — definitely the same device — merge.
- **BMC MAC matches** — very likely the same device — merge.
- **BMC IP matches AND vendor+model match** — likely the same — merge.
- **Only hostname matches** — not confident enough — keep separate, flag for review.

### Merge Procedure

1. **Keep the older DeviceID** (stable reference).
2. Attach the new endpoint to the existing device.
3. Add new observed facts with the new endpoint as source.
4. Union external identities (keep all, update confidence and `LastSeenAt`).
5. Publish `device.enriched` event (not `device.created`).

```
merge(existing Device, incoming DiscoveryResult):
    existing.Endpoints = union(existing.Endpoints, incoming.Endpoints)
    existing.ExternalIdentities = unionIdentities(existing.ExternalIdentities, incoming.ExternalIdentities)
    existing.Facts = appendFacts(existing.Facts, incoming.Facts)
    publish("device.enriched", existing.DeviceID)
```

---

## 4. Split Rules

When identity conflict is detected:

- If a serial number appears on two different BMC IPs that were previously separate devices — investigate.
- If a device's serial number changes — the original device was likely replaced.

In conflict cases:

1. Create a new device with a new DeviceID.
2. Mark the old device with `status: decommissioned`.
3. Flag for human review.
4. **Never silently overwrite identity** — always create an audit trail.

```
split(existing Device, conflicting DiscoveryResult):
    existing.Status = "decommissioned"
    existing.DecommissionedAt = now()
    newDevice = createDevice(conflicting)
    publish("device.conflict_detected", existing.DeviceID, newDevice.DeviceID)
    flagForReview(existing.DeviceID, newDevice.DeviceID, reason="identity_conflict")
```

---

## 5. Enrichment Rules

When a new protocol discovers facts about an already-known device:

1. Match using the identity priority order (Section 2).
2. **If match found** — enrich:
   - Attach the new endpoint (SSH, SNMP, etc.) to the existing device.
   - Add new observed facts with provenance (which endpoint, which adapter).
   - Update external identities (add new ones, refresh `LastSeenAt` on existing).
   - **Do NOT create a new device.**
3. **If no match found** — create a new device.

Enrichment is the common case. Most discovery cycles enrich existing devices rather than creating new ones. The system should be optimized for this path.

---

## 6. Mutation Handling

When identity attributes change between discovery cycles:

| Change                 | Action                                                                                       |
|------------------------|----------------------------------------------------------------------------------------------|
| **Hostname change**    | Update the hostname external identity. Keep DeviceID stable. Log the change in audit.        |
| **Management IP change** | Update the IP external identity. Keep DeviceID stable. Update endpoint address.            |
| **BMC IP change**      | Same as management IP. Update endpoint.                                                      |
| **Serial number change** | Hardware was replaced. Decommission old device, create new device. Flag for review.        |

The key distinction: **logical changes** (IP, hostname) mutate the identity record. **Physical changes** (serial number) mean the device itself is different and require a split.

---

## 7. Tombstoning

- Superseded external identities are marked with a `SupersededAt` timestamp — **never deleted**.
- This allows auditing the full identity history of a device.
- A device with status `decommissioned` retains all its identities and facts for audit purposes.

```go
// MarkSuperseded timestamps an identity without removing it.
// The identity remains queryable for audit but is excluded from active matching.
func (id *ExternalIdentity) MarkSuperseded(at time.Time) {
	id.SupersededAt = &at
}
```

The identity store must support querying both active and superseded identities. Active matching ignores superseded entries. Audit queries include them.

---

## 8. Confidence Scoring

Each external identity carries a confidence weight. When multiple independent sources agree (e.g., SSH discovery and Redfish discovery both report the same serial number), confidence increases.

```go
// CalculateMatchConfidence computes how likely it is that a set of candidate
// identities refers to the same device as a set of existing identities.
//
// Returns 0.0 (no match) to 1.0 (certain match).
// Matching high-confidence identities (serial, BIOS UUID) → high score.
// Matching low-confidence identities only (hostname) → low score.
// Conflicting high-confidence identities → negative signal (split, not merge).
func CalculateMatchConfidence(existing []ExternalIdentity, candidate []ExternalIdentity) float64
```

Scoring rules:

- **Matching high-confidence identity** (serial, BIOS UUID): score += identity's base confidence.
- **Matching low-confidence identity** (hostname, IP): score += identity's base confidence * 0.5.
- **Multiple agreeing identities**: corroboration bonus (score += 0.1 per additional match).
- **Conflicting high-confidence identity** (e.g., different serial numbers): returns -1.0 (signal to split, not merge).
- Final score is clamped to [0.0, 1.0] (except -1.0 for conflicts).

---

## 9. Identity Matcher Concurrency (H2)

Concurrent discovery from multiple sources (e.g., SSH and Redfish scanning the same subnet simultaneously) can create duplicate device records if two goroutines both conclude "no match found" and each creates a new device. This section defines the two-phase matching protocol that produces deterministic results under concurrency.

### Two-Phase Matching

**Phase 1 — Read-Only Scoring (fast path, no locks)**

1. Discovery result arrives with a set of `ExternalIdentity` values.
2. Query existing devices for matching attributes (tenant-scoped, using the priority order from Section 2).
3. Compute `CalculateMatchConfidence` for each candidate.
4. If a unique match is found with confidence > 0.7: proceed to Phase 2 merge.
5. If no match is found: proceed to Phase 2 create.
6. If ambiguous (multiple candidates): queue for background reconciliation.

Phase 1 performs no writes. It is safe to run concurrently without locks.

**Phase 2 — Lock + Merge/Create (serialized per identity)**

1. Acquire PostgreSQL advisory locks on normalized identity values:
   ```sql
   SELECT pg_advisory_xact_lock(hashtext(identity_value))
   FROM unnest(ARRAY['serial:ABC123', 'mac:00:11:22:33:44:55']) AS identity_value;
   ```
2. Re-run the match query under `SELECT ... FOR UPDATE` on candidate device rows.
3. If the match result is unchanged from Phase 1: execute merge or create.
4. If the match result changed (another goroutine merged first): re-evaluate and enrich instead of creating a duplicate.
5. Commit the transaction. Advisory locks are released automatically.

This ensures that concurrent discovery of the same physical device from multiple sources produces exactly one device record, regardless of timing.

### Guarantees

- **No duplicate devices from concurrent discovery.** Advisory locks on identity values serialize the create-or-merge decision.
- **No global lock.** Locking is scoped to specific identity values, so discoveries of unrelated devices proceed in parallel.
- **Deterministic results.** The same set of discovery results, regardless of arrival order, produces the same final device state.
- **Tenant isolation preserved.** Advisory locks include the tenant scope — the same serial number in two different tenants produces two separate devices, never merged.

---

## 10. Discovery Service Identity Flow

Step-by-step flow for every discovery result:

```
1. Discovery adapter produces []ExternalIdentity + []ObservedFact
                              │
2. Identity matcher searches existing devices by each
   external identity (highest confidence first)
                              │
              ┌───────────────┼───────────────────┐
              │               │                   │
3. Match found,        4. Match found,      5. No match
   confidence > 0.7       confidence 0.4–0.7      │
              │               │                   │
   Enrich existing      Merge but flag       Create new
   device                for review           device
              │               │                   │
              └───────┬───────┘                   │
                      │                           │
6. Conflicting match detected?                    │
   (same serial on different device)              │
              │                                   │
   YES → Split: decommission old,                 │
         create new, flag for review              │
              │                                   │
              └───────────────┬───────────────────┘
                              │
7. Publish event:
   - device.created
   - device.enriched
   - device.conflict_detected
```

### Decision Thresholds

| Confidence Range | Action                        |
|------------------|-------------------------------|
| > 0.7            | Merge automatically           |
| 0.4 – 0.7       | Merge but flag for review     |
| < 0.4            | No match — create new device  |
| -1.0 (conflict)  | Split — flag for review       |

---

## 11. Example Scenarios

### Scenario 1: Same Server Discovered via SSH and Redfish

1. SSH discovers hostname `web-01`, management IP `10.0.0.15`.
2. Redfish discovers serial `ABC123`, BMC IP `10.0.0.115`.
3. No direct identity match — different IPs, no shared high-confidence ID.
4. **But**: if SSH also discovers serial `ABC123` from `dmidecode` — serial matches — **merge**.
5. Result: single device with DeviceID, two endpoints (SSH + Redfish), enriched facts from both.

### Scenario 2: DHCP IP Change

1. Device was at `10.0.0.15` — DHCP assigns `10.0.0.20`.
2. Next discovery cycle: serial number matches — same device.
3. Update `management_ip` external identity. Mark old IP as superseded. Keep DeviceID stable.
4. Publish `device.enriched`.

### Scenario 3: Hardware Replacement

1. Server in rack slot 5 was serial `ABC123` — replaced with serial `DEF456`.
2. Redfish discovers new serial on the same BMC IP.
3. Serial mismatch on a high-confidence identity = hardware replacement.
4. Decommission old device (serial `ABC123`), create new device (serial `DEF456`).
5. Flag for human review. Publish `device.conflict_detected`.
