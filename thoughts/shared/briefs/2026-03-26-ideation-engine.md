---
type: event
created: 2026-03-26
status: active
---
# Idea Brief: Ideation Engine — Evolutionary Idea Exploration

**Date:** 2026-03-26
**Status:** Shaped → Parked (needs dedicated shaping session)

## Problem
The /shape skill is linear and convergent — human has idea, Claude explores, brainstorms 2-4 options, human picks. The combinatorial space of ideas is too large for a single sequential thinker. Parallel agents with different seeds find things a single agent never would.

## Core Concept
Apply the autoresearch pattern to ideation: seed idea → swarm of agents generate diverse mutations → each mutation grounded against evidence → strongest survive and cross-pollinate → idea space explored far beyond what human would think of alone.

## Key Design Questions (for next shaping session)
- MCP tool interface: ideate_start(seed) → ideate_evolve() → ideate_select() → ideate_converge()?
- Where does it live? New tool (tools/ideation-engine/) or extension to Scout?
- Mutation strategy: LLM-generated variants with structured diversity seeds?
- Grounding at scale: batch-validate 10 idea variants in parallel?
- Relationship to /shape: augments it (divergent engine underneath, human steers on top) or replaces it?
- Fitness function: grounding score + feasibility + novelty?

## Analogies
- Autoresearch: experiments = idea variants, thinker = mutation engine, evaluator = grounding function
- Genetic algorithms: population of ideas, crossover between strong variants, mutation for diversity
- Shape Up (Basecamp): appetite-bounded, rough solution sketches, betting table selection
- AIdeation (CHI 2025): breadth-first brainstorming then depth-by-research outperforms unstructured AI use

## Next Step
- Dedicated /shape session to design the MCP tool interface and architecture
- Consider whether this is a Scout extension or standalone tool
- May influence how experience memory (see companion brief) feeds back into future ideation
