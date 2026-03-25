# Idea Brief: PR Contract + Review Pipeline (Unified into Helm)

**Date:** 2026-03-03
**Status:** Shaped → Planning

## Problem
Between "agent writes code" and "code merges to main," there's a verification gap. Write-time guardrails (ruff/ty/import-linter) catch structural issues, but nobody — human or machine — performs semantic review of whether a change is correct, complete, and appropriate for the problem it claims to solve. Additionally, helm's merge loop only rebases the next PR (not all remaining), and has no review gate before squash-merging. As helm parallelizes more work, both the review gap and merge conflict problem get worse.

## Constraints
- Build, not buy — we control the rules, full codebase context is free with Claude agents
- Must compose with existing helm merge flow (extend `_execute_merge_loop`, not replace it)
- Plans already exist per-feature — there's a spec artifact we can verify against
- Pieces must be independently shippable (contract first, then checks, then agent)
- Token cost of adversarial review per PR is acceptable
- Agent-delegated conflict resolution requires spawning sessions (currently outside helm's scope — may use `claude -p` or similar)

## Research Context
- Latent Space "Kill the Code Review": 5-layer stack (competitive gen, deterministic guardrails, human-defined acceptance criteria, permission systems, adversarial verification)
- CodeRabbit data: AI code has 1.7x more issues, 75% more logic errors, 3x readability problems
- Addy Osmani: PR contracts (intent + proof + risk + AI disclosure), human sign-off non-negotiable
- Qodo 2026 patterns: context-first review, severity-driven triage, specialist agents
- Anti-slop practices: enforced agents, context hygiene, phase-based workflows

## Options Considered

### Adversarial Review Agent (standalone)
Dedicated agent reviews PRs against plan acceptance criteria, produces structured verdict.
- Gains: Full codebase context, references plan intent
- Costs: Doesn't address merge conflicts, no contract format, no deterministic pre-check
- Complexity: Medium

### Spec-Driven Verification Criteria (standalone)
Add formal "what must be TRUE" to plans, enforce via /validate_plan.
- Gains: Moves human checkpoint upstream, highest single-piece ROI
- Costs: Doesn't address PR-level review or merge conflicts
- Complexity: Low-Medium

### PR Contract + Review Pipeline (unified into helm)
Multi-stage: PR contract auto-generation → deterministic verification → adversarial review → merge with rebase-all-remaining → conflict resolution agent. All wired into helm's merge loop.
- Gains: Complete solution — review + merge + conflict resolution in one flow
- Costs: Most complex, multiple moving parts
- Complexity: High (but independently shippable pieces)

### Guardrail Layer 0 Completion
Close existing L0 gaps (ty in CI, mutation gate) + blast-radius diff check.
- Gains: Immediate value from closing known gaps
- Costs: Doesn't address semantic review or merge conflicts
- Complexity: Low

## Chosen Approach
**PR Contract + Review Pipeline (unified into helm)** — the complete solution that addresses both the semantic review gap and the merge conflict problem. Pieces are independently shippable: PR contract format, verification criteria in plans, rebase-all-remaining, adversarial review agent, conflict resolution agent, helm integration.

## Key Context Discovered During Shaping
- Helm already has rebase-after-merge but only for `steps[i+1]` not all remaining (`merge.py:344-347`)
- Helm already has conflict detection via `git merge-tree` (`conflict.py:39-72`) but conflicts don't block merge
- Natural insertion point for review: `merge.py:328` (before `_merge_single_pr`)
- Plan files already exist per-feature — verification criteria would extend existing template
- `/resolve-linear-issue` creates PRs — natural place to auto-generate PR contracts
- Existing CI: pytest + diff-cover(80%), ruff, import-linter, pip-audit, detect-secrets, scope-check
- Existing hooks: scope_guard (PreToolUse), ruff/ty/import-linter (PostToolUse), TaskCompleted gate, Stop closeout

## Next Step
- [Plan] → `/create_plan memory-bank/thoughts/shared/briefs/2026-03-03-pr-review-pipeline.md`
