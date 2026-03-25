# Idea Brief: Agent CI Feedback Loop (Event-Driven)

**Date:** 2026-02-26
**Status:** Shaped → Planning

## Problem
When agents (Cyrus) create PRs and CI fails (e.g., diff-cover threshold), there's no feedback loop — a human must notice, debug, and either fix or re-trigger the agent. This breaks the autonomous execution promise and creates a bottleneck on the human operator.

## Constraints
- Agent branches follow naming convention (`cyrus/eng-XXXX-*`) containing the Linear issue ID
- Linear delegation (`delegate` field) is how agents get triggered — Cyrus receives AgentSessionEvent webhooks
- `ci-failure-linear.yml` already exists for main-branch failures (different use case — creates new issues)
- `LINEAR_API_KEY` already in GH secrets; Cyrus's delegate ID is `ee7d5420-eeca-451c-b7ad-e14a3e72f6f4`
- Need max retry limits to prevent infinite loops
- CI is fast (~30s) — round-trip cost of "push → fail → re-trigger → fix → push" is acceptable
- Don't want agents polling/spinning — event-driven only

## Options Considered

### A: Comment with @mention ("CI Feedback Comment")
GH Action detects CI failure on agent branch → parses ENG-XXXX from branch name → looks up Linear issue UUID → posts comment `@Cyrus CI failed: [job] [logs]` → Cyrus receives AgentSessionEvent, reads failure context, fixes, pushes.
- Gains: Context-rich (logs in comment), visible audit trail, doesn't disrupt issue status
- Costs: Need to confirm @mention syntax triggers Cyrus reliably
- Complexity: Low

### B: Re-delegate with status reset
GH Action un-delegates → moves issue to "In Progress" → re-delegates. Failure context as separate comment.
- Gains: Most reliable trigger (delegation is primary trigger path)
- Costs: More API calls, may lose session history, disrupts issue status
- Complexity: Medium

### C: Hybrid — Comment first, re-delegate on timeout
Post comment, wait N minutes, escalate to re-delegation if no response.
- Gains: Best of both
- Costs: Two-phase automation, needs polling/timeout
- Complexity: High

## Chosen Approach
**A (Comment with @mention)** — simplest, builds on existing `ci-failure-linear.yml` pattern, provides full failure context. If @mention doesn't trigger reliably, upgrade to B.

## Key Context Discovered During Shaping
- Linear Agent Session API: `commentCreate` with @mention fires `created` AgentSessionEvent to agent's webhook
- `issueUpdate` with `assigneeId` (re-delegation) fires same event type — available as fallback
- Existing `ci-failure-linear.yml` (lines 1-213) provides the template: fetch failed jobs, extract logs, query Linear API
- Agent branch naming convention makes Linear issue extraction trivial: `cyrus/eng-2173-*` → `ENG-2173`
- PR #23 (ENG-2173) is the concrete case: diff-cover at 76% vs 80% threshold on `telemetry/core.py`

## Next Step
- [Plan] → `/create_plan memory-bank/thoughts/shared/briefs/2026-02-26-agent-ci-feedback-loop.md`
