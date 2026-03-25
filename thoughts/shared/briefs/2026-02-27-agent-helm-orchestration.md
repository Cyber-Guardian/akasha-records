# Idea Brief: Agent Helm Orchestration

**Date:** 2026-02-27
**Status:** Shaped → Planning

## Problem
Manual orchestration of Cyrus agents on Linear doesn't scale. Today, the human is the helm — scoping issues, setting scope.json, watching for conflicts, verifying completion, handling merge ordering. This breaks at 2-3 concurrent agents and blocks the move to nightly scheduled agents.

## Constraints
- Cyrus is external (Linear delegates → Claude Agent SDK) — influence via CLAUDE.md, hooks, `appendInstruction`, Linear guidance
- Cyrus already has native `Orchestrator` workflow label (breaks epics into sub-issues)
- Linear API supports full programmatic orchestration (create issues, assign, webhooks, session state)
- File-level non-overlap must be enforced at decomposition time (C compiler experiment: overlapping files = parallelism collapse)
- Cyrus worktrees provide filesystem isolation; merge conflicts surface at PR time
- Existing guardrails (CLAUDE.md scope discipline, scope_guard.py, CI feedback loop) work inside Cyrus sessions
- Agent Teams (experimental) is single-machine, not cross-Linear-session — different abstraction level

## Options Considered

### 1. Cyrus Native Orchestrator
Use built-in `Orchestrator` label. Zero infrastructure.
- Gains: Already exists, no code needed
- Costs: At Cyrus's mercy for decomposition quality, no merge ordering control
- Complexity: Low

### 2. Linear API Orchestrator (custom service)
Build a standalone orchestrator (Lambda/GH Action) that creates sub-issues, assigns to Cyrus, monitors webhooks, runs Clash, sequences merges.
- Gains: Full control over decomposition, file scoping, merge ordering
- Costs: Real infrastructure to build and maintain (webhook handler, state machine, error handling)
- Complexity: Medium-High

### 3. Claude Code as Helm (interactive session)
A `/helm` skill in Claude Code that decomposes, delegates via `/create-linear-issue`, monitors Linear, manages merge ordering.
- Gains: Leverages all existing skills/hooks, no new infrastructure
- Costs: Requires persistent session, context pressure for long-running orchestration
- Complexity: Medium

### 4. Hybrid: Claude Code Helm + Automated Webhook Monitor
Claude Code `/helm` skill for thinking (decompose, plan, approve merges) + lightweight GH Action for mechanical monitoring (session completions, CI results, conflict detection via Clash).
- Gains: Best of both — Claude reasoning for decisions, automation for monitoring. Handles "delegate and walk away."
- Costs: Two moving parts (skill + webhook handler), but both simple individually
- Complexity: Medium

## Chosen Approach
**Hybrid (Option 4)** — Claude Code `/helm` skill for decomposition + judgment calls, GH Action webhook monitor for async monitoring + conflict detection. The helm session is interactive but not permanent — spin up to decompose and delegate, close it, come back to review results. Phased: try Cyrus native Orchestrator first (Phase A), build hybrid if decomposition quality is insufficient (Phase B).

## Key Context Discovered During Shaping
- Cyrus `Orchestrator` label creates sub-issues with Linear blocking relationships automatically
- Linear API `issueUpdate` with `assigneeId` = Cyrus agent ID triggers delegation (no Cyrus API needed)
- Linear webhooks fire on `AgentSession` state changes (created, active, awaitingInput, error, complete, stale)
- Clash (github.com/clash-sh/clash) does read-only `git merge-tree` conflict detection across worktrees
- File-scope non-overlap at decomposition time is the #1 success factor (from C compiler 16-agent experiment)
- `scope.json` infrastructure exists but is dormant (currently `["**"]`) — needs auto-population per branch
- Claude Agent SDK supports programmatic session creation via `claude -p` or Python/TS SDK

## Next Step
- Plan → `/create_plan memory-bank/thoughts/shared/briefs/2026-02-27-agent-helm-orchestration.md`
