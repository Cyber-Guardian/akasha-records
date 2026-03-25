---
type: event
created: 2026-03-19
status: active
related:
  - thoughts/shared/plans/2026-03-18-autoresearch-dashboard-control-room.md
---
# Lab Retrospective: dedup-compression + tree-search-test

## Analytics Snapshot (updated 2026-03-20T00:24Z)

### dedup-compression
- 16 experiments | 7 keeps | 3 discards | 6 crashes | keep rate 43.8%
- Champion: 0.5355293 (adaptive 4MB/2MB chunks + zstd level 9, experiment bbb812daf95a)
- Previous champion (8903b998) held for 8 experiments — plateau confirmed
- Total cost: $7.82 | Cost/keep: $1.12
- Score trajectory: 0.250 → 0.536 (114% improvement over 16 experiments, 7 step-wise gains)
- Longest discard streak: 7 (experiments 9-15 all failed to beat champion)
- Personality imbalance: 87.5% pragmatic, 6.25% generalist, 6.25% radical, 0% academic
- **Optuna sweep module exists (sweep.py) but is never called** — $0 combinatorial search sitting unused
- **Eval timeout (120s) too tight for 13.5GB dataset** — lz4 (fastest algo) timed out

### tree-search-test
- 4 experiments | 0 keeps | 0 discards | 4 crashes | keep rate 0%
- **Completely broken** — all crashes at smoke eval load_target(), empty diffs
- Total cost: $1.33 wasted with zero signal
- Root cause: infrastructure failure, not research failure (tracebacks truncated)

## Proposals

### 1. Wire up the Optuna sweep (CRITICAL — highest ROI)
- **Data:** 8 experiments ($2.41) produced +0.004% improvement via sequential LLM hill-climbing. sweep.py has a ready Optuna TPE sweep that tests combinations at $0 LLM cost — but coordinator.py never calls it.
- **Target:** coordinator.py — add sweep trigger after N consecutive discards
- **Type:** Code-level
- **Expected impact:** 100-trial sweep over chunk_size x zstd_level costs $0 LLM, ~200 min eval. The champion has 2 clearly tunable params (chunk_size, zstd_level) — systematic search is strictly better than random LLM proposals for this parameter space.
- **Implementation:** When consecutive_discards >= 3, call run_parameter_sweep(config, n_trials=50). If best_score > champion_score, call apply_sweep_result(), then resume LLM exploration from new baseline.

### 2. Fix eval timeout vs dataset size mismatch
- **Data:** lz4 (fastest compression algo) timed out at 120s on 13.5GB. brotli also timed out. Dataset grew 10x (1.4GB → 13.5GB) mid-campaign but per_experiment_eval_timeout unchanged.
- **Target:** experiments/dedup-compression/config.yaml — increase to 300-600s. Or add dataset-size-aware timeout to coordinator.py.
- **Type:** Config-level (testable)
- **Expected impact:** Recover brotli + lz4 experiments that might beat zstd. 2 experiments ($0.91) wasted on false timeouts.

### 3. Fix tree-search-test infrastructure failure
- **Data:** 4/4 crashes at load_target(), empty diffs, truncated tracebacks, $1.33 wasted
- **Target:** validation.py:27-52, coordinator worktree setup
- **Type:** Code-level investigation
- **Expected impact:** Unblock entire campaign (0% → ~50% keep rate)

### 4. Surface available packages to researcher prompt
- **Data:** Experiment eff2c0de98be spent $0.75 on ModuleNotFoundError: fastcdc
- **Target:** coordinator.py:618-643 + prompts.py:321-446
- **Type:** Code-level
- **Expected impact:** Eliminate import_error crashes (~6% of experiments)
- **Implementation:** After install_requirements, run pip list in sandbox, inject into researcher prompt

### 5. Include champion eval_metadata in leader prompt
- **Data:** Leader told researchers champion uses "zstd level 3" (actually level 9). 3 experiments wasted ($1.36)
- **Target:** prompts.py:155-318 (build_lab_leader_prompt)
- **Type:** Code-level
- **Expected impact:** Reduce parameter confusion, ground leader in concrete performance data

### 6. Enforce campaign-level personality diversity
- **Data:** 87.5% pragmatic, academic never selected. Campaign stuck tweaking single variables in a flat landscape.
- **Target:** coordinator.py:400-466 + prompts.py leader prompt
- **Type:** Code-level
- **Expected impact:** Break local optimum by forcing literature-informed approaches

### 7. Capture failure tracebacks tail-first
- **Data:** All tree-search-test tracebacks truncated at same point, hiding actual error
- **Target:** validation.py:27-52
- **Type:** Code-level
- **Expected impact:** Faster crash diagnosis. Change stderr[:2000] → stderr[-2000:]

## Recommended Next Steps
1. **Wire Optuna sweep** (Proposal 1) — the campaign is in a flat landscape; systematic $0 search is strictly better than $0.49/experiment LLM guessing
2. **Increase eval timeout** (Proposal 2) — quick config fix, recovers 2 promising directions (brotli, lz4)
3. **Investigate** tree-search-test root cause (Proposal 3)
4. **Implement** Proposals 4, 5, 7 together — small targeted harness improvements
5. **Implement** Proposal 6 — after other fixes land, to diversify exploration
