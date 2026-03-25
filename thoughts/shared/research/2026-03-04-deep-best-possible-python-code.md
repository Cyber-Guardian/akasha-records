---
date: 2026-03-04T12:00:00-05:00
researcher: Claude
git_commit: 7da15bc
branch: main
repository: filescience
topic: "How to make Claude Code write the best possible Python code"
tags: [deep-research, python, code-quality, claude-code, extensions]
status: complete
research_depth: standard
iterations_completed: 3
last_updated: 2026-03-04
last_updated_by: Claude
---

# Deep Research: Making Claude Code Write Excellent Python

## Research Question
How to make Claude Code write the best possible Python code — skills, rules, hooks, prompting strategies, and other mechanisms to improve code quality beyond functional correctness (idiomaticity, readability, performance, design patterns, Pythonic style)

## Summary
Our codebase already has strong mechanical enforcement (ruff ALL, ty, import-linter as PostToolUse hooks) but zero design-level guidance for Claude Code. The codebase itself follows excellent Python patterns — frozen dataclasses, Protocol for subtyping, pathlib, StrEnum, modern typing, anyio TaskGroup — but these conventions are nowhere codified as instructions Claude can follow. The highest-leverage intervention is a **Python excellence rule file** that codifies existing codebase idioms with concrete file references, supplemented by an **adversarial Python reviewer agent** and an enhanced **/simplify skill**. Community evidence (Arize +10.87% accuracy, Trail of Bits config) confirms that project-specific prose guidance with real examples outperforms generic rules.

## Perspectives Explored
1. **Existing guardrails audit** — Revealed strong linting/typing/architecture enforcement but no design guidance
2. **Claude Code extension mechanisms** — Identified rules, hooks, skills, and agents as complementary layers; rules are the lightest-weight option for design guidance
3. **Industry best practices** — Trail of Bits hard limits, Arize pattern extraction, minimaxir type hints mandate; all confirm codebase-specific examples beat abstract guidelines
4. **Python-specific excellence criteria** — Built comprehensive inventory of idioms, typing, design, stdlib, and performance patterns; mapped against what the codebase already does
5. **Feedback loop & learning mechanisms** — Stop hook → reviewer subagent, claude-reflect correction capture, adversarial reviewer agents are all practical and implementable

## Detailed Findings

### 1. What We Already Have (and it's good)

**Enforcement hooks (PostToolUse):**
- Ruff with `select = ["ALL"]` — catches formatting, style, imports, and many idiom violations (`pyproject.toml:68-128`)
- ty type checker — catches type errors per-file
- import-linter — enforces Polylith architectural boundaries via 9 contracts (`pyproject.toml:193-297`)
- All three fire on every Edit/Write of `.py` files and block on failure (`.claude/settings.json:61-89`)

**Codebase conventions already in practice (but not codified):**
- `@dataclass(frozen=True)` for value objects (`throttling/models.py:39`, `cloud_api/client.py:27`)
- `Protocol` for structural subtyping (`discover/clouds/router.py:16`)
- `pathlib.Path` universal, no `os.path`
- `StrEnum` as dominant enum style
- Modern typing: `X | None`, `list[str]`, `collections.abc` over `typing` module
- `asynccontextmanager` for resource management (`dynamodb/utils.py:101`)
- `anyio.create_task_group()` + `except*` for concurrent async (`discover/manager.py:197-219`)
- Constructor-based DI — services receive deps as `__init__` params
- `lru_cache` for config singletons

**What's missing from the guardrails:**
- No design-level guidance (when to use dataclass vs Pydantic vs plain class)
- No Pythonic idiom preferences (stdlib leverage, comprehension style)
- No composition/pattern guidance
- No error handling strategy encoded
- Dev deps like `radon`, `mutmut`, `detect-secrets` installed but unhooked

### 2. Recommended Interventions (Priority Ordered)

#### A. Python Excellence Rule File (HIGH — do first)

Create `.claude/rules/python-excellence.md` — a prose rule file that codifies the project's established patterns and pushes toward remaining excellence frontiers. This is the single highest-leverage intervention based on all research.

**Proposed content structure:**

```
# Python Excellence

## Design patterns (this project's conventions)
- Value objects: `@dataclass(frozen=True)` — see throttling/models.py
- Boundary data: Pydantic `BaseModel` with validators — see valkey_queue_processor/models/schema.py
- Domain models: Plain classes with registries — see models/core.py
- Contracts: `Protocol` for structural subtyping, never ABC for duck typing
- DI: Constructor injection, optional factory functions alongside
- Config: Module-level constants with os.environ.get (per base)

## Modern Python (3.13 target)
- Typing: `list[str]`, `X | None`, `collections.abc.AsyncGenerator` — never `typing.List`, `Optional`, `typing.AsyncGenerator`
- Enums: `StrEnum` for string enums
- Async: `anyio.create_task_group()` + `except*` for concurrent work
- Match statements over elif chains for 3+ branches on the same value

## Stdlib over reinvention
- `pathlib.Path` — never `os.path`
- `contextlib.suppress` over empty except blocks
- `contextlib.asynccontextmanager` for async resource management
- `functools.lru_cache` / `cache` for expensive computations
- `itertools` for combinatorial/chained iteration
- `collections.defaultdict`, `Counter`, `deque` over manual equivalents
- `dataclasses.field(default_factory=...)` over mutable defaults

## Composition
- Prefer flat functions over deep class hierarchies
- Functions ≤50 lines, classes ≤200 lines (soft targets)
- ≤5 positional parameters — use keyword-only or a config dataclass beyond that
- Single responsibility: one reason to change per function/class
- Extract helper only when used ≥2 times or when it names a concept

## Error handling
- Catch specific exceptions — never bare `except:` or `except Exception:` outside CLI error boundaries
- `except Exception` at outermost boundary only, always with `logger.exception()`
- Include context in error messages: what was being done, what value caused the issue
- Raise early, catch late

## Testing voice
- Test behavior, not implementation
- Mock at boundaries (external services, I/O), not internal logic
- Fixtures for shared setup, inline for test-specific setup
- Descriptive test names: `test_<unit>_<scenario>_<expected>`
```

#### B. Adversarial Python Reviewer Agent (MEDIUM)

Create `.claude/agents/python-reviewer.md` — invoked by `/simplify` or manually after writing code. Focuses on Pythonic quality, not correctness (ruff/ty handle that).

**Checklist the agent should evaluate:**
1. Are value objects frozen dataclasses (not dicts or mutable classes)?
2. Is Protocol used instead of ABC for duck typing?
3. Is stdlib leveraged (pathlib, contextlib, functools, itertools, collections)?
4. Are type annotations modern style?
5. Are functions/classes within size targets?
6. Is error handling specific and contextual?
7. Does the code match established project patterns?

#### C. Enhanced /simplify Skill (MEDIUM)

The existing `/simplify` skill reviews changed code for reuse, quality, and efficiency. Enhance it to:
- Reference the python-excellence rule file
- Invoke the python-reviewer agent
- Produce concrete suggestions with file:line references

#### D. Hard Metric Limits in Hooks (LOW — consider later)

Inspired by Trail of Bits, add optional complexity checking:
- `radon` is already in dev deps — could add a PostToolUse hook for cyclomatic complexity >8
- Would need to balance enforcement (block) vs guidance (warn) — overly strict hooks slow development

#### E. Correction Capture / Learning Loop (LOW — consider later)

The claude-reflect pattern: PreToolUse hooks detect correction signals ("no, use X instead"), queue them, and a /reflect skill applies semantic filtering before syncing to rules or CLAUDE.md. This is sophisticated but high-maintenance. The /learn skill already partially covers this use case.

### 3. Why Prose Rules Beat More Hooks

Community evidence strongly supports this hierarchy:
1. **Prose guidance (rules/CLAUDE.md)** — shapes intent upstream, before code is written. Arize study: +10.87% from repo-specific context files alone.
2. **Enforcement hooks (PostToolUse)** — catches violations after writing. We already have this for formatting/typing/architecture.
3. **Review skills/agents** — catches design issues after completion. Useful but heavier-weight.

The gap in our setup is layer 1: we have excellent enforcement (layer 2) but no design guidance. Adding a rule file fills this gap with minimal infrastructure.

### 4. What NOT to Do

- **Don't add generic Python guidelines** — Claude already knows PEP 8. Rules should be project-specific.
- **Don't over-constrain with hooks** — Blocking on complexity metrics during rapid prototyping creates friction. Use soft targets in prose rules instead.
- **Don't duplicate ruff's job** — Ruff already enforces 500+ rules. The excellence rule file should cover what ruff can't: design, composition, and project idioms.
- **Don't add path-scoped rules yet** — Our existing rules are all always-loaded. A python-excellence rule is relevant to all `.py` files anyway.

## Key Sources

### Codebase files
- `pyproject.toml:68-128` — Ruff config (select ALL)
- `pyproject.toml:193-297` — Import-linter contracts
- `.claude/settings.json:61-89` — PostToolUse hook chain
- `.claude/rules/python-conventions.md` — Existing Python rules (formatting-focused)
- `components/filescience/throttling/models.py:39` — Frozen dataclass exemplar
- `bases/filescience/discover/clouds/router.py:16` — Protocol exemplar
- `components/filescience/dynamodb/utils.py:101` — asynccontextmanager exemplar
- `bases/filescience/discover/manager.py:197-219` — anyio TaskGroup + except* exemplar
- `components/filescience/models/core.py:6-141` — Plain-class domain model pattern
- `bases/filescience/valkey_queue_processor/models/schema.py:44-116` — Pydantic boundary model pattern

### External references
- [Arize: CLAUDE.md Best Practices (+10.87% accuracy gain)](https://arize.com/blog/claude-md-best-practices/)
- [Trail of Bits claude-code-config (hard metric limits, zero-warnings)](https://github.com/trailofbits/claude-code-config)
- [minimaxir Python CLAUDE.md gist](https://gist.github.com/minimaxir/c274d7cc12f683d93df2b1cc5bab853c)
- [O'Reilly: Auto-Reviewing Claude's Code (Stop hook architecture)](https://www.oreilly.com/radar/auto-reviewing-claudes-code/)
- [claude-reflect (correction capture → rule sync)](https://github.com/BayramAnnakov/claude-reflect)
- [Claude Code hooks guide](https://code.claude.com/docs/en/hooks-guide)
- [Claude Code custom subagents](https://code.claude.com/docs/en/sub-agents)
- [awesome-claude-code-subagents: architect-reviewer](https://github.com/VoltAgent/awesome-claude-code-subagents)

## Open Questions
- Should we enable `radon` complexity checking as a PostToolUse hook or keep it as a soft guideline in the rule file?
- Should the python-reviewer agent run automatically (Stop hook) or only when invoked via /simplify?
- How do we handle the tension between the rule file's design guidance and rapid prototyping (where you want to move fast and refine later)?
- Should we create a correction-capture mechanism (claude-reflect style) or is /learn sufficient?
- Would few-shot examples of good vs bad code in the rule file improve output, or would they bloat context?

## Research Metadata
- Depth mode: standard
- Iterations completed: 3 / 5
- Termination reason: gaps exhausted (1 synthesis gap remaining, no further research needed)
- Manifest: `.claude/deep-research/2026-03-04-best-possible-python-code.md`
