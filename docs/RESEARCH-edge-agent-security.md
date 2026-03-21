# Research: Edge Agent Security Model

> **Status:** Resolved — all design decisions made, specification written
> **Addresses:** GAP-ANALYSIS.md findings C1, C4, C5
> **Specification:** [EDGE-AGENT-SECURITY-SPEC.md](EDGE-AGENT-SECURITY-SPEC.md)
> **Owner:** Security Architecture
> **Last updated:** 2026-03-21

---

## 1. Problem Statement

The edge agent operates autonomously at remote sites with intermittent connectivity. Three critical security gaps exist in the current design:

1. **Credential caching** — The agent must authenticate to local devices while offline, but no local encryption, KEK management, or hardware binding is specified.
2. **Auto-update trust** — A single Ed25519 signing key with no multi-signature, staged rollout, or rollback mechanism. Compromised key = code execution on the entire fleet.
3. **Split-brain conflict resolution** — The "hub wins for workflow state, edge wins for observed facts" rule is insufficient for competing mutations during partition.

These gaps affect the foundational trust model of every edge deployment.

---

## 2. Edge Agent Credential Cache

### 2.1 Current State

- Credentials are resolved from `CredentialRef` via the hub's vault at connect time
- Vault architecture specifies memory protections (`mlock`, `PR_SET_DUMPABLE=0`, explicit zeroing) for the control plane process
- No document addresses how credentials survive offline periods on edge agents
- Edge agent uses SQLite for local state — no mention of encryption at rest

### 2.2 Threat Model

| Threat | Impact | Likelihood |
|--------|--------|------------|
| Physical theft of edge device | All device credentials for that site exposed | Medium (remote sites have variable physical security) |
| Root access via OS vulnerability | SQLite database readable, memory dumpable | Medium |
| Supply chain compromise of edge hardware | Pre-installed exfiltration before deployment | Low |
| Disk forensics after decommissioning | Credentials recoverable from unwiped storage | Medium |

### 2.3 Questions to Answer

1. **Where are credentials stored offline?** Options:
   - (a) In SQLite, encrypted with a local DEK wrapped by a per-agent KEK derived from a hardware secret
   - (b) In a separate encrypted file (like a local vault), memory-mapped with `mlock`
   - (c) In the agent's memory only (lost on restart — requires hub reconnect to re-fetch)

2. **Can we bind to hardware?** TPM 2.0 is available on most x86 edge hardware and Raspberry Pi 5 (via external module). Options:
   - TPM-sealed KEK that can only be unsealed on the specific device
   - LUKS full-disk encryption with TPM-backed key (OS-level, not LOOM-specific)
   - ARM TrustZone secure enclave on ARM devices

3. **What is the blast radius?** If one edge agent is compromised:
   - Only credentials for devices at that site are exposed (good — site-scoped)
   - But those credentials may be shared with other sites (e.g., a common SNMP community string or shared admin password across sites)
   - Credential rotation for the compromised site must be automated

4. **Credential revocation on decommission:**
   - Agent decommission must trigger credential rotation for all devices at that site
   - The hub must track which credentials have been distributed to which agents
   - Revoked agents must be unable to authenticate to the hub (certificate revocation)

### 2.4 Recommended Design

```
┌─────────────────────────────────────┐
│          Edge Agent                  │
│                                      │
│  ┌──────────────┐  ┌──────────────┐ │
│  │ Agent KEK    │  │ Credential   │ │
│  │ (TPM-sealed  │──│ Cache        │ │
│  │  or derived  │  │ (encrypted   │ │
│  │  from hub)   │  │  SQLite col) │ │
│  └──────────────┘  └──────────────┘ │
│         │                            │
│  ┌──────────────┐                    │
│  │ Credential   │                    │
│  │ Manifest     │                    │
│  │ (what was    │                    │
│  │  distributed)│                    │
│  └──────────────┘                    │
└─────────────────────────────────────┘
```

**Proposal:**
- Each agent receives a unique KEK from the hub at provisioning time
- KEK is sealed to the TPM (if available) or derived from a passphrase entered at first boot
- Credential cache uses SQLite with per-column encryption (AES-256-GCM, DEK per credential, wrapped by agent KEK)
- Hub maintains a **credential manifest** per agent: which credentials were distributed, when, and their version
- On agent decommission: hub marks all distributed credentials for rotation, revokes agent certificate
- Credential cache has a configurable maximum offline duration (default: 7 days). After expiry, cached credentials are wiped and the agent enters degraded mode (monitoring only, no mutations)

### 2.5 Resolved Questions

> All questions resolved in [EDGE-AGENT-SECURITY-SPEC.md](EDGE-AGENT-SECURITY-SPEC.md) Section 2.

- [x] Is TPM 2.0 a hard requirement for edge agents, or is it a recommended enhancement? **Decision: RECOMMENDED, not required. Fallback: Argon2id passphrase-derived KEK.**
- [x] What is the minimum viable local encryption for agents without TPM (passphrase-derived KEK)? **Decision: Argon2id (time=3, memory=256MiB, threads=4, keyLen=32) with passphrase delivered via systemd LoadCredential.**
- [x] Should credential cache wipe be time-based, or event-based (e.g., wipe if tamper detection triggers)? **Decision: Both. Time-based (7-day max offline duration) AND event-based (TPM PCR mismatch, integrity hash failure, hub-initiated revocation).**
- [x] How does the hub authenticate the agent's credential requests to prevent a rogue agent from fetching credentials for a different site? **Decision: mTLS with SiteID in certificate SAN (URI: loom://site/{site_id}/agent/{agent_id}). Hub extracts SiteID from cert and enforces scope.**

---

## 3. Auto-Update Security

### 3.1 Current State

- Edge agent has an auto-update component that checks the hub for new versions
- Binary verification uses Ed25519 signatures
- Public keys are "baked into the current binary"
- No multi-signature, no staged rollout, no rollback, no human approval gate

### 3.2 Threat Model

| Threat | Impact | Likelihood |
|--------|--------|------------|
| Signing key compromise | Attacker pushes malicious binary to entire fleet | Low (but catastrophic) |
| Hub compromise | Attacker serves malicious binary via legitimate update channel | Low-Medium |
| MITM on update channel | Binary replacement in transit | Low (TLS protects, but certificate pinning not specified) |
| Malicious insider | Legitimate signer pushes backdoored binary | Low |

### 3.3 Questions to Answer

1. **Multi-signature:** Should updates require M-of-N signatures (e.g., 2-of-3)? This prevents single-key compromise from being fleet-wide.

2. **Staged rollout:** Should updates deploy to a canary set (e.g., 5% of agents) first, with health checks before fleet-wide rollout? What health checks determine success?

3. **Rollback:** If an updated agent fails health checks, can it revert to the previous binary? Where is the previous binary stored? What if the update corrupts the rollback mechanism?

4. **Key rotation:** How do you rotate the signing key without redeploying the entire fleet? The current design (key baked into binary) creates a chicken-and-egg problem.

5. **Human approval:** Should the hub require an admin to approve an update before it is served to agents? Or is fully automated acceptable?

### 3.4 Recommended Design: TUF (The Update Framework)

[TUF](https://theupdateframework.io/) is the industry standard for secure software update systems, used by Docker, PyPI, and Uptane (automotive). It addresses all identified gaps:

| Gap | TUF Solution |
|-----|-------------|
| Single signing key | Threshold signatures (M-of-N) on the targets role |
| Key rotation | Root role rotation with cross-signing — old root signs new root |
| Rollback protection | Snapshot metadata with version numbers — agents reject downgrades |
| MITM/replay | Timestamp role with short expiry — stale metadata is rejected |
| Compromised repo | Consistent snapshots — metadata and targets are atomically consistent |

**Proposed integration:**
- Hub serves a TUF repository (metadata + agent binaries)
- Agent embeds a TUF client that verifies metadata chain before accepting updates
- Root keys stored offline (ceremony-based rotation, like PKI root CA)
- Targets key used for day-to-day signing (can be rotated via TUF delegation)
- Staged rollout via TUF custom metadata: agents check their rollout group before applying

### 3.5 Staged Rollout Design

```
Update phases:
1. Build & sign binary (CI pipeline)
2. Publish to TUF repository with rollout metadata: { canary: true, percentage: 5 }
3. Canary agents (tagged at provisioning) pick up update
4. Health check period (configurable, default: 1 hour)
   - Agent self-reports: boot success, connectivity to hub, device reachability
5. If canary healthy: operator approves (or auto-promote after 24h)
6. Rollout metadata updated: { percentage: 25 → 50 → 100 }
7. Each stage has a health gate before advancing

Rollback:
- Agent keeps previous binary at /opt/loom/agent.prev
- Watchdog timer: if agent does not report healthy within 5 minutes of update, systemd restarts with previous binary
- Hub can issue a "rollback" command that triggers fleet-wide revert to previous version
```

### 3.6 Resolved Questions

> All questions resolved in [EDGE-AGENT-SECURITY-SPEC.md](EDGE-AGENT-SECURITY-SPEC.md) Section 3.

- [x] Is TUF acceptable complexity for Phase 1, or should a simpler scheme (multi-sig + staged rollout without TUF) ship first? **Decision: TUF is accepted. Complexity is borne by the CI/release pipeline, not the edge agent. The go-tuf v2 client library is mature. TUF integration ships in Phase 1B.**
- [x] Where are root keys stored? Hardware security module (HSM), or offline on air-gapped machine? **Decision: Air-gapped machine, ceremony-based rotation. Root keys on a dedicated laptop stored in a safe. Each of the 3 targets signing keys on a YubiKey 5 NFC.**
- [x] What is the minimum health check for canary validation? (agent boots + hub connectivity + 1 device reachable?) **Decision: Yes, all three conditions within 5 minutes: boot success + hub NATS connectivity + at least one managed device reachable (ICMP/SNMP/Redfish).**
- [x] Should air-gapped environments use a different update mechanism (USB-delivered TUF repo)? **Decision: Yes, USB-delivered TUF repository. The agent's TUF client verifies the same metadata chain regardless of transport. Hub imports the USB repo via `loom admin update import`.**

---

## 4. Split-Brain Conflict Resolution

### 4.1 Current State

- Conflict resolution rule: "Hub state wins for workflow state, desired state, policy. Edge state wins for observed facts. Merge for device metadata (latest timestamp wins)."
- Offline queue: FIFO eviction when buffer exceeds 10,000 events
- Edge agent has no Temporal client — cannot know about hub-issued workflows during partition

### 4.2 Failure Scenarios

**Scenario 1: Competing mutations**
1. Hub issues teardown workflow for device D (device no longer needed)
2. Edge agent, offline, discovers device D is active and enriches with new facts
3. Partition heals: device is simultaneously "decommissioned" (workflow) and "active" (observed)
4. **No reconciliation protocol defined**

**Scenario 2: Clock skew**
1. Edge site loses NTP for 3 days (Raspberry Pi, no RTC battery)
2. Edge timestamps drift by minutes or hours
3. "Latest timestamp wins" selects the wrong state
4. **No clock skew detection or compensation defined**

**Scenario 3: Critical event eviction**
1. Edge discovers a new critical device (event #1)
2. 10,000 telemetry events follow, evicting event #1 from FIFO queue
3. Partition heals: hub never learns about the critical device
4. **FIFO eviction is the wrong strategy for mixed-priority events**

**Scenario 4: State machine divergence**
1. Edge observes: device transitions active → degraded → unreachable → active
2. Hub, during partition, assumes device is still in its last-known state (active)
3. Edge replays all transitions, but hub's workflows may have made decisions based on stale state
4. **No mechanism to replay hub decisions against the actual state timeline**

### 4.3 Questions to Answer

1. **Priority-based eviction:** Should the offline queue use priority classes instead of FIFO?
   - Priority 0 (never evict): discovery, state transitions, mutations, errors
   - Priority 1 (evict after P2): configuration changes, health status changes
   - Priority 2 (evict first): telemetry, metrics, routine heartbeats

2. **Clock skew handling:**
   - Should agents include monotonic clock offsets alongside wall clock timestamps?
   - Should the hub estimate clock drift based on the agent's last-known NTP sync time?
   - Is NTP a hard requirement for edge sites?

3. **Reconciliation protocol:**
   - When partition heals, should the hub run a "reconciliation workflow" that examines edge state vs. hub state and produces a reconciliation plan?
   - Should this plan require human approval for conflicting mutations?
   - Should the edge agent queue mutations separately from observations, so the hub can identify conflicts?

4. **Autonomous action boundaries:**
   - What is the edge agent allowed to do autonomously during partition? (Observe only? Execute pre-approved workflows? Execute emergency responses?)
   - Should autonomous actions be limited to a configurable policy set at provisioning time?

### 4.4 Recommended Design

```
┌──────────────────────────────────────────────────┐
│              Offline Queue (Priority-Based)        │
│                                                    │
│  P0 (never evict):  discoveries, state changes,   │
│                     mutations, errors              │
│  P1 (evict after P2): config changes, health       │
│  P2 (evict first):  telemetry, metrics, heartbeats │
│                                                    │
│  Max size: 10,000 events (P2 evicted first)        │
│  Overflow: P1 evicted next, P0 triggers disk spill │
└──────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────┐
│              Reconciliation Protocol               │
│                                                    │
│  1. Partition heals — agent syncs with hub         │
│  2. Agent sends: buffered events + clock metadata  │
│  3. Hub detects clock skew from NTP sync timestamp │
│  4. Hub replays edge events against hub state      │
│  5. Conflict detector identifies:                  │
│     - Competing mutations (human approval needed)  │
│     - Stale state assumptions (auto-resolve)       │
│     - Missing events (flagged for investigation)   │
│  6. Reconciliation plan generated                  │
│  7. Plan executed (auto or with approval gate)     │
└──────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────┐
│              Autonomous Action Policy              │
│                                                    │
│  Per-site configurable policy:                     │
│  - observe_only: agent monitors, queues events     │
│  - pre_approved: agent executes pre-listed         │
│    workflow templates (e.g., restart-service)       │
│  - emergency: agent can power-cycle devices        │
│    matching criteria (e.g., health < 10%)          │
│  - full_autonomy: agent executes any queued        │
│    workflow (dangerous, not recommended)            │
└──────────────────────────────────────────────────┘
```

### 4.5 Resolved Questions

> All questions resolved in [EDGE-AGENT-SECURITY-SPEC.md](EDGE-AGENT-SECURITY-SPEC.md) Section 4.

- [x] Should reconciliation be automatic or always require human approval? **Decision: Automatic for non-conflicting changes (metadata merges, new discoveries, telemetry). Human approval required for competing mutations and lifecycle desync conflicts. Reconciliation workflow has a 72-hour timeout before conservative auto-resolve (hub wins).**
- [x] What is the disk spill strategy for P0 events that exceed memory? (Write to local SQLite?) **Decision: Yes, P0 events spill to a dedicated SQLite database (/var/lib/loom/p0-events.db). Max 100,000 disk events. Overflow triggers compaction (oldest 1000 events summarized into a digest), never dropped.**
- [x] How does the reconciliation protocol interact with Temporal? (New workflow? Signal to existing?) **Decision: New Temporal workflow (ReconciliationWorkflow). Uses Temporal's signal-based human approval pattern for conflict resolution. One workflow per reconnecting agent.**
- [x] Should the edge agent maintain a local "decision log" of autonomous actions for hub review? **Decision: Yes. SQLite database at /var/lib/loom/decision-log.db. Every autonomous action recorded with full justification, triggering condition, device state, outcome, and policy rule. Hub reviews on reconnect. Retained 90 days after review.**
- [x] What is the maximum acceptable partition duration before the edge agent should halt all operations? **Decision: No hard halt. Instead, graduated degradation: (1) After 7 days without hub contact, credentials expire and agent enters DEGRADED mode (monitoring only). (2) Clock skew >5 minutes suspends autonomous actions. (3) The agent never fully halts -- passive monitoring (ICMP/ARP) continues indefinitely.**

---

## 5. Implementation Priority

| Item | Phase | Effort | Dependencies |
|------|-------|--------|--------------|
| Local credential encryption (agent KEK + encrypted SQLite) | 1A | 2-3 weeks | Vault architecture finalized |
| Credential manifest (hub tracks distribution) | 1B | 1-2 weeks | Agent provisioning flow |
| TUF integration for auto-update | 1B | 3-4 weeks | CI pipeline, signing ceremony |
| Priority-based offline queue | 1B | 1-2 weeks | NATS event model finalized |
| Reconciliation protocol design | 2 | 2-3 weeks | Temporal workflow patterns |
| Autonomous action policy | 2 | 1-2 weeks | Reconciliation protocol |
| TPM binding (optional enhancement) | 3+ | 2-3 weeks | Hardware availability |
| Staged rollout with canary | 3+ | 2-3 weeks | TUF integration |

---

## 6. References

- [The Update Framework (TUF)](https://theupdateframework.io/) — CNCF graduated project
- [Uptane](https://uptane.github.io/) — TUF for automotive OTA updates
- DEPLOYMENT-TOPOLOGY.md Section 3: Edge Agent Architecture
- VAULT-ARCHITECTURE.md Section 5: Break Glass Emergency Access
- EDGE-CASES.md: RC-04, RC-05, RC-06
- FAILURE-MODES.md Sections 3-4: NATS and Edge failures
