---
date: 2026-03-18T22:00:00-04:00
source_research: memory-bank/thoughts/shared/research/2026-03-18-deep-autoresearch-architectures-survey.md
last_generated: 2026-03-18T19:18:49.410202+00:00
---

# Research Brief: 2026-03-18-deep-autoresearch-architectures-survey

## TL;DR

The field has converged on **tree search** as the dominant orchestration pattern for autonomous research, replacing flat sequential loops. Three major variants exist: greedy best-first tree search (AI Scientist v2 / AIDE), population-based evolutionary archives (AlphaEvolve / FunSearch), and bandit-based selection. Our system already has a latent tree structure via `parent_experiment_id` but never traverses it — the highest-leverage upgrade is to build an in-memory solution tree with weighted fitness sampling for parent selection, which ShinkaEvolve shows beats both greedy and random approaches for 50-100 experiment budgets. No existing system has cross-campaign knowledge transfer, making our meta-analyst work ahead of the field. Multi-model ensembles (cheap model for leader, capable model for researcher) can cut costs 57-80%.

## Key facts / constraints

None captured.

## Key recommendations

None captured.

## Open questions

1. **Weighted fitness sampling implementation details** — What's the right offspring penalty function? Linear, exponential, or logarithmic decay?
2. **Draft frequency** — What ratio of draft (novel) vs improve (iterative) experiments is optimal? AIDE uses `num_drafts` as a fixed count; should it be adaptive?
3. **Cross-campaign knowledge format** — ELL framework proposes "skill abstraction" but doesn't specify a format. How should our knowledge base structure learnings for reuse across campaigns?
4. **Evaluator model selection** — Should the evaluator feedback model be a different provider (e.g., GPT-4o) to avoid same-model bias, or is temperature differentiation sufficient?
5. **Tree depth vs breadth** — At what tree depth do improvements plateau for code optimization tasks? AIDE's `max_debug_depth=3` is empirical — would a different value work for non-ML code?

## Links

- Source research: `memory-bank/thoughts/shared/research/2026-03-18-deep-autoresearch-architectures-survey.md`
