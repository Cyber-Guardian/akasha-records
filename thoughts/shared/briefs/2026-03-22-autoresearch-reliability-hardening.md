---
type: event
created: 2026-03-22
status: active
---
# Idea Brief: Autoresearch Reliability Hardening

**Date:** 2026-03-22
**Status:** Shaped -> Planning

## Problem
Autoresearch campaigns crash 30% of the time — all crashes are "Researcher returned no result" with $0 cost. We can't diagnose the root cause because the dispatch layer discards the SDK's diagnostic data (stop_reason, is_error, num_turns). The parallel dispatch infrastructure is shipped and working, but parallelizing crashes just wastes money faster. Reliability must come before throughput optimization.

## Constraints
- Must not change the SDK interface — work within the existing ClaudeAgentOptions/ResultMessage contract
- Parallel dispatch (DAG + TaskGroup) is already shipped — don't re-implement
- The thinker already produces multi-experiment plans — the router already fans them out
- Cost tracking is broken across the board ($0 for keeps too) — must fix alongside crash diagnosis
- Campaign is running — changes should be backwards-compatible with existing state.json/experiments.jsonl

## Options Considered

### Crash Forensics + Smart Thinker (bottom-up)
Fix crash causes surgically, then make the thinker reliability-aware.
- Gains: Fixes immediate 30% crash rate
- Costs: Multiple touch points, thinker prompt tuning
- Complexity: Medium

### Experiment Contract (top-down)
Define typed ExperimentContract validated before dispatch.
- Gains: Prevents crashes by construction
- Costs: More abstraction upfront, defining "complexity" quantitatively
- Complexity: Medium-High

### Adaptive Campaign Controller (feedback loop)
Real-time crash rate tracking with automatic adaptation.
- Gains: Self-healing, handles unknown failure modes
- Costs: Most complex, controller tuning is itself an experiment
- Complexity: High

## Chosen Approach
**Layered: surgical reliability fixes now, contract layer next, adaptive controller deferred.**

### Ship Now (this plan)
1. Dispatch diagnostics — capture stop_reason, is_error, num_turns, subtype in experiment record
2. Fix cost tracking — ResearcherUsage extraction from ResultMessage
3. Package constraint injection — requirements.txt contents in thinker + researcher prompts
4. max_turns scaling — thinker-specified complexity maps to turn budget

### Ship Next (Linear backlog)
- Typed ExperimentContract — Pydantic model validated before dispatch
- Thinker plans within contract constraints, marks actions independent/dependent

### Defer (Linear backlog)
- Adaptive campaign controller — real-time crash rate tracking + automatic adaptation
- Dynamic plan_size scaling based on stability
- Personality reallocation based on success rates

## Key Context Discovered During Shaping
- Parallel dispatch is already shipped: `router.py:346-369` uses asyncio.TaskGroup with DAG analysis
- All crashes are "Researcher returned no result" — dispatch.py:156 logs subtype but doesn't store it
- Cost tracking broken: `message.usage` dict may be None, ResearcherUsage gets $0 for all experiments
- `max_turns=30` hardcoded in dispatch.py:108; `researcher_timeout_seconds=600` in models.py:126
- The thinker keeps suggesting unavailable packages despite failure lessons
- Ground check: can't prove max_turns is the bottleneck without diagnostic data — fix #1 must come first
- Prior work: ENG-2642 overnight hardening plan addresses 9 gaps, several already shipped

## Next Step
- [Plan] -> `/create_plan memory-bank/thoughts/shared/briefs/2026-03-22-autoresearch-reliability-hardening.md`
