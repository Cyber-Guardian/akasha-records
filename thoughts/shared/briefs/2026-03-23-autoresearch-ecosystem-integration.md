---
type: event
status: active
created: 2026-03-23
---
# Idea Brief: Autoresearch Ecosystem Integration

**Date:** 2026-03-23
**Status:** Shaped → Planning

## Problem
The harness makes decisions at 4 levels — parent selection, prompt evolution, stagnation response, knowledge management — each using a heuristic that ecosystem research has shown can be replaced with a principled mechanism. The weaknesses compound: suboptimal parents → wasted experiments → blind prompt mutations → slow recovery from stagnation → flat knowledge that doesn't prevent re-exploration of dead ends.

## Constraints
- `weighted_fitness_sample()` is self-contained — clean replacement target
- `_score_experiment()` returns string only — near-miss needs a side-channel write
- `mutate_variant()` receives `list[str]` only — reflective mutation widens this
- `_maybe_run_sweep()` is sole stagnation handler — escalation ladder is a parallel hook
- `build_thinker_prompt()` already ~12K — new sections must be compact
- No backward compat concern — can break existing state files

## Options Considered

### Vertical Slices
Ship one batch at a time, each self-contained and measurable. CMP first (highest value), then safety, knowledge vault, prompts last.
- Gains: Each batch testable in isolation. Can measure CMP impact before adding complexity.
- Costs: Slower total delivery. Some cross-batch synergies delayed.
- Complexity: Low per-batch, medium total

### Horizontal Layer
All 7 ideas in one integrated system. Designed together, shipped as 2-3 PRs.
- Gains: Ideas compound when designed together.
- Costs: Bigger blast radius. Harder to attribute which change helped.
- Complexity: High

### Core + Extensions
Ship tree upgrade, design protocols for the rest. Deferred ideas slot into same interfaces.
- Gains: Gets highest-value change fast. Extension interfaces for future.
- Costs: Risk of over-abstracting for 1-2 implementations.
- Complexity: Medium-high upfront

## Chosen Approach
**Vertical Slices** — ship one batch at a time, measure before proceeding. Each batch is a PR with its own tests. Knowledge base redesigned as campaign-local Obsidian vault (not JSON schema upgrade).

## Batches (in order)

### Batch 1: Solution Tree — CMP + Thompson + Near-Misses
Replace `weighted_fitness_sample()` with CMP-based Thompson sampling. Track near-misses in `CampaignState`. Add near-miss section to thinker prompt.
- **Sources:** HGM (ICLR 2026 Oral), n-autoresearch
- **Extension point:** `solution_tree.py:296-351`, `coordinator.py:1753`, `thinker.py:112`
- **Key mechanism:** Score nodes by descendant success rate (Beta distribution), Thompson sample. Near-misses = experiments within threshold of champion.

### Batch 2: Safety — Escalation Ladder + Gain Monitoring
Replace binary `consecutive_discards → sweep` with graduated REFINE → PIVOT → search. Add gain rate acceleration detection as a gaming signal.
- **Sources:** codex-autoresearch, tennis XGBoost case study
- **Extension point:** `coordinator.py:1784` (`_maybe_run_sweep`), `coordinator.py:2291` (iteration hook)
- **Key mechanism:** 3 discards → REFINE (adjust params), 5 → PIVOT (abandon strategy), 2 PIVOTs → web search. Gain acceleration = convex score curve alert.

### Batch 3: Knowledge Vault — Campaign-Local Obsidian Vault
Replace `knowledge.json` with campaign-local markdown vault. Dead-end registry, strategy families, structured lesson entries with frontmatter. Cross-campaign read access for researcher/thinker.
- **Sources:** autocontext (dead-ends, staleness), codex-autoresearch (strategy families, outcomes), Akasha records concept
- **Extension point:** `knowledge.py` (full rewrite), `thinker.py:112` (new context sections)
- **Key mechanism:** `workspace/.autoresearch/knowledge/` vault with: `dead-ends.md`, strategy family files, individual lesson entries with frontmatter (outcome, family, confidence, generation). Other campaigns' vaults readable via path.

### Batch 4: Prompt Evolution — Reflective Mutation
Replace blind Haiku mutation with trace-informed reflective mutation. Feed execution traces (tool calls, sandbox output, eval logs) as Actionable Side Information to the mutation LLM.
- **Sources:** GEPA (ICLR 2026 Oral)
- **Extension point:** `prompt_pool.py:277` (`mutate_variant`), `coordinator.py:2054` (`_maybe_evolve_prompts`)
- **Key mechanism:** Extract researcher execution trace → structured reflection dataset → mutation LM diagnoses specific failures → targeted prompt fix. Can also pull from knowledge vault (dead-ends, strategy families).

### Deferred (shaped, not planned)

**Per-instance Pareto frontier** (GEPA) — Track which experiments are best on which eval subsets. Requires multi-subset eval scoring we don't have yet.

**Progressive BFTS stages** (AI-Scientist-v2) — Break experiments into stages: draft → tune → creative → ablation. Major rearchitecture; progressive effort scaling approximates this.

**Structural merge operator** (GEPA) — Merge configs from two Pareto-optimal experiments structurally. Needs Pareto frontier first.

**Multi-objective utility** (SICA) — `U = 0.5*score + 0.25*(1-cost) + 0.25*(1-time)`. Not needed while Sonnet researchers are cheap.

## Key Context Discovered During Shaping
- `SolutionNode.children` already exists — descendant traversal for CMP is trivial
- `_score_experiment()` uses strict inequality — equal-to-champion scores as discard, not near-miss
- `_maybe_run_sweep` resets `consecutive_discards=0` after sweep regardless of outcome — prevents re-trigger but loses signal
- `mutate_variant()` prompt is 5 simple rules — room to add trace context without major restructure
- `build_thinker_prompt()` sections are ordered: toolbox → champion → tree → history → personalities → AIDE → parallel → sweep → prior knowledge → plan size → curation
- HGM uses pseudo-descendant regularization for new nodes with few evals — needed for our early phase (<5 nodes)
- Tennis XGBoost documented exact 4-phase gaming progression: honest → gray-zone → name-gaming → eval manipulation

## Research
[[2026-03-23-awesome-autoresearch-ecosystem|Awesome Autoresearch Ecosystem Research]]

## Next Step
Plan → `/create_plan memory-bank/thoughts/shared/briefs/2026-03-23-autoresearch-ecosystem-integration.md`
