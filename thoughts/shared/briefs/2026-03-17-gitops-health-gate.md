---
type: event
created: 2026-03-17
status: active
---
# Idea Brief: Gitops Health Gate + Isolation Hardening

**Date:** 2026-03-17
**Status:** Shaped → Planning

## Problem
Git state management is the primary source of friction when using Claude Code with worktree agents and parallel sessions. Stale worktrees accumulate (21 found in one session), dirty generated files leak across branches blocking checkout/rebase, `core.bare` corruption silently breaks git operations, and pre-push hooks fail on untracked files from other features. The issues aren't individual bugs — they're a missing lifecycle layer for git state validation and cleanup.

## Constraints
- Claude Code bug #27753 (open) — worktrees with unpushed commits are auto-deleted on exit
- `uv.lock` must stay tracked (lockfile for reproducible CI) — can't gitignore it
- `git stash` is blocked by PreToolUse hook (correct — shared state)
- `session_start.py` already has self-healing pattern (`core.hooksPath` fix, `_cleanup_old_sessions()`)
- Claude Code provides `WorktreeCreate`/`WorktreeRemove` lifecycle hooks (not needed for v1 but available for upgrade)
- `routing_decisions.jsonl` is tracked but nothing in CI reads it — safe to untrack

## Options Considered

### Worktree Lifecycle Hooks
Use Claude Code's native `WorktreeCreate`/`WorktreeRemove` hooks with a manifest file for per-worktree tracking and audit trail. SessionStart sweeps manifest for orphans.
- Gains: Correct lifecycle hooks, per-worktree granularity, audit trail
- Costs: Manifest is new state that can get stale, more moving parts
- Complexity: Medium

### Git Health Gate (SessionStart-only)
Extend `session_start.py` with `core.bare` check, `git worktree prune`, age-based worktree cleanup, and worktree count warning. No manifest, no new hook types.
- Gains: Simplest fix, single file change, no new state, proven pattern
- Costs: Reactive only (next session start), can't distinguish active from stale
- Complexity: Low

### Isolation Hardening (independent fixes)
Targeted fixes: untrack `routing_decisions.jsonl`, fix `post-checkout` uv.lock churn, scope pre-push to tracked files only, add `core.bare` check.
- Gains: Each fix independent and shippable in minutes, eliminates daily friction
- Costs: Doesn't solve worktree accumulation
- Complexity: Low

## Chosen Approach
**Git Health Gate + Isolation Hardening** — SessionStart worktree pruning + `core.bare` check + targeted dirty-state fixes. Covers all 7 observed failure modes with minimal complexity. Upgrade path to Worktree Lifecycle Hooks exists if scale demands it.

## Key Context Discovered During Shaping
- Claude Code has `WorktreeCreate`/`WorktreeRemove` hooks with `worktree_path` + `session_id` in JSON input — purpose-built for lifecycle management (not needed for v1 but upgrade path)
- `git worktree list --porcelain` gives machine-parseable output for scripting
- `git worktree prune` cleans stale admin refs natively
- Issue #26725 (anthropics/claude-code) confirms stale worktrees are a known unresolved problem
- `post-checkout` hook runs `uv sync` which regenerates `uv.lock` — fix is `git checkout -- uv.lock` after sync
- `routing_decisions.jsonl` is written by `transcript_validator.py` (line 66-76) — analytics only, safe to untrack
- `core.bare` corruption incident on 2026-03-16 — root cause unknown but detection is trivial to add

## Next Step
- Plan → `/create_plan memory-bank/thoughts/shared/briefs/2026-03-17-gitops-health-gate.md`
