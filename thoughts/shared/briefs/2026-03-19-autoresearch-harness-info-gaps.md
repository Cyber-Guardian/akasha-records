---
type: event
created: 2026-03-19
status: active
---
# Idea Brief: Autoresearch Harness Information Gaps

**Date:** 2026-03-19
**Status:** Shaped -> Planning

## Problem
The autoresearch harness wastes experiments due to three information gaps: researchers lose pip-installed packages between fresh containers, don't see parent performance data (throughput, memory, dedup ratio), and chase a misleading 62.4% redundancy estimate that's actually a block-level hash collision rate (real dedup is ~9.5%). In a 7-experiment campaign, at least 2 experiments ($1.05) were directly degraded by these gaps.

## Constraints
- Dockerfile.sandbox is intentionally minimal — should stay generic across campaigns
- Researcher prompt is built in `_build_campaign_context()` (coordinator.py:1031-1047) — only score + hypothesis today
- Profiler runs once at campaign init and result is baked into dataset_profile.json
- All fixes are independent — no ordering dependency

## Options Considered

### A: Researcher-Controlled Dependencies (via requirements.txt)
Instead of hardcoding packages in Dockerfile.sandbox, let the sandbox bootstrap from a `requirements.txt` in the experiment workspace. The researcher can add deps to this file, and they persist across experiments (committed to the experiment branch). The container startup or pre-experiment step installs from it.
- Gains: Deps evolve with the project. No Dockerfile rebuilds. Researcher has agency.
- Costs: ~5-10s startup overhead per experiment for pip install. Need to wire up the install step.
- Complexity: Low-Medium

### B: Researcher Performance Context
Add parent experiment's eval_metadata to `_build_campaign_context()` so the researcher sees throughput, memory, dedup_ratio, and duration. Prevents blind regressions.
- Gains: Researcher can estimate headroom before making changes.
- Costs: Slightly longer prompt (~5 lines). Marginal token cost.
- Complexity: Low

### C1: Caveat the Redundancy Estimate
Add a note in `DatasetProfile.to_summary()` explaining that "estimated redundancy" is block-level collision rate, not achievable dedup savings. Actual file-level dedup should be referenced instead.
- Gains: Researchers stop targeting unreachable 62.4% dedup.
- Costs: Trivial string change.
- Complexity: Low

## Chosen Approach
**A + B + C1** — researcher-controlled deps, performance context in researcher prompt, redundancy caveat in profiler output.

## Key Context Discovered During Shaping
- `Dockerfile.sandbox` only installs pytest + numpy (line 8)
- `_build_campaign_context()` at coordinator.py:1031-1047 only includes score + hypothesis[:80] — no eval_metadata
- `_estimate_redundancy()` at profiler.py:168-191 uses 4KB block MD5 hashing across a 200-file sample — conflates within-file repetition with cross-file dedup
- The leader prompt DOES include eval_metadata (state.py:111-123) — only the researcher is missing it
- Experiment #7 regressed because it replaced xxhash with SHA-256 in a fresh container where xxhash wasn't installed
- Champion at zstd-19 runs in 96s/120s timeout — only 24s headroom, which the researcher didn't know

## Next Step
- [Plan] -> `/create_plan memory-bank/thoughts/shared/briefs/2026-03-19-autoresearch-harness-info-gaps.md`
