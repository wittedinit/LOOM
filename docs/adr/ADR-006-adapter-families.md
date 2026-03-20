# ADR-006: Operation-Family Adapter Model

## Status

Accepted

## Context

LOOM manages devices across many protocols (SNMP, Redfish, WS-Man, VMware, gNMI, IPMI, SSH). Each protocol supports different capabilities: SNMP can discover and read state but cannot execute configuration changes on most devices. Redfish can do all of these. VMware has a completely different operation model. An adapter abstraction is needed that accommodates this diversity without forcing artificial uniformity.

## Decision

Adapters implement operation families — small, focused interfaces representing a category of capability. An adapter registers only the families it actually supports.

Operation families:

- **Connector**: Establish and manage connections to devices (authentication, session lifecycle)
- **Discoverer**: Enumerate devices and their attributes (walk MIBs, query Redfish endpoints)
- **Executor**: Push configuration changes to devices (CLI commands, REST API calls, NETCONF edits)
- **StateReader**: Read current device state for verification and monitoring (interface status, firmware version)
- **Watcher**: Subscribe to real-time events from devices (SNMP traps, Redfish SSE, gNMI subscriptions)

An SNMP adapter might implement Connector, Discoverer, and StateReader but not Executor or Watcher (for devices without trap support). A Redfish adapter might implement all five. The system queries adapter capabilities at registration time.

### Rejected Alternative

- **Single universal interface**: One large interface with all operations. Every adapter must implement every method, even if it means returning "not supported" errors. This forces adapters to fake capabilities, makes capability discovery unreliable, and violates interface segregation. A 20-method interface that every adapter must satisfy discourages new adapter development.

## Consequences

- New adapters are easier to build — implement only the relevant families
- The orchestration layer checks adapter capabilities before attempting operations (no "not supported" errors at runtime)
- Capability discovery is reliable: `adapter.Supports(Executor)` returns a definitive answer
- Adding a new operation family is backward-compatible — existing adapters are unaffected
- Adapter registration includes a capability manifest that the scheduler uses for operation routing
- Testing is focused: each family interface has its own test suite
- Trade-off: more interfaces to understand, but each is small and cohesive
