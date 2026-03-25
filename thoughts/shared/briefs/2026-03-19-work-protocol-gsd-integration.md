---
type: event
created: 2026-03-19
status: active
---
# Idea Brief: Work Protocol — GSD-2 Integration

**Date:** 2026-03-19
**Status:** Shaped -> Planning

## Problem
Our system lacks a unified work ontology. The concept of "a unit of work" is fragmented across `/create_plan`, `plan_parser.py`, `decompose.py`, `manifest.py`, and `plan-implementer.md`. Each participant has a partial, incompatible understanding of what work is. Downstream agents start blind. Verification is freeform. Phases have no size constraint.

GSD-2 has solved this with a proven protocol: 3-level hierarchy, boundary maps, structured must-haves, task summaries, and context-window sizing.

## Constraints
- Protocol must not be coupled to Polylith — it's infrastructure, not application code
- Must work for both helm-orchestrated AND single-session flows
- Must be expressible in markdown and parseable
- Helm's novel capabilities (adversarial review, merge pipeline, Linear integration) must be preserved

## Chosen Approach
**Protocol Layer in `.work/`** — self-contained dot-directory with spec, Python models, parser, templates. Hard cutover — no legacy format support.

## Next Step
- [Plan] -> `/create_plan memory-bank/thoughts/shared/briefs/2026-03-19-work-protocol-gsd-integration.md`
