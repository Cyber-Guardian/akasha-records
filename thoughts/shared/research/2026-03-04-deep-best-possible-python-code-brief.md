---
date: 2026-03-04T12:00:00-05:00
source_research: memory-bank/thoughts/shared/research/2026-03-04-deep-best-possible-python-code.md
last_generated: 2026-03-05T02:45:25.632560+00:00
---

# Research Brief: 2026-03-04-deep-best-possible-python-code

## TL;DR

Our codebase already has strong mechanical enforcement (ruff ALL, ty, import-linter as PostToolUse hooks) but zero design-level guidance for Claude Code. The codebase itself follows excellent Python patterns — frozen dataclasses, Protocol for subtyping, pathlib, StrEnum, modern typing, anyio TaskGroup — but these conventions are nowhere codified as instructions Claude can follow. The highest-leverage intervention is a **Python excellence rule file** that codifies existing codebase idioms with concrete file references, supplemented by an **adversarial Python reviewer agent** and an enhanced **/simplify skill**. Community evidence (Arize +10.87% accuracy, Trail of Bits config) confirms that project-specific prose guidance with real examples outperforms generic rules.

## Key facts / constraints

None captured.

## Key recommendations

None captured.

## Open questions

- Should we enable `radon` complexity checking as a PostToolUse hook or keep it as a soft guideline in the rule file?
- Should the python-reviewer agent run automatically (Stop hook) or only when invoked via /simplify?
- How do we handle the tension between the rule file's design guidance and rapid prototyping (where you want to move fast and refine later)?
- Should we create a correction-capture mechanism (claude-reflect style) or is /learn sufficient?
- Would few-shot examples of good vs bad code in the rule file improve output, or would they bloat context?

## Links

- Source research: `memory-bank/thoughts/shared/research/2026-03-04-deep-best-possible-python-code.md`
