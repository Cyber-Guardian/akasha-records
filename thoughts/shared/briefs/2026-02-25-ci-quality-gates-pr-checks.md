# Idea Brief: CI Quality Gates — PR Checks

**Date:** 2026-02-25
**Status:** Shaped → Planning
**Linear:** ENG-2135

## Problem
PR checks are dangerously thin — only ruff lint + import-linter run in CI. 582 tests exist but pytest doesn't run in CI at all. No security scanning (dependency audit or secret detection). A PR can merge with broken tests, vulnerable deps, or leaked secrets undetected.

## Constraints
- Target: full PR check suite in <10 minutes
- Polylith layout requires `pythonpath = [".", "components", "bases", "test"]` for pytest
- `uv sync --dev` already proven in CI (architecture job)
- detect-secrets needs initial `.secrets.baseline` generation + tuning
- radon is report-only (no gate) — xenon gating deferred to ENG-2156

## Options Considered

### Parallel Jobs ("Fan-out")
Add each check as an independent GHA job. lint.yml for static analysis, new test.yml for pytest + diff-cover. Maximum parallelism, clear per-job failure signals.
- Gains: Fastest wall-clock time, independent failure reporting, matches existing lint.yml pattern
- Costs: More runners (trivial cost), each pays checkout + uv sync tax (~10s)
- Complexity: Low

### Grouped Steps ("Batched")
Group related checks into fewer jobs (e.g., one "security" job). Fewer runners but serial within jobs.
- Gains: Fewer runners, logical grouping
- Costs: Slower overall, sequential failure masking, harder to see which check failed
- Complexity: Low

### Composite Action + Matrix ("DRY")
Reusable composite action for setup, matrix strategy to fan out checks. DRY but adds indirection.
- Gains: DRY setup, easy to add checks
- Costs: Over-engineered for 4-5 checks, harder to customize per-check
- Complexity: Medium

## Chosen Approach
**Parallel Jobs** — follows existing pattern, clearest failure signals, trivial setup tax. Split:
- `lint.yml`: ruff, import-linter, pip-audit, detect-secrets, radon report (5 parallel jobs)
- `test.yml` (new): pytest + diff-cover (1 job, sequential)

## Key Context Discovered During Shaping
- 582 tests, collection in 1.6s — full suite likely <60s in CI
- radon is the reporter (never exits non-zero), xenon is the gater — use radon for baseline pulse, defer xenon to ENG-2156
- `radon cc --total-average -s` per package gives aggregate grade without gating
- Prior backpressure research (Feb 6) already vetted all tool choices
- Ruff S-rules (bandit) already partially active via `select = ["ALL"]`

## Next Step
- Plan → `/create_plan memory-bank/thoughts/shared/briefs/2026-02-25-ci-quality-gates-pr-checks.md`
