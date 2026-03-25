---
date: 2026-03-04T12:00:00-05:00
source_research: memory-bank/thoughts/shared/research/2026-03-04-deep-optimal-task-decomposition-ai.md
last_generated: 2026-03-04T20:20:59.881722+00:00
---

# Research Brief: 2026-03-04-deep-optimal-task-decomposition-ai

## TL;DR

Effective task decomposition in AI-assisted development converges on three universal principles across classical project management, modern agent architectures, and DAG-based systems: (1) **just-in-time decomposition** — break work down only when it approaches the execution horizon, never upfront; (2) **the coordinator document pattern** — a lightweight meta-plan that references and sequences sub-plans without duplicating their content, used identically in military OPLANs, PMI program plans, Linear initiatives, and multi-agent orchestrators; (3) **three enforced boundaries** — granularity (split when a task exceeds ~30-60 min of agent time or crosses an architectural boundary), responsibility (humans own the outer/strategic loop, AI owns the inner/execution loop), and scope (every addition to scope is a visible event requiring explicit trade-off). The optimal structure is a two-tier system: a coarse backlog of ideas/initiatives that decompose into detailed plans only when approaching execution, with plans themselves decomposable into sub-plans via a DAG expressed through named references in frontmatter — not a database.

## Key facts / constraints

None captured.

## Key recommendations

None captured.

## Open questions

- **Replanning triggers:** When an in-progress plan's assumptions are invalidated by a sibling plan's execution, what's the optimal detection and recovery mechanism? Academic work flags this as unsolved.
- **Decomposition method selection:** HTN treats this as the key intelligence layer. In an AI-assisted context, should the human or AI select the decomposition method (vertical slice vs. horizontal layer vs. risk-first vs. dependency-first)?
- **Cross-plan state:** When multiple plans share state (e.g., both modify the same component), how should the coordinator document track and communicate shared state without becoming a bottleneck?
- **Empirical granularity calibration:** The 30-60 min window is an average. How does this vary by task type (greenfield vs. refactor vs. bugfix) and by codebase complexity?

## Links

- Source research: `memory-bank/thoughts/shared/research/2026-03-04-deep-optimal-task-decomposition-ai.md`
