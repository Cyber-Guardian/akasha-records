---
type: event
created: 2026-03-17
status: active
---
# Idea Brief: Triage Bot Model Upgrade + Observability Guardrail

**Date:** 2026-03-17
**Status:** Shaped → Implementing

## Problem
The triage bot (Haiku 4.5) ignores its own "always create an issue" instruction and instead gatekeeps, redirects, or declines valid requests — despite V2 prompt improvements already deployed. Observed twice: Harsh's logo request (redirected as "not for engineering") and Jordan's /outloop idea (redirected to Anthropic). Prompt-only fixes are proven insufficient for Haiku on nuanced judgment calls.

## Constraints
- Volume: ~5-15 invocations/day, ~6K input + ~800 output tokens each
- Cost impact of model upgrade: +$6/month (Haiku $3/mo → Sonnet $9/mo). Negligible.
- Latency impact: +4s per response (4s → 8s). Fine for async Slack replies.
- Model is parameterized via `MODEL` env var — swap requires zero code changes
- `_handle_result` (handler.py:447-449) is the only enforcement point — currently marks IGNORED silently with no logging

## Options Considered

### Model Upgrade Only (Sonnet)
Swap MODEL env var. No code changes. Sonnet follows nuanced instructions much more reliably.
- Gains: Eliminates the judgment failure class. Trivial change.
- Costs: +$6/month, +4s latency. No safety net if Sonnet also fails.
- Complexity: Trivial

### Code Guardrail Only (Keep Haiku)
Force-create a thin issue on any non-noise IGNORED completion. Deterministic safety net.
- Gains: Impossible for messages to fall through.
- Costs: Ugly fallback issues. Noise heuristic is hard (greeting detection). Doesn't fix the root cause (model judgment).
- Complexity: Medium

### Sonnet + Lightweight Guardrail (Hybrid) — CHOSEN
Upgrade to Sonnet AND add structured logging on IGNORED completions. Trust Sonnet, but make failures visible.
- Gains: Best of both — Sonnet handles judgment, logging catches drift. No noise heuristic needed.
- Costs: +$6/month. Residual failures still result in no issue (but are visible).
- Complexity: Low

### Sonnet + Hard Guardrail (Belt and Suspenders)
Upgrade to Sonnet AND force-create on IGNORED. Zero messages fall through.
- Gains: Deterministic. Zero-loss.
- Costs: Need noise heuristic. Fallback issues are ugly. Overkill for current volume.
- Complexity: Medium

## Chosen Approach
**Sonnet + Lightweight Guardrail** — Sonnet eliminates the judgment problem at negligible cost. Structured logging on IGNORED provides observability without the noise-heuristic complexity. Can promote to hard guardrail later if Sonnet proves insufficient.

## Key Context Discovered During Shaping
- V2 improvements plan fully implemented and deployed (commit 4a9821e) — prompt already had "Creation Bias" with explicit anti-gatekeeping language. Still failed. Prompt-only approach proven insufficient for Haiku.
- V2 plan's Phase 2 proposed code-level `ask_human_count` guard but it was never implemented (deferred due to durable execution harness constraints)
- `_handle_result` (handler.py:447-449) silently marks IGNORED — no structured logging, no alerting
- Cost analysis: Haiku ~$3/mo, Sonnet ~$9/mo, Opus ~$15/mo at current volume. All negligible.
- Latency analysis: Haiku ~4s, Sonnet ~8s, Opus ~9s per response. All acceptable for async Slack.
- Prompt edits from this session (stronger Creation Bias, "Who posts here", scoped self-awareness) are compatible and additive — keep them.

## Next Step
- [Implementing] → no plan needed, proceeding directly
