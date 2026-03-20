Now I have comprehensive research data. Let me compile the complete analysis.

---

# LOOM LLM Integration Architecture: Research & Recommendations

## March 2026 State-of-the-Art Analysis

---

## 1. LLM Provider Strategy

### 1.1 Model Comparison for Infrastructure Reasoning

The LLM landscape as of early 2026 has settled into [specialized excellence rather than a single winner](https://lmcouncil.ai/benchmarks). For LOOM's use cases, the relevant breakdown is:

**Claude (Anthropic)**
- Dominates enterprise coding and architectural reasoning. Anthropic holds [54% of the enterprise coding market](https://vertu.com/lifestyle/the-best-llms-for-coding-in-2025-a-comprehensive-guide-to-ai-powered-development/) as of early 2026.
- Claude's extended thinking and tool-use capabilities make it the strongest candidate for complex multi-step infrastructure reasoning (dependency graph decomposition, placement optimization).
- The [Claude Agent SDK](https://platform.claude.com/docs/en/agent-sdk/overview) provides a production-grade agent runtime with built-in tool orchestration, hooks, subagents, and MCP integration -- directly aligned with LOOM's architecture.
- Native MCP support makes Claude the most natural fit for LOOM's tool-integration model.

**GPT-4o / GPT-5 (OpenAI)**
- GPT-5 achieves [perfect scores on AIME 2026](https://lmcouncil.ai/benchmarks) math benchmarks. The o-series reasoning models (o1, o3) excel at pure logic puzzles but cost $60/million output tokens and have [30+ second response times](https://ucstrategies.com/news/chatgpt-vs-claude-which-llm-should-you-choose-in-2026/).
- Strong structured output support with JSON mode and function calling.
- Less suited as the primary model due to cost and latency, but valuable as a secondary reasoning model for complex placement optimization problems where latency is acceptable.

**Open-Source (Llama 4, DeepSeek, Qwen, Mistral)**
- [Llama 4 Scout](https://www.ideas2it.com/blogs/llm-comparison) offers exceptional value: 2600 tokens/s, 0.33s latency, $0.11/$0.34 per 1M tokens, with a 10M token context window.
- [DeepSeek](https://collabnix.com/comparing-top-ai-models-in-2025-claude-grok-gpt-llama-gemini-and-deepseek-the-ultimate-guide/) delivers GPT-4-level performance as open-weights, critical for air-gapped deployments.
- Small fine-tuned models (7B-14B parameters) can be [specialized for network configuration](https://arxiv.org/html/2512.02861v1) tasks with higher accuracy than general-purpose large models.

### 1.2 Recommended Multi-Model Strategy

| Task | Model | Rationale |
|------|-------|-----------|
| Request decomposition & planning | Claude Opus/Sonnet | Best architectural reasoning, tool use |
| Config generation & translation | Fine-tuned SLM (7B-14B) | Higher syntactic accuracy, low latency, cacheable |
| Anomaly detection & correlation | Claude Sonnet | Good at multi-source reasoning |
| Protocol selection | Fine-tuned SLM or cached rules | Deterministic enough to use small models |
| Placement optimization | Claude Opus (or GPT o-series for hardest cases) | Complex multi-constraint reasoning |
| Natural language interface | Claude Sonnet | Best conversational quality + tool use |
| Config validation | Rule engine + SLM | Speed-critical, deterministic where possible |
| Remediation planning | Claude Sonnet with RAG | Needs vendor docs + creative reasoning |

### 1.3 On-Prem / Air-Gapped Hosting

For regulated and air-gapped environments, LOOM needs a self-hosted tier:

- **[vLLM](https://developers.redhat.com/articles/2025/07/08/ollama-or-vllm-how-choose-right-llm-serving-tool-your-use-case)** for production serving: PagedAttention reduces memory fragmentation by 40%+, achieving 793 TPS vs Ollama's 41 TPS -- a 19x throughput difference at scale.
- **[Ollama](https://medium.com/@rosgluk/local-llm-hosting-complete-2025-guide-ollama-vllm-localai-jan-lm-studio-more-f98136ce7e4a)** for development and lightweight deployments.
- Recommended on-prem models: Llama 4 (8B/70B), DeepSeek-V3, Qwen 2.5 variants.
- Hardware requirements: A single NVIDIA A100/H100 (80GB) can serve a 70B parameter model with 4-bit quantization. For production, 2-4 GPUs with vLLM sharding.
- [44% of organizations cite data privacy as the top barrier](https://blog.premai.io/self-hosted-llm-guide-setup-tools-cost-comparison-2026/) to LLM adoption -- on-prem support is a market differentiator, not a nice-to-have.

### 1.4 Cost at Scale

For LOOM processing thousands of decisions/day:

- Claude Sonnet at ~$3/$15 per 1M input/output tokens: ~1000 complex decisions/day with ~2K tokens each = ~$30-90/day.
- Fine-tuned SLM on-prem for routine tasks: hardware amortization only, no per-token cost.
- [Prompt caching reduces costs by up to 90%](https://introl.com/blog/prompt-caching-infrastructure-llm-cost-latency-reduction-guide-2025) for repeated system prompts (vendor documentation, config templates).
- [Semantic caching achieves ~73% cost reduction](https://ai.koombea.com/blog/llm-cost-optimization) in high-repetition workloads -- directly applicable to LOOM's repeated config patterns.
- **Combined strategy (caching + model cascading + SLMs for routine tasks) can achieve [60-80% cost reduction](https://www.hakia.com/tech-insights/llm-inference-optimization/)** vs. naive all-large-model approach.

---

## 2. Architecture Patterns

### 2.1 Agent-Based Architecture (Primary Pattern)

The [Claude Agent SDK](https://letsdatascience.com/blog/claude-agent-sdk-tutorial) has evolved from a coding assistant into a general-purpose agent runtime. Its core philosophy is "give agents a computer, not just a prompt" -- providing controlled access to tools, file systems, and external services.

Key architectural fit for LOOM:
- **Iterative loop**: gather context -> take action -> verify work -> repeat. This directly maps to LOOM's orchestration flow.
- **[Skills/meta-tool architecture](https://leehanchung.github.io/blogs/2025/10/26/claude-skills-deep-dive/)**: A `Skill` tool acts as a dispatcher, injecting specialized instructions into context. LOOM could define skills for each vendor domain (Cisco skill, Juniper skill, cloud placement skill).
- **Hooks system**: Pre/post execution hooks for validation, audit logging, human approval gates.
- **Subagent spawning**: A planner agent can spawn specialized subagents for parallel task execution.

### 2.2 MCP (Model Context Protocol) as Integration Backbone

[MCP](https://modelcontextprotocol.io/docs/learn/architecture) is now the industry standard for AI tool integration, with [97 million monthly SDK downloads](https://www.pento.ai/blog/a-year-of-mcp-2025-review) and first-class support in Claude, ChatGPT, Cursor, Gemini, VS Code, and Copilot.

**Why MCP is the right integration layer for LOOM:**

1. **Universal connector**: Each infrastructure backend (NETCONF, eAPI, REST, gNMI, SNMP) becomes an MCP Server that exposes standardized tools. The LLM doesn't need to know protocol specifics -- it calls tools.
2. **[Tool catalog pattern](https://gregrobison.medium.com/the-model-context-protocol-the-architecture-of-agentic-intelligence-cfc0e4613c1e)**: An "Adapter Hub" MCP server can proxy dozens of vendor APIs, dramatically reducing the combinatorial explosion of connectors.
3. **[November 2025 spec updates](https://medium.com/@dave-patten/mcps-next-phase-inside-the-november-2025-specification-49f298502b03)** add async execution, modern authorization (OAuth 2.1), and enterprise governance -- critical for long-running infrastructure operations.
4. **Bidirectional**: MCP supports server-initiated sampling (the infrastructure can ask the LLM questions) and elicitation (requesting structured input from operators).

**LOOM MCP Server Architecture:**
```
MCP Servers:
  ├── netconf-server      (Cisco IOS-XE, Junos NETCONF operations)
  ├── arista-eapi-server  (Arista eAPI operations)
  ├── mikrotik-server     (MikroTik REST API)
  ├── cloud-server        (AWS/Azure/GCP provisioning)
  ├── telemetry-server    (gNMI/SNMP telemetry retrieval)
  ├── batfish-server      (config verification)
  ├── inventory-server    (CMDB, topology queries)
  ├── cost-server         (pricing, budget queries)
  └── knowledge-server    (RAG retrieval from vendor docs)
```

### 2.3 Multi-Agent Orchestration

[72% of enterprise AI projects now use multi-agent architectures](https://www.adopt.ai/blog/multi-agent-frameworks), up from 23% in 2024. For LOOM, a multi-agent approach maps naturally to its domain decomposition:

**Framework Selection:**

[LangGraph](https://dev.to/pockit_tools/langgraph-vs-crewai-vs-autogen-the-complete-multi-agent-ai-orchestration-guide-for-2026-2d63) is the strongest fit for LOOM:
- Graph-based workflow design treats agent interactions as nodes in a directed graph.
- Exceptional flexibility for conditional logic, branching workflows, and dynamic adaptation.
- Production-grade state management and persistence.
- Built-in support for human-in-the-loop approval nodes.

However, given LOOM's Rust core, the recommended approach is to **use the Claude Agent SDK directly** for agent orchestration (avoiding Python framework dependencies) and implement the graph-based workflow patterns natively in Rust, with MCP as the tool integration layer.

**Proposed Agent Topology:**

```
Planner Agent (Claude Opus/Sonnet)
  ├── Network Agent      (VLAN, routing, firewall configs)
  ├── Compute Agent      (VM/container placement)
  ├── Storage Agent       (volume provisioning)
  ├── Security Agent      (policy compliance, ACL verification)
  ├── Cost Agent          (budget optimization)
  └── Verification Agent  (Batfish validation, dry-run)
```

The Planner Agent decomposes requests into a dependency graph, dispatches subtasks to domain agents, and assembles the final execution plan. Each domain agent uses MCP tools specific to its domain.

### 2.4 RAG (Retrieval-Augmented Generation)

RAG is **essential** for LOOM -- the LLM must be grounded in actual vendor documentation, not training-data knowledge that may be outdated or hallucinated.

**What goes into the RAG corpus:**
- Vendor configuration guides (Cisco IOS-XE, Junos, Arista EOS, MikroTik RouterOS)
- Configuration templates and validated examples
- RFC/standards documents
- Historical deployment records and post-mortems
- Organizational policies and compliance requirements
- Known-issue databases and vendor advisories

**Vector database selection:**

The [market has consolidated around Pinecone, Weaviate, Milvus, and Qdrant](https://dev.to/klement_gunndu_e16216829c/vector-databases-guide-rag-applications-2025-55oj). For LOOM:

- **Qdrant** (Rust-native, self-hostable, excellent for air-gapped): Best fit given LOOM's Rust architecture and on-prem requirements.
- **Milvus** (Kubernetes-native, horizontal scaling): Good alternative for large-scale cloud deployments.
- **Pinecone** (serverless, zero-ops): Good for SaaS/cloud-only LOOM deployments.

**RAG performance targets**: Sub-100ms retrieval latency is the production standard. [HNSW indexing](https://www.zenml.io/blog/vector-databases-for-rag) is the dominant approach for production systems.

### 2.5 Fine-Tuning for Specialized Tasks

The [SLM netconfig research](https://arxiv.org/html/2512.02861v1) demonstrates that fine-tuned small models (7B-14B) achieve **higher syntactic accuracy** (57% correct + 38% partially correct) than large general-purpose models (42% + 12%) for network configuration generation, while reducing latency from 5-7.5 minutes to 1-5 minutes.

**LOOM fine-tuning strategy:**

1. **Config translation models**: Fine-tune Llama 3.2 (8B) or Qwen 2.5 (7B) on paired datasets of (intent, vendor-config) for each supported vendor. Train on curated datasets [grounded in vendor documentation](https://arxiv.org/html/2512.02861v1).
2. **Protocol selection model**: A small classifier fine-tuned on (device-type, capability) -> protocol mappings.
3. **Anomaly classification model**: Fine-tuned on labeled telemetry anomaly datasets.
4. Use [LoRA/QLoRA](https://www.superannotate.com/blog/llm-fine-tuning) for parameter-efficient fine-tuning -- a single GPU can fine-tune a 7B model in hours.

---

## 3. Critical Design Decisions

### 3.1 Preventing Hallucination in Config Generation

This is the highest-risk area. One hallucinated subnet mask, one invented VLAN ID, one fabricated BGP AS number = production outage.

**Multi-layer defense:**

1. **RAG grounding**: Every config generation request retrieves relevant vendor documentation and validated templates. [RAG reduces hallucination by 35-60%](https://aws.amazon.com/blogs/machine-learning/reducing-hallucinations-in-large-language-models-with-custom-intervention-using-amazon-bedrock-agents/).
2. **Structured output with JSON Schema**: Force the LLM to output configs as [structured JSON conforming to strict schemas](https://medium.com/@emrekaratas-ai/structured-output-generation-in-llms-json-schema-and-grammar-based-decoding-6a5c58b698a6), not freeform text. Validate against YANG models for NETCONF devices.
3. **[Guardrails AI](https://github.com/guardrails-ai/guardrails)** validation: Post-generation validators that check syntax, semantic correctness, and policy compliance.
4. **[Batfish](https://batfish.org/) pre-deployment verification**: Analyze generated configs offline to verify routing correctness, ACL behavior, reachability, and redundancy -- all [without requiring device access](https://www.techtarget.com/searchnetworking/feature/Batfish-use-cases-for-network-validation-and-testing).
5. **Config diffing**: Compare generated configs against known-good baselines. Flag any unexpected changes for human review.
6. **Template-constrained generation**: For well-known patterns (VLAN creation, BGP neighbor), use templates with LLM-filled parameters rather than freeform generation. The LLM decides *what* to configure; templates ensure *how* it's expressed.
7. **Multi-model consensus**: For critical changes, [run the same request through 2+ models and compare outputs](https://proactivemgmt.com/blog/2025/03/06/reducing-ai-hallucinations-multi-llm-consensus/). Discrepancies trigger human review.

### 3.2 Human-in-the-Loop vs. Autonomous Decisions

Not all decisions carry equal risk. LOOM should implement a **tiered autonomy model** based on [confidence scoring and risk classification](https://www.comet.com/site/blog/human-in-the-loop/):

| Tier | Risk Level | Examples | Approval | 
|------|-----------|----------|----------|
| **Autonomous** | Low | Read-only queries, status checks, telemetry collection | None |
| **Auto-execute + notify** | Medium | Config backups, VLAN creation on non-production, scaling within pre-approved ranges | Post-hoc notification |
| **Propose + approve** | High | Production config changes, new BGP peers, firewall rule modifications | Human approval required |
| **Plan + review + approve** | Critical | Cross-site migrations, core router changes, security policy modifications | Multi-person review |

**Confidence scoring mechanism:**
- LLM self-assessed confidence (ask the model to rate its confidence 0-1)
- RAG retrieval quality score (were relevant docs found?)
- Template match score (does this match a known pattern vs. novel generation?)
- Historical success rate for this type of change
- Combined score determines tier assignment, with [adaptive sampling](https://kili-technology.com/blog/human-in-the-loop-human-on-the-loop-and-llm-as-a-judge-for-validating-ai-outputs) adjusting thresholds based on observed outcomes.

### 3.3 Latency Management

LLM inference takes 1-10 seconds. Some infrastructure decisions need to be faster.

**[Model cascading/routing](https://arxiv.org/html/2603.04445) strategy:**

```
Request → Router (< 1ms)
  ├── Cache hit?          → Return cached decision (< 10ms)
  ├── Deterministic rule? → Rule engine (< 1ms)
  ├── Simple pattern?     → Fine-tuned SLM (100-500ms)
  └── Complex reasoning?  → Claude Sonnet/Opus (2-10s)
```

- [Routing is 16x faster than cascading](https://arxiv.org/html/2603.04445) because it predicts the right model upfront rather than sequentially escalating.
- Use [LLM gateways like Bifrost (11us overhead)](https://www.getmaxim.ai/articles/top-5-llm-gateways-in-2025-the-definitive-guide-for-production-ai-applications/) or TensorZero (sub-ms P99 latency) for intelligent routing.
- For truly real-time paths (sub-100ms), bypass LLM entirely: use pre-computed decision tables, cached policies, or compiled rule engines. The LLM's role shifts to **authoring and updating** these fast-path rules rather than executing them in real-time.

### 3.4 Caching & Memoization

[31% of LLM queries exhibit semantic similarity to previous requests](https://introl.com/blog/prompt-caching-infrastructure-llm-cost-latency-reduction-guide-2025), representing massive savings potential.

**Multi-tier caching for LOOM:**

1. **Exact match cache**: Identical requests (same device type, same intent) return cached results. Redis or in-process cache.
2. **Semantic cache**: Semantically similar requests (e.g., "create VLAN 100 on switch-A" vs "create VLAN 200 on switch-B") can reuse the same template with parameter substitution. Use embedding similarity.
3. **[Agentic plan cache](https://arxiv.org/abs/2506.14852)**: Cache entire multi-step plans as reusable templates. New research shows this reduces costs by 50% and latency by 27% while maintaining performance.
4. **Prompt prefix cache**: Anthropic's native prompt caching for system prompts containing vendor documentation. [Up to 90% cost reduction and 85% latency reduction](https://introl.com/blog/prompt-caching-infrastructure-llm-cost-latency-reduction-guide-2025).

### 3.5 Context Management for Long-Running Workflows

Infrastructure orchestration can span minutes to hours. Managing LLM context across these workflows requires:

- **Session state persistence**: Store workflow state (dependency graph, completed steps, pending approvals) in a durable store (PostgreSQL, Redis). Resume by reconstructing context from state, not from conversation history.
- **Summarization checkpoints**: After each major phase, have the LLM summarize what was accomplished and what remains. Use the summary as context for the next phase rather than replaying the full history.
- **Scoped context injection**: Each agent call gets only the context relevant to its task (current device, current operation, relevant vendor docs via RAG) rather than the entire workflow history.
- **Llama 4 Scout's 10M token context window** provides a safety net for complex workflows, but context management discipline is still essential for cost and latency.

### 3.6 Evaluation & Feedback Loops

[Evaluation in 2025-2026 has shifted from one-time testing to continuous governance](https://wizr.ai/blog/llm-evaluation-guide/):

**Metrics for LOOM:**

| Metric | What it measures | Target |
|--------|-----------------|--------|
| Config syntax correctness | % of generated configs that parse without errors | >99% |
| Semantic correctness | % that pass Batfish verification | >95% |
| Placement optimality | Cost delta vs. optimal placement (computed post-hoc) | <15% |
| Decision latency | Time from request to actionable plan | <5s for routine, <30s for complex |
| Hallucination rate | % of generated values not grounded in RAG/input | <1% |
| Human override rate | % of auto-approved decisions reversed by operators | <5% |
| Remediation success rate | % of auto-remediation actions that resolved the issue | >80% |

**Feedback infrastructure:**
- Operator thumbs-up/down on generated configs and plans.
- Automatic success/failure tracking of executed changes.
- [Drift detection](https://blog.n8n.io/llm-evaluation-framework/) to identify when model quality degrades.
- [LLM-as-a-judge](https://www.getmaxim.ai/articles/llm-as-a-judge-vs-human-in-the-loop-evaluations-a-complete-guide-for-ai-engineers/) for automated evaluation of decision quality, calibrated against human expert ratings.
- Regression test suites run against every model update.

### 3.7 Model Versioning & Regression Testing

- Pin model versions in production (e.g., `claude-sonnet-4-20250514`, not `claude-sonnet-latest`).
- Maintain a golden test suite of (input, expected_output) pairs covering all supported vendors and operation types.
- Shadow testing: run new model versions in parallel with production, compare outputs, flag regressions before cutover.
- A/B testing for non-critical operations to evaluate new models on real traffic.

---

## 4. Grounding & Safety Architecture

### 4.1 Defense-in-Depth Pipeline

```
Operator Request
  │
  ▼
[1. Intent Parsing] ──── LLM decomposes request
  │
  ▼
[2. RAG Retrieval] ──── Fetch vendor docs, templates, policies
  │
  ▼
[3. Config Generation] ─ LLM generates config (constrained by templates + schemas)
  │
  ▼
[4. Schema Validation] ─ JSON Schema / YANG model validation
  │
  ▼
[5. Guardrails Check] ── Policy compliance, security rules, resource limits
  │
  ▼
[6. Batfish Analysis] ── Offline routing/ACL/reachability verification
  │
  ▼
[7. Diff Review] ─────── Compare against current config, flag unexpected changes
  │
  ▼
[8. Approval Gate] ────── Auto-approve (low risk) or human review (high risk)
  │
  ▼
[9. Dry Run] ──────────── Apply in sandbox/staging if available
  │
  ▼
[10. Execute] ─────────── Apply to production with rollback plan
  │
  ▼
[11. Verify] ──────────── Post-deployment verification (is the change working?)
  │
  ▼
[12. Record] ──────────── Audit trail, feedback loop update
```

### 4.2 Audit Trail

Every LLM decision must be fully traceable:
- **Input**: Original request, RAG-retrieved context, system prompt
- **Output**: Raw LLM response, parsed config, confidence score
- **Validation**: Schema check result, Batfish analysis result, guardrails pass/fail
- **Decision**: Approval tier, approver (human or auto), timestamp
- **Execution**: Device responses, success/failure, rollback actions if any
- **Evaluation**: Post-deployment verification result, operator feedback

Store in append-only audit log (PostgreSQL with immutable audit table, or dedicated audit service).

---

## 5. State Management

### 5.1 Vector Database (Qdrant)

- **Qdrant** is the recommended choice: written in Rust (aligns with LOOM's tech stack), self-hostable (air-gapped support), excellent horizontal scaling.
- Index vendor documentation with hierarchical chunking: section-level for broad retrieval, paragraph-level for precision.
- Hybrid search: combine vector similarity with keyword filtering (vendor name, device type, protocol).
- Incremental indexing: automatically re-index when vendor docs are updated.

### 5.2 Session/Workflow State

- Use PostgreSQL (already likely in LOOM's stack) for workflow state persistence.
- Each orchestration workflow gets a unique session ID.
- State machine tracks: pending -> planning -> approved -> executing -> verifying -> completed/failed.
- LLM context is reconstructed from state, not stored as raw conversation.

### 5.3 Decision Knowledge Base

- Store successful decisions as (intent, context, decision, outcome) tuples.
- Use for: few-shot examples in prompts, pattern matching for caching, institutional knowledge.
- Weight recent decisions higher; decay old patterns.
- This creates a self-improving system where LOOM gets better with use.

---

## 6. Cost Optimization Summary

| Technique | Savings | LOOM Application |
|-----------|---------|-----------------|
| [Prompt caching](https://introl.com/blog/prompt-caching-infrastructure-llm-cost-latency-reduction-guide-2025) | Up to 90% on input tokens | System prompts with vendor docs |
| [Semantic caching](https://ai.koombea.com/blog/llm-cost-optimization) | ~73% for repeated patterns | Config generation for similar devices |
| [Model cascading](https://arxiv.org/html/2603.04445) | 40-60% | SLM for routine, large model for complex |
| [Fine-tuned SLMs](https://arxiv.org/html/2512.02861v1) | 80-95% vs. large model API | On-prem config generation |
| [Agentic plan caching](https://arxiv.org/abs/2506.14852) | 50% cost, 27% latency | Reusing multi-step workflow templates |
| RAG context optimization | 70%+ token reduction | Retrieving only relevant docs vs. full context |

**Projected cost at 1000 decisions/day**: $15-40/day with full optimization stack, vs. $150-400/day with naive large-model-for-everything approach.

---

## 7. Recommended Architecture for LOOM

### Core Design Principles

1. **MCP as the universal integration layer**: Every infrastructure backend is an MCP Server. The LLM interacts with infrastructure exclusively through MCP tools. This decouples the AI reasoning from protocol specifics.

2. **Claude as the primary reasoning engine**: Use Claude Sonnet (via API) for planning, reasoning, and complex decisions. Use Claude's native tool-use and Agent SDK patterns for orchestration.

3. **Fine-tuned SLMs for the hot path**: Train small models for config generation, protocol selection, and anomaly classification. Serve via vLLM on-prem. These handle 60-80% of routine decisions at a fraction of the cost and latency.

4. **Qdrant for RAG**: Rust-native, self-hostable vector database for grounding all LLM decisions in vendor documentation and institutional knowledge.

5. **Tiered autonomy with confidence scoring**: Low-risk decisions execute autonomously; high-risk decisions require human approval. The boundary shifts as the system builds track record.

6. **Defense-in-depth validation**: Every generated config passes through schema validation, guardrails checks, Batfish verification, and diff review before execution.

7. **Continuous evaluation**: Automated metrics, operator feedback, and regression testing ensure quality doesn't degrade.

### High-Level Architecture Diagram

```
┌─────────────────────────────────────────────────────────────────┐
│                     LOOM Orchestrator (Rust)                     │
│                                                                  │
│  ┌──────────┐    ┌──────────────┐    ┌──────────────────────┐   │
│  │ Request   │───▶│ LLM Router   │───▶│ Claude API           │   │
│  │ Parser    │    │ (cache/      │    │ (Sonnet for planning)│   │
│  │           │    │  cascade/    │    └──────────────────────┘   │
│  └──────────┘    │  route)      │    ┌──────────────────────┐   │
│                  │              │───▶│ Fine-tuned SLM (vLLM)│   │
│                  └──────────────┘    │ (config generation)  │   │
│                        │             └──────────────────────┘   │
│                        ▼                                        │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │              MCP Client (built into LOOM core)            │   │
│  └──────────┬──────────┬──────────┬──────────┬──────────────┘   │
│             │          │          │          │                    │
│  ┌──────────▼┐ ┌───────▼──┐ ┌────▼─────┐ ┌─▼────────────┐     │
│  │ NETCONF   │ │ eAPI     │ │ REST     │ │ Telemetry    │     │
│  │ MCP Server│ │MCP Server│ │MCP Server│ │ MCP Server   │     │
│  └───────────┘ └──────────┘ └──────────┘ └──────────────┘     │
│                                                                  │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │ Validation Pipeline                                       │   │
│  │  Schema Check → Guardrails → Batfish → Diff → Approve    │   │
│  └──────────────────────────────────────────────────────────┘   │
│                                                                  │
│  ┌────────────┐  ┌────────────┐  ┌─────────────────────────┐   │
│  │ Qdrant     │  │ PostgreSQL │  │ Redis                   │   │
│  │ (RAG)      │  │ (state,    │  │ (cache, sessions)       │   │
│  │            │  │  audit)    │  │                         │   │
│  └────────────┘  └────────────┘  └─────────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
```

### Implementation Priority

1. **Phase 1**: MCP server framework + Claude API integration + basic RAG with Qdrant. Get the tool-use loop working end-to-end for a single vendor (e.g., Cisco IOS-XE via NETCONF).
2. **Phase 2**: Validation pipeline (schema validation + Batfish integration). Human-in-the-loop approval workflow. Multi-vendor support.
3. **Phase 3**: Fine-tuned SLMs for config generation. Caching layers (prompt cache, semantic cache, plan cache). Model routing/cascading.
4. **Phase 4**: Multi-agent orchestration with specialized domain agents. Continuous evaluation and feedback loops. On-prem/air-gapped deployment option with vLLM.
5. **Phase 5**: Institutional knowledge accumulation. Adaptive autonomy (system learns when it can be trusted). Cross-domain anomaly correlation.

### Key Technology Choices Summary

| Component | Choice | Alternative |
|-----------|--------|-------------|
| Primary LLM | Claude Sonnet/Opus (API) | GPT-4o for specific tasks |
| On-prem LLM | Llama 4 / DeepSeek via vLLM | Qwen 2.5 variants |
| Fine-tuning base | Llama 3.2 8B / Qwen 2.5 7B | Mistral 7B |
| Tool integration | MCP (Model Context Protocol) | Direct function calling |
| Vector DB (RAG) | Qdrant | Milvus, pgvector |
| Agent framework | Claude Agent SDK + native Rust orchestration | LangGraph (if Python acceptable) |
| Config verification | Batfish | Custom YANG validators |
| Guardrails | Guardrails AI + custom validators | NeMo Guardrails |
| LLM Gateway | TensorZero or Bifrost | LiteLLM |
| Caching | Redis (exact) + Qdrant (semantic) | Momento, custom |

---

### Sources

- [Claude Agent SDK Overview](https://platform.claude.com/docs/en/agent-sdk/overview)
- [Claude Agent SDK Best Practices](https://skywork.ai/blog/claude-agent-sdk-best-practices-ai-agents-2025/)
- [Claude Agent SDK Tutorial](https://letsdatascience.com/blog/claude-agent-sdk-tutorial)
- [Claude Skills Deep Dive](https://leehanchung.github.io/blogs/2025/10/26/claude-skills-deep-dive/)
- [MCP Architecture Overview](https://modelcontextprotocol.io/docs/learn/architecture)
- [MCP Enterprise Adoption Guide](https://guptadeepak.com/the-complete-guide-to-model-context-protocol-mcp-enterprise-adoption-market-trends-and-implementation-strategies/)
- [A Year of MCP: 2025 Review](https://www.pento.ai/blog/a-year-of-mcp-2025-review)
- [MCP November 2025 Specification](https://medium.com/@dave-patten/mcps-next-phase-inside-the-november-2025-specification-49f298502b03)
- [Multi-Agent Frameworks 2026](https://www.adopt.ai/blog/multi-agent-frameworks)
- [LangGraph vs CrewAI vs AutoGen 2026](https://dev.to/pockit_tools/langgraph-vs-crewai-vs-autogen-the-complete-multi-agent-ai-orchestration-guide-for-2026-2d63)
- [AI Model Benchmarks March 2026](https://lmcouncil.ai/benchmarks)
- [LLM Comparison 2026](https://www.ideas2it.com/blogs/llm-comparison)
- [SLM Netconfig: Fine-Tuned Small Language Models for Network Configuration](https://arxiv.org/html/2512.02861v1)
- [Fine-Tuning Small Language Models for Domain-Specific AI](https://arxiv.org/abs/2503.01933)
- [Self-Hosted LLM Guide 2026](https://blog.premai.io/self-hosted-llm-guide-setup-tools-cost-comparison-2026/)
- [vLLM vs Ollama Comparison](https://developers.redhat.com/articles/2025/07/08/ollama-or-vllm-how-choose-right-llm-serving-tool-your-use-case)
- [LLM Inference Optimization 2026](https://www.hakia.com/tech-insights/llm-inference-optimization/)
- [LLM Cost Optimization Guide](https://ai.koombea.com/blog/llm-cost-optimization)
- [Prompt Caching Infrastructure Guide](https://introl.com/blog/prompt-caching-infrastructure-llm-cost-latency-reduction-guide-2025)
- [Agentic Plan Caching (APC)](https://arxiv.org/abs/2506.14852)
- [Dynamic Model Routing and Cascading Survey](https://arxiv.org/html/2603.04445)
- [Top LLM Gateways 2025](https://www.getmaxim.ai/articles/top-5-llm-gateways-in-2025-the-definitive-guide-for-production-ai-applications/)
- [Vector Databases for RAG 2025](https://dev.to/klement_gunndu_e16216829c/vector-databases-guide-rag-applications-2025-55oj)
- [RAG Infrastructure Production Guide](https://introl.com/blog/rag-infrastructure-production-retrieval-augmented-generation-guide)
- [Reducing AI Hallucinations with Multi-LLM Strategy](https://proactivemgmt.com/blog/2025/03/06/reducing-ai-hallucinations-multi-llm-consensus/)
- [AWS Hallucination Reduction with Bedrock Agents](https://aws.amazon.com/blogs/machine-learning/reducing-hallucinations-in-large-language-models-with-custom-intervention-using-amazon-bedrock-agents/)
- [Batfish Network Configuration Analysis](https://batfish.org/)
- [Batfish Use Cases for Network Validation](https://www.techtarget.com/searchnetworking/feature/Batfish-use-cases-for-network-validation-and-testing)
- [Guardrails AI Framework](https://github.com/guardrails-ai/guardrails)
- [Structured Output JSON Schema for LLMs](https://medium.com/@emrekaratas-ai/structured-output-generation-in-llms-json-schema-and-grammar-based-decoding-6a5c58b698a6)
- [Human-in-the-Loop Review Workflows](https://www.comet.com/site/blog/human-in-the-loop/)
- [LLM-as-a-Judge vs Human-in-the-Loop](https://www.getmaxim.ai/articles/llm-as-a-judge-vs-human-in-the-loop-evaluations-a-complete-guide-for-ai-engineers/)
- [LLM Evaluation Guide 2026](https://wizr.ai/blog/llm-evaluation-guide/)
- [LLM Evaluation Framework Best Practices](https://www.datadoghq.com/blog/llm-evaluation-framework-best-practices/)