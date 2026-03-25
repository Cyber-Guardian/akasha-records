---
date: 2026-02-16T18:45:33.976584+00:00
source_research: memory-bank/thoughts/shared/research/2026-02-16-linear-claude-code-integration.md
last_generated: 2026-02-16T18:45:33.976584+00:00
---

# Research Brief: Linear + Claude Code Integration

## TL;DR

Linear has one of the most mature MCP integrations for Claude Code (40+ tools already connected via Anthropic-hosted MCP). The ecosystem spans official MCP, native "Linear for Agents" platform (delegation model), community skills/commands, and third-party agent integrations (Devin, Cursor, Copilot, Cyrus).

## Key facts / constraints

- **Already connected:** Anthropic-hosted Linear MCP provides 40+ tools (`mcp__claude_ai_Linear__*`) — no setup needed
- **Delegation model:** Linear treats agents as delegates (contributors), not assignees — humans maintain accountability
- **Rate limits:** 1,500 req/hr (API key), 500 req/hr (OAuth), 10K GraphQL complexity/query
- **Agent Interaction SDK:** Webhooks send full issue context on trigger; agents stream activities back
- **Issue sizing for AI:** Keep narrow enough for single context window, max 3 acceptance criteria
- **Ready-made repos:** svnlto/claude-code-linear-commands, ronanathebanana/claude-linear-gh-starter, wrsmith108/linear-claude-skill

## Key recommendations

- Add CLAUDE.md Route G for "Execute Linear issue" workflow
- Create `/create-linear-issue` and `/resolve-linear-issue` slash commands
- Use issue template: Summary + Acceptance Criteria + Scope + Context (with plan/research links)
- Follow Linear Method: action-verb titles, small scope, everyone writes their own issues
- Hook Linear status updates into plan lifecycle (`/implement_plan` phase transitions)
- Use branch naming `{type}/{ISSUE-ID}-{description}` for auto PR→Linear linking
- Consider TDD Guard hook pattern for test-first enforcement on Linear issues

## Open questions

- Multi-agent coordination on same Linear project (no clear docs)
- Webhook → Claude Code auto-trigger pattern (not yet documented)
- Cost management for high-volume agent API usage

## Links

- Source research: `memory-bank/thoughts/shared/research/2026-02-16-linear-claude-code-integration.md`
- Key external: [Linear MCP Docs](https://linear.app/docs/mcp) | [Linear for Agents](https://linear.app/agents) | [Ship Daily System](https://theailaunchpad.substack.com/p/my-ship-daily-system-with-claude) | [Cyrus Agent](https://github.com/ceedaragents/cyrus)
