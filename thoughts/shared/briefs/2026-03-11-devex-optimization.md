# Idea Brief: DevEx Optimization Beyond Testing & Agentic Harness

**Date:** 2026-03-11
**Status:** Shaped -> Planning

## Problem

The monorepo's DevEx infrastructure is mature (make onboard, ephemeral stack, 13 CI workflows, 3-layer quality gates, content-addressed deploys, local observability). But quantitative evidence from 153 sessions, 11,614 hook invocations, and memory-bank incident records reveals compounding friction across the inner loop, CI feedback, and local/CI parity. The highest-impact gaps are not missing features but mistuned existing systems.

**Evidence sources:** ruff_validator.log (997 Python file checks), ty_validator.log (996 checks), routing_decisions.jsonl (260 entries), scope_guard.log (11,614 invocations), 40+ session logs with meaningful content, memory-bank topic notes and incident research docs.

## Constraints

- 3-person team (Jordan + Niklas + Harsh) — improvements must be low-maintenance
- Agent-heavy workflow — changes must benefit both human and autonomous developer loops
- Blacksmith runners already in use — can leverage their cache infra
- Polylith namespace packages add complexity to tooling integration
- `tools/` workspace is architecturally separate from main workspace

## Options Considered

### A. Targeted Friction Reduction (tune existing systems)

Fix the specific pain points revealed by quantitative evidence: ruff hook cycle, CI caching, PR feedback, TaskCompleted gate, local/CI parity. No new systems — adjust what exists.

- **Gains:** Directly addresses measured friction; every change has a quantified baseline to measure improvement against
- **Costs:** Incremental; doesn't address structural gaps (tools/ CI, layer drift)
- **Complexity:** Low — each item is independent, can be done in any order

### B. Full DevEx Overhaul (A + structural fixes)

Everything in A plus: tools/ CI coverage, layer drift automation, watch mode, post-deploy smoke tests, documentation refresh, scripts discoverability.

- **Gains:** Comprehensive; closes both tactical friction and structural gaps
- **Costs:** Larger scope; some items (post-deploy smoke, tools/ CI) require design decisions
- **Complexity:** Medium — independent items but more surface area to maintain

### C. Platform-Level DevEx (B + new capabilities)

Everything in B plus: pytest-watcher for watch mode, grafana/otel-lgtm single container, pytest-split for parallel CI, CTRF test reporting, reviewdog for inline annotations.

- **Gains:** Step-change in developer velocity; aligns with 2025-2026 ecosystem best practices
- **Costs:** New dependencies to maintain; some tools (pytest-split, CTRF) need evaluation
- **Complexity:** Medium-High — new tooling has learning curve

## Chosen Approach

**Option B: Full DevEx Overhaul** — addresses both the quantified friction (Option A) and structural gaps, without taking on unproven new tooling (Option C). Individual items from C (watch mode, PR feedback) can be pulled in selectively.

## Key Context Discovered During Shaping

### Quantitative (from hook logs)
- **Ruff PostToolUse block rate: 42.4%** (423/997 Python writes). No mitigation since 2026-02-05. F401 (unused imports) is the dominant cause — the multi-step edit cycle forces block-fix-retry loops. (ruff_validator.log)
- **ty PostToolUse block rate: 26.3%** (262/996). `invalid-argument-type` accounts for 132/335 errors — mostly boto3/GlideClient types. (ty_validator.log)
- **import-linter block rate: 2.1%** (12/571). 10 of 12 blocks are false positives from `tools/` files hitting "no config found." Only 2 genuine architectural violations ever. (import_linter_validator.log)
- **CI has zero uv dependency caching** across all 13 workflows. `setup-uv@v5` used without `enable-cache` everywhere. Each `uv sync --dev` cold-installs ~60 packages.
- **Routing null-match rate: 28.8%** (75/260) — 1 in 4 prompts don't match any skill route.

### Qualitative (from session logs + memory-bank)
- **Hooks block on pre-existing issues**, causing scope creep — documented in 2026-02-05 hooks research
- **"Lasagna mocking" caused 4 sequential production failures** — every layer mocked independently, all failures at seams between layers (memory-bank research 2026-03-05)
- **Deploy gate has been fixed** — `tools/projects/triage-agent/Makefile:44` now reads `deploy: test build sync-memory`
- **OTel containers run alongside ephemeral stack** (docker-compose include) but no handler calls `setup_telemetry()` — `OTEL_ENABLED` not set in `.env.ephemeral.defaults`
- **3 of 9 CI PR checks have fully automatic local triggers** (ruff, import-linter, detect-secrets). pip-audit, extensions, scope-check, semantic-pr, diff-cover have no local equivalent.
- **Lambda layer boto3 1.37.3 vs pyproject.toml >=1.40.0** — intentional (aiobotocore constraint) but creates testing-vs-runtime divergence with no automated sync check
- **`tools/` workspace has zero CI test coverage** — no workflow runs `tools/test/`
- **TaskCompleted gate runs `make test` (full suite)**, not `test-fast` — stricter than the human pre-push gate
- **Poison record logging is fixed** (handler.py:438-453 has `logger.exception`) but topic notes are stale
- **`/investigate` vs `/debug` naming mismatch** across CLAUDE.md, skill-routes.json, and .claude/commands/

### External Research (2025-2026 ecosystem)
- **pytest-watcher** (Rust-backed watchfiles) is the maintained replacement for pytest-watch
- **`setup-uv@v5` with `enable-cache: true` keyed on `uv.lock`** is the consensus CI caching pattern
- **py-cov-action** (python-coverage-comment-action) provides PR coverage comments with zero external accounts
- **`ruff check --output-format=github`** provides free inline PR annotations
- **grafana/otel-lgtm** single container provides full observability stack in under 5 seconds
- **Discord's pytest daemon** achieved 10x test iteration speed via hot-reloading

## Next Step

- [x] **Plan** -> `/create_plan memory-bank/thoughts/shared/briefs/2026-03-11-devex-optimization.md`
