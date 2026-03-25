---
date: 2026-03-19T19:30:00-04:00
source_research: memory-bank/thoughts/shared/research/2026-03-19-deep-multi-experiment-research-planning.md
last_generated: 2026-03-19T23:48:51.171494+00:00
---

# Research Brief: 2026-03-19-deep-multi-experiment-research-planning

## TL;DR

Real R&D labs and autonomous AI research systems converge on three architectural patterns for multi-experiment planning: tree-search with branching/pruning (AI Scientist v2, AIDE), evolutionary islands with selection pressure (FunSearch), and stage-gate sequential review (pharma, clinical trials). All three separate planning from execution, review between steps, and prune/pivot when evidence warrants. The most effective systems combine a "thinker" role (PI, experiment manager, acquisition function) that plans and reviews with "researcher" roles that execute. Pre-registration/adversarial review raises replication rates to ~86% but should be tiered by risk. Context curation consensus: distill to 1-2K tokens per handoff (Anthropic guidance), place critical info at context boundaries (U-curve), and use selective retrieval not dumps.

## Key facts / constraints

None captured.

## Key recommendations

None captured.

## Open questions

- How should the thinker's multi-experiment plan schema look concretely? (Nodes with branching conditions? Sequential with checkpoints? Hybrid?)
- Should the thinker maintain an explicit Bayesian model of the solution space, or is LLM intuition sufficient?
- How do we prevent the thinker from over-planning — when does the plan become a constraint rather than a guide?
- What's the right format for thinker→researcher context curation instructions? (File paths? Experiment IDs? Natural language selection criteria?)

## Links

- Source research: `memory-bank/thoughts/shared/research/2026-03-19-deep-multi-experiment-research-planning.md`
