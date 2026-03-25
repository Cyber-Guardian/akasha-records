---
date: 2026-03-04T12:00:00-05:00
researcher: Claude
git_commit: bf8dd50f2a584ab7f07d4bd73446e9a7fef16be4
branch: main
repository: filescience
topic: "How to optimally decompose and plan work in AI-assisted development environments"
tags: [deep-research, planning, task-decomposition, agent-architecture, scope-management]
status: complete
research_depth: standard
iterations_completed: 3
last_updated: 2026-03-04
last_updated_by: Claude
---

# Deep Research: Optimal Task Decomposition for AI-Assisted Development

## Research Question
How to optimally decompose and plan work in AI-assisted development environments. The core problem: scope grows too large, task decomposition is ineffective, and it's hard to keep track of large multi-plan initiatives.

## Summary
Effective task decomposition in AI-assisted development converges on three universal principles across classical project management, modern agent architectures, and DAG-based systems: (1) **just-in-time decomposition** — break work down only when it approaches the execution horizon, never upfront; (2) **the coordinator document pattern** — a lightweight meta-plan that references and sequences sub-plans without duplicating their content, used identically in military OPLANs, PMI program plans, Linear initiatives, and multi-agent orchestrators; (3) **three enforced boundaries** — granularity (split when a task exceeds ~30-60 min of agent time or crosses an architectural boundary), responsibility (humans own the outer/strategic loop, AI owns the inner/execution loop), and scope (every addition to scope is a visible event requiring explicit trade-off). The optimal structure is a two-tier system: a coarse backlog of ideas/initiatives that decompose into detailed plans only when approaching execution, with plans themselves decomposable into sub-plans via a DAG expressed through named references in frontmatter — not a database.

## Perspectives Explored
1. **Classical decomposition theory** — Revealed that HTN's recursive decomposition with method selection as the intelligence layer, WBS's MECE 100% rule, and rolling-wave's JIT elaboration are the foundational primitives that all modern systems rediscover.
2. **AI agent planning architectures** — Showed convergence on markdown plan files + hierarchical planner/worker dispatch; conflict prevention via isolation (worktrees) over resolution; and Graph-of-Thoughts as the emerging structure replacing linear/tree plans.
3. **DAG-based work management** — Demonstrated that append-only graph expansion (new nodes as leaves, never mid-graph insertion) is the only pattern that works for evolving scope, portable to markdown via named references and checkbox state.
4. **Human-AI collaborative planning** — Established the 30-60 min optimal task window, the three developer loops framework (inner/middle/outer → AI/collaborative/human), and concrete escalation signals for when AI should defer to humans.
5. **Scope management & progressive elaboration** — Confirmed that the two-tier structure (coarse backlog + detailed horizon) with gated decomposition and visible scope events is the universal anti-scope-creep mechanism.

## Detailed Findings

### 1. Classical Decomposition Theory

**Hierarchical Task Networks (HTN)** provide the theoretical foundation. HTN decomposes compound tasks into primitive (directly executable) tasks through domain-specific "methods" — each method has preconditions, effects, and ordering constraints. Decomposition recurses until only leaf-level executable actions remain. The critical insight: **method selection** (choosing which decomposition to apply) is the key intelligence layer, not the decomposition itself.

**Work Breakdown Structure (WBS)** adds two crucial rules:
- **100% rule (MECE):** Every parent node must equal the sum of its children — no scope leakage or overlap. This is the completeness invariant.
- **8/80 rule:** Work packages should be 8-80 hours. Below 8h = over-decomposed (micromanagement). Above 80h = under-decomposed (unmanageable).

**Rolling-wave planning** adds the temporal dimension: only the next 1-2 execution horizons are fully detailed. Distant work stays at high abstraction until a gate condition is met: "do we have enough information to detail this?" This prevents premature decomposition.

**SAFe's hierarchy** (epic → feature → story → task) operationalizes progressive elaboration: each level decomposes only when its parent is committed to execution, not at inception.

**Transfer principle:** Decompose just-in-time to the execution horizon, enforce MECE at each level, and treat method selection (which decomposition strategy to apply) as a first-class gated decision.

### 2. AI Agent Planning Architectures

**Current convergence points:**
- **Markdown/JSON plan files on disk** — Claude Code uses markdown in folders, OpenHands proposes `.openhands/plan.json` with step-by-step mutation. The plan file is the source of truth, not agent memory.
- **Hierarchical multi-agent decomposition** — A planner/manager role dispatches subtasks to specialized executors (Devin's multi-agent mode, HyperAgent's Planner/Navigator/Editor/Executor quad, CrewAI's manager_llm hierarchy).
- **DAG-native orchestration** — LangGraph models control flow as a directed graph with conditional edges.
- **Iterative replanning** — Failures trigger re-decomposition rather than hard stops (GitHub Copilot Agent, ReWOO pattern).

**Graph-of-Thoughts (GoT)** is the emerging structure, replacing linear Chain-of-Thought and Tree-of-Thought. It models reasoning as an arbitrary DAG with three core transforms:
- **Generation:** One node spawns many (decomposition)
- **Aggregation:** Many nodes merge into one (synthesis)
- **Refinement:** Self-loop to improve a single node (iteration)

GoT cuts median error ~33% vs Tree-of-Thought by allowing aggregation across branches — something tree structures can't do.

**Multi-agent coordination** is solved by **prevention, not resolution:**
- Git worktrees + role hierarchy (Planner/Worker/Judge) isolate agents
- Cursor explicitly abandoned optimistic concurrency due to throughput degradation
- Where resolution exists: shared blackboard state (LangGraph), Execute→Replan→Execute cycles
- Academic work identifies plan disagreement, scope overlap, and cascading work invalidation as **open unsolved problems** (arXiv 2404.04834, 2503.13657)

### 3. DAG-Based Work Management

**Build systems** (Bazel, Nx, Turborepo) compute static task graphs with content-hash fingerprinting per node — the cache key is the node, invalidation propagates upstream. No database needed, just hash files.

**Workflow engines** differ on dynamism:
- Airflow: DAG fixed at parse time, branching via skip-marking downstream nodes
- Prefect/Temporal: Allow **runtime graph expansion** — but append-only. New nodes are always leaves or extend existing leaves. Never insert mid-graph.
- Dagster: Static structure per run, but conditional outputs (node yields nothing → downstream auto-skipped)

**Markdown-portable DAG patterns:**
1. **Node status as inline checkbox** — `- [ ]` / `- [x]`
2. **Dependency as named references** — `depends: [plan-a, plan-b]` in frontmatter
3. **Conditional edges as "blocked-if" annotations** — `blocked_by: plan-a` only when explicitly needed
4. **Scope additions as new leaf nodes with parent reference** — never rewriting existing edges

**File-based DAG implementations:**
- Taskwarrior: `depends:` field with comma-separated UUIDs in plain text `.data` files, virtual `+BLOCKED`/`+BLOCKING` tags computed at runtime
- `markdown-plan` Python library: Nested markdown lists parsed as tree-or-DAG, with `@(substring)` syntax for cross-branch dependencies
- plan.md (Digital-Tvilling): Too primitive for real use — flat task lists with status, no dependency graph

### 4. Human-AI Collaborative Planning

**Optimal AI task characteristics:**
- **Duration:** ~30-60 minutes of autonomous work. Doubling duration quadruples failure rate (non-linear degradation).
- **Scope:** Single architectural boundary (one component, one service, one directory)
- **Exit condition:** Single, verifiable acceptance check
- **Shape:** Vertical slice preferred over horizontal layer — self-contained with clear "done" signal

**The three developer loops** (Gene Kim/Steve Yegge, IT Revolution):
| Loop | Timeframe | Owner | Planning level |
|------|-----------|-------|----------------|
| Inner | Seconds–minutes | AI | Within-phase execution, file-level decisions |
| Middle | Hours | Human-supervised | Phase execution, integration, verification |
| Outer | Days–weeks | Human | Strategy, architecture, acceptance criteria |

**Scope creep is a loop boundary violation** — when inner-loop work starts requiring outer-loop decisions, something has gone wrong with the decomposition.

**Escalation signals** (when AI should defer to human):
1. Scope has expanded beyond the original task definition
2. Integration failures that can't be caught in isolation
3. Repeated mistake patterns (same error 2+ times)
4. Context window approaching saturation

**Planner-Worker architecture** (frontier model plans, cheaper models execute) cuts costs ~90% while keeping each worker within its effective context window.

**SWE-Bench Pro data:** Top models achieve only 23% on long-horizon tasks vs. 70%+ on standard single-issue benchmarks — confirming that multi-file, multi-day tasks systematically exceed current autonomous capacity.

### 5. Scope Management & Progressive Elaboration

**The two-tier structure** is the universal anti-scope-creep pattern:
- **Tier 1 (coarse):** Backlog of ideas, initiatives, high-level work items. Intentionally vague. Not decomposed.
- **Tier 2 (detailed):** Execution-ready plans with phases, file lists, verification criteria. Fully decomposed.

**Promotion gate:** Work moves from Tier 1 → Tier 2 only when approaching the execution horizon. The gate question: "Do we have enough information to detail this?"

**Scope control mechanisms:**
- **8/80 rule as split signal:** Tasks between 8-80 hours stop decomposing. Below = over-decomposed. Above = split further.
- **INVEST "Small" criterion:** ~3 days as the JIT entry gate. Stories only decompose when entering the sprint/execution horizon.
- **PI Planning as scope budget:** Fixed cadence forces trade-off negotiation rather than additive "also add this." Epic Burndown makes scope additions visible as spikes.

**Transfer principle:** Gate decomposition by proximity to execution. Make every scope addition visible as an event requiring an explicit trade-off — not silent growth.

### Cross-cutting Patterns

**The Coordinator Document** is the universal pattern for multi-plan work:
| Domain | Name | Mechanism |
|--------|------|-----------|
| Military | OPLAN/OPORD | Parent plan nests sub-plans as annexes |
| PMI | Program Management Plan | References project plans as sub-schedules with common data date |
| Linear | Initiatives | Sub-initiatives up to 5 levels deep, each referencing child projects |
| Obsidian/Logseq | Hub Note | Bi-directional wikilinks to phase/sub-plan notes |
| AI agents | Lead agent context | Holds coordination state, dispatches sub-agents per phase |

The coordinator document **references sub-plans, it doesn't contain them.** This is the key structural insight — it's a DAG of pointers, not a monolithic document.

**Just-in-time decomposition is the universal principle** — HTN decomposes when preconditions are met, rolling-wave at the execution horizon, DAG systems append leaves at runtime, AI agents replan on failure. The anti-pattern is upfront total decomposition.

**Three enforced boundaries:**
1. **Granularity:** Split when >30-60 min for AI / crosses architectural boundary / failure shouldn't cascade / sub-tasks can run independently
2. **Responsibility:** Human owns outer loop (strategy, acceptance), AI owns inner loop (execution, file-level), middle loop is collaborative
3. **Scope:** Every addition is a visible event requiring explicit trade-off — not silent backlog growth

## Key Sources

### Academic Papers
- [HTN Planning in AI](https://en.wikipedia.org/wiki/Hierarchical_task_network)
- [Graph of Thoughts (arXiv 2308.09687)](https://arxiv.org/abs/2308.09687)
- [LLM-Based Multi-Agent Systems for SE (arXiv 2404.04834)](https://arxiv.org/html/2404.04834v4)
- [Why Do Multi-Agent LLM Systems Fail? (arXiv 2503.13657)](https://arxiv.org/pdf/2503.13657)
- [SWE-Bench Pro: Long-Horizon Tasks (arXiv 2509.16941)](https://arxiv.org/abs/2509.16941)
- [Human-AI Collaborative Decision-Making (arXiv 2505.18066)](https://arxiv.org/abs/2505.18066)
- [Dynamic Task Decomposition for AI Agents (arXiv 2410.22457)](https://arxiv.org/abs/2410.22457)

### Industry / Practitioner
- [The Three Developer Loops — IT Revolution](https://itrevolution.com/articles/the-three-developer-loops-a-new-framework-for-ai-assisted-coding/)
- [Long-Running AI Agents and Task Decomposition — Zylos Research](https://zylos.ai/research/2026-01-16-long-running-ai-agents)
- [AI Coding Agents in 2026: Coherence Through Orchestration — Mike Mason](https://mikemason.ca/writing/ai-coding-agents-jan-2026/)
- [Task Decomposition for Coding Agents — atoms.dev](https://atoms.dev/insights/task-decomposition-for-coding-agents-architectures-advancements-and-future-directions/a95f933f2c6541fc9e1fb352b429da15)
- [Breaking Down Tasks for AI Agents — Brenndoerfer](https://mbrenndoerfer.com/writing/breaking-down-tasks-task-decomposition-ai-agents)
- [Parallel AI Agents for Massive Refactors — Tessl](https://tessl.io/blog/use-automated-parallel-ai-agents-for-massive-refactors/)

### Tools & Frameworks
- [OpenHands Planning Issue (#9970)](https://github.com/OpenHands/OpenHands/issues/9970)
- [Devin 2.0 — Cognition](https://cognition.ai/blog/devin-2)
- [LangGraph Multi-Agent Workflows](https://blog.langchain.com/langgraph-multi-agent-workflows/)
- [Linear Sub-Initiatives](https://linear.app/docs/sub-initiatives)
- [Graph-of-Thoughts GitHub](https://github.com/spcl/graph-of-thoughts)
- [Taskwarrior Dependency Management](https://deepwiki.com/GothenburgBitFactory/taskwarrior/3.5-dependency-management)

### Project Management
- [WBS 100% Rule](https://www.oodles.com/insights/what-is-the-100-rule-of-work-breakdown-structure-wbs)
- [Rolling Wave Planning — PMI](https://www.pmi.org/learning/library/rolling-wave-approach-project-management-10514)
- [PMI Program Management Practices](https://www.pmi.org/disciplined-agile/process/program-management/program-management-practices)

## Open Questions
- **Replanning triggers:** When an in-progress plan's assumptions are invalidated by a sibling plan's execution, what's the optimal detection and recovery mechanism? Academic work flags this as unsolved.
- **Decomposition method selection:** HTN treats this as the key intelligence layer. In an AI-assisted context, should the human or AI select the decomposition method (vertical slice vs. horizontal layer vs. risk-first vs. dependency-first)?
- **Cross-plan state:** When multiple plans share state (e.g., both modify the same component), how should the coordinator document track and communicate shared state without becoming a bottleneck?
- **Empirical granularity calibration:** The 30-60 min window is an average. How does this vary by task type (greenfield vs. refactor vs. bugfix) and by codebase complexity?

## Research Metadata
- Depth mode: standard
- Iterations completed: 3 / 5
- Termination reason: gaps exhausted (all 10 gaps closed)
- Manifest: `.claude/deep-research/2026-03-04-optimal-task-decomposition-ai.md`
