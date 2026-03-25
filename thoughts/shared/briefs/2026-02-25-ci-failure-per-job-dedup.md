# Idea Brief: CI Failure Per-Job Dedup + Comment-Aware Agent

**Date:** 2026-02-25
**Status:** Shaped → Implementing

## Problem
The CI failure → Linear automation deduplicates by workflow name (e.g., "Lint on main"), which conflates different linter failures (ruff, pip-audit, import-linter) into one issue. When the original failure gets fixed but a different linter failure is deduped as a comment, the resolution agent sees the original is fixed and no-ops — missing the unresolved failure in comments. Additionally, the issue description had hardcoded ruff-specific instructions regardless of which job actually failed.

## Constraints
- Dedup key was workflow-level: `CI Failure: {workflowName} on {branch}`
- Issue description always said "Run ruff check --fix" even for non-ruff failures
- Comments carried real failure data but weren't treated as authoritative by the agent
- The resolve-linear-issue skill is generic — no CI-failure-specific awareness

## Options Considered

### Per-Job Issues (structural fix)
Change dedup key to job-level: `CI Failure: {workflowName}/{jobName} on {branch}`. Each failed job gets its own issue. Eliminates conflation entirely.
- Gains: One issue = one failure type. Agent doesn't need CI-specific logic. Fixes hardcoded ruff instructions.
- Costs: More issues when multiple jobs fail simultaneously (but this is correct behavior).
- Complexity: Low

### Comment-Aware Agent (agent-side fix)
Keep current dedup. Add CI-failure-specific instructions to resolve-linear-issue skill so agent checks comments for additional unresolved failures.
- Gains: No workflow changes. Handles edge cases generically.
- Costs: Adds domain-specific logic to generic skill. Agent must parse logs to identify failure types. Doesn't fix misleading description.
- Complexity: Medium

### Dynamic Description + Comment-Aware Agent (belt and suspenders)
Update issue description on dedup (merge failures) + agent awareness.
- Gains: Description always current. Defense in depth.
- Costs: Loses clean original context. Complex merge logic.
- Complexity: Medium-High

## Chosen Approach
**Per-Job Issues + Comment-Aware Agent (hybrid)** — Structural fix prevents the problem going forward (different failure types get separate issues). Comment-awareness in the resolve skill provides defense in depth for any remaining dedup scenarios where comments contain additional context.

## Key Context Discovered During Shaping
- `ci-failure-linear.yml:132` had hardcoded "Run ruff check --fix" instructions regardless of actual failure type
- `ci-failure-linear.yml:98` dedup title was `CI Failure: ${workflowName} on ${branch}` — no job differentiation
- `resolve-linear-issue.md:16` already reads comments but had no CI-specific guidance on treating them as actionable
- ENG-2124 was the concrete incident: ruff fixed, pip-audit deduped as comment, agent no-op'd

## Next Step
- [Implementing] → no plan needed, proceeding directly
