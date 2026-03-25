---
type: event
created: 2026-03-20
status: active
---
# Idea Brief: Parallel Research Agents

**Date:** 2026-03-20 (updated 2026-03-21)
**Status:** Shaped -> Planning

## Problem
The thinker produces multi-experiment plans (3-5 actions), but the router executes them sequentially. Each experiment takes ~2-5 min (researcher LLM call + 3-tier eval pyramid). A 5-action plan takes 10-25 min wall time. The Anthropic API handles concurrent requests and ECS Fargate burst infrastructure is live — we're leaving throughput on the table.

## Constraints
- Gate conditions (`fallback_action_index`) create dependencies between actions — forms a DAG, not a flat list
- Shared mutable state: CampaignState, ExperimentLog, SolutionTree, CampaignStateStore — currently lock-free because sequential
- Champion scoring is order-dependent — two experiments finishing simultaneously could both "improve" on the same baseline
- Dashboard `live_experiment.json` is single-file — needs N-experiment support
- Novelty archive writes need coordination for concurrent experiments
- ContainerSandbox currently starts its own local Docker container — needs to delegate to workspace MCP fork_workspace or wrap it for ECS

## Options Considered

### Fan-Out at Router Level Only
Modify execute_plan to dispatch independent actions via asyncio.TaskGroup. Gate-dependent actions stay sequential. Scoring serialized through asyncio.Lock.
- Gains: Minimal change, works anywhere
- Costs: Concurrency limited by local Docker host
- Complexity: Medium

### Burst Dispatch via Elastic Compute (chosen)
Thinker produces plan. Router analyzes DAG, groups independent actions. Uses fork_workspace/parallel_run to dispatch N researchers to Fargate tasks concurrently. Results collected, best-wins scoring applied.
- Gains: True elastic parallelism on shipped ECS infra. No local resource constraint. Full isolation per researcher.
- Costs: Bridge autoresearch ContainerSandbox with workspace MCP burst primitives. State synchronization.
- Complexity: Medium-High

### Hybrid Local-First
Ship router fan-out locally, upgrade to ECS later.
- Gains: Ship value sooner
- Costs: Two rounds of work, local Docker is limited anyway
- Complexity: Medium + Low incremental

## Chosen Approach
**Burst Dispatch via Elastic Compute** — the elastic compute infrastructure is already on main (fork_workspace, snapshot_workspace, parallel_run, Fargate burst). The work is orchestration glue: DAG analysis in the router, concurrent dispatch via workspace MCP primitives, serialized scoring, multi-experiment dashboard.

## Key Context Discovered During Shaping
- Thinker collapsed into leader (commit f04c26ba) — thinker prerequisite is done
- Elastic compute plan fully shipped to main: fork_workspace, snapshot_workspace, parallel_run, Fargate burst
- Researcher invocation is already async (invoke_researcher returns coroutine)
- Each experiment already gets isolated worktree + Docker sandbox namespaced by experiment_id
- router.py:327 execute_plan iterates sequentially with gate/fallback logic — this is the integration point
- ContainerSandbox in sandbox.py manages per-experiment local Docker containers
- Workspace MCP server.py has multi-workspace pool with async-safe mutations

## Dependencies
- [[2026-03-21-elastic-agent-compute|Elastic Agent Compute Plan]] — shipped
- [[2026-03-19-autoresearch-wave3-thinker-architecture|Wave 3 Thinker Architecture]] — shipped (collapsed into leader)

## Next Step
- [Plan] -> `/create_plan memory-bank/thoughts/shared/briefs/2026-03-20-parallel-research-agents.md`
