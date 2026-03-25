---
date: 2026-03-04T12:00:00-05:00
source_research: memory-bank/thoughts/shared/research/2026-03-04-deep-context-window-optimization-token.md
last_generated: 2026-03-04T20:49:36.873306+00:00
---

# Research Brief: 2026-03-04-deep-context-window-optimization-token

## TL;DR

The largest controllable token costs are subagent spawns (~50K baseline each), large file reads (up to 25K tokens with 1.7x line-number overhead), and unbounded Bash stdout. The most impactful intervention is a **PreToolUse read-limiter hook** that auto-injects `limit` on large files and blocks wasteful paths before tokens enter context — this is strictly more effective than PostToolUse compression because `additionalContext` is additive-only (never replaces original output). Beyond hooks, **subagent model discipline** (haiku for search, sonnet for implementation, maxTurns limits) and **session rotation at 60-65% capacity** with handoff documents are the next highest-leverage practices. Our always-loaded instruction context (~3,290 tokens) is already lean and not worth further compression.

## Key facts / constraints

None captured.

## Key recommendations

None captured.

## Open questions

- How much does the Read tool's `cat -n` overhead actually cost us per session? (Need session-level telemetry)
- When will bug #15174 (SessionStart compact matcher) be fixed? This would unlock the PreCompact → re-injection pipeline.
- Can `updatedMCPToolOutput` be used reliably for Linear/Slack output compression, or does bug #24788 block it?
- What's the actual quality impact of session rotation at 60% vs 80% capacity? (Need A/B comparison)
- Would a TOON MCP proxy for our Linear/Slack integrations save meaningful tokens, given we already have MCP Tool Search?

## Links

- Source research: `memory-bank/thoughts/shared/research/2026-03-04-deep-context-window-optimization-token.md`
