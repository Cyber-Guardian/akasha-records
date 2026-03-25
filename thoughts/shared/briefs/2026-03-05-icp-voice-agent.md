# Idea Brief: ICP Voice Agent / Customer Advocate AI

**Date:** 2026-03-05
**Status:** Shaped -> Parked

## Problem
Product and marketing decisions at FileScience lack a structured way to pressure-test them against the actual sentiments, priorities, and objections of specific ICPs (IT/MSP operators, non-IT self-serve operators, Clio legal vertical). No systematic mechanism exists to ask "how would this persona react to this positioning/feature/priority?"

## Constraints
- No VoC data corpus exists yet (interviews, support tickets, NPS) — cold-start problem for grounded personas
- Sycophancy is the #1 failure mode — models agree with whatever you propose, undermining the "push back" requirement
- Grounded personas (RAG on real data) dramatically outperform synthetic ones (63% vs 80%+ accuracy) — but require data collection investment
- The space is validated (funded startups: Synthetic Users, Artificial Societies, Expected Parrot) but no one has built the "opinionated advocate" variant

## Options Considered

### Claude Code Extension — ICP Advocate Skill
Build as a `.claude/agents/` skill. Each ICP gets a dedicated agent with psychographic system prompt. Invoke during shape/plan sessions. Grounding data as markdown in memory-bank.
- Gains: Zero infrastructure, ships immediately, lives where decisions happen
- Costs: No RAG, single-agent, accuracy depends on prompt quality
- Complexity: Low

### Grounded Multi-Persona Panel (TinyTroupe-inspired)
Standalone Python tool in `tools/`. Three-layer architecture: persona construction from YAML + VoC data, deliberation with 2-4 ICP agents, synthesizer verdict. RAG on interview transcripts via local vector store.
- Gains: Real data grounding (80%+ accuracy), multi-persona deliberation, structured output
- Costs: Needs VoC data collected, more engineering, cold-start problem
- Complexity: Medium

### Adversarial Advocate Triad (D3-inspired)
Three roles: Customer Advocate (Socratic pushback), Business Advocate (revenue/feasibility), Synthesizer (balanced recommendation). Adversarial structure counters sycophancy by design.
- Gains: Directly addresses sycophancy, surfaces genuine tension, most intellectually honest
- Costs: Highest effort, needs both VoC and business context, three-agent coordination
- Complexity: High

## Chosen Approach
**Start with Claude Code extension, evolve toward grounded panel.** Ship prompt-grounded agents first to learn whether ICP models are useful in practice. Accumulate VoC data. Graduate to RAG-backed multi-persona panel when data justifies it.

## Key Context Discovered During Shaping
- "Opinionated customer advocate agent" is an open space — no product or practice exists (deep research confirmed)
- Socratic questioning (asking hard questions rather than stating objections) is the best sycophancy mitigation
- B2B buying committee simulation (Champion/IT Admin/CFO/Procurement) maps cleanly to FileScience's market
- Microsoft TinyTroupe and Expected Parrot EDSL are the key open-source references for future evolution
- Full deep research: `memory-bank/thoughts/shared/research/2026-03-05-deep-icp-voice-agents-customer-advocate.md`

## Next Step
- [Parked] — Linear backlog issue: ENG-2327. Unblock when VoC data collection begins or Clio Marketing kicks off.
