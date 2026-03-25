---
type: event
created: 2026-03-21
status: active
---
# Idea Brief: Collapse Thinker and Lab Leader Roles

**Date:** 2026-03-21
**Status:** Shaped → Planning

## Problem
The autoresearch loop has three decision-making tiers (thinker → leader → researcher) but PlannedAction and LabLeaderDecision are equivalent types (proven by `to_leader_decision()` at models.py:225). The leader is redundant in thinker mode — it's a weaker Sonnet call that produces the same output the thinker already produced. The fallback from thinker to leader is a divergent code path that drops prompt_pool, context curation, and gating.

## Constraints
- Must not break existing experiment records (LabLeaderDecision is used in build_record, novelty archive, solution tree)
- Researcher dispatch expects LabLeaderDecision — keep to_leader_decision() as the interface
- Novelty checking currently lives in _decide_hypothesis (leader path only) — needs a new home
- Thinker uses Claude Agent SDK (ClaudeSDKClient), leader uses pydantic_ai Agent — different dispatch mechanisms

## Options Considered

### Kill the Leader, Thinker Is Everything
Remove leader entirely. Thinker produces plans with plan_size=1 for what was "single mode". No fallback to a different agent.
- Gains: One code path, ~200 lines deleted, no divergence
- Costs: Opus cost on every planning call
- Complexity: Medium

### Leader Becomes Thinker Lite
Keep leader but make it produce PlannedAction. Unified output, model flexibility preserved.
- Gains: Cheap mode available, unified types
- Costs: Still two dispatch functions, two prompts
- Complexity: Low-Medium

### Thinker with Fallback to Self (chosen)
Remove leader. Thinker handles everything. Failure retries with simpler prompt (1 action) instead of falling back to a different code path. Post-plan novelty filter in router.
- Gains: One planner, one code path, one prompt family. Fallback stays in same abstraction. Fixes prompt_pool bug.
- Costs: Opus cost on fallback retries ($0.12/retry, negligible vs $2/experiment researcher cost)
- Complexity: Medium

## Chosen Approach
**Thinker with Fallback to Self + post-plan novelty filter** — The data models already prove these are the same concept. The fallback path is actively diverging (missing prompt_pool, context curation). Collapsing eliminates the divergence and simplifies the core loop.

## Key Context Discovered During Shaping
- `PlannedAction.to_leader_decision()` (models.py:225) — lossless conversion, proof of equivalence
- `router.py:177-178` — thinker path already skips novelty (creates synthetic NoveltyResult)
- `coordinator.py:1798` — fallback doesn't pass prompt_pool (bug)
- Leader cost is 0.6% of total experiment cost — removing it saves nothing meaningful
- Thinker cost is 6.3% — even doubling it for retry is negligible vs researcher cost (~$2/experiment)

## Next Step
- Plan → `/create_plan memory-bank/thoughts/shared/briefs/2026-03-21-collapse-thinker-leader.md`
