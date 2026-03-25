# Idea Brief: Centralized Memory + MCP Server with Per-Session Working Memory

**Date:** 2026-03-03
**Status:** Shaped → Planning

## Problem
Memory is trapped inside the filescience repo and only accessible to Claude Code sessions via file reads. Deployed agents (triage bot Lambdas) have zero project context — just a static YAML routing table. As the agent fleet grows (Cyrus, nightly agents, Helm workers, FS-1 sessions), each faces the same cold-start problem. Meanwhile, the current memory-bank architecture conflates stable project state, active working memory, and historical archive into a structure that bloats (current_work.md at 249 lines vs 20-60 target) and goes stale.

## Constraints
- Monorepo — MCP server and memory restructure happen in this repo, not a separate one
- Triage bot uses plain `anthropic.messages.create()` — no MCP client today; consumer wiring is a follow-on
- Current memory is ~242 archival files + 6 durable files — well within markdown scale (no vector DB needed)
- Claude Code subagents (9 in `.claude/agents/`) access memory via file reads — MCP is additive, not replacement
- FS-1 harness (Agent SDK + session cycling + progressive journaling) is being built in parallel — journal = per-session working memory
- Lambda infrastructure patterns already exist in this repo (triage bot)

## Options Considered

### MCP-First (expose current memory, iterate on structure later)
Build the MCP server now against the current messy memory-bank structure. Validate the interface with real consumers before restructuring.
- Gains: Fast to ship, real usage patterns inform restructure
- Costs: Noisy results from bloated/unstructured files, may need to redo MCP tool design
- Complexity: Medium

### Structure-First, Then MCP (V2 restructure → expose clean data)
Do the full hot/warm/cold restructure first, then build MCP on clean data.
- Gains: Clean data from day 1 of MCP, stable tool design
- Costs: Delays access story, two sequential efforts
- Complexity: High (total)

### Combined — Structure + MCP in parallel, structure-priority
Design memory structure and MCP interface together. Structure leads. MCP designed against target structure and lands shortly after.
- Gains: Cohesive design, no throwaway work, structure and interface are mutually reinforcing
- Costs: Largest scope, but well-informed by extensive SOTA research already completed
- Complexity: High

### Lightweight — No MCP, just structured repo access
Restructure memory-bank, bundle snapshots in deployment artifacts for deployed agents.
- Gains: Simplest, no new infrastructure
- Costs: No live queries, doesn't scale with agent count
- Complexity: Low

## Chosen Approach
**Combined — Structure + MCP in parallel, structure-priority** — with a critical revision from the FS-1 harness shaping: the centralized "warm/active" tier is eliminated. Active working memory is per-session (FS-1 journal pattern), not centralized. Centralized memory is either stable state (durable) or searchable history (archive).

### Three-layer model

| Layer | Scope | What lives here | Access |
|-------|-------|----------------|--------|
| **Durable** (hot) | Centralized, shared | Project identity, stable state, focus queue, blockers | MCP-served + file reads |
| **Session journal** (warm) | Per-session, ephemeral | Working notes, decisions, findings, progress | Local files in session tmp dir (FS-1 journal) |
| **Archive** (cold) | Centralized, shared | All historical knowledge — briefs, plans, research, decisions | MCP-served (search, get_briefing) |

### Why no centralized warm tier
The previous V2 proposal had an `active/` directory with ~15-20 shared topic notes (e.g., `active/throttling.md`). The FS-1 harness insight killed this: active working memory is inherently session-scoped. Two agents working on different tasks shouldn't share working notes — it's a concurrency and staleness nightmare. Cross-session topic knowledge belongs in the searchable archive, not a special tier. Sessions that produce valuable knowledge graduate their journals to the archive on close.

## Key Context Discovered During Shaping
- Extensive SOTA research completed in prior session — every production memory system converges on tiers, hybrid "files + derived index" is the winning pattern, automated consolidation differentiates good from great systems (see handoff: `memory-bank/thoughts/shared/handoffs/general/2026-03-02_16-31-03_memory-system-architecture-rethink.md`)
- Triage bot (`tools/triage-bot/`) uses plain Anthropic SDK, not Agent SDK — has zero runtime access to memory-bank, only a static YAML config for project routing
- 9 Claude Code subagents in `.claude/agents/` access memory via file reads — MCP is additive
- FS-1 harness (`memory-bank/thoughts/shared/plans/2026-03-03-fs1-agent-harness.md`) scopes working memory to `tmp/sessions/{session_id}/` with progressive journaling — this IS the warm tier implementation
- Current memory-rules.md targets are wildly violated (current_work.md 249 vs 20-60, next_up.md 114 vs 10-40) — validates structural change
- Lambda-backed MCP server is the pragmatic starting point for deployment

## Related Work
- FS-1 harness brief: `memory-bank/thoughts/shared/briefs/2026-03-02-fs1-agent-harness.md`
- FS-1 harness plan: `memory-bank/thoughts/shared/plans/2026-03-03-fs1-agent-harness.md`
- Memory system architecture rethink handoff: `memory-bank/thoughts/shared/handoffs/general/2026-03-02_16-31-03_memory-system-architecture-rethink.md`
- Memory-Bank OS brief (partially superseded): `memory-bank/thoughts/shared/briefs/2026-02-28-memory-bank-os.md`
- Hive Mind brief (overlapping): `memory-bank/thoughts/shared/briefs/2026-02-28-hive-mind-agent-communication.md`
- Helm V1 plan: `memory-bank/thoughts/shared/plans/2026-03-02-helm-v1-execution-layer.md`

## Next Step
- [Plan] → `/create_plan memory-bank/thoughts/shared/briefs/2026-03-03-centralized-memory-mcp.md`
