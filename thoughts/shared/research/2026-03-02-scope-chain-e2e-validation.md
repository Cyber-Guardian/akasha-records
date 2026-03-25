# Scope Chain E2E Validation — ENG-2228

**Date:** 2026-03-02
**Issue:** [ENG-2228](https://linear.app/filescience/issue/ENG-2228)

## What Was Validated

The scope enforcement chain: `cyrus-setup.sh` -> `generate_scope.py` -> `scope.json` -> `scope_guard.py` / `scope_guard_bash.py`

## Phase 1: generate_scope.py Regex Fix

**Problem:** Pattern 1 regex had an optional keyword group (`?`) that matched ANY `- \`code\`` line, producing noise entries like `MailFolder`, `delta_tracked=True`, `dispatcher.py:117`.

**Fix:** Made keyword group required and enforced space after keyword:
- Before: `r"^-\s+(?:CREATE|EDIT|MODIFY|DELETE|NEW)?\s*\`([^\`]+)\`"`
- After: `r"^-\s+(?:CREATE|EDIT|MODIFY|DELETE|NEW)\s+\`([^\`]+)\`"`

**Results:**
- Outlook plan: 17 noise entries -> 0 noise, 14 valid file paths preserved (via Pattern 0)
- Slack-relay-hook plan: all 5 real keyword matches preserved
- 13 new tests in `test/scripts/test_generate_scope.py`

## Phase 2: Local Chain Validation

All checks passed:

| Component | Test | Result |
|-----------|------|--------|
| cyrus-setup.sh | Happy path (ENG-2176) | Plan found by content, 5 paths generated |
| cyrus-setup.sh | Fallback (no match) | Minimal always-allowed scope (5 paths) |
| cyrus-setup.sh | Debug logging | SCRIPT_DIR, PLANS_DIR, plan count, path count all captured |
| scope_guard.py | In-scope file | ALLOW (empty JSON) |
| scope_guard.py | Out-of-scope file | BLOCK with reason |
| scope_guard_bash.py | Out-of-scope write | BLOCK with reason |
| scope_guard_bash.py | Safe command | ALLOW |
| scope_guard_bash.py | In-scope write | ALLOW |
| scope_guard.log | Enforcement decisions | All ALLOW/BLOCK decisions logged |

## Phase 3: Cyrus E2E — Status

**Blocker:** `cyrus-setup.sh` is not wired in any discoverable config (`.claude/settings.json`, `.claude/agents/`). It needs to be configured in Linear's Cyrus platform as a `setupScript` or equivalent mechanism.

**What's known:**
- Cyrus runs Claude Agent SDK with `settingSources: ["project"]` — reads CLAUDE.md, loads `.claude/agents/`, fires hooks
- PreToolUse hooks (scope_guard.py, scope_guard_bash.py) are already wired in `.claude/settings.json`
- The missing link is: who calls `cyrus-setup.sh` to generate `.claude/scope.json` before the session starts?

**Options to investigate:**
1. Linear Cyrus SDK `setupScript` config (external to this repo)
2. Convert to a Claude Code hook (e.g., a session-start hook if available)
3. Add scope generation to CLAUDE.md startup instructions (agent self-generates scope)

## Remaining Gaps

1. **Cyrus wiring** — Phase 3 blocked on confirming how to trigger cyrus-setup.sh
2. **Comprehensive scope guard tests** — follow-up issue for ~32 tests covering scope_guard.py, scope_guard_bash.py, check_scope.py
