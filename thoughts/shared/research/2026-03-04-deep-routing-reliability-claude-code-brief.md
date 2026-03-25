---
date: 2026-03-04T12:00:00-05:00
source_research: memory-bank/thoughts/shared/research/2026-03-04-deep-routing-reliability-claude-code.md
last_generated: 2026-03-04T19:27:19.470112+00:00
---

# Research Brief: 2026-03-04-deep-routing-reliability-claude-code

## TL;DR

The most effective and idiomatic approach is a **UserPromptSubmit hook** that performs keyword/regex matching on the user's prompt and injects routing directives via `additionalContext`. This appears to the model as a high-salience `<system-reminder>` block right alongside the user's message — far more reliable than instructions buried in CLAUDE.md. Community data shows hook-based routing at ~84% activation vs ~60-70% for CLAUDE.md routing tables alone. The CLAUDE.md routing table should remain as documentation and semantic fallback for intent-based routing that exact keywords can't capture.

## Key facts / constraints

None captured.

## Key recommendations

None captured.

## Open questions

- What's the optimal additionalContext format? Community leans toward XML-tagged explanations, but our codebase uses `<system-reminder>` style — should the hook's output complement or layer on top of that?
- Should the hook emit plain stdout (safer, per issue #17550) or JSON with additionalContext (more discrete)?
- How to handle partial matches and ambiguous triggers without over-routing?
- Should missed routes be logged for iterative improvement of the keyword table?

## Links

- Source research: `memory-bank/thoughts/shared/research/2026-03-04-deep-routing-reliability-claude-code.md`
