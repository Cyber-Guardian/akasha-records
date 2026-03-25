---
date: 2026-02-06T17:49:33Z
researcher: Claude
git_commit: 47f71bdda5244d9f4cfa1e3e2d490adf315ce15c
branch: main
repository: filescience
topic: "AI Agent Backpressure: Comprehensive Tool Research for Correctness, Fitness, and Beyond"
tags: [research, backpressure, mutation-testing, architecture, fitness-functions, security, complexity]
status: complete
last_updated: 2026-02-06
last_updated_by: Claude
---

# Research: AI Agent Backpressure — Comprehensive Tool Landscape

**Date**: 2026-02-06T17:49:33Z
**Researcher**: Claude
**Git Commit**: 47f71bdda5244d9f4cfa1e3e2d490adf315ce15c
**Branch**: main
**Repository**: filescience

## Research Question

Identify and evaluate tools that create backpressure for AI coding agents, organized by pressure axis. Starting points: correctness (mutation testing — mutmut, cosmic-ray) and fitness (architectural enforcement — import-linter, grimp, pytestarch). Also investigate whether additional axes exist beyond these two.

## Summary

We identified **four distinct axes** of backpressure for AI agents, not just two:

1. **Correctness** — Is the code doing what it should? (tests, mutation testing)
2. **Fitness** — Does the code maintain architectural integrity? (import rules, dependency boundaries, complexity)
3. **Safety** — Is the code free from security vulnerabilities and secrets? (SAST, secrets scanning, dependency auditing)
4. **Discipline** — Is the code staying within complexity/maintainability budgets? (cyclomatic complexity, diff-coverage, resource budgets)

The first two (correctness, fitness) are about *what the code does*. The latter two (safety, discipline) are about *what the code avoids*. All four can be automated and integrated into the existing Claude Code hook infrastructure at different stages.

---

## Axis 1: Correctness

### The Problem

If AI agents generate both code and tests, surviving tests don't prove correctness — they may just prove the agent is consistent with itself. Mutation testing breaks this circularity by asking: "If we introduce a bug, do the tests catch it?"

### Tool: mutmut (RECOMMENDED)

| Attribute | Value |
|-----------|-------|
| Version | 3.4.0 (Nov 2025) |
| Status | Actively maintained, 1.2k GitHub stars |
| Python | 3.10–3.14 |
| License | BSD-3-Clause |

**How it works**: Introduces small code changes (mutants) — operator swaps, literal changes, control flow modifications — and runs the test suite against each. Surviving mutants = test gaps.

**Mutation operators**: Arithmetic (`+`↔`-`), comparison (`<`↔`<=`), logical (`and`↔`or`), string methods (`lower()`↔`upper()`), literals (numbers increment), control flow (`break`→`return`), function arguments (removal/nullification).

**Monorepo support** (critical for Polylith):
```toml
[tool.mutmut]
paths_to_mutate = [
    "components/filescience/cloud_api/",
    "components/filescience/dynamodb/",
]
pytest_add_cli_args_test_selection = ["test/components/"]
tests_dir = "test/"
max_stack_depth = 10
```

**Performance**: Slow — not parallelized natively. A real-world example took 65 min unparallelized, ~15 min with ThreadPoolExecutor workaround. Supports incremental runs (caches in `.mutmut-cache`), coverage-guided mutations (`mutate_only_covered_lines = true`).

**Integration tier**: Stage 2 (CI, nightly) — too slow for hooks. Run `mutmut run --no-progress --CI` in GitHub Actions on schedule, cache `.mutmut-cache`.

**Output**: CLI results, interactive TUI (`mutmut browse`), HTML report (`mutmut html`), JUnit XML, JSON.

**Detection rate**: 88.5% in academic comparison (vs cosmic-ray's 82.7%).

### Tool: cosmic-ray (Alternative)

| Attribute | Value |
|-----------|-------|
| Version | 8.4.3 (Sep 2025) |
| Status | Maintained by Sixty North AS, 620 stars |
| Python | 3.9–3.13 |
| License | MIT |

**Advantages over mutmut**: More mutation operators (13 built-in), plugin system for custom operators/distributors/filters, HTTP-based distributed execution, kill matrix (which tests killed which mutants), git-based filtering (`cr-filter-git` — only mutate changed lines).

**Disadvantages vs mutmut**: Complex multi-step workflow (config → init → baseline → exec → report), lower detection rate (82.7%), modifies source in-place during execution, weaker HTML output.

**Git-based incremental filtering** (useful for CI):
```toml
[cosmic-ray.filters.git-filter]
branch = "main"
```

**Distributed execution**: HTTP workers for parallelism:
```toml
[cosmic-ray.distributor.http]
worker-urls = ["http://localhost:9876", "http://localhost:9877"]
```

### Recommendation: Correctness

**Use mutmut** for simplicity and better detection rate. Reserve cosmic-ray if you need distributed execution or custom operators. Neither is suitable for Claude Code hooks — run in nightly CI.

**Mutation score target**: 80%+ indicates strong test suite.

### Additional correctness tools

- **diff-cover**: Gates coverage on *changed lines only*, not legacy code. `diff-cover coverage.xml --compare-branch=origin/main --fail-under=80`. Fast, ideal for PR gates.
- **Hypothesis**: Property-based testing — generates randomized inputs to find edge cases. Integrates with pytest. Good complement to mutation testing.

---

## Axis 2: Fitness (Architectural Integrity)

### The Problem

AI agents don't inherently understand architectural boundaries. Without enforcement, they'll create cross-layer imports, circular dependencies, and coupling between components that should be independent.

### Tool: import-linter + grimp (RECOMMENDED for config-based enforcement)

| Attribute | Value |
|-----------|-------|
| Version | import-linter 2.9 (Dec 2025), grimp 3.14 (Dec 2025) |
| Status | Actively maintained, 941 stars |
| Python | ≥3.10 |
| License | BSD-2-Clause |

**How it works**: Define architectural contracts in `pyproject.toml`; `lint-imports` validates import graph against them.

**Five contract types**:

1. **Layers** — Higher layers can import lower, not vice versa:
```toml
[[tool.importlinter.contracts]]
name = "Polylith layer enforcement"
type = "layers"
layers = ["projects", "bases", "components"]
```

2. **Forbidden** — Block specific import paths:
```toml
[[tool.importlinter.contracts]]
name = "Components cannot import bases"
type = "forbidden"
source_modules = ["filescience.cloud_api", "filescience.dynamodb", "filescience.models"]
forbidden_modules = ["filescience.discover", "filescience.valkey_queue_processor"]
```

3. **Independence** — Modules must not depend on each other:
```toml
[[tool.importlinter.contracts]]
name = "Components are independent"
type = "independence"
modules = ["filescience.cloud_api", "filescience.dynamodb", "filescience.throttling"]
```

4. **Acyclic siblings** — No circular dependencies between sibling packages.

5. **Protected** — Only specified modules may import protected internals.

**Monorepo/Polylith support**: Use `root_packages` for multi-package layout. Known limitation: packages must be on PYTHONPATH simultaneously.

**Performance**: Graph building cached in `.grimp_cache/`, incremental rescanning of changed files only. Fast enough for CI, possibly for hooks with caching.

**Pre-commit integration**:
```yaml
repos:
  - repo: https://github.com/seddonym/import-linter
    rev: v2.9
    hooks:
      - id: import-linter
```

### Tool: pytestarch (RECOMMENDED for test-based enforcement)

| Attribute | Value |
|-----------|-------|
| Version | 4.0.1 (Aug 2025) |
| Status | Most actively maintained architectural test tool, 145 stars |
| Python | 3.9–3.13 |
| License | Open source |

**How it works**: Write architectural rules as pytest tests with a fluent API:

```python
from pytestarch import get_evaluable_architecture, Rule

evaluable = get_evaluable_architecture("/project/root", "/project/root/components")

def test_components_dont_import_bases():
    rule = (
        Rule()
        .modules_that()
        .are_sub_modules_of("filescience.cloud_api")
        .should_not()
        .be_imported_by_modules_that()
        .are_sub_modules_of("filescience.discover")
    )
    rule.assert_applies(evaluable)
```

**Key advantage over import-linter**: Runs as normal pytest tests — lives alongside your test suite, uses pytest fixtures, reports in pytest output.

**Features**: Fluent API, module filtering, visualization support (matplotlib), layer architecture rules, PlantUML integration.

### Tool: pytest-archon (Lighter alternative)

| Attribute | Value |
|-----------|-------|
| Stars | 77 |
| API | `archrule()` fluent builder |

```python
from pytest_archon import archrule

def test_components_independent():
    (
        archrule("components-independent")
        .match("filescience.cloud_api.*")
        .should_not_import("filescience.dynamodb.*")
        .check("filescience")
    )
```

Supports fnmatch/regex patterns, custom predicate functions, TYPE_CHECKING block skipping.

### Recommendation: Fitness

**Use both** import-linter (config-based, runs in CI/hooks) AND pytestarch or pytest-archon (test-based, runs with test suite). They complement each other:
- import-linter: fast gate, blocks violations before merge
- pytestarch: expressive rules in code, runs with `pytest`

---

## Axis 3: Safety

### The Problem

AI agents can introduce security vulnerabilities — hardcoded secrets, unsafe deserialization, SQL injection patterns, use of vulnerable dependencies. These are distinct from correctness (code may "work" but be insecure).

### Tool: Ruff security rules (ALREADY PARTIALLY ACTIVE)

The current Ruff configuration (`pyproject.toml:64`) selects `ALL` rules, which includes the `S` (flake8-bandit) security rules. However, some may be in the ignore list.

**Bandit rules via Ruff** (S-prefix): S101 (assert), S105 (hardcoded password), S301 (pickle), S603 (subprocess), S608 (SQL injection), etc.

**Speed**: < 5s for full scan. Already runs in Claude Code PostToolUse hooks.

**Action needed**: Audit which S-rules are currently suppressed and whether any should be restored.

### Tool: detect-secrets (RECOMMENDED — new addition)

| Attribute | Value |
|-----------|-------|
| Maintainer | Yelp |
| Approach | Baseline methodology (prevent NEW secrets) |
| Speed | Very fast (scans diffs, not full history) |

```bash
# Initialize baseline
detect-secrets scan > .secrets.baseline

# Pre-commit hook
detect-secrets-hook --baseline .secrets.baseline
```

**Why it matters for AI agents**: Agents may accidentally hardcode API keys, tokens, or credentials. detect-secrets catches this before commit.

**Integration**: Pre-commit hook or Claude Code hook (fast enough at < 2s).

### Tool: pip-audit (dependency vulnerabilities)

| Attribute | Value |
|-----------|-------|
| Maintainer | Google (PyPA) |
| Speed | Fast for incremental checks |

```bash
pip-audit --requirement requirements.txt
```

**Run conditionally**: Only when `pyproject.toml`, `uv.lock`, or requirements files change.

### Recommendation: Safety

- **Ruff S-rules**: Already active, audit suppressions
- **detect-secrets**: Add as Claude Code hook or pre-commit (< 2s)
- **pip-audit**: Add to CI, run on dependency changes

---

## Axis 4: Discipline (Complexity & Maintainability Budgets)

### The Problem

AI agents tend toward over-engineering — adding abstractions, deep nesting, long functions. Without budgets, complexity creeps silently. This is different from fitness (which is about *boundaries*) — discipline is about *budgets per unit of code*.

### Tool: xenon + radon (RECOMMENDED)

| Attribute | Value |
|-----------|-------|
| Speed | < 3s (AST parsing) |
| Gate-ready | Yes |

**Radon** computes: cyclomatic complexity, maintainability index, Halstead metrics, raw LOC.

**Xenon** wraps radon with threshold enforcement:
```bash
xenon --max-absolute B --max-modules A --max-average A components/ bases/
```

Letter grades: A (low complexity) through F (very high). Fails with non-zero exit on threshold breach.

**Pre-commit hook**: Available at `https://github.com/yunojuno/pre-commit-xenon`.

**Claude Code hook candidate**: Fast enough (< 3s) to run PostToolUse on Python files.

### Tool: diff-cover (coverage discipline on changed code)

Already mentioned under correctness, but it also serves discipline: ensures new/changed code meets coverage thresholds without penalizing legacy code.

```bash
diff-cover coverage.xml --compare-branch=origin/main --fail-under=80
```

### Not recommended: wily

Low maintenance (no releases in 12+ months). Use xenon instead for gating.

### Recommendation: Discipline

- **xenon**: Add as Claude Code hook (< 3s, perfect for PostToolUse)
- **diff-cover**: Add to CI for PR gates

---

## Why Four Axes, Not Two

The original framing of "correctness" and "fitness" captures the most important dimensions but misses two others that matter specifically for AI-generated code:

| Axis | Question | Failure Mode | Key Tools |
|------|----------|-------------|-----------|
| **Correctness** | Does it work? | Silent bugs, weak tests | mutmut, pytest, diff-cover |
| **Fitness** | Does it fit the architecture? | Cross-layer coupling, circular deps | import-linter, pytestarch |
| **Safety** | Is it secure? | Hardcoded secrets, vulnerable deps, injection | Ruff S-rules, detect-secrets, pip-audit |
| **Discipline** | Is it maintainable? | Over-complexity, deep nesting, bloat | xenon/radon, diff-cover |

**Safety** is distinct because code can be correct, well-structured, AND insecure. An AI agent that generates a perfectly working, architecturally clean `pickle.loads(user_input)` has failed on safety.

**Discipline** is distinct because code can be correct, safe, well-architected, AND unmaintainable. An AI agent that solves a problem with a 200-line function of cyclomatic complexity 47 has failed on discipline.

---

## Integration Map: Where Each Tool Runs

### Stage 0 — Claude Code Hooks (< 30s, every file write)

| Tool | Speed | Currently Active | Action |
|------|-------|-----------------|--------|
| Ruff (lint + security) | < 5s | ✅ Yes | Audit S-rule suppressions |
| ty (type check) | < 10s | ✅ Yes | No change |
| xenon (complexity) | < 3s | ❌ No | **ADD** as PostToolUse hook |
| detect-secrets | < 2s | ❌ No | **ADD** as PostToolUse hook |

### Stage 1 — CI / PR Gates (minutes)

| Tool | Speed | Action |
|------|-------|--------|
| Full pytest suite | ~30s | Already in Makefile |
| import-linter | ~10s | **ADD** to CI |
| pytestarch/pytest-archon | With pytest | **ADD** as test files |
| diff-cover | < 5s | **ADD** to PR checks |
| pip-audit | ~10s | **ADD** to CI (on dep changes) |

### Stage 2 — Nightly / Scheduled (minutes to hours)

| Tool | Speed | Action |
|------|-------|--------|
| mutmut | 15–65 min | **ADD** as scheduled CI |
| Full coverage report | With pytest | Already in Makefile (`test-cov`) |

---

## Current State of the Codebase

### What exists today
- **329 tests** across 43 files (throttling: 191, dynamodb: 61, bases: ~15)
- **Ruff**: ALL rules selected, 38 specific ignores, PostToolUse hook
- **ty**: Type checking, PostToolUse hook
- **pytest-cov**: 80% threshold configured in Makefile
- **GitHub Actions**: Ruff lint only (`.github/workflows/lint.yml`)
- **No mutation testing** currently
- **No architectural enforcement** currently
- **No secrets scanning** currently
- **No complexity gating** currently

### What the existing back-pressure research covers
The draft at `memory-bank/thoughts/shared/research/2026-02-05-agentic-back-pressure.md` established the signal taxonomy and back-pressure ladder concept. This research fills in the specific tooling for each stage.

---

## Tool Comparison Tables

### Mutation Testing: mutmut vs cosmic-ray

| Feature | mutmut | cosmic-ray |
|---------|--------|------------|
| Version | 3.4.0 (Nov 2025) | 8.4.3 (Sep 2025) |
| Maintenance | ✅ Very active | ✅ Active |
| Setup | Easy, works out-of-box | Complex multi-step |
| Detection rate | 88.5% | 82.7% |
| Parallelization | ❌ Not native | ✅ HTTP workers |
| Git-based filtering | ❌ No | ✅ cr-filter-git |
| Plugin system | ❌ Limited | ✅ Operators, filters, distributors |
| Monorepo | ✅ Path targeting | ✅ Module lists |
| **Recommendation** | **Start here** | Upgrade if scale demands |

### Architectural Testing: import-linter vs pytestarch vs pytest-archon

| Feature | import-linter | pytestarch | pytest-archon |
|---------|--------------|------------|---------------|
| Version | 2.9 (Dec 2025) | 4.0.1 (Aug 2025) | Unknown |
| Stars | 941 | 145 | 77 |
| Approach | Config-based | Test-based | Test-based |
| Contract types | 5 built-in | Fluent Rule API | archrule() builder |
| Custom rules | ✅ Plugin system | ✅ Via Rule API | ✅ Via predicates |
| Visualization | ❌ | ✅ matplotlib | ❌ |
| Pre-commit | ✅ Official hook | ❌ (runs as pytest) | ❌ (runs as pytest) |
| **Best for** | CI gates | Test suite | Quick rules |

---

## Open Questions

1. **import-linter + Polylith**: The multi-package setup (`root_packages`) may need PYTHONPATH tuning for Polylith. Needs hands-on verification.
2. **xenon thresholds**: What letter grades (A/B/C) are appropriate for this codebase? Need to baseline current complexity first.
3. **detect-secrets baseline**: Need to generate initial `.secrets.baseline` and tune for false positives.
4. **mutmut on this codebase**: How long does a full mutation run take for 329 tests across components/bases? Need to time a trial run.
5. **Hook ordering**: With xenon and detect-secrets added, the PostToolUse hook chain would be 4 tools. Need to verify cumulative latency stays under ~30s.
6. **Hypothesis integration**: Worth exploring for DynamoDB serializers and throttle service edge cases, but separate from backpressure tooling.

---

## Sources

### Mutation Testing
- [mutmut docs](https://mutmut.readthedocs.io/) | [GitHub](https://github.com/boxed/mutmut) | [PyPI](https://pypi.org/project/mutmut/)
- [cosmic-ray docs](https://cosmic-ray.readthedocs.io/) | [GitHub](https://github.com/sixty-north/cosmic-ray) | [PyPI](https://pypi.org/project/cosmic-ray/)
- [Comparison: Jakob Breu](https://jakobbr.eu/2021/10/10/comparison-of-python-mutation-testing-modules/)
- [IEEE comparison paper](https://ieeexplore.ieee.org/document/10818231/)
- [ACM study](https://dl.acm.org/doi/10.1145/3701625.3701659)
- [Getting Started with Mutmut (Codecov)](https://about.codecov.io/blog/getting-started-with-mutation-testing-in-python-with-mutmut/)
- [Automating Mutmut with GitHub Actions](https://blog.stackademic.com/automating-mutation-testing-with-mutmut-and-github-actions-9767b4fc75b5)

### Architectural Testing
- [import-linter docs](https://import-linter.readthedocs.io/) | [GitHub](https://github.com/seddonym/import-linter)
- [grimp docs](https://grimp.readthedocs.io/) | [GitHub](https://github.com/python-grimp/grimp)
- [pytestarch docs](https://zyskarch.github.io/pytestarch/latest/) | [GitHub](https://github.com/zyskarch/pytestarch)
- [pytest-archon GitHub](https://github.com/jwbargsten/pytest-archon)
- [Six lines of code to prevent spaghetti](https://seddonym.me/2025/11/12/six-lines-of-code/)
- [6 ways to improve Python architecture](https://www.piglei.com/articles/en-6-ways-to-improve-the-arch-of-you-py-project/)
- [Practical guide to import-linter](https://921kiyo.com/python-import-linter/)

### Security
- [Ruff rules (S-prefix)](https://docs.astral.sh/ruff/rules/)
- [detect-secrets GitHub](https://github.com/Yelp/detect-secrets)
- [pip-audit GitHub](https://github.com/pypa/pip-audit) | [PyPI](https://pypi.org/project/pip-audit/)
- [Safety vs pip-audit comparison](https://www.sixfeetup.com/blog/safety-pip-audit-python-security-tools)
- [Semgrep Python SAST](https://semgrep.dev/blog/2024/redefining-security-coverage-for-python-with-framework-native-analysis/)

### Complexity & Discipline
- [radon docs](https://radon.readthedocs.io/) | [PyPI](https://pypi.org/project/radon/)
- [xenon GitHub](https://github.com/rubik/xenon)
- [pre-commit xenon hook](https://github.com/yunojuno/pre-commit-xenon)
- [diff-cover PyPI](https://pypi.org/project/diff-cover/)

### Conceptual / Fitness Functions
- [Evolutionary architecture](https://evolutionaryarchitecture.com/precis.html)
- [AWS fitness functions](https://aws.amazon.com/blogs/architecture/using-cloud-fitness-functions-to-drive-evolutionary-architecture/)
- [Fitness functions in Python](https://makimo.com/blog/govern-software-architecture-with-fitness-functions-in-python/)
- [[2026-02-05-agentic-back-pressure|Existing backpressure research]]

### Pre-commit & Hooks
- [Pre-commit framework](https://pre-commit.com/)
- [Claude Code hooks documentation](https://code.claude.com/docs/en/hooks-guide)
