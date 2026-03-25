---
date: 2026-03-18T14:27:28.277471+00:00
source_research: memory-bank/thoughts/shared/research/2026-03-18-test-directory-duplication.md
last_generated: 2026-03-18T14:27:28.277471+00:00
---

# Research Brief: 2026-03-18-test-directory-duplication

## TL;DR

Two separate pytest projects exist because `filescience` (the main repo) and `filescience-tools` are **separate Python packages** with different dependency sets. The tools package depends on `pydantic-ai`, `claude-agent-sdk`, `slack-sdk`, `optuna`, `typer` — none of which are in the main package. This is the fundamental reason for the split: they have separate `pyproject.toml`, separate `uv.lock`, separate venvs.

## Key facts / constraints

None captured.

## Key recommendations

None captured.

## Open questions

None

## Links

- Source research: `memory-bank/thoughts/shared/research/2026-03-18-test-directory-duplication.md`
