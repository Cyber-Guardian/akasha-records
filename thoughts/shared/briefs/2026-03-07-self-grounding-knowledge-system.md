# Idea Brief: Self-Grounding Knowledge System

**Date:** 2026-03-07
**Status:** Shaped -> Parked

## Problem

Agent outputs are ungrounded — claims, architectural decisions, and recommendations aren't checked against a verified knowledge base. Knowledge generation is reactive (human asks a question, agent researches), and nothing compounds across sessions. There's no mechanism for agents to verify their own statements against what the organization actually knows, and no way for verification failures to automatically trigger research to fill the gap.

## Constraints

- The deep research harness (PydanticAI + pydantic-graph, STORM two-stage) is the execution engine — this system composes on top, not inside it
- Each harness run costs ~30-35 API calls, 3-5 minutes runtime — continuous operation means cost accumulates and needs budgeting
- Memory-bank + QMD (218+ indexed docs) is the existing knowledge store — the wiki either evolves from it or replaces it
- The graph-memory MCP sidecar is in progress (obsidian-graph-sidecar worktree) — provides graph analysis primitives
- Swarm harness / dev server infrastructure (macro strategy Phase 3) is the natural compute environment for always-on agents but isn't built yet
- The harness must ship first — this system is a consumer of it

## Options Considered

### Research Queue ("Background Worker")
Single daemon process with a priority queue fed by multiple sources (Slack webhook, cron jobs, gap detector scripts, adjacency suggester). Pops highest-priority item, runs the harness, integrates output into the vault. One run at a time.
- Gains: Simple to build, easy to debug, naturally rate-limits spend, follows existing triage agent pattern
- Costs: Sequential throughput cap. Intelligence lives in bolted-on feeder scripts, not the system itself. No grounding loop — just one-directional research.
- Complexity: Low-Medium

### Knowledge Gardener ("Graph-Driven Intelligence")
The vault graph IS the brain. A single agent reads the knowledge graph structure (node density, freshness, connectivity, hub centrality) and generates its own research agenda. Thin nodes trigger gap detection, dead-end nodes trigger adjacency exploration, stale nodes trigger validation. External inputs (Slack) seed new nodes that the gardener then tends.
- Gains: System understands its own knowledge. Priority emerges from structure. Gets smarter as vault grows. Unifies all research modes through graph health metrics.
- Costs: Requires well-structured graph to analyze (chicken-and-egg). More abstract, harder to debug priority decisions. Novel pattern with no existing repo precedent.
- Complexity: Medium-High

### Research Lab ("Multi-Agent Swarm")
Specialized concurrent agents — Queue Agent (Slack), Gardener Agent (gap detection), Explorer Agent (adjacency), Sentinel Agent (external source monitoring), Validator Agent (claim re-checking). Coordinator prioritizes across all proposals, dispatches parallel harness instances.
- Gains: Maximum parallelism and throughput. Each agent independently tunable. Scales with compute.
- Costs: Requires swarm harness infrastructure that doesn't exist yet. Coordination complexity for concurrent vault updates. Highest token spend.
- Complexity: High

## Chosen Approach

**Knowledge Gardener + Grounding Middleware** — a three-layer system that creates a self-reinforcing flywheel:

**Layer 1 — Wiki (Evergreen Obsidian Vault):** The knowledge base itself. Maintained by the deep research harness. Indexed by QMD for semantic search. Structured as an interlinked graph navigable by both agents and humans. Evolves from the current memory-bank.

**Layer 2 — Grounding Middleware:** Sits in the agent loop of the deep research harness. For every substantive claim the LLM produces during the agentic workflow, vector-searches the wiki for verification. Claims that can be grounded get confidence annotations. Claims that cannot be grounded or verified get pushed to the research queue. This turns every agent interaction into a knowledge audit.

**Layer 3 — Research Orchestrator:** Prioritizes and dispatches research from multiple sources:
- Grounding failures (reactive — the flywheel driver)
- Slack channel input (human-curated topics)
- Graph health analysis (thin nodes, dead ends, stale content)
- Adjacent topic discovery (spiderweb expansion from existing knowledge)
- Scheduled external monitoring (Claude Code subreddit, PydanticAI releases, etc.)
- Validation re-checks (periodic re-verification of prior research)

**The flywheel:** More agent work produces more grounding checks, which surface more gaps, which trigger more research, which enriches the wiki, which improves future grounding, which raises agent output quality.

The Research Queue is too simple — it doesn't compound. The Research Lab is premature — it depends on unbuilt swarm infrastructure. The Knowledge Gardener with grounding middleware is the approach that compounds over time while being buildable incrementally on current infrastructure.

## Key Context Discovered During Shaping

- Deep research harness is actively in development (worktree `deep-research-harness`, Phases 1-3 largely done) — PydanticAI + pydantic-graph, STORM two-stage architecture, ~30-35 API calls per run
- Obsidian graph sidecar is in progress (worktree `obsidian-graph-sidecar`) — provides graph structure analysis
- graph-memory MCP is available (`mcp__graph_memory__*` tools) — graph hubs, neighbors, paths, stats
- Meta-research on deep research identified "no self-critique before synthesis" as the biggest quality gap vs SOTA — the grounding middleware directly addresses this by making the wiki the verification oracle
- QMD pre-search (planned but not built) is a prerequisite for efficient grounding checks
- Nightly scheduled agents (ENG-2183, parked) is a narrow consumer of this broader system
- Triage agent provides a Slack integration pattern reusable for the queue feeder
- The macro agent strategy's "hive mind event log" concept overlaps with the research queue — could converge

## Next Step

- Parked — plan after the deep research harness ships (Phases 4-5 remaining in [[2026-03-05-deep-research-pydantic-harness|Deep Research PydanticAI Harness Plan]]). The harness is the execution engine this system depends on.
- When ready to plan, suggested phasing:
  1. Wiki layer — evolve memory-bank into evergreen Obsidian vault with QMD grounding API
  2. Research orchestrator — queue + Slack feeder + graph-driven prioritization
  3. Grounding middleware — embed verification into the harness agent loop
  4. Autonomous modes — adjacency discovery, scheduled monitoring, validation re-checks
- Invoke: `/create_plan memory-bank/thoughts/shared/briefs/2026-03-07-self-grounding-knowledge-system.md`
