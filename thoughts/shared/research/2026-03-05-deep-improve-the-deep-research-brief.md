---
date: 2026-03-05T18:00:00-05:00
source_research: memory-bank/thoughts/shared/research/2026-03-05-deep-improve-the-deep-research.md
last_generated: 2026-03-06T01:10:12.399709+00:00
---

# Research Brief: 2026-03-05-deep-improve-the-deep-research

## TL;DR

The deep research system's gap-chase loop with manifest externalization is structurally sound but has six diagnosed weaknesses: (1) all sessions converge at 3 iterations regardless of depth budget, making the depth setting decorative; (2) gaps are binary (open/closed) with no confidence gradient, so single-source findings are treated as settled; (3) there is no self-critique or verification pass before synthesis, making the system good at finding answers but bad at challenging them; (4) QMD semantic search is never called directly, missing a cheap pre-search that could eliminate 25-50% of first-iteration subagent spawns; (5) the every-2-iteration steering checkpoint is over-engineered vs industry consensus of plan-approve-then-autonomous; (6) Route A and Route K share agent infrastructure but have no unified skill architecture. The highest-leverage improvements are adding a verification step before synthesis (inspired by DeepVerifier/CRITIC), integrating QMD pre-search, and removing mid-run checkpoints. Longer-term, the linear gap-chase could evolve toward tree-shaped exploration with confidence-weighted path selection (MCTS-RAG pattern).

## Key facts / constraints

None captured.

## Key recommendations

None captured.

## Open questions

- How would the verification step interact with the iteration budget? Should verification agents count against the budget or be "free"?
- What's the right corroboration threshold for `standard` depth — 2 sources or 3?
- Could graduated gap confidence be represented compactly enough to not bloat the manifest? (e.g., `- [~] gap — confidence: 0.6 — sources: 2` vs the current `- [x]`)
- Would tree-shaped exploration (MCTS-RAG) add enough value over the simpler graduated-confidence fix to justify the implementation complexity?
- How should the system handle contradictions — flag them for user judgment, or attempt automated resolution?
- If Route A/K unify, what happens to existing Route A research docs that lack manifests?

## Links

- Source research: `memory-bank/thoughts/shared/research/2026-03-05-deep-improve-the-deep-research.md`
