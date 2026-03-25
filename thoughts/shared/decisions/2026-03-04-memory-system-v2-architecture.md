---
date: 2026-03-04T16:55:00-05:00
author: jordan + claude
topic: "Memory System v2 — Four-Tier Architecture"
status: complete
last_updated: 2026-03-04
last_updated_by: claude
---

# Decision: Memory System v2 — Four-Tier Architecture

## Context
- The three-tier memory model (durable + session + archive) had queue.md and blockers.md as manually-maintained files that went stale in multi-agent workflows
- No intermediate layer between always-loaded durable files and grep-only archive
- Discovery of relevant archive files relied on filename matching, which breaks on vocabulary mismatch

## Decision
- Rearchitect to four-tier model: hot (identity + CI-generated state) → warm (topic notes) → cold (dated archive) → procedural (.claude/)
- Delete queue.md and blockers.md — Linear becomes single source of truth for backlog and blockers
- Add memory-bank/topics/ as warm tier of living topic notes (kebab-case, YAML frontmatter, 30-80 lines each)
- Add CI-generated state.md for Polylith inventory, CI health, and deployed artifacts
- Install QMD as local MCP server for semantic search across the entire memory bank

## Options considered
- A: Keep queue/blockers as files, add topics/ alongside → still have stale file problem
- B: Delete queue/blockers, Linear-as-truth with local cache fallback + warm tier → chosen
- C: Full Memory MCP server with S3 sync → too complex for immediate needs, deferred

## Tradeoffs
- Gain: Linear is always up-to-date, no stale files, semantic discovery via QMD
- Give up: Instant file reads for backlog (now ~1-3s Linear query), dependency on Linear MCP availability
- Mitigation: Gitignored linear-cache.md as read-only fallback when Linear is unavailable

## Consequences
- All skills updated to query Linear instead of reading queue/blockers files
- CLAUDE.md routing table now points to topic note discovery step instead of queue/blockers updates
- Session closeout now includes topic note update scan
- CI runs nightly to regenerate state.md
- Future: nightly consolidation agent (ENG-2266) and Memory MCP server (ENG-2265) build on this substrate

## References
- Plan: `memory-bank/thoughts/shared/plans/2026-03-04-memory-system-v2-rearchitecture.md`
- Shaping handoff: `memory-bank/thoughts/shared/handoffs/general/2026-03-02_16-31-03_memory-system-architecture-rethink.md`
- Predecessor plan: `memory-bank/thoughts/shared/plans/2026-03-03-memory-system-rearchitecture.md`
