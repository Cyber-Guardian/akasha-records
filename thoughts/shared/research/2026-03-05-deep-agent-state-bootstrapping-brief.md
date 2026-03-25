---
date: 2026-03-05T18:00:00-05:00
source_research: memory-bank/thoughts/shared/research/2026-03-05-deep-agent-state-bootstrapping.md
last_generated: 2026-03-06T02:02:02.772468+00:00
---

# Research Brief: 2026-03-05-deep-agent-state-bootstrapping

## TL;DR

The field has converged on a hybrid "summary + pointer" architecture: a tiny always-loaded constitution (<500 tokens of identity and hard constraints) combined with on-demand semantic retrieval for everything else, yielding ~90% token savings versus naive preloading. Five complementary strategies dominate: tiered memory hierarchies (Cline Memory Bank, HiAgent), agentic RAG with vector-indexed retrieval (Cursor, Continue.dev), structured checkpoint files and status tokens as lightweight state machines, session handoff documents achieving 80-90% compression, and observation management techniques like SWE-agent's sliding window and Aider's PageRank repo maps. Context starvation is generally worse than bloat for task success, so the key design tension is loading enough to prevent hallucinated conventions and architectural amnesia while avoiding the "lost in the middle" effect that makes preloaded background material effectively invisible.

## Key facts / constraints

None captured.

## Key recommendations

None captured.

## Open questions

- **Context sufficiency detection:** No framework has a first-class mechanism for agents to detect they're missing critical context. How would you build one? (confidence calibration? convention validation pre-flight?)
- **Optimal tier boundaries:** The 3-4 tier pattern is consensus but the token budgets per tier are ad hoc. What's the empirically optimal split for different task types?
- **Cross-session learning:** Most approaches treat each session as independent. How should an agent's retrieval strategy improve over time based on which context turned out to be relevant?
- **Multi-agent state sharing:** When agents in a swarm need overlapping context, what's the optimal deduplication strategy? Shared vector index? Centralized state broker?
- **Dynamic context reordering:** Given the lost-in-the-middle effect, should agents actively reorder their context window during a session as relevance shifts?

## Links

- Source research: `memory-bank/thoughts/shared/research/2026-03-05-deep-agent-state-bootstrapping.md`
