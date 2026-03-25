---
date: 2026-03-23T14:48:58.955596+00:00
source_research: memory-bank/thoughts/shared/research/2026-03-23-awesome-autoresearch-ecosystem.md
last_generated: 2026-03-23T14:48:58.955596+00:00
---

# Research Brief: 2026-03-23-awesome-autoresearch-ecosystem

## TL;DR

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

## Key facts / constraints

None captured.

## Key recommendations

None captured.

## Open questions

None

## Links

- Source research: `memory-bank/thoughts/shared/research/2026-03-23-awesome-autoresearch-ecosystem.md`
