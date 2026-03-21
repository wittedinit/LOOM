# Device Locking

> Addresses ADVERSARIAL-REVIEW.md Finding 6.3: Two tenants' workflows could try to configure the same physical switch port simultaneously. Without per-device locking, concurrent workflows from different tenants produce race conditions on shared physical infrastructure.

## Problem

Physical devices are shared across tenants. A network switch carries VLANs for multiple tenants. A hypervisor hosts VMs from multiple tenants. The adapter's `IdempotencyKey` prevents duplicate operations within a single workflow, but does NOT prevent cross-tenant conflicts on the same device.

**Example:** Tenant A's workflow is reconfiguring SW-01's trunk ports while Tenant B's workflow is creating a VLAN on SW-01. Both workflows issue concurrent mutations to the same physical device. The result is unpredictable.

---

## Locking Model

LOOM uses PostgreSQL advisory locks for distributed device-level locking. Advisory locks are chosen because:

1. LOOM already depends on PostgreSQL — no new infrastructure.
2. Advisory locks survive connection pooling (session-level locks) or can be transaction-scoped.
3. Advisory locks do not block regular table operations — they only block other advisory lock requests on the same key.
4. They are automatically released on session disconnect (no stale lock cleanup needed for session-level locks).

### Lock Granularity

Two granularity levels, chosen per operation:

| Granularity | Lock Key | Use Case |
|-------------|----------|----------|
| **Per-device** (coarse) | `device_id` hash | Firmware upgrades, full config push, power cycle — operations that affect the entire device |
| **Per-endpoint** (fine-grained) | `device_id` + `resource_id` hash | Port config, VLAN on specific interface, ACL on specific interface — operations that affect one resource on the device |

**Rule:** A per-device lock blocks ALL per-endpoint locks on that device. A per-endpoint lock on port 1/1 does NOT block a per-endpoint lock on port 1/2.

This means: Tenant A's lock on switch port 1/1 blocks Tenant B's request for port 1/1 but NOT port 1/2.

---

## Go Types

```go
package locking

import (
    "context"
    "time"
)

// LockGranularity determines what scope a lock covers.
type LockGranularity string

const (
    LockGranularityDevice   LockGranularity = "device"   // entire device
    LockGranularityEndpoint LockGranularity = "endpoint"  // specific resource on a device
)

// LockRequest is submitted by a workflow before executing a mutating activity.
type LockRequest struct {
    DeviceID      string          `json:"device_id"`       // LOOM internal device ID
    ResourceID    string          `json:"resource_id"`     // specific resource (e.g., "port/1/1"), empty for device-level
    Granularity   LockGranularity `json:"granularity"`
    WorkflowID    string          `json:"workflow_id"`     // Temporal workflow ID holding the lock
    TenantID      string          `json:"tenant_id"`       // tenant requesting the lock
    OperationType string          `json:"operation_type"`  // what operation needs the lock
    Timeout       time.Duration   `json:"timeout"`         // how long the lock is valid
    RequestedAt   time.Time       `json:"requested_at"`
}

// LockResult is returned after a lock attempt.
type LockResult struct {
    Acquired   bool          `json:"acquired"`
    LockID     string        `json:"lock_id"`       // unique ID for this lock instance (for release)
    ExpiresAt  time.Time     `json:"expires_at"`    // when the lock auto-expires
    HeldBy     *LockHolder   `json:"held_by"`       // if not acquired, who holds it
    RetryAfter time.Duration `json:"retry_after"`   // suggested wait before retry, zero if acquired
}

// LockHolder describes who currently holds a conflicting lock.
type LockHolder struct {
    WorkflowID    string    `json:"workflow_id"`
    TenantID      string    `json:"tenant_id"`
    OperationType string    `json:"operation_type"`
    AcquiredAt    time.Time `json:"acquired_at"`
    ExpiresAt     time.Time `json:"expires_at"`
}

// DeviceLock represents an active lock in the system.
type DeviceLock struct {
    LockID        string          `json:"lock_id"`         // UUID v7
    DeviceID      string          `json:"device_id"`
    ResourceID    string          `json:"resource_id"`     // empty for device-level locks
    Granularity   LockGranularity `json:"granularity"`
    WorkflowID    string          `json:"workflow_id"`
    TenantID      string          `json:"tenant_id"`
    OperationType string          `json:"operation_type"`
    AcquiredAt    time.Time       `json:"acquired_at"`
    ExpiresAt     time.Time       `json:"expires_at"`
    Released      bool            `json:"released"`
    ReleasedAt    *time.Time      `json:"released_at"`
}

// DeviceLocker is the interface for acquiring and releasing device locks.
type DeviceLocker interface {
    // Acquire attempts to acquire a lock. Returns immediately with the result.
    // Does NOT block waiting for the lock — the caller decides whether to retry.
    Acquire(ctx context.Context, req LockRequest) (*LockResult, error)

    // Release releases a previously acquired lock by its LockID.
    Release(ctx context.Context, lockID string) error

    // Extend extends the expiration of an active lock. Used by long-running
    // operations that need more time than the original timeout.
    Extend(ctx context.Context, lockID string, extension time.Duration) error

    // Status returns the current lock state for a device or endpoint.
    Status(ctx context.Context, deviceID string, resourceID string) (*DeviceLock, error)

    // CleanupExpired removes locks that have passed their ExpiresAt time.
    // Called periodically by a background goroutine.
    CleanupExpired(ctx context.Context) (int, error)
}
```

---

## Lock Timeout Defaults

| Operation Category | Default Timeout | Rationale |
|-------------------|----------------|-----------|
| Config read / state check | 30 seconds | Short, non-mutating |
| Port / VLAN / ACL config | 5 minutes | Standard mutation |
| Firmware upgrade | 30 minutes | Long-running, device may reboot |
| Full config restore | 10 minutes | Snapshot restore may be slow |
| Power cycle | 5 minutes | Includes wait for reachability |

Timeouts are configurable per operation type via LOOM configuration. The lock holder can call `Extend` if the operation needs more time, but each extension is capped at 2x the original timeout.

---

## PostgreSQL Advisory Lock Implementation

Advisory locks use a 64-bit integer key derived from the device/resource identifiers:

```go
// lockKey generates a PostgreSQL advisory lock key from device and resource IDs.
// Device-level: hash(device_id)
// Endpoint-level: hash(device_id + resource_id)
func lockKey(deviceID string, resourceID string) int64 {
    h := fnv.New64a()
    h.Write([]byte(deviceID))
    if resourceID != "" {
        h.Write([]byte(":"))
        h.Write([]byte(resourceID))
    }
    return int64(h.Sum64() & 0x7FFFFFFFFFFFFFFF) // ensure positive
}
```

**Lock acquisition** uses `pg_try_advisory_lock(key)` (non-blocking). If the lock is not available, the function returns `false` immediately. The workflow decides whether to retry.

**Lock release** uses `pg_advisory_unlock(key)`.

**Device-level lock exclusion:** When a device-level lock is acquired, LOOM also records it in a `device_locks` table. Endpoint-level lock requests check this table first — if a device-level lock exists for the same device, the endpoint lock is denied.

### Lock Tracking Table

```sql
CREATE TABLE device_locks (
    lock_id       UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    device_id     UUID NOT NULL REFERENCES devices(id),
    resource_id   TEXT,           -- NULL for device-level locks
    granularity   TEXT NOT NULL,  -- 'device' or 'endpoint'
    workflow_id   TEXT NOT NULL,
    tenant_id     UUID NOT NULL REFERENCES tenants(id),
    operation_type TEXT NOT NULL,
    acquired_at   TIMESTAMPTZ NOT NULL DEFAULT now(),
    expires_at    TIMESTAMPTZ NOT NULL,
    released      BOOLEAN NOT NULL DEFAULT false,
    released_at   TIMESTAMPTZ
);

CREATE INDEX idx_device_locks_device ON device_locks(device_id) WHERE NOT released;
CREATE INDEX idx_device_locks_expiry ON device_locks(expires_at) WHERE NOT released;
```

---

## Stale Lock Detection and Cleanup

1. **Automatic expiration:** A background goroutine runs every 60 seconds, calling `CleanupExpired`. Locks past their `expires_at` are marked `released = true` and the corresponding advisory lock is released.

2. **Workflow crash detection:** If a Temporal workflow crashes or times out, its locks become stale. The cleanup goroutine detects these by checking whether the `workflow_id` corresponds to a running Temporal workflow. If the workflow is no longer running, the lock is released.

3. **Session disconnect:** If the PostgreSQL connection holding the advisory lock is dropped, PostgreSQL automatically releases the advisory lock. The tracking table is cleaned up on the next cleanup cycle.

---

## Workflow Behavior When Lock Is Unavailable

When a workflow attempts to acquire a lock and fails, three strategies are available (configured per workflow type):

### 1. Queue (default for non-urgent operations)
The workflow enters a wait state using a Temporal timer. It retries lock acquisition after `RetryAfter` (returned by the lock service). Maximum wait: 3x the lock timeout of the current holder.

### 2. Retry with Exponential Backoff
The workflow retries lock acquisition with exponential backoff: 1s, 2s, 4s, 8s, ... up to a configurable maximum (default: 5 minutes total). Used for time-sensitive operations that can tolerate brief delays.

### 3. Fail Immediately
The workflow fails with a `DeviceLockConflictError` (a `PermanentError` subtype). Used when the operation is time-critical and cannot wait. The operator is notified with details of who holds the lock.

```go
// DeviceLockConflictError is returned when a lock cannot be acquired.
type DeviceLockConflictError struct {
    Code          string     `json:"code"`           // "DEVICE_LOCK_CONFLICT"
    Message       string     `json:"message"`
    DeviceID      string     `json:"device_id"`
    ResourceID    string     `json:"resource_id"`
    HeldBy        LockHolder `json:"held_by"`
    CorrelationID string     `json:"correlation_id"`
}
```

---

## Temporal Integration

Locks are acquired and released within the Temporal workflow, bracketing the mutating activity:

```
Workflow Step:
  1. Acquire device lock (activity)
  2. Take pre-op snapshot if SnapshotCapable (activity)  — see ADAPTER-CONTRACT.md
  3. Execute mutation (activity)
  4. Verify result (activity)
  5. Release device lock (activity)

On failure at step 3 or 4:
  1. Compensate (activity)
  2. Release device lock (activity)  — lock is ALWAYS released, even on failure
```

**Lock release is in a `defer`-equivalent pattern** — the Temporal workflow ensures the lock is released in all code paths (success, failure, compensation, timeout). If the workflow crashes before releasing, the stale lock cleanup handles it.

**Lock is NOT held across approval gates.** If a workflow requires human approval between mutation steps, the lock is released before the approval gate and re-acquired after approval. This prevents a human's response time from blocking other workflows.

---

## Multi-Tenant Resource Conflicts

Beyond device-level locking, shared resources (VLAN IDs, IP ranges) require a resource allocation service. This is separate from device locking:

- **VLAN IDs** on a shared switch are globally scoped, not per-tenant. Tenant A cannot use VLAN 100 on SW-01 if Tenant B already owns it.
- **IP address ranges** assigned to a tenant cannot overlap with another tenant's ranges on the same network segment.

Resource allocation is tracked in the `resource_allocations` table (not in this doc — see DOMAIN-MODEL.md for the canonical model). Device locking prevents concurrent mutations; resource allocation prevents logical conflicts.
