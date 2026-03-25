---
type: event
created: 2026-03-16
status: active
---
# Idea Brief: Autoresearch Production Hardening

**Date:** 2026-03-16
**Status:** Shaped → Planning

## Problem
The autoresearch v3 harness runs end-to-end but can't run unattended overnight due to: (1) PreToolUse hooks silently ignored by `query()` — file-write boundary enforcement was never active, (2) SDK stderr noise from known bug #554, (3) need for split model config (Opus leader + Sonnet/Opus researcher), (4) daemon launcher for terminal use.

## Constraints
- Agent SDK `query()` ignores `hooks={}` (Issues #193, #213) — must use `ClaudeSDKClient` for real hook enforcement
- SDK v0.1.28 is the last clean version before #554 stderr regression
- CLOSE_WAIT socket leak (#665) affects long-running daemons — mitigate with short-lived client sessions
- `env={"CLAUDECODE": ""}` works for nested session suppression
- `setting_sources=[]` prevents loading project/user hooks from disk
- Budget: $20/experiment, Opus leader ~$0.025/turn, Sonnet researcher ~$0.60/turn, Opus researcher ~$1.00/turn

## Options Considered

### Quick Launch (query + weak guardrails)
Keep `query()`, remove dead `hooks={}`, rely on post-hoc `check_unexpected_files()` to reject boundary violations.
- Gains: Running tonight, minimal changes
- Costs: No real-time file-write enforcement, wasted researcher turns on violations
- Complexity: Low

### ClaudeSDKClient Migration (real hook enforcement)
Replace `query()` with `ClaudeSDKClient` context manager. Hooks fire properly, file-write boundaries enforced in real time. Add split model config, pin SDK, create daemon launcher.
- Gains: Real-time enforcement, proper per-role model config, clean logs
- Costs: ~1-2 hours, need to learn `ClaudeSDKClient` API
- Complexity: Medium

### Full Production Hardening
Everything above plus structured logging, metrics, retry logic, adaptive model selection.
- Gains: True hands-off operation with observability
- Costs: ~1-2 days, over-engineering for first campaign
- Complexity: High

## Chosen Approach
**ClaudeSDKClient Migration** — Real hook enforcement is worth the effort. The `query()` function was silently dropping our guardrails. `ClaudeSDKClient` is the SDK's intended pattern for hooks. Combined with split model config (Opus leader, configurable researcher), SDK pin, and a daemon launcher, this gets us to reliable overnight runs.

## Key Context Discovered During Shaping
- `query()` silently ignores `ClaudeAgentOptions.hooks` — only `ClaudeSDKClient` wires hook dispatch (Issues #193, #213)
- SDK bug #554 (v0.1.29+): `stream_input()` closes stdin prematurely, breaking hook callbacks. Fix in PR #580 not yet merged. Pin to v0.1.28.
- SDK bug #665: CLOSE_WAIT socket leak causes CPU spin in long-running daemons. No fix available. Mitigate with per-experiment client session teardown.
- `env` field in `ClaudeAgentOptions` MERGES with parent env (confirmed in `subprocess_cli.py:346-351`)
- All API tiers (1-4) have access to all models. The earlier 404s were wrong model IDs (`claude-sonnet-4-5-20250514` doesn't exist; correct: `claude-sonnet-4-5-20250929`).
- Opus 4.6 for lab leader is ~$0.01/experiment premium over Sonnet — negligible cost for better strategic diversity

## Next Step
- [Plan] → `/create_plan memory-bank/thoughts/shared/briefs/2026-03-16-autoresearch-production-hardening.md`
