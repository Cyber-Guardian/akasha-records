---
date: 2026-02-05
source_research: memory-bank/thoughts/shared/research/2026-02-05-claude-hooks-validators.md
last_generated: 2026-02-05T15:51:51.166727+00:00
---

# Research Brief: 2026-02-05-claude-hooks-validators

## TL;DR

- Claude Code hooks are configured in `.claude/settings.json` under `hooks` with
  event arrays like `PreToolUse` and `PostToolUse`.
- Each hook entry uses a `matcher` (regex over tool names like `Write`/`Edit`)
  and a `hooks` list. Command hooks read JSON from stdin and can block by exit
  code `2` or by returning a decision object.
- This repo already uses `PostToolUse` for `Edit|Write` to run `research_brief.py`.
  Add the validators (`ruff.py`, `ty.py`) to that same `hooks` list.

## Key facts / constraints

None captured.

## Key recommendations

None captured.

## Open questions

None

## Links

- Source research: `memory-bank/thoughts/shared/research/2026-02-05-claude-hooks-validators.md`
