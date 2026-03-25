---
type: event
created: 2026-03-20
status: active
---
# Idea Brief: Autoresearch Pyramid Evaluator

**Date:** 2026-03-20
**Status:** Shaped -> Planning

## Problem
The autoresearch evaluator has three critical bugs that invalidate the entire campaign's score history: (1) `original_size` is self-reported by the researcher's code — different implementations process different amounts of data (360KB to 13.5GB), making scores incomparable. The "champion" at 95.8% only processed 360KB. (2) No integrity validation — nothing verifies the output is actually correct. (3) No penalty for partial processing — timing out after processing 1% of data still gets a score.

## Constraints
- evaluate.py is "immutable" from the researcher's perspective but we own it
- The `compress_and_dedup(data_dir) -> dict` interface is the contract
- `EVAL_DATA_DIR` env var already controls dataset path (set by harness)
- `subset_dataset.py` already exists for generating reproducible subsets
- Evaluation is synchronous in the coordinator after researcher commits
- Current eval timeout is 180s

## Options Considered

### Pyramid Evaluator (chosen)
Three-tier evaluation: smoke (32MB, ~5s), signal (500MB, ~30s), validation (full dataset, ~120s). Evaluator independently computes true_original_size. Early termination for experiments that crash at tier 1 or don't beat champion at tier 2. Integrity checks at every tier.
- Gains: 3-5x faster feedback for bad experiments. Comparable scores. Catches garbage early. Budget savings.
- Costs: Pre-generate 3 dataset tiers. More complex evaluator. Tier 2 scores may not perfectly predict tier 3.
- Complexity: Medium

### Honest Single Evaluator
Fix the three bugs without changing architecture. Evaluator computes true_original_size, adds integrity check, fails partial processing.
- Gains: Simple. Fixes bugs. Comparable scores.
- Costs: No speed improvement. Bad experiments waste same time as good ones.
- Complexity: Low

### Multi-Metric Evaluator
Score on multiple axes (compression, throughput, memory, correctness). Pareto frontier tracking.
- Gains: Richer signal. Prevents single-metric tunnel vision.
- Costs: Complex. Requires changes to CampaignState, ExperimentRecord, dashboard. Over-engineered for now.
- Complexity: High — better as Wave 3 thinker scope.

## Chosen Approach
**Pyramid Evaluator** — tiered dataset sizes with evaluator-controlled true_original_size, early termination, integrity checks. Solves all three bugs AND adds speed.

## Key Context Discovered During Shaping
- evaluate.py:65 trusts `result["original_size"]` — this is the root of the comparability bug
- evaluate.py:68-69 returns 0.0 for original==0, but champion #22 reported 360KB original with 95.8% score — the code processed a tiny fraction
- EVAL_DATA_DIR at evaluator.py:67 already parameterizes the dataset — pyramid tiers just need different dirs
- `evaluator_runs` field exists on ExperimentConfig (default 1) — partial infra for multi-run
- The coordinator already has a pre-validation step (smoke eval at coordinator.py:1029) — tier 1 could replace/extend this
- Dataset: 13GB full, symlink at experiments/dedup-compression/dataset -> ~/Downloads/dataset. 32MB synthetic at dataset-synthetic/. subset_dataset.py can generate intermediate sizes.

## Additional Bugs to Fix (discovered in context)
- **Self-healing deps** was committed but config got overwritten — radical personality and lz4/brotli/lzma deps were lost when another branch merged. Need to re-add.
- **Config drift between branches** — the feat/ENG-2675 branch overwrote config.yaml, dropping radical + dimensions. Need to reconcile.

## Next Step
- [Plan] -> `/create_plan memory-bank/thoughts/shared/briefs/2026-03-20-autoresearch-pyramid-evaluator.md`
