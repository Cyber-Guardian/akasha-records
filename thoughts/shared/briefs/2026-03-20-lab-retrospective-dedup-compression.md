---
type: event
created: 2026-03-20
status: active
tags: [autoresearch, retrospective, dedup-compression]
---
# Lab Retrospective: dedup-compression (2026-03-20)

## Analytics Snapshot
- **7 experiments** in ~37 minutes, $3.92 total
- **Champion:** `a346e911b55a` — score 0.5936 (2KB chunks + xxhash + zstd 9)
- **Keep rate:** 57% (4/7), **Crash rate:** 43% (3/7, 0 discards)
- **Failure modes:** 1 runtime_error (CDC smoke), 2 eval_timeout (CDC 120s, zstd-15 191s)

## Score trajectory
0.5657 → 0.5794 → 0.5930 → 0.5936 (compression level hill-climbing then chunk size)

## Key Findings

### Radical personality was never running (FIXED)
`config.yaml` was committed without `radical`. Every git operation reverted manual additions. Fixed by committing with radical added — commit `5c811041`.

### Evaluator metadata has redundant fields
`original_size_bytes` and `true_original_size_bytes` are always identical (both from `compute_true_size()`). The third field `reported_original_size_bytes` is the target's self-report. Missing: `unique_data_bytes` (unique chunk size before compression) to separate dedup savings from compression savings. See `evaluate.py:139-153`.

### Timeout is the dominant failure mode
2/3 crashes are `eval_timeout`. Champion at zstd 9 already takes 127s against 120s config timeout. The adaptive timeout in `_commit_and_evaluate` is the only reason it passes. Zstd 15 timed out at 191s. CDC approaches can't complete on 13.5GB dataset.

## Proposals

### 1. Increase `per_experiment_eval_timeout` to 240s
- **Data:** 2/3 crashes are timeouts. Champion is 7s over the config timeout.
- **Target:** `config.yaml:per_experiment_eval_timeout`
- **Type:** Config-level
- **Impact:** Unblock CDC and higher compression levels

### 2. Clean up evaluator metadata
- **Data:** `original_size_bytes` duplicates `true_original_size_bytes` in all keeps
- **Target:** `evaluate.py:139-153` — drop `original_size_bytes`, add `unique_data_bytes`
- **Type:** Code-level (evaluator hash change → champion invalidation)
- **Impact:** Leader can reason about dedup vs compression savings

### 3. Pass `evaluations` through `_log_and_save` for crash records
- **Data:** All 3 crash records have `evaluations: []`
- **Target:** `coordinator.py:_log_and_save()` — add `evaluations` kwarg
- **Type:** Code-level (minor)
- **Impact:** Diagnostic — know which tier caught failures

### 4. Add eval duration + timeout to failure lessons
- **Data:** Failure lessons are truncated, no context on how close to timeout
- **Target:** `coordinator.py:_generate_failure_lesson()`
- **Type:** Code-level
- **Impact:** Leader makes timeout-aware decisions

### 5. Leader prompt should include champion implementation details
- **Data:** Exp 8 re-tried xxhash when champion already uses it
- **Target:** `prompts.py:build_lab_leader_prompt()`
- **Type:** Code-level
- **Impact:** Prevent duplicate work
