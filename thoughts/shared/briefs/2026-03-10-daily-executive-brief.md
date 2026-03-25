# Idea Brief: Daily Executive Brief

**Date:** 2026-03-10
**Status:** Shaped → Parked

## Problem
No aggregated daily visibility into what the FileScience team shipped. The CEO has to manually check Linear boards and git logs across engineers (Jordan, Niklas, Harsh) to understand progress. Needs something skimmable when busy but detailed when he has time.

## Constraints
- Audience is CEO — business-meaningful narrative, not ticket lists
- Must work in Slack (where he already lives)
- Linear API supports assignee + date range filtering (GraphQL, ISO-8601 durations)
- GitHub Search API supports org-wide commits/PRs per user per day (default branch only, 1K result cap — fine for daily)
- No EventBridge cron infrastructure exists yet — needs new Terraform
- Existing `SlackClient` component is thread-scoped — needs a `post_message` method for channel-root posts
- Secrets already in Secrets Manager: `slack-bot-token`, `linear-api-key`, `anthropic-api-key`
- Triage-agent Makefile/zip deployment pattern is directly reusable
- LLM pass (Haiku 4.5) is effectively free (~$0.000003/run)

## Options Considered

### Narrative + Stats Footer (flat message)
Single daily Slack message with LLM-synthesized narrative and raw numbers as footer.
- Gains: Simple, one message to read
- Costs: No drill-down — either too shallow or too long
- Complexity: Low

### Standup-Style Per-Engineer Breakdown
Shipped / Progressing / Blocked per engineer, no narrative synthesis.
- Gains: Structured, scannable, familiar format
- Costs: Not executive-friendly — reads like an eng standup, not a business update
- Complexity: Low

### Three-Layer Progressive Disclosure (chosen)
Layer 1 (glance): LLM narrative + one-line-per-engineer in channel root. Layer 2 (detail): Per-engineer Shipped/Progressing/Blocked as thread replies with Linear/GitHub links. Layer 3 (weekly): Friday/Monday rollup with velocity trends and blockers.
- Gains: Skimmable when busy, detailed when he has time. Weekly gives trend context. Links let him jump straight to issues/PRs.
- Costs: More complex Slack formatting (Block Kit), weekly mode adds a second Lambda trigger path
- Complexity: Medium

## Chosen Approach
**Three-Layer Progressive Disclosure** — matches the CEO's dual mode (skim vs. dig in). The LLM narrative is the product; raw data is just input. Weekly rollup adds the velocity/trend layer that daily alone can't provide.

## Key Context Discovered During Shaping
- [[2026-02-26-nightly-scheduled-agents|Nightly Scheduled Agents]] brief (parked, ENG-2183) identified "Morning standup digest" as Tier 1 use case — this is a more specific, CEO-focused version
- `SlackClient` at `tools/components/tooling/slack_client/client.py` — all methods thread-scoped, needs `post_message` addition
- `get_secret()` at `tools/components/tooling/config/secrets.py` — reusable, secrets already provisioned
- Triage-agent deployment at `tools/projects/triage-agent/` — Makefile + zip pattern reusable for new Lambda
- No EventBridge resources exist anywhere in `infrastructure/` — confirmed via codebase search
- GitHub Search API: 30 req/min rate limit on search, default-branch-only for commits — not an issue for daily brief of 3 engineers
- Linear API: `completedAt: { gt: "-P1D" }` filter works natively, cursor pagination, 5K req/hr

## Next Step
- [Parked] → Linear backlog issue: [ENG-2438](https://linear.app/filescience/issue/ENG-2438/add-daily-executive-brief-lambda-with-slack-delivery)
- Plan: [[2026-03-11-daily-executive-brief|Daily Executive Brief Plan]]
