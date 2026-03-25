# Idea Brief: Linear-Centric Agent Orchestration System

**Date:** 2026-02-24
**Status:** Shaped → Planning

## Problem

Software development has three phases — brainstorm/plan, execute, review — but they're not serial. Execution surfaces information that changes plans. Reviews reveal gaps that need re-planning. Today there's no system to handle these feedback loops across multiple concurrent agents. When an agent discovers during execution that the plan needs to change, there's no mechanism to check whether that change conflicts with sibling agents or violates higher-level project decisions.

Pair copiloting and fully agentic development are fundamentally different modes with different constraints, but are currently treated the same. No system distinguishes them or optimizes each independently.

## Constraints

- Linear Agent SDK is Developer Preview (since July 2025) — API may change but is well-documented
- Claude Code has no native Linear agent integration (open issue #12925) — Cursor/Codex do
- Linear triage rules are the proven delegation primitive (already working for CI failures)
- Linear hierarchy (initiatives → projects → issues) maps naturally to scope tiers
- Memory-bank is per-repo, not per-branch — limits agent isolation
- Solo developer + agents team — planning and reviewing are already the bottleneck

## Options Considered

### Linear as Passive Dispatcher (extend what exists)
Linear stays as work queue and status board. Triage rules dispatch to agents. Each agent works in isolation. Alignment is manual — human periodically reviews active branches and Linear state.

- Gains: Simplest, builds on existing patterns, full human control
- Costs: Human remains alignment bottleneck, review burden scales linearly with agents
- Complexity: Low

### Linear as Active Orchestrator (event-driven alignment)
Webhook-driven alignment checker monitors agent activity (PRs, status changes). Compares agent output against plans, acceptance criteria, and sibling work. Flags conflicts as Linear comments for human resolution.

- Gains: Alignment becomes event-driven, proactive flagging, scales better, extends triage-bot pattern
- Costs: New service to build, needs git + Linear + plan context, false positive risk
- Complexity: Medium

### Swarm with Autonomous Orchestrator Agent
Alignment layer is an agent with its own Linear identity. Can take action: update plans, reprioritize issues, create issues, block drifting PRs, post alignment reports. A "tech lead agent" that reads all PRs but doesn't write code.

- Gains: Closes alignment loop autonomously, sees the "multiverse" of branches, generates DAGs, reduces review burden to reviewing orchestrator decisions
- Costs: Hardest to get right, bad judgment cascades, hard context problem, needs iteration
- Complexity: High

## Chosen Approach

**Linear as Active Orchestrator** — with a graduation path to the Swarm Orchestrator.

The passive dispatcher is essentially what exists today. The full swarm orchestrator has a cold-start problem: we don't yet know what "good alignment checking" looks like, so giving an agent authority to act is premature. The event-driven checker lets us learn what conflicts actually look like in practice (flag-only mode), then graduate to autonomous action once we trust its judgment.

## Key Concepts from Shaping

- **Two execution modes**: `autonomous` (well-scoped, no human needed) and `co-dev` (needs judgment/creativity). Tagged during planning, determines process rigor.
- **Per-branch memory**: branch-scoped `current_work.md` so agents don't pollute each other's context. Shared files (project_brief, product_context) remain global.
- **Atomic commits per phase**: autonomous agents commit after each plan phase, not just at the end. Enables cross-agent communication via git history.
- **Cross-agent visibility**: structured branch names + commit messages let agents (and the orchestrator) inspect what siblings are doing.
- **Feedback loops are first-class**: execution and review can change plans. Changes must be justified against the larger project context. The alignment checker validates this.
- **Plans map to Linear artifacts**: scope determines tier — issue for small work, project for multi-issue features, initiative for strategic themes.
- **Research orchestration**: even research/exploration can be tracked as Linear issues, creating a complete episodic memory.
- **DAG-to-Linear mapping**: dependencies between issues form a DAG. The orchestrator can surface critical paths and bottlenecks.

## Linear Artifacts Created

| Issue | Title | Priority | Blocked By |
|-------|-------|----------|------------|
| ENG-2128 | Define execution discipline for autonomous agents | High | — |
| ENG-2129 | Build event-driven alignment checker | Medium | ENG-2128 |
| ENG-2130 | Graduate alignment checker to orchestrator agent | Low | ENG-2129 |

**Project:** Agent Orchestration (under DevOps & Quality initiative)
**Labels created:** `autonomous`, `co-dev`

## Next Step

Plan → Start with ENG-2128 (Execution Discipline) — it's the foundation the other two build on.
