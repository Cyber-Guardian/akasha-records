# Idea Brief: CI Failure Detection + Linear-Centric Agent Orchestration

**Date:** 2026-02-23
**Status:** Shaped -> Planning

## Problem

CI failures on `main` go unnoticed until someone manually checks the GitHub Actions tab. Lint and mutmut workflows failed for 12+ consecutive days with no one acting on them. There is no feedback loop from CI failure to tracked work to resolution. More broadly, there is no centralized place where agent-automatable work is queued and dispatched.

## Constraints

- Linear Agent SDK is Developer Preview (since July 2025, 7+ months) — API may change but is well-documented and used by Cursor, Codex, Copilot, Sentry
- Claude Code has no native Linear agent integration (open issue #12925, 51 upvotes, no Anthropic response) — would require a custom bridge service
- Cursor and Codex both have shipping native Linear agent integrations — off-the-shelf, no bridge needed
- GitHub Actions `workflow_run` + `conclusion == 'failure'` is the right trigger for CI failure detection (GHA-native, no infra)
- Linear GraphQL `issueCreate` mutation is stable and trivial to call from `actions/github-script`
- Linear triage rules can auto-delegate issues to agents based on labels, team, or priority — this is the automation primitive
- The triage bot already creates Linear issues from Slack messages — this extends the same "everything becomes a Linear issue" pattern

## Options Considered

### GHA-Direct Auto-Fix (no Linear orchestration)
CI failure triggers `workflow_run`, `claude-code-action@v1` fetches logs and fixes directly in GitHub Actions. Linear issue created only for visibility. This is Anthropic's canonical `ci-failure-auto-fix.yml` pattern.

- Gains: Simplest to build, fastest response, canonical example exists
- Costs: Linear is passive (record keeper, not orchestrator). Each signal source needs its own automation. No single agent work queue. Doesn't extend to non-CI work.
- Complexity: Low

### Linear-Centric with Custom Claude Code Bridge
CI failure creates Linear issue. Linear delegates to a custom agent (our code) that bridges to `claude-code-action` via `repository_dispatch`. Requires a Lambda/Worker webhook receiver to handle Linear's 5-second response requirement.

- Gains: Single work queue for all agent work. Claude Code as the agent. Full control over agent behavior.
- Costs: Claude Code has no native Linear integration — requires building and maintaining a bridge service. More moving parts. Bridge must handle 5-second webhook timeout, session lifecycle, activity reporting.
- Complexity: High

### Linear-Centric with Off-the-Shelf Agent (Cursor or Codex)
CI failure creates Linear issue. Linear triage rule auto-delegates to Cursor or Codex (both have native Linear agent integrations). Agent resolves the issue, opens a PR, updates Linear status — all built-in.

- Gains: Single work queue. Native integration (no bridge). Triage rules as automation primitive. Extends to any Linear issue, not just CI. Audit trail and human gate built-in. Aligns with industry direction (Cursor's own team uses this pattern).
- Costs: Dependent on third-party agent quality. Less control over agent behavior than Claude Code. Monthly cost for Cursor/Codex seats.
- Complexity: Low-Medium

## Chosen Approach

**Linear-Centric with Off-the-Shelf Agent** — Use Cursor or Codex's native Linear integration for agent execution. Build only the CI failure detection piece ourselves (a GitHub Actions workflow that creates Linear issues on failure). Let Linear triage rules handle delegation to the agent.

This avoids building a custom bridge service while still centralizing all agent work through Linear. When/if Claude Code ships a native Linear integration, we can swap the agent without changing the architecture.

**Why not the others:**
- GHA-Direct is fast but fragments work tracking and doesn't generalize beyond CI
- Custom Claude Code bridge is premature — high effort for an integration that Anthropic may ship natively

## Key Context Discovered During Shaping

- Linear is explicitly building toward being an agent orchestration platform — agents are first-class citizens with OAuth identity, session state, and activity lifecycle
- The idiomatic pattern (Feb 2026) is: issue tracker as intake queue + status board, agent as async worker, human as reviewer — Cursor, Cognition (Devin), OpenAI Codex, and GitHub Copilot all converge on this
- Linear triage rules are the key automation primitive — "all issues with label X -> delegate to agent Y" requires zero custom code
- Anthropic's `ci-failure-auto-fix.yml` example covers: `workflow_run` trigger, log fetching via `github-script`, branch creation with loop guard (`claude-auto-fix-ci-*` prefix), and `claude-code-action@v1` invocation — useful reference even if we don't use it directly
- Linear `issueCreate` mutation needs: `title`, `description`, `teamId`, optional `labelIds`, `projectId`, `priority`
- The triage bot (`tools/triage-bot/`) already has a working `linear_client.py` with GraphQL helpers and `tools/shared/linear-config.yaml` with team/project IDs

## Architecture (two pieces)

```
Piece 1: CI Failure Detection (we build this)
  GitHub Actions workflow fails (lint, mutmut, etc.)
    -> workflow_run trigger fires (conclusion == 'failure')
    -> actions/github-script:
       - Fetch failed job names + log excerpts via GitHub API
       - Call Linear GraphQL issueCreate mutation
       - Label: "ci-failure", include run URL + error context

Piece 2: Agent Resolution (off-the-shelf)
  Linear triage rule: label="ci-failure" -> delegate to Cursor/Codex
    -> Agent picks up issue automatically
    -> Reads issue context (error logs, run URL)
    -> Clones repo, analyzes, fixes, opens PR
    -> Updates Linear issue status
    -> Human reviews PR
```

## Next Step

Plan -> `/create_plan memory-bank/thoughts/shared/briefs/2026-02-23-ci-failure-linear-agent-orchestration.md`
