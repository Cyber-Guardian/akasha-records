---
type: event
created: 2026-03-16
status: active
related:
  - thoughts/shared/briefs/2026-03-16-autoresearch-quality-revolution.md
  - thoughts/shared/plans/2026-03-16-autoresearch-production-hardening.md
---
# Idea Brief: Autoresearch Research Quality — Fork Scanner + Knowledge-First

**Date:** 2026-03-16
**Status:** Shaped → Planning

## Problem
The autoresearch loop uses a single mode (LLM-guesses-hypothesis → LLM-implements → evaluate) for two fundamentally different tasks: discovering novel approaches (research) and tuning parameters on a working approach (optimization). 5 experiments produced zero new champions after an initial 5% gain. The system is good at not wasting money but bad at finding improvements because the lab leader lacks domain knowledge, evaluator metadata is discarded, and parameter tuning is done by expensive LLM guessing instead of systematic search.

## Constraints
- Evaluator is cheap (~5s, $0.00/call) — systematic search can run hundreds without LLM cost
- Optuna is the right tool for parameter sweeps (grounding confirmed, not Hypothesis RBSM)
- Knowledge base should use existing memory-bank + QMD infrastructure
- Must work within Claude Agent SDK + PydanticAI stack
- Cross-campaign knowledge persistence is novel territory (beyond existing literature)

## Options Considered

### DeepEvolve-Inspired — Full research + evolve dual loop
6-module architecture mirroring DeepEvolve (plan → search → propose → code → debug → evaluate). Lab leader reads papers, builds knowledge base, generates grounded hypotheses.
- Gains: Proven architecture (9 benchmarks, 5x convergence). Knowledge compounds.
- Costs: Significant refactor. New Optuna integration. Research adds latency.
- Complexity: High

### Fork Scanner — Identify forks, sweep systematically
Pre-campaign analysis phase: lab leader identifies parameter forks, generates Optuna search space. Run 50-200 evaluator calls at $0. Only invoke research loop when sweep plateaus.
- Gains: Immediate ROI. Cheap. Simple. Clear mode separation.
- Costs: Doesn't build deep knowledge. Hand-wavy "when to research" heuristic.
- Complexity: Medium

### Knowledge-First Research Lab — Persistent domain knowledge
Deep research sprint before experiments: read papers, profile dataset, understand evaluator. Write to persistent knowledge base. Each campaign inherits prior knowledge.
- Gains: Compounds across campaigns. Dataset profiling transforms hypothesis quality.
- Costs: Upfront investment. Knowledge curation burden.
- Complexity: Medium

## Chosen Approach
**Fork Scanner + Knowledge-First (layered)** — Parameter sweep first for immediate gains ($0 cost), then knowledge layer for structural innovations. Gets 80% of DeepEvolve value with less complexity. Knowledge base naturally evolves into full DeepEvolve pattern over time.

## Key Context Discovered During Shaping
- DeepEvolve (arXiv 2510.06056, Oct 2025): augmenting AlphaEvolve with deep research yields 5x faster convergence, best algorithm in 5 generations vs 100+
- Optuna vs Code Llama (ICCV 2025): systematic search needs ~100 trials but explores full space; LLMs competitive for initial config but unstable for refinement
- ShinkaEvolve meta-scratchpad: periodic summarization of successful strategies into actionable recommendations — validates persistent knowledge pattern
- The evaluator produces rich metadata (dedup_ratio, throughput, memory, chunk counts) that never reaches the lab leader — this alone would transform hypothesis quality
- The dataset has never been profiled — hypotheses about "Office docs with version similarity" are ungrounded guesses

## Next Step
- [Plan] → `/create_plan memory-bank/thoughts/shared/briefs/2026-03-16-autoresearch-research-quality.md`
