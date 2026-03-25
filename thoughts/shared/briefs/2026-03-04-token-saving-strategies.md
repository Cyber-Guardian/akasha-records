# Idea Brief: Token Saving Strategies Beyond RTK

**Date:** 2026-03-04
**Status:** Shaped → Planning

## Problem
RTK covers CLI output filtering (60-90% savings on dev ops), but other major token waste categories — MCP server idle overhead, context lifecycle, thinking budget, CLAUDE.md size — remain unoptimized and likely account for significant additional spend.

## Constraints
- Must not degrade response quality for complex tasks
- Must be low-friction (no manual steps per session)
- RTK already handles PreToolUse Bash command filtering
- MCP servers (Linear, Figma, Slack, Chrome DevTools) are needed for some workflows
- CLAUDE.md routing table is load-bearing — can't just shrink it arbitrarily

## Options Considered

### MCP Server Audit + Pruning
Disable MCP servers rarely used in sessions. Each adds tool definitions to context even when idle. Figma and Chrome DevTools are session-specific — only enable when needed.
- Gains: Immediate, zero-effort token reduction per session (1-5K tokens/server)
- Costs: Slight friction to re-enable when needed
- Complexity: Low

### Thinking Budget Throttle
Set MAX_THINKING_TOKENS=8000 for routine work. Default is 32K — overkill for simple tasks.
- Gains: ~70% thinking token reduction on routine tasks
- Costs: May hurt complex architectural reasoning
- Complexity: Low — env var or Option+T toggle

### Context Lifecycle Discipline (Proactive Compaction)
Set CLAUDE_AUTOCOMPACT_PCT_OVERRIDE=50 so compaction fires at 50% fill instead of 95%. Combined with aggressive /clear between tasks.
- Gains: 40-60% session waste reduction, better response quality
- Costs: More frequent compaction pauses
- Complexity: Low

### RTK v2 — PostToolUse Output Filtering
Extend RTK's hook pattern to filter tool results (Read output, agent results) via PostToolUse hooks. File reads and agent results are the #1 token consumer.
- Gains: Could match or exceed current RTK savings
- Costs: Significant engineering effort, risk of filtering useful info
- Complexity: High

## Chosen Approach
**Stack approaches 1 + 2 + 3** — all low-effort, high-impact, independent config changes. MCP audit is one-time. Thinking throttle is an env var. Compaction threshold is a setting. Together they reduce baseline spend before investing in harder RTK v2 work.

## Key Context Discovered During Shaping
- 60+ deferred tool definitions across Linear, Figma, Slack, Chrome DevTools MCP servers
- ENABLE_TOOL_SEARCH=auto already helps by deferring tool loading
- Default thinking budget is 31,999 tokens — billed as output tokens
- Auto-compaction default is 95% context fill — quality degrades before it fires
- RTK intercepts 80+ command patterns via PreToolUse hook in ~/.claude/hooks/rtk-rewrite.sh
- PostToolUse filtering for Read/Agent results is less documented but feasible

## Next Step
- [Plan] → /create_plan memory-bank/thoughts/shared/briefs/2026-03-04-token-saving-strategies.md
