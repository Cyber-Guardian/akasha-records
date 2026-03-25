# Idea Brief: Curated Context Architecture ("Curator")

**Date:** 2026-03-06
**Status:** Parked
**Related:** [[2026-03-02-fs1-agent-harness|FS-1 Agent Harness]], [[2026-02-28-agent-swarm-harness|Agent Swarm Harness]], [[2026-03-04-serverless-agent-harness|Serverless Agent Harness]]

## Problem

Long-running autonomous agents degrade over time. Context rots — important information drifts into the attention dead zone (the U-curve's middle), stale observations consume token budget, compaction loses critical detail, and without a human copilot there's nobody to correct drift. Current mitigations (session cycling in FS-1, Letta's sleep-time agents, simple summarization) are reactive and lossy. They manage the symptom (context too big) not the disease (context quality degrades).

**This is specifically a long-running autonomous agent problem.** Copiloted agents (Claude Code, Cursor) don't suffer from it — the human unconsciously curates context every few turns: steering ("no, that's not what I meant"), correcting ("you already tried that"), refocusing ("go back to the original task"), pruning ("forget that approach"). The human IS the context curator. Long-running agents need a synthetic equivalent.

The cost model is inverted between the two modes:
- **Copiloted agents:** Curation overhead is pure cost — the human already provides it free. Adding 2-3s latency kills UX.
- **Long-running agents:** Curation overhead is insurance. One prevented drift saves 30-50 wasted turns. One avoided restart pays for 100 curation passes.

## Core Insight

**Treat the context window as a managed workspace, not an append-only log.** A dedicated curation system actively restructures context between turns — evicting stale content, compressing retained content, reordering for attention optimization, and checking that the agent's actions match its stated intent. The agent sees a clean workspace every turn, not a growing pile of messages.

This is the memory management unit (MMU) for the agent OS.

## What Curator Is

- A **framework-agnostic Python library** providing context lifecycle management for long-running agents
- A **processor pipeline** (following Google ADK's model) that compiles structured state into an optimized working context per LLM call
- Manages external memory (evicted content, retrievable on demand)
- Tracks intent-action-outcome coherence across turns
- Observable — emits metrics on context utilization, drift score, curation events
- Fully serializable for durable execution (Temporal, DBOS, AWS Durable Lambda)

## What Curator Is NOT

- Not a replacement for FS-1, the swarm harness, or the serverless harness — it's a library they consume
- Not for copiloted/interactive agents — humans already provide curation, and the latency cost hurts UX
- Not an orchestrator — Helm handles task decomposition and delegation, Curator handles per-agent context quality
- Not an agent framework — it doesn't manage tools, agent loops, or model selection

## Harness Engineering Context

"The model is the CPU, the context window is RAM, the harness is the operating system." The 2026 consensus (OpenAI, Anthropic, Google, Phil Schmid, Martin Fowler) is that **harness quality — not model capabilities — is the competitive differentiator** for reliable agent systems. Context engineering > model shopping.

Phil Schmid's "Deep Agents" (Agents 2.0) framework identifies four pillars for long-running agents:

| Pillar | Our implementation | Status |
|--------|-------------------|--------|
| Explicit Planning | Plans in memory-bank | Have it |
| Hierarchical Delegation | Helm orchestrator | Have it |
| Persistent Memory | memory-bank + QMD | Have it |
| **Context Engineering** | **Curator** | **Building this** |

Curator fills the missing fourth pillar.

### SOTA Harness Patterns Informing the Design

**Google ADK — Context Compilation Pipeline:** Context is a compiled view over richer state, assembled by named, ordered processors. Each processor is independently testable, replaceable, and observable. This is the architectural model Curator follows — NOT monolithic message rewriting, but a pipeline of composable processors.

**OpenAI Codex — Item/Turn/Thread Primitives:** Atomic I/O units (Items) grouped into agent work units (Turns) inside durable containers (Threads) that support resume/fork/archive. Informs Curator's Block/CurationEvent/Workspace model.

**JetBrains "Complexity Trap" (NeurIPS 2025):** Simple observation masking (hiding verbose tool outputs) is as effective as LLM summarization for agent context management. 50% cost reduction, no quality loss. Validates that Tier 0 (rule-based, free) carries more weight than expected.

**Agent Loop Patterns:** ReAct (interleave reasoning + action) and Plan-and-Execute (decompose then execute) are the production standards. Curator is loop-agnostic — it curates the context regardless of which loop pattern the harness uses.

## Architecture

```
Helm (swarm orchestrator — macro: task decomposition, delegation, merge)
  |
  v
Agent Harness (FS-1, serverless, swarm worker — agent loop, tools, state)
  |
  v
Curator (context quality — micro: curation pipeline, workspace, external memory)
  |
  v
LLM API (Claude, GPT, etc.)
```

### Module Structure (framework-agnostic core + thin adapters)

```
curator/
  core/                    <-- Framework-agnostic, pure Python + Pydantic models
    workspace.py               # Block, Workspace, Zone — structured context model
    pipeline.py                # CurationPipeline — ordered processor chain
    processors/                # Individual pipeline stages
      system_pin.py            #   Pin system blocks, never evict
      staleness.py             #   Track turns-since-referenced
      token_budget.py          #   Enforce token budget
      observation_mask.py      #   Compress verbose tool outputs (Tier 0)
      relevance.py             #   LLM-scored relevance (Tier 1)
      compressor.py            #   Compress warm blocks (Tier 1)
      reorderer.py             #   Attention-aware U-curve placement (Tier 1)
      intent_auditor.py        #   Intent-action-outcome check (Tier 2)
      orientation.py           #   Inject "you are here" summary (Tier 1/2)
      consolidator.py          #   Merge redundancies, extract patterns (Tier 2)
    memory.py                  # External memory store (evicted blocks)
    intent.py                  # Intent-action-outcome tracking
    metrics.py                 # Observability: utilization, drift score, events
    serialization.py           # Workspace <-> flat message list conversion

  curation_agent/            <-- Uses Pydantic AI for structured LLM output
    agent.py                   # Haiku-powered curation calls
    models.py                  # CurationDecision, RelevanceScore, CompressionResult

  adapters/                  <-- Thin integrations per framework
    pydantic_ai.py             # history_processor adapter
    anthropic.py               # Raw Anthropic SDK middleware
    langgraph.py               # State modifier adapter
```

**Why framework-agnostic core:** Curator needs to work with FS-1 (Claude Agent SDK), the serverless harness (Durable Lambda + raw API), the swarm harness (Pydantic AI agents), or anything else. The core is pure Python + Pydantic models. Adapters are thin wrappers (<50 lines each) that plug Curator into whatever agent loop the harness uses.

**Why Pydantic AI for the curation agent:** The curation calls (Tier 1/2) ask a cheap model (Haiku) to produce structured output — relevance scores, compression decisions, eviction lists. Pydantic AI's structured output + model-agnostic support is purpose-built for this. The curation agent is an internal implementation detail, not exposed to harness consumers.

### Processor Pipeline (ADK-inspired)

Instead of monolithic curation, Curator runs a pipeline of named, ordered processors:

```python
pipeline = CurationPipeline([
    # Tier 0 — every turn, rule-based, ~0 cost
    SystemPinProcessor(),          # Pin system blocks, never evict
    StalenessProcessor(),          # Increment turns-since-referenced counters
    TokenBudgetProcessor(),        # Check budget, mark eviction candidates
    ObservationMaskProcessor(),    # Compress verbose tool outputs

    # Tier 1 — every N turns, 1 LLM call (Haiku)
    RelevanceScorer(),             # Score blocks against current task
    Compressor(),                  # Compress warm blocks (3-5x target)
    AttentionReorderer(),          # U-curve: high-relevance to start + end
    OrientationInjector(),         # "You are here" summary at context end

    # Tier 2 — threshold-triggered, 2-3 LLM calls
    IntentAuditor(),               # Intent vs action vs outcome alignment
    Consolidator(),                # Merge redundancies, extract patterns
])
```

Tier 1 and Tier 2 processors are skipped on most turns (controlled by the pipeline scheduler). Each processor receives the Workspace and returns a modified Workspace. Each is independently testable and replaceable.

## Context Model

The internal representation is NOT a flat message list. It's a structured workspace:

```python
class Zone(str, Enum):
    SYSTEM = "system"           # Pinned, never evicted
    HOT = "hot"                 # Active task, highest-relevance findings
    WARM = "warm"               # Compressed recent history
    ORIENTATION = "orientation" # End-of-context supervision notes

class Block(BaseModel):
    content: str
    zone: Zone
    relevance: float = 1.0       # 0-1, scored by curation
    staleness: int = 0           # turns since last referenced
    compression_level: int = 0   # 0=raw, 1=light, 2=heavy
    source_turn: int             # which turn produced this
    block_type: str              # "system_prompt", "tool_result", "assistant", "user", etc.
    metadata: dict = {}          # intent tags, original token count, etc.

class Workspace(BaseModel):
    blocks: list[Block]
    external_memory: ExternalMemory
    intent_log: list[Intent]
    metrics: CurationMetrics
    turn_count: int = 0
    last_light_sleep: int = 0
    last_deep_sleep: int = 0
```

When the LLM API is called, the workspace serializes to a flat message list via attention-optimized ordering:
1. **System zone** — first (highest attention)
2. **Hot zone** — next (high attention, near start)
3. **Warm zone** — middle (lowest attention — acceptable for compressed/background content)
4. **Orientation zone** — last (second-highest attention, read right before generation)

The model sees messages; Curator sees structure. Fully serializable to JSON for durable execution checkpointing.

## Three-Tier Curation Model

### Tier 0 — Heartbeat (every turn, ~0 cost)
Rule-based, no LLM call. Processors: SystemPin, Staleness, TokenBudget, ObservationMask.
- Token budget check: is context approaching threshold?
- Staleness increment: each block's "turns since referenced" counter ticks up
- TTL eviction: blocks past staleness threshold -> external memory
- Observation masking: compress verbose tool outputs (JetBrains NeurIPS 2025 validated: 50% cost reduction, no quality loss)

### Tier 1 — Light Sleep (every N turns, 1 LLM call)
LLM-powered curation with a small/fast model (Haiku). Processors: Relevance, Compressor, Reorderer, Orientation.
- Score each block for relevance to current task
- Compress warm blocks (target 3-5x, <1% degradation)
- Evict low-relevance blocks to external memory
- Reorder: high-relevance to start + end positions (U-curve optimization)
- Inject orientation summary at context end

### Tier 2 — Deep Sleep (threshold-triggered, 2-3 LLM calls)
Full context reconstruction. Processors: IntentAuditor, Consolidator, plus all Tier 1 processors.
Triggers: capacity >70%, N turns since last deep sleep, or task phase transition.
- Re-evaluate ALL blocks against current task state
- Merge redundant findings into consolidated blocks
- Extract patterns from episodic observations (Generative Agents-style reflection)
- Intent-action-outcome audit: compare stated plan against actual actions taken
- Rebuild workspace from scratch with optimal structure
- Produce a "you are here" orientation block for the end of context

## Intent-Action-Outcome Tracking

Each curation pass (Tier 1+) extracts:
- **Stated intent:** what the agent said it would do ("I'll refactor the auth module")
- **Actions taken:** what tools were called, what files were edited
- **Outcome:** what changed, did tests pass, did the goal advance

When drift is detected (actions don't match intent, or outcome doesn't match goal), the curator injects a correction into the orientation zone:
> "Drift detected: You stated intent to refactor auth to JWT (turn 12). Last 5 turns edited logging module instead. Auth module unchanged. Refocusing recommended."

This leverages the U-curve: the orientation zone at the end of context is the second-highest attention position. The agent reads this right before generating its next response.

**Framing distinction:** Lasso Security's Intent Deputy does intent monitoring as security guardrails ("is this agent doing something it shouldn't?"). Curator does it as cognitive coherence ("is this agent drifting from what it told itself it would do?"). Security vs self-awareness.

## Relationship to Existing Harnesses

| Harness | Curator Role |
|---------|-------------|
| **FS-1** (Agent SDK + session cycling) | Curator extends session useful life via the `anthropic` adapter. When a cycle IS needed, the workspace's structured state produces a higher quality state transfer than a journal dump. Complements session cycling — Curator extends, cycling is the fallback. |
| **Swarm Harness** (CLI -> SDK) | Each worker agent gets its own Curator instance via appropriate adapter. Workers maintain context quality independently. Helm monitors Curator metrics across the swarm. |
| **Serverless Harness** (Durable Lambda) | Curator's Workspace (Pydantic BaseModel) serializes to JSON for durable Lambda checkpoints (or S3 offload for >256KB). External memory persists in DynamoDB/S3. Uses `anthropic` adapter. |

## Technical Foundation: Pydantic AI

**Why Pydantic AI for the curation agent (not the whole library):**

Pydantic AI's `history_processors` feature is the exact interception hook Curator's adapters need — a callable that receives the full message list before every model call and returns a modified list. But the Curator core is framework-agnostic.

Key Pydantic AI features used:
- **`history_processors`** — adapter hook for Pydantic AI-based harnesses
- **Structured output** (`output_type=CurationDecision`) — curation agent returns typed Pydantic models
- **Model-agnostic** — worker on Opus/Sonnet, curation agent on Haiku
- **`RunContext.usage`** — real-time token tracking for Tier 0 budget checks
- **Durable execution** — Temporal/DBOS wrappers for checkpoint-friendly execution

The community is actively pushing toward context management: [issue #4137](https://github.com/pydantic/pydantic-ai/issues/4137) (context compaction API), [issue #4538](https://github.com/pydantic/pydantic-ai/issues/4538) (expose context window size), [issue #4267](https://github.com/pydantic/pydantic-ai/issues/4267) (Anthropic compaction support).

**Adapter example (Pydantic AI):**
```python
from curator.core import CurationPipeline, Workspace
from curator.curation_agent import CurationAgent

workspace = Workspace()
curation_agent = CurationAgent(model="anthropic:claude-haiku-4-5-20251001")
pipeline = CurationPipeline.default(curation_agent=curation_agent)

async def curate_context(messages: list[ModelMessage], info: RunContext) -> list[ModelMessage]:
    workspace.ingest_messages(messages)
    workspace = await pipeline.run(workspace, usage=info.usage)
    return workspace.to_messages()

worker = Agent(
    "anthropic:claude-sonnet-4-6-20250514",
    history_processors=[curate_context],
)
```

**Alternatives considered:**

| Option | Verdict |
|--------|---------|
| Raw Anthropic SDK only | Reimplement agent loop, tools, streaming, provider abstraction — rejected for same reasons as FS-1 brief |
| LiteLLM | Model abstraction only, no agent framework features — could be used under Pydantic AI as a provider backend |
| LangGraph | Heavy graph-based state machine, harder to escape ecosystem, adds complexity without benefit for curation |
| Instructor | Structured output only, not an agent framework — Pydantic AI already handles structured output natively |

## MVP — Simplest Thing That Proves the Concept

1. Framework-agnostic core: Workspace + Block + Zone (Pydantic models)
2. Processor pipeline with Tier 0 processors: SystemPin, Staleness, TokenBudget, ObservationMask
3. Tier 1 processors: Relevance scorer + Compressor (using Haiku via curation agent)
4. Serialization: Workspace -> flat message list with attention-optimized ordering
5. External memory: in-memory dict of evicted blocks
6. One adapter: `pydantic_ai` (history_processor hook)
7. Basic metrics: context utilization %, blocks evicted, turns since last curation
8. No intent tracking yet (add after core curation is validated)

**Test protocol:** Run the same long task (50+ turns) with and without Curator. Measure:
- Task completion quality (human-graded or automated eval)
- Total tokens consumed (including curation overhead)
- Turn at which output quality visibly degrades
- Number of drift events / wasted turns
- Cost comparison (curation overhead vs saved retries/restarts)

## What the Research Validates

- **U-curve is real:** >30pp accuracy swing between boundary and middle positions (Liu et al. TACL 2024)
- **OS analogy is sound:** MemGPT (2023) established it, Letta productionized it, EverMemOS (2026) extended it
- **Sleep cycles work:** 74% vs 68.5% on LoCoMo for periodic consolidation (Letta)
- **Compression is practical:** 2-5x with <1-3% degradation (LLMLingua-2)
- **Observation masking is surprisingly effective:** 50% cost reduction, no quality loss (JetBrains NeurIPS 2025)
- **Agent drift is formally measured:** ASI across 12 dimensions (Rath, Jan 2026)
- **Intent monitoring exists as security:** Lasso Intent Deputy (<50ms, 99.83% accuracy, Feb 2026)
- **Processor pipelines work at scale:** Google ADK's named processor architecture is production-validated
- **Harness > model:** 2026 consensus (OpenAI, Anthropic, Google) — context engineering beats model shopping
- **Deep Agents need four pillars:** Explicit Planning + Hierarchical Delegation + Persistent Memory + Context Engineering (Phil Schmid)
- Full research: `memory-bank/thoughts/shared/research/2026-03-06-deep-validate-curated-context-architecture.md`

## What's Novel (No Existing Precedent)

1. **Synchronous per-turn curation by a dedicated system** — everything existing is async (Letta sleep-time) or self-managed (MemGPT). Nobody curates the context synchronously between turns via a processor pipeline.
2. **Non-linear attention-optimized workspace** — everyone treats context as chronological message streams. Nobody structures it by attention zone and serializes with U-curve-aware ordering.
3. **Intent-action-outcome as cognition** — Lasso does intent checking as security guardrails. Nobody does it as a cognitive self-awareness function to correct drift within the agent's own context.
4. **Three-tier curation within a session** — JetBrains proved the cheap tier works. Nobody combines it with expensive tiers in a unified pipeline model.
5. **Framework-agnostic curation library** — every existing context management system is locked to its own agent framework. Curator is a library any harness can consume via thin adapters.

## Open Questions

1. **Curation model:** Use Haiku for Tier 1 curation (cheap, fast) or the same model as the worker (higher quality but expensive)? Leaning Haiku — curation is a well-defined task that doesn't need the most capable model.
2. **Session cycling interaction:** Curator complements FS-1's session cycling — extends session life, cycling is the fallback when even curated context exceeds budget.
3. **External memory retrieval:** When should evicted content be brought back? Agent-driven (it asks) vs curation-driven (Tier 1 notices it's relevant again)? Likely both — agent can request explicitly, and relevance scorer can re-promote.
4. **Multi-agent memory merge:** When Helm workers converge, how do their Curator workspaces merge? Letta uses git; we could use structured diffs of Workspace models or orchestrator synthesis.
5. **Workspace serialization format:** What's the best format for converting structured workspace -> flat messages? Markdown sections? Separate system messages? XML blocks? Needs experimentation.
6. **Light sleep frequency:** Every 5 turns? 10? Adaptive based on context growth rate? Start with fixed (every 5), graduate to adaptive.
7. **Eval infrastructure:** How do we measure context quality over long runs? Need custom evals — no standard benchmark tests behavior at turn 50+.

## Competitive Landscape

| Project | Closest to | Gap from Curator |
|---------|-----------|-----------------|
| Letta | Context Repos + sleep-time agents | Async only, self-managed, no attention-aware reordering, locked to Letta framework |
| EverMemOS | Engram lifecycle (93% LoCoMo) | Cross-session focus, not within-session curation |
| ACE | Three-agent curation pipeline (Generator/Reflector/Curator) | Curates the playbook/instructions, not live context |
| Factory.ai | Progressive distillation | Context assembly (what goes in), not live management |
| Google ADK | Processor pipeline + tiered storage | Closest architectural model but context compaction is simple LLM summarization, no attention-aware reordering or intent tracking |
| OpenAI Codex | Item/Turn/Thread + harness engineering | Context management is opaque compaction, no structured curation pipeline |
| Lasso Intent Deputy | Intent monitoring (<50ms, 99.83%) | Security framing, not cognitive coherence |

## Key References

### Harness Engineering
- [OpenAI: Harness Engineering](https://openai.com/index/harness-engineering/)
- [OpenAI: Unrolling the Codex Agent Loop](https://openai.com/index/unrolling-the-codex-agent-loop/)
- [OpenAI: Codex App Server Architecture](https://openai.com/index/unlocking-the-codex-harness/)
- [Phil Schmid: Agent Harness 2026](https://www.philschmid.de/agent-harness-2026)
- [Phil Schmid: Agents 2.0 — Deep Agents](https://www.philschmid.de/agents-2.0-deep-agents)
- [Martin Fowler: Harness Engineering](https://martinfowler.com/articles/exploring-gen-ai/harness-engineering.html)

### Context Engineering
- [Google ADK Context Architecture](https://developers.googleblog.com/architecting-efficient-context-aware-multi-agent-framework-for-production/)
- [Anthropic: Effective Context Engineering](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents)
- [Letta: Anatomy of a Context Window](https://www.letta.com/blog/guide-to-context-engineering)
- [JetBrains: The Complexity Trap (NeurIPS 2025)](https://blog.jetbrains.com/research/2025/12/efficient-context-management/)

### Memory & Attention Research
- [Lost in the Middle (Liu et al., TACL 2024)](https://arxiv.org/abs/2307.03172)
- [MemGPT (Packer et al., 2023)](https://arxiv.org/abs/2310.08560)
- [EverMemOS (2026)](https://arxiv.org/abs/2601.02163)
- [Agent Drift (Rath, Jan 2026)](https://arxiv.org/abs/2601.04170)
- [Lasso Security Intent Deputy (Feb 2026)](https://www.lasso.security/platform/intent-security)
- Full validation: `memory-bank/thoughts/shared/research/2026-03-06-deep-validate-curated-context-architecture.md`

### Durable Execution
- [Pydantic AI + DBOS](https://pydantic.dev/articles/pydantic-ai-dbos)
- [Temporal: Durable Execution for AI](https://temporal.io/blog/durable-execution-meets-ai-why-temporal-is-the-perfect-foundation-for-ai)

### Framework
- [Pydantic AI docs](https://ai.pydantic.dev/)
- [Pydantic AI history_processors](https://ai.pydantic.dev/message-history/)
- [Context compaction feature request #4137](https://github.com/pydantic/pydantic-ai/issues/4137)

## Next Step

- Parked 2026-03-09. Plan structure approved (6 phases). Resume with `/create_plan` when ready.
