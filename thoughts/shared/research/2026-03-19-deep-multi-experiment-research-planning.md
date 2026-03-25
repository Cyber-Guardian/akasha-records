---
type: event
created: 2026-03-19
status: active
date: 2026-03-19T19:30:00-04:00
researcher: Claude
git_commit: ef2eb7220052d95e8ee398c628d3c7981d48de4d
branch: feat/ENG-2675-dashboard-control-room
repository: filescience
topic: "How do real research labs and AI agent frameworks handle multi-experiment research planning?"
tags: [deep-research, autoresearch, agent-architecture, experiment-planning, research-methodology]
research_depth: standard
iterations_completed: 3
last_updated: 2026-03-19
last_updated_by: Claude
---

# Deep Research: Multi-Experiment Research Planning

## Research Question
How do real research labs and AI agent frameworks handle multi-experiment research planning? Specifically: (1) How do PIs/research directors plan experiment campaigns with branching logic? (2) How do autonomous AI research systems (FunSearch, AI Scientist v2, AIDE) handle research direction, sequencing, review, and context curation?

## Summary
Real R&D labs and autonomous AI research systems converge on three architectural patterns for multi-experiment planning: tree-search with branching/pruning (AI Scientist v2, AIDE), evolutionary islands with selection pressure (FunSearch), and stage-gate sequential review (pharma, clinical trials). All three separate planning from execution, review between steps, and prune/pivot when evidence warrants. The most effective systems combine a "thinker" role (PI, experiment manager, acquisition function) that plans and reviews with "researcher" roles that execute. Pre-registration/adversarial review raises replication rates to ~86% but should be tiered by risk. Context curation consensus: distill to 1-2K tokens per handoff (Anthropic guidance), place critical info at context boundaries (U-curve), and use selective retrieval not dumps.

## Perspectives Explored
1. **Real-world R&D lab operations** — Revealed PI-as-director model with if-then branching plans, stage-gate reviews, and moderate specification as optimal briefing granularity
2. **Autonomous AI research systems** — Revealed three distinct architectures: BFTS (AI Scientist v2), evolutionary islands (FunSearch), and phased tree search (AIDE)
3. **Experiment planning theory** — Revealed Bayesian optimization as the mathematical framework: model unknowns as distributions, greedily select experiments by expected information gain
4. **Adversarial review in research** — Revealed pre-registration works but should be tiered by risk; AI systems currently only do post-execution review
5. **Context/information curation** — Revealed distillation-based handoff as the consensus pattern, with U-curve attention placement and selective retrieval

## Detailed Findings

### 1. Real-World R&D Lab Operations

**Planning structure:** PIs create branching contingency plans upfront. NIH grant guidance explicitly requires if-then decision trees: "if result X, follow pathway X; if result Y, follow pathway Y." This mirrors what our thinker should produce — not a single hypothesis, but a research direction with contingencies.

**The PDCA loop:** Real labs run Plan-Do-Check-Act cycles with interim reviews at predefined gates. Stage-gate processes (dominant in pharma) impose formal go/no-go decisions between phases — leading companies report 40% faster outcomes from disciplined gate reviews. Gate outcomes: go, kill, hold, or recycle.

**PI→researcher handoff:** Layered, mostly verbal. Weekly 1-on-1s carry strategic "what and why"; written protocols capture procedural detail. PIs rarely write full protocols — they specify the question and key variables, then gate-approve the researcher's draft. The "why" behind decisions is poorly institutionalized (lives in verbal lore).

**Optimal briefing granularity:** Research shows moderate specification (goal + key variables + prior context) outperforms both vague direction and over-specified step-lists for creative problem-solving. This directly informs thinker→researcher handoff design.

**Pivot signals:** Accumulated interim data triggers pivots. "Double-loop learning" describes deeper pivots where a failed result challenges the underlying theory, not just tactics. Adaptive clinical trial designs formalize this with pre-specified interim analyses.

### 2. Autonomous AI Research Systems

**AI Scientist v2 (Sakana):** Best-First Tree Search (BFTS) with `num_workers` parallel workers (default 4) expanding nodes across four sequential experimental stages. Branch selection by LLM-scored node quality — not just scalar metrics. Each node stores: code/hypothesis, execution results, VLM figure feedback, debug history. Pruning: `max_debug_depth=3` (abandon after 3 failed fixes), `debug_prob=0.5` (randomize retry vs move on). Reviewer is **post-execution only** — scores completed manuscripts against ICLR criteria via VLM loop. Built directly on AIDE's tree search.

**FunSearch (DeepMind):** Clearest proposer-evaluator split. ~1,000 islands, each initialized with seed program. Programs clustered by score signature within each island. Sampler picks island uniformly, then samples k=2 programs favoring higher-scoring/shorter ones. LLM proposes new candidates from in-context examples. **Diversity maintenance:** Periodic culling every ~4 hours — bottom 50% of islands reset by copying a surviving island. Selection pressure replaces explicit planning. Scoring is continuous (user-supplied evaluation function).

**AIDE (Weco AI):** Three-phase hard-coded policy: (1) Draft 5 diverse seeds (diversity via shuffled prompt environment), (2) Debug broken nodes with prob 0.5 up to depth 3, (3) Greedily improve best node by scalar metric. Strictly sequential (one node per iteration). Each node: code, plan, metric, is_buggy, parent, children, analysis, exec_time, debug_depth. AI Scientist v2 extends AIDE by adding parallel workers and LLM-scored (not just scalar) selection.

**Key insight:** The most successful systems (AI Scientist v2) added LLM-based evaluation on top of scalar metrics — the "thinker" evaluates quality dimensions that a number can't capture.

### 3. Experiment Planning Theory (DoE/Bayesian Optimization)

**Unifying framework:** Model the unknown as a distribution → compute expected value of each candidate experiment → select greedily or via lookahead → update model → repeat.

**Bayesian optimization:** Fits a surrogate model (Gaussian Process) after each experiment. Acquisition functions (UCB, Expected Improvement, Thompson Sampling) balance exploration vs exploitation. This is the mathematical version of what the thinker should do: maintain a mental model of the solution space and select the next experiment that maximizes expected information gain.

**Multi-armed bandits:** Frame as regret minimization under a round budget. Explore-first strategies shift to exploitation as evidence accumulates. Applicable to personality/model selection — which personality "arm" to pull next.

**Adaptive clinical trials:** Formalize stopping rules as posterior probability thresholds at pre-specified interim analyses. Directly applicable to our stale_threshold and budget exhaustion logic.

**Active learning:** Select queries that maximize expected uncertainty reduction per cost unit. The thinker should ask: "which experiment would teach us the most per dollar spent?"

### 4. Adversarial Review in Research

**Pre-registration works:** Raised psychology replication to ~86% (vs 30-70% without). Sharp rise in null results (reduced publication bias). Strongest effect with detailed pre-analysis plans; bare registration shows weaker results.

**Cost is real:** Surveys find pre-registration adds work stress and extends timelines. Peer review runs 4-10x longer than researchers consider optimal.

**AI Scientist v2's approach:** Reviewer is **post-execution only** — scores completed papers, doesn't gate experiments before they run. This means it catches bad science but doesn't prevent wasted compute.

**Emerging consensus:** Tier review intensity by risk:
- **Trivial experiments** (knob turns): no review, just run
- **Standard experiments**: lightweight pre-check (is this hypothesis novel? does it have a chance of beating champion?)
- **Architectural experiments**: formal adversarial review (challenge assumptions, check feasibility, estimate resource cost)

**Implication for our harness:** The thinker's plan review is most valuable for expensive/risky experiments. For cheap knob turns, the review overhead exceeds the experiment cost.

### 5. Context/Information Curation

**Anthropic's guidance:** Sub-agents should return 1-2K token condensed summaries from tens-of-thousands of tokens of exploration. Lead agents consume only filtered outputs. This is the blueprint for researcher→thinker feedback.

**Manus pattern:** Externalize long-term state to file system. Use todo.md recitation to keep goals in the recency window — directly counteracting U-curve attention degradation.

**U-curve (Lost in the Middle):** >30pp performance swing between boundary and middle positions. GPT-3.5 performed worse WITH the answer present in middle than without it. Implication: context curation must be attention-aware — place critical info at start or end.

**Agentic RAG:** Evolved toward "context sufficiency" — not just relevance but whether the LLM has enough to answer correctly. Retrieve only what's needed.

**AgentRxiv pattern:** Agent labs sharing compressed research outputs via a shared preprint-style store. Downstream agents retrieve selectively rather than ingesting all prior work. This maps to our experiment log — the thinker should select which prior experiments are relevant for each researcher, not dump all of them.

**Real lab pattern:** The "why" behind decisions lives in verbal lore, not structured documents. This is the gap our thinker can fill — by making the research rationale explicit in the plan.

### Cross-Cutting Patterns

**Three architectural archetypes:**

| Pattern | Example | Planning | Diversity | Review | Best for |
|---------|---------|----------|-----------|--------|----------|
| Tree search + branching | AI Scientist v2, AIDE, our solution tree | Explicit (node expansion) | Parent selection | Post-execution scoring | Directed optimization |
| Evolutionary islands | FunSearch | Implicit (selection pressure) | Island isolation + culling | Continuous scoring | Open-ended discovery |
| Stage-gate sequential | Pharma, clinical trials | Explicit (if-then plans) | Gate outcomes (go/kill/hold/recycle) | Formal gate reviews | High-stakes, regulated |

**Universal principles:**
1. **Separate planning from execution** — every effective system does this, whether via a PI, an experiment manager, or an acquisition function
2. **Review between steps** — the thinker must see results before deciding the next experiment
3. **Prune early** — max_debug_depth, stage gates, island culling all share the principle: cut losses fast
4. **Moderate specification** — give researchers goal + key variables + relevant context, not step-by-step instructions
5. **Tier effort by risk** — don't spend opus-level thinking on ZSTD_LEVEL changes

## Key Sources

### Academic Papers
- [Mathematical discoveries from program search with LLMs (FunSearch) — Nature 2024](https://www.nature.com/articles/s41586-023-06924-6)
- [AI Scientist v2: Agentic Tree Search — Sakana AI 2025](https://arxiv.org/abs/2504.08066)
- [AIDE: AI-Driven Exploration in the Space of Code — Weco AI 2025](https://arxiv.org/html/2502.13138v1)
- [Lost in the Middle: How Language Models Use Long Contexts — MIT/TACL 2024](https://direct.mit.edu/tacl/article/doi/10.1162/tacl_a_00638/119630)
- [Deep Adaptive Design: Amortizing Sequential Bayesian Experimental Design](https://arxiv.org/abs/2103.02438)
- [Pre-registration raises replication rate to ~86% — Science/AAAS](https://www.science.org/content/article/preregistering-transparency-and-large-samples-boost-psychology-studies-replication-rate)
- [Organizing Creativity with Constraints — SAGE Journals](https://journals.sagepub.com/doi/full/10.1177/10564926231191087)

### Industry/Practice
- [Effective Context Engineering for AI Agents — Anthropic](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents)
- [Context Engineering for AI Agents: Lessons from Building Manus](https://manus.im/blog/Context-Engineering-for-AI-Agents-Lessons-from-Building-Manus)
- [NIH NIAID: Write Your Research Plan — if-then branching](https://www.niaid.nih.gov/grants-contracts/write-research-plan)
- [Stage-Gate Process in Pharma R&D](https://www.pharmafocusasia.com/articles/increasing-speed-randd-stage-gate)
- [Ten Simple Rules for Productive Lab Meetings — PMC](https://pmc.ncbi.nlm.nih.gov/articles/PMC8158921/)

### Code Repositories
- [google-deepmind/funsearch — GitHub](https://github.com/google-deepmind/funsearch)
- [WecoAI/aideml — GitHub](https://github.com/WecoAI/aideml)
- [SakanaAI/AI-Scientist-v2 — GitHub](https://github.com/SakanaAI/AI-Scientist-v2)

### Memory-bank
- [[2026-03-12-agent-harness-composition|Agent Harness Composition Brief]]
- [[2026-03-06-deep-validate-curated-context-architecture|Curated Context Architecture Research]]
- [[2026-03-19-autoresearch-wave3-thinker-architecture|Wave 3 Thinker Architecture Brief]]

## Open Questions
- How should the thinker's multi-experiment plan schema look concretely? (Nodes with branching conditions? Sequential with checkpoints? Hybrid?)
- Should the thinker maintain an explicit Bayesian model of the solution space, or is LLM intuition sufficient?
- How do we prevent the thinker from over-planning — when does the plan become a constraint rather than a guide?
- What's the right format for thinker→researcher context curation instructions? (File paths? Experiment IDs? Natural language selection criteria?)

## Research Metadata
- Depth mode: standard
- Iterations completed: 3 / 5
- Termination reason: gaps exhausted (all 8 gaps closed in 3 iterations)
- Manifest: `.claude/deep-research/2026-03-19-how-do-real-research-labs.md`
