---
date: 2026-03-10T12:00:00-05:00
source_research: memory-bank/thoughts/shared/research/2026-03-10-deep-linear-best-practices.md
last_generated: 2026-03-10T13:46:59.090255+00:00
---

# Research Brief: 2026-03-10-deep-linear-best-practices

## TL;DR

Linear's opinionated PM model converges around 2-week cycles, ~15-20 grouped labels (`Type/Bug`, `Area/API`), daily rotating triage, and a flat Issue → Cycle → Project → Initiative hierarchy. The Agent Interaction SDK (July 2025) formalizes agent delegation with activity streaming, session states, and non-billable OAuth identities — a pattern Cursor, Devin, and others are adopting. Our current skills (`/create-linear-issue`, `/resolve-linear-issue`) are strong on scope discipline (action-verb titles, max-3 AC, plan lifecycle) but miss 10 key capabilities including Agent SDK adoption, label-group enforcement, triage automation, stale-issue archiving, and webhook-driven workflows. The research identifies 8 concrete `/extend` targets ranging from quick wins (label taxonomy rule) to strategic investments (Agent SDK integration).

## Key facts / constraints

None captured.

## Key recommendations

None captured.

## Open questions

- What Linear plan tier (Free/Standard/Business/Enterprise) does FileScience currently use? Triage Rules and some AI features require Business/Enterprise.
- Should we pursue the Agent Interaction SDK now (Developer Preview) or wait for GA? The 10-second responsiveness requirement implies a persistent process.
- Would event-driven dispatch (T3.2) replace or complement the current `helm` orchestration for issue resolution?
- How would label taxonomy migration work without breaking existing saved views and automations?

## Links

- Source research: `memory-bank/thoughts/shared/research/2026-03-10-deep-linear-best-practices.md`
