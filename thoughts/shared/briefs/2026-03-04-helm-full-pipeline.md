# Idea Brief: Helm Full Pipeline Command

**Date:** 2026-03-04
**Status:** Shaped → Planning

## Problem
Helm V1 execution layer is complete but operator-driven — a human must invoke each step (decompose → status → promote → merge) and manually unblock phases when dependencies complete. For a one-person team, this switchboard operation is the bottleneck. The shape → plan → execution pipeline needs a single "approve and walk away" command.

## Constraints
- Helm owns the lifecycle loop; Claude is a callable tool for repairs (architectural decision)
- LINEAR_API_KEY required — must run from user's terminal
- One-level subagent nesting limit (Claude subprocess)
- Existing `invoke_review` and `invoke_conflict_resolution` patterns work and should be extended
- Failure modes are finite: review block, merge conflict, CI failure, agent timeout, unknown
- Fresh contexts beat degraded contexts — Claude repair invocations should be short-lived subprocesses
- `helm promote` (W18) doesn't exist yet — prerequisite for the pipeline

## Options Considered

### Promote + Auto-Chain (incremental)
Build `helm promote` standalone, then a `helm run` that chains decompose → poll → promote → merge. Each piece independently useful.
- Gains: Low risk, each piece ships value alone
- Costs: Still requires manual Linear issue creation. Two invocations minimum.
- Complexity: Medium

### Full Pipeline Command
One command: `helm execute <plan-file>` — creates parent issue, decomposes, delegates, polls, auto-promotes, auto-merges. Plan approval is the last human gate.
- Gains: True "approve plan, walk away, come back to merged code." Eliminates 6 manual steps.
- Costs: Larger blast radius on failure. Need robust abort/resume. LINEAR_API_KEY requirement.
- Complexity: Medium-High

### Event-Driven Daemon
Helm watches Linear webhooks / GH Action events. Fully reactive, no polling.
- Gains: No polling overhead, truly autonomous
- Costs: Infrastructure overkill for one-person team. Daemon debugging harder than CLI.
- Complexity: High

## Chosen Approach
**Full Pipeline Command** — `helm execute <plan-file>` creates the parent Linear issue, decomposes into sub-issues, delegates to agents, polls for completion, auto-promotes unblocked phases, runs adversarial review, and executes serial squash merges. Deterministic Python state machine with Claude as a callable repair tool. Unknown failures stop the loop and write structured errors for human pickup via `helm retry` / `helm recover`.

## Key Context Discovered During Shaping
- V1 execution layer fully complete — all 19 Linear issues Done
- 17/19 MVP weaknesses resolved; W18 (promote) and W11 (GH Action tests) remain
- `invoke_review` and `invoke_conflict_resolution` already work as Claude subprocess patterns
- Merge loop already handles partial progress (exits on failure, sets state to ACTIVE)
- `retry` and `recover` exist as escape hatches for failed orchestrations
- Finite failure modes: review block, merge conflict, CI failure, agent timeout, unknown

## Next Step
- [Plan] → `/create_plan memory-bank/thoughts/shared/briefs/2026-03-04-helm-full-pipeline.md`
