---
date: 2026-03-04T12:00:00-05:00
source_research: memory-bank/thoughts/shared/research/2026-03-04-deep-advanced-agent-architectural-patterns.md
last_generated: 2026-03-05T03:13:46.189146+00:00
---

# Research Brief: 2026-03-04-deep-advanced-agent-architectural-patterns

## TL;DR

Multi-agent LLM systems (2024-2026) have matured into a rich taxonomy of 9+ distinct topological orientations — from supervisor hierarchies and DAG pipelines to blackboard architectures, stigmergic coordination, debate/adversarial, ensemble/voting, market/auction, ring, and society/mesh patterns. Production experience consistently shows that topology choice must match task structure: multi-agent excels only for genuinely parallelizable, context-bounded, error-isolatable subtasks, while high-interdependency work is better served by single agents. The MAST taxonomy documents 41-87% failure rates across 7 frameworks with 14 failure modes, with externalized verification gates (+15.6% improvement) and hierarchical topologies (5.5% vs 23.7% degradation under faults) as the strongest mitigations. Cost is managed via model-tier routing (57-94% savings), prompt caching (90% reduction), and adaptive topology selection (12-23% gains). The field is converging on complementary protocol layers — MCP for vertical tool access, A2A for horizontal agent coordination — with context engineering at boundaries as the critical design surface.

## Key facts / constraints

None captured.

## Key recommendations

None captured.

## Open questions

- How will A2A adoption evolve — will it achieve MCP-level traction or remain niche?
- Can CRDTs move from theoretical (CodeCRDT) to mainstream for multi-agent file coordination?
- What does the optimal "topology selection classifier" look like in production (beyond AdaptOrch)?
- How do multi-agent systems handle long-running tasks (hours/days) where agents need to persist state across sessions?
- What are the security implications of cross-agent communication — can a compromised agent poison the topology?
- How do reward/RL-based topology optimizers compare to rule-based selection in production?

## Links

- Source research: `memory-bank/thoughts/shared/research/2026-03-04-deep-advanced-agent-architectural-patterns.md`
