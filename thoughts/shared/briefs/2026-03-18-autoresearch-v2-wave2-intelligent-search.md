---
type: event
created: 2026-03-18
status: active
---
# Idea Brief: Autoresearch V2 Wave 2 — Intelligent Search

**Date:** 2026-03-18
**Status:** Shaped → Planning

## Problem

The autoresearch coordinator runs a sequential loop where every experiment builds on the latest champion. Discarded experiments are thrown away, stagnation is tracked globally, and the system cannot backtrack to promising earlier nodes. The dedup-compression campaign stagnated for 13/26 experiments — once the champion plateaued, the system kept grinding forward from the same stuck point. The architecture survey shows tree search (AIDE, AI Scientist v2, AlphaEvolve) delivers 4x improvement over linear agents by allowing backtracking, branch-level stagnation detection, and structured node types.

The data model already has a latent tree via `parent_experiment_id`, but the coordinator never traverses it for branching decisions.

## Context: Three-Wave V2 Reframe

The original V2 brief proposed two waves. Based on the architecture survey, V2 is now three waves:

- **Wave 1 (Remove Constraints)** — gate removal, evaluator isolation, harness versioning, researcher prompt update. Already in progress. Fixes 54% experiment waste.
- **Wave 2 (Intelligent Search)** — THIS BRIEF. Solution tree, weighted fitness sampling, per-branch debug depth, draft/debug/improve taxonomy, tree-aware summarization. Fixes stagnation problem.
- **Wave 3 (Leader Evolution)** — leader harness design, human approval gate, harness evolution loop. Unlocks evaluator evolution. (Previously Wave 2.)

## Constraints

- Wave 1 must ship first — tree search composes on top of gate removal and evaluator isolation
- JSONL log format and `ExperimentRecord` schema must stay backward-compatible (append new fields, don't break existing)
- `parent_experiment_id` linkage already exists — build on it, don't replace
- Git worktree isolation model is sound and stays
- Budget enforcement (cost, time, experiment count) stays as-is
- Weighted fitness sampling is the right policy at our scale (50-100 experiments per campaign) — MAP-Elites/UCB only at 500+
- The "champion" concept needs to evolve: from "latest keep on linear branch" to "best-scoring node in the tree"

## Options Considered

### Incremental Grafting
Add tree search items as extensions to existing Wave 1.
- Gains: Least disruption to existing brief
- Costs: Muddles Wave 1 with two different concerns
- Complexity: Low

### Three-Wave Reframe (chosen)
Clean separation — each wave has one value proposition.
- Gains: Each wave independently shippable, clean concerns
- Costs: Three waves looks bigger
- Complexity: Medium

### Solution Tree First
Build tree search before gate removal.
- Gains: Highest-leverage change first
- Costs: Front-loads the hardest change, delays immediate 54% waste fix
- Complexity: High

## Chosen Approach

**Three-Wave Reframe** — Wave 2 delivers intelligent search as a standalone value increment after Wave 1 removes constraints.

## Key Context Discovered During Shaping

- `ExperimentRecord.parent_experiment_id` + `_resolve_parent_experiment_id()` at `coordinator.py:142` form a latent tree — nodes linked by ID prefix in JSONL, but no in-memory tree built (`coordinator.py:137-177`)
- `create_worktree()` at `git_ops.py:44` always forks from `HEAD` — needs to accept arbitrary commit SHA for backtracking
- `accept_champion()` at `git_ops.py:107` cherry-picks onto linear campaign branch — tree search changes what "champion" means (best node, not linear HEAD)
- `ExperimentLog.summarize()` at `state.py:49` operates on recency window of 20 records — tree-aware Σ(T) should mine ancestors of target node
- `consecutive_discards` at `state.py:224` is global — per-branch depth requires tree traversal
- `keep/crash/discard/rejected` statuses at `models.py:31` map to AIDE's draft/debug/improve node types
- Weighted fitness sampling (score-proportional, offspring-count penalty) beats greedy and random for 50-200 experiments (ShinkaEvolve, arXiv 2509.19349)
- AIDE's `num_drafts` parameter controls exploration breadth — initial drafts provide diversity, best-first provides exploitation
- Novelty archive + offspring penalty in weighted sampling provides sufficient diversity pressure at our scale

## Research Foundation

- [[2026-03-18-deep-autoresearch-architectures-survey|Architecture Survey]] — full findings
- [[2026-03-18-autoresearch-v2-platform|V2 Platform Brief]] — parent brief (waves 1 and 3)
- [AIDE (WecoAI/aideml)](https://github.com/WecoAI/aideml) — reference implementation for tree search in code space
- [ShinkaEvolve (arXiv 2509.19349)](https://arxiv.org/abs/2509.19349) — weighted fitness sampling validation

## Next Step

Plan → `/create_plan memory-bank/thoughts/shared/briefs/2026-03-18-autoresearch-v2-wave2-intelligent-search.md`
