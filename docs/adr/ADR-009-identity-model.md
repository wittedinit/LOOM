# ADR-009: DeviceID + ExternalIdentity with Confidence

## Status

Accepted

## Context

Infrastructure devices are identified by many external attributes: hostnames, IP addresses, serial numbers, MAC addresses, SNMP sysName, Redfish service entry URIs. All of these can change. Hostnames get reassigned. IPs change with DHCP. Devices are moved between racks and get new management IPs. Discovery scans from different protocols may find the same device through different identifiers, creating duplicates if identity matching is naive.

## Decision

Every device gets an immutable internal `DeviceID` (UUID). External identifiers are stored as `ExternalIdentity` records with confidence scores, enabling merge and split decisions.

Model:
- **DeviceID**: Immutable UUID assigned at first discovery. Never changes, never reused. All internal references (workflows, events, topology edges) use DeviceID.
- **ExternalIdentity**: A set of (identifier_type, identifier_value, confidence, source, last_seen) records associated with a DeviceID. Examples: (hostname, "switch-01", 0.9, "dns"), (ip, "10.0.1.5", 0.7, "snmp_discovery"), (serial, "FOC12345", 1.0, "redfish").
- **Confidence scoring**: Each external identity has a confidence score. Serial numbers from Redfish get 1.0 (hardware-anchored). Hostnames from DNS get 0.9. IPs from SNMP discovery get 0.7 (may change). Discovery from different protocols can corroborate identities, increasing confidence.
- **Merge/split decisions**: When two DeviceIDs appear to reference the same physical device (matching serial numbers from different discovery sources), a merge is proposed with a confidence score. High confidence merges are automatic; low confidence merges require human approval.

## Consequences

- Devices survive hostname changes, IP reassignment, and rack moves without losing history
- Discovery deduplication is probabilistic rather than brittle exact-matching
- Merge operations preserve full history from both DeviceIDs
- Split operations (when a DeviceID turns out to be two devices sharing an IP) are supported
- External integrations (CMDB, ticketing) can map their IDs as ExternalIdentity records
- More complex than a simple hostname-based identity, but eliminates the chronic duplicate device problem
- Confidence thresholds for automatic merge/split are configurable per tenant
