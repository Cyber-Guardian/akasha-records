# Helm MVP — Weaknesses & Gaps

**Date:** 2026-03-03
**Context:** Post-MVP analysis of Helm V1 execution layer (Phases 1-5 merged to main)

---

## Critical (could cause wrong behavior silently)

### W1. Dual Source of Truth
**Where:** CLI uses local `.claude/helm/*.json` manifests (gitignored). GH Action uses Linear.
**Risk:** Drift between the two. Manual Linear edits aren't reflected in manifests. Manifest deletion (worktree cleanup, machine switch) loses merge order and blocking info.
**Impact:** Merge order could be wrong; status dashboard shows stale data.

### W2. Silent Error Swallowing
**Where:**
- `status.py:196` — bare `except Exception` on per-sub-issue Linear fetches hides API failures
- `conflict.py:57-60` — bad git ref (exit 128) treated as "no conflict" hides real errors
- `merge.py:109-111` — topo sort cycles silently resolved by appending remaining in original order
**Risk:** User sees "no conflicts" or "correct order" when the tool actually failed to check.
**Impact:** Merging in wrong order or merging conflicting PRs.

### W3. PR List Hard Cap at 100
**Where:** `status.py:158` (`--limit 100`), `helm-monitor.yml:108` (`per_page: 100`)
**Risk:** Repos with >100 open PRs silently drop matches.
**Impact:** Sub-issues shown as "no PR" when they have one. Merge skips ready PRs.

---

## High (architectural debt, fragile coupling)

### W4. Cross-Module Private Imports
**Where:** `merge.py:25` imports `_fetch_pr_map` and `_match_prs_to_sub_issues` from `status.py`
**Risk:** Underscore-prefixed "private" helpers used across module boundaries. If status.py internals change, merge.py breaks with no contract enforcement.
**Fix:** Extract shared functions to a `pr_utils.py` module or make them public.

### W5. PR Matching Reimplemented in Two Languages
**Where:** Python (`status.py:76`) and JavaScript (`helm-monitor.yml:124`)
**Risk:** Same logic, two implementations, no shared contract. One could diverge.
**Fix:** Define the matching contract explicitly (e.g., branch must contain `{issue-id}` case-insensitive). Consider extracting to a shared spec or at minimum linking the two with comments.

### W6. `find_plan_for_issue` Uses Relative Path
**Where:** `plan_parser.py:99` defaults to `"memory-bank/thoughts/shared/plans"` instead of `PLANS_DIR` from `config.py`
**Risk:** Only works when CWD is repo root. Fails silently (returns `None`) from other directories.
**Fix:** Use `PLANS_DIR` from config, or resolve relative to repo root.

### W7. `sys.exit(1)` vs `raise typer.Exit` Inconsistency
**Where:** `decompose.py:238` uses `sys.exit(1)`, others use `typer.Exit(code=1)`
**Risk:** `sys.exit` bypasses Typer's cleanup. Harder to test (need `pytest.raises(SystemExit)`). Inconsistent behavior.
**Fix:** Standardize on `raise typer.Exit(code=1)` everywhere.

---

## Medium (missing features, operational gaps)

### W8. Missing Commands
- **`helm list`** — no way to see all active orchestrations
- **`helm cancel`** — no way to cleanly abort an orchestration
- **`helm validate`** — no pre-flight check before decompose (plan parses? Linear reachable? API key set?)

### W9. No End-to-End Lifecycle Test
128 unit/integration tests but no single test that runs `decompose -> status -> merge`. Each command tested in isolation. Integration bugs between them (e.g., manifest format changes) won't surface.

### W10. Recovery Gaps
- `MERGING` state blocks re-entry — if merge fails halfway, `retry` only resets one sub-issue, not the merge state machine
- No bulk retry
- No way to manually override merge order
- Manifest deletion = start over from scratch (no Linear-based reconstruction)

### W11. GH Action is Untested JavaScript
~300 lines of JavaScript in YAML with no unit tests. Linear GraphQL queries, dedup logic, escalation thresholds — all untested beyond manual `workflow_dispatch`.

### W12. Stale Threshold is Fixed
24-hour stale threshold in GH Action hardcoded. Some tasks legitimately take longer. No per-orchestration configuration.

---

## Low (polish, minor UX)

### W13. No Progress Feedback During Merge
`_execute_merge_loop` in `merge.py` prints minimal output. No progress bar, no ETA, no "merging 2/5" indicator.

### W14. `design` Command is a Dead Stub
`cli.py:51-54` echoes "V2 placeholder" and exits 0. Could confuse users who expect functionality.

### W15. Config Fallback to Empty API Key
`config.py` defaults `LINEAR_API_KEY` to `""`. Many code paths check `if not LINEAR_API_KEY` but the error messages vary in helpfulness.

### W16. No Audit Trail
Beyond Linear comments, there's no local log of what helm did — no `helm.log`, no structured event history. If Linear comments get deleted, history is lost.

---

### W17. Decompose Assigns All Sub-Issues Regardless of Dependencies
**Where:** `decompose.py:215` assigns `CYRUS_USER_ID` to every sub-issue unconditionally.
**Risk:** Agent picks up all sub-issues simultaneously. Blocked phases start work before their dependencies merge, causing conflicts and wasted effort.
**Impact:** Discovered during dogfood run — all 5 phases assigned to Cyrus at once despite 4 having `blocked_by` dependencies.
**Fix:** Only assign/delegate unblocked phases (empty `blocked_by`). Leave blocked phases in Todo with no assignee. After each phase merges, promote newly-unblocked phases.

### W18. No Phase Promotion Command
**Where:** No `helm promote` command exists. After a phase's PR merges, there is no automated or CLI-driven way to advance the orchestration.
**Risk:** Operator must manually find unblocked phases, update Linear state, assign to agent, and update the manifest — error-prone and blocks the operator.
**Impact:** Discovered during dogfood run — Phase 1 PR opened but no way to advance to Phase 2 without manual Linear/manifest surgery.
**Fix:** Add `helm promote ENG-XXXX` that detects unblocked phases (all `blocked_by` are DONE), sets them to In Progress on Linear, assigns to Cyrus, and updates the manifest.

### W19. Sub-Issue Branch Names Contain Parent ID — Auto-Closes Parent on Merge
**Where:** `decompose.py:113` `build_sub_issue_title` formats titles as `[ENG-XXXX] Phase N: ...` where `ENG-XXXX` is the parent ID. Agent derives branch name from the title, producing branches like `cyrus/eng-2242-eng-2240-phase-1-...` that contain both sub-issue and parent IDs.
**Risk:** Linear's GitHub integration pattern-matches issue IDs in branch names. When a sub-issue PR merges, Linear closes both the sub-issue (correct) AND the parent (wrong) because both IDs appear in the branch name.
**Impact:** Discovered during dogfood run — Phase 1 PR merge auto-closed ENG-2240 (parent with 4 remaining phases). Required manual reopen.
**Fix:** Change `build_sub_issue_title` to omit the parent ID prefix. Use format `Phase N of PARENT: title` or similar where the parent reference is in natural language (not a matchable identifier pattern). Alternatively, strip `[ENG-XXXX]` from the title and put the parent reference only in the description body.

---

## Priority Matrix

| # | Severity | Effort | Recommendation |
|---|----------|--------|----------------|
| W1 | Critical | Large | Shape: manifest-as-cache-of-Linear vs Linear-as-truth |
| W2 | Critical | Small | Fix: replace bare excepts with typed handlers + warnings |
| W3 | Critical | Small | Fix: add pagination or raise when truncated |
| W4 | High | Small | Fix: extract to `pr_utils.py` |
| W5 | High | Small | Document: add contract spec + cross-reference comments |
| W6 | High | Small | Fix: use `PLANS_DIR` from config |
| W7 | High | Small | Fix: standardize on `typer.Exit` |
| W8 | Medium | Medium | Build: `helm list`, `helm cancel`, `helm validate` |
| W9 | Medium | Medium | Build: e2e lifecycle test |
| W10 | Medium | Medium | Build: manifest reconstruction from Linear |
| W11 | Medium | Large | Shape: extract JS to testable module or accept risk |
| W12 | Low | Small | Config: make threshold configurable via env var |
| W13 | Low | Small | Polish: add step counter to merge loop |
| W14 | Low | Tiny | Remove or hide `design` command |
| W15 | Low | Small | Polish: consistent "no API key" messaging |
| W16 | Low | Medium | Shape: decide if local audit trail is worth it |
| W17 | High | Small | Fix: only delegate unblocked phases, leave blocked in Todo |
| W18 | Medium | Medium | Build: `helm promote` command for phase advancement |
| W19 | High | Small | Fix: remove parent ID from sub-issue title to prevent auto-close |
