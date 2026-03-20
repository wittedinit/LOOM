Now I have enough research to compile the comprehensive analysis. Here it is:

---

# LOOM Workflow Engine Analysis: Comprehensive Evaluation

## Executive Summary

This analysis evaluates seven workflow engine options for LOOM -- a universal infrastructure orchestrator requiring multi-step orchestration, cross-domain rollback, LLM-driven dynamic workflows, human-in-the-loop approval, and long-running durability. The evaluation considers fit against LOOM's twelve core requirements, deployment constraints (air-gapped/on-prem), language compatibility (Go/Python), and scalability to 1000+ concurrent workflows.

---

## 1. Temporal

**What it is:** A durable execution platform originally forked from Uber's Cadence. Workflows are written as ordinary code (Go, Python, Java, TypeScript). State is persisted via event history replay -- if a worker crashes, the workflow replays its event history and resumes exactly where it left off.

**Architecture:** A central Temporal Service coordinates workflow execution. Workers are stateless processes that poll task queues. All state lives in a persistent data store (PostgreSQL, MySQL, or Cassandra). The service itself is composed of four internal services: Frontend, History, Matching, and Worker.

### Fit Against LOOM Requirements

| Requirement | Fit | Notes |
|---|---|---|
| Multi-step orchestration (100+ steps) | Excellent | Workflows are unbounded sequences of activities. Child workflows handle decomposition. |
| Cross-domain dependencies | Excellent | Native -- activity results feed into subsequent activity inputs, all in code. |
| Parallel execution | Excellent | `workflow.Go()` goroutines in Go SDK; `asyncio.gather` patterns in Python SDK. |
| Conditional branching | Excellent | Standard if/else/switch in the host language. No DSL limitations. |
| Human-in-the-loop approval | Excellent | Signals pause workflows indefinitely waiting for external input. Updates provide synchronous request/response. |
| Cross-system rollback | Excellent | First-class Saga pattern support with LIFO compensation. `ParallelCompensation` option for concurrent undo. |
| Long-running workflows | Excellent | Workflows can run for months/years. State survives service restarts via event replay. |
| Event-driven triggers | Good | Signals from external systems start or resume workflows. Not a native event bus -- requires integration with something like NATS or Kafka for event ingestion. |
| Retry and error handling | Excellent | Per-activity retry policies with backoff, max attempts, non-retryable error types. Heartbeating for long activities. |
| Audit trail | Excellent | Complete event history is the persistence model itself. Every state transition is recorded. |
| Sub-workflows | Excellent | Child workflows are a first-class concept with configurable parent-close policies. |
| Compensation logic | Excellent | Saga pattern with ordered compensations, parallel compensation option. |

### LLM-Driven Dynamic Workflows

Temporal's architecture separates deterministic orchestration (Workflows) from non-deterministic execution (Activities). An LLM call is an Activity. The Workflow can dynamically decide the next step based on LLM output -- this is just an if/else on an activity result. Temporal has [explicitly positioned itself for AI agent orchestration](https://temporal.io/blog/of-course-you-can-build-dynamic-ai-agents-with-temporal), with patterns for multi-turn LLM conversations, tool calling, and adaptive workflow routing all running inside durable workflows.

### Dynamic Mid-Execution Modification

Temporal supports three message-passing mechanisms for modifying running workflows:
- **Signals**: Asynchronous writes that can inject new data or trigger new branches in a running workflow.
- **Updates**: Synchronous, transactional modifications with validation. The caller waits for acknowledgment.
- **Queries**: Read-only state inspection of running workflows.

This means LOOM could signal a running deployment workflow to add steps, change parameters, or redirect execution -- all without restarting the workflow.

### Deployment and Operations

- **Self-hosted**: Two Go binaries (server + UI server). Requires PostgreSQL/MySQL/Cassandra. Production recommendations: ~10 CPU cores and ~13 GiB memory across the four services.
- **Air-gapped**: Fully self-contained. No external network dependencies at runtime. Container images available for offline deployment.
- **Temporal Cloud**: Managed SaaS option. Not suitable for air-gapped environments.
- **Kubernetes**: Helm charts available but not required. Can run on bare VMs.

### Language Compatibility

Go SDK is mature and is actually the original SDK. Python SDK is production-ready. Both are first-class citizens.

### Scale

Temporal can handle thousands of concurrent workflows, but throughput is bounded by the persistence layer. At very high scale, every state transition is a database write, which becomes the bottleneck. Multi-cluster replication is available for HA across regions.

### Community and Trajectory

Temporal has significant momentum. Backed by a well-funded company (Temporal Technologies). Active open-source community. Strong adoption at Snap, Netflix, Stripe, HashiCorp, Datadog, and others. The project is on a clear upward trajectory with heavy investment in AI/agent use cases.

### Risks

- Event history size: Workflows with thousands of activities accumulate large histories. Requires "continue-as-new" patterns to bound history size.
- Operational complexity: Running a production Temporal cluster requires database expertise (PostgreSQL tuning, Cassandra operations).
- Vendor concentration: While open-source, Temporal Inc. controls the roadmap.

---

## 2. Argo Workflows

**What it is:** A Kubernetes-native workflow engine. Each workflow step runs as a container (Pod) on Kubernetes. Workflows are defined in YAML as DAGs or step sequences.

### Fit Against LOOM Requirements

| Requirement | Fit | Notes |
|---|---|---|
| Multi-step orchestration | Good | DAG and step templates handle complex workflows. |
| Cross-domain dependencies | Good | DAG edges express dependencies. |
| Parallel execution | Good | DAG parallelism is native. |
| Conditional branching | Moderate | `when` expressions in YAML. Less flexible than code-based branching. |
| Human-in-the-loop approval | Poor | No native approval gate. Requires external webhook + suspend/resume hack. |
| Cross-system rollback | Poor | No saga pattern. Exit handlers provide limited cleanup. |
| Long-running workflows | Moderate | Bound by Kubernetes Pod lifecycle. Workflows surviving cluster restarts require additional configuration. |
| Event-driven triggers | Moderate | Argo Events is a companion project but adds complexity. |
| Retry and error handling | Good | Per-step retry with backoff. |
| Audit trail | Good | Workflow status stored in Kubernetes CRDs. Limited long-term retention without external archival. |
| Sub-workflows | Good | Template references and workflow-of-workflows pattern. |
| Compensation logic | Poor | Must be manually implemented. |

### Critical Limitation for LOOM

**Argo requires Kubernetes.** LOOM is an infrastructure orchestrator that provisions bare metal, configures network switches, and manages VMs -- many of which exist outside Kubernetes. Requiring K8s as a prerequisite for the orchestrator that manages infrastructure creates a circular dependency. LOOM needs to work in environments where Kubernetes does not yet exist (it may be deploying Kubernetes).

### LLM-Driven Dynamic Workflows

YAML-based workflow definitions are static. Generating YAML at runtime from an LLM is possible but fragile -- the LLM must produce valid YAML that conforms to Argo's CRD schema. No support for dynamically adding steps to an already-running workflow.

### Deployment

Requires a Kubernetes cluster. All workflow state is stored as Kubernetes custom resources (CRDs) in etcd. Cannot run air-gapped without a K8s cluster.

### Verdict for LOOM

**Not recommended.** The Kubernetes dependency is a fundamental mismatch. LOOM provisions infrastructure that may not include Kubernetes. The YAML-based definition model is too rigid for LLM-driven dynamic workflows, and the lack of native saga/compensation patterns makes cross-system rollback painful.

---

## 3. Apache Airflow

**What it is:** A Python-based DAG scheduler originally built by Airbnb for batch data pipeline orchestration. Workflows (DAGs) are defined in Python and scheduled by a central scheduler.

### Fit Against LOOM Requirements

| Requirement | Fit | Notes |
|---|---|---|
| Multi-step orchestration | Good | Python DAGs with operators. |
| Cross-domain dependencies | Good | Task dependencies in DAGs. |
| Parallel execution | Moderate | Parallelism limited by executor type. CeleryExecutor scales better than LocalExecutor. Max ~250-300 parallel tasks per service. |
| Conditional branching | Moderate | BranchPythonOperator, but less natural than code-based branching. |
| Human-in-the-loop approval | Poor | No native support. External sensor pattern is a polling workaround. |
| Cross-system rollback | Poor | No saga pattern. `on_failure_callback` provides limited cleanup. |
| Long-running workflows | Poor | **Fundamental limitation.** Airflow is batch-oriented with defined start/end. Not designed for workflows running hours/days. Tasks time out. |
| Event-driven triggers | Poor | Airflow is schedule-driven, not event-driven. Airflow 3.0 adds event-driven scheduling but it is still fundamentally a scheduler. |
| Retry and error handling | Good | Per-task retries with configurable backoff. |
| Audit trail | Good | Task instance logs and XCom metadata stored in database. |
| Sub-workflows | Moderate | SubDAGs (deprecated) or TaskGroups (presentation only, not true sub-workflows). |
| Compensation logic | Poor | Must be manually implemented. |

### Critical Limitations for LOOM

1. **Batch-oriented**: Airflow assumes workflows have a defined start and end. Infrastructure deployments that take hours and wait for human approval are a poor fit.
2. **Scheduler bottleneck**: The central scheduler is a single point of failure. DAG parsing overhead grows with the number of DAGs.
3. **No durable execution**: If the Airflow scheduler dies, running workflows do not automatically resume from where they left off.

### LLM-Driven Dynamic Workflows

DAGs are parsed at schedule time, not at runtime. Dynamic DAG generation is possible but requires writing Python files that are picked up by the scheduler -- a fundamentally different model from runtime dynamic step creation.

### Verdict for LOOM

**Not recommended.** Airflow is designed for batch data pipelines, not real-time infrastructure orchestration. The lack of durable execution, poor support for long-running workflows, and no saga pattern make it unsuitable for LOOM's requirements.

---

## 4. Prefect

**What it is:** A modern Python workflow orchestration framework. Workflows are defined using Python decorators (`@flow`, `@task`). Prefect 3.0 uses a hybrid model where orchestration metadata lives in Prefect Cloud while execution happens in your infrastructure.

### Fit Against LOOM Requirements

| Requirement | Fit | Notes |
|---|---|---|
| Multi-step orchestration | Good | Python flows with tasks. |
| Cross-domain dependencies | Good | Task dependencies via data passing. |
| Parallel execution | Good | `task.submit()` for concurrent execution, DaskTaskRunner for distributed parallelism. |
| Conditional branching | Good | Standard Python control flow. |
| Human-in-the-loop approval | Poor | No native approval gate. Would require custom pause/resume logic. |
| Cross-system rollback | Poor | No saga pattern. |
| Long-running workflows | Moderate | Better than Airflow but not designed for multi-hour workflows with durable state. |
| Event-driven triggers | Good | Prefect 3.0 has native events and automations. |
| Retry and error handling | Good | Decorator-based retry with backoff. |
| Audit trail | Good | Flow run history in Prefect backend. |
| Sub-workflows | Good | Flows can call sub-flows. |
| Compensation logic | Poor | Must be manually implemented. |

### Critical Limitations for LOOM

1. **Python-only**: No Go SDK. If LOOM's core is in Go, Prefect cannot be the backbone.
2. **No durable execution**: Prefect does not replay workflow state from event history. If a worker dies, the task must be retried from scratch.
3. **Cloud dependency**: Prefect's best features (events, automations, dashboards) require Prefect Cloud or self-hosted Prefect Server. The self-hosted server is less mature than the cloud offering.
4. **Data engineering focus**: Prefect's ecosystem is optimized for data pipelines, not infrastructure orchestration.

### Verdict for LOOM

**Not recommended.** Python-only limitation is disqualifying if LOOM is in Go. No durable execution or saga pattern. Designed for data engineering, not infrastructure orchestration.

---

## 5. Netflix Conductor (Orkes)

**What it is:** A microservice orchestration engine originally built at Netflix. Workflows are defined in JSON/DSL. Now maintained by Orkes and the Conductor OSS community. The platform handles 15-20% of US internet traffic at Netflix.

### Fit Against LOOM Requirements

| Requirement | Fit | Notes |
|---|---|---|
| Multi-step orchestration | Excellent | JSON workflow definitions with rich task types. |
| Cross-domain dependencies | Good | Task dependencies in workflow definition. |
| Parallel execution | Excellent | Fork/join for parallel execution. Handles tens of thousands of parallel tasks per workflow. |
| Conditional branching | Good | Switch tasks with JavaScript expression evaluation. |
| Human-in-the-loop approval | Good | HUMAN task type is built-in. |
| Cross-system rollback | Moderate | No native saga. Must be implemented using failure workflows. |
| Long-running workflows | Good | Persistent state. Workflows can run for extended periods. |
| Event-driven triggers | Good | Event handlers can trigger workflows. |
| Retry and error handling | Good | Per-task retry policies. |
| Audit trail | Good | Execution history stored in database. |
| Sub-workflows | Good | Sub-workflow task type. |
| Compensation logic | Moderate | Failure workflows provide partial compensation. Not as elegant as Temporal's saga. |

### Strengths

- **Performance at scale**: Conductor was designed for Netflix scale. Reports of running a billion workflows per month. Generally [outperforms Temporal in high-parallelism scenarios](https://medium.com/@chucksanders22/netflix-conductor-v-s-temporal-uber-cadence-v-s-zeebe-vs-airflow-320df0365948) due to lower per-state-transition overhead.
- **Polyglot workers**: Workers can be written in Go, Java, Python, JavaScript, C#. A single workflow can have tasks executed by workers in different languages.
- **Built-in HUMAN task**: Native approval gates without workarounds.
- **LLM integration**: Recent additions include native LLM task types with 14+ provider integrations.

### Weaknesses

- **JSON workflow definitions**: Workflows are defined in JSON, not code. While "dynamic workflows using code" are supported, the primary model is declarative JSON. This is less flexible than Temporal's code-first approach for complex branching logic.
- **Java-centric server**: The Conductor server is Java/Spring Boot. Operationally, you are running a JVM application.
- **Go SDK maturity**: The Go SDK exists but is less mature than Temporal's Go SDK, which is the original and most battle-tested.
- **Community trajectory**: While actively maintained by Orkes, the community is smaller than Temporal's. Netflix itself has moved to a newer internal system (Maestro).

### Deployment

- Self-hosted with PostgreSQL + Redis (or Elasticsearch).
- Can run on VMs or Kubernetes.
- Air-gapped deployment is feasible -- the server is a Java application with database dependencies.

### Verdict for LOOM

**Viable but not optimal.** Conductor's strengths in parallel execution and polyglot workers are relevant. However, the JSON-first workflow definition model is less suited for LLM-driven dynamic workflows compared to Temporal's code-first approach. Netflix's own migration away from Conductor (to Maestro, which they [made 100x faster](https://netflixtechblog.com/100x-faster-how-we-supercharged-netflix-maestros-workflow-engine-028e9637f041)) is a yellow flag. The Java server adds operational complexity if LOOM's team is Go-focused.

---

## 6. NATS JetStream + Custom Engine

**What it is:** Use NATS JetStream as the messaging and persistence backbone, and build workflow semantics (DAG execution, state management, compensation) on top.

### Architecture Concept

NATS JetStream provides: durable message streams, exactly-once delivery, consumer groups, key-value store, and object store. A custom workflow engine would use JetStream streams as event logs, the KV store for workflow state, and consumers for task distribution to workers.

### Fit Against LOOM Requirements

| Requirement | Fit | Notes |
|---|---|---|
| Multi-step orchestration | Must build | You'd build the DAG executor. |
| Cross-domain dependencies | Must build | Dependency resolution is custom code. |
| Parallel execution | Good | NATS consumer groups naturally distribute work. |
| Conditional branching | Must build | Custom routing logic. |
| Human-in-the-loop approval | Must build | Pause on a stream, resume on signal. |
| Cross-system rollback | Must build | Saga pattern implementation from scratch. |
| Long-running workflows | Good | JetStream persistence survives restarts. |
| Event-driven triggers | Excellent | This is NATS's core strength. Native event-driven architecture. |
| Retry and error handling | Good | JetStream has built-in redelivery with backoff. |
| Audit trail | Good | Streams are append-only logs -- natural audit trail. |
| Sub-workflows | Must build | Custom implementation. |
| Compensation logic | Must build | Custom implementation. |

### Strengths

- **Lightweight**: NATS server is a single ~20MB binary. Runs anywhere -- edge devices, VMs, containers.
- **Embeddable**: Can be embedded directly into a Go application.
- **Go-native**: NATS is written in Go. The Go client is excellent.
- **Edge/distributed**: NATS leaf nodes and superclusters enable geo-distributed deployment. Perfect for edge infrastructure management.
- **Air-gapped**: Trivial. Single binary, no external dependencies.
- **Event-driven**: LOOM's event-driven triggers (alerts, capacity thresholds) map directly to NATS subjects and streams.

### Critical Weakness

**You must build the workflow engine.** DAG execution, state machines, saga patterns, compensation ordering, child workflows, workflow versioning, history replay -- all of this is custom code. This is months of engineering effort to reach production quality, and years to match the reliability of Temporal's battle-tested implementation.

### Verdict for LOOM

**Not recommended as the primary workflow engine.** The engineering cost of building workflow semantics from scratch is prohibitive. However, NATS JetStream is an excellent **complement** to a workflow engine like Temporal -- use NATS for event ingestion, inter-service messaging, and edge communication, while Temporal handles workflow orchestration.

---

## 7. Custom Engine Built on Event Sourcing

**What it is:** Build a bespoke workflow engine from scratch using event sourcing as the persistence model. Every state transition is an event appended to an immutable log. Workflow state is reconstructed by replaying events.

### When Custom Makes Sense

Custom engines are justified when:
1. Existing engines impose architectural constraints that conflict with core product requirements.
2. The workflow semantics are deeply domain-specific and cannot be expressed in general-purpose engines.
3. The team has deep expertise in distributed systems and event sourcing.
4. The deployment environment has extreme constraints (embedded, real-time, or exotic hardware).

### What LOOM Would Need to Build

- Event store (append-only log with snapshots)
- Workflow definition DSL or API
- DAG executor with topological sort
- Activity task queue with at-least-once delivery
- Saga coordinator with compensation ordering
- Timer service for delays, timeouts, and scheduled actions
- Replay engine for state reconstruction
- Worker framework for task execution
- Visibility/query layer for searching workflows
- UI for workflow monitoring
- Versioning system for workflow definition changes
- Multi-tenancy if needed

### Realistic Assessment

This is **12-18 months of engineering effort** by a team of 3-5 experienced distributed systems engineers to reach production quality. It is, in essence, rebuilding Temporal. The [consensus from the Go community](https://medium.com/@saatvikawasthi1998/mastering-workflow-orchestration-a-practical-guide-to-go-workflows-018945ab320c) is clear: "Don't build custom orchestration from scratch unless you have very specific needs. Mature workflow engines handle edge cases you haven't considered yet."

### Go-Based Options If Building

If building custom is chosen, consider starting from existing Go libraries:
- **go-workflows**: Event-sourced workflow library for Go, inspired by Temporal. Much lighter weight.
- **Microsoft durabletask-go**: Embeddable durable task framework in Go.
- **River Pro**: Postgres-backed workflow engine for Go.

### Verdict for LOOM

**Not recommended unless a fundamental gap is identified in Temporal.** The engineering investment cannot be justified when Temporal provides 90%+ of what LOOM needs out of the box.

---

## 8. Honorable Mention: Restate

**What it is:** A newer durable execution engine that takes a [different approach from Temporal](https://www.restate.dev/blog/building-a-modern-durable-execution-engine-from-first-principles). Instead of a central server replaying event histories, Restate acts as a lightweight proxy that intercepts function calls and journals them. State is managed per-invocation with lower latency than Temporal's replay model.

### Key Properties

- **Go SDK available** (plus TypeScript, Java/Kotlin, Python, Rust)
- **No external database required** -- Restate embeds its own storage
- **Lower latency** than Temporal for short workflows
- **Single binary deployment** -- simpler operations than Temporal
- **Supports workflows, event-driven handlers, and virtual objects**

### Limitations

- Younger project, smaller community
- Less proven at scale than Temporal or Conductor
- Compensation/saga patterns are less mature
- No managed cloud offering as mature as Temporal Cloud

### Relevance to LOOM

Restate is worth monitoring. Its simpler deployment model (single binary, no external database) is attractive for LOOM's air-gapped/on-prem requirements. However, for a critical infrastructure orchestrator, Temporal's larger community and proven reliability at scale are more important than Restate's operational simplicity.

---

## Comparative Matrix

| Criterion | Temporal | Argo | Airflow | Prefect | Conductor | NATS+Custom | Custom ES | Restate |
|---|---|---|---|---|---|---|---|---|
| Multi-step (100+ steps) | +++  | ++ | ++ | ++ | +++ | Build | Build | ++ |
| Cross-domain rollback | +++ | + | + | + | ++ | Build | Build | ++ |
| Parallel execution | +++ | ++ | + | ++ | +++ | ++ | Build | ++ |
| Human approval gates | +++ | - | - | - | +++ | Build | Build | + |
| Long-running workflows | +++ | + | - | + | ++ | ++ | Build | ++ |
| LLM dynamic workflows | +++ | - | + | + | ++ | Build | Build | ++ |
| Event-driven triggers | ++ | + | - | ++ | ++ | +++ | Build | ++ |
| Air-gapped deployment | +++ | + | ++ | + | ++ | +++ | +++ | +++ |
| Go compatibility | +++ | + | - | - | ++ | +++ | +++ | ++ |
| Operational simplicity | + | + | ++ | ++ | + | +++ | - | +++ |
| Community/maturity | +++ | ++ | +++ | ++ | ++ | ++ | N/A | + |
| Scale (1000+ workflows) | ++ | + | + | + | +++ | +++ | ? | ++ |

Legend: `+++` = excellent, `++` = good, `+` = adequate, `-` = poor, `Build` = you build it

---

## Recommendation

### Primary Choice: Temporal

**Temporal is the recommended workflow engine backbone for LOOM.** The rationale:

1. **Code-first workflows in Go**: LOOM's orchestration logic -- provisioning VMs, configuring switches, deploying applications -- is complex branching logic that belongs in code, not YAML or JSON. Temporal's Go SDK is the most mature SDK available.

2. **Durable execution solves LOOM's hardest problem**: Infrastructure deployments take hours, span multiple systems, and must survive service restarts. Temporal's event replay model guarantees workflows resume from exactly the right point after any failure.

3. **First-class saga/compensation**: Cross-system rollback (undo Terraform + undo Ansible + undo network config) maps directly to Temporal's saga pattern with LIFO compensation ordering.

4. **LLM-driven dynamic workflows**: Temporal's architecture cleanly separates deterministic orchestration (Workflow) from non-deterministic execution (Activities). An LLM Activity returns the next step to execute; the Workflow dynamically routes based on that result. Signals and Updates allow modifying running workflows mid-execution -- critical for LLM agents that may need to inject new steps.

5. **Human-in-the-loop**: Signals provide clean approval gates. A workflow waits indefinitely on a signal; a human (or automated system) sends the signal to proceed. No polling, no timeouts.

6. **Air-gapped/on-prem**: Two Go binaries + PostgreSQL. No cloud dependencies. Can run on bare VMs.

### Complementary Architecture: Temporal + NATS JetStream

For the complete LOOM architecture, pair Temporal with NATS JetStream:

- **Temporal**: Workflow orchestration, state management, saga compensation, human approval gates.
- **NATS JetStream**: Event ingestion (alerts, capacity thresholds, schedule changes), inter-service messaging between LOOM components, edge device communication, lightweight pub/sub for real-time notifications.

This gives LOOM durable workflow execution **and** a lightweight, embeddable event bus -- without requiring Kafka's operational overhead.

### Architecture Sketch

```
                    +-----------------+
                    |   LOOM API /    |
                    |   CLI / UI      |
                    +--------+--------+
                             |
                    +--------v--------+
                    |  NATS JetStream |  <-- Event ingestion, triggers
                    |  (Event Bus)    |
                    +--------+--------+
                             |
                    +--------v--------+
                    | Temporal Server |  <-- Workflow orchestration
                    | (Go binaries +  |
                    |  PostgreSQL)    |
                    +--------+--------+
                             |
              +--------------+--------------+
              |              |              |
     +--------v---+  +------v-----+  +-----v--------+
     | Terraform  |  | Ansible    |  | Network      |
     | Worker     |  | Worker     |  | Config Worker|
     | (Activity) |  | (Activity) |  | (Activity)   |
     +------------+  +------------+  +--------------+
```

### What Temporal Does Not Cover (and How to Fill Gaps)

1. **Event ingestion at the edge**: Use NATS JetStream. Temporal is not an event bus.
2. **Infrastructure-specific protocol handling**: Build as Temporal Activities. Redfish, IPMI, SNMP adapters are Activities that the workflow invokes.
3. **LLM workflow generation**: Build a "workflow planner" Activity that calls an LLM and returns a structured plan. The parent Workflow iterates over the plan steps, executing each as a child workflow or activity. This is a standard Temporal pattern.
4. **UI/Dashboard**: Temporal provides a basic web UI. LOOM will likely need a custom UI for infrastructure-specific visualization.

### Risk Mitigation

- **History size**: Implement "continue-as-new" patterns for workflows exceeding ~10,000 events. Design workflows with bounded history from the start.
- **Database operations**: PostgreSQL tuning is critical. Plan for connection pooling, vacuuming, and potential sharding at scale.
- **Vendor risk**: Temporal is open-source (MIT license). If Temporal Inc. changes direction, the community can fork. The codebase is Go -- well within LOOM's team capabilities.

---

## Sources

- [Temporal Architecture Principles](https://temporal.io/blog/workflow-engine-principles)
- [Temporal Internal Architecture Breakdown](https://medium.com/data-science-collective/system-design-series-a-step-by-step-breakdown-of-temporals-internal-architecture-52340cc36f30)
- [Durable Execution Engines: Temporal and Restate](https://www.kai-waehner.de/blog/2025/06/05/the-rise-of-the-durable-execution-engine-temporal-restate-in-an-event-driven-architecture-apache-kafka/)
- [Temporal for AI Agents](https://temporal.io/blog/of-course-you-can-build-dynamic-ai-agents-with-temporal)
- [Temporal Saga Compensation](https://temporal.io/blog/compensating-actions-part-of-a-complete-breakfast-with-sagas)
- [Temporal Self-Hosted Deployment Guide](https://docs.temporal.io/self-hosted-guide/deployment)
- [Temporal Workflow Message Passing (Signals, Queries, Updates)](https://docs.temporal.io/encyclopedia/workflow-message-passing)
- [Temporal vs Argo Workflows Comparison](https://pipekit.io/blog/temporal-vs-argo-workflows)
- [Temporal vs Airflow vs Argo Guide](https://www.xgrid.co/resources/temporal-vs-airflow-vs-argo-workflow-orchestration/)
- [Apache Airflow 3.0 Enhancements](https://pirgee.com/blogs/apache-airflow-in-2025-enhancements-in-workflow-orchestration)
- [Airflow Scaling at Shopify](https://shopify.engineering/lessons-learned-apache-airflow-scale)
- [Prefect Open Source](https://www.prefect.io/prefect/open-source)
- [Workflow Orchestration: Kestra vs Temporal vs Prefect 2025](https://procycons.com/en/blogs/workflow-orchestration-platforms-comparison-2025/)
- [Conductor OSS GitHub](https://github.com/conductor-oss/conductor)
- [Conductor Billion Workflows/Month](https://medium.com/orkes/running-a-billion-workflows-a-month-with-netflix-conductor-5fd358d9b6c1)
- [Netflix Maestro 100x Faster](https://netflixtechblog.com/100x-faster-how-we-supercharged-netflix-maestros-workflow-engine-028e9637f041)
- [Conductor Go SDK](https://pkg.go.dev/github.com/conductor-sdk/conductor-go)
- [NATS JetStream Documentation](https://docs.nats.io/nats-concepts/jetstream)
- [Building Event-Driven Systems with Go and NATS](https://dasroot.net/posts/2026/02/building-event-driven-systems-go-nats/)
- [Restate Durable Execution Engine](https://www.restate.dev/blog/building-a-modern-durable-execution-engine-from-first-principles)
- [Restate Go SDK](https://pkg.go.dev/github.com/restatedev/sdk-go)
- [Workflow Orchestration Showdown: Temporal vs Conductor vs Camunda](https://medium.com/@easwaranvijayakumar/workflow-orchestration-showdown-temporal-io-vs-orkes-conductor-vs-camunda-e59fd79c2b65)
- [go-workflows: Event-Sourced Workflows in Go](https://medium.com/@saatvikawasthi1998/mastering-workflow-orchestration-a-practical-guide-to-go-workflows-018945ab320c)
- [Microsoft durabletask-go](https://github.com/microsoft/durabletask-go)
- [Temporal Alternatives for Enterprise](https://akka.io/blog/temporal-alternatives)