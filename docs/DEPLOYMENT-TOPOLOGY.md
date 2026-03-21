# Deployment Topology

> How to deploy LOOM from a single laptop to a globally distributed fleet.
> This document covers four deployment modes, component placement, edge agent architecture,
> scaling strategies, failure modes, and resource requirements with concrete numbers.

---

## 1. Deployment Modes

LOOM supports four deployment modes. Each is a strict superset of the one before it — the same
Go binary is used in all modes, configured differently.

### Mode 1: Single Binary (Development / Small Sites)

Everything runs in one process. A single `./loom serve` command starts the API server,
Temporal worker, embedded NATS server, and adapter runtime. Only PostgreSQL runs externally
(there is no embeddable PostgreSQL for Go).

```
Suitable for: <100 devices, single site, development, demos, home labs
```

```
┌─────────────────────────────────────────────────────────┐
│                   loom serve (single process)            │
│                                                          │
│  ┌──────────────┐  ┌───────────────┐  ┌──────────────┐  │
│  │   HTTP API   │  │ Temporal      │  │  Embedded    │  │
│  │   (port 8080)│  │ Worker        │  │  NATS Server │  │
│  │              │  │ (in-process)  │  │  (port 4222) │  │
│  └──────┬───────┘  └───────┬───────┘  └──────┬───────┘  │
│         │                  │                  │          │
│  ┌──────▼──────────────────▼──────────────────▼───────┐  │
│  │              Adapter Runtime                       │  │
│  │  Redfish │ SNMP │ SSH │ IPMI │ gNMI │ NETCONF     │  │
│  └──────────────────────┬─────────────────────────────┘  │
│                         │                                │
│  ┌──────────────────────▼─────────────────────────────┐  │
│  │         LLM Engine (provider-agnostic)              │  │
│  │  Ollama local │ Claude API │ vLLM │ llama.cpp      │  │
│  └────────────────────────────────────────────────────┘  │
└─────────────────────────┬────────────────────────────────┘
                          │
              ┌───────────▼────────────┐
              │      PostgreSQL 16+    │
              │  + TimescaleDB + AGE   │
              │  + pgvector            │
              │  (external, required)  │
              └────────────────────────┘
```

**How to run:**

```bash
# Start PostgreSQL (if not already running)
pg_ctl start -D /var/lib/postgresql/data

# Start LOOM — everything else is embedded
./loom serve \
  --db "postgres://loom:secret@localhost:5432/loom?sslmode=disable" \
  --nats-embedded \
  --temporal-embedded \
  --llm-endpoint "http://localhost:11434/v1"   # Ollama, or omit for cloud
```

**What "embedded" means:**
- **NATS**: The Go NATS server library (`github.com/nats-io/nats-server/v2/server`) supports
  in-process embedding. LOOM starts a NATS server inside the same Go process. JetStream is
  enabled with file-based storage under `./data/nats/`.
- **Temporal**: LOOM embeds the Temporal server using `go.temporal.io/server`. The Temporal
  server shares the same PostgreSQL database (separate schema: `temporal`). The Temporal worker
  runs as goroutines in the same process.
- **Valkey**: Not used in Mode 1. Hot state is kept in an in-process Go cache
  (`sync.Map` or `github.com/dgraph-io/ristretto`).

---

### Mode 2: Distributed (Production)

Separate processes for API, workers, and shared infrastructure. Each process can be scaled
independently. This is the standard production deployment for a single site or data center.

```
Suitable for: 100-10,000 devices, single site or campus, production
```

```
                    ┌──────────────────┐
                    │   Load Balancer  │
                    │  (HAProxy/Nginx) │
                    └────────┬─────────┘
                             │
              ┌──────────────┼──────────────┐
              │              │              │
     ┌────────▼───────┐ ┌───▼────────┐ ┌───▼────────┐
     │   loom-api (1) │ │ loom-api(2)│ │ loom-api(3)│
     │   port 8080    │ │ port 8080  │ │ port 8080  │
     └────────┬───────┘ └───┬────────┘ └───┬────────┘
              │             │              │
              └──────────┬──┘──────────────┘
                         │
         ┌───────────────┼───────────────────────┐
         │               │                       │
┌────────▼───────┐ ┌─────▼──────────┐ ┌──────────▼─────────┐
│ Temporal Server│ │  NATS Cluster  │ │   Valkey Cluster   │
│ (2-3 replicas) │ │  (3 nodes)     │ │   (3 nodes, HA)    │
│ + Temporal UI  │ │  JetStream R3  │ │                    │
└────────┬───────┘ └─────┬──────────┘ └────────────────────┘
         │               │
         │    ┌──────────┬┘
         │    │          │
┌────────▼────▼──┐  ┌───▼──────────────────────────────────┐
│   PostgreSQL   │  │        loom-worker (N instances)      │
│   Cluster      │  │                                       │
│  (Patroni HA)  │  │  ┌───────────┐ ┌───────────────────┐  │
│                │  │  │ Temporal   │ │ Adapter Runtime   │  │
│  Primary +     │  │  │ Worker    │ │ (all protocols)   │  │
│  1-2 replicas  │  │  │ Goroutines│ │                   │  │
│                │  │  └───────────┘ └───────────────────┘  │
│  Schemas:      │  │                                       │
│  - loom        │  │  ┌───────────────────────────────────┐│
│  - temporal    │  │  │ LLM Engine (provider-agnostic)    ││
│  - timescale   │  │  └───────────────────────────────────┘│
└────────────────┘  └───────────────────────────────────────┘
```

**Process roles:**

| Process | Role | Stateless? | Scale |
|---------|------|------------|-------|
| `loom-api` | HTTP API, WebSocket, SSE, authentication, request validation | Yes | Horizontal behind LB |
| `loom-worker` | Temporal activity/workflow workers, adapter execution, LLM calls | Yes | Horizontal (add instances) |
| Temporal Server | Workflow orchestration, history, task queues | No (uses PG) | 2-3 replicas |
| NATS | Event bus, edge communication, JetStream persistence | No (file store) | 3-node cluster |
| PostgreSQL | All persistent state, Temporal storage, telemetry | No | Patroni HA (primary + replicas) |
| Valkey | Hot cache, session tokens, pub/sub for WebSocket fan-out | No (optional persistence) | 3-node cluster |

**How to run:**

```bash
# On each API node
./loom api \
  --db "postgres://loom:secret@pg-primary:5432/loom" \
  --nats "nats://nats1:4222,nats://nats2:4222,nats://nats3:4222" \
  --temporal "temporal-server:7233" \
  --valkey "valkey1:6379,valkey2:6379,valkey3:6379" \
  --listen ":8080"

# On each worker node
./loom worker \
  --db "postgres://loom:secret@pg-primary:5432/loom" \
  --nats "nats://nats1:4222,nats://nats2:4222,nats://nats3:4222" \
  --temporal "temporal-server:7233" \
  --valkey "valkey1:6379" \
  --llm-endpoint "http://vllm-server:8000/v1" \
  --worker-concurrency 50
```

---

### Mode 3: Hub + Edge (Multi-Site / Edge)

A central hub runs the full Mode 2 deployment. Lightweight edge agents run at remote sites.
Edge agents connect to the hub via NATS leaf nodes and operate autonomously during network
partitions.

```
Suitable for: geographically distributed infrastructure, edge/colo sites,
              branch offices, 10-10,000+ devices across many sites
```

```
                         ┌─────────────────────────────────────────┐
                         │            CENTRAL HUB (Mode 2)         │
                         │                                          │
                         │  loom-api ── Temporal ── PostgreSQL      │
                         │      │                      │            │
                         │  loom-worker ── Valkey ── NATS Hub       │
                         │                          (port 7422)     │
                         └─────────────┬──────────┬────────────────┘
                                       │          │
                            ┌──────────┘          └──────────┐
                            │ NATS Leaf Connection            │ NATS Leaf Connection
                            │ (TLS, auto-reconnect)           │ (TLS, auto-reconnect)
                            │                                 │
               ┌────────────▼──────────────┐    ┌─────────────▼─────────────┐
               │  EDGE SITE A (Denver)      │    │  EDGE SITE B (London)     │
               │                            │    │                           │
               │  ┌──────────────────────┐  │    │  ┌──────────────────────┐ │
               │  │    loom-agent        │  │    │  │    loom-agent        │ │
               │  │                      │  │    │  │                      │ │
               │  │  ┌────────────────┐  │  │    │  │  ┌────────────────┐  │ │
               │  │  │ NATS Leaf Node │  │  │    │  │  │ NATS Leaf Node │  │ │
               │  │  │ (embedded)     │  │  │    │  │  │ (embedded)     │  │ │
               │  │  └────────────────┘  │  │    │  │  └────────────────┘  │ │
               │  │  ┌────────────────┐  │  │    │  │  ┌────────────────┐  │ │
               │  │  │ SQLite (local  │  │  │    │  │  │ SQLite (local  │  │ │
               │  │  │ state + queue) │  │  │    │  │  │ state + queue) │  │ │
               │  │  └────────────────┘  │  │    │  │  └────────────────┘  │ │
               │  │  ┌────────────────┐  │  │    │  │  ┌────────────────┐  │ │
               │  │  │ Adapter Runtime│  │  │    │  │  │ Adapter Runtime│  │ │
               │  │  │ (site-local    │  │  │    │  │  │ (site-local    │  │ │
               │  │  │  protocols)    │  │  │    │  │  │  protocols)    │  │ │
               │  │  └────────────────┘  │  │    │  │  └────────────────┘  │ │
               │  └──────────────────────┘  │    │  └──────────────────────┘ │
               │                            │    │                           │
               │   10 servers, 2 switches   │    │  50 servers, 8 switches   │
               └────────────────────────────┘    └───────────────────────────┘
```

**NATS leaf node topology:**

The edge agent embeds a NATS server configured as a leaf node. The leaf node connects to the
hub's NATS cluster over a single TLS connection. From the hub's perspective, the leaf node
appears as a single client. From the edge's perspective, local publishers and subscribers
operate at LAN speed even when the WAN link is down.

```
Hub NATS Cluster (3 nodes, port 7422 for leaf connections)
    │
    ├── Leaf: edge-denver (1 TLS conn, auto-reconnect)
    │     └── Local subjects: loom.site-denver.>
    │         Imported: loom.hub.commands.site-denver.>
    │         Exported: loom.site-denver.events.>
    │
    ├── Leaf: edge-london (1 TLS conn, auto-reconnect)
    │     └── Local subjects: loom.site-london.>
    │         Imported: loom.hub.commands.site-london.>
    │         Exported: loom.site-london.events.>
    │
    └── Leaf: edge-tokyo (1 TLS conn, auto-reconnect)
          └── Local subjects: loom.site-tokyo.>
```

---

### Mode 4: Air-Gapped

Same architecture as Mode 2 or Mode 3, with all external dependencies replaced by
on-premises alternatives. No outbound network connections of any kind.

```
Suitable for: classified environments, SCIF, high-security government,
              critical infrastructure with no internet connectivity
```

**What changes vs. connected deployment:**

| Component | Connected | Air-Gapped Replacement |
|-----------|-----------|----------------------|
| LLM | Claude API / OpenAI | vLLM + Llama 4 70B or DeepSeek-V3 (local GPU server) |
| KMS | AWS KMS / GCP KMS / Azure Key Vault | File-based KMS (Go `crypto/aes` + encrypted keyring on local disk or HSM) |
| TLS Certificates | Let's Encrypt / public CA | Internal CA (step-ca, cfssl, or OpenSSL) |
| Binary Updates | GitHub Releases / container registry | Signed binary + SHA256 checksum transferred via USB or secure sneakernet |
| Container Images | Docker Hub / GHCR | Local registry (Harbor, Zot, or `registry:2`) |
| NTP | Public NTP pools | Internal NTP server (GPS-disciplined or Stratum 1) |
| DNS | Public DNS resolvers | Internal DNS (CoreDNS, Unbound, or BIND) |
| Telemetry export | Cloud monitoring | None — all telemetry stays local |

**Air-gapped LLM deployment:**

```bash
# Option 1: vLLM on GPU server (production, highest throughput)
vllm serve meta-llama/Llama-4-70B-Instruct \
  --tensor-parallel-size 2 \
  --max-model-len 8192 \
  --port 8000

# Option 2: Ollama (simpler, single-GPU or CPU)
ollama serve
ollama pull deepseek-v3:70b

# Option 3: llama.cpp (CPU-only, no GPU at all)
./llama-server -m llama-4-8b-instruct-q4.gguf --port 8080

# LOOM connects identically — just point to local endpoint
./loom serve --llm-endpoint "http://gpu-server:8000/v1"
```

**Binary update procedure (air-gapped):**

```
1. On internet-connected build machine:
   $ make release
   $ sha256sum loom-v1.2.3-linux-amd64 > loom-v1.2.3-linux-amd64.sha256
   $ gpg --sign --armor loom-v1.2.3-linux-amd64.sha256

2. Transfer to air-gapped environment via approved media (USB, DVD, etc.)

3. On air-gapped machine:
   $ gpg --verify loom-v1.2.3-linux-amd64.sha256.asc
   $ sha256sum -c loom-v1.2.3-linux-amd64.sha256
   $ systemctl stop loom
   $ cp loom-v1.2.3-linux-amd64 /usr/local/bin/loom
   $ systemctl start loom
```

---

## 2. Component Placement Table

| Component | Mode 1 (Single Binary) | Mode 2 (Distributed) | Mode 3 Hub | Mode 3 Edge | Mode 4 (Air-Gapped) |
|-----------|----------------------|--------------------|-----------|-----------|--------------------|
| **loom-api** | In-process | 2-3 instances behind LB | Same as Mode 2 | Not present | Same as Mode 2 |
| **loom-worker** | In-process (goroutines) | N instances (horizontal) | Same as Mode 2 | Not present | Same as Mode 2 |
| **loom-agent** | Not used | Not used | Not used | 1 per edge site | 1 per edge site |
| **Temporal Server** | Embedded in-process | 2-3 replicas | Same as Mode 2 | Not present (hub only) | Same as Mode 2 |
| **Temporal Worker** | In-process | Inside loom-worker | Same as Mode 2 | Not present | Same as Mode 2 |
| **NATS** | Embedded in-process | 3-node cluster | 3-node cluster (hub port for leaves) | Embedded leaf node | 3-node cluster |
| **PostgreSQL** | External, single instance | Patroni HA (primary + replicas) | Same as Mode 2 | Not present | Same as Mode 2 |
| **TimescaleDB** | PG extension | PG extension | PG extension | Not present | PG extension |
| **Apache AGE** | PG extension | PG extension | PG extension | Not present | PG extension |
| **pgvector** | PG extension | PG extension | PG extension | Not present | PG extension |
| **Valkey** | In-process cache (ristretto) | 3-node cluster | Same as Mode 2 | Not present | Same as Mode 2 |
| **SQLite** | Not used | Not used | Not used | Local state + command queue | Not used (hub), per-agent (edge) |
| **DuckDB** | Not used | Not used | Not used | Local telemetry aggregation | Not used (hub), per-agent (edge) |
| **LLM Engine** | Local (Ollama) or cloud API | vLLM / Ollama / cloud API | Same as Mode 2 | Not present (hub decision) | vLLM or llama.cpp (local only) |
| **Adapter Runtime** | In-process | Inside loom-worker | Inside loom-worker | Inside loom-agent (site protocols) | Same as Mode 2 |
| **Frontend (Next.js)** | Embedded via go:embed | Standalone or embedded | Same as Mode 2 | Not present | Same as Mode 2 |
| **Load Balancer** | Not needed | HAProxy / Nginx / cloud LB | Same as Mode 2 | Not needed | HAProxy / Nginx |
| **KMS** | Local keyring file | Cloud KMS or Vault | Same as Mode 2 | Inherits hub keys | File-based KMS or HSM |
| **TLS** | Self-signed or Let's Encrypt | Public CA or internal CA | Same as Mode 2 | Hub-issued certs | Internal CA (step-ca) |

---

## 3. Edge Agent Architecture

### What the Edge Agent Binary Contains

The `loom-agent` binary is a single statically-linked Go executable (~30-50 MB) that contains:

```
loom-agent binary
├── Embedded NATS leaf node server
├── SQLite database engine (via modernc.org/sqlite — pure Go, no CGO)
├── DuckDB engine (for telemetry aggregation)
├── Adapter runtime (only adapters relevant to the site)
├── Discovery scanner (subnet/range scanning)
├── Command queue processor
├── Health check runner
├── Metrics collector (Prometheus-compatible)
└── Auto-update agent (checks hub for new versions)
```

### Edge Agent Configuration

```go
package agent

// AgentConfig defines the configuration for a loom-agent edge instance.
type AgentConfig struct {
    // SiteID uniquely identifies this edge site within the LOOM deployment.
    SiteID   string `yaml:"site_id"   json:"site_id"`

    // SiteName is a human-readable label for this site.
    SiteName string `yaml:"site_name" json:"site_name"`

    // TenantID scopes this agent's operations.
    TenantID string `yaml:"tenant_id" json:"tenant_id"`

    // HubNATS is the address of the hub's NATS cluster leaf port.
    HubNATS HubNATSConfig `yaml:"hub_nats" json:"hub_nats"`

    // LocalDB is the path to the SQLite database file.
    LocalDB string `yaml:"local_db" json:"local_db" default:"./data/agent.db"`

    // TelemetryDB is the path to the DuckDB file for local telemetry.
    TelemetryDB string `yaml:"telemetry_db" json:"telemetry_db" default:"./data/telemetry.duckdb"`

    // Adapters lists which protocol adapters to load at this site.
    Adapters []string `yaml:"adapters" json:"adapters"`

    // Discovery configures automatic device discovery.
    Discovery DiscoveryConfig `yaml:"discovery" json:"discovery"`

    // SyncInterval is how often to sync state with the hub when connected.
    SyncInterval Duration `yaml:"sync_interval" json:"sync_interval" default:"30s"`

    // HealthCheckInterval is how often to run device health checks.
    HealthCheckInterval Duration `yaml:"health_check_interval" json:"health_check_interval" default:"60s"`

    // OfflineQueueMax is the maximum number of queued commands when disconnected.
    OfflineQueueMax int `yaml:"offline_queue_max" json:"offline_queue_max" default:"10000"`

    // Metrics configures the local Prometheus metrics endpoint.
    Metrics MetricsConfig `yaml:"metrics" json:"metrics"`
}

// HubNATSConfig configures the NATS leaf node connection to the hub.
type HubNATSConfig struct {
    // URLs is the list of hub NATS addresses (leaf port).
    URLs []string `yaml:"urls" json:"urls"`

    // TLSCert is the path to the client TLS certificate.
    TLSCert string `yaml:"tls_cert" json:"tls_cert"`

    // TLSKey is the path to the client TLS key.
    TLSKey string `yaml:"tls_key" json:"tls_key"`

    // TLSCA is the path to the CA certificate for verifying the hub.
    TLSCA string `yaml:"tls_ca" json:"tls_ca"`

    // ReconnectWait is the delay between reconnection attempts.
    ReconnectWait Duration `yaml:"reconnect_wait" json:"reconnect_wait" default:"5s"`

    // MaxReconnects is the maximum number of reconnection attempts (-1 = unlimited).
    MaxReconnects int `yaml:"max_reconnects" json:"max_reconnects" default:"-1"`
}

// DiscoveryConfig configures automatic device discovery at the edge site.
type DiscoveryConfig struct {
    // Enabled controls whether automatic discovery is active.
    Enabled bool `yaml:"enabled" json:"enabled" default:"true"`

    // Subnets lists the IP ranges to scan.
    Subnets []string `yaml:"subnets" json:"subnets"`

    // Interval is how often to run a full discovery sweep.
    Interval Duration `yaml:"interval" json:"interval" default:"1h"`

    // Protocols lists which protocols to probe during discovery.
    Protocols []string `yaml:"protocols" json:"protocols"`
}

// MetricsConfig configures the local metrics endpoint.
type MetricsConfig struct {
    Enabled bool   `yaml:"enabled" json:"enabled" default:"true"`
    Listen  string `yaml:"listen"  json:"listen"  default:":9090"`
}
```

**Example agent config file (`/etc/loom/agent.yaml`):**

```yaml
site_id: "site-denver-01"
site_name: "Denver Colo Rack A"
tenant_id: "tenant-acme-corp"

hub_nats:
  urls:
    - "nats://hub-nats1.acme.internal:7422"
    - "nats://hub-nats2.acme.internal:7422"
    - "nats://hub-nats3.acme.internal:7422"
  tls_cert: "/etc/loom/certs/agent.crt"
  tls_key: "/etc/loom/certs/agent.key"
  tls_ca: "/etc/loom/certs/ca.crt"
  reconnect_wait: "5s"
  max_reconnects: -1

local_db: "/var/lib/loom/agent.db"
telemetry_db: "/var/lib/loom/telemetry.duckdb"

adapters:
  - "redfish"
  - "ipmi"
  - "snmp"
  - "ssh"

discovery:
  enabled: true
  subnets:
    - "10.20.1.0/24"
    - "10.20.2.0/24"
  interval: "1h"
  protocols:
    - "snmp"
    - "redfish"
    - "ssh"

sync_interval: "30s"
health_check_interval: "60s"
offline_queue_max: 10000

metrics:
  enabled: true
  listen: ":9090"
```

### Hub Communication via NATS Leaf Node

The edge agent's embedded NATS server connects to the hub as a leaf node. Subject mappings
control what data flows in each direction:

```
┌─────────────────────────────────────────────────────────────┐
│                      NATS Subject Mapping                    │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  EDGE → HUB (exported from leaf to hub):                     │
│                                                              │
│    loom.{site_id}.device.discovered    → Device found         │
│    loom.{site_id}.device.updated       → State change         │
│    loom.{site_id}.health.*             → Health check results │
│    loom.{site_id}.telemetry.*          → Aggregated metrics   │
│    loom.{site_id}.command.result       → Command completion   │
│    loom.{site_id}.agent.heartbeat      → Agent liveness       │
│                                                              │
│  HUB → EDGE (imported from hub to leaf):                     │
│                                                              │
│    loom.hub.commands.{site_id}.*       → Commands to execute  │
│    loom.hub.config.{site_id}           → Config updates       │
│    loom.hub.firmware.{site_id}         → Firmware push        │
│    loom.hub.agent.{site_id}.update     → Agent binary update  │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

### Autonomous vs. Hub-Required Operations

| Operation | Autonomous (edge-local) | Requires Hub |
|-----------|------------------------|--------------|
| Device discovery (subnet scan) | Yes | No |
| Health checks (ping, SNMP, Redfish) | Yes | No |
| Telemetry collection (SNMP polling, gNMI streaming) | Yes | No |
| Power state monitoring | Yes | No |
| Queued commands (power cycle, reboot) | Yes (from local queue) | No |
| Read-only state queries | Yes (from SQLite) | No |
| Multi-step workflow execution | No | Yes (Temporal on hub) |
| LLM-powered decisions | No | Yes (LLM engine on hub) |
| Approval gates | No | Yes (approval service on hub) |
| Cross-site orchestration | No | Yes (hub coordinates) |
| Config generation (vendor-specific) | No | Yes (LLM on hub) |
| Compliance verification | No | Yes (policy engine on hub) |
| Cost attribution / FinOps | No | Yes (cost engine on hub) |
| Topology graph updates | Partial (local edges) | Yes (full graph on hub) |

### Offline Behavior

When the NATS leaf connection to the hub is severed, the edge agent enters **offline mode**:

```
1. DETECTION
   - NATS leaf node detects connection loss (TCP keepalive + NATS ping/pong)
   - Agent transitions to offline state
   - Agent logs: "Hub connection lost. Operating autonomously."

2. CONTINUED OPERATIONS
   - Discovery scans continue on schedule
   - Health checks continue on schedule
   - Telemetry collection continues; data buffered to DuckDB
   - Pre-queued commands execute from SQLite command queue
   - New device discoveries are stored locally in SQLite

3. BUFFERING
   - Events that would be sent to hub are buffered in SQLite
   - Buffer is bounded by offline_queue_max (default: 10,000 events)
   - Oldest events are dropped when buffer is full (FIFO eviction)
   - Each buffered event has a monotonic sequence number for ordering

4. RECONNECTION
   - NATS leaf node auto-reconnects (unlimited retries, 5s interval)
   - On reconnection:
     a. Agent sends loom.{site_id}.agent.reconnected event
     b. Buffered events are replayed in sequence order
     c. SQLite device state is synced to hub PostgreSQL
     d. DuckDB telemetry is flushed to hub TimescaleDB
     e. Pending commands from hub are pulled and queued
   - Agent logs: "Hub connection restored. Syncing N buffered events."

5. CONFLICT RESOLUTION
   - Hub state wins for: workflow state, desired state, policy
   - Edge state wins for: observed facts (edge saw the device more recently)
   - Merge for: device metadata (union of hub + edge facts, latest timestamp wins)
```

---

## 4. Scaling Dimensions

| Dimension | Bottleneck | Scale Strategy | Threshold |
|-----------|-----------|----------------|-----------|
| **Devices managed** | PostgreSQL row count + adapter concurrency | Vertical (bigger PG) then horizontal (read replicas). Partition device tables by tenant. Adapter concurrency bounded by worker count x goroutine limit. | <1K: single PG. 1K-10K: Patroni HA. >10K: read replicas + partitioning |
| **Concurrent workflows** | Temporal task queue throughput | Add `loom-worker` instances. Each worker polls a Temporal task queue. Temporal distributes activities across workers. | Each worker handles ~50-200 concurrent activities. Add workers linearly. |
| **Events/sec** | NATS JetStream throughput | Add NATS cluster nodes. JetStream supports horizontal scaling. Use stream-per-tenant for isolation. | Single NATS node: ~1M msg/sec. 3-node cluster: ~3M msg/sec. |
| **Telemetry ingestion** | TimescaleDB write throughput | Continuous aggregates reduce query load. Compression (10-20x). Beyond 500K datapoints/sec: add VictoriaMetrics as a dedicated TSDB. | <100K dp/s: TimescaleDB. >500K dp/s: VictoriaMetrics. |
| **API requests** | CPU-bound request validation + DB queries | Add `loom-api` instances behind load balancer. Each instance is stateless. | Each instance: ~2K req/sec. Add instances linearly. |
| **LLM decisions** | GPU throughput (local) or API rate limits (cloud) | Local: add GPU servers, use vLLM with tensor parallelism. Cloud: request rate limit increases. Cache repeated decisions (20-50% hit rate). | 7B model on 1 GPU: ~30-50 tok/s. 70B on 2x A100: ~10-20 tok/s. |
| **Edge sites** | Hub NATS leaf connection capacity + sync bandwidth | Each leaf node is 1 TCP connection. NATS supports thousands of leaf nodes. Stagger sync intervals to avoid thundering herd. | Single hub: ~500 edge sites. Beyond: federated hubs. |
| **Graph queries** | Apache AGE query complexity (>4 hops) | For most queries (<4 hops): AGE is sufficient. For complex graph algorithms (PageRank, community detection): add Neo4j read replica. | <10K nodes: AGE. >10K nodes with GDS: Neo4j. |
| **Vector search** | pgvector index size | For <1M embeddings: pgvector with HNSW index. Beyond: dedicated Qdrant instance. | <1M vectors: pgvector. >5M vectors: Qdrant. |

### Scaling Diagram

```
Devices     Workers     NATS Nodes    PG Replicas    Edge Sites
────────    ────────    ──────────    ───────────    ──────────
  <100         1            1             0              0       Mode 1
   500         3            3             1              0       Mode 2 (small)
  2000        10            3             2              0       Mode 2 (medium)
  5000        20            5             2             10       Mode 3
 10000        40            5             3             50       Mode 3 (large)
 50000       100            7             5            200       Mode 3 (enterprise)
```

---

## 5. Failure Modes & Recovery

### Mode 1: Single Binary

| Component Failure | Impact | Recovery | Data Loss Risk | RTO | RPO |
|-------------------|--------|----------|----------------|-----|-----|
| **LOOM process crash** | Total outage — all functions stop | Restart process. Temporal replays incomplete workflows from PG history. NATS JetStream replays from file store. | None (all state in PG + JetStream files) | <30s (systemd restart) | 0 (PG is durable) |
| **PostgreSQL crash** | Total outage — LOOM cannot start without PG | Restart PG. If data corruption: restore from backup. | Potential loss of last few uncommitted transactions | 1-5 min (PG restart) | Seconds (WAL) |
| **Disk full** | PG and NATS JetStream stop accepting writes | Free disk space. Purge old telemetry/events. | None if caught before PG corruption | Minutes | 0 |
| **OOM kill** | Process killed by OS | Restart. Reduce worker concurrency or increase RAM. | None (state in PG) | <30s | 0 |

### Mode 2: Distributed

| Component Failure | Impact | Recovery | Data Loss Risk | RTO | RPO |
|-------------------|--------|----------|----------------|-----|-----|
| **loom-api instance** | Reduced API capacity. LB routes to surviving instances. | Auto-recovery. LB health checks detect failure. Start replacement. | None (stateless) | 0 (immediate failover) | 0 |
| **loom-worker instance** | Reduced workflow throughput. Temporal redistributes activities. | Temporal detects worker timeout. Activities re-dispatched to surviving workers. Start replacement. | None (Temporal replays) | <60s (activity timeout) | 0 |
| **Temporal server** | New workflows cannot start. Running workflows pause. | Temporal replicas take over (if HA). Restart failed replica. | None (state in PG) | <30s (replica failover) | 0 |
| **NATS node** | Cluster remains operational with 2/3 nodes. Events continue flowing. | Restart failed node. NATS cluster self-heals. | None (JetStream R3 replication) | 0 (cluster handles) | 0 |
| **PostgreSQL primary** | Writes stop. Reads may continue on replicas. | Patroni promotes replica to primary. Application reconnects. | Last few uncommitted txns (synchronous replication avoids this) | <30s (Patroni failover) | 0 (sync replication) or seconds (async) |
| **PostgreSQL replica** | Reduced read capacity. No write impact. | Rebuild replica from primary. | None | Minutes (rebuild) | 0 |
| **Valkey node** | Cache misses increase. Slightly higher PG load. | Restart or replace. Cache refills automatically. | None (cache is ephemeral) | 0 (graceful degradation) | N/A |
| **All loom-api instances** | Complete API outage. Running workflows continue. | Start new instances. LB routes when healthy. | None | Minutes | 0 |
| **All loom-worker instances** | Workflows stall. No new activities execute. | Start new workers. Temporal replays all in-flight activities. | None | Minutes | 0 |
| **Full NATS cluster** | Events stop flowing. Edge agents enter offline mode. | Restart cluster. JetStream recovers from file store. | Events published during full outage are lost unless edge-buffered. | Minutes | Seconds to minutes |
| **Full PostgreSQL cluster** | Total data outage. Everything stops. | Restore from backup. Point-in-time recovery with WAL. | Transactions since last backup (if no offsite WAL). | 30 min - hours | Depends on backup frequency |

### Mode 3: Hub + Edge

| Component Failure | Impact | Recovery | Data Loss Risk | RTO | RPO |
|-------------------|--------|----------|----------------|-----|-----|
| **Hub total failure** | Edge agents continue autonomously. No new workflows, no LLM decisions, no cross-site coordination. | Restore hub from backup. Edge agents auto-reconnect and sync. | Hub data per backup schedule. Edge data preserved locally. | Hours (hub restore) | Hub: backup interval. Edge: 0 |
| **Edge agent crash** | Site loses monitoring and command execution. Hub loses visibility into site. | Restart agent. SQLite state survives restart. Resumes operations. | None (SQLite is durable) | <30s (systemd restart) | 0 |
| **WAN link down** | Edge enters offline mode. Hub marks site as unreachable. | Automatic. Agent buffers events. NATS leaf auto-reconnects when link restores. | Events dropped if buffer exceeds offline_queue_max | 0 (agent autonomous) | Events beyond buffer size |
| **Edge disk failure** | Agent cannot persist state. Must rebuild. | Replace disk. Agent re-discovers all local devices on startup. | Local state history lost. Hub retains last-synced state. | Minutes to hours | Since last sync |
| **Hub NATS failure** | All edge agents enter offline mode simultaneously. | Restore hub NATS. All agents reconnect and sync. | Buffered events at each edge. | Minutes to hours | Buffer size |

### Mode 4: Air-Gapped (Additional Failure Modes)

| Component Failure | Impact | Recovery | Data Loss Risk | RTO | RPO |
|-------------------|--------|----------|----------------|-----|-----|
| **Local LLM server** | No LLM decisions. Workflows requiring LLM reasoning stall. | Restart LLM server. If GPU failure: fall back to CPU-only inference (llama.cpp). | None (LLM is stateless) | Minutes | 0 |
| **Internal CA** | Cannot issue new TLS certificates. Existing certs continue working until expiry. | Restore CA from backup. Re-issue certificates if CA key compromised. | CA state since last backup | Hours | Backup interval |
| **HSM / KMS failure** | Cannot decrypt credentials. Operations requiring credential access fail. | Replace HSM. Restore KMS from backup keys. | None if backup keys exist | Hours | 0 (keys are recoverable) |
| **GPU hardware failure** | LLM throughput reduced or eliminated. | Fail over to CPU-only inference (slower). Replace GPU. | None | Minutes (failover) | 0 |

### RTO/RPO Summary

| Deployment Mode | Target RTO | Target RPO | Backup Strategy |
|----------------|-----------|-----------|-----------------|
| Mode 1 | <5 min | <1 min | pg_dump daily + WAL archiving |
| Mode 2 | <1 min (per-component) | 0 (sync replication) | Patroni streaming replication + WAL archiving + daily base backup |
| Mode 3 Hub | <30 min (full hub) | <5 min | Same as Mode 2 + edge state survives independently |
| Mode 3 Edge | <30s | 0 (local SQLite) | SQLite is the backup (edge is disposable — re-discovery rebuilds state) |
| Mode 4 | Same as Mode 2/3 | Same as Mode 2/3 | Same as Mode 2/3 + offline backup to tape/external media |

---

## 6. Resource Requirements

### Mode 1: Single Binary (Development / Small Sites)

```
┌─────────────────────────────────────────────────────────────┐
│  MINIMUM (< 50 devices, development)                         │
│                                                              │
│  CPU:    4 cores (x86_64 or ARM64)                           │
│  RAM:    8 GB                                                │
│           - LOOM process: ~2 GB                              │
│           - PostgreSQL: ~2 GB (shared_buffers)               │
│           - Embedded NATS: ~500 MB                           │
│           - Embedded Temporal: ~500 MB                       │
│           - OS + headroom: ~3 GB                             │
│  Disk:   50 GB SSD                                           │
│           - PostgreSQL data: ~10 GB                          │
│           - NATS JetStream: ~5 GB                            │
│           - Logs + telemetry: ~10 GB                         │
│           - OS + binaries: ~25 GB                            │
│  Network: 1 Gbps (management network access)                │
│  GPU:    None required (CPU-only LLM or cloud API)          │
│                                                              │
│  RECOMMENDED (50-100 devices)                                │
│                                                              │
│  CPU:    8 cores                                             │
│  RAM:    16 GB                                               │
│  Disk:   100 GB NVMe SSD                                    │
│  GPU:    Optional — any 8GB+ VRAM GPU for local 7B model    │
└─────────────────────────────────────────────────────────────┘
```

**Runs on:** Laptop, NUC, single VM, Raspberry Pi 5 (8GB) for very small sites.

### Mode 2: Distributed (Production)

Per-component specifications for a 1,000-device deployment:

```
┌─────────────────────────────────────────────────────────────┐
│  loom-api (per instance, run 2-3)                            │
│  CPU:    4 cores                                             │
│  RAM:    4 GB                                                │
│  Disk:   20 GB SSD (logs only, stateless)                   │
├─────────────────────────────────────────────────────────────┤
│  loom-worker (per instance, run 3-10)                        │
│  CPU:    8 cores                                             │
│  RAM:    8 GB                                                │
│           - Adapter connections: ~50 MB per 100 devices      │
│           - Temporal worker cache: ~1 GB                     │
│  Disk:   20 GB SSD                                           │
├─────────────────────────────────────────────────────────────┤
│  Temporal Server (per replica, run 2-3)                       │
│  CPU:    4 cores                                             │
│  RAM:    4 GB                                                │
│  Disk:   20 GB SSD                                           │
├─────────────────────────────────────────────────────────────┤
│  NATS (per node, run 3)                                      │
│  CPU:    2 cores                                             │
│  RAM:    4 GB                                                │
│           - JetStream in-memory buffer: ~2 GB                │
│  Disk:   50 GB NVMe SSD (JetStream file store)              │
├─────────────────────────────────────────────────────────────┤
│  PostgreSQL Primary                                          │
│  CPU:    8 cores                                             │
│  RAM:    32 GB                                               │
│           - shared_buffers: 8 GB                             │
│           - work_mem: 256 MB                                 │
│           - effective_cache_size: 24 GB                      │
│  Disk:   500 GB NVMe SSD (RAID 10 recommended)              │
│           - Device inventory: ~1 GB per 10K devices          │
│           - TimescaleDB telemetry: ~10 GB/month (compressed) │
│           - Workflow history: ~5 GB/month                    │
│           - WAL: ~50 GB reserved                             │
├─────────────────────────────────────────────────────────────┤
│  PostgreSQL Replica (run 1-2)                                │
│  CPU:    4 cores                                             │
│  RAM:    16 GB                                               │
│  Disk:   500 GB NVMe SSD (matches primary)                  │
├─────────────────────────────────────────────────────────────┤
│  Valkey (per node, run 3)                                    │
│  CPU:    2 cores                                             │
│  RAM:    4 GB (2 GB for data, 2 GB for OS/overhead)         │
│  Disk:   10 GB SSD (AOF persistence)                        │
├─────────────────────────────────────────────────────────────┤
│  TOTAL for 1,000-device Mode 2 deployment:                   │
│                                                              │
│  Nodes:  12-20 (VMs or bare metal)                          │
│  CPU:    64-120 cores total                                  │
│  RAM:    120-200 GB total                                    │
│  Disk:   1.5-3 TB NVMe SSD total                            │
│  Network: 10 Gbps between components                         │
└─────────────────────────────────────────────────────────────┘
```

### Mode 3: Hub + Edge

**Hub:** Same as Mode 2 above, plus NATS configured with leaf node listener port.

**Edge agent (per site):**

```
┌─────────────────────────────────────────────────────────────┐
│  Edge Agent — Minimum (< 20 devices at site)                 │
│                                                              │
│  CPU:    2 cores (ARM64 or x86_64)                           │
│  RAM:    2 GB                                                │
│           - loom-agent process: ~500 MB                      │
│           - Embedded NATS leaf: ~200 MB                      │
│           - SQLite + DuckDB: ~300 MB                         │
│           - OS: ~1 GB                                        │
│  Disk:   16 GB SD card or SSD                                │
│  Network: 100 Mbps (management VLAN)                         │
│  WAN:    1 Mbps minimum to hub (events + sync)              │
│                                                              │
│  Runs on: Raspberry Pi 4/5 (4 GB), Intel NUC, any small VM  │
├─────────────────────────────────────────────────────────────┤
│  Edge Agent — Standard (20-100 devices at site)              │
│                                                              │
│  CPU:    4 cores                                             │
│  RAM:    4 GB                                                │
│  Disk:   32 GB SSD                                           │
│  Network: 1 Gbps (management VLAN)                           │
│  WAN:    10 Mbps to hub                                      │
│                                                              │
│  Runs on: Intel NUC, small 1U server, VM on existing infra   │
├─────────────────────────────────────────────────────────────┤
│  Edge Agent — Large (100-500 devices at site)                │
│                                                              │
│  CPU:    8 cores                                             │
│  RAM:    8 GB                                                │
│  Disk:   100 GB SSD                                          │
│  Network: 1 Gbps                                             │
│  WAN:    50 Mbps to hub                                      │
│                                                              │
│  Runs on: 1U server, dedicated VM                            │
└─────────────────────────────────────────────────────────────┘
```

### LLM Hosting: GPU Requirements for On-Premises Models

```
┌─────────────────────────────────────────────────────────────┐
│  TIER 1: CPU-Only (no GPU, minimum viable)                   │
│                                                              │
│  Model:    Llama 4 8B (Q4 quantized) via llama.cpp           │
│  Hardware: Any server with 16 GB RAM, no GPU                 │
│  Speed:    5-10 tokens/sec                                   │
│  Use for:  Config validation, protocol selection, basic      │
│            reasoning. Sufficient for <100 decisions/day.     │
│  Cost:     $0 incremental (runs on existing hardware)       │
├─────────────────────────────────────────────────────────────┤
│  TIER 2: Single Consumer GPU                                 │
│                                                              │
│  Model:    Llama 4 8B or Qwen 2.5 14B (Q4) via Ollama       │
│  Hardware: 1x NVIDIA RTX 4090 (24 GB VRAM)                  │
│            or Apple M2 Pro/Max (16-32 GB unified memory)     │
│  Speed:    20-50 tokens/sec                                  │
│  Use for:  All LLM tasks for <1,000 devices.                │
│  Cost:     ~$1,600 (RTX 4090) or $2,000-3,500 (Mac)        │
├─────────────────────────────────────────────────────────────┤
│  TIER 3: Data Center GPU (Production)                        │
│                                                              │
│  Model:    Llama 4 70B or DeepSeek-V3 via vLLM               │
│  Hardware: 2x NVIDIA A100 80GB or 1x H100 80GB              │
│  Speed:    10-25 tokens/sec (70B), 30-50 tokens/sec (8B)    │
│  Use for:  Production workloads, 1,000+ devices,            │
│            complex multi-step reasoning, config generation.  │
│  Cost:     ~$15,000-30,000 per GPU server                   │
├─────────────────────────────────────────────────────────────┤
│  TIER 4: Multi-GPU Cluster (Enterprise / Air-Gapped)         │
│                                                              │
│  Model:    DeepSeek-V3 (full) or multiple models in parallel │
│  Hardware: 4-8x A100/H100 GPUs across 1-2 servers           │
│  Speed:    50-100+ tokens/sec with vLLM tensor parallelism   │
│  Use for:  10,000+ devices, high-throughput decision making, │
│            multiple concurrent LLM requests.                 │
│  Cost:     ~$50,000-120,000 per server                      │
└─────────────────────────────────────────────────────────────┘
```

### Quick Reference: What Do I Need?

| Scenario | Mode | Hardware | Estimated Cost |
|----------|------|----------|---------------|
| Home lab, 10 devices | Mode 1 | Raspberry Pi 5 (8GB) + external PG | ~$100 |
| Small office, 50 devices | Mode 1 | Intel NUC or spare server | ~$500 |
| Single data center, 500 devices | Mode 2 | 8-12 VMs on existing infra | VMs only |
| Single data center, 2,000 devices | Mode 2 | 15-20 VMs + 1 GPU server for LLM | ~$20,000 |
| 5 sites, 1,000 devices total | Mode 3 | Hub (Mode 2 small) + 5 RPi/NUC edge agents | ~$5,000 |
| 50 sites, 10,000 devices total | Mode 3 | Hub (Mode 2 medium) + 50 edge agents + GPU server | ~$40,000 |
| Air-gapped gov't, 5,000 devices | Mode 4 | Mode 2 + 2x A100 GPU server + HSM + internal CA | ~$80,000 |

---

## Appendix A: Port Map

| Service | Default Port | Protocol | Notes |
|---------|-------------|----------|-------|
| loom-api | 8080 | HTTP/HTTPS | REST API, SSE, WebSocket |
| loom-api (gRPC) | 8081 | gRPC | Optional gRPC API |
| loom-agent metrics | 9090 | HTTP | Prometheus metrics endpoint |
| NATS client | 4222 | NATS | Client connections |
| NATS cluster | 6222 | NATS | Inter-node cluster traffic |
| NATS leaf | 7422 | NATS+TLS | Edge agent leaf node connections |
| NATS monitoring | 8222 | HTTP | NATS monitoring/health |
| Temporal frontend | 7233 | gRPC | Temporal client connections |
| Temporal UI | 8233 | HTTP | Temporal web dashboard |
| PostgreSQL | 5432 | PostgreSQL | Database connections |
| Valkey | 6379 | RESP | Cache connections |
| LLM (Ollama) | 11434 | HTTP | OpenAI-compatible API |
| LLM (vLLM) | 8000 | HTTP | OpenAI-compatible API |
| LLM (llama.cpp) | 8080 | HTTP | OpenAI-compatible API |
| Next.js frontend | 3000 | HTTP | Development server (or embedded via go:embed) |

## Appendix B: Configuration Environment Variables

```bash
# Database
LOOM_DB_URL="postgres://loom:secret@localhost:5432/loom?sslmode=require"

# NATS
LOOM_NATS_URLS="nats://nats1:4222,nats://nats2:4222,nats://nats3:4222"
LOOM_NATS_EMBEDDED=false          # true for Mode 1

# Temporal
LOOM_TEMPORAL_HOST="temporal:7233"
LOOM_TEMPORAL_NAMESPACE="loom"
LOOM_TEMPORAL_EMBEDDED=false      # true for Mode 1

# Valkey
LOOM_VALKEY_ADDRS="valkey1:6379,valkey2:6379,valkey3:6379"

# LLM
LOOM_LLM_ENDPOINT="http://localhost:11434/v1"
LOOM_LLM_MODEL="llama4:8b"
LOOM_LLM_API_KEY=""               # empty for local models

# Edge agent
LOOM_AGENT_SITE_ID="site-denver-01"
LOOM_AGENT_HUB_NATS="nats://hub:7422"
LOOM_AGENT_LOCAL_DB="/var/lib/loom/agent.db"

# Security
LOOM_TLS_CERT="/etc/loom/certs/server.crt"
LOOM_TLS_KEY="/etc/loom/certs/server.key"
LOOM_KMS_TYPE="file"              # "file", "aws", "gcp", "azure", "hsm"
LOOM_KMS_PATH="/etc/loom/kms/keyring.enc"
```
