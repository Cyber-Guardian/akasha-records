---
date: 2026-03-01T23:13:37.637943+00:00
source_research: memory-bank/thoughts/shared/research/2026-03-01-multi-agent-architecture-research.md
last_generated: 2026-03-01T23:13:37.637943+00:00
---

# Research Brief: 2026-03-01-multi-agent-architecture-research

## TL;DR

**Date:** 2026-03-01
**Purpose:** Survey of production and academic multi-agent coding systems to inform the FileScience agent helm orchestration design.
## TL;DR
Conflict prevention beats conflict resolution. Design/decomposition quality is the #1 bottleneck at scale. Two agents coordinating achieve half the success of one (CooperBench). The only successful tournament systems use automated evaluators (test execution, fitness functions), not generic LLM judges. Stacked diffs + serial merge queue is more battle-tested than parallel merge. Advisory file locks work in practice with capable models.
---
## Production Systems
### Gas Town (Steve Yegge) — The Reference Implementation
**Scale:** 20-30 concurrent Claude Code instances. ~$100/hr at full scale.
**Source:** [GitHub](https://github.com/steveyegge/gastown) | [HN discussion](https://news.ycombinator.com/item?id=46734302)
**Architecture:** Kubernetes-inspired hierarchical fleet orchestration.

## Key facts / constraints

None captured.

## Key recommendations

None captured.

## Open questions

None

## Links

- Source research: `memory-bank/thoughts/shared/research/2026-03-01-multi-agent-architecture-research.md`
