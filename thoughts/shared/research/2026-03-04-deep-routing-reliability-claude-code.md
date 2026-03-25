---
date: 2026-03-04T12:00:00-05:00
researcher: Claude
git_commit: 5b7c5d1
branch: main
repository: filescience
topic: "How to improve Claude Code routing reliability for keyword-triggered skill dispatch"
tags: [deep-research, claude-code, routing, hooks, skills]
status: complete
research_depth: standard
iterations_completed: 3
last_updated: 2026-03-04
last_updated_by: Claude
---

# Deep Research: Routing Reliability in Claude Code

## Research Question
How to most effectively and idiomatically improve Claude Code's routing system so that keyword triggers in user messages (like "deep research", "shape this", "debug", etc.) are reliably matched and routed to the correct skill.

## Summary
The most effective and idiomatic approach is a **UserPromptSubmit hook** that performs keyword/regex matching on the user's prompt and injects routing directives via `additionalContext`. This appears to the model as a high-salience `<system-reminder>` block right alongside the user's message — far more reliable than instructions buried in CLAUDE.md. Community data shows hook-based routing at ~84% activation vs ~60-70% for CLAUDE.md routing tables alone. The CLAUDE.md routing table should remain as documentation and semantic fallback for intent-based routing that exact keywords can't capture.

## Perspectives Explored
1. **Claude Code architecture** — Revealed UserPromptSubmit hook as the native mechanism for pre-processing user input, with additionalContext injection as a system-reminder block
2. **Cognitive/attention patterns** — Confirmed lost-in-the-middle effects for instructions, shorter isolated blocks beat monolithic ones, XML-tagged explanations outperform terse directives
3. **Community patterns** — Established hook-based routing as the community-validated pattern (~84% activation), with concrete open-source examples
4. **Our current system** — Identified the gap: no UserPromptSubmit hook exists despite a well-structured hooks setup in settings.json

## Detailed Findings

### The Mechanism: UserPromptSubmit Hook

Claude Code's `UserPromptSubmit` hook fires after the user submits a prompt but **before the model processes it**. It:
- Receives JSON on stdin with a `prompt` field containing the raw user text
- Can inject context via `additionalContext` (appears as `<system-reminder>` — high salience, discrete)
- Can also emit plain stdout (visible in transcript) or block with `decision: "block"`
- Has no matcher support — fires on every prompt submission
- Cannot modify the prompt itself — only append context

This is the idiomatic mechanism for keyword-triggered routing. The hook script reads the prompt, matches against a keyword/regex table, and emits additionalContext telling Claude which skill to invoke.

**Key technical detail:** `additionalContext` appears as a `<system-reminder>` block. This is the highest-salience injection point available — it's processed alongside the user's message, not buried in session-start context. XML-tagged longer explanations (not terse directives) work best per community testing.

**Known issue:** GitHub issue #17550 reports JSON output may error on first message. Plain stdout is a safer fallback, though less discrete.

Sources:
- https://code.claude.com/docs/en/hooks.md
- https://github.com/anthropics/claude-code/issues/17550
- https://github.com/anthropics/claude-code/issues/14281

### Why CLAUDE.md Routing Tables Miss

The "lost in the middle" effect (Liu et al., 2024 — TACL) primarily affects retrieval but spills into instruction following. The "Instruction Gap" paper (2025, arXiv 2601.03269) found three failure modes for instructions in long contexts: repetition, irrelevance, and non-compliance — all tied to attention dilution over length.

Our CLAUDE.md is 144 lines with the routing table comprising ~70 lines (~half the file). While under the recommended 200-line limit, the routing table competes with constraints, reference links, and other instructions for attention. The table format (trigger → route → skill) is dense and requires the model to scan, match, and dispatch — a multi-step cognitive task that's vulnerable to attention dilution.

Anthropic's own best practices recommend: XML tags for semantic separation, named directive blocks, rationale alongside imperatives, and shorter isolated instruction blocks over monolithic ones.

Sources:
- https://aclanthology.org/2024.tacl-1.9/
- https://arxiv.org/html/2601.03269
- https://research.trychroma.com/context-rot
- https://platform.claude.com/docs/en/build-with-claude/prompt-engineering/claude-4-best-practices

### Community Validation

The community has converged on two complementary approaches:
1. **Semantic routing** via skill description clarity ("Use when" fields in SKILL.md)
2. **Hook-based forced dispatch** via UserPromptSubmit keyword matching + additionalContext

CLAUDE.md routing tables are characterized as "Level 2" activation (~60-70% reliable). Hook-based routing reaches ~84% activation. The combination of both approaches is the recommended pattern.

Concrete examples:
- ChrisWiles/claude-code-showcase — skill routing via UserPromptSubmit
- severity1/claude-code-prompt-improver — prompt rewriting via hooks
- disler/claude-code-hooks-mastery — comprehensive hook patterns
- claudefa.st/blog/tools/hooks/skill-activation-hook — documented pattern

Sources:
- https://claudefa.st/blog/tools/hooks/skill-activation-hook
- https://github.com/ChrisWiles/claude-code-showcase
- https://github.com/disler/claude-code-hooks-mastery

### Recommended Architecture: Layered Routing

The idiomatic solution is a three-layer system:

**Layer 1 — UserPromptSubmit hook (highest reliability)**
A Python script that reads the prompt from stdin JSON, matches against a keyword/regex table, and emits additionalContext with the routing directive. Handles exact phrase triggers like "deep research", "shape this", "debug", "create an issue".

**Layer 2 — CLAUDE.md routing table (semantic fallback)**
Remains as documentation and handles intent-based routing for ambiguous cases that exact keywords can't capture. The table format stays, but it's no longer the primary dispatch mechanism.

**Layer 3 — Skill "Use when" descriptions (last resort)**
SKILL.md files describe when each skill should be used. This catches cases where neither the hook nor the routing table triggers.

### Implementation Shape

The hook script should:
- Be a Python script (consistent with existing hooks in `.claude/hooks/`)
- Read stdin JSON, extract `prompt` field
- Match against a config file (`.claude/hooks/skill-routes.json`) mapping patterns to skills
- Emit `additionalContext` with XML-tagged routing directive when matched
- Be fast (< 100ms) since it fires on every prompt
- Support both exact phrases and regex patterns
- Include the skill name AND the "Use when" rationale in the additionalContext (per cognitive research: rationale alongside directives improves following)

The config file allows adding/modifying routes without editing the hook script.

## Key Sources

### Codebase
- `CLAUDE.md` — current routing table (144 lines)
- `.claude/settings.json` — hooks configuration
- `.claude/hooks/` — existing hook scripts (scope_guard, validators, session_start)
- `.claude/rules/` — 7 path-scoped rules files

### External Documentation
- https://code.claude.com/docs/en/hooks.md — Hook lifecycle, UserPromptSubmit spec
- https://code.claude.com/docs/en/hooks-guide.md — Hook automation patterns
- https://code.claude.com/docs/en/memory.md — CLAUDE.md and rules loading
- https://platform.claude.com/docs/en/build-with-claude/prompt-engineering/claude-4-best-practices

### Research
- Liu et al., 2024 — "Lost in the Middle" (TACL) — https://aclanthology.org/2024.tacl-1.9/
- "The Instruction Gap" (2025) — https://arxiv.org/html/2601.03269
- "Context Rot" (Chroma Research) — https://research.trychroma.com/context-rot

### Community Examples
- https://claudefa.st/blog/tools/hooks/skill-activation-hook
- https://github.com/ChrisWiles/claude-code-showcase
- https://github.com/severity1/claude-code-prompt-improver
- https://github.com/disler/claude-code-hooks-mastery
- https://gist.github.com/johnlindquist/23fac87f6bc589ddf354582837ec4ecc

## Open Questions
- What's the optimal additionalContext format? Community leans toward XML-tagged explanations, but our codebase uses `<system-reminder>` style — should the hook's output complement or layer on top of that?
- Should the hook emit plain stdout (safer, per issue #17550) or JSON with additionalContext (more discrete)?
- How to handle partial matches and ambiguous triggers without over-routing?
- Should missed routes be logged for iterative improvement of the keyword table?

## Research Metadata
- Depth mode: standard
- Iterations completed: 3 / 5
- Termination reason: gaps exhausted (all active gaps closed, source overlap at ~40%)
- Manifest: `.claude/deep-research/2026-03-04-routing-reliability-claude-code.md`
