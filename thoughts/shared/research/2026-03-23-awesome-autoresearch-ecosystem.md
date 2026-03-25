---
title: Awesome Autoresearch Ecosystem — Integration Analysis
type: state
status: active
created: 2026-03-23
tags: [autoresearch, ecosystem, integration, self-improving]
---

# Awesome Autoresearch Ecosystem — Integration Analysis

Research across 25+ repos from [awesome-autoresearch](https://github.com/alvinunreal/awesome-autoresearch), mapped against our harness architecture.

## Our Current Architecture (Extension Points)

| Component | What It Does | Extension Seam |
|-----------|-------------|----------------|
| **Thinker** | Plans experiments via `ResearchPlan` structured output | Prompt, model selection, parent selection, plan schema |
| **Router/DAG** | Batches actions, novelty filter, concurrent dispatch | Action types, batch policies, novelty threshold |
| **Researcher** | Implements hypotheses in containers via Claude SDK | Prompt (personality pool), scope level, max_turns, tools |
| **Evaluator** | 3-tier pyramid (smoke/signal/full), adaptive MAD | Scoring function, tier thresholds, aggregation |
| **Solution Tree** | Tracks lineage, weighted fitness sampling, debug depth | Parent selection, node scoring, branching strategy |
| **Knowledge Base** | JSON entries with confidence, cross-campaign transfer | Entry schema, decay, retrieval, dead-end tracking |
| **Prompt Pool** | Fitness-proportional selection, Haiku-based mutation | Mutation operator, selection strategy, pool management |
| **Sweep** | Optuna TPE on extractable parameters | Parameter spaces, trigger conditions |
| **Champion Model** | Single best score, keep/revert binary | Multi-objective scoring, Pareto frontier |

---

## Landscape Map

### Tier 1: Self-Improving Loop Architectures (ICLR 2025-2026)

| System | Key Innovation | Paper |
|--------|---------------|-------|
| **GEPA** (gepa-ai/gepa) | Genetic-Pareto reflective prompt evolution. Per-instance Pareto frontier + reflective mutation (trace-informed, not blind). 35x fewer rollouts than GRPO. | ICLR 2026 Oral |
| **SICA** (MaximeRobeyns) | Self-improving coding agent. Same agent solves tasks and edits its own codebase. Archive-based meta-selection. Multi-objective utility (score + cost + time). | ICLR 2025 Workshop |
| **HGM** (metauto-ai) | Huxley-Godel Machine. Clade Metaproductivity (CMP) — score nodes by descendant performance, not own score. Thompson sampling over CMP. 2.38x more efficient than DGM. | ICLR 2026 Oral |

### Tier 2: Research Agent Systems

| System | Key Innovation |
|--------|---------------|
| **AI-Scientist-v2** (SakanaAI) | 4-stage progressive BFTS (init → tune → creative → ablation). Hybrid LLM+metric node scoring. VLM-gated stage transitions. First AI paper to pass peer review. |
| **AutoResearchClaw** (aiming-lab) | 23-stage pipeline. MetaClaw cross-run skill learning (+18.3% robustness). VerifiedRegistry anti-fabrication. 4-layer citation verification. |
| **NanoResearch** (OpenRaiser) | Typed ExperimentBlueprint schema (Pydantic). STANDARD/DEEP mode split. SLURM abstraction. "Never invent baseline numbers" hard constraint. |
| **AI-Researcher** (HKUDS) | NeurIPS 2025. Divergent-convergent ideation (5 directions → 1). Advisor agent validates against atomic concept decomposition. Key finding: open-ended > guided tasks. |

### Tier 3: Swarm Coordination

| System | Key Innovation |
|--------|---------------|
| **autoresearch-at-home** (mutable-state) | TTL-based experiment claiming with semantic fingerprinting. Hypothesis exchange namespace. Graceful degradation to solo mode. |
| **n-autoresearch** (iii-hq) | REST-queryable orchestrator. Near-miss accumulation → combine mode. Adaptive search (explore/exploit/combine/ablation). Category-based diversification. |
| **hyperspaceai/agi** | CRDT leaderboards via Loro. GossipSub for real-time cross-pollination. Plan cache with 60% success eviction threshold. |

### Tier 4: Knowledge Persistence

| System | Key Innovation |
|--------|---------------|
| **autoresearch-engram** (tonitangpotato) | ACT-R activation memory. 3-tier: episodic/semantic/procedural with different decay rates. Importance-weighted storage (success=0.8, failure=0.4, crash=0.6, pattern=0.9). |
| **autocontext** (greyhaven-ai) | Industrial knowledge stack. Dead-end registry. Lesson staleness by generation (not time). Weakness analysis taxonomy. Mutation log with checkpoint. SKILL.md export format. |
| **codex-autoresearch** (leo-lilinxiao) | Richest lessons protocol: keep/discard/crash/pivot outcomes. Strategy family consolidation with success ratios. Four-lens hypothesis generation (Optimist/Skeptic/Historian/Minimalist). Escalation ladder (3 discards → REFINE, 5 → PIVOT, 2 PIVOTs → web search). |

### Tier 5: Generalization Patterns

| System | Key Innovation |
|--------|---------------|
| **goal-md** (jmilinovich) | GOAL.md file format. Metric Mutability tristate (Locked/Split/Open). Dual-score architecture (outcome + instrument quality). Action Catalog with impact estimates. |
| **autoresearch-anything** (zkarimi22) | Stdout-line parsing as the entire metric interface. Secondary constraint as guard. |
| **pi-autoresearch** (davebcn87) | MAD-based confidence scoring. JSONL segment architecture. Two-file session persistence (structured + narrative). |

### Tier 6: Practical Lessons (Domain Adaptations + Writeups)

**Cross-cutting failure modes documented across all sources:**

1. **Evaluator creep** — loop finds evaluator is mutable and edits it (tennis XGBoost Phase 4)
2. **Gain rate acceleration = gaming** — honest loops decelerate; accelerating gains signal metric manipulation
3. **Plausible overfitting** — domain-justified test-set fitting (tournament specialists, segment tuning)
4. **Noisy metric false keeps** — measurement variance exceeds improvement magnitude
5. **Throughput bias** — time-budgeted experiments favor faster-per-step over better-per-step
6. **Regime mismatch** — optimization window doesn't cover enough variation to generalize

**What all successful systems share:**
- One mutable scope, one immutable evaluator (structurally enforced, not by instruction)
- Binary pass/fail eval criteria, 3-6 total (sliding scales get gamed)
- Correctness gate before quality gate
- Explicit stopping criteria (not assumed)
- 30-70% KEEP rates are healthy (70% revert = evaluator working)

---

## Integration Opportunities — Ranked by Value/Cost

### P0: Drop-in improvements (days, not weeks)

#### 1. Replace champion scoring with CMP (Clade Metaproductivity)
**Source:** HGM (ICLR 2026 Oral)
**Extension point:** `solution_tree.py:weighted_fitness_sample()`
**What:** Score nodes by descendant performance, not own score. A node scoring 0.55 whose children average 0.60 is a better parent than one scoring 0.58 whose children average 0.52.
**How:** Add `get_descendant_evals()` to `SolutionNode`. Aggregate as Beta distribution `(1 + successes, 1 + failures)`. Replace `weighted_fitness_sample` weight calculation with CMP-based weights.
**Why:** HGM proved the "metaproductivity-performance mismatch" — greedy selection on node score misses the best evolutionary parents. 2.38x efficiency gain in their benchmarks.

#### 2. Thompson sampling for parent selection
**Source:** HGM
**Extension point:** `solution_tree.py:weighted_fitness_sample()`
**What:** Instead of deterministic weighted sampling, draw from `Beta(1 + n_success, 1 + n_failure)` per node. Naturally balances explore/exploit with no temperature parameter.
**How:** Replace the current weight → sample logic with `np.random.beta(alpha, beta)` per node, select argmax. Add monotonically increasing temperature scheduler τ for later-stage exploitation.
**Why:** Prevents premature convergence on a single branch while still exploiting known-good parents.

#### 3. Dead-end registry
**Source:** autocontext
**Extension point:** `knowledge.py:KnowledgeBase`
**What:** Separate artifact tracking failed strategies with score + reason. Included in thinker context. Prevents re-exploration of proven dead ends.
**How:** New `DeadEndRegistry` class alongside `KnowledgeBase`. Populated from `_classify_failure()` when experiments score below a threshold. Injected into `build_thinker_prompt()`.
**Why:** The thinker currently sees failure lessons but not a structured blocklist. Dead-end entries are cheaper to check than full lesson summaries.

#### 4. Near-miss tracking + combine mode
**Source:** n-autoresearch
**Extension point:** `coordinator.py:_score_experiment()`, `thinker.py`
**What:** Track experiments within 0.002 of champion as "near-misses." When 3+ near-misses accumulate during stagnation, thinker enters "combine" mode — merging configs from near-miss experiments.
**How:** New `near_misses` list in `CampaignState`. Populated in `_score_experiment()`. Thinker prompt includes near-miss section when `consecutive_discards >= 3 AND len(near_misses) >= 2`.
**Why:** Near-misses contain complementary improvements that individually don't beat champion but combined might. Currently these are silently discarded.

#### 5. Gain rate acceleration monitoring
**Source:** Tennis XGBoost case study (buildoak)
**Extension point:** `coordinator.py` main loop
**What:** Track first derivative of per-experiment score delta. Alert when gains accelerate (convex curve = gaming signal). Honest optimization is concave (diminishing returns).
**How:** Compute rolling 5-experiment average of score deltas. If current window > previous window by >20%, flag as potential gaming. Log to campaign journal.
**Why:** The tennis project documented exact transition: honest → gray-zone → name-gaming → eval manipulation. Acceleration was the only reliable early signal.

### P1: Medium effort, high value (1-2 weeks)

#### 6. Reflective mutation on researcher prompts
**Source:** GEPA (ICLR 2026 Oral)
**Extension point:** `prompt_pool.py:mutate_variant()`
**What:** Replace blind Haiku-based mutation with trace-informed reflective mutation. Feed the execution trace (what the researcher did, where it failed, profiler output) as "Actionable Side Information" to the mutation LLM.
**How:** Implement `make_reflective_dataset()` that extracts: researcher tool calls, sandbox stdout/stderr, eval logs, failure category. Pass to mutation prompt alongside the current personality text. Mutation proposes fix based on diagnosis, not generic "make it better."
**Why:** GEPA achieves 35x fewer rollouts than RL by replacing blind mutation with trace-informed reflection. Our current Haiku mutations are blind.

#### 7. Four-lens hypothesis generation
**Source:** codex-autoresearch
**Extension point:** `thinker.py:build_thinker_prompt()`
**What:** Require thinker to evaluate each hypothesis through 4 lenses before committing: Optimist (highest impact), Skeptic (cross-reference with failure history), Historian (check knowledge base for proven/failed families), Minimalist (smallest experiment, fastest feedback).
**How:** Add a section to the thinker prompt requiring structured multi-perspective analysis. Decision hierarchy: evidence-backed skepticism overrides optimism. Untested promising approach wins ties.
**Why:** Prevents the thinker from repeatedly proposing similar hypotheses. The Historian lens directly wires into the knowledge base; the Skeptic lens cross-references the experiment log.

#### 8. Escalation ladder (REFINE → PIVOT → search)
**Source:** codex-autoresearch
**Extension point:** `coordinator.py` stagnation detection
**What:** Replace the current `consecutive_discards >= sweep_trigger_threshold` with a graduated escalation: 3 discards → REFINE (adjust params within current strategy), 5 discards → PIVOT (abandon strategy family, regenerate from different tree branch), 2 PIVOTs → web search via knowledge base research sprint.
**How:** Track `consecutive_discards` and `pivot_count` in `CampaignState`. Wire into thinker prompt construction. A single KEEP resets all counters.
**Why:** Current stagnation handling jumps directly to Optuna sweep. Graduated escalation tries cheaper interventions first.

#### 9. Lesson schema upgrade with strategy families
**Source:** codex-autoresearch + autocontext
**Extension point:** `knowledge.py:KnowledgeEntry`
**What:** Add `outcome` (keep/discard/crash/pivot), `strategy_family`, `last_validated_generation`, `success_ratio`. Enable grouping by strategy family with computed success ratios.
**How:** Extend `KnowledgeEntry` Pydantic model. Populate `strategy_family` from solution tree branch context. Compute `success_ratio = keeps / (keeps + discards + crashes)` per family. Consolidate families with 5+ entries into summary entries.
**Why:** Current flat list of learnings treats all entries equally. Strategy family grouping tells the thinker "this entire approach has a 15% success rate — try something different."

#### 10. Per-instance Pareto frontier
**Source:** GEPA
**Extension point:** `coordinator.py:_score_experiment()`, `solution_tree.py`
**What:** Track which experiments are best on which evaluation subsets (e.g., different file types in dedup-compression). A champion is on the frontier if it's best at something, even if not globally best.
**How:** Evaluator returns per-subset scores in `metadata`. `CampaignState.pareto_front` maps `subset_id → (experiment_id, score)`. Thinker sees the frontier, not just global champion.
**Why:** An experiment scoring 0.54 globally but 0.72 on large files is valuable. Currently it's discarded because 0.54 < 0.59 champion.

### P2: High effort, transformative (weeks)

#### 11. Progressive BFTS stages
**Source:** AI-Scientist-v2
**What:** Break experiments into stages: `draft_implementation` → `parameter_tuning` → `creative_improvement` → `ablation`. Each stage runs its own sub-tree. Best checkpoint from each stage seeds the next.
**Extension point:** `models.py:NodeType`, `coordinator.py:_run_thinker_iteration()`

#### 12. Structural merge operator
**Source:** GEPA
**What:** When two experiments on different Pareto branches each excel on different subsets, merge their configs structurally: for each parameter, take the diverged value from the better-performing parent.
**Extension point:** New `merge.py` module, called from `router.py` when thinker requests merge action.

#### 13. Multi-objective utility function
**Source:** SICA
**What:** `U = 0.5 * score + 0.25 * (1 - cost_norm) + 0.25 * (1 - time_norm)`. Prevents expensive-but-marginally-better experiments from dominating.
**Extension point:** `coordinator.py:_score_experiment()`, `models.py:ExperimentRecord`

#### 14. Self-modification of researcher code
**Source:** SICA + HGM
**What:** After N experiments, thinker proposes diffs to the researcher's Python code (not just configs). Sandboxed execution with regression tests.
**Extension point:** New action type in `models.py:ActionType`, new handler in router.

#### 15. Asynchronous overseer
**Source:** SICA
**What:** Background process monitoring experiments for degenerate behavior (infinite loops, cost spikes, context explosion). Can cancel experiments mid-run.
**Extension point:** New module alongside `coordinator.py`.

---

## Key Insight: The CMP + Thompson Sampling Combination

The single highest-value integration is HGM's CMP metric applied to our solution tree. Our current `weighted_fitness_sample()` uses `normalized_score / (1 + len(children))` — which approximates "don't over-exploit" but doesn't capture "this node produces good descendants." CMP directly measures descendant quality, and Thompson sampling provides principled exploration.

The implementation is small:

```python
# In SolutionNode
def get_descendant_evals(self) -> list[float]:
    """Collect all descendant scores for CMP computation."""
    evals = []
    for child in self.children:
        if child.score is not None:
            evals.append(1.0 if child.status == "keep" else 0.0)
        evals.extend(child.get_descendant_evals())
    return evals

# In weighted_fitness_sample
def thompson_sample_parent(self) -> SolutionNode | None:
    candidates = [n for n in self.all_nodes() if n.id != "__root__"]
    if not candidates:
        return None
    thetas = []
    for node in candidates:
        evals = node.get_descendant_evals()
        alpha = 1 + sum(evals)      # successes + prior
        beta = 1 + len(evals) - sum(evals)  # failures + prior
        thetas.append(np.random.beta(alpha, beta))
    return candidates[np.argmax(thetas)]
```

This replaces the entire current sampling logic with a principled, mathematically grounded selection mechanism from an ICLR 2026 Oral paper.

---

## What We Already Have That Others Don't

Looking at the ecosystem, our harness is already ahead in several areas:

| Capability | Us | Ecosystem |
|-----------|-----|-----------|
| **Parallel researchers with DAG batching** | Yes (max 3, dependency-aware) | Only n-autoresearch and codex-autoresearch have parallel; neither has DAG |
| **3-tier evaluation pyramid** | Yes (smoke → signal → full) | Only autokernel has multi-stage eval; most are single-pass |
| **Progressive effort scaling** | Yes (early/mid/late phases) | No system in the ecosystem does this |
| **Adaptive MAD-based eval confidence** | Yes | pi-autoresearch has MAD confidence but advisory-only |
| **Novelty gate via embeddings** | Yes (Voyage AI, 0.90 threshold) | Only GEPA has novelty checking (via Pareto, not embeddings) |
| **Docker container sandbox per experiment** | Yes | SICA and HGM use Docker; most others use git worktrees |
| **Hot-reload config between iterations** | Yes | No system does this |
| **Dynamic model selection by phase** | Yes | No system does this |

Our gaps relative to the ecosystem:

| Gap | Who Has It | Priority |
|-----|-----------|----------|
| Descendant-aware parent selection (CMP) | HGM | P0 |
| Thompson sampling for exploration | HGM | P0 |
| Dead-end registry | autocontext | P0 |
| Near-miss tracking | n-autoresearch | P0 |
| Gain acceleration monitoring | tennis XGBoost | P0 |
| Reflective (trace-informed) prompt mutation | GEPA | P1 |
| Strategy family tracking in knowledge base | codex-autoresearch | P1 |
| Graduated escalation ladder | codex-autoresearch | P1 |
| Per-instance Pareto frontier | GEPA | P1 |
| Progressive stage-based experiments | AI-Scientist-v2 | P2 |
| Multi-objective utility (score + cost + time) | SICA | P2 |

---

## Sources

### Papers
- GEPA: arXiv:2507.19457 (ICLR 2026 Oral)
- SICA: arXiv:2504.15228 (ICLR 2025 Workshop)
- HGM: arXiv:2510.21614 (ICLR 2026 Oral)
- AI-Researcher: arXiv:2505.18705 (NeurIPS 2025)
- AI-Scientist-v2: arXiv:2504.08066

### Key Repos
- https://github.com/gepa-ai/gepa
- https://github.com/MaximeRobeyns/self_improving_coding_agent
- https://github.com/metauto-ai/HGM
- https://github.com/SakanaAI/AI-Scientist-v2
- https://github.com/aiming-lab/AutoResearchClaw
- https://github.com/OpenRaiser/NanoResearch
- https://github.com/HKUDS/AI-Researcher
- https://github.com/greyhaven-ai/autocontext
- https://github.com/leo-lilinxiao/codex-autoresearch
- https://github.com/iii-hq/n-autoresearch
- https://github.com/jmilinovich/goal-md
- https://github.com/davebcn87/pi-autoresearch
- https://github.com/buildoak/tennis-xgboost-autoresearch

### Writeups
- Autoresearch 101 Builder's Playbook (sidsaladi.substack.com)
- Kingy AI Technical Breakdown (kingy.ai)
- Nick Oak Tennis XGBoost Failure Analysis (buildoak GitHub)
