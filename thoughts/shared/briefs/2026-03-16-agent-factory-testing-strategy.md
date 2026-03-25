---
type: event
created: 2026-03-16
status: active
---
# Idea Brief: Agent Factory Testing Strategy

**Date:** 2026-03-16
**Status:** Shaped → Planning
**Linear:** ENG-2414

## Problem

Agents can't be trusted to ship code autonomously because the gates that exist are all deterministic/static (ruff, import-linter, type checking). The gates that catch semantic regressions — mutation, contracts, state machine exploration — are either missing or advisory-only. Without machine-verifiable correctness, every agent PR requires human review, defeating the autonomy goal.

## Chosen Approach

**Incremental Mutation Gate** with **Strict Red/Green for Pass 1 bootstrap**. Diff-scoped mutation testing on changed lines as a PR gate (70% kill rate). Full nightly as trend/ratchet. Protocol-based contracts at cross-brick boundaries. Human writes red tests for Pass 1, agents graduate to test authorship in Pass 2+.

## References

- [[2026-03-09-platform-v2-roadmap-strategy|Platform V2 Roadmap Strategy]]
- [[2026-03-11-testing-infrastructure-portfolio|Testing Infrastructure Portfolio Brief]]
- [[2026-03-13-test-speed-optimization|Test Speed Optimization Plan]]
