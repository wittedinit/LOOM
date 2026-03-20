# LOOM Tech Stack Decision

## Summary

After researching every major option across 5 dimensions (language, data store, LLM integration, workflow engine, UI framework), the recommended stack is:

```
┌─────────────────────────────────────────────────────────┐
│                      LOOM Stack                          │
│                                                          │
│  Core Language:     Go                                   │
│  Workflow Engine:   Temporal                              │
│  Event Bus:         NATS JetStream                        │
│  Database:          PostgreSQL + TimescaleDB + AGE +      │
│                     pgvector                              │
│  Cache:             Valkey (Redis-compatible)              │
│  LLM:               Provider-agnostic (Claude/Ollama/vLLM/ │
│                     llama.cpp/LocalAI/any OpenAI-compat)  │
│  Frontend:          React + Next.js + shadcn/ui           │
│  Topology Viz:      Cytoscape.js                          │
│  Workflow Viz:      React Flow                            │
│  Charts:            Apache ECharts + shadcn charts        │
│  Edge Agents:       SQLite + DuckDB + NATS leaf nodes     │
└─────────────────────────────────────────────────────────┘
```

---

## Decision 1: Core Language — Go

**Weighted Score: 8.6/10** (vs Rust 5.9, Python 7.3, TypeScript 5.2)

### Why Go

- **Every protocol library exists**: gosnmp (SNMP), gofish (Redfish), go-wsman-messages (Intel AMT), openconfig/gnmi (gNMI reference impl), govmomi (vSphere), go-libvirt, goipmi, x/crypto/ssh — all production-grade
- **Official Anthropic Go SDK** with tool use, streaming, structured output
- **Goroutines** for managing 1000+ concurrent device connections at ~2KB per goroutine
- **Single static binary** — `./loom serve` with zero runtime dependencies (critical for on-prem/air-gapped)
- **Infrastructure ecosystem alignment** — K8s, Terraform, Nomad, Docker, Prometheus all written in Go
- **HashiCorp go-plugin** escape hatch if Python is ever needed for LLM reasoning

### What We Considered and Rejected

| Language | Score | Rejection Reason |
|----------|-------|-----------------|
| **Rust** | 5.9 | No IPMI, no Redfish, no AMT, no vSphere libraries. Months to build from scratch. Performance advantage irrelevant for I/O-bound orchestrator. |
| **Python** | 7.3 | Richest libraries but: GIL/concurrency at scale, deployment complexity (virtualenvs), no single-binary. Wrong for a production infrastructure orchestrator. |
| **TypeScript** | 5.2 | Missing IPMI, Redfish, AMT, libvirt, vSphere, gNMI. Web ecosystem, not systems ecosystem. |

---

## Decision 2: Data Store — PostgreSQL + Extensions

**3 systems to operate** (vs 6 in a best-of-breed approach)

### Architecture

```
PostgreSQL 16+ (single engine, multiple paradigms)
  ├── Core PostgreSQL    → Device inventory, workflow state, cost data, audit trail
  ├── TimescaleDB        → Telemetry, metrics, time-series (continuous aggregates, compression)
  ├── Apache AGE         → Network topology, dependency graphs (openCypher queries)
  ├── pgvector           → LLM embeddings, RAG over infrastructure docs
  └── JSONB              → Device capability profiles, config snapshots

NATS JetStream (event backbone)
  ├── Device state change events
  ├── Telemetry ingestion pipeline
  ├── Workflow events
  ├── Audit log streaming
  └── Edge-to-central communication (leaf nodes)

Valkey (caching layer)
  ├── Hot device state cache
  ├── Session/auth tokens
  ├── Real-time dashboard state
  └── Pub/Sub for WebSocket fan-out

Edge Agents (embedded, zero-ops)
  ├── SQLite     → Local device inventory, command queue, offline buffer
  └── DuckDB    → Local telemetry aggregation before sync
```

### Why This Approach

- **Single backup/restore pipeline** for all core data
- **SQL JOINs across all data models** — "show me switches in rack A3 with >80% bandwidth in the blast radius of router R1" is one query
- **NATS runs anywhere** — single 20MB binary, runs on Raspberry Pi to cloud, leaf nodes for edge sites
- **All open source** — PostgreSQL License, Apache 2.0, BSD, MIT, Public Domain. No BSL, no SSPL.

### Scale-Out Path (Add When Needed, Not Day 1)

| Trigger | Add |
|---------|-----|
| Telemetry >500K datapoints/sec | VictoriaMetrics |
| Graph queries >4 hops / need GDS algorithms | Neo4j read replica |
| Vector search >5M embeddings | Qdrant |

---

## Decision 3: LLM Integration — Provider-Agnostic with Local-First Support

**LOOM must NEVER be dependent on any single cloud LLM API.** The LLM engine is designed as a provider-agnostic abstraction that connects to ANY compatible backend — cloud APIs, local models, or hybrid.

### Architecture

```
LOOM LLM Engine (Provider-Agnostic)
  │
  ├── LLM Router / Gateway
  │   ├── Selects provider based on: availability, cost, latency, task complexity
  │   ├── Automatic failover: cloud unavailable → local model
  │   └── Policy enforcement: "production configs MUST use local models only"
  │
  ├── Cloud Providers (optional, not required)
  │   ├── Anthropic Claude (Opus/Sonnet/Haiku)
  │   ├── OpenAI GPT-4o / o-series
  │   ├── Google Gemini
  │   └── Any OpenAI-compatible API
  │
  ├── Local Model Frameworks (first-class, not fallback)
  │   ├── Ollama         → Simplest local deployment, 100+ models, REST API
  │   ├── vLLM           → Production serving (19x faster than Ollama at scale)
  │   ├── llama.cpp      → CPU-only inference, no GPU required
  │   ├── LocalAI        → OpenAI-compatible API for any local model
  │   ├── LM Studio      → Desktop GUI + API server
  │   ├── text-generation-inference (TGI) → HuggingFace production serving
  │   └── ExLlamaV2      → Optimized quantized inference
  │
  ├── Recommended Local Models
  │   ├── Llama 4 Scout (8B/70B)  → Best open-weights general reasoning
  │   ├── DeepSeek-V3             → GPT-4-level, fully open weights
  │   ├── Qwen 2.5 (7B/14B/72B)  → Strong for structured output / config generation
  │   ├── Mistral Large / Small   → Good balance of speed and quality
  │   ├── CodeLlama / DeepSeek-Coder → For config generation tasks
  │   └── Fine-tuned domain models → Network config specialists (7B-14B)
  │
  └── MCP Servers → Tool integration (infrastructure adapters as MCP tools)
```

### Provider-Agnostic Design Principles

1. **No cloud dependency** — LOOM must function fully with only local models. Cloud APIs are an enhancement, not a requirement.
2. **Unified interface** — All providers (Claude, GPT, Ollama, vLLM, llama.cpp) are accessed through a single Go interface. Adapters translate to each provider's API.
3. **OpenAI-compatible API as common protocol** — Most local frameworks (Ollama, vLLM, LocalAI, LM Studio) expose an OpenAI-compatible API. LOOM's gateway speaks this protocol natively, making any compatible backend plug-and-play.
4. **Per-task routing** — Different tasks can use different providers: local SLM for config validation, cloud API for complex planning (if available), fine-tuned local model for vendor-specific configs.
5. **Deployment modes**:
   - **Air-gapped**: Local models only (Ollama/vLLM + Llama/DeepSeek/Qwen). Zero external network calls.
   - **Hybrid**: Local models for routine + cloud APIs for complex reasoning. Automatic failover.
   - **Cloud-assisted**: Cloud APIs primary with local fallback.
6. **Model hot-swap** — Switch models at runtime without restart. A/B test local vs cloud models on real traffic.

### Hardware Requirements for Local Models

| Model Size | VRAM Required | Example Hardware | Throughput |
|-----------|---------------|-----------------|------------|
| 7B (4-bit) | 4-6 GB | Any modern GPU, Apple M-series | 30-50 tokens/s |
| 14B (4-bit) | 8-10 GB | RTX 3090/4090, M2 Pro | 20-35 tokens/s |
| 70B (4-bit) | 35-40 GB | A100 40GB, 2x RTX 4090 | 10-20 tokens/s |
| 70B (8-bit) | 70-80 GB | A100 80GB, H100 | 15-25 tokens/s |
| CPU-only (7B) | 8 GB RAM | Any server, no GPU | 5-10 tokens/s |

**Minimum viable local deployment**: A single machine with 16GB RAM can run a 7B model via llama.cpp on CPU. No GPU required. This is sufficient for config validation, protocol selection, and basic reasoning.

### Key Patterns

| Pattern | Use Case |
|---------|----------|
| **Multi-agent** | Domain-specific agents (network, compute, cost) coordinated by a planner agent |
| **RAG** | Grounded on vendor docs (Cisco, Juniper, Arista config guides) via pgvector |
| **Tiered autonomy** | Routine ops = auto-execute; production changes = human-in-the-loop |
| **Structured output validation** | JSON schema + config syntax check + Batfish-style verification before any change |
| **Decision caching** | Repeated patterns cached to avoid redundant LLM calls |

### Hallucination Prevention (Critical)

```
LLM generates config
  → JSON schema validation
    → Config syntax check (vendor-specific parser)
      → Batfish-style offline verification
        → Dry-run / sandbox execution
          → Human approval (if production)
            → Live execution
              → Post-deployment verification
```

### Cost Optimization

- **Local models for 80% of decisions** — $0/decision (hardware amortization only)
- Cloud API for 20% of complex decisions (if connected) — ~$0.10/decision
- Semantic caching reduces calls 20-50%
- **Air-gapped cost: $0/day** (hardware only, no per-token fees)
- **Hybrid cost: ~$5-20/day** at 1000 decisions/day (80% local, 20% cloud)
- **Cloud-only cost: ~$15-40/day** at 1000 decisions/day

---

## Decision 4: Workflow Engine — Temporal

### Why Temporal

| LOOM Requirement | Temporal Capability |
|-----------------|-------------------|
| 100+ step workflows | Unlimited activity sequences + child workflows |
| Cross-domain rollback | First-class Saga pattern with LIFO compensation |
| Parallel execution | `workflow.Go()` goroutines in Go SDK |
| Human approval gates | Signals pause workflows indefinitely |
| Survive service restarts | Event history replay — resume from exact point |
| LLM-driven dynamic steps | Activity returns next step; workflow routes dynamically |
| Modify running workflows | Signals + Updates inject data/steps mid-execution |
| Air-gapped deployment | Two Go binaries + PostgreSQL, no cloud dependencies |

### What We Considered and Rejected

| Engine | Rejection Reason |
|--------|-----------------|
| **Argo Workflows** | Requires Kubernetes — LOOM provisions infra that may not include K8s |
| **Airflow** | Batch-oriented, no durable execution, no saga pattern |
| **Prefect** | Python-only, no Go SDK, no durable execution |
| **Conductor** | JSON-first (not code-first), Java server, Netflix migrated away |
| **Custom engine** | 12-18 months to rebuild what Temporal provides |

### Complementary: NATS JetStream

Temporal handles workflow orchestration. NATS handles everything else:
- Event ingestion (alerts, capacity thresholds)
- Inter-service messaging between LOOM components
- Edge device communication (leaf nodes)
- Lightweight pub/sub for real-time notifications

---

## Decision 5: UI — React + Next.js + shadcn/ui

### Stack

| Layer | Technology | Why |
|-------|-----------|-----|
| Framework | React + Next.js | Largest ecosystem, every viz library has React support |
| Components | shadcn/ui (Tailwind + Radix) | Source ownership, dark mode native, built-in charts |
| Data Grid | TanStack Table | Virtualized rows for 1000+ device inventory |
| Charts (simple) | shadcn charts (Recharts) | KPI cards, sparklines — included with shadcn |
| Charts (advanced) | Apache ECharts | Cost analytics, large time-series, WebGL rendering |
| Topology Map | Cytoscape.js | 1000+ device network graphs, WebGL, graph algorithms |
| Workflow Viz | React Flow | Orchestration step visualization, custom node types |
| Server State | TanStack Query v5 | Caching, background refetch, optimistic updates |
| Client State | Zustand | Tenant context, dark mode, filter state (1KB) |
| Real-time | SSE (primary) + WebSocket | SSE for telemetry, WebSocket for bidirectional ops |
| Metrics | Grafana embed (optional) | Reuse existing Grafana dashboards where applicable |

### What We Rejected

| Option | Rejection Reason |
|--------|-----------------|
| **Grafana as primary UI** | Metrics viz tool, not an application platform. Can't do CRUD, workflows, request building. |
| **Backstage** | Developer portal, not infrastructure operations center. Fixed data model doesn't fit. |
| **Vue/Nuxt** | Thinner viz ecosystem — React Flow is React-only, ECharts React wrapper is better. |
| **SvelteKit** | Best performance but 3-5x smaller library ecosystem. |

---

## Complete Architecture

```
                         ┌──────────────────────┐
                         │   React + Next.js     │
                         │   shadcn/ui + ECharts  │
                         │   Cytoscape + ReactFlow│
                         └──────────┬───────────┘
                                    │ SSE / WebSocket / REST
                         ┌──────────▼───────────┐
                         │      LOOM Core        │
                         │      (Go binary)      │
                         │                       │
                         │  ┌─────────────────┐  │
                         │  │  LLM Engine     │  │
                         │  │ (provider-agno- │  │
                         │  │ stic: Claude /  │  │
                         │  │ Ollama / vLLM / │  │
                         │  │ llama.cpp /any) │  │
                         │  └────────┬────────┘  │
                         │           │           │
                         │  ┌────────▼────────┐  │
                         │  │  Temporal        │  │
                         │  │  (Workflows)     │  │
                         │  └────────┬────────┘  │
                         │           │           │
                         │  ┌────────▼────────┐  │
                         │  │  Adapter Layer   │  │
                         │  │  (25-30 protocols│  │
                         │  └────────┬────────┘  │
                         └──────────┼───────────┘
                                    │
                    ┌───────────────┼───────────────┐
                    │               │               │
             ┌──────▼──────┐ ┌─────▼──────┐ ┌─────▼──────┐
             │  PostgreSQL  │ │   NATS     │ │  Valkey    │
             │  + Timescale │ │ JetStream  │ │  (Cache)   │
             │  + AGE       │ │ (Events)   │ │            │
             │  + pgvector  │ │            │ │            │
             └─────────────┘ └─────┬──────┘ └────────────┘
                                   │
                            ┌──────▼──────┐
                            │ Edge Agents  │
                            │ (SQLite +    │
                            │  DuckDB +    │
                            │  NATS leaf)  │
                            └─────────────┘
```

---

## Deployment Model

### Single Binary + Dependencies

```bash
# LOOM deploys as:
./loom serve                    # Go binary (API + adapters + LLM engine)

# Required infrastructure:
postgresql                      # + TimescaleDB + AGE + pgvector extensions
nats-server --jetstream         # Event backbone
valkey-server                   # Caching (optional, can start without)
temporal-server                 # Workflow engine (uses same PostgreSQL)

# Frontend:
# Embedded via go:embed OR served by Next.js standalone
```

### Air-Gapped / On-Prem

All components run as native binaries or containers. **Zero cloud dependencies.**

LLM in air-gapped mode:
```bash
# Option 1: Ollama (simplest)
ollama serve                          # Starts local model server
ollama pull llama4:8b                 # Download model once, runs forever offline

# Option 2: vLLM (production, GPU)
vllm serve meta-llama/Llama-4-8B     # High-throughput serving

# Option 3: llama.cpp (CPU-only, no GPU needed)
./llama-server -m model.gguf         # Runs on any hardware

# LOOM connects to any of these via OpenAI-compatible API
LOOM_LLM_ENDPOINT=http://localhost:11434/v1  # Ollama
LOOM_LLM_ENDPOINT=http://localhost:8000/v1   # vLLM
LOOM_LLM_ENDPOINT=http://localhost:8080/v1   # llama.cpp
```

**No internet required. No API keys required. No cloud accounts required.**

---

## Research Documents

| Document | Scope |
|----------|-------|
| [Core Language Analysis](../research/tech-stack/01-core-language.md) | Go vs Rust vs Python vs TypeScript — protocol libraries, LLM SDKs, concurrency, deployment |
| [Data Store Analysis](../research/tech-stack/02-data-store.md) | PostgreSQL, Neo4j, TimescaleDB, NATS, Valkey, pgvector — unified vs best-of-breed |
| [LLM Integration Analysis](../research/tech-stack/03-llm-integration.md) | Claude architecture, multi-agent, RAG, hallucination prevention, cost optimization |
| [Workflow Engine Analysis](../research/tech-stack/04-workflow-engine.md) | Temporal vs Argo vs Airflow vs Prefect vs Conductor vs custom |
| [UI/Dashboard Analysis](../research/tech-stack/05-ui-dashboard.md) | React/Next.js, shadcn/ui, ECharts, Cytoscape.js, React Flow, real-time patterns |
