# Idea Brief: Extension Validation Pipeline — Phases 3-5 Redesign

**Date:** 2026-03-04
**Status:** Shaped → Planning (integrated into existing plan)

## Problem
Phases 3-5 of the extension validation pipeline had weak test designs: Phase 4's routing coherence test duplicated Phase 1 statically (strictly worse via LLM), Phase 5's promptfoo tests used Haiku as judge (empirically unreliable, omega 0.42-0.71) and checked presence not exclusivity, and Phase 3's Stop hook interaction with the existing agent-type hook was unanalyzed.

## Constraints
- All Stop hooks run in parallel (confirmed via docs) — no ordering conflict
- All `claude -p` flags verified (--no-session-persistence, --json-schema, --max-budget-usd, --stream-json)
- Haiku omega 0.42-0.71 for nuanced rubrics — not reliable as judge
- LLM judges add no signal over deterministic checks when structured output is available
- No production regressions documented — building prevention not remediation
- Only 2 of 5 Phase 1-2 blind spots are catchable by live CI tests

## Options Considered

### A. Lean Live + Rich Trace
Enrich Phase 3 transcript validation with Skill tool-call assertions. Replace Phase 4 routing coherence with stream-json tool-call trace. Replace Phase 5 LLM-judge with javascript contrastive-pair assertions.
- Gains: Cheapest ($6-11/mo), most reliable, leverages trace-first pattern
- Costs: No subjective quality testing (not needed for routing)
- Complexity: Low

### B. Two-Tier with Minimal Judge
Same as A but keep 2-3 Sonnet llm-rubric assertions for ambiguous boundaries.
- Gains: Catches edge cases deterministic checks can't express
- Costs: ~$0.10/run flakiness, marginal value over A
- Complexity: Medium

### C. Kill Phases 4-5, Double Down on Phase 3
Full trace-first only. No proactive CI testing.
- Gains: Zero CI LLM cost
- Costs: Only catches issues after they happen in real sessions
- Complexity: Low but reactive

## Chosen Approach
**A (Lean Live + Rich Trace)** — Best balance of proactive + reactive. Deterministic assertions are more reliable than LLM judges for structural routing correctness. Trace-first (Phase 3) catches everything else post-hoc.

## Key Context Discovered During Shaping
- promptfoo `javascript` assertions enable deterministic routing checks without LLM cost (promptfoo docs)
- Contrastive pairs (minimal-edit inputs with different expected routes) are the established pattern for routing exclusivity testing (arxiv 2410.01627)
- AgentAssay trace-first pattern achieves 78-100% cost reduction vs live testing (arxiv 2603.02601)
- `--output-format stream-json` captures intermediate tool calls via `content_block_start` events but is incompatible with `--json-schema`
- Anthropic recommends deterministic assertions every commit, LLM-judge at most on merge-to-main or nightly

## Next Step
- [Plan updated] → Changes integrated directly into `memory-bank/thoughts/shared/plans/2026-03-04-extension-validation-pipeline.md` (Phases 3-5 rewritten)
