# Idea Brief: FS-1 — Custom Agent Harness with Progressive Journaling

**Date:** 2026-03-02
**Status:** Shaped → Planning

## Problem
Claude Code's context management is opaque — compaction fires automatically, you can't control when it happens or what gets preserved. For long, complex sessions (especially Helm-orchestrated swarm work), losing nuance from early turns is real damage. The current `01-active/` directory (`current_work.md`, `next_up.md`, `blockers.md`) is manually curated summary state that bloats (249 lines vs 20-60 target) and goes stale because "distill at end of session" doesn't work reliably.

## Constraints
- Claude Agent SDK gives full coding tool harness (Read/Write/Edit/Bash/Glob/Grep) + hooks + permissions + subagents
- SDK exposes `input_tokens` via `message_start` streaming events with `include_partial_messages=True` — real-time context monitoring
- `ClaudeSDKClient` supports `interrupt()`, session resume/fork, multi-turn
- Context window is 200k tokens (Opus/Sonnet)
- One-person team — build delta must be minimal
- Separate from Helm — FS-1 is the agent runtime, Helm is the swarm orchestrator above it
- Python codebase (Agent SDK has Python package: `claude-agent-sdk`)

## Options Considered

### Raw API + Custom Tools ("Own Everything")
Build on raw `anthropic` client with `compact-2026-01-12` beta. Implement all coding tools. Full message array control.
- Gains: Total context window control, message-level access
- Costs: Months of work reimplementing Claude Code's tool harness
- Complexity: High

### Agent SDK + Journal-Makes-Compaction-Irrelevant ("Work Alongside")
Use Agent SDK, journal via hooks, tolerate opaque compaction because journal has the important state.
- Gains: All tools for free, fast to ship
- Costs: No context window control, compaction can still lose un-journaled info
- Complexity: Low-Medium

### Agent SDK + Session Cycling ("Never Hit Compaction")
Use Agent SDK with context monitoring. Monitor `input_tokens` from streaming events. When context hits threshold, interrupt session, start fresh with journal loaded into system prompt. Never reach compaction.
- Gains: All SDK tools for free. Full lifecycle control without reimplementing anything. Model journals with built-in Write tool. Session cycling is clean — interrupt + new query with journal in prompt.
- Costs: Small build (ContextMonitor, SessionManager, JournalLoader, system prompt). Relies on model discipline to journal.
- Complexity: Low

### Raw API + Letta Memory Layer
Raw API for agent loop + Letta for 3-tier memory management. Build minimal coding tools.
- Gains: Most powerful memory primitives (core/archival/recall)
- Costs: Two complex systems, Letta server/DB requirements, DIY coding tools
- Complexity: Medium-High

## Chosen Approach
**Agent SDK + Session Cycling** — Python harness wrapping `ClaudeSDKClient` with real-time context monitoring via `message_start` streaming events, proactive session cycling before compaction fires, and inline progressive journaling to per-session tmp directories. Model uses built-in `Write`/`Read` tools to manage its own journal — no custom MCP tools needed.

Replaces `memory-bank/durable/01-active/` with the journal tree as the "hot" active memory layer.

## Key Context Discovered During Shaping
- Claude Agent SDK streams `message_start` events with `input_tokens` count — exact context size per turn (requires `include_partial_messages=True`)
- `ClaudeSDKClient` has `interrupt()`, session resume/fork, `set_model()`, `set_permission_mode()` mid-session
- No Python framework gives both coding tools AND explicit context window control — session cycling is the pragmatic middle path
- Letta has the most mature programmable memory (3-tier: core/archival/recall, Context Repositories for git-versioned memory) — worth watching for future integration
- Mastra's Observer/Reflector pattern (configurable token thresholds, background compression, two-level rolling compaction) is the conceptual gold standard but TypeScript-only
- `ResultMessage.usage` gives cumulative totals; `message_start` gives per-turn snapshot — the per-turn snapshot is what you need for monitoring

## Relationship to Other Work
- **Helm V1** — FS-1 is the agent runtime, Helm is the swarm orchestrator. Helm dispatches work to FS-1 sessions.
- **Memory System V2** (compound memory + graduation) — the journal tree IS the hot tier implementation. Cross-session graduation from journal → warm → cold is a future evolution.
- **Cyrus** — FS-1 could eventually replace Cyrus as the agent runtime for delegated work, with journal-based context management built in.

## Next Step
- [Plan] → `/create_plan memory-bank/thoughts/shared/briefs/2026-03-02-fs1-agent-harness.md`
