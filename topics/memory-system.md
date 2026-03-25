---
topic: Memory System
status: active
touched: 2026-03-24
related:
  - thoughts/shared/plans/2026-03-04-memory-system-v2-rearchitecture.md
  - thoughts/shared/briefs/2026-03-04-memory-discovery-and-naming.md
  - thoughts/shared/plans/2026-03-03-memory-mcp-server.md
  - thoughts/shared/research/2026-03-04-deep-obsidian-vaults-for-agentic-brief.md
  - thoughts/shared/briefs/2026-03-07-self-grounding-knowledge-system.md
---

# Memory System

## Current State
Four-tier architecture is live and operational:
- **Hot** (`memory-bank/durable/`): `identity.md` (49 lines) + CI-generated `state.md` (24 lines)
- **Warm** (`memory-bank/topics/`): 13 living topic notes — all with complete frontmatter, none stale
- **Cold** (`memory-bank/thoughts/`): 348+ dated archive files across briefs, plans, research, decisions, handoffs, reference, series
- **Procedural** (`.claude/`): Skills, agents, rules, hooks

QMD (local BM25 + vector + reranking) indexes 378 docs. Auto-reindexes on SessionStart via hook. Obsidian vault configured for human browsing.

Frontmatter compliance enforced by PostToolUse hook on `thoughts/shared/` writes — blocks on missing/invalid `type`, `created`, `status` fields.

## Key Decisions
- 2026-03-04: Delete queue.md/blockers.md — Linear is single source of truth for backlog/blockers
- 2026-03-04: CI-generated state.md replaces hand-maintained state info
- 2026-03-04: Flat topics/ directory with kebab-case slugs — no subdirectories
- 2026-03-04: YAML frontmatter on new cold-tier files only — no backfill
- 2026-03-04: QMD for semantic discovery — tags rejected (tag rot is dominant failure mode)
- 2026-03-12: QMD auto-reindex on SessionStart (startup/clear only, skips compact/resume)
- 2026-03-12: PostToolUse frontmatter validator — blocks writes to thoughts/shared/ without valid lifecycle frontmatter

## Open Questions
- Nightly consolidation agent (ENG-2183) not yet implemented — archive is growing (~348 files) and needs rollup
- Self-grounding knowledge system shaped (2026-03-07) — parked until deep research harness ships. See [[2026-03-07-self-grounding-knowledge-system|Self-Grounding Knowledge System Brief]]
- state.md CI health shows "not found" for lint/test/release — CI reporting may need investigation

## Artifacts
- `memory-bank/` — the memory system itself
- `.claude/reference/memory-rules.md` — tier definitions and rules
- `.claude/hooks/session_start.py` — QMD reindex + session cleanup
- `.claude/hooks/validators/frontmatter.py` — lifecycle frontmatter enforcement
