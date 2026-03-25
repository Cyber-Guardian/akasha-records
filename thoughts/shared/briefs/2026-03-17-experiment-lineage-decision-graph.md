---
type: event
created: 2026-03-17
status: active
---
# Idea Brief: Experiment Lineage & Decision Graph

**Date:** 2026-03-17
**Status:** Shaped → Planning

## Problem
Autoresearch experiments are stored as a flat sequential log, but the actual exploration is a tree of decisions — the leader sees prior results, reasons about them, and either builds on a success or backtracks from a failure. This decision structure is invisible in the data model and therefore impossible to visualize. The web dashboard's Multiverse tab needs the real exploration graph, not a heuristic guess.

## Constraints
- The leader already sees experiment history via `ExperimentLog.summarize()` (state.py:49) — 20 recent experiments sorted by score, capped at 12,000 chars
- History shows `#{idx}` sequential numbers but NOT experiment IDs — leader should reference by index, coordinator maps to ID
- `LabLeaderDecision` (models.py:114) uses PydanticAI structured output — adding optional fields is backward-compatible with validation + retry
- The atomic hypothesis constraint ("propose exactly ONE change") structurally implies single-parent lineage (tree, not DAG)
- `CampaignState` stores `champion_commit` but NOT `champion_experiment_id` — sweep mode needs this to record lineage
- Existing experiments (26 in dedup-compression) have no lineage — they'll show as a flat pre-lineage timeline
- Two distinct exploration modes need visual differentiation: LLM creative (DFS with backtracking) vs sweep systematic (BFS fan-out from champion)

## Options Considered

### Leader-Declared Lineage (Full Pipeline)
Add `parent_experiment: int = 0` to LabLeaderDecision. Leader references `#{idx}` from history. Coordinator validates index, maps to experiment_id, stores on ExperimentRecord. Sweep mode references champion_experiment_id from CampaignState. Web dashboard renders the decision tree.
- Gains: Ground truth lineage, real branching/backtracking visible, enables future leader prompt improvements
- Costs: 3 model changes + prompt + coordinator + viz. Leader may pick arbitrary parents initially.
- Complexity: Medium

### Inferred Lineage (Viz-Only)
Infer tree from temporal order + hypothesis text similarity in the dashboard. Zero harness changes.
- Gains: Ships immediately, works on existing data
- Costs: Noisy inference, can't distinguish backtrack from creative leap, throwaway when real lineage exists
- Complexity: Low

### Hybrid: Infer Now, Declare Later
Ship inferred lineage immediately. Add real lineage to harness in parallel. Viz prefers explicit, falls back to inferred.
- Gains: Immediate payoff + clean migration
- Costs: Two code paths, inference is throwaway
- Complexity: Medium-High

## Chosen Approach
**Leader-Declared Lineage (Full Pipeline)** — the inferred approach is a lie that doesn't capture the leader's actual decision process. The harness changes are small (3 fields, 1 format string, 1 validation) and the payoff is permanent. Existing pre-lineage data shows honestly as a flat timeline.

## Key Context Discovered During Shaping
- `ExperimentLog.summarize()` at state.py:102 formats as `#{idx} [{status}] score=...` — adding experiment_id here is one format string change
- Leader should reference parents by sequential `#{idx}` not hex IDs — LLMs handle short numbers more reliably ([/ground finding](https://pr-peri.github.io/llm/2026/02/13/why-hallucination-happens.html))
- MLflow uses identical single-parent "parent and child runs" model for experiment hierarchy
- PydanticAI validates structured output and retries on failure — hallucinated parent indices get caught
- `apply_sweep_result` at sweep.py:260-275 writes ONE record — needs `champion_experiment_id` from CampaignState for lineage
- The [[2026-03-17-multiverse-constellation-viz|Constellation Viz Brief]] is superseded by this approach — the decision graph IS the visualization

## Next Step
- [Plan] → `/create_plan memory-bank/thoughts/shared/briefs/2026-03-17-experiment-lineage-decision-graph.md`
