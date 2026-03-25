---
type: event
created: 2026-03-19
status: active
---
# Idea Brief: Autoresearch Wave 3 — Thinker Architecture

**Date:** 2026-03-19
**Status:** Shaped -> Parked (pending Wave 1+2 overnight validation)

## Problem
The autoresearch leader is a single PydanticAI agent trying to be three things: router, architect, and reviewer. It runs on sonnet with structured output — good for classification, bad for deep reasoning. The result: shallow hypotheses, no research planning, no context curation, no model selection. The researcher gets a flat prompt dump and implements blind. In the dedup-compression campaign, this led to $1.05 in wasted experiments that deeper planning would have prevented.

## Constraints
- Thinker must run via Claude SDK (opus) for full reasoning capability — PydanticAI structured output is too shallow
- Must work unattended — no human in the loop during campaigns
- Thinker cost (~$0.10/experiment) is justified: cheaper than one wasted researcher call (~$0.45)
- The thinker IS the lab leader — the current "leader" is really just a router
- Plans should be multi-experiment (research direction with branching), not single-hypothesis
- Must draw from existing planning system patterns and research findings below

## Research Findings (from deep research)

Full doc: [[2026-03-19-deep-multi-experiment-research-planning|Multi-Experiment Research Planning]]

### Three architectural archetypes in the wild

| Pattern | Example | Planning | Diversity | Review | Our analogue |
|---------|---------|----------|-----------|--------|-------------|
| Tree search + branching | AI Scientist v2 (BFTS), AIDE | Explicit (node expansion) | Parent selection | Post-execution LLM scoring | Our solution tree |
| Evolutionary islands | FunSearch (~1000 islands, periodic culling) | Implicit (selection pressure) | Island isolation + culling | Continuous scoring | Could inform personality diversity |
| Stage-gate sequential | Pharma R&D, clinical trials | Explicit (if-then plans) | Gate outcomes (go/kill/hold/recycle) | Formal gate reviews | Our budget/stagnation gates |

### Key patterns to adopt

**1. AI Scientist v2 is built on AIDE's tree search** — same node structure (code, plan, metric, debug_depth), but adds parallel workers + LLM-scored selection (not just scalar metrics). Our harness already has the solution tree; the thinker adds the LLM-scored planning layer on top.

**2. PIs use moderate specification** — Research shows goal + key variables + relevant context outperforms both vague direction and over-specified step-lists for creative problem-solving (SAGE Journals). The thinker should brief researchers at this granularity — not step-by-step protocols.

**3. Adversarial review tiered by risk** — Pre-registration raised replication to 86% but adds overhead (Royal Society Open Science). AI Scientist v2's reviewer is post-execution only. Our thinker should pre-review architectural experiments but skip review for knob turns.

**4. Context curation consensus** — Sub-agents should return 1-2K token distilled summaries (Anthropic guidance). Place critical info at context boundaries per U-curve (>30pp swing, MIT/TACL 2024). Use selective retrieval not dumps (AgentRxiv pattern from Nature Biotechnology 2026).

**5. FunSearch island culling** — Bottom 50% of islands reset every ~4h. Maps to personality diversity: if pragmatic keeps failing, the thinker should stop assigning it and reallocate.

**6. NIH if-then branching** — Real PIs write branching contingency plans. The thinker's multi-experiment plan should have explicit branch conditions: "if experiment 1 scores >X, try Y; if it crashes, try Z instead."

**7. AIDE's 3-phase policy** — Draft diverse seeds (5) → debug broken nodes (prob 0.5, depth 3) → greedily improve best. Simple, effective. Our thinker could adopt a similar phase structure within each research plan.

### What no one does yet
- **Pre-execution adversarial review** — AI Scientist v2 reviews post-execution. Real labs review plans informally in lab meetings. No system has a formal pre-execution adversarial reviewer. This is our opportunity.
- **Explicit context curation instructions** — No autonomous system has the thinker specify WHAT context the researcher should see. They all dump everything or use RAG. Thinker-directed curation is novel.
- **Multi-experiment plans with LLM-as-planner** — AI Scientist v2 plans one node at a time (BFTS expansion). FunSearch has no planner at all. A thinker that produces a multi-experiment plan with branching is architecturally novel.

## Options Considered

### Thinker-Default Layered Dispatch (chosen)
Thinker (opus, Claude SDK) runs by default as the research director. Produces multi-experiment research plans with branching logic (NIH if-then model). Selects models/personalities/context per experiment. Reviews results between experiments and adapts (stage-gate model). Router is mechanical code — executes the thinker's decisions. Reviewer is optional, invoked by the thinker for risky experiments (tiered review model).
- Gains: Deep planning prevents wasted experiments. Multi-experiment coherence. Model selection. Context curation. Adversarial pre-review for risky experiments.
- Costs: Higher per-experiment overhead (~$0.10 thinker). More complex coordinator. Need to define plan schema with branching.
- Complexity: High

### Upgraded Single Leader
Keep one PydanticAI agent but enrich LabLeaderDecision with model_override, context_selections, and structured plan fields.
- Gains: Simple. Incremental.
- Costs: PydanticAI/sonnet can't do deep reasoning. No multi-experiment planning. No adversarial review.
- Complexity: Medium

### Full Pipeline Every Time
Every experiment goes through Thinker → Reviewer → Researcher with no skip path.
- Gains: Consistent quality.
- Costs: Overkill for trivial changes. Higher cost floor.
- Complexity: High

## Chosen Approach
**Thinker-Default Layered Dispatch** — informed by all three archetypes: tree search (AI Scientist v2 node structure), stage-gate (pharma go/kill/hold/recycle), and evolutionary diversity (FunSearch island culling for personality allocation).

## Key Architecture Decisions

### Roles
- **Thinker = lab leader.** Opus via Claude SDK. Plans multi-experiment campaigns. Reviews between experiments. Decides everything: hypothesis, model, personality, context, whether to invoke reviewer. Runs by default (not optional).
- **Router = mechanical dispatch.** No LLM. Coordinator code that executes the thinker's plan: builds prompts, curates context per thinker's instructions, selects model, dispatches researcher.
- **Researcher = executor.** Sonnet or opus per thinker's decision. Implements one experiment from the plan. Returns distilled result (1-2K tokens, Anthropic pattern).
- **Reviewer = adversarial check.** Optional. Thinker invokes for architectural/risky experiments only (tiered review). Challenges assumptions before researcher burns resources.

### Multi-experiment plans
The thinker produces a research plan with:
- **Sequence of experiments** with explicit branching (NIH if-then model)
- **Per-experiment specification:** hypothesis, model, personality, context selections, success criteria
- **Gate conditions** between experiments: go (continue), kill (abandon this direction), pivot (try alternative branch)
- **Moderate specification** per experiment: goal + key variables + relevant context, NOT step-by-step (SAGE research on optimal briefing granularity)

### Context curation
The thinker specifies per experiment:
- Which prior experiment results to include (by ID, not all)
- Which code sections are relevant
- Which eval_metadata fields matter for this hypothesis
- Placement guidance (critical info at context boundaries per U-curve)

### Personality diversity
Adopt FunSearch's culling insight: if a personality's keep rate drops below threshold, the thinker should deprioritize it and reallocate experiments to higher-performing personalities.

## Prior Art

### In this repo
- [[2026-03-12-agent-harness-composition|Agent Harness Composition]] — cognitive profiles, consensus protocols, rail sets
- [[2026-03-06-deep-validate-curated-context-architecture|Curated Context Architecture]] — context as managed memory, U-curve
- [[2026-03-04-deep-optimal-task-decomposition-ai|Task Decomposition Research]] — just-in-time decomposition
- [[2026-02-28-agent-swarm-harness|Agent Swarm Harness]] — CLI swarm with model selection
- [[2026-03-19-deep-multi-experiment-research-planning|Multi-Experiment Research Planning]] — deep research backing this brief

### External
- [AI Scientist v2 — BFTS architecture](https://arxiv.org/abs/2504.08066) + [GitHub](https://github.com/SakanaAI/AI-Scientist-v2)
- [FunSearch — evolutionary islands](https://www.nature.com/articles/s41586-023-06924-6) + [GitHub](https://github.com/google-deepmind/funsearch)
- [AIDE — 3-phase tree search](https://arxiv.org/html/2502.13138v1) + [GitHub](https://github.com/WecoAI/aideml)
- [Anthropic — Context engineering for agents](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents)
- [Manus — Context engineering lessons](https://manus.im/blog/Context-Engineering-for-AI-Agents-Lessons-from-Building-Manus)
- [NIH — if-then branching in research plans](https://www.niaid.nih.gov/grants-contracts/write-research-plan)
- [Pre-registration raises replication to 86%](https://www.science.org/content/article/preregistering-transparency-and-large-samples-boost-psychology-studies-replication-rate)

## Next Step
- [Parked] -> Linear backlog issue, pending overnight Wave 1+2 validation
- When ready: `/create_plan memory-bank/thoughts/shared/briefs/2026-03-19-autoresearch-wave3-thinker-architecture.md`
