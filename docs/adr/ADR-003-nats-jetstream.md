# ADR-003: NATS JetStream for Messaging

## Status

Accepted

## Context

LOOM needs an event bus for device state changes, telemetry streams, discovery events, and inter-service communication. The messaging system must work at edge sites with limited resources, support exactly-once delivery for state mutations, and be operationally simple.

## Decision

NATS JetStream is the messaging layer for all event-driven communication in LOOM.

Key factors:

- **Single 20MB binary**: The entire NATS server (including JetStream persistence) is a single binary under 20MB. No JVM, no ZooKeeper, no external dependencies. Starts in milliseconds.
- **Edge-native with leaf nodes**: NATS leaf nodes allow edge sites to run a local NATS instance that connects to the central cluster. Events are produced locally and forwarded when connectivity allows. Edge sites continue operating during network partitions.
- **Exactly-once delivery**: JetStream provides exactly-once semantics through message deduplication and consumer acknowledgment. Critical for state mutation events where duplicate processing would cause incorrect device state.
- **Embeddable in Go**: NATS server can be embedded directly in a Go process. For small edge deployments, the NATS server runs inside the LOOM binary itself — no separate process to manage.
- **Subject-based routing**: NATS subjects (`tenant.device.event_type`) provide natural multi-tenant event routing without topic management overhead.

### Rejected Alternatives

- **Apache Kafka**: Operationally heavy — requires JVM, ZooKeeper/KRaft, significant memory and disk. Not viable at edge sites with 2-4GB RAM. Excellent for high-throughput centralized streaming but over-engineered for LOOM's distributed topology.
- **Redpanda**: Kafka-compatible with better operational profile, but Business Source License (BSL) creates licensing risk. Cannot embed in Go binary for edge deployment.

## Consequences

- All inter-service events flow through NATS JetStream
- Edge sites run NATS leaf nodes (or embedded NATS) for local event processing
- Subject naming convention: `{tenant}.{domain}.{event_type}` (e.g., `acme.device.discovered`)
- JetStream streams are configured per event type with appropriate retention policies
- Monitoring uses NATS built-in metrics exposed to Prometheus
- Team members need familiarity with NATS subjects, streams, and consumer groups
