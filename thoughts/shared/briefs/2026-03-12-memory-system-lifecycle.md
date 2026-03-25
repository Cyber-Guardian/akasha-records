# Idea Brief: Memory System Lifecycle & Multi-Agent Access

**Date:** 2026-03-12
**Status:** Shaped → Planning

## Problem
FileScience's memory-bank architecture (v2 four-tier) is sound but has no lifecycle management, no temporal awareness in search, and only serves local Claude Code sessions. The bizdev operating system plan adds ~156 recurring files/year. Nightly scheduled agents (ENG-2183), the daily executive brief Lambda, and future autonomous agents need memory access from outside the local machine. The system needs to serve both in-loop (local, human-reviewed) and out-loop (remote, autonomous) agents — with different access patterns, different write permissions, and different working memory needs.

## Constraints
- Local files + QMD must keep working (primary authoring surface for in-loop work)
- Git remains version control for shared knowledge (free rollback, auditability, PR review)
- Remote agents need sub-second read access (can't clone repo per invocation)
- Only 19% of archive files have frontmatter — enforce going forward, don't backfill
- Memory MCP Server plan exists (ENG-2265) but is stale — needs v2 update
- `memory-sync.yml` workflow already exists
- Triage agent already has per-agent DynamoDB state — extend this pattern
- 3 live bugs to fix (stray scorecard-template, stale queue.md ref, broken state.md CI health)

## The Three-Concern Decomposition (Grounded)

Grounding research confirmed: every production multi-agent system (Google ADK, AWS AgentCore, Letta, Mem0, LangGraph) separates these three concerns. FileScience already partially implements this (triage agent DynamoDB state, agent-discipline read-only rules, session journals). The work is making it explicit and extending it.

### 1. Shared Knowledge (the "wiki")
- **What:** Project identity, topic notes, plans, briefs, research, intelligence, decisions
- **Read:** All agents (local via QMD, remote via S3 + Memory MCP Server)
- **Write:** Controlled — in-loop agents write directly, out-loop agents write via PR
- **Lifecycle:** Event notes roll up (weekly → monthly → delete originals). State notes update in place. Frontmatter metadata enables programmatic staleness detection and rollup.
- **Storage:** Git-primary, S3 mirror for remote reads

### 2. Agent Working Memory (per-agent state)
- **What:** Last-run timestamps, diff baselines, checkpoint data, intermediate results, thread state
- **Read/Write:** Owning agent only
- **Lifecycle:** TTL-based or checkpoint-based. No human review needed.
- **Storage:** Per-agent S3 prefix (`s3://filescience-memory/{agent-name}/`) or DynamoDB (extending triage agent pattern). Local agents use `.agent-state/{agent-name}/` (gitignored).

### 3. Output Routing (where agent results go)
- **Shared knowledge promotion:** Agent output → PR → merge → S3 sync (for durable intelligence briefs, research)
- **Notification:** Agent output → Slack/Linear (for alerts, standup digests, CI failures)
- **Ephemeral:** Agent output → log only (for one-time audits, intermediate checks)
- **Convention:** Each skill/agent declares its output destinations in its template/config

## Options Considered

### Lifecycle-Only (no remote access)
Add lifecycle management (rollup, pointers, frontmatter, maintenance) to the local system. Defer remote agent access.
- Gains: Simplest scope. Fixes the immediate problems (growing files, no temporal search, no staleness detection).
- Costs: Doesn't serve out-loop agents. They remain context-blind. Delays the bizdev intelligence pipeline (nightly competitive-intel agent has nowhere to read/write).
- Complexity: Low

### Full Three-Concern Architecture
Build all three layers: lifecycle management for shared knowledge, per-agent working memory infrastructure, output routing conventions, AND the Memory MCP Server for remote reads + PR-gated writes.
- Gains: Complete system. Serves both in-loop and out-loop agents. Future-proof for the nightly agent and daily brief plans.
- Costs: Largest scope. Memory MCP Server is a real deployment (Lambda + S3 + API). Multiple moving parts.
- Complexity: High

### Incremental — Lifecycle + Conventions + Deferred Infrastructure
Build lifecycle management (rollup, pointers, frontmatter, maintenance scripts). Define per-agent working memory conventions and output routing conventions as documented patterns. Defer the Memory MCP Server infrastructure until the first remote agent actually ships.
- Gains: All the lifecycle improvements land immediately. Conventions are in place so remote agents can follow them when they ship. No premature infrastructure. The "latest pointer" pattern and frontmatter metadata work for both local and future remote access.
- Costs: Remote agents still can't read shared knowledge until the MCP server ships. But nothing is blocked — the conventions are designed to be compatible.
- Complexity: Medium

## Chosen Approach
**Incremental — Lifecycle + Conventions + Deferred Infrastructure.** Build the lifecycle management and define the multi-agent conventions now. The Memory MCP Server (ENG-2265) ships when the first remote agent needs it — and when it does, the shared knowledge is already well-structured with frontmatter, pointers, and lifecycle metadata.

This approach:
- Fixes the immediate lifecycle gap (the reason we started this)
- Designs for multi-agent access without building premature infrastructure
- Makes the Memory MCP Server plan updatable (the stale ENG-2265 plan gets rewritten against the new conventions)
- Is compatible with the bizdev plan (Phase 0 series/ directory, latest pointers, rollup convention)

## Scope (Plannable Now)

### Immediate Fixes
- Move `scorecard-template.md` from `durable/` to `.claude/reference/`
- Fix CLAUDE.md Route K `queue.md` → Linear reference
- Clean up 40+ accumulated session dirs
- Fix `state.md` CI health workflow name mismatch

### Lifecycle Management
- Create `series/` directory for recurring bizdev files
- Define and document event vs state note classification
- Implement "latest" pointer pattern (`durable/latest-{series}.md`)
- Define frontmatter lifecycle schema (`type`, `created`, `status`, `series`)
- Document monthly rollup convention (weeklies → monthly summary → delete weeklies)
- Enforce frontmatter on all new files (update skill/command templates)

### Multi-Agent Conventions (documented, not yet infrastructure)
- Access matrix: which agent types get read/write to which artifact types
- Per-agent working memory convention: `.agent-state/{name}/` locally, `s3://.../{name}/` remotely
- Output routing convention: each skill declares output destination (shared-knowledge, slack, linear, ephemeral)
- PR-gated write path for out-loop agents (document the flow, don't build the automation yet)

### Maintenance Automation
- Weekly maintenance script (orphan detection, broken wikilinks, stale state notes, un-rolled-up event notes)
- Mem0-style dedup check (vector-search before creating new files)

### Deferred (Ships with First Remote Agent)
- Memory MCP Server infrastructure (Lambda + S3 + API) — update ENG-2265 plan
- S3 sync automation (extend `memory-sync.yml`)
- Per-agent DynamoDB/S3 working memory provisioning

## Key Context Discovered During Shaping

### From deep research (9 agents, 100+ sources):
- Every production memory system converges on tiered architecture — FileScience already has this
- Two-class content model (event vs state) is the most important lifecycle pattern — prevents unbounded accumulation
- Mem0's ADD/UPDATE/DELETE pipeline prevents duplication at creation time
- SimpleMem achieves 89-95% compression via recursive consolidation
- QMD is fine at current scale (362 docs) — the problem is temporal, not volumetric
- "Latest" pointer files are the simplest temporal solution — bypass search entirely
- obsidiantools enables programmatic maintenance (orphan detection, broken links, staleness)

### From grounding:
- Three-concern decomposition (shared knowledge, agent working memory, output routing) is the convergent pattern across Google ADK, AWS AgentCore, Letta, Mem0, Microsoft Agent Framework — not over-engineering
- FileScience already partially implements this (triage agent DynamoDB state, agent-discipline read-only rules, session journals)
- PR-gated writes are viable for weekly/nightly cadences (GitHub Agentic Workflows confirms: "PRs never merged automatically")
- Per-agent isolated storage is proven but can be lighter than DynamoDB at current scale (S3 prefixes or gitignored local dirs)

### Live bugs found:
- `scorecard-template.md` in `durable/` (263 lines in hot tier — shouldn't be there)
- CLAUDE.md Route K references deleted `queue.md`
- `state.md` CI health shows "not found" (workflow name mismatch)
- 40+ session dirs with no cleanup

## Related Artifacts
- [[2026-03-11-memory-system-deep-dive|Memory System Deep Research]] — full findings from 9 agents
- [[2026-03-11-bizdev-operating-system|BizDev Operating System Brief]] — the trigger for this work
- [[2026-03-03-centralized-memory-mcp|Centralized Memory MCP Brief]] — prior art, needs v2 update
- [[2026-03-03-memory-system-rearchitecture|Memory System Rearchitecture Brief]] — v1→v2 history
- [[2026-03-04-memory-system-v2-rearchitecture|Memory System v2 Plan]] — current architecture

## Next Step
- [Plan] → `/create_plan memory-bank/thoughts/shared/briefs/2026-03-12-memory-system-lifecycle.md`
