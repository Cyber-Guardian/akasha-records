---
date: 2026-03-05T12:00:00-05:00
source_research: memory-bank/thoughts/shared/research/2026-03-05-deep-local-vs-ci-testing-divergence.md
last_generated: 2026-03-05T20:41:51.383592+00:00
---

# Research Brief: 2026-03-05-deep-local-vs-ci-testing-divergence

## TL;DR

The root cause of PR failures in this repo is **not environment divergence** — `pyproject.toml` already ensures identical tool config between local and CI, and no platform-specific test issues exist. The real problem is that **linting and tests don't run locally before push**: CI runs 7 jobs on PRs, but only 3 have automatic local triggers (ruff + import-linter via Claude Code hooks, detect-secrets via git pre-commit). The fix is a two-tier git hook expansion — `pre-commit` for fast lint/format (<1s), `pre-push` for tests + import-linter (~5s) — plus new Makefile targets (`make lint`, `make check`) that CI workflows call instead of inline commands. Expanding the existing `.githooks/` scripts is the recommended approach over adopting the `pre-commit` framework or `lefthook`, given the small team, existing infrastructure, and sub-4-second total check time.

## Key facts / constraints

None captured.

## Key recommendations

None captured.

## Open questions

- What are the actual historical CI failure rates by job type? (Would need `gh run list` analysis to quantify)
- Should `make check` be required before PR creation, or is pre-push sufficient?
- Would adopting `nektos/act` for local CI simulation add value, or is the hook approach sufficient?
- As team grows, at what point does pre-commit framework become worth the abstraction cost?

## Links

- Source research: `memory-bank/thoughts/shared/research/2026-03-05-deep-local-vs-ci-testing-divergence.md`
