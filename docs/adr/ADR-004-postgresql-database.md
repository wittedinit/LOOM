# ADR-004: PostgreSQL as Primary Database

## Status

Accepted

## Context

LOOM stores device inventory, topology graphs, configuration history, telemetry time-series, and vector embeddings for LLM-assisted classification. Using separate databases for each data type (graph DB, time-series DB, vector DB) creates operational complexity, backup fragmentation, and cross-store query limitations.

## Decision

PostgreSQL with extensions is the primary database, providing multi-model storage without separate systems.

Key factors:

- **Extensions provide multi-model capability**:
  - **TimescaleDB**: Time-series hypertables for telemetry data (CPU, temperature, interface counters) with automatic partitioning and retention policies
  - **Apache AGE**: Graph queries for topology relationships (device-to-device, device-to-rack, rack-to-room) using openCypher syntax within SQL
  - **pgvector**: Vector similarity search for LLM embedding-based device classification and documentation retrieval
- **SQL JOINs across all data types**: A single query can join device inventory (relational), topology paths (graph), recent telemetry (time-series), and similar devices (vector) without cross-database ETL.
- **Single backup pipeline**: One `pg_dump` or WAL-based replication covers all data. No coordination between separate backup systems.
- **Mature ecosystem**: Connection pooling (PgBouncer), monitoring (pg_stat_statements), migration tools (golang-migrate), HA (Patroni) are well-established.

### Rejected Alternatives

- **CockroachDB**: Abandoned its open-source commitment. License changes make it unsuitable for edge deployment without commercial licensing.
- **Neo4j**: Purpose-built graph database, but requires a separate system alongside a relational DB. GPL license complicates distribution. Apache AGE provides sufficient graph capability within PostgreSQL.
- **ArangoDB**: Multi-model database with graph, document, and search. Business Source License (BSL) creates the same licensing concerns as Redpanda.

## Consequences

- PostgreSQL is the single database for all persistent state (inventory, topology, telemetry, embeddings)
- Extensions (TimescaleDB, AGE, pgvector) are required dependencies
- Database schema includes tenant_id on every table (see ADR-005)
- Temporal uses PostgreSQL as its persistence backend (shared infrastructure)
- Edge sites may run a local PostgreSQL instance with selective replication to central
- Team members need familiarity with PostgreSQL extensions beyond standard SQL
- Extension upgrades require coordination but are less complex than managing separate databases
