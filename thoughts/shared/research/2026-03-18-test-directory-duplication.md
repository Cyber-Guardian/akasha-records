---
type: event
created: 2026-03-18
status: active
topic: "Test directory structure and duplication between test/ and tools/test/"
tags: [research, testing, polylith, migration]
---
# Research: Test Directory Structure and Duplication

## Research Question
Why do two separate test roots exist (`test/` and `tools/test/`)? What's duplicated vs unique? Could they be consolidated?

## Summary

Two separate pytest projects exist because `filescience` (the main repo) and `filescience-tools` are **separate Python packages** with different dependency sets. The tools package depends on `pydantic-ai`, `claude-agent-sdk`, `slack-sdk`, `optuna`, `typer` — none of which are in the main package. This is the fundamental reason for the split: they have separate `pyproject.toml`, separate `uv.lock`, separate venvs.

## The Two Test Roots

### Main repo: `test/` (run via `make test`)
- **pytest config**: `pyproject.toml` → `pythonpath = [".", "components", "bases", "test"]`, `testpaths = ["test"]`
- **What it tests**: Production Polylith bricks — `components/filescience/` (cloud_api, dynamodb, throttling, models, etc.) and `bases/filescience/` (discover, entity_discovery_trigger, valkey_queue_processor)
- **Test count**: ~1077 tests
- **Dependencies**: boto3, moto, httpx, valkey-glide — production deps only
- **conftest.py**: xfail policy, Hypothesis profiles (dev/ci/nightly)

### Tools repo: `tools/test/` (run via `cd tools && make test`)
- **pytest config**: `tools/pyproject.toml` → `pythonpath = [".", "../components", "../bases", "test", "dev-harness/src"]`, `testpaths = ["test", "test/bases/autoresearch"]`
- **What it tests**: Tooling bricks — triage_agent, autoresearch, helm_cli, and tool components (tool_config, tool_slack_client, tool_durable_harness, tool_linear_client)
- **Test count**: ~750 tests
- **Dependencies**: pydantic-ai, claude-agent-sdk, slack-sdk, optuna, moto, hypothesis — AI/agent deps
- **conftest.py**: Hypothesis profiles, RBSM viz server, pre-push tracked-only filter

### Source code lives in Polylith (shared)
Both test roots import from the same `components/` and `bases/` directories. The tools pyproject achieves this via:
```toml
[tool.hatch.build]
dev-mode-dirs = [".", "../components", "../bases"]

[tool.pytest.ini_options]
pythonpath = [".", "../components", "../bases", "test", "dev-harness/src"]
```

## Duplicated Files

### `test/shared/recording_fakes.py` vs `tools/test/shared/recording_fakes.py`
**Status: Identical** (confirmed via diff). Contains `RecordingSlackClient` only.

### `test/shared/fixtures.py` vs `tools/test/shared/fixtures.py`
**Status: DIVERGENT.** Both contain `MockCheckpointContext`, `EventFactory`, moto fixtures. The tools/ version is a superset:
- `MockCheckpointContext`: tools/ version has `callback_replies` parameter, `step_names`/`callback_names` properties, typed signatures. test/ version is simpler (no call tracking, untyped).
- Same moto fixtures and `EventFactory` in both.

### `test/shared/conftest.py` vs `tools/test/shared/conftest.py`
**Status: Identical.** Both re-export from `test.shared.fixtures`.

### `test/shared/test_fixtures.py` vs `tools/test/shared/test_fixtures.py`
**Status: Identical.** Both test `RecordingSlackClient`, `EventFactory`, moto fixtures.

### `test/harness/` vs `tools/test/harness/`
Both exist. `test/harness/` has: `test_infra_validation.py`, `test_triage_harness.py`, `triage_harness_utils.py`. `tools/test/harness/` has: `test_adapter_registry.py`, `test_triage_harness.py`, `harness_utils.py`. Different files — no direct duplication but overlapping intent.

## Why They Can't Cross-Import

Each test root uses `from test.shared...` imports. Since `test` is in both pythonpaths but resolves to different directories (`./test/` for main, `tools/test/` for tools), the `test.shared` namespace is local to each pytest project. There's no cross-import path.

## What's Unique to Each

### Only in `test/`
- `test/components/filescience/` — all component tests (dynamodb, throttling, cloud_api, models, telemetry, lambda_utils, domain_backup)
- `test/bases/filescience/` — discover, entity_discovery_trigger, valkey_queue_processor tests
- `test/scripts/` — CI/CD script tests (affected_tests, generate_scope, scope_guard, validators, etc.)
- `test/e2e_concurrency_gates.py`

### Only in `tools/test/`
- `tools/test/bases/triage_agent/` — RBSM, handler, tools, memory_tools tests
- `tools/test/bases/autoresearch/` — coordinator, dispatch, evaluator, git_ops, etc.
- `tools/test/bases/helm_cli/` — CLI command tests
- `tools/test/components/` — config, durable_harness, linear_client, slack_client tests
- `tools/test/shared/rbsm_viz.py` — RBSM visualization (unique to tools)

## Could They Be Consolidated?

### Obstacle 1: Different dependency sets
The main package doesn't depend on `pydantic-ai`, `claude-agent-sdk`, `slack-sdk`. If tools tests were moved under `test/`, running `make test` would fail because those deps aren't installed in the main venv.

### Obstacle 2: Different venvs
`uv` manages two separate virtual environments (`.venv` at root, `.venv` at `tools/`). Tests must run in the venv that has their deps.

### Obstacle 3: CI runs them separately
`make test` and `cd tools && make test` are separate CI jobs with different dependency installs.

### What would need to change for consolidation
1. Merge the two `pyproject.toml` dependency sets (add AI/agent deps to main) — this bloats the main package
2. Or: keep separate venvs but use a single test directory with markers to select which tests run in which venv — complex pytest configuration
3. Or: make `test/shared/` a real package that both install from (e.g., a Polylith component) — over-engineering

## Practical Implication for ENG-2593

The duplication is a **structural consequence** of the two-package architecture, not a bug to fix. The pragmatic approach is:
- Accept that `test/shared/` and `tools/test/shared/` are separate namespaces
- Keep them in sync manually (or add a CI diff check)
- When upgrading `RecordingSlackClient`, update both copies
- Promote the tools/ `MockCheckpointContext` to both copies to fix the divergence

## Code References
- Main pytest config: `pyproject.toml:186-187`
- Tools pytest config: `tools/pyproject.toml:42-51`
- Tools dev-mode-dirs: `tools/pyproject.toml:37`
- Main conftest: `test/conftest.py`
- Tools conftest: `tools/test/conftest.py`
- Stale MockCheckpointContext: `test/shared/fixtures.py:19-41`
- Better MockCheckpointContext: `tools/test/shared/fixtures.py:19-56`
