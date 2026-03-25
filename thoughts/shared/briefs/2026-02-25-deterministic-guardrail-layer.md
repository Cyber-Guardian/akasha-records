# Idea Brief: Deterministic Guardrail Layer for AI Coding

**Date:** 2026-02-25
**Status:** Shaped — Exploring (AST graph infrastructure deep-dive in progress)

## Problem
This repo has strong post-hoc guardrails (ruff/ty/import-linter hooks catch bad output after it's written) but no pre-change constraints that scope what an AI agent should touch, what it should see, or how to verify that a change is complete. The guardrails catch syntax/style/boundary violations but don't catch wrong-scope, incomplete, or subtly incorrect changes.

## Constraints
- Repo is small: 12 Polylith bricks, 81 source files, ~279 top-level definitions, max 3-hop DAG
- `settings.json` deny rules have known bugs — PreToolUse hooks are the only reliable enforcement
- Python-only monorepo — stdlib `ast` module is sufficient (tree-sitter adds complexity without semantic benefit)
- `scripts/diagrams/parsers/polylith.py` already computes `allowed_imports` from contracts
- `scripts/diagrams/parsers/python_ast.py` parses code structure but skips imports
- Context/token pressure is low at current repo size — capsules have lower ROI now, higher later
- No standard for "change contracts" exists in the industry

## Input Research
- Deterministic layer claims validation: [[2026-02-25-deterministic-layer-ai-coding-claims|2026-02-25-deterministic-layer-ai-coding-claims.md]]
- vexp local context engine review: [[2026-02-25-vexp-local-context-engine-review|2026-02-25-vexp-local-context-engine-review.md]]

## Full Vision — All Layers

### Layer 0: Fill Existing Gaps (no new tooling)
- pytest in CI (tests only gate TaskCompleted today, not PRs) — ~1hr, HIGH value
- ty in CI — ~30min, MEDIUM value
- Mutation testing as hard gate — ~30min, LOW-MEDIUM value
- Coverage as CI gate — ~30min, MEDIUM value (only throttling+dynamodb today)

### Layer 1: Dependency Graph Infrastructure (foundation)
- Import extractor: AST-based `from filescience.X` extraction across all bricks — ~half day, FOUNDATION
- Reverse-reachability query (blast radius): given file(s), return all transitive dependents — ~2hrs, HIGH
- Forward-reachability query: given file(s), return all transitive dependencies — ~1hr, MEDIUM
- Function-level call graph: track cross-module function calls via `ast.NodeVisitor` — ~1 day, MEDIUM (dynamic dispatch limits)
- CLI interface: `scripts/dep-graph.py --blast-radius <file>` etc. — ~2hrs, ergonomics

### Layer 2: Change Contract Enforcement
- PreToolUse hook: reads `.claude/change-contract.txt`, blocks Edit/Write to unlisted files — ~2hrs, HIGH
- Bash write detection: second PreToolUse on Bash, regex for `>`, `sed -i`, etc. — ~3hrs, MEDIUM
- Contract file format: glob patterns, `#` comments, optional metadata header — ~30min
- Auto-generation from blast-radius: seed files → blast-radius → contract file — ~2hrs, HIGH
- Skill integration: `/resolve-linear-issue` and `/implement_plan` generate/respect contracts — ~half day, HIGH

### Layer 3: Context Scoping (vexp brain-drain)
- Pivot + skeleton capsule: full file + signatures-only of blast radius — ~half day, MEDIUM now / HIGH at scale
- Handoff context capsule: auto-include capsule in `/create_handoff` — ~half day, MEDIUM
- Task-scoped context injection: UserPromptSubmit hook injects focused preamble — ~3hrs, MEDIUM
- Aider-style repo map: tree-sitter tag extraction for whole repo as persistent context — ~1-2 days, LOW now

### Layer 4: Post-Change Semantic Verification
- Blast-radius diff check: compare `git diff` vs blast radius, flag untouched affected files — ~half day, HIGH
- Change summary assertion: TaskCompleted hook verifies agent's declared changes vs actual diff — ~3hrs, MEDIUM
- Coverage delta: before/after coverage on affected files — ~1 day, MEDIUM
- Mutation delta: mutation testing only on changed files — ~1-2 days, LOW-MEDIUM

### Layer 5: Policy & Risk Controls
- Fail-closed cross-boundary: changes touching 2+ bricks require explicit confirmation — ~3hrs, MEDIUM
- Forbidden pattern deny-list: reject `eval()`, hardcoded creds, unscoped `# type: ignore` — ~half day, MEDIUM
- Confidence-based review routing: agent self-declares confidence, low = flagged — ~2hrs, LOW
- Codemod playbooks: pre-written AST transforms for common refactors — ~2-3 days each, HIGH per-playbook

## Dependencies Between Layers
- Layer 1 (graph) is the foundation for Layer 2 auto-gen, Layer 3 capsules, Layer 4 verification
- Layer 0 is fully independent — can ship immediately
- Layer 2 hook works without graph (hand-written contracts) but improves with auto-gen
- Layer 4 is complementary to Layer 2, not sequential
- Layer 5 pieces are all independent/additive

## Where the Inputs Land
- vexp learnings: graph retrieval → L1, pivot+skeleton → L3, blast-radius → L1+L4, bounded capsules → L3
- Deterministic research: structural → L0+L1, behavioral → L0+L4, policy → L2+L5, intent → L2+L3

## Diminishing Returns Frontier
- Layers 0-2: high-value, low-to-medium effort. Clearly worth doing.
- Layer 3: medium-value now, high-value later. Timing depends on repo growth.
- Layer 4 blast-radius diff check: high-value. Rest of L4 is diminishing.
- Layer 5: fail-closed + forbidden patterns worth it. Codemod playbooks high-effort per unit.

## Open — Decomposition Pending
User to decide how to decompose these layers into shippable work units. AST graph infrastructure (Layer 1) deep-dive in progress to inform that decision.
