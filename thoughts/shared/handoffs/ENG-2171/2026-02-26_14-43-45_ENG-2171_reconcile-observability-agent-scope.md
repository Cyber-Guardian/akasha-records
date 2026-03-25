---
date: 2026-02-26T19:43:45+0000
author: claude-opus
git_commit: 8684a3dace521fd8ea5d42fae28e13cc68771a4f
branch: main
repository: filescience
topic: "ENG-2171 Observability Reconciliation + Agent Scope Fixes Handoff"
tags: [handoff, agent-ci-feedback, scope-creep, observability, cyrus]
status: complete
last_updated: 2026-02-26
last_updated_by: claude-opus
---

# Handoff: Reconcile ENG-2171 observability sub-issues after Cyrus scope creep

## Task(s)

### 1. Fix agent CI feedback loop — COMPLETED
Cyrus (Linear's AI agent) created PR #26 for ENG-2176 but bundled 3 issues into 1 PR. The CI feedback workflow burned through 3 retries in ~1 minute because Lint+Test failures each counted as separate attempts, and duplicate Cyrus sessions were spawned.

**Fix committed to main** (`c0ee3a8`): Added 5 guards to `.github/workflows/agent-ci-feedback.yml`:
- Concurrency group per branch (serializes parallel Lint+Test failures)
- PR HEAD staleness check (skips feedback for old commits)
- Active session detection (skips if Cyrus session already running)
- SHA-based dedup (one feedback comment per unique commit)
- Round-based retry counting (distinct commit SHAs, not raw CI failures)

### 2. Agent scope creep prevention — COMPLETED
Research confirmed Cyrus runs Claude Agent SDK with `settingSources: ["project"]` — reads CLAUDE.md, loads `.claude/agents/*.md`, fires `.claude/settings.json` hooks. All tools enabled including Task (can use plan-implementer).

**Fix committed to main** (`8684a3d`): Added "Agent scope discipline" section to CLAUDE.md:
- Only modify files in assigned issue scope
- One issue = one branch = one PR
- Discover out-of-scope work → comment, don't do it
- 3+ file changes → use plan-implementer subagent

### 3. Reconcile ENG-2171 sub-issues — PLANNED (not started)
Assessed PR #26 quality per sub-issue. Recommendation: close PR #26, re-delegate issues to Cyrus one at a time with new scope rules.

## Critical References
- Plan (updated with Phase 1.1 incident + fixes): `memory-bank/thoughts/shared/plans/2026-02-26-agent-ci-feedback-loop.md`
- Brief (scope creep shaping): `memory-bank/thoughts/shared/briefs/2026-02-26-agent-scope-creep-prevention.md`
- PR #26 (Cyrus's scope-creeping PR): https://github.com/Cyber-Guardian/filescience/pull/26

## Recent changes
- `.github/workflows/agent-ci-feedback.yml` — complete rewrite of guard logic (concurrency, HEAD check, session detection, SHA dedup, round counting, better log extraction)
- `CLAUDE.md:72-78` — new "Agent scope discipline" subsection in agent execution discipline
- `memory-bank/durable/01-active/current_work.md` — updated with both fixes

## Learnings

### Cyrus internals (key finding)
- Cyrus is open-source: `ceedaragents/cyrus` on GitHub
- Runs Claude Agent SDK `query()` with `settingSources: ["user", "project", "local"]`
- This means: CLAUDE.md read, `.claude/settings.json` hooks fire, `.claude/agents/*.md` discovered
- `CLAUDE_CODE_ENABLE_TASKS: "true"` = task tracking (NOT subagents)
- `CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS: "1"` = agent teams enabled
- All tools enabled for our repo → Task tool available → plan-implementer usable
- `appendInstruction` in Cyrus repo config is an additional lever for per-repo instructions
- `cyrus-setup.sh` at repo root runs before each session with `LINEAR_ISSUE_ID` env var

### CI feedback loop failure modes
- `workflow_run` fires separately for each parent workflow (Lint, Test) — need concurrency group
- Linear "session start" comments: `"This thread is for an agent session with cyrus."` (author: null)
- Session end markers: `"stopping work"`, `"unassigned"`, `"out of extra usage"`
- Log extraction: coverage tables fill 3000-char limit, burying actual diff-cover failure message — fixed by extracting error summary lines first

## Artifacts
- `memory-bank/thoughts/shared/plans/2026-02-26-agent-ci-feedback-loop.md` — updated with Phase 1.1 incident analysis + fix details
- `memory-bank/thoughts/shared/briefs/2026-02-26-agent-scope-creep-prevention.md` — new brief for scope creep prevention
- `memory-bank/durable/01-active/current_work.md` — updated
- `.github/workflows/agent-ci-feedback.yml` — updated workflow
- `CLAUDE.md` — updated with scope discipline

## Action Items & Next Steps

### Immediate: Reconcile PR #26 / ENG-2171 sub-issues
1. **Close PR #26** — don't merge (scope-creeping, partial work, failing CI)
2. **Remove Cyrus delegate** from ENG-2174, ENG-2176, ENG-2178 (reset to assignee-only)
3. **Re-delegate one at a time** starting with ENG-2176 (EDT, closest to done):

**ENG-2171 sub-issue status map:**

| Phase | Issue | Status | On Main? | In PR #26? | Action |
|-------|-------|--------|----------|-----------|--------|
| 1 | ENG-2172 VM Stack | Done | Yes | - | None |
| 2 | ENG-2173 Telemetry Component | Done | Yes (PR #23) | - | None |
| 3 | ENG-2174 Discover Migration | In Progress | No | Yes (partial) | **Redo** — PR #26 only adds imports, doesn't delete old module or move tests |
| 4 | ENG-2175 VQP Migration | Investigation | No | No | Not started, co-dev label, highest risk |
| 5 | ENG-2176 EDT Instrumentation | In Progress | No | Yes (decent) | **Redo** — close to done but needs diff-cover fix (4 lines) |
| 6 | ENG-2177 Harness Framework | Investigation | No | No | Not started |
| 7 | ENG-2178 Grafana Dashboards | In Progress | No | Yes (JSON only) | **Redo** — pure JSON, low risk |

4. **After each re-delegation**: verify Cyrus reads new CLAUDE.md scope rules and creates scoped PRs
5. **ENG-2175 (VQP)** is `co-dev` — don't delegate to Cyrus autonomously (highest risk, X-Ray removal)

### Pending verification for CI feedback fixes
- Verify fixes work on next agent PR CI failure (concurrency, dedup, session detection)
- Known limitation: if agent session ends while CI is still failing and no new commits trigger CI, feedback won't fire (no event). Human sees failed status on PR directly.

## Other Notes
- Both fix commits are pushed to main and live
- The old-style `<!-- agent-ci-feedback -->` comments (without SHA) from before the fix won't match the new regex — effectively a fresh start for retry counting
- ENG-2176 Linear issue still shows "In Progress" with Cyrus as delegate — needs cleanup
- PR #26 has 5 commits from cyrusagent, CI status: pytest FAILURE (diff-cover 17%), ruff SUCCESS, architecture SUCCESS
