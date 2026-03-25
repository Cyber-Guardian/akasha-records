---
type: event
created: 2026-03-20
status: active
---
# Idea Brief: Autoresearch Overnight Hardening

**Date:** 2026-03-20
**Status:** Shaped -> Planning
**Linear:** ENG-2642

## Problem
Autoresearch can't run 50+ experiments overnight unattended because several failure modes crash the entire campaign instead of recovering. Individual experiment resilience is solid (`execute_experiment_from_decision` try/except/finally works), but the layers above and below it have 9 specific gaps across 3 categories: campaign-killing crashes, silent waste, and operational blindness.

## Constraints
- Slack notification code already exists in `notify.py` but is dead code in several paths
- `run_experiment_loop` has NO try/except — exceptions from `_run_single_experiment` or `_run_thinker_iteration` crash the process
- `_log_and_save` is unprotected — state persistence failure causes recursive crash
- `_init_campaign` has minimal error handling — most calls are bare
- CLI has no top-level crash handler
- Dispatch has no retry logic — API failures return None immediately
- `tenacity` is a transitive dependency (available but not directly declared)
- Existing patterns to model: DynamoDB retry in `queue.py`, SIGTERM handling in `coordinator.py`, adaptive backoff in `budget.py`
- `CampaignState` persists to disk — supervisor restart is feasible

## Options Considered

### A. "Crash Shield" — Surgical try/except at each gap
5 targeted try/except blocks at exact crash points + wire existing Slack notifications.
- Gains: Campaign survives any single-experiment failure
- Costs: Transient API errors still waste experiments, no heartbeat for hung processes
- Complexity: Low (~60 lines)

### B. "Supervisor + Retry" — Auto-restart with smart dispatch
Everything in A plus campaign supervisor (max 3 restarts from persisted state) and dispatch retry with transient/permanent error classification.
- Gains: Process-level crash recovery, API hiccup retry, heartbeat
- Costs: More moving parts, need to verify `_init_campaign` idempotency
- Complexity: Medium (~200 lines)

### C. "Full Operations Stack" — Production-grade overnight runner
Everything in B plus orphan worktree scanner, eval log capture, campaign journal (JSONL), and wall-clock watchdog.
- Gains: Full operational picture, post-hoc diagnosis from Slack + journal, disk exhaustion prevention, hang detection
- Costs: More files to maintain
- Complexity: High (~400 lines across 5+ files)

## Chosen Approach
**C — "Full Operations Stack"** — B is the functional core (crash shields + supervisor + retry + heartbeat), C adds the operational layer (journal, watchdog, eval logs, orphan cleanup) that makes overnight runs diagnosable without SSH. The incremental cost over B is small once the supervisor is in place.

## Key Context Discovered During Shaping

### Campaign-killing crash points (Category 1)
- `run_experiment_loop` has no try/except — `coordinator.py:1870` while loop body is flat
- `_run_single_experiment` has no try/except — `coordinator.py:1375-1440` is flat
- `_run_thinker_iteration` has no try/except — `coordinator.py:1683-1752` is flat
- `_log_and_save` has no error handling — `coordinator.py:1618-1666`, if `state_store.save()` raises inside the `except Exception` handler of `execute_experiment_from_decision`, it causes recursive propagation
- `_init_campaign` — `coordinator.py:347-427`, only one narrow try/except for branch HEAD check
- CLI `start` command — `cli.py:25-40`, no top-level handler

### Silent waste (Category 2)
- Dispatch (`dispatch.py:64-66, 138-140`) catches all exceptions identically — no transient vs permanent distinction
- SDK exception hierarchy supports classification: `APIConnectionError`, `RateLimitError`, `InternalServerError` are transient; `AuthenticationError`, `CLINotFoundError` are permanent
- `ClaudeSDKClient` has no built-in retry and no wall-clock timeout — can block indefinitely if subprocess hangs

### Operational blindness (Category 3)
- `notify_crash` not called on researcher=None path (`coordinator.py:1204-1218`) or smoke_failed path (`coordinator.py:1253-1272`)
- No `notify_campaign_started` or `notify_campaign_complete` functions exist
- No heartbeat mechanism — hung process is silent
- Evaluator stdout/stderr discarded after parsing — no logs for post-hoc diagnosis
- Only `git worktree prune` at startup — no temp dir orphan cleanup

### Existing patterns to model
- DynamoDB transaction retry with exponential backoff: `discover/queue.py:446-545`
- SIGTERM/SIGINT + threading.Event: `coordinator.py:73-83`
- Adaptive backoff: `budget.py:48-62`
- Slack graceful degradation + rate limiting: `notify.py`
- `_init_campaign` has idempotency guards (`if state.champion_commit`, `if state.linear_issue_id`)

### From user's operational notes (Slack digest)
- MCP tool loading race condition observed — agent falls back to wrong path
- `core.bare` git corruption observed in agent worktrees
- Lab meta MCP returns too many tokens — context blow-out risk
- Dataset size mutation across experiments makes scores incomparable (evaluator pins dataset via env var, but researcher could theoretically cause side effects)

## Next Step
- [Plan] -> `/create_plan memory-bank/thoughts/shared/briefs/2026-03-20-autoresearch-overnight-hardening.md`
