---
type: event
created: 2026-03-14
status: active
---
# Idea Brief: Autonomous Experiment Harness

**Date:** 2026-03-14
**Status:** Shaped -> Planning

## Problem
There's no reusable harness for autonomous R&D — give an agent a metric, a mutation target, and a budget, and let it research, hypothesize, implement, measure, and iterate 24/7. Karpathy's autoresearch nails this for LLM training, but is domain-specific. The core loop (research -> hypothesize -> implement -> measure -> select winner -> iterate) is domain-agnostic and should be a generic framework.

## Key Insight
The agent isn't just twiddling knobs — it does real R&D. It reads papers, synthesizes ideas from academia and the web, implements novel approaches in real code, measures them against a metric, and evolves. The mutation space is unbounded: the agent brings in techniques from research literature, not just parameter sweeps.

## Constraints
- Runs 24/7 as a persistent daemon, not a CLI invocation
- Needs durable state that survives restarts (checkpoint after every experiment)
- Must have web access for research (papers, blogs, prior art)
- Needs isolated execution sandbox for each experiment
- Cost management critical — budget controls and idle backoff for continuous Claude API usage
- Single agent first, but architecture must assume parallelism ("scale is king")
- Eval dataset for first use case (dedup/compression) is a future consideration — framework is the priority

## Options Considered

### Git-Native Experiment Loop
Each experiment is a git branch/worktree. Daemon creates worktree, agent modifies code, runs metric, merges winners to champion branch. State = git history.
- Gains: Zero new infra, every experiment is a reviewable diff
- Costs: Git bad at high-frequency writes, no natural parallelism, metadata awkward in commits
- Complexity: Low-Medium

### Daemon with Experiment Queue
Long-running process with experiment queue. Research agent populates hypotheses, worker agent(s) execute experiments. State in DynamoDB, code snapshots in S3, notifications via Slack.
- Gains: Clean research/execution separation, queue gives natural parallelism (add workers), fits existing AWS infra
- Costs: More moving parts (queue, worker, state store, scheduler)
- Complexity: Medium-High

### Agent Swarm with Tournament Selection
N concurrent agents exploring different research directions. Top K survive each round, approaches cross-pollinated. Full parallel exploration from day one.
- Gains: Maximum exploration bandwidth, cross-pollination finds novel combinations
- Costs: Expensive, complex coordination, hard to debug
- Complexity: High

## Chosen Approach
**Daemon with Experiment Queue** — the natural middle ground. Queue architecture makes single->parallel transition trivial (v1 = one worker, v2 = N workers). DynamoDB + S3 + Slack already in the stack. Research/execution separation allows independent tuning. Swarm with tournament selection is the endgame evolution, not the starting point.

## The Core Loop
```
loop:
  Research    -> web search, papers, prior art on the objective
  Hypothesize -> "technique X from paper Y could improve metric Z"
  Implement   -> write/modify real Python code in isolated sandbox
  Measure     -> run metric function against eval dataset
  Score       -> compare to current champion
  Select      -> keep winner as new baseline, log everything
```

## First Use Case
Dedup/compression optimization for FileScience backup data. Clear metric (compression ratio, dedup rate, bytes saved), real knobs (algorithms, chunk sizes, strategies), rich literature (FastCDC, Rabin fingerprinting, gear-based rolling hash). Eval dataset TBD — can start with synthetic data.

## Inspiration
- [[2026-03-04-deep-advanced-agent-harness-techniques-brief|Agent Harness Techniques Research]]
- [[2026-03-04-deep-advanced-agent-architectural-patterns-brief|Agent Architectural Patterns Research]]
- Karpathy's autoresearch: https://github.com/karpathy/autoresearch

## Next Step
- Plan -> `/create_plan memory-bank/thoughts/shared/briefs/2026-03-14-autonomous-experiment-harness.md`
