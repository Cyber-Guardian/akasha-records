---
type: event
created: 2026-03-17
status: active
---
# Idea Brief: Worktree Merge Enforcement (Defense-in-Depth)

**Date:** 2026-03-17
**Status:** Shaped -> Planning

## Problem
Worktree agents implement features and push to branches, but no PR is created to merge them to main. Plans are marked "archived/done" based on agent success reports without verifying code landed on main. Result: 2500+ lines of working code sat on orphan branches for days, discovered only when the system failed at runtime.

## Constraints
- Worktree isolation is necessary (branch-switching clobber bug is real)
- Sessions end with no awareness of unmerged branches from previous sessions
- implement_plan skill has no mandatory PR step
- Plan status goes directly active -> archived with no intermediate state
- SessionStart context already loads git log and branch info

## Options Considered

### PR-or-Bust Skill Gate
Modify implement_plan to make PR creation mandatory after last phase. Plan status: active -> in-review -> archived.
- Gains: Prevents the gap at source, structural not disciplinary
- Costs: Some trivial plans don't need PRs
- Complexity: Low

### Orphan Detector Session Hook
Session-start check: git branch -r --no-merged main cross-referenced with gh pr list. Flags branches with no open PR.
- Gains: Safety net across session boundaries, catches all orphan work
- Costs: Doesn't prevent the gap, only detects after the fact
- Complexity: Medium

### Worktree Agent PostToolUse Reminder
PostToolUse hook on Agent tool when worktree agent returns. Reminds main session to create PR.
- Gains: Catches the exact moment the gap opens
- Costs: Just a reminder, can be ignored, single-session only
- Complexity: Medium

### Defense-in-Depth (Combined)
PR gate in implement_plan + orphan detector at session start.
- Gains: Prevention + detection, no single point of failure
- Costs: Two things to implement
- Complexity: Medium

## Chosen Approach
**Defense-in-Depth** — PR gate prevents the common case, orphan detector catches edge cases (crashed sessions, manual worktree work, forgotten branches). Layered defense follows the same pattern as IAM enforcement (pike + simulation + smoke test).

## Key Context Discovered During Shaping
- worktree-isolation.md line 15 already says "Main session creates PR" — rule existed but wasn't enforced
- implement_plan skill has no PR creation step and no plan status intermediate state
- 8 features across multiple sessions were lost — the gap opens at session boundaries
- Plan status values (active/archived) have no "implemented but not merged" state
- SessionStart hook context already loads recent git log — extending it is natural

## Next Step
- Plan -> `/create_plan memory-bank/thoughts/shared/briefs/2026-03-17-worktree-merge-enforcement.md`
