---
date: 2026-03-05T18:00:00-05:00
researcher: Claude
git_commit: 4b407a5
branch: main
repository: filescience
topic: "Improve the deep research system — explore current implementation, identify weaknesses, source ideas for enhancements"
tags: [deep-research, meta-improvement, agent-architecture, context-management, research-methodology]
status: complete
research_depth: standard
iterations_completed: 3
last_updated: 2026-03-05
last_updated_by: Claude
---

# Deep Research: Improving the Deep Research System

## Research Question
Improve the deep research system — explore current implementation, identify weaknesses, source ideas for enhancements

## Summary
The deep research system's gap-chase loop with manifest externalization is structurally sound but has six diagnosed weaknesses: (1) all sessions converge at 3 iterations regardless of depth budget, making the depth setting decorative; (2) gaps are binary (open/closed) with no confidence gradient, so single-source findings are treated as settled; (3) there is no self-critique or verification pass before synthesis, making the system good at finding answers but bad at challenging them; (4) QMD semantic search is never called directly, missing a cheap pre-search that could eliminate 25-50% of first-iteration subagent spawns; (5) the every-2-iteration steering checkpoint is over-engineered vs industry consensus of plan-approve-then-autonomous; (6) Route A and Route K share agent infrastructure but have no unified skill architecture. The highest-leverage improvements are adding a verification step before synthesis (inspired by DeepVerifier/CRITIC), integrating QMD pre-search, and removing mid-run checkpoints. Longer-term, the linear gap-chase could evolve toward tree-shaped exploration with confidence-weighted path selection (MCTS-RAG pattern).

## Perspectives Explored
1. **Structural weaknesses** — Revealed that 15/17 manifests terminate at iteration 3 (gap exhaustion), binary gap tracking hides finding quality, and no self-critique exists before synthesis.
2. **Context economics** — Quantified 450K-750K tokens per session in subagent overhead; identified QMD pre-search as saving 150K-300K tokens; confirmed distillation protocol as the critical context lever.
3. **SOTA iterative research patterns** — Catalogued tree search (MCTS-RAG), backtracking (OpenAI), self-critique (Google Gemini), and graduated confidence (Self-RAG/CRAG) as structural innovations absent from our linear design.
4. **User experience & steering** — Industry consensus is plan-approve-then-autonomous; our every-2-iteration checkpoint adds latency without clear value.
5. **Integration & composability** — Route A/K differ on scope/intent/infrastructure (not just depth); unification viable as mode flag; QMD never called directly; brief hook works correctly; Step 6 journal has no enforcement.

## Detailed Findings

### 1. Structural Weaknesses — The 3-Iteration Ceiling

Analysis of all 17 manifests reveals a consistent pattern: sessions terminate at iteration 3 regardless of configured depth (standard=5, deep=8). Gap exhaustion is the universal trigger — source saturation (>50%) never fires. The gap trajectory is monotonically decreasing: 4-5 gaps close vs 1-3 open per iteration, converging to zero by iteration 3.

**Root cause**: The system closes gaps on first encounter. If a subagent returns *any* answer, the gap is marked `[x]`. There's no mechanism to reopen gaps when later evidence contradicts earlier findings, no confidence gradient to prioritize weak findings for re-investigation, and no adversarial pass to challenge accumulated conclusions.

**What SOTA does differently**:
- **OpenAI deep research**: RL-trained backtracking — the model learns to pivot from dead ends through reinforcement, not hand-coded rules
- **MCTS-RAG** (Yale, EMNLP 2025): Monte Carlo Tree Search over reasoning paths with UCT scoring (Q̄ + exploration bonus). Nodes = reasoning states, scored by visit count and quality. Enables dynamic branching, rollback, and confidence-weighted path selection. 7B models match GPT-4o on knowledge-intensive tasks
- **Self-RAG**: Emits reflection tokens (IsREL, IsSUP, IsUSE) per generation segment with configurable weights forming continuous quality scores
- **CRAG**: T5-large evaluator classifies retrieval into Correct/Incorrect/Ambiguous. "Ambiguous" triggers deeper investigation rather than accepting

**Proposed fix — Graduated gap confidence**:
Replace binary `- [ ]`/`- [x]` with a confidence lifecycle:
```
open → tentative (single source) → corroborated (multi-source agreement) → closed (high confidence)
                                  → contradicted (conflicting evidence) → requires re-investigation
                                  → superseded (new evidence invalidates prior finding)
```
Gaps at "tentative" are reprioritized for corroboration in subsequent iterations. This alone would break the 3-iteration ceiling by generating meaningful work beyond iteration 3.

Sources:
- `.claude/deep-research/*.md` — all 17 manifests analyzed
- [MCTS-RAG](https://arxiv.org/html/2503.20757v1) — tree search over reasoning paths
- [Self-RAG](https://arxiv.org/abs/2310.11511) — reflection tokens for quality scoring
- [CRAG](https://arxiv.org/abs/2401.15884) — retrieval confidence classification

### 2. Missing Verification Pass — The Biggest Quality Gap

The system accumulates findings across iterations and synthesizes them in Step 5 without ever challenging them. No contradiction detection, no source triangulation, no adversarial review. This is the biggest quality gap vs every major SOTA system.

**What SOTA does**:
- **DeepVerifier** (arxiv 2601.15808): Decomposes synthesis into targeted sub-questions, runs verification agent against external evidence, scores 1-4 before accepting
- **Google AI co-scientist**: Reflection Agent + tournament ranking — hypotheses compete in pairwise Elo debates, each recursively broken into sub-assumptions for adversarial review
- **Self-consistency sampling**: Generates N independent synthesis passes, flags claims with low inter-generation agreement as hallucination candidates
- **CRITIC** (arxiv 2305.11738): Critique agent actively retrieves counter-evidence for each major claim post-synthesis and revises accordingly
- **Source triangulation threshold**: 3+ independent sources before treating a claim as reliable (empirical norm in RAG literature)

**Proposed fix — Add Step 4.5: Verification Pass**:
After the research loop completes but before synthesis:
1. Extract the top 5-8 claims from accumulated findings
2. Dispatch a verification agent per claim that searches for *counter-evidence*
3. Flag claims with conflicting evidence or single-source backing
4. Reopen gaps for flagged claims if iteration budget remains
5. Mark unverified claims in the synthesis with explicit confidence notes

This is the single highest-impact improvement. It transforms the system from "efficient finder" to "reliable researcher."

Sources:
- [DeepVerifier](https://arxiv.org/html/2601.15808)
- [Google AI co-scientist](https://research.google/blog/accelerating-scientific-breakthroughs-with-an-ai-co-scientist/)
- [CRITIC](https://arxiv.org/abs/2305.11738)
- [Multi-agent hallucination mitigation](https://www.mdpi.com/2078-2489/16/7/517)

### 3. Context Economics — Where Budget Leaks

**Cost hierarchy per session** (typical standard-depth, 3 iterations):

| Component | Tokens | Controllable? |
|---|---|---|
| Subagent spawns (9-15 × 50K baseline) | 450K-750K | Yes — QMD pre-search, model discipline |
| Manifest re-reads (3 × 2K-4.5K) | 6K-13.5K | Partially — manifest compression |
| Manifest write-backs (3 × growing) | 3K-9K | No — structural requirement |
| Distilled returns (9-15 × 200 tokens) | 1.8K-3K | No — already optimized |
| Orchestration overhead | ~5K | No |

**Subagents are 95%+ of the cost**. The distillation protocol is the critical lever — without it, returns would be 600K+ instead of 3K. Manifest costs are noise.

**QMD pre-search could save 150K-300K tokens per session**: With ~55 unique research docs in `memory-bank/`, running `vector_search` (~2s) per gap before dispatch could pre-resolve 3-6 gaps already covered by prior research. Design: score > 0.7 → extract finding directly (no agent spawn); score 0.4-0.7 → inject prior research as context into subagent prompt.

Sources:
- `.claude/commands/deep_research.md:160-176` — dispatch budget logic
- [[2026-03-04-deep-context-window-optimization-token|Context Window Optimization Research]] — confirms 50K baseline per spawn
- `.claude/rules/qmd-usage.md` — QMD tool selection

### 4. UX — Checkpoint Over-Engineering

**Industry consensus**: Every major deep research system uses plan-approve-then-autonomous:
- **Google Gemini**: Structured plan → user approve/modify → autonomous execution to completion
- **Perplexity**: Fire-and-forget + streaming progress disclosure. Post-hoc follow-ups, not mid-run redirects
- **OpenAI**: Fully autonomous. Submit query, wait 5-30 min, get report

Our every-2-iteration checkpoint adds latency and decision fatigue without evidence of value. The plan approval gate (Step 1) already provides the critical user input moment.

**Proposed fix**: Remove Step 4 (steering checkpoints). Replace with:
1. Keep Step 1 plan approval (this is the high-value checkpoint)
2. Print 1-sentence progress notes between iterations: `[2/5] Investigated 3 gaps, found... Next: ...`
3. Support post-hoc follow-up via `/deep_research resume`
4. Use the statusline for persistent metrics (subagent count, gaps remaining, iteration N/max)

**Terminal UX patterns**: Claude Code streams text + inline tool blocks. Best approach is structured ASCII headers as plain streamed text: `[1/4] Searching: topic A... → Done: 3 findings`. The statusline can show persistent session-level metrics.

Sources:
- [Gemini Deep Research](https://gemini.google/overview/deep-research/)
- [Perplexity Deep Research](https://www.perplexity.ai/hub/blog/introducing-perplexity-deep-research)
- [OpenAI Deep Research](https://openai.com/index/introducing-deep-research/)
- [ZenML Steerable Deep Research](https://www.zenml.io/blog/steerable-deep-research-building-production-ready-agentic-workflows-with-controlled-autonomy)
- [CLI UX patterns](https://evilmartians.com/chronicles/cli-ux-best-practices-3-patterns-for-improving-progress-displays)

### 5. Integration — QMD Gap and Route Unification

**QMD is never called directly** in the deep research loop. All search goes through subagents, which is expensive. A `vector_search` call takes ~2s and costs zero subagent overhead vs 50K tokens per subagent spawn.

**Proposed fix — Step 3b.5: QMD Pre-search**:
```
For each selected gap:
  1. Run mcp__qmd__vector_search(gap text, collection="memory-bank", minScore=0.4)
  2. If match score > 0.7: mcp__qmd__get(doc) → extract finding → mark gap "pre-resolved"
  3. If match score 0.4-0.7: inject doc summary into subagent prompt as prior context
  4. If no match: proceed with normal agent dispatch
```

**Route A / Route K unification**: Both skills use identical agent taxonomy (codebase-locator, codebase-analyzer, thoughts-locator, etc.) but differ on 3 axes:
- **Scope**: Route A = internal codebase. Route K = multi-perspective including external
- **Intent**: Route A = "documentarian only" (describe, don't evaluate). Route K = evaluative understanding
- **Infrastructure**: Route A = no manifest, single-pass. Route K = manifest, iterative, gap tracking

Unification is viable as a **mode flag** — Route A becomes `shallow` mode (1 iteration, no manifest, documentarian constraint preserved). Route K remains `standard`/`deep`. The "documentarian only" constraint is architecturally meaningful and must be preserved as an explicit mode, not just a budget setting.

Sources:
- `.claude/commands/research_codebase.md:10-17` — documentarian constraint
- `.claude/commands/deep_research.md:344` — boundary statement
- `.claude/rules/qmd-usage.md:14-20` — tool selection table

### 6. Depth Budget — From Iterations to Outcomes

The current depth budget (shallow=3, standard=5, deep=8 iterations) is decorative — all sessions converge at 3. Alternative models from the literature:

- **BATS**: SUCCESS/CONTINUE/PIVOT outcomes per verification step. Terminates on SUCCESS without consuming remaining budget. Budget state machine: HIGH→MEDIUM→LOW→CRITICAL
- **CoRL**: RL-trained dual-reward controller (performance × cost-compliance). Distinct learned behaviors per budget tier
- **FlashResearch**: Adaptive breadth/depth — expands breadth after initial phases, only increases depth when promising threads emerge post-search
- **TALE**: Token-budget-aware reasoning — preset complexity-calibrated token budgets with "token elasticity" as natural stopping floor

**Proposed fix**: Replace iteration count with a **quality-aware stopping criterion**:
1. After each iteration, count "tentative" vs "corroborated" findings
2. If all findings are corroborated and no new gaps: STOP (quality satisfied)
3. If tentative findings remain and budget allows: continue (quality unsatisfied)
4. Depth modes set the **corroboration threshold**, not iteration count:
   - `shallow`: accept tentative findings (fast, 1-source answers OK)
   - `standard`: require corroboration for top claims (2+ sources)
   - `deep`: require corroboration for all claims + run verification pass

Sources:
- [BATS](https://arxiv.org/html/2511.17006v1)
- [CoRL](https://arxiv.org/html/2511.02755v1)
- [FlashResearch](https://arxiv.org/html/2510.05145v1)
- [TALE](https://arxiv.org/html/2412.18547v5)
- [Deep Research Agents survey](https://arxiv.org/html/2506.18096v1)

### Cross-cutting Patterns

**The improvement priority stack** (ordered by impact/effort ratio):

| Priority | Improvement | Impact | Effort | Why |
|---|---|---|---|---|
| 1 | Add verification step before synthesis | High — transforms quality | Medium — ~50 LOC in skill | Missing self-critique is the biggest quality gap vs SOTA |
| 2 | QMD pre-search before subagent dispatch | High — saves 150K-300K tokens/session | Low — ~20 lines in skill | Pure cost savings, no structural change |
| 3 | Remove mid-run checkpoints → plan-approve-only | Medium — reduces latency/friction | Low — delete Step 4 | Aligns with industry consensus |
| 4 | Graduated gap confidence (tentative→corroborated→closed) | High — breaks 3-iteration ceiling | Medium — requires gap format change | Addresses root cause of shallow convergence |
| 5 | Progressive disclosure UX (per-iteration progress notes) | Low-Medium — better user experience | Low — add print statements | Replaces checkpoint with lighter-weight visibility |
| 6 | Route A/K mode unification | Low — architectural cleanliness | Medium — refactor two skills | Reduces maintenance surface, not user-facing |

**What to keep unchanged**:
- Distillation protocol — the most valuable innovation, makes the system context-viable
- Manifest externalization — sound design that survives context compression
- Perspective enumeration — drives diversity on focused technical topics
- Brief hook integration — works correctly as-is

## Key Sources

### Codebase files
- `.claude/commands/deep_research.md` — skill definition (345 lines)
- `.claude/commands/research_codebase.md` — Route A skill for comparison
- `.claude/hooks/research_brief.py` — auto-brief generation hook
- `.claude/rules/qmd-usage.md` — QMD tool selection guide
- `.claude/deep-research/*.md` — 17 manifests analyzed
- `CLAUDE.md` — routing table (Routes A and K)

### Memory-bank docs
- [[2026-03-04-deep-context-window-optimization-token|Context Window Optimization Research]] — token cost baselines

### Academic papers
- [MCTS-RAG](https://arxiv.org/html/2503.20757v1) — Monte Carlo Tree Search over reasoning paths (Yale, EMNLP 2025)
- [Self-RAG](https://arxiv.org/abs/2310.11511) — reflection tokens for graduated quality scoring (ICLR 2024)
- [CRAG](https://arxiv.org/abs/2401.15884) — corrective retrieval with confidence classification
- [Adaptive-RAG](https://aclanthology.org/2024.naacl-long.389/) — complexity-based routing (NAACL 2024)
- [DeepVerifier](https://arxiv.org/html/2601.15808) — inference-time verification for research agents
- [CRITIC](https://arxiv.org/abs/2305.11738) — tool-interactive self-critiquing
- [BATS](https://arxiv.org/html/2511.17006v1) — budget-aware tool-use with SUCCESS/CONTINUE/PIVOT
- [CoRL](https://arxiv.org/html/2511.02755v1) — RL budget controllers for multi-agent systems
- [FlashResearch](https://arxiv.org/html/2510.05145v1) — adaptive breadth/depth orchestration
- [TALE](https://arxiv.org/html/2412.18547v5) — token-budget-aware reasoning
- [Deep Research Agents survey](https://arxiv.org/html/2506.18096v1) — comprehensive survey and roadmap
- [Comprehensive Deep Research survey](https://arxiv.org/html/2506.12594v1) — systems, methodologies, applications

### Industry systems
- [Anthropic multi-agent research](https://www.anthropic.com/engineering/multi-agent-research-system) — Opus lead + Sonnet workers
- [OpenAI deep research](https://openai.com/index/introducing-deep-research/) — RL-trained backtracking
- [Perplexity Deep Research](https://www.perplexity.ai/hub/blog/introducing-perplexity-deep-research) — recursive search trees
- [Google Gemini Deep Research](https://ai.google.dev/gemini-api/docs/deep-research) — self-critique + plan approval
- [Google AI co-scientist](https://research.google/blog/accelerating-scientific-breakthroughs-with-an-ai-co-scientist/) — tournament-ranked hypothesis verification
- [ZenML Steerable Deep Research](https://www.zenml.io/blog/steerable-deep-research-building-production-ready-agentic-workflows-with-controlled-autonomy) — controlled autonomy patterns

### UX references
- [CLI UX best practices](https://evilmartians.com/chronicles/cli-ux-best-practices-3-patterns-for-improving-progress-displays) — progress display patterns
- [Claude Code statusline](https://code.claude.com/docs/en/statusline) — persistent status bar
- [claude-esp](https://github.com/phiat/claude-esp) — stream hidden output to separate terminal

## Open Questions
- How would the verification step interact with the iteration budget? Should verification agents count against the budget or be "free"?
- What's the right corroboration threshold for `standard` depth — 2 sources or 3?
- Could graduated gap confidence be represented compactly enough to not bloat the manifest? (e.g., `- [~] gap — confidence: 0.6 — sources: 2` vs the current `- [x]`)
- Would tree-shaped exploration (MCTS-RAG) add enough value over the simpler graduated-confidence fix to justify the implementation complexity?
- How should the system handle contradictions — flag them for user judgment, or attempt automated resolution?
- If Route A/K unify, what happens to existing Route A research docs that lack manifests?

## Research Metadata
- Depth mode: standard
- Iterations completed: 3 / 5
- Termination reason: gaps exhausted (all 11 gaps closed by iteration 3)
- Manifest: `.claude/deep-research/2026-03-05-improve-the-deep-research.md`
