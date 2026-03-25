---
type: event
created: 2026-03-18
status: active
---
# Idea Brief: Autoresearch V2 — Platform Architecture

**Date:** 2026-03-18
**Status:** Shaped → Planning

## Problem

The autoresearch harness treats experiments as single-file patches against a rigid evaluator interface. This causes 54% experiment waste (19% rejected for writing extra files, 35% crashing from interface violations) and a hard strategy ceiling once single-file optimization levers are exhausted. More fundamentally, the harness conflates the experiment lifecycle (platform concern) with the experiment semantics (lab concern) — the evaluator interface, project structure, metrics, and entry facade are all hardcoded in the harness rather than being designable by the lab leader.

## Constraints

- The git sandbox model (worktrees, cherry-pick, champion lineage) is sound and should be preserved
- Budget enforcement, cost tracking, and orchestration are platform concerns that stay
- The lab leader agent becomes significantly more capable — it designs infrastructure, not just hypotheses
- Harness changes that affect scoring must be human-gated to prevent metric gaming
- Backward compatibility with existing campaign data is desirable (JSONL format, state.json)
- The researcher agent must not have Bash access (security boundary stays)

## Options Considered

### Incremental Gate Removal
Remove the three file-write gates, move evaluator to workspace, keep everything else.
- Gains: Simplest, mostly code deletion, fixes 54% waste
- Costs: Doesn't address rigid evaluator interface, no harness evolution, no metric versioning
- Complexity: Low

### Project Sandbox with Evaluator Isolation
Remove all gates, run evaluator from workspace with absolute path. Researcher gets full project access.
- Gains: Clean separation of mutable/immutable. Fixes waste, unblocks multi-file
- Costs: Evaluator interface still rigid. Leader still just picks hypotheses
- Complexity: Low-Medium

### Two-Wave Platform Rebuild (chosen)
Rebuild autoresearch as a platform in two waves. Wave 1: platform foundation (gates, versioning, rescore). Wave 2: leader evolution (designs harness, human-gated changes).
- Gains: Addresses all issues incrementally. Each wave independently valuable
- Costs: Significant rearchitecture across two waves
- Complexity: Medium-High

## Chosen Approach

**Two-Wave Platform Rebuild** — separates platform concerns from lab concerns with versioned harnesses and a more capable lab leader. Each wave is independently valuable and shippable.

## Grounding Results

Six claims were validated against codebase, memory-bank, and industry sources:

| Claim | Verdict | Key Evidence |
|-------|---------|-------------|
| Decoupled harness versioning | **Holds** | lakeFS pattern (git for data) applied to evaluation code. `evaluator_dir_hash` in `CampaignState` is a primitive form. No existing system does this for agent R&D specifically. |
| Contract-as-code adapters | **Partly holds (YAGNI for now)** | Pattern exists in MLOps (MLflow MLmodel format, database migrations). If entry contract stays minimal ("stdout JSON with metric_value"), adapters aren't needed initially. |
| Lab-to-production pipeline | **Axed** | Feasible but speculative. Depends on lab producing production-quality champions. |
| Campaign-to-Linear integration | **Holds** | Natural extension of existing `/create-linear-issue` + `/resolve-linear-issue` primitives. |
| Leader designs experiment infrastructure | **Partly holds — highest risk/reward** | No existing system does this. AI Scientist v2 works within a fixed BFTS framework. Anthropic's multi-agent research uses human-designed infrastructure. Capability is plausible but unvalidated. |
| Human-gated harness changes | **Holds** | Dominant 2026 agentic pattern (Block testing pyramid, MLflow model registry, Copilot/Devin PR flow). |

Sources: [[2026-03-18-autoresearch-testing-pyramid|Testing Pyramid Plan]], [Block Engineering Testing Pyramid](https://engineering.block.xyz/blog/testing-pyramid-for-ai-agents), [AI Scientist v2](https://github.com/SakanaAI/AI-Scientist-v2), [MLflow](https://mlflow.org/classical-ml/experiment-tracking), [lakeFS](https://lakefs.io/)

## Key Context Discovered During Shaping

- The git machinery is already multi-file capable — `commit_experiment` does `git add -A`, `accept_champion` cherry-picks full commits (`git_ops.py`)
- Cherry-pick conflicts are impossible in the sequential model — each worktree forks from the exact commit the campaign branch HEAD points to
- After first keep, `accept_champion` permanently moves workspace to the campaign branch — all subsequent worktrees fork from the latest champion
- `Bash` in `RESEARCHER_DISALLOWED_TOOLS` is the real security boundary, not file gates
- The evaluator uses `importlib.load("target.py")` — tight coupling is root cause of 35% crash rate
- Smoke eval is still valuable as a cheap pre-check in the new model
- Lab retrospective: 26 experiments, $13.27 cost, champion at 0.838, stagnated 13 experiments. Pragmatic personality: all 3 keeps. Academic: 0 keeps, 2x cost
- AI Scientist v2 validates "autonomous hypothesis + experiment" pattern but NOT "autonomous harness design"
- lakeFS git-for-data pattern is the strongest analogue for harness versioning
- No existing system versions the evaluation harness as a first-class artifact — this is architecturally novel

## Architecture

### Three-Layer Model

```
Platform Layer (lifecycle primitives — harness-agnostic)
  ├── Git sandbox: worktree create/commit/cherry-pick/cleanup
  ├── Champion lineage: keep/discard/crash, branch evolution
  ├── Budget enforcement: cost, time, experiment count, early stopping
  ├── Orchestration: leader → researcher dispatch loop
  ├── State persistence: JSONL log, campaign state, novelty archive
  ├── Harness versioning: tag harness commits, rescore, filter by version
  └── Human approval gate: for harness changes

Lab Leader Layer (experiment semantics — per-campaign)
  ├── Designs initial harness: entry facade, evaluator, metrics
  ├── Generates hypotheses for researchers
  ├── Reviews results, adjusts strategy
  ├── Proposes harness evolution (human-gated)
  └── Decides when harness needs to change (new metrics, better scoring)

Researcher Layer (implementation — per-experiment)
  └── Full project sandbox: read/write any file, no Bash
  └── Implements hypothesis within the project
  └── Builds on champion's codebase (worktree forks from latest champion)
```

### Harness Versioning Model

```
Campaign
  └── Harness v1 (entry facade, evaluator, metrics)
  │     └── Experiments scored against v1
  │     └── Champion within v1
  │
  └── Harness v2 (leader proposes change, human approves)
  │     └── Rescore: old champion against v2 → new baseline
  │     └── Experiments scored against v2
  │     └── Champion within v2
```

- Champion is per-harness-version
- Harness version = tagged commit of evaluation code
- Experiment records tagged with harness version for fair comparison
- Rescore = re-run champion's code against new harness version

### What Changes vs Current

| Concern | Current (v1) | New (v2) |
|---------|-------------|----------|
| File constraints | 3 gates enforcing target_files | None — full project sandbox |
| Evaluator | Hardcoded importlib interface | Leader-designed, versioned, evolvable |
| Entry facade | Implicit (evaluator imports target.py) | Explicit, leader-defined, part of harness |
| Metrics | Single metric_value float | Leader-defined, potentially multiple |
| Project structure | Single file (target.py) | Full project — researcher decides |
| Lab leader role | Pick hypothesis + personality | Architect harness + generate hypotheses + evolve evaluation |
| Harness changes | Manual, no versioning | Versioned, human-gated, rescorable |
| Scoring comparison | All experiments comparable | Comparable within harness version |

## Implementation Waves

### Wave 1: Platform Foundation
Make the current harness multi-file capable and add harness versioning. High confidence, mostly proven patterns.

- **1a. Remove file gates** — delete Gate A (PreToolUse hook), Gate B (check_unexpected_files), Gate C (validate_experiment target scoping). Keep smoke eval.
- **1b. Evaluator isolation** — run evaluator from workspace (absolute path), delete `_copy_evaluator_to_worktree`
- **1c. Harness version tagging** — tag evaluator state at campaign start, store `harness_version` in experiment records, filter/group by version
- **1d. Rescore capability** — re-run any experiment's code against any harness version
- **1e. Early stopping** — `max_consecutive_discards` budget parameter
- **1f. Campaign-to-Linear integration** — create Linear issue at campaign start, update with results
- **1g. Researcher prompt update** — "full project access" instead of "modify only target.py", include evaluator interface context

### Wave 2: Leader Evolution
Upgrade the lab leader from hypothesis picker to experiment architect.

- **2a. Leader harness design** — leader creates initial project structure, entry facade, evaluator, metrics definition
- **2b. Human approval gate** — harness changes pause for human review before accepting
- **2c. Harness evolution loop** — leader proposes evaluator/metrics changes mid-campaign → human approves → new harness version → rescore → new baseline
- **2d. Leader prompt expansion** — include campaign analytics, failure patterns, and harness design context in leader prompt

## Next Step

Decompose into Linear issues → `/create-linear-issue` for each work item
