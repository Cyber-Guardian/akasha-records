---
date: 2026-02-06T15:25:08-05:00
author: Claude
git_commit: 536084d5a95139db7195db41659feba5aa79504e
branch: main
repository: filescience
topic: "mutmut Mutation Testing Setup — Segfault Investigation Handoff"
tags: [handoff, implementation, mutmut, mutation-testing, backpressure, segfault]
status: complete
last_updated: 2026-02-06
last_updated_by: Claude
---

# Handoff: mutmut configured but segfaults need investigation

## Task(s)

- **mutmut mutation testing setup** — COMPLETE (config, Makefile, CI workflow)
- **segfault investigation** — NOT STARTED (all 3,130 tested mutants segfault)

## Critical References

- Research: `memory-bank/thoughts/shared/research/2026-02-06-ai-agent-backpressure-tools.md` — full tool landscape
- Brief: `memory-bank/thoughts/shared/research/2026-02-06-ai-agent-backpressure-tools-brief.md`
- Auto memory patterns: `~/.claude/projects/.../memory/patterns.md` — section "mutmut with Polylith"

## Recent changes

All changes are **uncommitted** on `main`. Key files modified:

- `pyproject.toml:57` — added `mutmut>=3.4.0` to dev deps
- `pyproject.toml:254-262` — added `[tool.mutmut]` section with `paths_to_mutate = ["src/filescience/"]`, `tests_dir = ["test/"]`
- `Makefile:1` — added phony targets `_mutmut-setup _mutmut-teardown mutmut mutmut-results mutmut-html`
- `Makefile:175-201` — full mutation testing section with src/ copy tree builder
- `.github/workflows/mutation.yml` — nightly cron + manual dispatch workflow with cache
- `.gitignore` — added `.mutmut-cache`, `html/`, `src/`, `mutants/`
- `memory-bank/durable/01-active/current_work.md` — updated with mutmut status
- `memory-bank/durable/01-active/next_up.md` — marked mutmut done, noted segfault issue

## Learnings

### Polylith + mutmut path mismatch (SOLVED)

mutmut derives mutant names from file paths. With `paths_to_mutate = ["components/filescience/"]`, mutant keys become `components.filescience.dynamodb.delete.x_...` but Python's import system (via `sys.path`) resolves modules as `filescience.dynamodb.delete`. The trampoline's `record_trampoline_hit(orig.__module__ + '.' + orig.__name__)` uses the Python module name, so the lookup at `mutmut.tests_by_mangled_function_name.get(mangled_name)` fails — zero tests mapped.

**Fix**: `make mutmut` builds a flat copy tree at `src/filescience/` merging both `components/filescience/*/` and `bases/filescience/*/`. mutmut's hardcoded `strip_prefix('src.')` (line 285 of `__main__.py`) then produces `filescience.dynamodb.delete` — matching Python's module names.

### mutmut 3.x gotchas

- `tests_dir` must be a **list** `["test/"]` — a string gets iterated char-by-char at line 432
- `runner` config is a TODO (line 1029) — always uses `PytestRunner()` calling `pytest.main()` directly
- No `--no-progress`, `--CI`, or `--paths-to-mutate` CLI flags exist in 3.x
- `os.walk()` default `followlinks=False` — symlinks don't work, must use hard copies

### The segfault problem (UNSOLVED — THIS IS THE CONCERN)

**Symptom**: All 3,130 mutants that had mapped tests resulted in `worker exit code -11` (SIGSEGV). Zero mutants were killed or survived. The mutation score is effectively 0% due to infrastructure failure, not test quality.

**Root cause hypothesis**: mutmut uses `os.fork()` to run each mutant in isolation (line 1103 of `__main__.py`). C extensions from `moto[dynamodb]` (boto3 mock) and `valkey-glide` (Rust FFI) are not fork-safe. When the forked child process tries to use these libraries, they segfault because their internal state (file descriptors, thread pools, Rust runtime) was corrupted by the fork.

**Evidence**:
- Exit code -11 = SIGSEGV on all tested mutants, no exceptions
- The test suite itself runs fine (329 pass in ~22s) — only fails in forked workers
- moto uses threading internally; valkey-glide has a Rust runtime — both are known fork-unsafe patterns

**Potential fixes to investigate** (in order of effort):
1. **`--max-children 1`** — run mutations sequentially in a single process (slow but may avoid fork issues)
2. **Subprocess mode** — check if mutmut 3.x has or plans a subprocess-based runner instead of `os.fork()`
3. **`do_not_mutate` config** — exclude files that import fork-unsafe libraries and test the rest
4. **Test isolation** — restructure tests to use lighter mocks that don't pull in C extensions
5. **pytest-forked / xdist** — use pytest plugins that handle fork cleanup properly

**Key diagnostic step**: Run mutmut on a single, simple module to confirm the hypothesis:
```bash
make _mutmut-setup
uv run mutmut run "filescience.throttling.valkey.keys.*"  # pure Python, no C deps
make _mutmut-teardown
```
If throttling/keys mutants work (no segfault), the C extension fork theory is confirmed.

## Artifacts

- `pyproject.toml` — `[tool.mutmut]` config + `mutmut>=3.4.0` dev dep
- `Makefile` — `mutmut`, `mutmut-results`, `mutmut-html`, `_mutmut-setup`, `_mutmut-teardown` targets
- `.github/workflows/mutation.yml` — nightly + manual dispatch workflow
- `.gitignore` — mutmut exclusions
- `memory-bank/durable/01-active/current_work.md` — updated
- `memory-bank/durable/01-active/next_up.md` — updated
- `~/.claude/projects/.../memory/patterns.md` — mutmut Polylith patterns added

## Action Items & Next Steps

1. **Diagnose segfaults**: Run single pure-Python module test (throttling/keys) to confirm C extension fork theory
2. **Try `--max-children 1`**: See if sequential execution avoids segfaults
3. **Try `do_not_mutate`**: Exclude modules importing moto/glide-heavy code, test remaining
4. **If all fail**: Consider cosmic-ray (uses subprocess, not fork) or mutmut issue/PR for subprocess mode
5. **Commit all changes** once segfault is resolved or workaround is in place
6. **Remaining backpressure axes**: xenon (complexity), detect-secrets (safety) — see research doc

## Other Notes

- The full run takes ~11 minutes for 6,720 mutants at 10.27 mutations/second (mostly the segfaulting workers cycling fast)
- mutmut caches results in `.mutmut-cache` and `mutants/*.meta` — incremental reruns skip already-tested mutants
- The `make mutmut` wrapper handles src/ copy tree lifecycle: setup before, teardown after (even on failure via `status=$$?` trap)
- mutmut `browse` TUI (`uv run mutmut browse`) is the best way to explore surviving mutants interactively
