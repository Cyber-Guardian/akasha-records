# Idea Brief: Harness Self-Improvement — Pattern Staleness Detection

**Date:** 2026-03-02
**Status:** Shaped → Parked

## Problem
Our agentic development harness (orchestration model, memory architecture, workflow lifecycle, scope control, tool/platform patterns, meta-patterns) encodes patterns that were best-practice when written — but the field moves weekly. No mechanism exists to detect when our harness foundations have been superseded by new research, tools, or community patterns.

## Constraints
- Runs on nightly agent infrastructure (ENG-2183) once built
- Must produce actionable findings, not noise — "genuinely better" vs "just different" requires judgment
- Field doesn't have a single changelog — scattered across Anthropic docs, research, community repos, blogs, forums
- Agent doing this needs deep understanding of our harness to spot meaningful drift

## Harness Surface Area
- **Orchestration model** — main context + plan-implementer subagents, agent-to-agent patterns
- **Memory architecture** — durable/archival/distill, context management approach
- **Workflow lifecycle** — shape → plan → implement → validate loop
- **Scope control** — agent discipline, scope creep prevention rules
- **Tool/platform patterns** — Claude Code features, Agent SDK, MCP conventions, Linear integration
- **Meta patterns** — plan-as-artifact, decisions-as-docs, briefs-before-plans

## Options Considered

### Self-Audit Agent — structured review against latest sources
Nightly agent takes a watchlist of harness files, web-searches current best practices per area, compares, produces findings report ("still good" / "drifted" / "new opportunity").
- Gains: Comprehensive, systematic. Catches staleness and missed opportunities. "Last verified" dates per area.
- Costs: Broad web search is noisy. High false-positive risk. Needs well-tuned watchlist.
- Complexity: Medium

### Changelog Tracker — monitor specific sources for delta
Registry of trusted sources (Anthropic changelog, Claude Code releases, Agent SDK repo, key community repos, vetted blogs/papers). Nightly fetch + diff against last-seen state. Flags changes that might affect our harness.
- Gains: Low noise. Fast runs. Clear audit trail. Easy to add/remove sources.
- Costs: Only catches what's in registry. Misses emergent patterns from unknown sources. Flags changes but doesn't evaluate impact.
- Complexity: Low

### Adversarial Self-Review — argue against the current harness
Picks one harness area per run (rotating), reads our implementation, actively argues against it — "why might this be wrong now?", "what would someone building this today do differently?" Uses web search for evidence.
- Gains: Catches subtle structural drift, not just new features. Forces harness to defend itself.
- Costs: Highest false-positive risk. Needs human review. One area per night = slow full-cycle.
- Complexity: Medium-High

### Hybrid — changelog tracking + periodic deep review
Changelog Tracker nightly (low cost, catches concrete changes). Adversarial Self-Review weekly on rotating area (catches structural drift). Changelog findings can reprioritize which area gets the deep review.
- Gains: Fast reaction to concrete changes + deep structural questioning. Two systems reinforce each other.
- Costs: Two systems to maintain. Rotation schedule + priority override logic needed.
- Complexity: Medium

## Chosen Approach
**Hybrid** — the changelog tracker alone misses structural drift; the adversarial review alone is too slow/noisy without grounding in actual changes. Together they cover both "something changed externally" and "our assumptions may have shifted." Start with changelog tracker, layer adversarial review once nightly infrastructure is proven.

Side benefit: building the source registry forces us to curate which sources we actually trust for agentic development patterns.

## Key Context Discovered During Shaping
- The 2026-02-09 overlap analysis was a manual proof of concept — found 2 redundant improvements and 5 new opportunities in one pass
- Judge/taste agent concept (macro strategy brief) is a related pattern — "how do you prevent judge agents from going stale?" is an open question there too
- Depends on ENG-2183 (nightly agent infrastructure) for execution

## Next Step
- [Parked] → Linear backlog issue: [ENG-2225](https://linear.app/filescience/issue/ENG-2225/add-harness-self-improvement-agent-for-pattern-staleness-detection) (related to ENG-2183)
