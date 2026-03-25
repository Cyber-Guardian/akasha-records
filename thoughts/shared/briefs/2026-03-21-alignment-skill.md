---
type: event
created: 2026-03-21
status: active
---
# Idea Brief: /alignment Skill

**Date:** 2026-03-21
**Status:** Shaped -> Planning

## Problem
During complex agentic work — especially multi-session or out-loop execution — output drifts from original human intent. Scope creep, misunderstood requirements, yak shaves, and silent requirement drops compound without detection. No existing skill catches this mid-flight: `/ground` checks facts not intent, `/validate_plan` checks implementation vs plan specs but not whether the plan matches the original ask, `/wrapup` is retrospective (weekly), `/optimal` picks next actions without questioning trajectory.

## Constraints
- Must work mid-session (not just post-mortem) — lightweight enough to invoke without breaking flow
- Must trace back to an original source of intent (plan, brief, Linear issue, user's initial message)
- Should be adaptive — activate only the lenses relevant to the current context
- Must produce actionable output, not just a report
- Should work for both in-loop (codev) and out-loop (autonomous) contexts
- Must not duplicate `/validate_plan` (implementation correctness) or `/wrapup` (weekly audit)

## Options Considered

### Single-lens: Intent Trace only
Trace from original intent source forward to current state. Diff what was asked vs what exists.
- Gains: Simple, grounded in concrete source documents
- Costs: Misses dimensions like effort allocation, yak shaves, memory consistency
- Complexity: Low

### Adaptive Multi-Lens (chosen)
Run all applicable alignment lenses from a full set, synthesize into a single verdict. Lenses activated based on what's available (plan? brief? Linear issue? how much work done?).
- Gains: Comprehensive — catches drift from multiple angles. Adaptive — doesn't waste effort on irrelevant lenses. Combines automated detection with Socratic questions for ambiguous findings.
- Costs: More complex skill definition. Needs clear lens activation rules.
- Complexity: Medium

## Chosen Approach
**Adaptive Multi-Lens** — the full lens set covers complementary dimensions of alignment. Each lens is individually simple; the skill's intelligence is in selecting which to activate and synthesizing their findings.

### The Lens Set
1. **Intent Trace** — source-backward comparison of what was asked vs what exists
2. **Drift Detector** — classify changes as aligned/adjacent/tangent with a drift ratio
3. **Yak-Shave Detector** — trace dependency chains ("why are we doing this?") to surface deep yak shaves
4. **Scope Boundary Check** — files/systems touched vs expected scope from intent
5. **Effort Allocation** — where LOC/time went vs where intent says it should go
6. **Memory Consistency** — cross-reference plan/brief/issue/topic note chain for contradictions
7. **Silent Drops** — requirements in original intent never addressed and never explicitly deferred
8. **Counterfactual Test** — plain-language summary of what was built, asking "would the human recognize this?"
9. **Socratic Questions** — pointed questions for ambiguous findings that need human judgment

## Key Context Discovered During Shaping
- `/validate_plan` is the closest existing skill but is scoped to plan-vs-implementation, not intent-vs-output
- `/wrapup` dispatches parallel auditor agents per workstream — similar pattern could work for parallel lens execution
- The skill set already has a good taxonomy: `/ground` = facts, `/validate_plan` = specs, `/wrapup` = retrospective. `/alignment` = intent. Clean conceptual boundary.
- Adaptive lens selection is key — a quick mid-task check shouldn't run Memory Consistency or Effort Allocation

## Next Step
- [Plan] -> `/create_plan memory-bank/thoughts/shared/briefs/2026-03-21-alignment-skill.md`
