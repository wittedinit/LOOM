# Failure Mode Catalog

> Every failure mode in the LOOM stack: what breaks, how we detect it, what the impact is, what LOOM does automatically, and what operators must do.

This document is prescriptive. Every behavior described here must be implemented and tested. If LOOM does not behave as described during a failure, that is a bug.

---

## 1. PostgreSQL Failures

PostgreSQL holds device inventory, workflow projections, topology (AGE), telemetry (TimescaleDB), embeddings (pgvector), cost data, and audit records. Temporal also uses PostgreSQL as its persistence backend.

### 1.1 Primary Down (Single Node)

- **Failure**: The sole PostgreSQL instance becomes unreachable (process crash, host failure, network partition to the DB host).
- **Detection**: Connection pool returns errors on all checkout attempts. Health check endpoint (`/healthz`) reports `database: unhealthy` within one failed probe cycle (default 5s). LOOM emits `loom._system.database.connection_lost` event to any still-functioning NATS connection.
- **Impact**: Total loss of new workflow submissions (API returns 503). Running Temporal workflows that need DB projection writes will have those activities fail. Temporal Server itself loses persistence and cannot schedule new tasks or record completions. Read APIs that query inventory/topology return 503. The UI shows stale data from Valkey cache (if populated) but cannot load fresh queries. LLM decisions that require RAG over pgvector are unavailable; deterministic fallbacks activate.
- **Behavior**: LOOM API rejects new requests with `503 Service Unavailable` and `Retry-After` header. Temporal workers stop polling because Temporal Server is also affected. NATS continues operating independently -- events are still published and buffered. Valkey continues serving cached reads. Edge agents continue operating with local SQLite; they buffer events for later sync.
- **Recovery**: Restart PostgreSQL. On startup, Temporal Server reconnects and replays any incomplete histories. The DB projector NATS consumer replays unacknowledged events to rebuild any missed projections. Verify with `loom admin db check` which compares Temporal workflow completion records against DB projection state and reconciles gaps.
- **Data Loss Risk**: None if WAL is on durable storage. Temporal's event history is the source of truth for execution state -- the DB projection can always be rebuilt from Temporal.
- **RTO**: < 5 minutes for process restart. < 30 minutes if host replacement is needed. Temporal and API resume immediately on reconnection -- no manual intervention beyond restarting PostgreSQL.

### 1.2 Primary Down (Patroni Cluster -- Failover)

- **Failure**: The PostgreSQL primary in a Patroni-managed cluster fails. Patroni promotes a synchronous replica to primary.
- **Detection**: Patroni detects primary failure via DCS (etcd/Consul) heartbeat timeout (default 30s TTL, 10s loop). Patroni promotes the sync standby. LOOM's connection pool detects connection errors and re-resolves the service endpoint (via DNS or VIP managed by Patroni). Health check recovers within one probe cycle after failover completes.
- **Impact**: Brief unavailability during failover window (typically 10-30 seconds). In-flight database transactions are rolled back. Temporal activities that were mid-DB-write fail and are retried by Temporal's retry policy. No completed work is lost because Temporal replays activities. API requests during the failover window receive 503 and retry.
- **Behavior**: Connection pool drains failed connections and establishes new ones to the promoted primary. Temporal Server reconnects to the new primary. All LOOM services resume without restart. The DB projector re-processes any unacknowledged NATS messages.
- **Recovery**: Automatic. Operator should verify replication is re-established by adding a new replica to replace the failed node. Run `patronictl list` to confirm cluster health. Run `loom admin db check` to verify projection consistency.
- **Data Loss Risk**: None with synchronous replication (Patroni `synchronous_mode: true`). Low risk with asynchronous replication -- at most a few transactions that were committed on the old primary but not replicated.
- **RTO**: 10-30 seconds (automatic).

### 1.3 Replication Lag Exceeds Threshold

- **Failure**: A read replica falls behind the primary by more than the configured threshold (default: 10 seconds / 16 MB of WAL).
- **Detection**: Monitoring queries `pg_stat_replication.replay_lag` and `pg_stat_replication.write_lag`. Alert fires when lag exceeds threshold. LOOM emits `loom._system.database.replication_lag_high` event.
- **Impact**: Read queries routed to the lagging replica return stale data. Dashboard shows outdated device state. If LOOM routes read-after-write queries to the replica, users may not see their own changes immediately.
- **Behavior**: LOOM's connection pool stops routing read queries to the lagging replica (connection pool lag awareness). All reads fall back to the primary until the replica catches up. No data loss. No workflow impact.
- **Recovery**: Investigate cause of lag (replica under-resourced, long-running queries on replica, network bandwidth). If replica cannot catch up, rebuild it from a fresh base backup. No LOOM restart needed.
- **Data Loss Risk**: None. The primary has all data. This is a read-path degradation only.
- **RTO**: Automatic rerouting within seconds. Replica recovery depends on root cause.

### 1.4 Disk Full

- **Failure**: PostgreSQL's data directory partition reaches capacity. PostgreSQL enters read-only mode or crashes.
- **Detection**: Filesystem monitoring alerts at 80% and 90% thresholds. PostgreSQL logs `PANIC: could not write to file` or `ERROR: could not extend file`. Health check fails. LOOM emits `loom._system.database.disk_full`.
- **Impact**: Identical to primary-down (1.1). All writes fail. Temporal Server cannot persist workflow state. Additionally, recovery is blocked until disk space is freed.
- **Behavior**: Same as 1.1. LOOM returns 503 on all write paths.
- **Recovery**: Free disk space immediately: truncate old WAL segments (`pg_archivecleanup`), drop old TimescaleDB chunks (`SELECT drop_chunks(...)` for data beyond retention), vacuum full on bloated tables, or expand the volume. Restart PostgreSQL if it crashed. Run `loom admin db check` after recovery.
- **Data Loss Risk**: Low. If PostgreSQL crashed mid-write, WAL recovery handles consistency. Risk increases if WAL segments were manually deleted -- never do this.
- **RTO**: Minutes if disk can be expanded online. Hours if physical disk replacement is needed.

### 1.5 Connection Pool Exhaustion

- **Failure**: All connections in the pool (pgxpool, default max 25) are checked out and none are returned. New connection requests block and then time out.
- **Detection**: Pool metrics show `pool_waiting` > 0 sustained, `pool_acquire_duration` exceeds threshold (default 5s). Health check includes pool saturation metric. API responses slow down before failing. LOOM emits `loom._system.database.pool_exhausted`.
- **Impact**: API requests time out. Temporal activities that need DB writes time out and are retried (consuming Temporal retry budget). If sustained, looks like a full DB outage to callers.
- **Behavior**: LOOM returns 503 with `Retry-After` for new requests. Temporal retries failed activities. Existing long-running transactions continue (they hold the connections).
- **Recovery**: Identify the connection leak or slow query. Check `pg_stat_activity` for long-running or idle-in-transaction connections. Kill stuck sessions with `pg_terminate_backend()`. Increase `pool_max_conns` if load is legitimate. Deploy PgBouncer as a connection pooler for high-connection-count scenarios. No LOOM restart needed -- the pool self-heals when connections are returned.
- **Data Loss Risk**: None. Operations either succeed or fail cleanly with retries.
- **RTO**: Minutes once the root cause is identified. Immediate if pool size is simply too small.

### 1.6 Corrupted WAL

- **Failure**: Write-ahead log segments are corrupted (disk error, incomplete write due to power loss without battery-backed write cache).
- **Detection**: PostgreSQL fails to start with `PANIC: could not redo WAL record` or `FATAL: invalid resource manager ID in primary checkpoint record`. This is visible only at startup or during replication.
- **Impact**: PostgreSQL cannot start. Full outage equivalent to 1.1. If corruption is on a replica, that replica is lost but primary continues serving.
- **Behavior**: LOOM cannot connect to PostgreSQL. Same behavior as 1.1.
- **Recovery**: If primary WAL is corrupted: restore from the most recent base backup plus WAL archive (PITR). If only replica WAL is corrupted: rebuild the replica from a fresh base backup of the primary. In extreme cases, `pg_resetwal` can force startup but may lose committed transactions -- last resort only. After DB recovery, run `loom admin db check` to reconcile Temporal state against DB projections.
- **Data Loss Risk**: Medium to High. PITR recovery is up to the last archived WAL segment. Transactions between the last archive and the crash are lost from the DB, but Temporal's event history (if it used a separate WAL or different storage) can be used to replay and rebuild projections.
- **RTO**: 30 minutes to hours depending on database size and backup freshness.

---

## 2. Temporal Server Failures

Temporal is the execution source of truth (ADR-008). It manages workflow state via event sourcing and replay. Temporal consists of four services: Frontend, History, Matching, and Worker (internal worker for system workflows).

### 2.1 All Temporal Services Down

- **Failure**: The entire Temporal cluster is unavailable (all services crashed, or the underlying PostgreSQL for Temporal is down).
- **Detection**: LOOM's Temporal client fails to connect. Health check reports `temporal: unhealthy`. gRPC calls return `UNAVAILABLE`. LOOM emits `loom._system.temporal.connection_lost`.
- **Impact**: No new workflows can be started. No running workflows can make progress (activities cannot be dispatched). Workflows that are in-flight are effectively frozen -- their state is preserved in Temporal's database but no execution occurs. The API can still serve read queries from PostgreSQL (device inventory, topology) and Valkey cache. NATS events continue flowing. Edge agents continue operating.
- **Behavior**: LOOM API returns `503 Service Unavailable` for any operation that requires starting or querying a workflow. Read-only API endpoints (device list, topology queries) continue working from PostgreSQL. The UI shows a degraded-mode banner. LLM decision engine is unaffected (it does not depend on Temporal directly).
- **Recovery**: Restart Temporal services. On restart, Temporal recovers all workflow state from its PostgreSQL database. Frozen workflows resume from their last recorded event. No manual intervention needed for workflow recovery. Verify with `tctl workflow list --status running` to confirm workflows resumed.
- **Data Loss Risk**: None. Temporal's event history is persisted in PostgreSQL. Workflow state survives full restart. Activities that were executing at crash time are retried from the beginning (activities must be idempotent).
- **RTO**: < 5 minutes for service restart. Workflows resume within seconds of Temporal becoming available.

### 2.2 Single Temporal Service Down

#### 2.2.1 History Service Down

- **Failure**: The History service, which manages workflow execution state and event persistence, is down.
- **Detection**: Frontend service returns `UNAVAILABLE` errors for workflow operations. Workflow create and signal calls fail. gRPC health check for History fails.
- **Impact**: No workflow state transitions can occur. New workflows cannot start. Running workflows cannot progress. Query-only operations (list workflows, get workflow status) may still work if the Frontend can serve from cache.
- **Behavior**: LOOM API returns 503 for workflow operations. The Temporal Frontend may return cached results for read queries briefly. Activities on workers are not dispatched. Workflows are frozen but safe.
- **Recovery**: Restart the History service. It reconnects to PostgreSQL and resumes processing. All workflow state is intact.
- **Data Loss Risk**: None.
- **RTO**: < 2 minutes.

#### 2.2.2 Matching Service Down

- **Failure**: The Matching service, which routes activities and workflows to workers via task queues, is down.
- **Detection**: Workers stop receiving tasks. Task queue depth grows. gRPC health check for Matching fails.
- **Impact**: Activities are not dispatched to workers. Workflows stall at activity invocations. New workflows can be created (they are persisted by History) but cannot begin execution. Existing activities that are already running on workers continue until completion, but their results cannot be acknowledged.
- **Behavior**: LOOM workflows enter a "stalled" state. The API accepts new workflow requests (they are queued in History) but they do not progress.
- **Recovery**: Restart the Matching service. It rebuilds its in-memory task queues from the database. Workers reconnect and receive pending tasks. Backlog clears.
- **Data Loss Risk**: None.
- **RTO**: < 2 minutes. Backlog clearance depends on queue depth.

#### 2.2.3 Frontend Service Down

- **Failure**: The Frontend service, which provides the gRPC API for clients, is down.
- **Detection**: All Temporal client connections fail. LOOM cannot call any Temporal API.
- **Impact**: LOOM cannot start, query, or signal any workflow. Functionally equivalent to a full Temporal outage from LOOM's perspective. However, History and Matching may still be running -- if a worker is mid-activity, it continues executing but cannot report results.
- **Behavior**: Same as 2.1 from the LOOM API perspective.
- **Recovery**: Restart the Frontend service. It is stateless -- it proxies to History and Matching.
- **Data Loss Risk**: None.
- **RTO**: < 1 minute.

#### 2.2.4 Temporal Internal Worker Down

- **Failure**: The internal Worker service, which runs Temporal's system workflows (archival, replication, scheduled workflows), is down.
- **Detection**: System workflows stop executing. Scheduled workflows do not fire. Archival stops.
- **Impact**: LOOM's user-defined workflows are not affected (they run on LOOM's own workers). Impact is limited to Temporal housekeeping: old histories are not archived, scheduled system tasks do not run.
- **Behavior**: No visible impact to LOOM operations in the short term. Long-term: Temporal's database grows without archival.
- **Recovery**: Restart the internal Worker service.
- **Data Loss Risk**: None.
- **RTO**: Not urgent. Can be deferred to maintenance window.

### 2.3 Temporal Database Unreachable

- **Failure**: Temporal Server cannot reach its PostgreSQL database (which may be the same or separate instance from LOOM's application database).
- **Detection**: Temporal services log database connection errors. All Temporal operations fail with internal errors.
- **Impact**: Identical to 2.1 (all Temporal services effectively down). Note: if Temporal shares the same PostgreSQL instance as LOOM's application database, this is simultaneously a PostgreSQL failure (1.1) affecting both systems.
- **Behavior**: Same as 2.1.
- **Recovery**: Restore PostgreSQL connectivity. Temporal reconnects automatically.
- **Data Loss Risk**: None if PostgreSQL itself is intact.
- **RTO**: Depends on PostgreSQL recovery time.

### 2.4 Worker Process Crash Mid-Activity

- **Failure**: A LOOM Temporal worker process panics or is OOM-killed while executing an activity (e.g., mid-way through a Redfish config push or SSH session).
- **Detection**: Temporal detects the activity heartbeat timeout expiring (heartbeat required for activities > 60s per WORKFLOW-CONTRACT.md). For short activities, Temporal detects the task lease expiring.
- **Impact**: The specific activity is marked as timed out by Temporal. The activity may have partially executed on the target device -- for example, a config push may have been sent but the response was never received by LOOM.
- **Behavior**: Temporal reschedules the activity on another available worker (or the same worker after restart). Because activities carry an `IdempotencyKey` (ADAPTER-CONTRACT.md), the adapter detects the duplicate and returns the previous result rather than re-executing. If the adapter does not support idempotency, the activity re-executes -- the operation must be inherently idempotent or the verification step will catch inconsistencies. Verification runs after re-execution to confirm convergence.
- **Recovery**: Automatic. Temporal handles retry and rescheduling. Operator should investigate the cause of the crash (check OOM killer logs, panic stack traces). If the crash is repeatable, the activity will exhaust its retry budget and the workflow will enter compensation (saga rollback).
- **Data Loss Risk**: None for LOOM state. The target device may be in a partially-configured state, which verification will detect and the workflow will compensate.
- **RTO**: Automatic. Activity retries within seconds. Full recovery depends on activity timeout + retry backoff.

### 2.5 Workflow History Exceeds Size Limit

- **Failure**: A long-running workflow accumulates more than ~10,000 events in its history (e.g., a discovery workflow scanning thousands of devices in a loop without `continue-as-new`).
- **Detection**: Temporal returns `workflow execution history size exceeds limit` error. The workflow is terminated by Temporal.
- **Impact**: The workflow is forcefully terminated. Any in-progress activities complete but their results are not recorded. Compensation does not run (Temporal terminates the workflow, it does not fail it through the normal path).
- **Behavior**: LOOM marks the workflow as `terminated` in the DB projection. An alert is emitted: `loom._system.workflow.history_exceeded`. The underlying infrastructure changes that were already executed are NOT automatically rolled back.
- **Recovery**: Operator must review the terminated workflow to determine what changes were applied. Manually trigger a teardown workflow for any partially-completed operations. Fix the workflow code to implement `continue-as-new` checkpointing (required by WORKFLOW-CONTRACT.md rule 9). Re-submit the workflow.
- **Data Loss Risk**: Medium. The workflow execution history is preserved (it is queryable), but the workflow did not complete its verification step. Device state may not match expectations.
- **RTO**: Manual. Requires workflow code fix and re-submission. Hours to days depending on complexity.

### 2.6 Task Queue Backlog Growing

- **Failure**: The Temporal task queue depth is increasing faster than workers can process tasks. This is not a crash but a capacity problem.
- **Detection**: Temporal metrics: `temporal_task_queue_backlog` increasing. `schedule_to_start_latency` increasing. LOOM monitoring alerts when backlog exceeds threshold (default: 100 tasks) or schedule-to-start latency exceeds 30 seconds.
- **Impact**: Workflows slow down. New activities wait in queue before starting. Approval timeouts may fire before workflows reach the approval step. SLA on workflow completion time is at risk.
- **Behavior**: LOOM continues operating but with degraded throughput. Workflows complete, just slower. No data loss. No incorrect behavior.
- **Recovery**: Scale out LOOM worker processes (add more replicas). Investigate whether a specific workflow type is dominating the queue (a runaway discovery scan, a thundering herd of provisioning requests). If a single workflow is submitting excessive activities, consider cancelling it and re-submitting with better batching.
- **Data Loss Risk**: None.
- **RTO**: Minutes to scale workers. Backlog clearance depends on depth and worker throughput.

---

## 3. NATS JetStream Failures

NATS JetStream is the event backbone for all domain events (device state changes, workflow lifecycle, audit records, telemetry, cost data). It is configured with 3 replicas, file-based storage, and 90-day retention (EVENT-MODEL.md).

### 3.1 NATS Cluster Loses Quorum (2 of 3 Nodes Down)

- **Failure**: Two of three NATS nodes fail. The remaining node cannot form a Raft quorum for JetStream operations.
- **Detection**: NATS client connections to the surviving node succeed (NATS Core still works), but JetStream publish operations return `no suitable peers for placement` or `insufficient replicas`. Health check reports `nats_jetstream: unhealthy`.
- **Impact**: JetStream publishes fail -- domain events are not persisted. NATS Core pub/sub still works for transient messaging (but LOOM does not rely on non-persistent messaging for critical paths). The DB projector stops receiving new events to project. Real-time dashboard updates stop (they rely on NATS events). Workflow execution via Temporal is NOT affected -- Temporal does not depend on NATS. Adapter operations continue. Verification continues. The system is operating but not emitting observable events.
- **Behavior**: LOOM buffers events in memory (bounded buffer, default 10,000 events) and retries publishing with backoff. If the buffer fills, oldest events are dropped and a `loom._system.nats.buffer_overflow` metric is incremented. Workflows continue executing through Temporal. The API continues serving requests. LOOM operates in "blind mode" -- it works, but operators cannot see real-time event streams.
- **Recovery**: Restore at least one additional NATS node to re-establish quorum. Once quorum is restored, JetStream operations resume. Buffered events are flushed. For events that were dropped from the buffer: the DB projector consumer replays from its last acknowledged sequence -- but if events were never published to JetStream, they are lost from the event stream. Run `loom admin events reconcile` to compare Temporal workflow completion records against the event stream and re-publish any missing `workflow.completed` / `workflow.failed` events.
- **Data Loss Risk**: Low to Medium. Events generated during the outage that exceed the in-memory buffer are permanently lost from the NATS stream. However, the authoritative state is in Temporal (execution) and PostgreSQL (projections written directly by activities), not in NATS. NATS events are derived/secondary.
- **RTO**: Minutes once nodes are restored. Quorum recovery is automatic.

### 3.2 Single NATS Node Down

- **Failure**: One of three NATS nodes fails. The cluster retains quorum (2 of 3).
- **Detection**: NATS cluster health shows one node offline. Client connections to that node fail and reconnect to surviving nodes (NATS client auto-reconnect). JetStream operations continue with degraded redundancy (R=2 instead of R=3).
- **Impact**: No operational impact. JetStream continues publishing and consuming. Durability is temporarily reduced (a second node failure would cause quorum loss). Performance may be slightly reduced if the lost node was serving as consumer leader for some streams.
- **Behavior**: Transparent. NATS clients reconnect automatically. No LOOM-level impact.
- **Recovery**: Restore the failed node. It rejoins the cluster and catches up on missed data from the surviving nodes (Raft log replay). No LOOM intervention needed.
- **Data Loss Risk**: None while quorum is maintained.
- **RTO**: Automatic. Zero downtime.

### 3.3 JetStream Storage Full

- **Failure**: The NATS JetStream storage directory reaches its configured `MaxBytes` (50 GB per EVENT-MODEL.md) or the underlying filesystem fills.
- **Detection**: NATS returns `maximum bytes exceeded` on publish. NATS server logs storage errors. LOOM monitoring tracks JetStream storage utilization.
- **Impact**: New events cannot be published. The `DiscardOld` policy (configured in EVENT-MODEL.md) should automatically discard the oldest messages to make room -- if `DiscardOld` is working, this is a non-event. If the filesystem itself is full (not just the stream limit), NATS may crash or enter read-only mode.
- **Behavior**: With `DiscardOld`: oldest events beyond the 90-day retention are automatically removed. New publishes succeed. With filesystem full: same behavior as NATS quorum loss (3.1) -- event publishing fails, LOOM buffers in memory.
- **Recovery**: If `DiscardOld` is functioning: no action needed. If filesystem full: expand storage, clean up old data, or reduce retention period. Review whether telemetry data (often the largest event category) should be moved to a dedicated stream with shorter retention.
- **Data Loss Risk**: Low. `DiscardOld` only removes events beyond retention. If filesystem full, same risk as 3.1.
- **RTO**: Automatic with `DiscardOld`. Minutes for filesystem expansion.

### 3.4 Consumer Falls Behind (Slow Consumer)

- **Failure**: A NATS consumer (e.g., the DB projector) processes messages slower than they are produced. The pending message count grows unbounded.
- **Detection**: Consumer `num_pending` metric increases. `ack_floor` falls behind stream `last_seq`. LOOM monitoring alerts when pending exceeds threshold (default: 10,000 messages) or when consumer lag exceeds 60 seconds.
- **Impact**: Depends on the consumer. DB projector behind: PostgreSQL projections are stale, dashboard shows outdated data, but Temporal workflows and API mutations are unaffected. Alert engine behind: failure alerts are delayed. Tenant notifier behind: webhooks are delayed.
- **Behavior**: JetStream retains messages for the slow consumer (up to stream limits). No data loss. The consumer continues processing at its own rate. LOOM does not back-pressure event publishers.
- **Recovery**: Investigate the slow consumer. Common causes: slow SQL writes (DB projector), slow webhook delivery (tenant notifier), resource starvation (CPU/memory). Scale the consumer horizontally (add replicas with the same consumer group). If the consumer is permanently unable to keep up, increase its `MaxAckPending` to allow more concurrent processing. If historical events are no longer needed, reset the consumer's delivery position with `nats consumer next --reset`.
- **Data Loss Risk**: None. Messages are retained in JetStream.
- **RTO**: Depends on root cause. Scaling is immediate. Backlog clearance depends on depth.

### 3.5 Leaf Node Disconnected from Hub (Edge Partition)

- **Failure**: A NATS leaf node at an edge site loses connectivity to the hub NATS cluster. This is the expected failure mode for edge deployments.
- **Detection**: Leaf node logs `disconnected from hub`. Hub cluster logs `leaf node disconnected`. LOOM central emits `loom._system.edge.partition_detected` with the site identifier.
- **Impact**: The edge site continues operating autonomously. Edge agents continue running workflows via local SQLite and local NATS. Events produced at the edge are stored locally in the leaf node's JetStream. The central LOOM instance cannot see real-time state from the edge site. Central inventory for edge devices becomes stale.
- **Behavior**: Edge agent enters autonomous mode. It continues executing local workflows (discovery, monitoring, remediation) against local devices using its local SQLite database. Events are buffered in the leaf node's local JetStream. When connectivity is restored, the leaf node reconnects and forwards buffered events to the hub. The DB projector processes them and updates central inventory. LOOM explicitly designs for this mode -- edge autonomy is a first-class requirement.
- **Recovery**: Automatic on reconnection. Buffered events are forwarded and processed. Central inventory catches up. Operator should review any actions taken autonomously by the edge agent during the partition. No data loss if the leaf node's local storage did not fill during the partition.
- **Data Loss Risk**: None under normal partition durations. Risk increases if the partition lasts long enough for the leaf node's local JetStream storage to fill -- in that case, `DiscardOld` applies locally.
- **RTO**: Automatic on reconnection. Catchup time depends on event volume during partition.

### 3.6 Message Delivery Timeout

- **Failure**: A NATS consumer does not acknowledge a message within the `AckWait` period (30 seconds per EVENT-MODEL.md). The message is redelivered.
- **Detection**: Consumer `num_redelivered` metric increases. If redeliveries exceed `MaxDeliver` (5), the message is moved to a dead-letter subject (if configured) or dropped.
- **Impact**: Temporary delay in processing. If the message is eventually acknowledged, no impact. If it exceeds `MaxDeliver` and is dropped, that event is lost from the consumer's perspective.
- **Behavior**: JetStream redelivers the message up to `MaxDeliver` times. LOOM consumers are idempotent (EVENT-MODEL.md: "Consumers must be idempotent. Deduplication is by event ID"), so redelivery causes no incorrect behavior -- just redundant processing.
- **Recovery**: If messages are repeatedly timing out, investigate the consumer (likely too slow or stuck). If messages are dropped after max redeliveries, run `loom admin events reconcile` to detect and re-publish missed events.
- **Data Loss Risk**: Low. Only affects a single event at a time. Dead-lettered events can be reprocessed manually.
- **RTO**: Automatic via redelivery. Manual for dead-lettered messages.

---

## 4. Valkey/Redis Failures

Valkey serves as LOOM's caching layer: hot device state, session tokens, real-time dashboard state, and WebSocket fan-out pub/sub. Valkey is explicitly non-authoritative -- it is a performance optimization, not a source of truth.

### 4.1 Valkey Down Entirely

- **Failure**: Valkey process crashes or host is unreachable.
- **Detection**: Connection attempts fail. Health check reports `valkey: unhealthy`. LOOM emits `loom._system.valkey.connection_lost`.
- **Impact**: Cache misses on every read -- all reads fall through to PostgreSQL. Increased PostgreSQL load. API response latency increases (cache miss penalty). Real-time dashboard pub/sub stops -- WebSocket clients stop receiving live updates (they can still poll the API). Session token validation falls back to PostgreSQL-based session lookup (slower but functional).
- **Behavior**: LOOM treats all cache operations as no-ops. `Get` returns cache miss, `Set` silently fails. All data paths fall through to PostgreSQL. The system is slower but fully functional. No workflow impact. No data loss.
- **Recovery**: Restart Valkey. Cache warms organically as requests come in. No explicit cache warming is needed -- the cache is populated lazily. For faster recovery, run `loom admin cache warm` to pre-populate hot device state.
- **Data Loss Risk**: None. Valkey holds no authoritative data.
- **RTO**: < 1 minute for restart. Performance returns to normal as cache warms (minutes).

### 4.2 Valkey Eviction Under Memory Pressure

- **Failure**: Valkey reaches its `maxmemory` limit and begins evicting keys according to its eviction policy (recommended: `allkeys-lfu`).
- **Detection**: Valkey `evicted_keys` metric increases. Cache hit ratio drops. LOOM monitoring alerts when hit ratio falls below threshold (default: 70%).
- **Impact**: Similar to partial cache loss. Frequently accessed keys survive (LFU policy). Rarely accessed device state is evicted and must be re-fetched from PostgreSQL. Slight increase in PostgreSQL load.
- **Behavior**: Transparent. Cache misses fall through to PostgreSQL. No errors. No data loss. LOOM does not distinguish between eviction and TTL expiry.
- **Recovery**: Increase Valkey `maxmemory` or add a Valkey replica for read scaling. Review what is being cached -- if telemetry data is consuming cache space, move it to a separate Valkey instance or reduce its cache TTL.
- **Data Loss Risk**: None.
- **RTO**: Not applicable -- system remains fully operational, just slower for evicted keys.

### 4.3 Cache Poisoning (Stale Data)

- **Failure**: Valkey contains cached data that no longer matches PostgreSQL. This can happen if a PostgreSQL write succeeds but the corresponding cache invalidation fails, or if a cache entry outlives its TTL due to clock skew.
- **Detection**: Users report seeing outdated device state in the dashboard. Verification checks observe a value that differs from what the API returns (the API returns cached data, verification reads live from the adapter). LOOM's cache-vs-DB consistency check (periodic, default every 5 minutes for hot keys) detects mismatches.
- **Impact**: Users see incorrect data. Decisions based on stale cache (e.g., LLM placement decisions using cached capacity data) may be suboptimal. No impact on workflows -- Temporal activities always read from PostgreSQL or adapters directly, never from Valkey.
- **Behavior**: The periodic consistency check identifies stale keys and invalidates them. On next access, the correct value is loaded from PostgreSQL.
- **Recovery**: Flush specific keys with `loom admin cache invalidate --resource <id>` or flush the entire cache with `loom admin cache flush`. The system self-heals via TTL expiry (default 5 minutes for device state, 30 seconds for telemetry).
- **Data Loss Risk**: None. The correct data is always in PostgreSQL.
- **RTO**: Automatic within TTL period. Immediate with manual flush.

---

## 5. Adapter Failures

Adapters bridge LOOM to physical devices via protocols like Redfish, IPMI, NETCONF, gNMI, SSH, SNMP, AMT, PiKVM, and others. Adapter failures are the most common failure category because they depend on external devices, networks, and credentials.

### 5.1 Device Unreachable (Network Issue)

- **Failure**: The target device does not respond to connection attempts. TCP SYN times out. ICMP unreachable received. DNS resolution succeeds but no route to host.
- **Detection**: Adapter's `Connect()` or `Ping()` returns `ADAPTER_UNREACHABLE` (TransientError). Activity records the failure with the target device, protocol, and error detail.
- **Impact**: The specific operation on this device fails. If the operation is part of a multi-device workflow, other devices are unaffected. If the operation is part of a saga, it will be retried per the retry policy (5 attempts, 1s-30s exponential backoff per ERROR-MODEL.md).
- **Behavior**: Temporal retries the activity per `TransientRetryPolicy`. Between retries, the adapter re-attempts connection. If all retries are exhausted, the activity fails permanently. For saga workflows, compensation runs for previously completed steps. LOOM emits `loom.{tenant}.workflow.failed` with `error_code: ADAPTER_UNREACHABLE`. For multi-device operations, LOOM returns a `PartialError` listing which devices succeeded and which failed.
- **Recovery**: Investigate network connectivity to the device. Check routing, firewall rules, and device power state. Once the device is reachable, re-submit the workflow. No LOOM-side recovery needed.
- **Data Loss Risk**: None. LOOM state is consistent. The device is in whatever state it was before the workflow attempted to change it.
- **RTO**: Depends on network resolution. LOOM retries automatically for transient issues (seconds to minutes).

### 5.2 Device Rejects Credentials (Auth Failure)

- **Failure**: The device responds to connection attempts but rejects authentication. Wrong password, expired certificate, locked account, or credential rotation not propagated to LOOM's credential vault.
- **Detection**: Adapter returns `ADAPTER_AUTH_FAILED` (PermanentError). Activity records the failure.
- **Impact**: The operation fails immediately with no retries (auth failure is permanent per ERROR-MODEL.md). If part of a saga, compensation runs.
- **Behavior**: LOOM does not retry (permanent error). The workflow fails at the current step. Compensation runs for completed steps. LOOM emits a `workflow.failed` event with `error_code: ADAPTER_AUTH_FAILED`. The alert engine notifies operators with the device identifier and credential vault path (but never the credential itself -- ERROR-MODEL.md rule 5).
- **Recovery**: Update credentials in the credential vault. Verify connectivity with `loom admin adapter test --device <id> --protocol <protocol>`. Re-submit the workflow.
- **Data Loss Risk**: None.
- **RTO**: Minutes once credentials are updated.

### 5.3 Device Responds with Unexpected Data (Protocol Error)

- **Failure**: The device accepts the connection and authentication, but returns data that the adapter cannot parse. Examples: a Redfish endpoint returns HTML instead of JSON, a NETCONF response has invalid XML, an SNMP walk returns unexpected OIDs, a firmware version changes the API schema.
- **Detection**: Adapter returns `ADAPTER_UNSUPPORTED_OP` (PermanentError) or a `TransientError` if the parse failure might be intermittent (corrupt response on a noisy link). The adapter logs the raw response for debugging.
- **Impact**: The specific operation fails. If this is a discovery operation, the device is marked as `discovered_but_incompatible` or `unknown_schema`.
- **Behavior**: If classified as transient: retry with backoff. If classified as permanent: fail immediately, trigger compensation. LOOM's LLM engine may be consulted for device classification if the standard adapter cannot parse the response (LLM-BOUNDARIES.md use case: device classification with 0.7 confidence threshold). The LLM's deterministic fallback for classification is `sysObjectID` / Redfish schema lookup.
- **Recovery**: Investigate the device's firmware version and API behavior. The adapter may need an update to handle the new response format. File an issue against the adapter. Workaround: use a different protocol adapter for this device if available (e.g., fall back from Redfish to IPMI for power operations).
- **Data Loss Risk**: None.
- **RTO**: Depends on whether an alternative adapter/protocol exists. Immediate if fallback protocol is available. Days if adapter code needs updating.

### 5.4 Adapter Timeout (Device Slow to Respond)

- **Failure**: The device accepts the connection but does not respond within the activity timeout. Common with BMC interfaces under load, old switches with slow CLI, or large SNMP walks.
- **Detection**: Context deadline exceeded. Adapter returns `ADAPTER_TIMEOUT` (TransientError). Activity timeout fires in Temporal.
- **Impact**: The operation fails for this attempt. Temporal retries per `AdapterTimeoutRetryPolicy` (3 attempts, 5s-60s backoff per ERROR-MODEL.md).
- **Behavior**: Retry with increasing timeout tolerance. If the device is consistently slow, all retries will fail. The activity fails permanently. Saga compensation runs. Operator is alerted.
- **Recovery**: Investigate why the device is slow. Common causes: BMC under load (reduce concurrent operations), switch with large config (increase timeout), network latency (check path). If the device is inherently slow, increase the activity timeout for this device type in the workflow configuration. Re-submit the workflow.
- **Data Loss Risk**: None. Timeout means the response was never received, so no partial state change is applied (for stateless queries). For state-changing operations (config push), the device may have applied the change but the response was lost -- verification will detect whether the change was applied.
- **RTO**: Automatic for transient slowness. Manual investigation for persistent timeouts.

### 5.5 Adapter Crash (Panic in Go)

- **Failure**: The adapter code panics due to a nil pointer dereference, index out of bounds, or other unrecoverable error. This crashes the Temporal worker process (or the goroutine running the activity, if recovered by a `defer recover()`).
- **Detection**: If the panic crashes the worker: Temporal detects the heartbeat/lease timeout and reschedules the activity (same as 2.4). If the panic is recovered by `defer recover()`: the activity returns a `TransientError` wrapping the panic message.
- **Impact**: The specific activity fails. If it was a worker crash, the entire worker process restarts and all activities on that worker are rescheduled.
- **Behavior**: Temporal reschedules the activity. If the panic is deterministic (always happens for this input), every retry will panic, exhausting the retry budget. The activity fails permanently. Saga compensation runs. The panic stack trace is logged with the correlation ID for debugging.
- **Recovery**: Fix the adapter bug. The panic stack trace in the logs identifies the exact failure point. The correlation ID links to the specific workflow and device that triggered the panic. Deploy the fix and re-submit the workflow.
- **Data Loss Risk**: None for LOOM state. The target device may be in a partially-modified state if the panic occurred after sending a command but before receiving the response. Verification detects this.
- **RTO**: Automatic for non-deterministic panics (retry succeeds). Manual for deterministic panics (requires code fix).

### 5.6 Protocol Mismatch (Expected Redfish, Got IPMI-Only)

- **Failure**: LOOM attempts to use a protocol (e.g., Redfish) against a device that only supports a different protocol (e.g., IPMI). This happens when device inventory is incorrect, or when a device is replaced with a different model.
- **Detection**: Adapter `Connect()` fails with `ADAPTER_UNREACHABLE` (port not open) or `ADAPTER_UNSUPPORTED_OP` (connection succeeds but the protocol handshake fails -- e.g., HTTPS connects but the response is not a Redfish service root).
- **Impact**: The operation fails for this protocol. If LOOM's device record lists alternative protocols for this device, LOOM can automatically fall back to an alternative adapter.
- **Behavior**: The activity fails. If the device has multiple protocol registrations in its capability profile, the workflow logic selects the next available protocol and retries with the alternative adapter. If no alternative protocol is available, the activity fails permanently. LOOM's discovery workflow can be triggered to re-scan the device and update its protocol capabilities. The LLM classification engine may be invoked to determine the correct protocol based on available information.
- **Recovery**: Run a discovery workflow against the device to update its protocol capabilities. Update the device record if it was replaced. Re-submit the original workflow.
- **Data Loss Risk**: None.
- **RTO**: Minutes if auto-fallback to an alternative protocol works. Hours if discovery and manual record update are needed.

---

## 6. LLM Failures

The LLM engine is provider-agnostic and uses a tiered approach: cloud APIs (Claude, GPT), local models (Ollama, vLLM, llama.cpp), and deterministic fallbacks. Every path works without an LLM (LLM-BOUNDARIES.md rule 5).

### 6.1 Cloud LLM API Unreachable (Claude API Down)

- **Failure**: The cloud LLM API (Anthropic Claude, OpenAI, etc.) returns errors, times out, or is unreachable (DNS failure, network issue, API outage).
- **Detection**: LLM client receives HTTP 5xx, connection timeout, or DNS resolution failure. LOOM emits `LLM_UNAVAILABLE` (TransientError). Health check reports `llm_cloud: unhealthy`.
- **Impact**: LLM-enhanced features degrade to fallback behavior. The LLM Router automatically fails over to local models (if configured) or to deterministic fallbacks. No operations are blocked -- every LLM-dependent path has a non-LLM fallback.
- **Behavior**: LLM Router detects the failure and routes requests to the next provider in priority order: cloud API (failed) -> local model (Ollama/vLLM) -> deterministic fallback. The fallback hierarchy is: classification uses sysObjectID/schema lookup, placement uses rule-based capacity + affinity, config generation uses Go templates, oversight detection uses checklist validation (LLM-BOUNDARIES.md). LOOM logs the fallback activation and emits `loom._system.llm.fallback_activated`.
- **Recovery**: Automatic when the cloud API recovers. The LLM Router periodically probes the failed provider (default every 60 seconds) and re-enables it when healthy. No manual intervention needed.
- **Data Loss Risk**: None. Fallback results may be less optimal (e.g., template-based config instead of LLM-generated config) but are always correct.
- **RTO**: Automatic failover within seconds. Quality returns to full when cloud API recovers.

### 6.2 Local LLM (vLLM) Process Crash

- **Failure**: The local LLM serving process (vLLM, Ollama, llama.cpp server) crashes, runs out of GPU memory, or becomes unresponsive.
- **Detection**: LLM client receives connection refused or timeout from the local endpoint. Health check reports `llm_local: unhealthy`.
- **Impact**: If cloud LLM is available, traffic routes there (with higher latency and cost). If no cloud LLM is configured (air-gapped deployment), all LLM requests fall back to deterministic fallbacks.
- **Behavior**: LLM Router fails over to the next available provider. In air-gapped mode, deterministic fallbacks activate for all LLM use cases. Operations continue at reduced intelligence -- decisions are rule-based rather than LLM-reasoned.
- **Recovery**: Restart the local LLM process. Check GPU memory availability. If OOM, reduce model size (switch to a smaller quantization) or batch size. The LLM Router re-enables the local provider when it responds to health probes.
- **Data Loss Risk**: None.
- **RTO**: < 5 minutes for process restart. Automatic failover during restart.

### 6.3 LLM Returns Invalid/Unparseable Output

- **Failure**: The LLM returns a response that does not conform to the expected Go struct schema (LLM-BOUNDARIES.md rule 3). JSON parse failure, missing required fields, or malformed config syntax.
- **Detection**: Response parsing fails. The validation layer rejects the response. LOOM logs the raw response for debugging.
- **Impact**: The specific LLM request fails. The operation falls back to deterministic logic for this single request.
- **Behavior**: LOOM retries the LLM request once with a rephrased/reinforced prompt (including an explicit example of the expected output format). If the second attempt also fails, LOOM uses the deterministic fallback. The `LLMRecommendation.Fallback` field (LLM-BOUNDARIES.md) provides the pre-computed deterministic alternative. LOOM logs the parse error with the raw LLM response and the expected schema for debugging.
- **Recovery**: Automatic. The deterministic fallback produces a valid result. If a specific LLM model consistently produces unparseable output, update the model configuration (switch to a different model or adjust the prompt template).
- **Data Loss Risk**: None.
- **RTO**: Automatic. Seconds.

### 6.4 LLM Confidence Below Threshold

- **Failure**: The LLM returns a valid, parseable response but with a confidence score below the configured threshold for the operation type (e.g., < 0.9 for production config generation, < 0.7 for device classification per LLM-BOUNDARIES.md).
- **Detection**: `LLMRecommendation.Confidence` is below the threshold. LOOM returns `LLM_CONFIDENCE_LOW` (PermanentError per ERROR-MODEL.md -- requires human review, not retry).
- **Impact**: The operation is escalated to a human operator for review. The workflow pauses at a human-in-the-loop approval gate with the LLM's recommendation, its confidence score, its reasoning, and the deterministic fallback alternative presented side-by-side.
- **Behavior**: If confidence is very low (< 0.4): LOOM uses the deterministic fallback without human review. If confidence is moderate but below threshold (0.4 - threshold): LOOM presents the LLM recommendation to a human via an approval signal. The workflow blocks on the approval signal (Temporal signal, not polling per WORKFLOW-CONTRACT.md rule 7). The approval request includes the LLM's reasoning, confidence, provenance, and the deterministic alternative.
- **Recovery**: A human reviews the recommendation and either approves it, modifies it, or selects the deterministic alternative. The workflow resumes with the human's decision.
- **Data Loss Risk**: None.
- **RTO**: Depends on human response time. The approval request has a configurable timeout (default 24 hours) after which the workflow auto-rejects and uses the deterministic fallback.

### 6.5 LLM Rate Limited

- **Failure**: The cloud LLM API returns HTTP 429 (rate limited) or equivalent.
- **Detection**: LLM client receives 429 with a `Retry-After` header. LOOM classifies this as `LLM_UNAVAILABLE` (TransientError).
- **Impact**: LLM requests are delayed. If rate limiting persists, the LLM Router shifts load to alternative providers (other cloud APIs, local models).
- **Behavior**: Retry per `LLMRetryPolicy` (3 attempts, 10s-2min backoff per ERROR-MODEL.md). Respect the `Retry-After` header. If all retries are exhausted, fail over to the next provider. If no providers are available, use deterministic fallback. LOOM tracks rate limit events to optimize request batching and caching (semantic cache).
- **Recovery**: Automatic. Rate limits are transient. If persistent, increase the API rate limit tier, batch LLM requests more aggressively, or rely more on local models.
- **Data Loss Risk**: None.
- **RTO**: Seconds to minutes (automatic).

### 6.6 LLM Hallucinates Invalid Configuration

- **Failure**: The LLM generates a configuration that is syntactically valid (passes JSON parsing) but semantically incorrect -- e.g., a VLAN ID outside the valid range, an IP address that conflicts with an existing allocation, a firewall rule that would block management access.
- **Detection**: The hallucination prevention pipeline (TECH-STACK.md) catches this at one of several gates: JSON schema validation, vendor-specific config syntax check, Batfish-style offline verification, or dry-run/sandbox execution. Each gate is deterministic -- the LLM is never trusted to validate its own output.
- **Impact**: The invalid configuration is rejected before it reaches any device. The workflow does not execute the invalid config. No device state is affected.
- **Behavior**: LOOM logs the invalid config with the specific validation failure for model improvement. The workflow retries with the LLM (different prompt or temperature) once. If the second attempt also produces invalid config, LOOM falls back to template-based config generation (deterministic, guaranteed valid). If the operation is production-critical (config generation threshold 0.9), it is escalated for human review regardless.
- **Recovery**: Automatic via deterministic fallback. For long-term improvement: the rejected config is stored for fine-tuning data and prompt improvement.
- **Data Loss Risk**: None. The hallucination never reaches a device.
- **RTO**: Automatic. Seconds.

---

## 7. Network Partitions

### 7.1 Hub Loses Connectivity to Edge Site

- **Failure**: The WAN link between the central LOOM hub and a remote edge site goes down. The edge site's NATS leaf node disconnects from the hub cluster.
- **Detection**: NATS leaf node disconnection event at the hub. LOOM central emits `loom._system.edge.partition_detected` with site ID. Hub-side health dashboard shows the edge site as `partitioned`.
- **Impact**: Central LOOM cannot manage devices at the edge site. Edge device inventory in the central DB becomes stale. Workflows targeting edge devices cannot be started from the hub. However, the edge agent continues operating autonomously.
- **Behavior**: Edge agent enters autonomous mode (this is by design, not degradation). It continues: local discovery scans, health monitoring, event buffering in local NATS/SQLite, and executing pre-approved remediation workflows. The edge agent does NOT execute new provisioning or teardown workflows without central approval (safety constraint). When connectivity is restored, the leaf node reconnects and forwards all buffered events. The central DB projector processes them to update inventory. Any workflows that were queued for the edge site during the partition are dispatched.
- **Recovery**: Automatic on reconnection. Operator should review edge-agent autonomous actions during the partition via `loom admin edge audit --site <id> --since <partition_start>`. Reconcile any divergent state.
- **Data Loss Risk**: None if edge storage did not fill.
- **RTO**: Automatic on reconnection. Catchup time proportional to partition duration and event volume.

### 7.2 Edge Site Loses All External Connectivity

- **Failure**: The edge site is completely isolated -- no WAN, no internet, no connection to the hub. This is more severe than 7.1 because the edge site may also lose connectivity to cloud LLM APIs.
- **Detection**: Edge agent detects all outbound connections failing. It logs the partition event locally.
- **Impact**: Same as 7.1 for the hub. At the edge: cloud LLM is unavailable, so the edge agent uses local models or deterministic fallbacks. If the edge agent's local model is also unavailable (no GPU, no local model configured), it operates on deterministic logic only. Credential rotation from a central vault is unavailable -- the edge agent uses cached credentials.
- **Behavior**: Edge agent continues with maximum autonomy within safety constraints. It uses local SQLite for state, local NATS for events, local models or deterministic logic for decisions, and cached credentials for device access. It buffers all events and state changes for eventual sync.
- **Recovery**: Restore connectivity. Same as 7.1. Additionally, verify credentials are not expired if the partition was long enough for credential rotation to have occurred at the hub.
- **Data Loss Risk**: Low. Risk increases with partition duration (local storage fill, credential expiry).
- **RTO**: Automatic on reconnection. May require credential re-sync.

### 7.3 Split Brain Between NATS Nodes

- **Failure**: A network partition divides the NATS cluster such that each partition believes it is the active cluster. This can occur if all three nodes are in different failure domains and a partition isolates one from the other two.
- **Detection**: NATS Raft consensus prevents true split brain -- the partition with quorum (2 of 3) continues as the active cluster. The isolated node detects it has lost quorum and stops serving JetStream writes. NATS clients on the isolated node's network receive JetStream errors and reconnect to reachable nodes.
- **Impact**: The minority partition (1 node) is read-only for JetStream. LOOM clients connected to that node cannot publish events but can reconnect to the majority partition. The majority partition operates normally.
- **Behavior**: NATS Raft protocol prevents data divergence. The isolated node does not accept writes. LOOM clients auto-reconnect to the majority partition. No split-brain data corruption occurs.
- **Recovery**: Resolve the network partition. The isolated node rejoins the cluster and syncs from the Raft log.
- **Data Loss Risk**: None. Raft consensus prevents divergent writes.
- **RTO**: Automatic once partition heals. Client reconnection takes seconds.

### 7.4 DNS Resolution Failure

- **Failure**: DNS resolution fails for LOOM's service endpoints (PostgreSQL hostname, NATS hostname, Temporal hostname, LLM API hostname).
- **Detection**: Connection attempts fail with `no such host` or `temporary failure in name resolution`. Health checks for affected services fail.
- **Impact**: Depends on which service's DNS fails. If PostgreSQL DNS fails: equivalent to 1.1. If NATS DNS fails: equivalent to 3.1. If all DNS fails: LOOM is fully degraded. Edge agents using IP addresses directly (common for on-prem) are unaffected.
- **Behavior**: LOOM retries DNS resolution with backoff. If using service discovery (Consul, Kubernetes DNS), the retry includes fallback resolvers. LOOM caches DNS results with a TTL to survive brief DNS outages.
- **Recovery**: Restore DNS service. If using static hosts (common for on-prem), configure `/etc/hosts` fallbacks. For production deployments, use multiple DNS resolvers.
- **Data Loss Risk**: None. DNS failure prevents connections but does not corrupt data.
- **RTO**: Immediate once DNS is restored. LOOM uses cached DNS entries during brief outages.

---

## 8. Cascading Failures

### 8.1 PostgreSQL Down + Active Workflows

- **Failure**: PostgreSQL goes down while Temporal workflows are actively executing.
- **Detection**: Both LOOM health check and Temporal health check report database failures simultaneously.
- **Impact**: This is the most impactful single-component failure because both Temporal and LOOM use PostgreSQL. Temporal cannot persist workflow state, so all workflows freeze. LOOM cannot update projections. API cannot serve read queries. The entire system is inoperable.
- **Behavior**: All in-flight activities that are currently executing against devices continue to completion (the activity code does not depend on PostgreSQL mid-execution -- it calls the adapter, gets a result, and then returns to Temporal). However, Temporal cannot record the activity result. When the activity completes, the worker cannot report the result. The activity is marked as timed out by Temporal after the heartbeat/lease expires. When PostgreSQL recovers, Temporal replays the workflow history. The activity that completed during the outage is re-dispatched. Because adapters honor the `IdempotencyKey`, the re-execution returns the cached result without re-applying the change to the device.
- **Recovery**: Restore PostgreSQL. Temporal and LOOM reconnect. Workflows resume from their persisted state. Activities that were in-flight are re-dispatched (idempotent). Run `loom admin db check` to verify projection consistency. Monitor for duplicate events in NATS (consumers are idempotent so this is safe).
- **Data Loss Risk**: None if PostgreSQL data is intact. Activities that completed during the outage are re-executed safely due to idempotency.
- **RTO**: Equal to PostgreSQL recovery time + minutes for backlog clearance.

### 8.2 NATS Down + Temporal Running

- **Failure**: NATS is completely down while Temporal continues executing workflows.
- **Detection**: Event publishing fails. Health check reports `nats: unhealthy`. Temporal health check remains healthy.
- **Impact**: Workflows continue executing through Temporal. Adapters continue operating against devices. Verification continues. However: domain events are not published, so the DB projector does not receive events, the alert engine does not receive failure events, real-time dashboard updates stop, and edge sites do not receive central events. The system is "blind" -- it works, but nobody can observe it in real time.
- **Behavior**: LOOM buffers events in memory (bounded buffer). Workflows complete successfully (Temporal does not depend on NATS). DB projections that are written directly by activities (via SQL) continue working. Only event-driven projections (via NATS consumers) fall behind. The UI shows a `real-time updates unavailable` warning but can still query PostgreSQL via API.
- **Recovery**: Restore NATS. Buffered events are flushed. NATS consumers catch up from their last acknowledged position. Run `loom admin events reconcile` to detect any events that were dropped from the in-memory buffer.
- **Data Loss Risk**: Low. Events that overflowed the in-memory buffer are lost from the NATS stream, but the authoritative state is in Temporal and PostgreSQL.
- **RTO**: Minutes after NATS recovery. Catchup time depends on event volume during outage.

### 8.3 LLM Down + Pending Approval Workflows

- **Failure**: All LLM providers (cloud and local) are down while workflows are waiting at approval gates that require LLM-generated context (e.g., risk assessment, cost estimate, config review).
- **Detection**: LLM Router reports all providers unhealthy. Pending approval requests that need LLM-generated context cannot be enriched.
- **Impact**: Approval requests are sent to humans without LLM-enhanced context. The approval request contains all raw data (device facts, proposed changes, workflow parameters) but lacks the LLM's risk assessment, cost estimate, and oversight analysis. Humans can still make informed decisions -- just with less synthesized context.
- **Behavior**: LOOM sends the approval request with a `llm_context_unavailable: true` flag. The approval UI shows a warning: "LLM analysis unavailable -- review raw data." The workflow remains paused at the approval gate (Temporal signal wait). The human can approve, reject, or defer. If they defer, the workflow waits until the LLM recovers and the approval is re-enriched.
- **Recovery**: LLM recovery triggers re-enrichment of pending approval requests. Humans are notified that enhanced context is now available.
- **Data Loss Risk**: None.
- **RTO**: Depends on human response. If humans approve without LLM context, no delay. If they wait for LLM recovery, RTO equals LLM recovery time.

### 8.4 Multiple Adapter Failures Simultaneously (Thundering Herd)

- **Failure**: Many adapters fail at the same time. Common causes: upstream switch failure (all devices behind it unreachable), credential rotation affecting all devices, DHCP scope exhaustion, or a network-wide event.
- **Detection**: Adapter error rate spikes across multiple devices. LOOM's circuit breaker triggers when the failure rate for a specific adapter type or network segment exceeds a threshold (default: 50% of operations failing within a 60-second window).
- **Impact**: Many workflows fail simultaneously. Saga compensation runs for all affected workflows. If compensation itself requires adapter connectivity (and the adapters are also failing), compensation fails. Temporal task queue fills with retry attempts. The system is operationally overloaded.
- **Behavior**: LOOM's circuit breaker opens for the affected adapter type or network segment. While the circuit is open, new operations against affected devices are rejected immediately with `ADAPTER_UNREACHABLE` instead of attempting connection (fail-fast). This prevents resource exhaustion from thousands of concurrent timeout waits. The circuit breaker probes periodically (default every 30 seconds) with a single canary operation. When the canary succeeds, the circuit transitions to half-open (allowing limited traffic) and eventually closes. Workflows that were in compensation and could not reach their devices are marked as `compensation_incomplete` and an operator alert fires.
- **Recovery**: Identify the root cause (upstream switch, credential issue, network event). Resolve it. The circuit breaker re-enables traffic as probes succeed. For workflows with incomplete compensation: operator reviews with `loom admin workflows --state compensation_incomplete` and either re-runs compensation or manually reconciles device state.
- **Data Loss Risk**: Low for LOOM state. Medium for device state consistency -- devices may be in partially-configured states if compensation could not complete.
- **RTO**: Depends on root cause. LOOM self-heals via circuit breaker once devices are reachable. Manual review needed for incomplete compensation.

---

## 9. Operational Failures

### 9.1 Disk Full on Any Component

| Component | Detection | Impact | Recovery |
|-----------|-----------|--------|----------|
| **PostgreSQL** | See 1.4 | All writes fail. Full outage. | Free space. See 1.4. |
| **Temporal** | Temporal uses PostgreSQL -- see 1.4. If separate DB, same pattern. | Workflow state cannot be persisted. | Free space on Temporal's DB volume. |
| **NATS** | See 3.3 | JetStream cannot persist new events. `DiscardOld` activates. | Expand storage or reduce retention. |
| **Valkey** | Valkey is memory-only (RDB/AOF to disk for persistence). Disk full prevents RDB/AOF writes but in-memory operations continue. | Background save fails. Valkey continues serving from memory. Risk of data loss on restart (no snapshot). | Free disk space. Trigger manual `BGSAVE`. |
| **LOOM binary** | Log volume fills disk. LOOM process may fail to write logs and continue running, or crash depending on log configuration. | Depends on log-on-failure behavior. | Rotate logs. Configure log retention. |

- **Data Loss Risk**: Varies. See individual component sections.
- **RTO**: Minutes for log rotation or volume expansion. Hours for major storage issues.

### 9.2 OOM Kill on Any Process

| Process | Detection | Impact | Behavior | Recovery |
|---------|-----------|--------|----------|----------|
| **LOOM API** | Process exits with signal 9. Container/systemd restarts it. Health check fails. | API requests fail during restart. Temporal workflows continue (workers may be separate processes). | Auto-restart via supervisor. | Investigate memory leak. Increase memory limit. Check for unbounded caches or goroutine leaks (`pprof`). |
| **Temporal Worker** | See 2.4. Temporal reschedules activities. | In-flight activities are rescheduled. | Automatic via Temporal retry. | Same as above. |
| **Temporal Server** | Process exits. Workflows freeze. | See 2.1. | Auto-restart. | Increase memory. Check for large workflow histories consuming History service memory. |
| **NATS Server** | Process exits. JetStream unavailable. | See 3.1 (single node) or 3.2 (if cluster). | Auto-restart. Rejoins cluster. | Increase memory. Check for large messages or many consumers. |
| **Valkey** | Process exits. Cache lost. | See 4.1. | Auto-restart. Cache is empty (warm from scratch). | Increase `maxmemory`. Use `allkeys-lfu` eviction. |
| **Local LLM** | Process exits. GPU memory released. | See 6.2. | Auto-restart or failover. | Reduce model size. Lower batch size. Check for GPU memory fragmentation. |

- **Data Loss Risk**: None for stateful services (PostgreSQL WAL, Temporal event replay). Valkey loses cached data.
- **RTO**: < 1 minute if auto-restart is configured. Longer if OOM is recurring (root cause must be fixed).

### 9.3 Clock Skew Between Nodes

- **Failure**: System clocks on different nodes diverge by more than a few seconds. This affects Temporal (lease timeouts, activity deadlines), NATS (message deduplication windows), PostgreSQL (statement timeouts, `now()` comparisons), and TLS certificate validation.
- **Detection**: NTP monitoring shows drift exceeding threshold (default: 500ms). Temporal logs warnings about clock skew. NATS deduplication may fail (accepting duplicate messages or rejecting valid ones).
- **Impact**: With moderate skew (< 5s): Temporal activity timeouts may fire early or late. NATS deduplication window (5 minutes per EVENT-MODEL.md) tolerates this. TLS is unaffected. With severe skew (> 30s): Temporal workflows may behave incorrectly (timers fire at wrong times). TLS certificates may appear expired. NATS Raft leader election may be affected.
- **Behavior**: LOOM does not compensate for clock skew -- it expects NTP to keep clocks synchronized. If skew is detected, LOOM emits `loom._system.clock.skew_detected` and logs a warning.
- **Recovery**: Fix NTP configuration. Restart `chronyd`/`ntpd`. Verify synchronization with `chronyc tracking` or `ntpq -p`. After clocks are synchronized, no LOOM restart is needed.
- **Data Loss Risk**: None directly. Clock skew can cause incorrect timeout behavior which may lead to unnecessary activity retries or premature workflow failures, but no data corruption.
- **RTO**: Minutes to synchronize clocks. No LOOM restart needed.

### 9.4 TLS Certificate Expiration

- **Failure**: A TLS certificate used by a LOOM component expires. This can be: LOOM's API server certificate, PostgreSQL's server certificate, NATS inter-node TLS, Temporal's mTLS, or an adapter certificate for device communication.
- **Detection**: TLS handshake fails with `certificate has expired`. LOOM's certificate monitoring alerts 30 days, 7 days, and 1 day before expiry. On expiry: health checks fail for affected connections.
- **Impact**: Depends on which certificate expired. API server cert: all HTTPS clients get errors. PostgreSQL cert: LOOM and Temporal cannot connect. NATS cert: inter-node communication fails (possible quorum loss). Adapter cert: specific device communications fail.
- **Behavior**: LOOM treats certificate errors as connection failures and follows the relevant component's failure behavior (1.1, 2.1, 3.1, or 5.1 depending on which certificate expired).
- **Recovery**: Renew the certificate. Reload the service (most services support certificate reload without restart via SIGHUP or API call). If using cert-manager or ACME, investigate why auto-renewal failed.
- **Data Loss Risk**: None.
- **RTO**: Minutes if a new certificate can be issued quickly. Hours if CA infrastructure is involved.

### 9.5 Credential Rotation Failure

- **Failure**: LOOM's credential vault rotates device credentials (passwords, SSH keys, API tokens), but the new credentials are not propagated to all components or edge agents. Some LOOM components use old credentials (which now fail) while others use new ones.
- **Detection**: Adapter operations start failing with `ADAPTER_AUTH_FAILED` for devices that were previously accessible. The failure correlates with a recent credential rotation event.
- **Impact**: Operations against affected devices fail. Workflows fail at adapter steps. Partial failures across the fleet if rotation was phased.
- **Behavior**: LOOM's credential management emits `loom._system.credentials.rotation_completed` and `loom._system.credentials.rotation_failed` events. When adapters encounter auth failures after a rotation, LOOM checks whether the credential was recently rotated and attempts with both the new and previous credential (credential grace period). If both fail, the device is marked as `auth_failed` and an operator alert fires.
- **Recovery**: Verify which credential version each component is using. Re-push the current credential to edge agents and components that missed the rotation. Test with `loom admin adapter test --device <id>`. If the device itself was not updated with the new credential, roll back to the previous credential and re-attempt the rotation.
- **Data Loss Risk**: None. Auth failures prevent operations; they do not corrupt state.
- **RTO**: Minutes if the issue is propagation delay. Hours if the rotation partially applied to devices and needs manual reconciliation.

---

## 10. Design Principles: LOOM's Failure Philosophy

These principles are not aspirational -- they are design constraints enforced in code review, architecture review, and testing.

### 10.1 Fail Loud, Not Silent

Every failure emits a structured event. No component silently swallows errors, drops messages, or degrades without announcing it. Specifically:

- Every adapter error is classified as `TransientError`, `PermanentError`, `PartialError`, or `ValidationError` (ERROR-MODEL.md). No bare `error` strings cross component boundaries.
- Every component failure triggers a `loom._system.{component}.{failure_type}` event on NATS (or logs if NATS itself is down).
- Health check endpoints (`/healthz`) report per-component status. A degraded component shows as `unhealthy` immediately, not after an accumulation window.
- The UI shows a degradation banner the moment any component is unhealthy, with specific context about what is affected.

### 10.2 Degrade Gracefully

Losing a component reduces capability, it does not halt the system. The degradation hierarchy:

| Component Lost | What Stops | What Continues |
|---------------|-----------|----------------|
| Valkey | Cached reads, real-time WebSocket updates | Everything else (reads fall through to PostgreSQL) |
| NATS | Real-time events, event-driven projections | Workflows (Temporal), API reads (PostgreSQL), adapter operations |
| LLM (all providers) | Enhanced reasoning, smart placement, config generation | All operations via deterministic fallbacks (templates, rules, checklists) |
| Single Temporal service | Depends on service (see section 2.2) | Other services continue |
| PostgreSQL | Almost everything (see 1.1) | Edge agents (local SQLite), in-flight adapter operations |
| All infrastructure | Central operations | Edge agents operate autonomously on local state |

### 10.3 Never Corrupt State

LOOM uses multiple mechanisms to prevent state corruption:

- **PostgreSQL transactions**: All multi-table writes are atomic. No partial writes to the device inventory.
- **Temporal event sourcing**: Workflow state is reconstructed from the event history, not read from a mutable record. Corruption of a workflow's in-memory state is corrected by replay.
- **Temporal as source of truth (ADR-008)**: If PostgreSQL projections diverge from Temporal's event history, Temporal wins. Projections can be rebuilt from Temporal.
- **NATS JetStream deduplication**: Duplicate event publishing (from retries) is detected and ignored via the `Nats-Msg-Id` header.
- **Adapter idempotency**: Every adapter operation carries an `IdempotencyKey`. Re-execution of the same operation returns the cached result without re-applying the change.
- **Verification**: Every side-effecting workflow step has a mandatory verification activity. State corruption on devices is detected by comparing observed state against expected state.

### 10.4 Edge Autonomy

Edge agents are designed to operate without central connectivity indefinitely:

- Local state in SQLite provides device inventory and workflow context.
- Local NATS leaf node (or embedded NATS) provides event persistence.
- Local LLM (or deterministic fallback) provides decision-making.
- Cached credentials provide device access.
- Safety constraints prevent destructive operations (provisioning, teardown) without central approval.
- On reconnection, all locally-generated events are forwarded to the hub and reconciled.

### 10.5 Human Escalation

When automated recovery fails, LOOM escalates to operators with full context:

- **What failed**: Component, error code, error message, correlation ID.
- **What was affected**: Which workflows, which devices, which tenants.
- **What LOOM already tried**: Retry count, alternative providers attempted, fallback results.
- **What the operator should do**: Specific recovery steps from this document.
- **How to verify recovery**: Commands to run after the fix is applied.

Escalation is via the configured notification channels (webhook, email, PagerDuty, Slack) with structured payloads that include all the above fields. Operators never receive a bare "something failed" alert.

---

## Summary: Recovery Runbook Quick Reference

| Failure | Automatic Recovery? | Operator Action | RTO |
|---------|---------------------|-----------------|-----|
| PostgreSQL primary down (standalone) | No | Restart PostgreSQL, run `loom admin db check` | < 5 min |
| PostgreSQL primary down (Patroni) | Yes (failover) | Verify replication, replace failed node | 10-30s |
| Temporal all services down | No | Restart Temporal services | < 5 min |
| Temporal single service down | No | Restart affected service | < 2 min |
| Worker crash mid-activity | Yes (Temporal retry) | Investigate crash cause | Seconds |
| NATS quorum loss | No | Restore nodes | Minutes |
| NATS single node down | Yes (cluster) | Replace node | Zero downtime |
| Valkey down | Yes (fallback to PG) | Restart Valkey | < 1 min |
| Adapter unreachable | Yes (retry) | Investigate network | Seconds-minutes |
| Adapter auth failure | No | Update credentials | Minutes |
| LLM cloud down | Yes (failover to local) | None | Seconds |
| LLM all providers down | Yes (deterministic fallback) | Restore LLM | Seconds |
| LLM hallucination | Yes (validation pipeline) | None | Seconds |
| Edge partition | Yes (autonomous mode) | Review edge actions on reconnect | Automatic |
| Thundering herd | Yes (circuit breaker) | Investigate root cause | Depends |
| Disk full | No | Free space | Minutes-hours |
| OOM kill | Yes (auto-restart) | Investigate leak, increase limits | < 1 min |
| Clock skew | No | Fix NTP | Minutes |
| TLS cert expiry | No | Renew certificate | Minutes |
| Credential rotation failure | Partial (grace period) | Re-push credentials | Minutes |
