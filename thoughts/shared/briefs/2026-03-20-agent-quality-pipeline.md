---
type: event
created: 2026-03-20
status: active
---
# Idea Brief: Agent Quality Pipeline

**Date:** 2026-03-20
**Status:** Shaped → Parked

## Problem
Agents optimize for "make tests pass" and take shortcuts — blanket `#noqa`, code bloat, no dead code removal. No refactor phase exists anywhere in the system (the word "refactor" only appears as a prohibition in agent prompts). The adversarial code reviewer is blind — Read/Glob/Grep only, no runtime data. Subagent context is free-form prose with no structured curation beyond ~2,500-token upstream summaries.

## Constraints
- `noqa_guard.py` blocks C901/PLR0912/PLR0915 in new writes, but agents write bloated-but-passing code instead of decomposing
- 3 pre-existing complexity suppressions slipped through before hooks existed
- Claude Code agent dispatch is prompt-driven — skills + mcpServers in frontmatter are the structured levers
- The `simplify` skill is referenced in the system but doesn't exist — natural home for refactor phase
- implement_plan orchestrator already pauses between phases — inserting refactor step is structurally easy
- `dyno` (brick-level test runner with OTel) exists and no agent uses it proactively

## Options Considered

### Refactor Agent Phase (post-green insertion)
New agent that runs after plan-implementer completes each phase and tests pass, before adversarial review. Gets the diff, the plan, and stricter quality thresholds. Pipeline becomes: implement → test → refactor → test → adversarial review.
- Gains: Directly fixes core problem. Separation of concerns mirrors how humans work.
- Costs: Extra agent dispatch per phase (~2-4min). Risk of breaking what implementer built (mitigated by re-running tests).
- Complexity: Medium

### Enriched Adversarial Reviewer (upgrade in place)
Give adversarial-code-reviewer dev_harness MCP, Edit tools, two-pass workflow (run code first, review with evidence). Combines review + cleanup in one agent.
- Gains: Single agent, no orchestrator change. Solves "blind reviewer" and "no refactor" together.
- Costs: Conflates review and editing. More complex prompt. Stretches reviewer role.
- Complexity: Medium

### Context Packages + Quality Pipeline (system-level redesign)
Structured context curation layer with YAML/markdown templates per agent type, assembled from plan + codebase analysis + observability data before dispatch.
- Gains: Holistic solution. Every agent gets exactly the context it needs.
- Costs: Significant build-out. Over-engineering risk.
- Complexity: High

## Chosen Approach
**Refactor Agent Phase + targeted upgrades** — Build the refactor agent (natural home: `simplify` skill), insert it into implement_plan after green tests. Bolt on two targeted pieces from other approaches:
1. Add `dev_harness` to adversarial reviewer frontmatter + prescribe `dyno` in its workflow (no role change)
2. Formalize refactor agent context as lightweight template (plan excerpt + diff + ruff complexity report)

## Key Context Discovered During Shaping
- `simplify` skill is listed as available but doesn't exist yet — ready-made slot for refactor agent
- adversarial-code-reviewer has NO dev_harness in frontmatter — one-line fix: `.claude/agents/adversarial-code-reviewer.md`
- `dyno` tool runs brick tests with OTel instrumentation but zero agents use it proactively
- plan-implementer already has `mcpServers: [dev_harness]` but only uses `check_stack` + `query_logs` as "best effort"
- Both codebase-analyzer and codebase-locator explicitly prohibit suggesting refactoring — this is a cultural constraint baked into agent prompts
- implement_plan orchestrator pauses between phases — insertion point exists at `.claude/commands/implement_plan.md`

## Next Step
- [Parked] → Linear backlog issue: ENG-2730
