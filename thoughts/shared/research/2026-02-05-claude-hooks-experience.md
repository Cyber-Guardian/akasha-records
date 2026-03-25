# Claude Hooks Experience: Ruff and Ty Validators

**Date**: 2026-02-05
**Context**: Implementing Phase 4 of concurrency gates plan with PostToolUse hooks for ruff and ty validation

## Summary

Used ruff (linter) and ty (type checker) hooks during implementation of Outlook gate specs. The hooks run automatically after each Edit tool use, blocking commits that introduce lint or type errors.

## What Works Well

### Immediate Feedback
- Catches issues right after each edit, before moving on
- Prevents cascading errors where subsequent work builds on broken code
- Forces discipline: every edit leaves the codebase in a valid state

### Rich Error Context
- Error output includes line numbers, code snippets, and `help:` suggestions
- Easy to understand what's wrong and how to fix it
- Example output:
  ```
  F401 [*] `filescience.throttling.models.ConcurrencyGateSpec` imported but unused
   --> components/filescience/throttling/policies/m365/gate_specs.py:7:43
  help: Remove unused import
  ```

### Enforces Code Quality
- No way to "forget" to run the linter
- Type errors caught before tests even run
- Consistent enforcement across all edits

## Pain Points

### 1. Atomic Multi-Step Changes

**The Problem**: Many changes require multiple edits that are only valid together.

**Example**: Adding an import and using it
1. Add `from foo import Bar` → Hook blocks: "unused import"
2. Can't proceed to add the code that uses `Bar`

**Workaround**: Make both changes in a single Edit with a large `old_string` match. This is fragile and can fail if the match isn't unique.

### 2. Pre-existing Issues Block Progress

**The Problem**: Hooks flag issues that existed before my changes.

**Example**: `models.py` had a pre-existing TYPE_CHECKING circular import issue. When I added one line to an enum, the hook blocked on the unrelated pre-existing issue.

**Impact**:
- Forces scope creep (I fixed it, but it wasn't my task)
- Could block critical fixes if pre-existing issues are complex

### 3. Duplicate Hook Execution

**Observation**: Error messages appeared twice in output, suggesting hooks may run multiple times per edit.

### 4. All-or-Nothing Severity

**The Problem**: Unused imports and syntax errors are treated the same - both block.

**Reality**: An unused import during a multi-step change is a temporary state, not a real error.

## Improvement Ideas

### 1. Transaction Mode
```
// Begin transaction - defer validation
Edit: add import
Edit: add function using import
// End transaction - validate now
```

### 2. Diff-Aware Checking
Only error on issues *introduced* by the current edit:
- Track baseline issues before edit
- After edit, only block on new issues
- Pre-existing issues become warnings, not blockers

### 3. Severity Tiers
| Severity | Example | Behavior |
|----------|---------|----------|
| Error | Syntax error, undefined name | Block |
| Warning | Unused import, type mismatch | Warn, don't block |
| Info | Style suggestions | Log only |

### 4. Intent-Aware Validation
If the edit adds an import AND references it (even if in different parts of the file), don't flag as unused.

## Metrics from This Session

- **Edits attempted**: ~12
- **Blocked by hooks**: ~6 (50%)
- **Blocks due to incomplete multi-step**: 4
- **Blocks due to pre-existing issues**: 2
- **Legitimate catches**: 0 (all blocks were false positives in context)

## Verdict

**Net positive** - The discipline is valuable and catches real issues. But the 50% false-positive block rate for legitimate multi-step edits creates significant friction.

**Recommendation**: Implement diff-aware checking as the highest-impact improvement. This would eliminate pre-existing issue blocks and could potentially recognize "add import + use it" as a single logical change.

## Related Files

- Hook validators: `.claude/hooks/validators/ruff.py`, `.claude/hooks/validators/ty.py`
- Hook config: `.claude/settings.json` (PostToolUse hooks section)
