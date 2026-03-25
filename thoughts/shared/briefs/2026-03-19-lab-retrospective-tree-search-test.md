---
type: event
created: 2026-03-19
status: active
---
# Lab Retrospective: tree-search-test

First campaign run with Wave 2 tree search architecture on 1.5GB dataset.

## Analytics Snapshot
- 8 experiments, 5 keeps (62%), 2 discards, 1 crash
- Champion: 0.3572 → 0.3852 (+7.8%)
- Cost: $3.16 total, $0.63/keep
- Tree depth 5, single productive lineage (#1→#4→#6→#7→#8)
- Dead-end branch correctly abandoned (#3)
- No stagnation, still improving at experiment #8

## Tree Search Validation
- Parent linkage: working (5 experiments build on specific parents)
- Node types: draft (#1), improve (#2-8) — leader correctly classifying
- Weighted fitness sampling: tree was too shallow (1 branch) to demonstrate backtracking
- save_experiment_ref: 7 git refs created, all reachable
- update_champion_ref: campaign branch updated on each keep
- Uniform crash early-stop: did not trigger (only 1/8 crashed)
- Eval timeout classification: correctly tagged #2 as eval_timeout

## Proposals

### 1. Force personality rotation in leader prompt
- **Data:** 8/8 experiments used pragmatic despite generalist being configured
- **Target:** prompts.py:build_lab_leader_prompt() line 270
- **Type:** code-level
- **Expected impact:** Increase hypothesis diversity, avoid monoculture exploitation of single strategy axis

### 2. Surface eval_metadata in leader context
- **Data:** Leader proposed "increase chunk size" 4x without knowing dedup_ratio was dropping
- **Target:** prompts.py — inject eval_metadata from champion into tree summary
- **Type:** code-level
- **Expected impact:** Data-informed hypotheses instead of blind axis exploitation

### 3. Increase budget to 15-20 experiments
- **Data:** 8 experiments too few for meaningful branching (only 1 active branch)
- **Target:** config.yaml budget.max_experiments
- **Type:** config-level
- **Expected impact:** 2-3 competing branches, real backtracking demonstration

### 4. Add academic personality
- **Data:** No research-backed approaches explored (only standard zlib/zstd/chunk-sizing)
- **Target:** config.yaml personalities
- **Type:** config-level
- **Expected impact:** Novel algorithms via WebSearch (FastCDC, content-defined chunking)

## Recommended Next Steps
1. Apply proposals #1 and #2 (code-level, high leverage)
2. Run 20-experiment campaign with 3 personalities (pragmatic, generalist, academic)
3. Compare tree topology: does personality diversity produce more branches?
