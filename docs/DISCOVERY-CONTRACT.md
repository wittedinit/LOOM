# Discovery Contract

> Behavioral contract for device discovery in LOOM. Discovery is the process of probing a target address to determine what lives there, what protocols it speaks, and how to manage it.

---

## Core Principles

1. **Discovery is tenant-scoped.** Every discovery request, result, and cache entry belongs to exactly one tenant. Cross-tenant discovery is impossible by design.
2. **Discovery does not mutate.** Discovery reads from targets. It never writes, configures, or changes device state. Any adapter used for discovery must be invoked in read-only mode.
3. **Discovery produces identity candidates.** The discovery pipeline feeds into the identity model (see [IDENTITY-MODEL.md](IDENTITY-MODEL.md)) for merge/split decisions. Discovery does not decide identity -- it provides evidence.
4. **Every protocol gets a fair shot, in order.** The probe sequence is deterministic. Protocols are tried in preference order. The first successful probe does not skip remaining probes -- all reachable protocols are recorded.

---

## Protocol Preference Order

Protocols are probed in this order. The order reflects richness of data, reliability, and security of the protocol.

| Priority | Protocol | Default Port | Rationale |
|----------|----------|-------------|-----------|
| 1 | gNMI | 9339 | Richest structured data, streaming capable, TLS by default |
| 2 | Redfish | 443 | REST-based, strong schema, standard for server BMC management |
| 3 | NETCONF | 830 | XML-modeled config, transaction support, standard for network devices |
| 4 | SNMP | 161 | Universal but limited, read-only discovery, wide device support |
| 5 | SSH/CLI | 22 | Fallback for devices without structured APIs, screen-scraping |
| 6 | IPMI | 623 | Legacy BMC protocol, limited data but common on older servers |
| 7 | PiKVM | 443 | KVM-over-IP devices, niche but important for out-of-band access |

**All protocols are probed.** A successful gNMI probe does not skip Redfish. The full set of reachable protocols is recorded on the device's endpoint list. Protocol preference determines which adapter is used as the *primary* management channel, not which protocols are attempted.

---

## Types

```go
package discovery

import (
    "net"
    "time"
)

// ProbeMethod identifies a protocol used during discovery probing.
type ProbeMethod string

const (
    ProbeGNMI    ProbeMethod = "gnmi"
    ProbeRedfish ProbeMethod = "redfish"
    ProbeNETCONF ProbeMethod = "netconf"
    ProbeSNMP    ProbeMethod = "snmp"
    ProbeSSH     ProbeMethod = "ssh"
    ProbeIPMI    ProbeMethod = "ipmi"
    ProbePiKVM   ProbeMethod = "pikvm"
)

// ProbeMethodOrder is the canonical probe sequence.
var ProbeMethodOrder = []ProbeMethod{
    ProbeGNMI, ProbeRedfish, ProbeNETCONF, ProbeSNMP,
    ProbeSSH, ProbeIPMI, ProbePiKVM,
}

// DiscoveryRequest initiates discovery of one or more targets.
type DiscoveryRequest struct {
    ID          string      `json:"id"`           // UUID v7
    TenantID    string      `json:"tenant_id"`
    Targets     []Target    `json:"targets"`       // explicit targets (IPs, hostnames)
    Subnets     []string    `json:"subnets"`       // CIDR ranges for batch scanning
    Methods     []ProbeMethod `json:"methods"`     // override probe methods (empty = all, in order)
    Credentials []CredentialHint `json:"credentials"` // credential refs to try per protocol
    Force       bool        `json:"force"`         // ignore cache, re-probe everything
    RequestedBy string      `json:"requested_by"`  // actor identity
    SubmittedAt time.Time   `json:"submitted_at"`
}

// Target is a single address to probe.
type Target struct {
    Address  string `json:"address"`  // IP, hostname, or CIDR
    Port     int    `json:"port"`     // override default port (0 = use protocol default)
    Metadata map[string]string `json:"metadata"` // hints: "vendor", "expected_type", etc.
}

// CredentialHint maps a protocol to credential refs to try during probing.
type CredentialHint struct {
    Protocol      ProbeMethod `json:"protocol"`
    CredentialRef string      `json:"credential_ref"` // vault path
}

// ProbeResult records the outcome of probing a single target with a single protocol.
type ProbeResult struct {
    Method      ProbeMethod        `json:"method"`
    Reachable   bool               `json:"reachable"`
    Port        int                `json:"port"`
    Transport   string             `json:"transport"`    // "tcp", "tls", "udp"
    Identities  []ExternalIdentity `json:"identities"`   // identities discovered via this protocol
    Facts       []ObservedFact     `json:"facts"`         // facts gathered via this protocol
    Latency     time.Duration      `json:"latency"`       // probe round-trip time
    Error       string             `json:"error"`         // empty if reachable
    Attempts    int                `json:"attempts"`      // how many attempts were made
    ProbedAt    time.Time          `json:"probed_at"`
}

// DiscoveryResult is the complete outcome of discovering a single target.
type DiscoveryResult struct {
    ID            string        `json:"id"`              // UUID v7
    TenantID      string        `json:"tenant_id"`
    Address       string        `json:"address"`         // the target address that was probed
    Probes        []ProbeResult `json:"probes"`          // one per protocol attempted
    BestMethod    ProbeMethod   `json:"best_method"`     // highest-priority reachable protocol
    DeviceType    string        `json:"device_type"`     // classified device type (may be "unknown")
    Merged        bool          `json:"merged"`          // true if merged into existing device
    DeviceID      string        `json:"device_id"`       // LOOM DeviceID (new or existing)
    DiscoveredAt  time.Time     `json:"discovered_at"`
    CachedUntil   time.Time     `json:"cached_until"`
    CorrelationID string        `json:"correlation_id"`
}
```

---

## Timeout Policy

| Parameter | Value | Rationale |
|-----------|-------|-----------|
| Per-protocol probe timeout | 5 seconds | Long enough for slow BMCs, short enough to not stall the pipeline |
| Total per-target timeout | 30 seconds | 7 protocols x ~4s average with early exits for unreachable ports |
| Connection timeout | 3 seconds | TCP SYN timeout before declaring unreachable |
| Authentication timeout | 5 seconds | Time for credential validation after connection |
| Batch scan total timeout | 10 minutes | Upper bound for a /24 subnet scan |

If a probe exceeds its timeout, it is recorded as unreachable with a timeout error. The next protocol in the sequence begins immediately.

---

## Retry Policy

Each protocol probe is retried on transient failures (connection reset, temporary auth failure, timeout).

| Parameter | Value |
|-----------|-------|
| Max attempts per protocol | 3 |
| Backoff strategy | Exponential with jitter |
| Initial backoff | 500ms |
| Max backoff | 4s |
| Retryable errors | Connection reset, timeout, 503 Service Unavailable, rate limit |
| Non-retryable errors | Auth rejected (401/403), protocol not supported, connection refused |

A connection refused on the protocol's default port is not retried -- the protocol is not available on this target.

---

## Discovery Caching

Discovery results are cached to avoid re-probing devices that have already been successfully discovered.

| Parameter | Value |
|-----------|-------|
| Cache TTL | 24 hours |
| Cache key | `{tenant_id}:{address}:{protocol}` |
| Cache store | Valkey (Redis-compatible) |
| Refresh triggers | On-demand (`Force: true`), scheduled re-scan, NATS event |
| Eviction | TTL-based; no LRU eviction |

**Cache behavior:**

1. Before probing, check cache for `{tenant_id}:{address}`. If a valid (non-expired) result exists and `Force` is false, return the cached result.
2. On successful probe, write the result to cache with a 24h TTL.
3. On device removal event (`loom.{tenant}.device.deleted`), evict all cache entries for that device's addresses.
4. Scheduled re-scans bypass cache (`Force: true`) and overwrite existing entries.

---

## Batch Discovery (Subnet Scanning)

When `Subnets` is provided in the `DiscoveryRequest`, LOOM expands CIDR ranges into individual target addresses and probes them concurrently.

| Parameter | Value |
|-----------|-------|
| Max concurrent goroutines per scan | 50 |
| Max subnet size | /16 (65,534 hosts) |
| Rate limiting | 100 probes/second per tenant |
| Progress reporting | Via NATS events every 10% completion |

**Subnet scan procedure:**

1. Expand CIDR to individual IPs. Exclude network and broadcast addresses.
2. Check cache for each IP. Skip cached (non-expired, non-forced) targets.
3. Launch probe goroutines via a semaphore-bounded worker pool (max 50 concurrent).
4. Each goroutine runs the full protocol probe sequence for its target.
5. Results are streamed to NATS as they complete -- callers do not wait for the entire scan.
6. On completion, emit a summary event with counts: discovered, updated, unreachable, cached.

**Subnet size limits:**

- `/24` (254 hosts): Completes in ~30 seconds with 50 concurrent probes.
- `/20` (4,094 hosts): Completes in ~5 minutes.
- `/16` (65,534 hosts): Completes in ~60 minutes. Requires explicit confirmation.
- Larger than `/16`: Rejected. Break into smaller ranges.

---

## Discovery Events

All discovery activity emits events to NATS JetStream. See [EVENT-MODEL.md](EVENT-MODEL.md) for the envelope format.

| Subject | Trigger |
|---------|---------|
| `loom.{tenant}.discovery.started` | Discovery request accepted and probing begins |
| `loom.{tenant}.discovery.completed` | All targets in a request have been probed |
| `loom.{tenant}.discovery.progress` | Batch scan progress update (every 10%) |
| `loom.{tenant}.device.discovered` | A new device was found (no prior DeviceID) |
| `loom.{tenant}.device.updated` | An existing device was re-probed with new/changed facts |
| `loom.{tenant}.device.removed` | A previously known device is no longer reachable after 3 consecutive scan failures |
| `loom.{tenant}.discovery.conflict_detected` | Probe results conflict with stored identity (see merge rules) |

**Removal policy:** A device is not marked as removed on the first unreachable scan. It takes 3 consecutive scan failures (across separate scan runs, not retries within a single probe) before a removal event is emitted. The device transitions to `unreachable` status after the first failure and to `decommissioned` only after manual confirmation following the third failure.

---

## Identity Merge on Discovery

When discovery finds identities (serial numbers, MAC addresses, etc.), the results are fed into the identity matcher defined in [IDENTITY-MODEL.md](IDENTITY-MODEL.md).

**Merge flow:**

1. Discovery completes for a target, producing a set of `ExternalIdentity` values.
2. The identity matcher searches for existing devices with overlapping identities, scoped to the tenant.
3. If a match is found above the confidence threshold (see Identity Type Priority Order in IDENTITY-MODEL.md):
   - The existing `DeviceID` is reused.
   - New endpoints and facts are attached to the existing device.
   - The `DiscoveryResult.Merged` flag is set to `true`.
4. If no match is found:
   - A new `DeviceID` (UUID v7) is created.
   - The device is created with status `discovered`.
5. If a conflict is detected (e.g., serial number matches device A but MAC matches device B):
   - A `discovery.conflict_detected` event is emitted.
   - The discovery result is stored but not merged. Human review is required.

**Split detection:** If a device that was previously one entity now presents with different serial numbers on different endpoints (e.g., a chassis was replaced but the BMC IP stayed), a conflict event is emitted. LOOM never auto-splits a device -- splits require human confirmation.

---

## Tenant Scoping

Discovery is always tenant-scoped. This is enforced at multiple levels:

1. **Request validation:** Every `DiscoveryRequest` must carry a non-empty `TenantID`. Requests without a tenant are rejected.
2. **Cache isolation:** Cache keys are prefixed with `tenant_id`. Tenant A cannot see tenant B's cached results.
3. **Identity matching:** The identity matcher only searches within the requesting tenant's device inventory. A serial number match in tenant B's inventory is invisible to tenant A's discovery.
4. **Event isolation:** Discovery events are published to tenant-scoped NATS subjects (`loom.{tenant}.discovery.*`). NATS authorization prevents cross-tenant subscription.
5. **Credential isolation:** Credential hints reference vault paths that are tenant-scoped. A discovery probe for tenant A cannot use tenant B's credentials.
