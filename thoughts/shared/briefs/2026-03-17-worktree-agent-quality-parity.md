---
type: event
created: 2026-03-17
status: active
---
# Idea Brief: Worktree Agent Quality Parity

**Date:** 2026-03-17
**Status:** Shaped → Planning

## Problem
Worktree agents (plan-implementer subagents) bypass the backpressure systems built for the main Claude Code instance. Of 23 enforcement mechanisms, 4 are broken in worktrees — the 3 most critical real-time validators (ruff, ty, import-linter PostToolUse hooks) and the task completion gate. Agents get zero lint/type/architecture feedback as they write code, then suppress violations with `# noqa` when they hit them during manual verification. The backpressure investment is wasted for the primary implementation model.

## Constraints
- The `.claude` path skip guard in validators exists for a valid reason — hook scripts, agent definitions, and skill markdown should not be linted
- Worktree paths are structurally `.claude/worktrees/agent-XXXX/<repo-mirror>/...` — the `.claude` prefix is baked in
- The 3 validators share identical skip logic — one pattern fix covers all three
- `task_completed_gate.sh` is a separate class of problem (CWD, not path guard)
- Rules files load correctly in worktree agents — the policy layer works
- Pre-commit and pre-push git hooks already work in worktrees, but agents bypass with `--no-verify`

## Options Considered

### A: Surgical Hook Fix
Fix the 4 broken mechanisms only. Refine `.claude` path guard, fix task gate CWD.
- Gains: Real-time feedback restored, cheapest fix
- Costs: Doesn't address behavioral layer — agents can still suppress with noqa
- Complexity: Low

### B: Surgical Fix + Noqa Policy Rule
Fix A plus add `.claude/rules/noqa-discipline.md` and update plan-implementer prompt.
- Gains: Closes mechanical and behavioral gap
- Costs: Rule is guidance, not enforcement — relies on agent compliance
- Complexity: Low

### C: Full Agent Quality Parity — Hooks + Policy + Enforcement Gate
Fix A + B plus a PreToolUse hook that blocks writes containing complexity `# noqa` comments. Hard gate — agents cannot write complexity suppressions.
- Gains: Mechanically impossible to ship noqa-suppressed complexity. Three-layer defense (feedback + policy + enforcement). Legitimate edge cases get surfaced to humans.
- Costs: ~20 extra lines for the PreToolUse hook
- Complexity: Low-Medium

## Chosen Approach
**C: Full Agent Quality Parity** — The backpressure stack only works when the feedback loop is closed: agent writes code → gets lint feedback → knows the right response → physically can't do the wrong thing. C closes all three gaps (mechanical, behavioral, enforcement) with minimal additional machinery.

## Key Context Discovered During Shaping
- `validators/ruff.py:64`, `ty.py:64`, `import_linter.py:74` — the `.claude` path guard (`rel.parts[0] == ".claude"`) was designed to skip hook infrastructure but unintentionally covers all worktree source files
- `task_completed_gate.sh:5` — `cd "$CLAUDE_PROJECT_DIR"` runs tests against main repo, not the worktree
- Zero noqa discipline exists in rules, agent prompts, or CLAUDE.md — `python-conventions.md` documents `ty: ignore` discipline but is silent on `# noqa:`
- Only 1 complexity noqa exists in the entire human-authored codebase (`query.py:99` — C901)
- `scope_guard.py` provides a working PreToolUse content-inspection pattern to follow
- Investigation: [[2026-03-17-debug-worktree-agent-noqa|Worktree Agent Noqa Investigation]] (session log)

## Next Step
- [Plan] → `/create_plan memory-bank/thoughts/shared/briefs/2026-03-17-worktree-agent-quality-parity.md`
