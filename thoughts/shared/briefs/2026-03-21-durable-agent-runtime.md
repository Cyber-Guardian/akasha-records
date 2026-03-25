---
type: event
created: 2026-03-21
status: active
---
# Idea Brief: Durable Agent Runtime (Git-as-Working-Memory)

**Date:** 2026-03-21
**Status:** Shaped → Planning

## Problem
When a deep agent (plan-implementer in a workspace container) hits maxTurns or exhausts its budget, all uncommitted work is lost. Today we lost Phase 2 of ENG-2642 three times — the agent spent turns on dep install, reading, and writing code, ran out before commit+push, and the container was destroyed. The workspace container survives agent death, but there's no mechanism to checkpoint work incrementally or recover it.

## Constraints
- Claude Code's `maxTurns` is a hard kill — no structured handback to parent
- Workspace containers survive agent death (Docker stays running until `destroy_workspace`)
- The workspace MCP server (`tools/workspace-mcp/`) is the integration point — it already handles create, exec, push, destroy
- Auto-commit must be transparent (zero agent turns consumed) to not eat into the budget it's protecting
- Must work with existing plan-implementer agent definition and dispatch patterns
- Squash-merge on push to keep remote branch history clean (conventional commits on mainline)

## Options Considered

### Checkpoint-First Agent Convention
Change agent workflow to commit+push early and often. Zero infra changes.
- Gains: Works today, no code changes
- Costs: Agent spends turns on git ops, doesn't solve unexpected death
- Complexity: Low

### Parent Rescue Protocol
Parent always inspects workspace after agent returns, commits orphaned work.
- Gains: No work ever lost, parent stays in control
- Costs: Requires updating orchestration patterns, adds overhead per dispatch
- Complexity: Medium

### Git-as-Durable-Runtime (chosen)
Auto-commit every write inside workspace containers. Timeline tool for temporal awareness. Checkpoint tool for named milestones. Baked into workspace MCP server.
- Gains: Zero-turn-cost persistence. Full temporal audit trail. Trivial crash recovery. Re-dispatch reads timeline to pick up where last agent left off.
- Costs: More git history (squashed on push). New MCP tools to build and maintain.
- Complexity: Medium

## Chosen Approach
**Git-as-Durable-Runtime** — Every file mutation inside a workspace container auto-commits with a `wip:` prefix message. The git log becomes the agent's temporal memory. Three new capabilities in the workspace MCP server: transparent auto-commit (internal), `timeline` tool (compact git log for agent orientation), `checkpoint` tool (named milestones for recovery). Push squashes `wip:` commits into one conventional commit.

## Key Context Discovered During Shaping
- Temporal.io's event-sourced checkpointing is the conceptual model — every step checkpointed, replay returns cached results
- Cursor 1.7 uses hidden git refs for timeline snapshots during execution — same idea, different mechanism
- Our existing atomic-commit-per-phase pattern is already a coarse checkpoint — this makes it continuous
- The workspace MCP server at `tools/workspace-mcp/src/workspace_mcp/server.py` is the natural integration point
- `container.py` exec method runs commands via `docker exec` — auto-commit hooks here
- Research: [[2026-03-04-serverless-agent-harness|Serverless Agent Harness]] — durable Lambda pattern with deterministic step naming

## Next Step
- Plan → `/create_plan memory-bank/thoughts/shared/briefs/2026-03-21-durable-agent-runtime.md`
