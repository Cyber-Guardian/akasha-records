---
type: event
created: 2026-03-16
status: active
---
# Idea Brief: Custom Deterministic Agentic Backpressure Tooling

**Date:** 2026-03-16
**Status:** Shaped → Planning

## Problem
Claude Code agents can write code that passes all existing hooks (Ruff, ty, import-linter) while violating domain-specific invariants that no off-the-shelf tool knows about. The import-linter contracts have 8 verified gaps — the `tool_*` bricks have zero enforcement, and some product contracts are incomplete. As autonomous agent work scales via the in-loop/out-loop model, these unenforced invariants become probabilistic bugs.

## Constraints
- PostToolUse hook budget: ~25s remaining (existing chain takes ~4-5s, zero timeouts in 5,600+ invocations)
- Pure Python validators (no subprocess) add negligible latency (<100ms)
- Hook JSON contract is proven: stdin JSON → stdout `{"decision": "block", "reason": "..."}`
- Plan `### Files` format is 97% consistent (57/58 plans), helm's `plan_parser.py` already parses it
- AST parsing is reliable for static list literals and decorators
- pytestarch rejected for Polylith multi-root incompatibility — custom validators sidestep this

## Options Considered

### Contract Completeness Linter
A declarative YAML adjacency list of the intended dependency graph, plus a validator that diffs it against actual pyproject.toml import-linter contracts. Catches the 8 verified gaps where import-linter contracts are incomplete or missing.
- Gains: Directly encodes architectural knowledge; catches entire categories of missing enforcement; YAML becomes single source of truth replacing prose rules
- Costs: YAML needs maintenance as architecture evolves; another artifact to keep in sync
- Complexity: Low (~150 lines, pure file I/O + set operations)

### Plan-Scoped Write Gate
Connect helm's existing plan_parser.py to existing scope_guard.py — when /implement_plan starts a phase, populate .claude/scope.json from the plan's ### Files list.
- Gains: Makes plan-driven execution trustworthy; prevents scope creep in autonomous agents; zero new hooks needed
- Costs: Requires plans to have complete ### Files lists (95% already do); older plans need fallback
- Complexity: Low (~100 lines, reuses existing infrastructure)

### Agent Prompt-Tool Alignment Checker
A pytest that cross-validates default tool list ↔ @register_tool decorators ↔ prompt text references. Start as test, promote to hook when multiple agents exist.
- Gains: Catches the prompt-tool drift failure mode; scales with new agents
- Costs: Prompt text parsing is approximate (regex, not AST); only one agent exists today
- Complexity: Low (~100 lines as a pytest)

## Chosen Approach
**All three, tiered.** Contract Completeness Linter first (highest evidence, verified gaps). Plan-Scoped Write Gate second (structural play for autonomous execution). Agent Alignment Checker third (scaling risk, start as test).

## Key Context Discovered During Shaping
- 8 specific gaps between prose rules and import-linter contracts (verified by scanning all .py files in components/ and bases/)
- `tool_*` bricks (4 components, 3 bases) have zero contracts in root pyproject.toml; tooling contracts exist only in tools/pyproject.toml
- Product/tooling boundary contracts commented out pending ENG-2484 Phase 2 namespace flatten
- `dynamodb` can import `lambda_utils` today with nothing catching it
- `post_slack_message` is registered but excluded from default tool list — discovered retroactively, not by design
- helm's plan_parser.py already has FILES_SECTION_PATTERN regex that parses 252 ### Files sections across 57 plans
- scope_guard.py already reads .claude/scope.json and does fnmatch — zero new hooks needed for plan-scoped gate

## Next Step
- [Plan] → `/create_plan memory-bank/thoughts/shared/briefs/2026-03-16-custom-deterministic-agentic-backpressure.md`
