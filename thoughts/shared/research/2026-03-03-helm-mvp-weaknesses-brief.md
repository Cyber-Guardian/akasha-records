---
date: 2026-03-03T17:43:04.316205+00:00
source_research: memory-bank/thoughts/shared/research/2026-03-03-helm-mvp-weaknesses.md
last_generated: 2026-03-03T17:43:04.316205+00:00
---

# Research Brief: 2026-03-03-helm-mvp-weaknesses

## TL;DR

**Date:** 2026-03-03
**Context:** Post-MVP analysis of Helm V1 execution layer (Phases 1-5 merged to main)
---
## Critical (could cause wrong behavior silently)
### W1. Dual Source of Truth
**Where:** CLI uses local `.claude/helm/*.json` manifests (gitignored). GH Action uses Linear.
**Risk:** Drift between the two. Manual Linear edits aren't reflected in manifests. Manifest deletion (worktree cleanup, machine switch) loses merge order and blocking info.
**Impact:** Merge order could be wrong; status dashboard shows stale data.
### W2. Silent Error Swallowing
**Where:**

## Key facts / constraints

None captured.

## Key recommendations

None captured.

## Open questions

None

## Links

- Source research: `memory-bank/thoughts/shared/research/2026-03-03-helm-mvp-weaknesses.md`
