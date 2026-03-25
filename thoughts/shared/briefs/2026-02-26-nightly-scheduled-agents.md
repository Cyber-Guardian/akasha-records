# Idea Brief: Nightly Scheduled Agents

**Date:** 2026-02-26
**Status:** Shaped → Parked

## Problem
No mechanism for Cyrus to do useful work overnight — research, codebase health, memory hygiene — and surface results by morning. Currently all agent work is human-triggered.

## Constraints
- Must integrate with existing Cyrus + `/resolve-linear-issue` flow (don't build a parallel execution system)
- Linear as the scheduling surface, result surface, and audit trail
- Cost bounded — Haiku for subagents, cap turns
- Results land in repo (memory-bank/thoughts/) AND Linear (comments, status)
- The "scheduler" piece should be thin (just creates issues with well-formed prompts)

## Options Considered

### A) Linear Recurring Issues — thin cron, full Cyrus
GH Actions cron creates well-formed Linear issues nightly. Cyrus picks them up via existing `/resolve-linear-issue`. Each nightly run is a real issue with comments, status, linkable history.
- Gains: Reuses 100% of existing Cyrus infrastructure. Linear is single surface for scheduling + results + audit.
- Costs: Need "nightly issue template" system. Need Cyrus to auto-trigger on issue creation.
- Complexity: Low

### B) Agent SDK Nightly Runner — Python script with subagents
Python script using `claude-agent-sdk` on cron. Registry of nightly tasks, each runs as Haiku subagent, posts results to Linear via API.
- Gains: More control (retry, cost caps, parallel tasks). Direct Agent SDK features.
- Costs: Bypasses existing Cyrus flow. Two execution paths to maintain.
- Complexity: Medium

### C) Hybrid — claude-code-scheduler + Linear issues
`claude-code-scheduler` plugin schedules local `claude` invocations running `/resolve-linear-issue` against pre-created issues. Manages launchd plists, git worktree isolation.
- Gains: Natural language scheduling. Local MCP access. Worktree isolation.
- Costs: Requires Mac to be awake. Local-only. Community-maintained plugin.
- Complexity: Low

## Chosen Approach
**Linear Recurring Issues (A)** — thinnest new infrastructure, fully reuses Cyrus + `/resolve-linear-issue`, Linear as single surface.

## Key Context Discovered During Shaping

### Execution landscape
- Claude Code headless: `claude -p` with `--dangerously-skip-permissions` or Agent SDK `bypassPermissions`
- Claude Agent SDK (Python): `claude-agent-sdk` package, `ClaudeSDKClient`, `AgentDefinition` for subagents, async-first
- Official research-agent demo: multi-agent pipeline (researcher + data-analyst + report-writer), all Haiku subagents
- GH Actions cron: `anthropics/claude-code-action@v1` supports `schedule:` trigger (known P1 OIDC bug #814 — use `ANTHROPIC_API_KEY` secret directly)
- claude-code-scheduler: cross-platform scheduling plugin (macOS launchd, Linux crontab)
- Cost: ~$3/night Sonnet, ~$0.80/night Haiku, ~$0.15/night Haiku+caching

### Use case tiers (prioritized for FileScience stack)

**Tier 1 — high signal, low effort:**
- Stale TODO/FIXME scanner → Linear issues (grouped by brick, 90-day threshold)
- `pip-audit` CVE monitor → triaged draft PRs or Linear issues
- Morning standup digest → Slack/markdown (PRs, Linear changes, CI failures)

**Tier 2 — medium effort, compounds:**
- Import-linter drift monitoring (coupling trends beyond pass/fail)
- Memory-bank staleness detection (doc-vs-code drift, orphan artifacts)
- AWS deprecation/changelog monitor (boto3, Lambda runtime, DynamoDB API)
- Codebase audit playbooks (perf, security, dead code — capped at 20 findings/run)

**Tier 3 — higher effort, highest ceiling:**
- Pre-research upcoming Linear issues (scoping notes as comments)
- Dependency update PRs (minor/patch only, tests must pass)
- Coverage gap → test generation (recently-modified files only)
- Flaky test root-cause analysis
- Difficulty pre-assessment for autonomous issues

**Novel:**
- Regression test gen from closed bugs
- Knowledge gap detection (code changed, docs not updated)
- Technical debt trend reports (weekly, prioritized by change frequency)

### Key references
- [claude-agent-sdk-python](https://github.com/anthropics/claude-agent-sdk-python)
- [claude-agent-sdk-demos/research-agent](https://github.com/anthropics/claude-agent-sdk-demos/tree/main/research-agent)
- [claude-code-scheduler](https://github.com/jshchnz/claude-code-scheduler)
- [claude-mcp-scheduler](https://github.com/tonybentley/claude-mcp-scheduler)
- [modal-claude-agent-sdk-python](https://github.com/sshh12/modal-claude-agent-sdk-python)
- [anthropics/claude-code-action](https://github.com/anthropics/claude-code-action)
- [GH Actions cron OIDC bug #814](https://github.com/anthropics/claude-code-action/issues/814)
- [Effective harnesses for long-running agents (Anthropic)](https://www.anthropic.com/engineering/effective-harnesses-for-long-running-agents)

## Next Step
- [Parked] → Linear backlog issue: [ENG-2183](https://linear.app/filescience/issue/ENG-2183/add-nightly-scheduled-agent-infrastructure-linear-triggered-cyrus)
