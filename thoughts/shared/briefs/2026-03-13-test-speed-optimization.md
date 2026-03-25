---
type: event
created: 2026-03-13
status: active
---
# Idea Brief: Pre-Push Test Speed Optimization

**Date:** 2026-03-13
**Status:** Shaped → Planning

## Problem
The pre-push hook takes 30-60s, blocking developer flow. The root test suite (44s) is 83% of wall clock time, but 75% of that 44s is wasted on three fixable problems: function-scoped moto DynamoDB fixtures (19.6s), unmocked subprocesses in session_start tests (7.3s), and a real anyio.sleep loop in test_telemetry (6.0s).

## Constraints
- Pre-push must stay reliable — false greens defeat the purpose
- Polylith `--import-mode=importlib` is required for namespace packages
- `asyncio_mode=auto` with function-scoped event loops
- moto DynamoDB fixtures need isolation (tests mutate table state)
- Two independent pytest roots (test/ and tools/test/) with separate venvs
- 10-core Mac locally, 4-vCPU Blacksmith runners in CI

## Options Considered

### Surgical Hot-Spot Fixes
Fix the three identified time sinks: mock anyio.sleep, module-scope moto fixtures, mock subprocess.run in session_start.
- Gains: 44s → ~18s. Zero new dependencies. Each fix is a 5-line change.
- Costs: Doesn't address structural scaling — 903 sequential tests will slow as suite grows.
- Complexity: Low

### pytest-xdist Parallelization
Add pytest-xdist with -n auto. Research confirms compatibility with importlib mode (pytest ≥ 8.2), asyncio_mode=auto (xdist ≥ 3.6.0), and moto (process isolation).
- Gains: Scales with hardware. Combined with hot-spot fixes: ~5s.
- Costs: New dependency. Slightly more complex failure output.
- Complexity: Medium

### Polylith-Aware Differential Pre-Push
Script that maps git diff to test directories via import-linter dependency graph. Only run tests for changed bricks + dependents.
- Gains: Typical push 2-5s regardless of total test count. Leverages existing architectural metadata.
- Costs: Custom script to maintain. Could miss behavioral cross-brick regressions (mitigated by CI running full suite).
- Complexity: Medium

### Lighten Gate (lint only, tests in CI)
Reduce pre-push to lint-imports + ruff (~1.6s). Full tests run in CI only.
- Gains: Near-instant push.
- Costs: Broken code reaches remote. Wastes CI cycles.
- Complexity: Low

## Chosen Approach
**All three (A + B + C)** — layered implementation. Surgical fixes first for immediate ROI, then xdist for structural scalability, then differential pre-push for the long tail.

## Key Context Discovered During Shaping
- DynamoDB moto fixtures: all 55 use identical schema (pk/sk composite, PAY_PER_REQUEST), all function-scoped, no inter-test state dependencies — safe to hoist to module scope
- `manager.run()` at `bases/filescience/discover/manager.py:219` has `await anyio.sleep(1)` that loops ~6 real seconds in test_telemetry
- `session_start.py:150-159` spawns two unmocked `git config` subprocesses per test (36 total across 18 tests)
- pytest 9.0.2, moto 5.1.20, pytest-asyncio 1.3.0 — all compatible with xdist ≥ 3.6.0
- 10 CPU cores available locally
- import-linter contracts in pyproject.toml encode the full Polylith dependency graph

## Next Step
- [Plan] → `/create_plan memory-bank/thoughts/shared/briefs/2026-03-13-test-speed-optimization.md`
