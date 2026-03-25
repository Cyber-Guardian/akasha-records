# Idea Brief: Hive Mind — Inter-Agent Communication Layer

**Date:** 2026-02-28
**Status:** Seed (not yet shaped)

## Raw Thinking
- Agents should talk to each other — not just report to a human
- Tournament agents need to debate and challenge findings (real-time)
- Execution agents need to know what other agents changed (cross-session)
- Everything learned should persist for future agents (memory)
- The system should feel like a hive mind, not isolated workers

## Core Idea
Build a layered communication stack so agents at every level of the macro strategy (tournament, execution, review) are aware of what other agents are doing, have done, and have learned. Two complementary mechanisms:

1. **Agent Teams (real-time, single-machine)** — Claude Code's native experimental feature. Shared task list, direct messaging between teammates, self-coordination. Used for tournament debates (N agents challenge each other's approaches) and within-session parallel work.

2. **Event Log / MCP Server (cross-session, persistent)** — Hivemind-style append-only event log exposed as an MCP server. Any agent (local Claude Code, Cyrus worktree, plan-implementer subagent) can publish structured events and query the log. Used for execution coordination across concurrent agent sessions.

## Communication Stack

```
┌──────────────────────────────────────────────┐
│  Memory-Bank / Obsidian (weeks+)             │
│  Persistent knowledge: decisions, patterns,  │
│  taste. Every agent reads on startup.        │
├──────────────────────────────────────────────┤
│  Event Log — MCP server (hours)              │
│  Append-only: "I changed X", "I discovered   │
│  Y", "I'm blocked on Z". Cross-session.      │
│  Queryable by any agent.                     │
├──────────────────────────────────────────────┤
│  Agent Teams messaging (minutes)             │
│  Direct: debate approaches, challenge        │
│  findings, coordinate on shared files.       │
│  Single-machine, within one session.         │
├──────────────────────────────────────────────┤
│  Linear comments (async notifications)       │
│  Status updates, escalations, progress.      │
│  Already built via helm monitor + CI loop.   │
└──────────────────────────────────────────────┘
```

## How Each Layer Serves the Macro Strategy

| Macro strategy leg | Agent Teams (real-time) | Event Log (cross-session) | Memory-Bank (persistent) | Linear (async) |
|---|---|---|---|---|
| **Tournament ideation** | N agents debate approaches, challenge findings, judge ranks | — | Tournament results + rationale persisted as briefs | — |
| **Execution (helm)** | — | "I changed X", "I'm blocked on Y", file lock coordination | Decisions and patterns from execution | Session progress, escalations |
| **Visual review** | — | — | Visual artifacts linked in Obsidian graph | PR status |
| **Judge/taste agents** | Judge challenges worker output in real-time | Judge reads execution log to understand what happened | Taste evolves in persistent memory | — |

## Key Capabilities of the Event Log

**Publish (any agent writes):**
- `file_changed`: "I modified `components/filescience/models/entity.py` — added `MailFolder` type"
- `decision_made`: "Chose approach B for throttling because of X"
- `blocker_hit`: "Can't proceed — `cloud_api` component doesn't expose Y"
- `task_completed`: "Phase 2 of ENG-XXXX done, tests pass"

**Query (any agent reads):**
- "What has changed in `components/filescience/models/` since I started?"
- "Has anyone encountered issues with the throttling component?"
- "What decisions have been made about the queue processor today?"
- "Is anyone currently working on files in `bases/filescience/discover/`?"

**File awareness:**
- Advisory file locks (agent declares intent to modify, others see it)
- Conflict prevention before `git merge-tree` (earlier signal than merge time)
- Complements scope.json (scope defines allowed boundaries, event log shows actual activity)

## Relationship to Existing Work
- **Macro Agent Strategy** (2026-02-28) — this is the communication infrastructure connecting all three legs
- **Agent Helm Orchestration** (2026-02-27) — helm monitor already uses Linear comments as state; event log is the richer version for agent-to-agent awareness
- **Agent Swarm Harness** (2026-02-28) — swarm lifecycle management needs the event log to know what agents are doing
- **Judge/Taste Agents** (2026-02-28) — judges use Agent Teams for real-time evaluation, event log for understanding execution context
- **Memory-Bank OS** (2026-02-28) — memory-bank is the long-term layer; event log is the medium-term layer; they compose vertically
- **Agent CI Feedback Loop** (2026-02-26) — existing pattern of GH Action → Linear comment state; event log generalizes this to agent-to-agent

## External References
- [Claude Code Agent Teams](https://code.claude.com/docs/en/agent-teams) — native experimental feature. Shared task list, inbox messaging, lead/teammate model. Teammates message each other directly.
- [Hivemind](https://news.ycombinator.com/item?id=47088912) — persistent append-only event log as MCP server. File locking, semantic search, task awareness across sessions.
- [agent-recall](https://github.com/mnardit/agent-recall) — SQLite-backed knowledge graph with MCP server, extracted from 30+ concurrent agent system.
- [ClawVault](https://github.com/Versatly/clawvault) — structured memory for AI agents, markdown-native, graph-aware wikilinks.
- [Gemini Idea Generation](https://docs.cloud.google.com/gemini/enterprise/docs/idea-generation) — tournament pattern where agents generate and rank ideas via structured competition.

## Open Questions
- Event log implementation: SQLite (like agent-recall), flat files (like Hivemind), or something else?
- MCP server: build custom or adopt/fork an existing one (Hivemind, agent-recall)?
- How does the event log integrate with Cyrus? Managed Cyrus can't install custom MCP servers — event log may need to be file-based so Cyrus reads/writes via normal file operations.
- Agent Teams is experimental and single-machine — what's the fallback if it doesn't stabilize?
- How does the event log get pruned? Append-only grows forever — need a compaction/archival strategy.
- Semantic search: is it necessary, or is structured event types + file path queries sufficient?

## Next Step
- Shape when ready — needs to be shaped alongside the swarm harness (Phase 3 of macro plan) since both are about the execution environment. Could also influence Phase 4 (tournament) since Agent Teams is the mechanism for tournament debates.
