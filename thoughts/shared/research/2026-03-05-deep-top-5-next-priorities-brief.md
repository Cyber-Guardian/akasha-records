---
date: 2026-03-05T12:00:00-05:00
source_research: memory-bank/thoughts/shared/research/2026-03-05-deep-top-5-next-priorities.md
last_generated: 2026-03-05T20:52:27.981231+00:00
---

# Research Brief: 2026-03-05-deep-top-5-next-priorities

## TL;DR

Product work has been completely dormant for 2 weeks — every commit since Feb 20 is meta/agentic (triage agent, CI, extensions, research docs). The highest-leverage product item is clearing 36 poison DynamoDB stream records that block both the discover pipeline and VQP observability. The VQP Terraform stream trigger is now unblocked (ENG-2133 shipped) and can be wired. On the agentic side, Helm Phase 1 (`promote.py`) is tightly scoped and gates the entire automation pipeline. The DDD test investment plan is dual-purpose — it addresses the "5 components with zero tests" debt while being a hard prerequisite for the DDD restructuring. A quick Linear housekeeping pass would close 4 resolved-but-never-closed CI issues and improve backlog signal.

## Key facts / constraints

None captured.

## Key recommendations

None captured.

## Open questions

- Are the 36 poison records a schema issue, a one-time data corruption, or a systemic bug? Investigation needed before fix approach is clear.
- Should Dashboard Facelift and Clio Marketing get shaped/planned, or are they blocked on product decisions outside engineering?
- Is ENG-1827 (async DynamoDB writer) worth picking up now, or is it premature optimization before VQP is live?
- Should the agent testing pyramid be prioritized before more helm development, given the triage agent's production failures?

## Links

- Source research: `memory-bank/thoughts/shared/research/2026-03-05-deep-top-5-next-priorities.md`
