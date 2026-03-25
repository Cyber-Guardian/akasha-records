---
title: Claude hooks validators integration
date: 2026-02-05
status: draft
---

# Claude hooks validators integration

## Summary
- Claude Code hooks are configured in `.claude/settings.json` under `hooks` with
  event arrays like `PreToolUse` and `PostToolUse`.
- Each hook entry uses a `matcher` (regex over tool names like `Write`/`Edit`)
  and a `hooks` list. Command hooks read JSON from stdin and can block by exit
  code `2` or by returning a decision object.
- This repo already uses `PostToolUse` for `Edit|Write` to run `research_brief.py`.
  Add the validators (`ruff.py`, `ty.py`) to that same `hooks` list.

## Current repo state
- Validators live at:
  - `.claude/hooks/validators/ruff.py`
  - `.claude/hooks/validators/ty.py`
- Both scripts:
  - Parse `tool_input.file_path` from PostToolUse stdin.
  - Skip non-`.py` files.
  - Run `uvx ruff check <file>` or `uvx ty check <file>`.
  - Print a JSON `{"decision": "block", "reason": ...}` on failure.
- Existing hook config in `.claude/settings.json`:
  - `PreToolUse`: `strategic_compact_suggester.sh`
  - `PostToolUse` (`Edit|Write`): `research_brief.py`

## Recommended integration (config)
- Add the validator commands under the existing `PostToolUse` matcher
  (`Edit|Write`) so all three run after file edits/writes.
- Use `$CLAUDE_PROJECT_DIR` in commands for portability.
- Optional: add `timeout` to prevent long-running checks.
- Ensure the validator scripts are executable or invoke them via
  `uv run --script` / `python3`.

Example (conceptual):
```
"PostToolUse": [
  {
    "matcher": "Edit|Write",
    "hooks": [
      { "type": "command", "command": "python3 .../research_brief.py" },
      { "type": "command", "command": ".../validators/ruff.py", "timeout": 120 },
      { "type": "command", "command": ".../validators/ty.py", "timeout": 120 }
    ]
  }
]
```

## Notes / constraints
- Claude Code docs show multiple hooks in the same event as valid and
  independent; keep hooks fast and idempotent.
- Validators already filter non-Python files, so running them on every
  `Edit|Write` is safe.
- `uv`/`uvx` must be available on PATH for the validator scripts.

## Sources
- https://github.com/anthropics/claude-code (hook development docs, examples)
