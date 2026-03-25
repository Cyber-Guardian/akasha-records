---
type: event
created: 2026-03-18
status: active
date: 2026-03-18T22:00:00-04:00
researcher: Claude
git_commit: 45a6d4fed2eed607d91c0e080a38e28824d1a531
branch: main
repository: filescience
topic: "Autoresearch architectures — survey of existing automated research systems, design patterns, and application to our system"
tags: [deep-research, autoresearch, architecture, tree-search, evolutionary-search, multi-agent]
research_depth: standard
iterations_completed: 3
last_updated: 2026-03-18
last_updated_by: Claude
---

# Deep Research: Autoresearch Architectures Survey

## Research Question
Survey existing automated research systems, their design patterns, orchestration strategies, quality mechanisms, and how learnings could be applied to our autoresearch system.

## Summary
The field has converged on **tree search** as the dominant orchestration pattern for autonomous research, replacing flat sequential loops. Three major variants exist: greedy best-first tree search (AI Scientist v2 / AIDE), population-based evolutionary archives (AlphaEvolve / FunSearch), and bandit-based selection. Our system already has a latent tree structure via `parent_experiment_id` but never traverses it — the highest-leverage upgrade is to build an in-memory solution tree with weighted fitness sampling for parent selection, which ShinkaEvolve shows beats both greedy and random approaches for 50-100 experiment budgets. No existing system has cross-campaign knowledge transfer, making our meta-analyst work ahead of the field. Multi-model ensembles (cheap model for leader, capable model for researcher) can cut costs 57-80%.

## Perspectives Explored
1. **Orchestration & Lifecycle** — Revealed tree search as the universal pattern, with AIDE's draft/debug/improve taxonomy and AlphaEvolve's async evolutionary loop as the two poles
2. **Quality & Evaluation** — Split evaluator architecture, cascaded early elimination, circuit-breaker stagnation detection
3. **Search Strategy & Exploration** — Weighted fitness sampling optimal at our scale; MAP-Elites/UCB only at 500+ experiments
4. **Sandboxing & Code Generation** — Git worktrees are a valid and emerging pattern; AIDE's structural isolation is the main alternative
5. **Meta-Learning & Self-Improvement** — A gap in the field; no system does cross-campaign knowledge transfer

## Detailed Findings

### 1. Orchestration & Lifecycle

#### The Systems Landscape

| System | Org | Architecture | Key Innovation |
|--------|-----|-------------|----------------|
| **AI Scientist v2** | Sakana AI | Agentic tree search (BFTS) with Experiment Progress Manager | End-to-end paper generation with VLM critique |
| **Karpathy autoresearch** | Independent | Minimal single-agent loop (630 lines) | Simplicity — program.md + train.py, ~50 experiments/night |
| **AIDE** | Weco AI | Tree search in code space | Draft/debug/improve node types, 4x medals vs linear agents |
| **AlphaEvolve** | DeepMind | Async evolutionary loop with MAP-Elites archive | Multi-model ensemble, parent+inspiration sampling, multi-file |
| **FunSearch** | DeepMind | Evolutionary island model | Parallel islands prevent premature convergence |
| **SWE-agent** | Princeton | Single-agent sandboxed loop | Docker + gVisor isolation, test suite evaluation |
| **OpenHands** | UIUC | Multi-agent (LangGraph) sandboxed loop | Leader-worker via runtime delegation |
| **MLAgentBench** | Princeton | ReAct-based agent harness | Standardized ML task benchmarking |

#### AIDE — The Most Relevant Architecture

AIDE treats ML engineering as tree search over complete scripts. Key design choices:

- **Node types:** Draft (new approach from scratch), Debug (repair buggy parent from error logs), Improve (one atomic change to valid parent)
- **Selection policy:** Three-phase heuristic — fill `num_drafts` initial drafts → debug buggy leaves with probability `debug_prob` (below cap) → greedy best-first expansion via `journal.get_best_node()`
- **No explicit diversity mechanism** — relies on drafts for diversity, best-first for exploitation
- **Summarization operator Σ(T):** Mines the full tree for debugging hints that inform repair attempts — richer than a flat history
- **Results:** On MLE-Bench (75 Kaggle competitions), AIDE+o1-preview: 59.1% above-median (vs 13.6% solo), 36.4% medals (vs 7.6%), 4x more medals than best linear agent (OpenHands)

#### AlphaEvolve — The Evolutionary Alternative

- **Async propose-evaluate-select loop** — not tree-structured but population-based
- **Multi-model ensemble:** Gemini Flash for high-volume generation, Gemini Pro for depth/quality. Split is cost/latency-driven
- **MAP-Elites archive:** Programs stored across a feature grid (not just fitness leaderboard). Samples both "parent" (direct improvement basis) and "inspiration" (diversity injection)
- **Multi-file:** Evolves entire codebases (C++, Python, Verilog) — not limited to single-function templates like FunSearch
- **Open-source:** OpenEvolve implementation available

#### How Our System Compares

Our coordinator (`coordinator.py`) runs a sequential loop: leader → researcher → eval → score. The `ExperimentRecord.parent_experiment_id` field and `_resolve_parent_experiment_id()` form a **latent tree structure** in the data model — nodes are linked by ID prefix in the JSONL log, but no in-memory tree is maintained and the coordinator never traverses it for branching decisions.

**The minimal retrofit to tree search:**
1. Stop discarding losing branches — maintain a scored node registry
2. Leader selects a parent node by **weighted fitness sampling** (score-proportional, penalized by offspring count) rather than always picking the latest champion
3. "Draft" = generate novel approach (parent = root); "Improve" = patch selected parent's code
4. Backtracking in git = `git checkout <ancestor-sha>` + new worktree branch (no cherry-pick needed — the node's full state IS the commit)

### 2. Quality & Evaluation

#### Split Evaluator Architecture
AI Scientist v2 uses a **separate feedback LLM** (GPT-4o at lower temperature) to score node outputs independently from the coding LLM. This decouples scoring from generation and allows the scorer to evolve without touching experiment code. This maps to our system as: use a dedicated evaluation model distinct from the researcher model, potentially at lower temperature for consistent scoring.

#### Stateless Evaluation Interface
AIDE's evaluator returns `{is_bug, summary, metric, lower_is_better}` — model-agnostic, with a separate `agent.feedback.model` LLM judging outputs. Our `EvaluatorResult(metric_value, metadata, duration_seconds)` is structurally similar but lacks an explicit `is_bug` boolean — crashes are detected via subprocess return code and `ValidationResult.passed`.

#### Stagnation Detection Patterns
- **AI Scientist v2:** Staged iteration caps (`stage1_max_iters` through `stage4_max_iters`) + `max_debug_depth=3` per branch
- **AIDE:** Per-node debug retry cap (distinct from global stagnation)
- **AlphaEvolve:** Cascaded multi-objective evaluation with early elimination
- **Cross-cutting pattern:** Circuit-breaker style — hard caps on debug depth, iteration count, cost, or wall time
- **Our system:** Global `consecutive_discards` / `stale_threshold` — needs to become **per-branch depth tracking** to match AIDE's granularity

#### Evaluation Versioning
- **lakeFS:** Git-for-data — branches isolate evaluation dataset state per experiment run, commit IDs logged as MLflow inputs
- **W&B:** DAG-tracked artifact promotion with CI-enforced metric thresholds
- **Our system:** `evaluator_dir_hash` in `CampaignState` is a primitive form. The V2 brief's harness versioning model (tag evaluator commits, rescore old champions) is well-grounded — lakeFS pattern validates the approach

### 3. Search Strategy & Exploration

#### Selection Policy Comparison at Different Scales

| Scale | Best Policy | System Example | Why |
|-------|------------|----------------|-----|
| 50-100 experiments | **Weighted fitness sampling** | ShinkaEvolve | Low overhead, sustains improvement where greedy plateaus |
| 100-500 experiments | UCB / Thompson sampling | I-MCTS, bandit frameworks | Exploration bonus becomes valuable |
| 500+ experiments | MAP-Elites / quality-diversity | AlphaEvolve, Monte-Carlo-Elites | Diversity grid prevents convergence |

**For our system (~50-100 experiments per campaign):** Weighted fitness sampling is the right choice. It's score-proportional with offspring-count penalty, giving natural exploration without explicit tuning. Near-zero implementation overhead — just change how the leader selects which experiment to build on.

#### Novelty and Diversity Mechanisms
- **Our novelty archive:** Cosine similarity threshold on hypothesis text — good for deduplication but doesn't incentivize diversity
- **AIDE:** No explicit diversity mechanism — relies on initial drafts
- **AlphaEvolve:** MAP-Elites feature grid + "inspiration" sampling = structural diversity
- **FunSearch:** Island model = population isolation prevents convergence
- **Recommendation:** Our novelty archive + weighted fitness sampling (offspring penalty = diversity pressure) is sufficient at our scale. MAP-Elites feature grid would be overkill until we reach 500+ experiments, but our `dimensions` config is already a primitive feature grid

### 4. Sandboxing & Code Generation

| System | Isolation | Code Model | Accumulation |
|--------|-----------|------------|-------------|
| AI Scientist v2 | Operator-provided Docker | Diffs/patches | Within tree branch |
| AIDE | Structural (independent scripts) | Complete scripts per node | None — stateless |
| SWE-agent | Docker + gVisor | Diffs/patches | Within session |
| OpenHands | Docker | Full file writes | Within session |
| AlphaEvolve | Cluster execution | Diffs to codebase | Population archive |
| **Our system** | Git worktrees | Full agent (Read/Write/Edit) | Champion lineage (cherry-pick) |

**Our git worktree model is well-positioned.** It provides:
- Strong isolation (separate directory per experiment)
- Full state capture (commit = complete snapshot)
- Natural backtracking (checkout any ancestor)
- No Docker overhead

The main gap vs. AIDE's structural isolation: AIDE nodes are stateless (each is a complete script), while our worktrees accumulate state from the champion branch. Moving to tree search where any node can be a parent would address this — each worktree forks from the selected parent commit rather than always from the champion HEAD.

### 5. Meta-Learning & Self-Improvement

**This is the biggest gap in the field.** No existing system has:
- Cross-campaign knowledge transfer
- Automated retrospectives ("hypotheses about X always fail")
- Persistent experience databases or knowledge graphs across runs

The closest approaches:
- **AlphaEvolve:** Meta-prompting where instructions co-evolve alongside solutions (within-campaign only)
- **FunSearch/AlphaEvolve:** Program database as evolutionary memory — high-scoring solutions sampled as few-shot context
- **ELL framework** (arXiv 2508.19005): Formalizes long-term memory and skill abstraction from trajectories, but is theoretical
- **Self-Evolving Agents survey:** Documents the gap — no implemented system combines all of: experience memory, skill abstraction, and cross-run knowledge

**Our meta-analyst + knowledge base (`knowledge.py`) already addresses this gap.** The knowledge base accumulates findings across experiments within a campaign, and the meta-analyst (planned) would analyze patterns across campaigns. This positions our system ahead of published work.

### Cross-cutting: Multi-Model Ensemble Opportunity

| Role | Current Model | Recommended | Cost Impact |
|------|--------------|-------------|-------------|
| Lab leader (hypothesis gen) | Sonnet | **Haiku** | ~80% leader cost reduction |
| Researcher (code impl) | Sonnet | Sonnet (stay) | No change — needs coding capability |
| Evaluator feedback | N/A (subprocess) | **Separate model at lower temp** | New cost, but improves scoring consistency |

A nine-agent pipeline study demonstrated **57% cost reduction** by assigning cheaper models to extraction/transformation steps and capable models to generation/reasoning. AlphaEvolve's Flash+Pro split validates this pattern at scale. AI Scientist v2's use of GPT-4o specifically as reviewer (avoiding positivity bias from the generation model) suggests the evaluator feedback model should be a *different family* than the researcher, or at minimum a lower temperature.

## Key Sources

### Codebase Files
- `bases/filescience/tool_autoresearch/coordinator.py:137` — `_resolve_parent_experiment_id()`, latent tree structure
- `bases/filescience/tool_autoresearch/state.py:49` — `ExperimentLog.summarize()`, Σ(T) analogue
- `bases/filescience/tool_autoresearch/models.py:137` — `ExperimentRecord` with `parent_experiment_id`
- `bases/filescience/tool_autoresearch/evaluator.py:26` — `run_evaluator()` interface
- `bases/filescience/tool_autoresearch/models.py:58` — `BudgetConfig.stale_threshold`

### Memory-Bank Documents
- [[2026-03-18-autoresearch-v2-platform|Autoresearch V2 Platform Brief]]
- [[2026-03-16-autoresearch-research-quality|Autoresearch Research Quality Plan]]
- [[2026-03-14-autonomous-experiment-harness|Autonomous Experiment Harness Plan]]
- [[2026-03-17-autoresearch-meta-analyst|Meta-Analyst Plan]]
- [[2026-03-01-multi-agent-architecture-research|Multi-Agent Architecture Research]]

### External — Core Systems
- [AI Scientist v2 paper (arXiv 2504.08066)](https://arxiv.org/abs/2504.08066) — BFTS, split evaluator, staged caps
- [AI Scientist v2 GitHub](https://github.com/SakanaAI/AI-Scientist-v2) — reference implementation
- [AIDE paper (arXiv 2502.13138)](https://arxiv.org/html/2502.13138v1) — tree search in code space
- [AIDE GitHub (WecoAI/aideml)](https://github.com/WecoAI/aideml) — open-source implementation
- [AlphaEvolve paper (arXiv 2506.13131)](https://arxiv.org/abs/2506.13131) — evolutionary coding agent
- [AlphaEvolve technical paper (DeepMind)](https://storage.googleapis.com/deepmind-media/DeepMind.com/Blog/alphaevolve-a-gemini-powered-coding-agent-for-designing-advanced-algorithms/AlphaEvolve.pdf)
- [FunSearch (Nature)](https://www.nature.com/articles/s41586-023-06924-6) — evolutionary island model
- [Karpathy autoresearch](https://github.com/karpathy/autoresearch) — minimal single-agent loop

### External — Techniques & Patterns
- [ShinkaEvolve (arXiv 2509.19349)](https://arxiv.org/abs/2509.19349) — weighted fitness sampling beats greedy at 50-200 samples
- [I-MCTS (arXiv 2502.14693)](https://arxiv.org/html/2502.14693v4) — introspective MCTS for AutoML
- [OpenEvolve (HuggingFace)](https://huggingface.co/blog/codelion/openevolve) — open-source AlphaEvolve
- [ELL framework (arXiv 2508.19005)](https://arxiv.org/abs/2508.19005) — cross-run knowledge accumulation theory
- [Self-Evolving Agents survey](https://syhya.github.io/posts/2026-02-20-self-evolving-agents/) — landscape overview
- [Multi-model routing cost analysis](https://www.infralovers.com/blog/2026-02-19-ki-agenten-modell-optimierung/) — 57% cost reduction with model tiering
- [lakeFS + MLflow data versioning](https://lakefs.io/blog/mlflow-data-versioning/) — evaluation dataset versioning
- [Git worktrees for AI agents (Nx)](https://nx.dev/blog/git-worktrees-ai-agents) — emerging isolation pattern

## Open Questions
1. **Weighted fitness sampling implementation details** — What's the right offspring penalty function? Linear, exponential, or logarithmic decay?
2. **Draft frequency** — What ratio of draft (novel) vs improve (iterative) experiments is optimal? AIDE uses `num_drafts` as a fixed count; should it be adaptive?
3. **Cross-campaign knowledge format** — ELL framework proposes "skill abstraction" but doesn't specify a format. How should our knowledge base structure learnings for reuse across campaigns?
4. **Evaluator model selection** — Should the evaluator feedback model be a different provider (e.g., GPT-4o) to avoid same-model bias, or is temperature differentiation sufficient?
5. **Tree depth vs breadth** — At what tree depth do improvements plateau for code optimization tasks? AIDE's `max_debug_depth=3` is empirical — would a different value work for non-ML code?

## Research Metadata
- Depth mode: standard
- Iterations completed: 3 / 5
- Termination reason: gaps effectively exhausted (11/12 closed, remaining gap is planning not research)
- Manifest: `.claude/deep-research/2026-03-18-autoresearch-architectures-survey.md`
