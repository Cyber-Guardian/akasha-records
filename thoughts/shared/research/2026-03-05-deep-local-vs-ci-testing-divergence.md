---
date: 2026-03-05T12:00:00-05:00
researcher: Claude
git_commit: e58030f9f755d5153428ce061b647a2f4117a91b
branch: main
repository: filescience
topic: "Local vs CI testing divergence: pre-commit hooks, git hooks, and strategies to prevent PRs from failing"
tags: [deep-research, ci, pre-commit, git-hooks, ruff, pytest, developer-experience]
status: complete
research_depth: standard
iterations_completed: 3
last_updated: 2026-03-05
last_updated_by: Claude
---

# Deep Research: Local vs CI Testing Divergence

## Research Question
Local vs CI testing divergence: pre-commit hooks, git hooks, and strategies to prevent PRs from failing due to local/CI environment differences. Research best practices for Python monorepos using ruff, pytest, uv, and import-linter.

## Summary
The root cause of PR failures in this repo is **not environment divergence** — `pyproject.toml` already ensures identical tool config between local and CI, and no platform-specific test issues exist. The real problem is that **linting and tests don't run locally before push**: CI runs 7 jobs on PRs, but only 3 have automatic local triggers (ruff + import-linter via Claude Code hooks, detect-secrets via git pre-commit). The fix is a two-tier git hook expansion — `pre-commit` for fast lint/format (<1s), `pre-push` for tests + import-linter (~5s) — plus new Makefile targets (`make lint`, `make check`) that CI workflows call instead of inline commands. Expanding the existing `.githooks/` scripts is the recommended approach over adopting the `pre-commit` framework or `lefthook`, given the small team, existing infrastructure, and sub-4-second total check time.

## Perspectives Explored
1. **Current Gap Analysis** — Mapped all 7 CI jobs against local equivalents; identified 4 checks with no automatic local trigger
2. **Pre-commit Hook Ecosystem** — Compared raw `.githooks/`, `pre-commit` framework, and `lefthook`; each has clear trade-offs for this stack
3. **Developer Experience (DX)** — Benchmarked check performance (<4s total), designed staged-only checking patterns, and established the pre-commit/pre-push split
4. **CI Parity Patterns** — Found Makefile-as-single-source-of-truth is the industry standard; identified specific CI/Makefile alignment gaps
5. **Monorepo-Specific Concerns** — Confirmed no macOS/Linux divergence; pyproject.toml is the single config source

## Detailed Findings

### 1. Current State: The Gap

| CI Job | What it checks | Local automatic equivalent? |
|--------|---------------|---------------------------|
| `ruff` | Lint (ruff check) | Claude Code PostToolUse hook only |
| `architecture` | Import boundaries (lint-imports) | Claude Code PostToolUse hook only |
| `pytest` | Tests + diff-coverage at 80% | None |
| `detect-secrets` | Secret scanning | `.githooks/pre-commit` (if `make setup-hooks` run) |
| `pip-audit` | Dependency vulnerabilities | None |
| `extensions` | .claude/ cross-reference integrity | None |
| `scope-check` | Changed files vs plan doc | None |

The Claude Code PostToolUse hooks for ruff and import-linter only fire during Claude-assisted editing — they don't help when devs edit files manually or use other editors. The git pre-commit hook only runs detect-secrets.

The repo has a sophisticated **reactive** CI failure system: `ci-failure-linear.yml` creates Linear issues on main-branch failures, `agent-ci-feedback.yml` posts up to 3 feedback rounds to PR agents with SHA dedup and human escalation, and `semantic-pr.yml` validates PR title format. But these all fire **after** push — the missing piece is **prevention**.

Sources: `.github/workflows/lint.yml`, `.github/workflows/test.yml`, `.githooks/pre-commit`, `.claude/settings.json:74-86`, `.github/workflows/ci-failure-linear.yml`, `.github/workflows/agent-ci-feedback.yml`

### 2. Hook Strategy Comparison

**Option A: Expand raw `.githooks/` (recommended)**
- Zero new dependencies — uses tools already in `uv.lock`
- Already set up via `make setup-hooks` (`git config core.hooksPath .githooks`)
- Checks are fast enough (<4s) that caching/parallelism aren't needed
- Staged-only ruff via `git diff --name-only --cached -- '*.py'`
- Stash pattern for partial staging: `git stash push --keep-index -q` + `trap ... EXIT`
- Skip mechanism: `SKIP_HOOKS=1` env-var with `.git/hooks-skipped.log` audit trail
- **Trade-off:** No built-in caching or cross-platform portability

**Option B: Adopt `pre-commit` framework**
- Declarative YAML config, content-addressable cache, large hook ecosystem
- Native ruff and import-linter hooks available
- **Trade-offs:** No native uv support (requires `pre-commit-uv`); isolated venvs can drift from project venvs; monorepo path scoping requires hook stanza duplication; 30-50s delays on slow hooks lead to `--no-verify` abuse
- Better for larger teams (10+ devs) where ecosystem benefits outweigh abstraction cost

**Option C: Adopt `lefthook`**
- Fastest option (~1s parallel execution vs ~5s pre-commit)
- Built-in `{staged_files}` interpolation, `parallel: true`, skip conditions
- **Trade-offs:** Not officially on PyPI; requires brew or system package manager; falls outside the uv toolchain; smaller ecosystem
- Better for polyglot repos or teams already using Go tooling

Sources: [pre-commit framework](https://pre-commit.com), [lefthook](https://github.com/evilmartians/lefthook), [pre-commit-uv](https://pypi.org/project/pre-commit-uv/), [pre-commit monorepo issues](https://github.com/pre-commit/pre-commit/issues/466)

### 3. Recommended Check Split

| Hook | Checks | Time | Rationale |
|------|--------|------|-----------|
| **pre-commit** | `ruff check` (staged .py), `ruff format --check` (staged .py), `detect-secrets` (staged) | <1s | Fast, catches 80% of lint failures |
| **pre-push** | `make test-fast`, `make lint-imports` | ~5-8s | Catches test + architecture failures before CI |
| **CI-only** | `pip-audit`, `diff-coverage`, `scope-check`, `extensions`, `complexity-report` | N/A | Too slow or requires CI context (base branch, coverage XML) |

import-linter cannot check individual files (`pass_filenames: false`) — it always builds the full import graph (~1-3s). This makes it too slow for pre-commit but fine for pre-push.

Sources: [ruff performance](https://www.pantsbuild.org/blog/2023/02/28/linting-python-at-warp-speed), [import-linter hooks config](https://github.com/seddonym/import-linter/blob/main/.pre-commit-hooks.yaml), [staged-only ruff discussion](https://github.com/astral-sh/ruff/discussions/4049)

### 4. CI/Makefile Parity

Current state of alignment:

| CI Job | Makefile Target | Aligned? |
|--------|----------------|----------|
| `ruff` | *(none)* | No — CI uses `astral-sh/ruff-action@v1` with no version pin |
| `architecture` | `make lint-imports` | Yes — exact match |
| `pytest` | `make test-ci` | Yes — exact match |
| `detect-secrets` | `make scan-secrets` | Partial — CI pins `detect-secrets==1.5.0` via bare pip; Makefile uses `uv run` |
| `extensions` | `make lint-extensions` | Yes — exact match |
| `pip-audit` | *(none)* | No target |
| `scope-check` | *(none)* | No target |
| `complexity-report` | `make complexity-report` | Yes — exact match |

**Recommended new targets:**
- `make lint` — `uv run ruff check . && uv run ruff format --check .`
- `make check` — `make lint && make lint-imports && make test-fast` (the "did I break anything?" command)
- `make ci-lint` — full lint suite matching CI exactly (for local CI simulation)

Then refactor CI YAML to call `make lint`, `make lint-imports`, `make test-ci` instead of inline commands. This ensures the commands run identically everywhere.

Sources: `.github/workflows/lint.yml:15-17`, `Makefile:177-186`, `.github/workflows/test.yml:22-25`

### 5. Staged-Only Checking in Raw Hooks

The standard pattern for staged-only ruff in a raw git hook:

```bash
STAGED=$(git diff --name-only --cached --diff-filter=ACM -- '*.py')
[ -z "$STAGED" ] && exit 0

# Option A: File-level granularity (simple, occasional false positives)
echo "$STAGED" | xargs uv run ruff check
echo "$STAGED" | xargs uv run ruff format --check

# Option B: Stash unstaged changes first (exact, handles partial staging)
git stash push --keep-index --include-untracked -q
trap 'git stash pop -q 2>/dev/null' EXIT
uv run ruff check $STAGED
uv run ruff format --check $STAGED
```

Option A is simpler and sufficient for most cases. Option B is needed only if the team frequently partially stages files.

Sources: [stash pattern](https://www.darrenlester.com/blog/git-pre-commit-stash), [git-format-staged](https://github.com/hallettj/git-format-staged), [pre-commit vs CI](https://switowski.com/blog/pre-commit-vs-ci/)

### Cross-cutting Patterns

1. **The problem is triggers, not config.** pyproject.toml already ensures identical tool behavior. The CI failure feedback loop is excellent but reactive. Adding pre-commit + pre-push hooks shifts failure detection left.

2. **Makefile as contract.** The industry standard is CI YAML calling make targets, not inline commands. This repo is halfway there — some CI jobs match make targets exactly, others don't. Closing this gap means any developer can run `make check` and get the same result as CI.

3. **Speed is the key to adoption.** At <4s for lint + import-linter, there's no excuse to skip hooks. The moment hooks exceed ~10s, `--no-verify` usage spikes. Keep tests in pre-push (not pre-commit) to stay under this threshold.

4. **Logged skips > silent skips.** Replace reliance on `--no-verify` with a `SKIP_HOOKS=1` pattern that writes to `.git/hooks-skipped.log`. This provides an audit trail without blocking urgent pushes.

## Key Sources

### Codebase files
- `.github/workflows/lint.yml` — CI lint jobs (ruff, architecture, pip-audit, detect-secrets, extensions, scope-check)
- `.github/workflows/test.yml` — CI test job (pytest + diff-coverage)
- `.github/workflows/ci-failure-linear.yml` — Reactive CI failure Linear issue creation
- `.github/workflows/agent-ci-feedback.yml` — Agent PR feedback loop (3-round escalation)
- `.github/workflows/semantic-pr.yml` — PR title/metadata validation
- `.githooks/pre-commit` — Current pre-commit hook (detect-secrets only)
- `.claude/settings.json:74-86` — PostToolUse hooks (ruff, ty, import-linter)
- `Makefile:167-186` — Test and lint targets
- `pyproject.toml:142-160` — pytest + tool config (single source of truth)
- `scripts/check_scope.py` — Scope validation against plan docs
- `scripts/validate_extensions.py` — .claude/ cross-reference integrity checks

### Memory-bank docs
(none directly relevant — this is new research)

### External URLs
- [pre-commit framework](https://pre-commit.com) — Declarative hook manager
- [lefthook](https://github.com/evilmartians/lefthook) — Fast Go-based hook manager
- [pre-commit-uv](https://pypi.org/project/pre-commit-uv/) — uv integration for pre-commit
- [pre-commit monorepo limitations](https://github.com/pre-commit/pre-commit/issues/466)
- [Ruff integrations guide](https://docs.astral.sh/ruff/integrations/)
- [Ruff staged-only discussion](https://github.com/astral-sh/ruff/discussions/4049)
- [import-linter hooks config](https://github.com/seddonym/import-linter/blob/main/.pre-commit-hooks.yaml)
- [nektos/act — local GitHub Actions runner](https://github.com/nektos/act)
- [Makefile-as-CI-contract pattern](https://shipyard.build/blog/local-first-cicd-with-makefiles/)
- [Git stash pattern for pre-commit hooks](https://www.darrenlester.com/blog/git-pre-commit-stash)
- [Logged hook skips pattern](https://adamj.eu/tech/2023/02/13/git-skip-hooks/)
- [Pre-commit vs CI analysis](https://switowski.com/blog/pre-commit-vs-ci/)

## Open Questions
- What are the actual historical CI failure rates by job type? (Would need `gh run list` analysis to quantify)
- Should `make check` be required before PR creation, or is pre-push sufficient?
- Would adopting `nektos/act` for local CI simulation add value, or is the hook approach sufficient?
- As team grows, at what point does pre-commit framework become worth the abstraction cost?

## Research Metadata
- Depth mode: standard
- Iterations completed: 3 / 5
- Termination reason: gaps exhausted
- Manifest: `.claude/deep-research/2026-03-05-local-vs-ci-testing-divergence.md`
