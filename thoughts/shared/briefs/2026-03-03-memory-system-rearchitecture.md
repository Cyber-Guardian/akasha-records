# Idea Brief: Memory System Rearchitecture — Tiered Injection

**Date:** 2026-03-03
**Status:** Shaped → Planning

## Problem
The durable memory tier conflates project memory with session/agent state. `current_work.md` (272 lines, 4.5x over budget) is a session log masquerading as project memory — written by every route, loading stale context for every session. `next_up.md` is a task queue that depends on who's asking. The result: every session pays a context tax loading other sessions' state, and the "distill to current_work.md" pattern causes unbounded growth with no clear owner.

## Constraints
- ~10 files reference durable paths — any restructure requires path migration
- `thoughts/shared/` paths (189 files) are stable, don't change
- Autonomous agents (Cyrus) need compaction-safe session state — files on disk survive, conversation doesn't
- FS-1 harness already defines a per-session journal pattern — should align, not diverge
- MCP server plan (separate) depends on knowing target durable file structure
- Claude Code has no native per-session scratch directory — must be created via hooks
- CLAUDE.md adherence degrades past 200 lines (~92% at <200, ~71% at 400+)
- Auto-memory (MEMORY.md) becomes unreliable over time (~40% obsolete after 6 months, can conflict with CLAUDE.md)

## Options Considered

### Session Scratch + Lean Durable (minimal change)
Radically simplify durable to just identity + blockers. Per-session scratch file for everything else. Git log + MCP for cross-session continuity.
- Gains: Clean separation, minimal infrastructure
- Costs: Cold start for new sessions — no "here's what's been happening"
- Complexity: Low

### Tiered Injection (project identity + session journal + on-demand archive)
Three distinct tiers with different injection points: project identity in system prompt layer, per-session journal with todo-rewrite pattern, archive via MCP search.
- Gains: Each tier has clear lifecycle. Todo-rewrite counters lost-in-the-middle. Compaction-safe. Aligns with FS-1 journal. Eliminates shared mutable log antipattern.
- Costs: More moving parts (SessionStart hook, PreCompact hook, journal management). Need graduation policy.
- Complexity: Medium

### Git-Native Memory (minimal custom infrastructure)
Git as the memory system. Branch-scoped sessions, commit messages as journal, git log as activity feed.
- Gains: Maximally simple, works in CI naturally
- Costs: Noisy for "thinking out loud" state, no compaction survival for structured todos, poor fit for autonomous agents
- Complexity: Low

## Chosen Approach
**Tiered Injection** — Three tiers with distinct lifecycles: (1) project identity loaded via system prompt, survives compaction natively; (2) per-session journal with file-based state that survives compaction, including todo-rewrite as lost-in-the-middle countermeasure; (3) on-demand archive via MCP search, not loaded at startup.

Eliminates the "distill to current_work.md" antipattern. Sessions journal locally and graduate findings to archive explicitly. Clean upgrade path from Claude Code sessions to FS-1.

## Key Context Discovered During Shaping
- Manus uses todo.md as a lost-in-the-middle countermeasure — rewriting objectives to end of context for recency attention bias (backed by academic literature: 2307.03172)
- Cognition/Devin found explicit memory management outperforms model self-managed approach — models lack awareness of what info will be needed later
- rjmurillo project: shared mutable HANDOFF.md reached 122KB with 80% merge conflicts, replaced with immutable per-session artifacts + queryable index
- Anthropic's own harness guidance: `claude-progress.txt` (append-only) + git history + feature JSON, NOT a shared mutable log
- OpenHands condenser: 54% SWE-bench solve rate with condensation vs 53% baseline — near-zero accuracy loss at half the cost
- Auto-memory conflicts with CLAUDE.md (loads after, can override), is machine-local, and accumulates ~40% obsolete info without curation
- CLAUDE.md @imports load at session start (not lazily) — splitting into imports helps maintenance, not token cost
- `session_id` available in every hook payload; `CLAUDE_ENV_FILE` pattern propagates it as env var

## Related Work
- FS-1 harness plan: `memory-bank/thoughts/shared/plans/2026-03-03-fs1-agent-harness.md`
- Memory MCP server plan: `memory-bank/thoughts/shared/plans/2026-03-03-memory-mcp-server.md`
- Prior combined plan: `memory-bank/thoughts/shared/plans/2026-03-03-centralized-memory-mcp.md`
- Memory architecture rethink handoff: `memory-bank/thoughts/shared/handoffs/general/2026-03-02_16-31-03_memory-system-architecture-rethink.md`
- Planning session handoff: `memory-bank/thoughts/shared/handoffs/general/2026-03-03_13-14-44_centralized-memory-mcp-planning.md`
- Claude Code session mechanisms research: this shaping session (2026-03-03)

## Next Step
- [Plan] → `/create_plan memory-bank/thoughts/shared/briefs/2026-03-03-memory-system-rearchitecture.md`
