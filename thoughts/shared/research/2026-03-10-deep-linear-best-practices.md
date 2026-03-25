---
date: 2026-03-10T12:00:00-05:00
researcher: Claude
git_commit: 636f79e
branch: fix/ENG-2397-receiver-dedup-s3-memory
repository: filescience
topic: "Linear best practices for project management — workflow design, issue hygiene, triage, labeling, automation, and agentic patterns"
tags: [deep-research, linear, project-management, agentic-pm, extend-targets]
status: complete
research_depth: standard
iterations_completed: 3
last_updated: 2026-03-10
last_updated_by: Claude
---

# Deep Research: Linear Best Practices for Project Management

## Research Question
Best practices for Linear project management — workflow design, issue hygiene, triage patterns, labeling taxonomies, cycle/project structuring, status workflows, and automation patterns. Goal: ground future Claude `/extend` work for improving our Linear integration skills and agents.

## Summary
Linear's opinionated PM model converges around 2-week cycles, ~15-20 grouped labels (`Type/Bug`, `Area/API`), daily rotating triage, and a flat Issue → Cycle → Project → Initiative hierarchy. The Agent Interaction SDK (July 2025) formalizes agent delegation with activity streaming, session states, and non-billable OAuth identities — a pattern Cursor, Devin, and others are adopting. Our current skills (`/create-linear-issue`, `/resolve-linear-issue`) are strong on scope discipline (action-verb titles, max-3 AC, plan lifecycle) but miss 10 key capabilities including Agent SDK adoption, label-group enforcement, triage automation, stale-issue archiving, and webhook-driven workflows. The research identifies 8 concrete `/extend` targets ranging from quick wins (label taxonomy rule) to strategic investments (Agent SDK integration).

## Perspectives Explored
1. **Workflow Architecture** — Revealed status substates, cycle cadence, hierarchy patterns, and triage inbox design for small teams
2. **Issue Hygiene & Taxonomy** — Established label-group conventions, title formats, scope limits, and grooming rituals
3. **Automation & Integration Patterns** — Mapped Linear's full GraphQL API, webhook system, SLA events, and built-in automation
4. **Agentic/AI-Assisted PM** — Documented Cursor/Devin integration patterns, the Agent Interaction SDK, and delegation model
5. **Anti-Patterns & Failure Modes** — Catalogued six compounding failure categories that inform guardrail design

## Detailed Findings

### Workflow Architecture

**Status flows**: Mature teams expand Linear's default 5-status flow (Backlog → Todo → In Progress → Done → Canceled) with granular substates. Common additions:
- **In Review** — code is in PR, awaiting review
- **Ready to Merge** — approved, waiting for merge window
- **Won't Fix / Duplicate / Could Not Reproduce** — specialized cancel variants that preserve triage context

Per-team workflow customization via Settings → Teams means engineering, design, and ops can have distinct flows.
- Source: https://linear.app/docs/configuring-workflows

**Cycle cadence**: Linear recommends 2-week cycles as the default — short enough to maintain focus, long enough to ship meaningful work. Issues roll over automatically, which is a feature but becomes cycle debt without scope trimming.
- Source: https://linear.app/method/introduction

**Hierarchy**: Issues → Cycles (sprint-level) → Projects (deliverable-scoped, multi-team) → Initiatives (strategic umbrella). For 2-5 engineers: single team, one cycle cadence, projects tied directly to initiatives. Projects should be milestone-scoped buckets, not permanent hierarchies.
- Source: https://linear.app/docs/conceptual-model, https://linear.app/docs/initiatives

**Triage**: Shared inbox for externally-created or cross-team issues. Best practice: rotating daily reviewer. Each issue is accepted, duplicated, declined, or snoozed before entering main workflow. Business/Enterprise supports on-call-style rotation via PagerDuty/OpsGenie integration.
- Source: https://linear.app/docs/triage

**Small team specifics**: Estimation is opt-in — Linear recommends breaking large estimates into smaller issues rather than accepting high-point work. Power users keep 2-3 saved views (My Issues excluding Done/Blocked, Open by Project, Triage inbox). Cap WIP at 1-2 issues per person. Key metrics: cycle time (in-progress to done) and throughput per cycle. Teams scaling past 5-6 add Triage status, cap WIP, introduce project leads for weekly updates.
- Source: https://linear.app/now/descript-internal-guide-for-using-linear, https://linear.app/docs/estimates

### Issue Hygiene & Taxonomy

**Labels**: Keep workspace labels to ~15-20 total using label groups with `Group/Label` syntax (e.g., `Type/Bug`, `Area/API`). Archive stale labels rather than deleting (preserves history). Never duplicate built-in fields as labels — priority, status, and assignee already have first-class UI.
- Source: https://linear.app/docs/labels

**Issue titles**: `[Verb] [What] [Context]` pattern — "Fix broken scroll in mobile navbar", "Implement POST /api/v1/users". Vague noun-only titles degrade triage speed because reviewers must open the issue to understand intent.
- Source: https://terem.tech/naming-guide-for-task-bug-and-user-story-titles/

**Scope discipline**: Quality enforced via form templates with required fields per team. Issues should fit within one cycle — if a checklist grows beyond 3-4 items, convert to sub-issues. Weekly 20-30 min grooming ritual. Auto-archive for issues untouched 30+ days. Saved view for "unassigned high-priority issues older than 14 days."
- Source: https://linear.app/docs/issue-templates, https://linear.app/docs/parent-and-sub-issues

### Automation & Integration Patterns

**API surface**: Full GraphQL API identical to their internal API — mutations, real-time subscriptions, all entity types. Rate limits: 1,500 req/min, 250,000 complexity/min standard tier.
- Source: https://linear.app/developers/webhooks, https://developers.linear.app/docs/graphql/working-with-the-graphql-api/rate-limiting

**Webhooks**: Create/update/remove events for all entity types. `updatedFrom` diffs on update events enable change detection without polling. SLA-specific events: `IssueSLA.breaching` and `IssueSLA.breached`.
- Source: https://inventivehq.com/blog/linear-webhooks-guide

**Built-in automation**:
- Auto-close (inactivity threshold) and auto-archive
- Triage Intelligence (AI-powered label/dedup/assignment suggestions with auto-apply)
- Triage Rules (Enterprise) — trigger on priority, creator, template, SLA status; set team/assignee/label/project
- GitHub PR ↔ issue status sync
- Source: https://linear.app/changelog/2020-08-19-auto-close-and-auto-archive, https://linear.app/changelog/2025-06-05-asks-fields-and-triage-routing

**External**: Zapier, Make, n8n, Pipedream extend Linear without code for multi-step workflows.

### Agentic/AI-Assisted PM

**Linear Agent Interaction SDK** (July 2025, Developer Preview):
- Five activity types via `agentActivityCreate`: `thought`, `action`, `elicitation`, `response`, `error`
- `thought` and `action` support an `ephemeral` flag for transient display
- Five auto-managed session states: `pending`, `active`, `awaitingInput`, `error`, `complete`
- OAuth `actor=app` creates non-billable agent workspace identity
- Scopes: `app:assignable` (delegate field), `app:mentionable` (@mention triggers)
- Must emit `thought` within 10 seconds of session creation or be flagged unresponsive
- Delegate filtering supported in custom views and search
- Source: https://linear.app/developers/agent-interaction, https://linear.app/developers/agents

**Cursor integration** (Aug 2025):
- Webhook-based dispatch when issue assigned to Cursor user or @Cursor mentioned in comment
- Ingests full issue context (title, description, comments, linked refs)
- Inline directives: `[repo=owner/repo]`, `[branch=...]`, `[model=...]`
- Scope follows description literally — no autonomous exploration
- Creates branch, commits, opens draft PR linked back to Linear
- Limitations: requires real user for triage rules, ~$4-5/PR, stalls on underspecified issues
- Source: https://linear.app/now/how-cursor-integrated-with-linear-for-agents, https://cursor.com/blog/linear

**Devin integration**: `devin` label triggers sessions, auto-links PRs via `externalUrls`, own pre-merge quality pass.
- Source: https://linear.app/integrations/devin

**Key insight**: No mature "agent lane" workflow exists in the community yet. Our plan-driven, multi-phase approach with review contracts is more sophisticated than Cursor's literal-description execution, but we don't leverage the formal Agent SDK for status streaming or delegation.

### Anti-Patterns & Failure Modes

Six compounding failure categories:

1. **Label sprawl** — Duplicating workspace-level labels at team level, creating one-off labels never cleaned up, shadowing built-in fields. Linear reserves priority/state names to prevent this.

2. **Cycle debt** — Unfinished work rolls over automatically without scope trimming. Overloaded cycles cause fatigue, and the planning ritual collapses. Fix: trim scope at cycle boundaries, cap WIP.

3. **Zombie projects** — No mandatory updates, silent scope expansion goes undetected. Fix: weekly reviews, metric gates (cycle time, throughput), project leads who own status.

4. **Triage failures** — Rotating ownership is a suggestion, not enforcement. Inboxes grow unbounded when no one owns the queue. Fix: explicit daily reviewer rotation, SLA on triage response time.

5. **Over-automation cascades** — Auto-close on inactivity closes valid slow-moving issues and creates false "done" signals. Parent/sub-issue auto-close chain can cascade unintentionally. Fix: whitelist certain labels or projects from auto-close.

6. **Backlog rot** — Teams ignore Linear's own advice ("important issues will resurface"), producing infinite stale backlogs and undetected duplicates. Fix: aggressive archiving, periodic backlog bankruptcy.

- Source: https://linear.app/method/introduction, https://news.ycombinator.com/item?id=38294623

### Cross-cutting: Gap Analysis of Our Current Skills

**What we do well:**
- Label enforcement (at least one required per issue)
- Action-verb title convention enforced
- Max 3 AC scope limit with split suggestions
- Plan-driven implementation with Linear document lifecycle (local-first → committed → uploaded → checkpointed)
- Status transitions (In Progress → In Review)
- Cloudflare WAF workaround documented and enforced (no backtick code formatting)
- CI failure auto-filing with title-based dedup
- Delegate field used for autonomous agent routing (Cyrus)
- Review contract on PRs (intent, risk, proof, plan criteria)

**What we're missing (10 gaps):**
1. **No Agent Interaction SDK adoption** — no activity streaming, no formal session states, no ephemeral progress updates visible in Linear UI
2. **No label-group syntax enforcement** — current labels are flat (`ci-failure`) not grouped (`Type/CI-Failure`)
3. **Only 2 status transitions** — no substates like "Ready for Review" or "Ready to Merge"
4. **No triage automation** — no rotating ownership, no triage inbox processing skill
5. **No stale-issue archiving** — no auto-archive, no grooming automation
6. **No cycle-debt guards** — no WIP limits, no cycle scope enforcement
7. **No webhook-based automation** — CI filing uses direct GraphQL, not event-driven
8. **No saved-view recommendations** — no workspace setup guidance in our skills
9. **No estimation guidance** — issue creation doesn't suggest estimation or size-splitting
10. **No backlog hygiene** — no duplicate detection, no periodic cleanup

## Concrete /extend Targets

Prioritized by impact and feasibility for a small team:

### Tier 1 — Quick Wins (rules/skill updates, no infrastructure)

**T1.1: Label Taxonomy Rule** → `.claude/rules/linear-labels.md`
- Encode Group/Label convention: `Type/{Bug,Feature,Improvement,Integration,Operations,CI-Failure}`, `Area/{API,Infrastructure,Dashboard,Clio,Agent}`
- Update `/create-linear-issue` to validate labels match Group/Label format
- Migrate existing flat labels to grouped format in Linear workspace

**T1.2: Status Substates in resolve-linear-issue** → update `.claude/commands/resolve-linear-issue.md`
- Add transitions: In Progress → In Review → Ready to Merge → Done
- Add cancel variants: Won't Fix, Duplicate, Could Not Reproduce (for triage)
- Map substates to PR lifecycle events

**T1.3: Issue Estimation Guidance** → update `.claude/commands/create-linear-issue.md`
- Add estimation step: if scope seems large, suggest splitting before applying a size estimate
- Reference Linear's guidance: "break large estimates into smaller issues"

**T1.4: Backlog Hygiene View Recommendations** → `.claude/rules/linear-workspace.md`
- Document recommended saved views: My Issues, Triage Inbox, Stale (>14 days unassigned high-priority), Agent Work
- Reference when setting up or auditing the Linear workspace

### Tier 2 — Medium Effort (new skills/hooks)

**T2.1: /triage-linear Skill** → `.claude/commands/triage-linear.md`
- Process triage inbox: for each unprocessed issue, suggest accept/decline/duplicate/snooze
- Apply label-group taxonomy, suggest project routing
- Could run on a `/loop` schedule for periodic triage

**T2.2: /groom-backlog Skill** → `.claude/commands/groom-backlog.md`
- Identify stale issues (>30 days untouched), suggest archive or re-prioritize
- Detect potential duplicates using title/description similarity
- Report cycle debt: issues rolling over >2 cycles
- WIP check: flag people with >2 in-progress issues

**T2.3: Linear Webhook Hook** → `.claude/hooks/linear-webhook.md`
- React to Linear webhook events for status sync
- Auto-update issue status when PR is merged (Done), PR is opened (In Review)
- Could replace manual status transitions in resolve-linear-issue

### Tier 3 — Strategic Investment (infrastructure required)

**T3.1: Agent Interaction SDK Integration**
- Register Claude Code as a Linear agent with OAuth `actor=app`
- Stream `thought`/`action` activities during `/resolve-linear-issue` execution
- Use `elicitation` for agent-initiated escalation when blocked
- Report `error` with structured context on failures
- Enable Delegate field filtering for "agent work" views
- **Prerequisite**: Persistent process or webhook receiver to handle the 10-second responsiveness requirement

**T3.2: Event-Driven Linear Automation**
- Webhook receiver for Linear events → trigger Claude Code sessions
- Pattern: issue assigned to agent → webhook → spawn `/resolve-linear-issue` session
- Replaces polling/manual invocation with event-driven dispatch
- Could mirror Cursor's assignee-trigger pattern but with our plan-driven sophistication

## Key Sources
### Codebase
- `.claude/commands/create-linear-issue.md` — current issue creation skill
- `.claude/commands/resolve-linear-issue.md` — current issue resolution skill
- `.github/workflows/ci-failure-linear.yml` — CI failure auto-filing

### Linear Documentation
- [Configuring Workflows](https://linear.app/docs/configuring-workflows)
- [Triage](https://linear.app/docs/triage)
- [Triage Intelligence](https://linear.app/docs/triage-intelligence)
- [Labels](https://linear.app/docs/labels)
- [Issue Templates](https://linear.app/docs/issue-templates)
- [Parent and Sub-Issues](https://linear.app/docs/parent-and-sub-issues)
- [Cycles](https://linear.app/docs/use-cycles)
- [Estimates](https://linear.app/docs/estimates)
- [Initiatives](https://linear.app/docs/initiatives)
- [Conceptual Model](https://linear.app/docs/conceptual-model)
- [Agents in Linear](https://linear.app/docs/agents-in-linear)
- [Linear Asks](https://linear.app/docs/linear-asks)

### Linear Developer Docs
- [Agent Interaction SDK](https://linear.app/developers/agent-interaction)
- [Agent OAuth (Getting Started)](https://linear.app/developers/agents)
- [Webhooks](https://linear.app/developers/webhooks)
- [Rate Limiting](https://developers.linear.app/docs/graphql/working-with-the-graphql-api/rate-limiting)

### Linear Blog / Changelog
- [Agent Interaction SDK Launch](https://linear.app/changelog/2025-07-30-agent-interaction-guidelines-and-sdk)
- [Building the Agent Interaction SDK](https://linear.app/now/our-approach-to-building-the-agent-interaction-sdk)
- [Cursor Integration](https://linear.app/now/how-cursor-integrated-with-linear-for-agents)
- [Cursor Background Agents](https://linear.app/changelog/2025-08-21-cursor-agent)
- [MCP Server](https://linear.app/changelog/2025-05-01-mcp)
- [Asks Fields and Triage Routing](https://linear.app/changelog/2025-06-05-asks-fields-and-triage-routing)
- [Auto-close and Auto-archive](https://linear.app/changelog/2020-08-19-auto-close-and-auto-archive)
- [Linear Method](https://linear.app/method/introduction)
- [Descript's Internal Guide](https://linear.app/now/descript-internal-guide-for-using-linear)

### External
- [Cursor Blog: Linear Integration](https://cursor.com/blog/linear)
- [Cursor Linear Docs](https://cursor.com/docs/integrations/linear)
- [Cursor Background Agents Experience](https://lgallardo.com/2025/06/11/cursor-background-agents-experience/)
- [Devin Integration](https://linear.app/integrations/devin)
- [Devin Performance Review 2025](https://cognition.ai/blog/devin-annual-performance-review-2025)
- [Claude Code Linear Integration Request](https://github.com/anthropics/claude-code/issues/12925)
- [Composio: Linear MCP in Claude Code](https://composio.dev/blog/how-to-set-up-linear-mcp-in-claude-code-to-automate-issue-tracking)
- [Morgen: Linear Guide](https://www.morgen.so/blog-posts/linear-project-management)
- [Terem: Task Naming Guide](https://terem.tech/naming-guide-for-task-bug-and-user-story-titles/)
- [Linear Webhooks Guide (Inventive)](https://inventivehq.com/blog/linear-webhooks-guide)
- [HN: Planning for Unplanned Work](https://news.ycombinator.com/item?id=38294623)
- [Screenful: Lead and Cycle Times](https://screenful.com/blog/tracking-lead-and-cycle-times-of-linear-issues)

## Open Questions
- What Linear plan tier (Free/Standard/Business/Enterprise) does FileScience currently use? Triage Rules and some AI features require Business/Enterprise.
- Should we pursue the Agent Interaction SDK now (Developer Preview) or wait for GA? The 10-second responsiveness requirement implies a persistent process.
- Would event-driven dispatch (T3.2) replace or complement the current `helm` orchestration for issue resolution?
- How would label taxonomy migration work without breaking existing saved views and automations?

## Research Metadata
- Depth mode: standard
- Iterations completed: 3 / 5
- Termination reason: gaps exhausted (1 remaining gap is synthesis, not research — rolled into recommendations)
- Manifest: `.claude/deep-research/2026-03-10-best-practices-for-linear-project.md`
