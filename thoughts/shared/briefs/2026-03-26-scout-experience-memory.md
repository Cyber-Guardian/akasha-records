---
type: event
created: 2026-03-26
status: active
---
# Idea Brief: Scout Experience Memory

**Date:** 2026-03-26
**Status:** Shaped → Parked

## Problem
Every Scout exploration starts cold — no memory of prior searches, angles, or findings. Agents re-search the same topics and waste tokens on covered ground. The ScoutRegistry persists exploration state for resume but there's no mechanism to learn from completed explorations.

## Constraints
- ScoutRegistry already persists full exploration state to ~/.claude/scout-state.json
- ExplorationState serializes fully (findings, angles, reflections survive restart)
- CuratedPackage is NOT persisted (separate dataclass, not in to_dict)
- Agent-generated content re-entering retrieval corpus causes feedback loops (A-MEM, KARMA research)
- propose() PR gate is the safety mechanism against quality degradation

## Options Considered

### Option 1: Seeding + Cache (layers A+B only)
Search result LRU cache within session + exploration seeding across sessions via question cosine similarity.
- Gains: Fast wins, no feedback risk
- Costs: No long-term learning
- Complexity: Low

### Option 2: Zettelkasten-style linked notes (A-MEM inspired)
Distillation edits existing topic notes with new findings.
- Gains: Consolidated knowledge
- Costs: Risk of overwriting human content, requires LLM merge
- Complexity: High

### Option 3: Dedicated experience store (separate LanceDB table)
Store experience without human review gate.
- Gains: Fastest, no PR noise
- Costs: Feedback loop risk, no human review
- Complexity: Medium

### Option 4: Full layered experience memory (chosen)
Three independent layers: (A) LRU search cache with TTL, (B) exploration seeding via question similarity, (C) experience distillation — curated packages written to vault via propose() as searchable "exploration summary" docs. Future presearches discover them naturally via hybrid_search.
- Gains: Compounding intelligence, human review gate, natural vault integration
- Costs: Vault growth (~1KB/exploration), PR noise, relies on prompt merging
- Complexity: Medium

## Chosen Approach
**Option 4 — Full layered experience memory**. Layers A+B already designed as tasks in scout-search-quality plan (1.6-1.7, 2.3-2.4). Layer C (distillation) needs a new phase or separate plan.

## Key Context Discovered During Shaping
- A-MEM (Feb 2025) uses Zettelkasten for agent memory with retroactive note enrichment
- Bias amplification research confirms agent-generated content in retrieval corpus degrades quality without provenance tagging + human gates
- LangMem SDK stores lessons as JSON docs in namespaced stores with procedural memory as updated system prompts
- Scout's propose() PR gate is the natural safety mechanism — distilled docs get human review before entering the corpus
- Semantic caching shows 70-75% hit rates with 93% latency reduction in production RAG systems

## Next Step
- Layers A+B: already in scout-search-quality plan (tasks 1.6-1.7, 2.3-2.4)
- Layer C (distillation): needs its own phase in scout-search-quality or a separate plan
- Parked pending ideation engine shaping (may influence how experience memory works)
