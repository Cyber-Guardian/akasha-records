---
date: 2026-03-04T19:00:00-05:00
source_research: memory-bank/thoughts/shared/research/2026-03-04-deep-advanced-agent-harness-techniques.md
last_generated: 2026-03-05T02:50:10.614249+00:00
---

# Research Brief: 2026-03-04-deep-advanced-agent-harness-techniques

## TL;DR

The agent harness landscape in early 2026 has converged on a shared core loop (tool-use iteration + structured outputs + streaming) while diverging sharply on orchestration model, state management, and execution substrate. Four orchestration topologies dominate production: hierarchical supervisor, mesh/swarm, DAG pipelines, and hybrid approaches. Memory architecture is converging on the MemGPT-inspired 3-tier model (core/recall/archival) with LLM-as-memory-manager. Tool composition is being reshaped by Tool RAG (semantic retrieval from large registries) and MCP (97M+ monthly downloads, universal connectivity). Observability centers on OpenTelemetry's experimental GenAI semantic conventions. For durable execution, Temporal leads production use while AWS Lambda Durable Functions offer serverless-native checkpoint-and-replay. Guardrails have solidified into a 4-layer model (I/O, execution, human-in-the-loop, behavioral testing), and context engineering has eclipsed prompt engineering as the critical discipline.

## Key facts / constraints

None captured.

## Key recommendations

None captured.

## Open questions

- How do context engineering techniques (KV-cache optimization, instruction caching) translate to serverless agent harnesses where there's no persistent process?
- What are the practical cost/quality trade-offs of reflection loops in production — when does the extra LLM call pay for itself?
- How do meta-agents (agents that design other agents) work in practice, and are they production-viable or still research?
- How will the A2A + MCP combination reshape multi-agent architectures — A2A for agent-to-agent, MCP for agent-to-tool?
- What's the right granularity for durable checkpoints in an agent loop — per-turn vs. per-tool-call vs. per-reasoning-step?

## Links

- Source research: `memory-bank/thoughts/shared/research/2026-03-04-deep-advanced-agent-harness-techniques.md`
