# Idea Brief: CI Fix Loop — /create-pr Skill

**Date:** 2026-03-04
**Status:** Shaped → Parked

## Problem
When working locally with Claude Code, after pushing a PR there's no feedback loop for CI failures. The cloud-side system (agent-ci-feedback.yml) handles this for Cyrus via Linear comments + round tracking, but local interactive sessions require manual check-fix-push cycles.

## Constraints
- `gh` CLI available and used extensively (pr_utils, merge, review)
- `gh run watch` blocks until run completes; `gh run view --log-failed` fetches failure logs
- Cloud system has MAX_ROUNDS=3 — local version should match
- Claude Code `run_in_background` Bash lets agent go idle while waiting
- Agent gets notified automatically when background task completes

## Options Considered

### Pure skill + background Bash
A `/create-pr` skill that runs `gh pr create`, launches `gh run watch` via `run_in_background`, then fixes on failure. Zero new code — just a skill markdown file using existing primitives.
- Gains: Simplest approach, no scripts to maintain
- Costs: Relies on gh run watch behaving well, skill prompt must be clear for fix loop
- Complexity: Low

### Shell script + skill wrapper
A `scripts/ci-watch.sh` script for structured polling/log extraction, wrapped by a skill.
- Gains: Structured JSON output, reusable outside Claude Code
- Costs: More code to maintain
- Complexity: Medium

### Hook-based auto-trigger
PostToolUse hook detects `git push` → auto-starts background CI watcher.
- Gains: Fully automatic
- Costs: Triggers on every push, harder to control, stretches hook pattern
- Complexity: Medium-High

## Chosen Approach
**Pure skill + background Bash** — simplest thing that works. `gh run watch` and `gh run view --log-failed` already exist. No new scripts needed.

## Key Context Discovered During Shaping
- agent-ci-feedback.yml has sophisticated round tracking, SHA dedup, and escalation logic that could inform the local version
- Helm's pr_utils.py already uses `gh pr list --json` with statusCheckRollup — pattern to follow
- `run_in_background` Bash is the key primitive — agent goes idle (no token consumption) until notified

## Next Step
- [Parked] → Linear backlog issue: ENG-2326
