# Idea Brief: Plan Trees / Initiative Envelopes

**Date:** 2026-03-05
**Status:** Shaped → Parked (brainstorm phase — no approach committed yet)

## Problem
The current plan system is single-tier — every idea gets the same treatment (brief → plan → phases → implement). There's no coordinator document for multi-plan initiatives, no formal decomposition gate, and no DAG between plans. When scope grows, plans either bloat or fragment into disconnected files with only prose references between them. The core pain is ineffective task decomposition: scope grows too large and there's no way to keep track of large multi-plan work.

## Constraints
- Markdown flat files in `memory-bank/thoughts/shared/plans/` — no database
- Existing tooling (Helm decompose, implement_plan, generate_scope) parses plan structure — can't break those
- Linear is a projection, not the source of truth
- Most work is still one brief → one plan → done — overhead for simple work must be near zero
- Plans already have phases — the relationship between "phase" and "sub-plan" needs to be clear, not blurry

## Options Considered (brainstorm — none committed)

### Initiative Envelope
New artifact type: a thin coordinator document (DAG of references to sub-plans). Plans stay unchanged. Phases that grow too big get promoted to sub-plans. Initiative materializes lazily — only when the first promotion happens.
- Gains: Zero overhead for simple work. One place to see the big picture. Existing tooling untouched.
- Costs: New artifact type. Skills need to learn about it.
- Complexity: Medium
- Pattern fit: Universal coordinator pattern (military OPLAN, PMI, Linear initiatives)

### Frontmatter DAG
No new artifact. Add optional `parent:` and `depends:` to plan frontmatter. Graph emerges from links. A `/plan-status` skill walks the DAG on demand.
- Gains: Simplest change — just frontmatter fields. No new artifact type.
- Costs: No single zoom-out view without querying. Root plans accumulate `→ see:` pointers.
- Complexity: Low
- Pattern fit: Taskwarrior `depends:`, Obsidian emergent graph. Lacks explicit coordinator document.

### Two-Tier with Scope Budget
Formalize Tier 1 (lightweight initiative backlog, no phases) and Tier 2 (today's plans). Gate promotion. Declare estimated plan count upfront; exceeding it is a visible scope event requiring trade-off.
- Gains: Directly addresses silent scope growth. Forces rough sizing without premature decomposition.
- Costs: More ceremony. Could feel heavy for organic work.
- Complexity: High
- Pattern fit: SAFe PI Planning scope budgets. Most structured/opinionated.

## Preliminary Recommendation
Initiative Envelope looked strongest during brainstorm — zero overhead for simple work, lazy materialization, mirrors universal coordinator pattern. But no approach is committed yet. Needs more thought on: schema design, skill integration points, how Helm would interact with initiatives.

## Key Context Discovered During Shaping
- Deep research conducted: `memory-bank/thoughts/shared/research/2026-03-04-deep-optimal-task-decomposition-ai.md`
- Just-in-time decomposition is the universal principle across HTN, rolling-wave, DAG systems, and AI agents
- Optimal AI task: ~30-60 min, single architectural boundary, vertical slice preferred
- Three enforced boundaries: granularity, responsibility (human outer / AI inner loop), scope (every addition visible)
- Coordinator document pattern is universal: military OPLAN, PMI program plans, Linear initiatives, Obsidian hub notes
- Multi-agent conflict solved by prevention (isolation) not resolution — open unsolved problem academically
- Existing ad-hoc patterns in codebase: `macro-agent-strategy.md` navigator, informal `Parent plan:` references, Helm manifests

## Next Step
- [Parked] → Linear backlog issue: ENG-2329
