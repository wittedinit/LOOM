Now I have comprehensive research data. Here is the complete analysis:

---

# LOOM Data Storage Architecture Analysis

## 1. Relational Databases (Structured Inventory, Workflow, Cost, Audit)

### PostgreSQL

**Strengths for LOOM:**
- Mature, battle-tested with decades of production reliability
- Extensible architecture allows adding time-series (TimescaleDB), graph (Apache AGE), and vector (pgvector) capabilities without separate database systems
- JSONB columns handle semi-structured device capability profiles natively
- Row-level security and fine-grained access control for multi-tenant audit data
- Rich ecosystem: logical replication, pg_partman for partition management, pg_cron for scheduled jobs
- Apache 2.0 / PostgreSQL License -- fully open source with no usage restrictions

**Weaknesses:**
- Single-node by default; horizontal scaling requires Citus, Patroni + HAProxy, or an external distributed layer
- Write-heavy audit trails at extreme scale (millions of events/sec) can stress WAL and replication
- Multi-site/multi-region requires careful orchestration (BDR, logical replication, or external tooling)

**Scaling:** Vertical scaling is straightforward to ~96 cores / 1TB RAM. Horizontal read scaling via streaming replicas. For write scaling, Citus (now part of Microsoft) provides distributed tables. For multi-region, consider Citus or move to a distributed SQL system.

### CockroachDB

**Strengths:** Native multi-region, strong consistency via Raft, automatic rebalancing, geo-partitioning for data residency compliance.

**Weaknesses:** [CockroachDB abandoned open source in 2024](https://www.infoq.com/news/2024/09/cockroachdb-license-concerns/), moving to a proprietary Enterprise license. Companies with >$10M revenue must pay per-CPU fees. Mandatory telemetry on free tiers. [ZITADEL migrated away from CockroachDB back to PostgreSQL](https://zitadel.com/blog/move-to-postgresql) citing complexity and performance issues. Latency overhead from distributed consensus on every write.

**Verdict for LOOM:** The licensing change makes CockroachDB a risky long-term bet for an open infrastructure project. PostgreSQL with Citus or Patroni is a safer foundation.

### TiDB

**Strengths:** Apache 2.0 licensed, MySQL-compatible wire protocol, separates compute (TiDB) from storage (TiKV), enabling independent scaling. Strong in OLTP+OLAP hybrid workloads via TiFlash.

**Weaknesses:** MySQL compatibility (not PostgreSQL), which limits extension ecosystem. Complex multi-component architecture (TiDB + TiKV + PD + TiFlash). Smaller community outside of PingCAP's ecosystem.

**Verdict for LOOM:** Good technology, but the PostgreSQL ecosystem is richer for LOOM's multi-model needs (AGE, TimescaleDB, pgvector all require PostgreSQL).

---

## 2. Graph Databases (Topology, Dependencies, Impact Analysis)

### Apache AGE on PostgreSQL

**Strengths for LOOM:**
- Runs as a PostgreSQL extension -- no separate database to operate
- Supports openCypher query language for graph traversal ("which devices are affected if switch X fails?")
- Graph data lives alongside relational data; JOINs between graph and relational tables are possible
- Apache 2.0 licensed
- Eliminates the operational burden of synchronizing a separate graph database

**Weaknesses:**
- Less mature than Neo4j for deep graph analytics (PageRank, community detection, shortest path algorithms)
- Performance on deeply recursive traversals (6+ hops) lags behind native graph engines
- Smaller community and fewer graph visualization tools

### Neo4j

**Strengths:** Most mature graph database, rich ecosystem (APOC library, Graph Data Science library with 65+ algorithms), excellent visualization tools (Bloom, Browser), Cypher is the de facto graph query standard. [Proven in network topology use cases](https://medium.com/self-study-notes/exploring-graph-database-capabilities-neo4j-vs-postgresql-105c9e85bb5d).

**Weaknesses:** Community Edition is GPL (copyleft); Enterprise Edition is commercial with significant licensing costs. Clustering (causal clusters) only in Enterprise. Cannot do relational JOINs -- requires data duplication or ETL from the relational store.

### ArangoDB

**Strengths:** [Multi-model: document + graph + key-value in one engine](https://arango.ai/blog/arangodb-vs-neo4j/). [Benchmarks show up to 8x faster than Neo4j](https://arango.ai/blog/benchmark-results-arangodb-vs-neo4j-arangodb-up-to-8x-faster-than-neo4j/) on certain graph computations. AQL (ArangoDB Query Language) handles all three data models. SmartGraphs for sharded graph traversal.

**Weaknesses:** [Shifted from Apache 2.0 to BSL 1.1 starting with version 3.12](https://en.wikipedia.org/wiki/ArangoDB). Smaller community than PostgreSQL or Neo4j. Being a multi-model database means it is "jack of all trades" -- relational queries, time-series, and vector search are not its strength.

**Verdict for LOOM:** Apache AGE on PostgreSQL is the recommended starting point. It keeps graph data co-located with relational data, eliminates a separate database to operate, and handles LOOM's topology queries (VLAN membership, cable maps, impact analysis for 2-4 hop traversals) adequately. If LOOM grows to need advanced graph algorithms (community detection across thousands of devices, complex shortest-path computations), Neo4j Community Edition can be added as a read-only projection of the topology.

---

## 3. Time-Series Databases (Telemetry, Metrics, Performance Data)

### TimescaleDB (on PostgreSQL)

**Strengths for LOOM:**
- PostgreSQL extension -- same database engine, same SQL, same operational tooling
- Continuous aggregates for automatic rollup (1-min -> 5-min -> 1-hour -> 1-day)
- Retention policies with automatic chunk dropping
- Compression achieves 90-95% storage reduction on time-series data
- Full SQL for complex analytical queries (JOIN metrics with device inventory)
- [Apache 2.0 licensed (community edition)](https://github.com/timescale/timescaledb)

**Weaknesses:**
- [At extreme ingestion rates, VictoriaMetrics achieves 2.2M datapoints/sec vs TimescaleDB's 480K datapoints/sec](https://valyala.medium.com/high-cardinality-tsdb-benchmarks-victoriametrics-vs-timescaledb-vs-influxdb-13e6ee64dd6b)
- [VictoriaMetrics uses up to 70x less storage](https://valyala.medium.com/high-cardinality-tsdb-benchmarks-victoriametrics-vs-timescaledb-vs-influxdb-13e6ee64dd6b) for the same data
- High-cardinality scenarios (100K+ unique metric series) require careful partitioning

### VictoriaMetrics

**Strengths:** Purpose-built for Prometheus-compatible metrics. Exceptional ingestion performance (2.2M datapoints/sec) and compression. PromQL and MetricsQL support. Trivial to operate (single binary). Excellent Grafana integration. Open source (Apache 2.0 for single-node; Enterprise for cluster).

**Weaknesses:** Not SQL -- requires PromQL/MetricsQL. Cannot JOIN with relational data without an intermediary. Separate system to operate. Cluster version has enterprise licensing considerations.

### InfluxDB

**Strengths:** Purpose-built time-series with Flux query language, built-in dashboarding (Chronograf), good documentation.

**Weaknesses:** [Ingestion rate of only 330K datapoints/sec with 20.5GB RAM](https://valyala.medium.com/high-cardinality-tsdb-benchmarks-victoriametrics-vs-timescaledb-vs-influxdb-13e6ee64dd6b) -- worst performer in benchmarks. InfluxDB 3.0 rewrote the engine in Rust/Arrow but changed licensing. Flux is being deprecated in favor of SQL in v3, creating migration churn.

**Verdict for LOOM:** Two-tier approach:

- **Primary (moderate scale, <500K metrics/sec):** TimescaleDB on the same PostgreSQL instance as inventory/workflow data. This eliminates a separate database, enables SQL JOINs between metrics and device inventory, and simplifies operations enormously.
- **Secondary (high-scale, >500K metrics/sec):** If LOOM scales to managing thousands of devices with high-frequency telemetry, deploy VictoriaMetrics as a dedicated metrics store fed by Prometheus scrapers or NATS, with Grafana dashboards. VictoriaMetrics handles this scale with minimal resources.

---

## 4. Key-Value / Document Stores (Real-time State, Caching)

### Redis (Valkey)

**Strengths for LOOM:**
- Sub-millisecond latency for real-time device state lookups
- Pub/Sub for broadcasting state changes to UI and other consumers
- Redis Streams for lightweight event processing
- Data structures (sorted sets for leaderboards, hashes for device state, sets for group membership)
- [Valkey](https://www.openlogic.com/blog/exploring-redis-alternatives) is the Linux Foundation fork after Redis changed to dual SSPL/RSAL licensing -- fully open source (BSD)

**Weaknesses:** In-memory by default -- data loss on restart without persistence configured (RDB/AOF). Not suitable as a primary datastore for critical state. Memory-bound scaling.

**Use in LOOM:** Caching layer for hot device state, session data, real-time dashboard state. NOT the source of truth -- PostgreSQL is.

### etcd

**Strengths:** [The backbone of Kubernetes](https://medium.com/@harshithgowdakt/redis-vs-etcd-choosing-the-right-data-store-for-your-distributed-system-a911b736952d). Strong consistency via Raft. Purpose-built for configuration storage and service discovery. Watch API for real-time change notifications.

**Weaknesses:** Limited to ~8GB data recommended. Not designed for high-throughput data storage. Only key-value, no complex queries.

**Use in LOOM:** If LOOM uses Kubernetes for its own control plane, etcd is already present. Could store LOOM cluster coordination state (leader election, distributed locks). Not suitable for device inventory or telemetry.

### FoundationDB

**Strengths:** [ACID transactions across distributed key-value store](https://notes.eatonphil.com/whats-the-big-deal-about-key-value-databases.html). Apple uses it at massive scale. "Layer" architecture allows building custom data models on top. Apache 2.0 licensed.

**Weaknesses:** Steep learning curve. Small community. No built-in query language -- you build everything yourself. Few managed offerings.

**Verdict for LOOM:** Valkey (Redis fork) for caching and real-time state. etcd only if LOOM's control plane is Kubernetes-native. FoundationDB is overkill for LOOM's needs given that PostgreSQL already covers ACID transactional requirements.

---

## 5. Event Streaming (Event-Driven Architecture, Audit Ingestion)

### NATS / NATS JetStream

**Strengths for LOOM:**
- [Single binary under 20MB](https://nats.io/about/) -- runs on Raspberry Pi to cloud servers
- [Purpose-built for edge computing with intermittent connectivity](https://www.synadia.com/customer-stories/machinemetrics)
- JetStream adds persistence, exactly-once delivery, key-value store, and object store
- Sub-millisecond latency, handles millions of messages/sec
- Built-in multi-tenancy via accounts
- Leaf nodes enable hub-and-spoke topology (central LOOM cluster <-> edge sites)
- Apache 2.0 licensed
- [Used in industrial IoT for edge-to-cloud telemetry with unreliable networks](https://nats.io/blog/real-time-monitoring-solution-jetstream-risingwave-superset/)

**Weaknesses:** Smaller ecosystem than Kafka. Fewer connectors (no equivalent of Kafka Connect's 200+ connectors). Stream processing is more basic (no equivalent of Kafka Streams or ksqlDB).

### Apache Kafka / Redpanda

**Strengths:** [Kafka handles >1M messages/sec at p99 <5ms](https://medium.com/@BuildShift/kafka-is-old-redpanda-is-fast-pulsar-is-weird-nats-is-tiny-which-message-broker-should-you-32ce61d8aa9f). Massive ecosystem (Kafka Connect, Kafka Streams, ksqlDB, Schema Registry). Redpanda is a [C++ Kafka-compatible alternative](https://www.redpanda.com/guides/kafka-alternatives) with no JVM, no ZooKeeper, and lower latency.

**Weaknesses:** Kafka is operationally heavy (JVM tuning, ZooKeeper/KRaft migration, broker management). Minimum 3-node cluster. Not suited for edge deployment. Redpanda is source-available (BSL), not open source.

**Verdict for LOOM:** NATS with JetStream is the clear winner for LOOM. Its edge-native design matches LOOM's need to run agents on remote sites. The leaf node topology maps perfectly to LOOM's hub-and-spoke architecture. The small footprint means LOOM agents can embed NATS on edge devices. JetStream provides the persistence and replay needed for audit trails and workflow events. Kafka/Redpanda is overkill for LOOM's messaging needs and cannot run at the edge.

---

## 6. Embedded / Edge Databases

### SQLite

**Strengths for LOOM edge agents:**
- [Unbeatable for transactional SQL in resource-constrained environments](https://www.explo.co/blog/embedded-sql-databases)
- Single file, zero configuration, zero administration
- Handles offline-first operation natively -- agents can queue operations and sync when connectivity returns
- [Perfect for mobile apps and lightweight embedded systems](https://motherduck.com/learn-more/duckdb-vs-sqlite-databases/)
- Public domain license

**Use in LOOM:** Each LOOM edge agent embeds SQLite for local device inventory, queued commands, cached credentials, and offline workflow state.

### DuckDB

**Strengths:** [Optimized for analytical workloads](https://www.datacamp.com/blog/duckdb-vs-sqlite-complete-database-comparison) -- columnar storage, vectorized execution. Excellent for local telemetry aggregation before shipping to the central store.

**Use in LOOM:** Edge agents use DuckDB for local telemetry rollup (aggregate 1-second metrics into 1-minute summaries before transmitting to central). Not a replacement for SQLite for transactional workloads.

**Verdict:** SQLite for transactional state at the edge + DuckDB for local analytics/telemetry pre-aggregation.

---

## 7. Vector Databases (LLM Context, RAG, Semantic Search)

### pgvector (PostgreSQL extension)

**Strengths for LOOM:**
- [Runs inside PostgreSQL](https://liveblocks.io/blog/whats-the-best-vector-database-for-building-ai-products) -- no separate database
- [Adequate performance for sub-10M vectors](https://getathenic.com/blog/pinecone-vs-weaviate-vs-qdrant-vs-pgvector)
- Supports HNSW and IVFFlat indexes
- Can combine vector similarity search with SQL WHERE clauses (e.g., "find similar configs for devices of type 'switch' in datacenter 'us-east-1'")

**Weaknesses:** [At 50M+ vectors, purpose-built vector DBs are 10x+ faster](https://getathenic.com/blog/pinecone-vs-weaviate-vs-qdrant-vs-pgvector). Vector indexes consume significant memory.

### Qdrant

**Strengths:** [Written in Rust, purpose-built for performance](https://www.zenml.io/blog/vector-databases-for-rag). Fastest at scale (41.47 QPS at 99% recall on 50M vectors). Built-in filtering, quantization for memory reduction. [Self-hosted costs $100-300/month](https://introl.com/blog/vector-database-infrastructure-pinecone-weaviate-qdrant-scale). Apache 2.0 licensed.

**Verdict for LOOM:** Start with pgvector. LOOM's vector needs (embedding infrastructure docs, config snippets, conversation history) will likely be under 5M vectors for years. pgvector eliminates a separate system. If LOOM grows to need massive-scale semantic search, Qdrant is the best self-hosted upgrade path.

---

## 8. Unified / Multi-Model Approaches

### PostgreSQL + Extensions (Recommended Unified Core)

Running TimescaleDB, Apache AGE, and pgvector as PostgreSQL extensions creates a single database engine that handles:

| Data Type | Extension | Query Language |
|-----------|-----------|---------------|
| Relational (inventory, workflow, cost, audit) | Core PostgreSQL | SQL |
| Time-Series (telemetry, metrics) | TimescaleDB | SQL |
| Graph (topology, dependencies) | Apache AGE | openCypher + SQL |
| Vector (LLM embeddings, semantic search) | pgvector | SQL |
| Document (device profiles, configs) | JSONB (native) | SQL |

**Advantages:**
- Single backup/restore pipeline
- Single connection pool
- Single monitoring stack
- Transactions span all data models (update device inventory AND topology AND metrics in one transaction)
- One team to train, one system to tune

**Limitations:**
- Cannot scale each workload independently (time-series ingestion vs. graph traversal vs. OLTP)
- At extreme scale, purpose-built systems outperform extensions
- PostgreSQL upgrades must consider all extensions' compatibility

### ArangoDB Multi-Model

**Assessment:** While ArangoDB elegantly combines document + graph + key-value in one engine, it [shifted to BSL 1.1 licensing](https://en.wikipedia.org/wiki/ArangoDB), lacks time-series capabilities, lacks vector search maturity, and has a smaller community than PostgreSQL. It solves one problem (graph + document) but creates others (no time-series, no SQL compatibility, licensing risk).

---

## Recommended Architecture for LOOM

### Tier 1: Core Data Platform (PostgreSQL + Extensions)

```
PostgreSQL 16+
  |-- TimescaleDB     (telemetry, metrics, cost time-series)
  |-- Apache AGE       (network topology, dependency graphs)
  |-- pgvector         (LLM embeddings, semantic search over configs)
  |-- JSONB            (device capability profiles, config snapshots)
  |-- pg_partman       (partition management for audit trail)
  |-- pg_cron          (scheduled maintenance, retention)
```

**Handles:** Device inventory, infrastructure state, topology/graph data, workflow state, cost data, audit trail, configuration history, LLM context, moderate-scale telemetry.

**Scaling path:** Single node -> Patroni + HAProxy for HA -> Citus for horizontal write scaling.

### Tier 2: Event Backbone (NATS + JetStream)

```
NATS Cluster (central)
  |-- JetStream streams:
  |     |-- DEVICE.STATE.*     (device state changes)
  |     |-- TELEMETRY.*        (metrics ingestion pipeline)
  |     |-- WORKFLOW.*         (orchestration events)
  |     |-- AUDIT.*            (audit log streaming)
  |     |-- CONFIG.CHANGE.*    (configuration change events)
  |-- Leaf nodes (edge sites)
  |-- KV Store (ephemeral real-time state)
```

**Handles:** Event-driven architecture, telemetry ingestion pipeline, audit log streaming, inter-service messaging, edge-to-cloud communication, device state change propagation.

### Tier 3: Caching Layer (Valkey/Redis)

```
Valkey (Redis-compatible, BSD licensed)
  |-- Device state cache (hash per device)
  |-- Session/auth tokens
  |-- Rate limiting
  |-- Real-time dashboard state
  |-- Pub/Sub for WebSocket fan-out to UI
```

**Handles:** Sub-millisecond reads for hot data, UI real-time updates, ephemeral state.

### Tier 4: Edge Agents (SQLite + DuckDB)

```
LOOM Edge Agent (per remote site)
  |-- SQLite: local device inventory, command queue, credentials cache
  |-- DuckDB: local telemetry aggregation before sync
  |-- NATS leaf node: edge-to-central communication
```

**Handles:** Offline-first operation, local autonomy during network outages, telemetry pre-aggregation.

### Tier 5: Scale-Out (Added When Needed, Not Day 1)

| Trigger | Add |
|---------|-----|
| Telemetry >500K datapoints/sec | VictoriaMetrics cluster |
| Graph queries >4 hops or need GDS algorithms | Neo4j read replica |
| Vector search >5M embeddings | Qdrant cluster |

---

## Data Flow Architecture

```
Edge Sites                        Central LOOM Cluster
-----------                       --------------------

[Device] --metrics--> [LOOM Agent]
                         |
                    [SQLite/DuckDB]     -- NATS Leaf Node -->  [NATS JetStream]
                    (offline buffer)                                |
                                                    +--------------+--------------+
                                                    |              |              |
                                              [Telemetry      [Audit         [State
                                               Consumer]       Consumer]      Consumer]
                                                    |              |              |
                                                    v              v              v
                                              [PostgreSQL + TimescaleDB + AGE + pgvector]
                                                                   |
                                                              [Valkey Cache]
                                                                   |
                                                              [API / UI]
```

---

## System Count and Operational Complexity

| System | Purpose | License | Operational Burden |
|--------|---------|---------|-------------------|
| PostgreSQL + extensions | Core data platform | PostgreSQL/Apache 2.0 | Medium (one DB to manage) |
| NATS + JetStream | Event streaming + edge comms | Apache 2.0 | Low (single binary) |
| Valkey | Caching | BSD | Low |
| SQLite | Edge agent state | Public domain | None (embedded) |
| DuckDB | Edge analytics | MIT | None (embedded) |

**Total distinct systems to operate: 3** (PostgreSQL, NATS, Valkey) -- plus embedded databases that require zero operational overhead.

Compare this to a "best-of-breed" approach that would require operating PostgreSQL + Neo4j + VictoriaMetrics + Kafka + Redis + Qdrant = **6 systems**, each with its own backup strategy, monitoring, upgrade cycle, and failure modes.

---

## Why This Architecture Works for LOOM

1. **Minimal operational surface.** Three systems to monitor, back up, and upgrade. LOOM is an infrastructure orchestrator -- it should not require more infrastructure to run than the infrastructure it manages.

2. **Edge-native.** NATS leaf nodes + SQLite/DuckDB give LOOM agents full autonomy during network outages. When connectivity returns, JetStream replays queued events.

3. **Unified query model.** SQL queries can JOIN device inventory with telemetry with topology with cost data. "Show me all switches in rack A3 with >80% bandwidth utilization that are in the blast radius of router R1" is a single query spanning TimescaleDB, AGE, and core tables.

4. **Progressive scaling.** Start with a single PostgreSQL instance handling everything. Add Valkey when you need sub-millisecond caching. Scale NATS horizontally as message volume grows. Only add purpose-built systems (VictoriaMetrics, Neo4j, Qdrant) when PostgreSQL extensions hit their limits -- which for most deployments will be years out.

5. **Fully open source.** Every component uses a permissive open-source license (PostgreSQL, Apache 2.0, BSD, MIT, Public Domain). No BSL, no SSPL, no revenue-based licensing triggers.

6. **Proven at scale.** PostgreSQL runs the world's most demanding workloads. NATS powers edge IoT platforms at companies like MachineMetrics. Valkey/Redis is the most deployed caching layer on the planet. SQLite is the most widely deployed database engine in existence.

---

Sources:
- [Distributed SQL Comparison: CockroachDB vs TiDB](https://sanj.dev/post/distributed-sql-databases-comparison)
- [CockroachDB vs PostgreSQL](https://www.bytebase.com/blog/cockroachdb-vs-postgres/)
- [CockroachDB License Concerns](https://www.infoq.com/news/2024/09/cockroachdb-license-concerns/)
- [ZITADEL's Migration from CockroachDB to PostgreSQL](https://zitadel.com/blog/move-to-postgresql)
- [Neo4j vs PostgreSQL Graph Capabilities](https://medium.com/self-study-notes/exploring-graph-database-capabilities-neo4j-vs-postgresql-105c9e85bb5d)
- [ArangoDB vs Neo4j Benchmark](https://arango.ai/blog/benchmark-results-arangodb-vs-neo4j-arangodb-up-to-8x-faster-than-neo4j/)
- [Apache AGE vs Neo4j](https://dev.to/pawnsapprentice/apache-age-vs-neo4j-battle-of-the-graph-databases-2m4)
- [Open Source Graph Databases 2026](https://www.index.dev/blog/top-10-open-source-graph-databases)
- [TSDB Benchmarks: VictoriaMetrics vs TimescaleDB vs InfluxDB](https://valyala.medium.com/high-cardinality-tsdb-benchmarks-victoriametrics-vs-timescaledb-vs-influxdb-13e6ee64dd6b)
- [Best Time Series Databases 2026](https://cratedb.com/blog/best-time-series-databases)
- [Kafka vs Redpanda vs NATS vs Pulsar 2025](https://medium.com/@BuildShift/kafka-is-old-redpanda-is-fast-pulsar-is-weird-nats-is-tiny-which-message-broker-should-you-32ce61d8aa9f)
- [NATS at the Edge](https://www.synadia.com/customer-stories/machinemetrics)
- [NATS About](https://nats.io/about/)
- [Vector Database Comparison](https://getathenic.com/blog/pinecone-vs-weaviate-vs-qdrant-vs-pgvector)
- [Best Vector Databases 2026](https://www.firecrawl.dev/blog/best-vector-databases)
- [Vector Databases for RAG](https://www.zenml.io/blog/vector-databases-for-rag)
- [Redis Alternatives and Valkey](https://www.openlogic.com/blog/exploring-redis-alternatives)
- [Redis vs etcd](https://medium.com/@harshithgowdakt/redis-vs-etcd-choosing-the-right-data-store-for-your-distributed-system-a911b736952d)
- [FoundationDB Overview](https://notes.eatonphil.com/whats-the-big-deal-about-key-value-databases.html)
- [Embedded SQL Databases 2025](https://www.explo.co/blog/embedded-sql-databases)
- [DuckDB vs SQLite](https://motherduck.com/learn-more/duckdb-vs-sqlite-databases/)
- [TimescaleDB on GitHub](https://github.com/timescale/timescaledb)
- [Apache AGE on GitHub](https://github.com/apache/age)
- [ArangoDB Wikipedia (BSL licensing)](https://en.wikipedia.org/wiki/ArangoDB)