# Linear + Claude Code Integration Research

**Date:** 2026-02-16
**Status:** RESEARCH COMPLETE
**Scope:** Linear MCP in Claude Code, agentic patterns, general best practices

---

## Executive Summary

Linear is one of the most mature and well-supported MCP integrations for Claude Code. The ecosystem includes:
- **Official Linear MCP server** (hosted by Linear, OAuth 2.1, 22 tools)
- **Anthropic-hosted integration** (already connected in this workspace via `mcp__claude_ai_Linear__*`)
- **Native "Linear for Agents" platform** (delegation model, Agent Interaction SDK, agent sessions)
- **Community skills/commands** (`wrsmith108/linear-claude-skill`, Linear Skill Pack 24 skills, `czottmann/using-linear`)
- **Third-party agent integrations** (Devin, Cursor, GitHub Copilot, Cyrus)

Key insight: Linear treats agents as **delegates** (contributors) not **assignees** — humans maintain accountability while agents do the work.

---

## Part 1: Linear MCP Technical Setup

### What's Already Available

This workspace already has Anthropic's hosted Linear MCP connected. Available tools (40+):

**Issues:** `get_issue`, `list_issues`, `create_issue`, `update_issue`, `list_issue_statuses`, `get_issue_status`, `list_issue_labels`, `create_issue_label`
**Projects:** `list_projects`, `get_project`, `create_project`, `update_project`
**Milestones:** `list_milestones`, `get_milestone`, `create_milestone`, `update_milestone`
**Initiatives:** `list_initiatives`, `get_initiative`, `create_initiative`, `update_initiative`
**Documents:** `get_document`, `list_documents`, `create_document`, `update_document`
**Comments:** `list_comments`, `create_comment`
**Teams:** `list_teams`, `get_team`
**Users:** `list_users`, `get_user`
**Cycles:** `list_cycles`
**Status Updates:** `get_status_updates`, `save_status_update`, `delete_status_update`
**Attachments:** `get_attachment`, `create_attachment`, `delete_attachment`
**Search:** `search_documentation`
**Other:** `extract_images`

### Official Linear MCP Server

- **Endpoint:** `https://mcp.linear.app/mcp` (HTTP) or `https://mcp.linear.app/sse` (SSE)
- **Auth:** OAuth 2.1 with dynamic client registration (no API key needed)
- **Also supports:** `Authorization: Bearer <token>` for API keys / restricted read-only keys
- **Rate limits:** 1,500 req/hr (API key), 500 req/hr per user/app (OAuth), 250K complexity points/hr
- **GraphQL complexity:** 10,000 points per query, auto-split if exceeded

### Setup (if not using Anthropic-hosted)

```bash
# Claude Code (one-liner)
claude mcp add --transport http linear-server https://mcp.linear.app/mcp

# Then in session:
/mcp  # completes OAuth flow
```

### Community MCP Implementations

| Server | Status | Best For |
|--------|--------|----------|
| **Official (linear.app)** | Active | Default choice, zero maintenance |
| **tacticlaunch/mcp-linear** | Active | Self-hosted, good docs, natural language |
| **dvcrn/mcp-server-linear** | Active | Multiple workspace support (TOOL_PREFIX) |
| **jerhadf/linear-mcp-server** | Deprecated | Don't use |

---

## Part 2: Workflow Patterns (Claude Code + Linear)

### Pattern A: "Ship Daily" (Kenny Liao / AI Launchpad)

**Source:** [theailaunchpad.substack.com](https://theailaunchpad.substack.com/p/my-ship-daily-system-with-claude)

Three-phase system saving ~45min-2hr/day:

1. **Planning (5-10min):** Collaborate with Claude to create well-scoped Linear issues via `/create-linear-issue` slash command
2. **Execution (10-30min):** Claude autonomously resolves issues via `/resolve-linear-issue-tdd` (TDD-driven)
3. **Review (minimal):** Human reviews and merges PRs

Key principles:
- Issues narrow enough to fit in a single context window
- Max 3 acceptance criteria per issue (or split)
- Each issue has in-scope/out-of-scope boundaries
- Self-contained: Claude needs zero clarifying questions

### Pattern B: Spec-Driven Development (OpenSpec + Linear)

**Source:** [intent-driven.dev](https://intent-driven.dev/blog/2026/01/11/linear-mcp-openspec-sdd-workflow/)

Automatic status sync as agent advances through stages:
- **Propose** stage → Linear issue: Backlog
- **Apply** stage → Linear issue: In Progress
- **Archive** stage → Linear issue: Done

Separates "the what" (business cases in Linear) from "the how" (technical specs in git).

### Pattern C: Full Lifecycle Automation (Cyrus)

**Source:** [atcyrus.com](https://www.atcyrus.com/stories/linear-claude-code-integration-guide) | [github.com/ceedaragents/cyrus](https://github.com/ceedaragents/cyrus)

Open-source agent powered by Claude Code:
- Monitors assigned Linear issues
- Creates isolated git worktrees per task
- Runs Claude Code sessions autonomously
- Streams activity updates back to Linear
- Opens/updates PRs via GitHub app
- Self-hostable (Mac/Linux) or managed hosting

### Pattern D: Bug-to-Fix Pipeline

Agent encounters exception → creates Linear P1 issue with stack trace, labels, assignee → agent (or another agent) picks up issue → implements fix → creates PR → updates Linear status.

### Pattern E: CLAUDE.md Route Integration

Add Linear awareness to your routing:
```markdown
### Route G — Execute Linear issue
**Use when:** user provides a Linear issue ID/URL, or says "work on [issue]"
**Flow:** Fetch issue details → implement → create PR → update Linear status
**Distill:** `current_work.md` (issue link + what changed)
```

---

## Part 3: Slash Commands & Skills

### Community Slash Commands

**`/create-linear-issue`** — Guided issue creation:
```markdown
# Template structure
- Summary (1-2 sentences)
- Current behavior vs. expected behavior
- Acceptance criteria (max 3)
- In scope / out of scope
- Additional context / root cause
```

**`/create-linear-project`** — Breaks initiatives into milestones with dependent issues

**`/resolve-linear-issue-tdd`** — Full resolution:
1. Fetch issue details via MCP
2. Plan implementation (subagent)
3. Write tests first (TDD)
4. Implement solution
5. Run tests + lints
6. Create PR
7. Update Linear issue status

### Ready-Made Command Repos

**svnlto/claude-code-linear-commands** — [GitHub](https://github.com/svnlto/claude-code-linear-commands)
Commands: `quick-issue.md`, `generate-fr.md`, `generate-us.md`, `generate-prd.md`, `convert-to-linear.md`, `analyze-issue.md`

**ronanathebanana/claude-linear-gh-starter** — [GitHub](https://github.com/ronanathebanana/claude-linear-gh-starter)
Setup wizard (`/setup-linear`) + 8 commands: `/bug-linear`, `/feature-linear`, `/start-issue`, `/my-work-linear`, etc.

**ChrisWiles/claude-code-showcase** — [GitHub](https://github.com/ChrisWiles/claude-code-showcase)
Example `ticket.md` command: full workflow from Linear fetch → implementation → PR creation.

### TDD + Linear Pattern

Claude defaults to implementation-first. Solution: multi-agent system with hooks enforces RED-GREEN-REFACTOR:
- Parse Linear acceptance criteria → one AC = one RED-GREEN-REFACTOR cycle
- TDD Guard hook enforces test-first via PostToolUse
- Update Linear with progress comments after each cycle
- Sources: [alexop.dev TDD](https://alexop.dev/posts/custom-tdd-workflow-claude-code-vue/), [aihero.dev TDD Skill](https://www.aihero.dev/skill-test-driven-development-claude-code)

### PR + Linear Auto-Linking

Branch naming: `{type}/{ISSUE-ID}-{description}` (e.g., `feat/WEB-123-oauth-flow`)
- Issue ID in branch name enables automatic PR→Linear linking
- PostToolUse hooks extract issue ID from branch for status updates
- GitHub Actions: PR opened → "In Review", merged → "Done"

### Subagent Delegation (7-Agent Pattern)

For complex Linear issues, delegate to specialized subagents:
1. **plan-agent** (strategy) → 2. **reader-agent** (codebase analysis) → 3. **maker-agent** (implementation) → 4. **security-agent** (review) → 5. **test-agent** (validation) → 6. **docs-agent** (documentation) → 7. **debug-agent** (verification)

Parallelize up to 7 independent Linear issues simultaneously, each in own context window via Task tool.

### wrsmith108/linear-claude-skill

**Source:** [github.com/wrsmith108/linear-claude-skill](https://github.com/wrsmith108/linear-claude-skill)

Three integration patterns:
1. **MCP Tools** — Simple CRUD via Linear MCP
2. **SDK Automation** — Complex operations via TypeScript scripts
3. **GraphQL API** — Direct queries for advanced operations

Structure: `SKILL.md` (entry point) + `api.md` + `sdk.md` + `sync.md` + `docs/` + `scripts/`

Install: `git clone https://github.com/wrsmith108/linear-claude-skill ~/.claude/skills/linear`

### czottmann/using-linear

**Source:** [claude-plugins.dev](https://claude-plugins.dev/skills/@czottmann/claude-code-stuff/using-linear)

Lightweight skill for inline Linear usage.

### Linear Skill Pack (24 Skills)

**Source:** [claudecodeplugins.io](https://claudecodeplugins.io/learn/linear/)

Comprehensive pack: install-auth, hello-world, local-dev-loop, sdk-patterns, core-workflow-a/b, common-errors, debug-bundle, rate-limits, security-basics, prod-checklist, upgrade-migration, ci-integration, deploy-integration, webhooks-events, performance-tuning, cost-tuning.

Install: `/plugin install linear-pack@claude-code-plugins-plus`

---

## Part 4: Linear for Agents (Native Platform)

### Delegation Model

**Source:** [linear.app/docs/agents-in-linear](https://linear.app/docs/agents-in-linear)

Linear's core principle: **agents are delegated to, not assigned.**
- Issues can only be **assigned** to humans (accountable)
- Issues can only be **delegated** to agents (responsible for action)
- Human remains primary assignee; agent is contributor

### Agent Interaction SDK

**Source:** [linear.app/developers/agents](https://linear.app/developers/agents) | [linear.app/developers/aig](https://linear.app/developers/aig)

When an agent is invoked (via assignment, @-mention, or label), Linear sends a webhook with:
- What happened (trigger type)
- Where (which issue)
- Who (which user)
- Full issue context

Agents communicate progress via **Agent Activities**:
- Thoughts/reasoning steps
- Tool calls
- Clarification prompts
- Final responses
- Error states

**Agent Interaction Guidelines (AIG):**
1. Clear Identity — never mistaken for a human
2. Standard UI Patterns — work through existing platform actions
3. Immediate Feedback — unobtrusive confirmation on invocation
4. Status Transparency — active/waiting/error/complete
5. Human Oversight — humans maintain approval rights

### Integrated Agent Platforms

| Agent | How It Works | Trigger |
|-------|-------------|---------|
| **Devin** | Scopes issues, suggests implementation plans, creates PRs | Assignment, @-mention, "devin" label |
| **Cursor** | Cloud agent works on issues, streams updates back | Assignment from assignee menu |
| **GitHub Copilot** | Analyzes issues, opens draft PRs, runs tests | Linear integration config |
| **Cyrus** | Open-source, Claude Code powered, isolated worktrees | Assignment |

### Triage Intelligence (Built-in AI)

- Auto-assesses incoming issues against existing data
- Suggests: teams, projects, assignees, labels
- Can auto-apply when configured
- Processing: 1-4 min/issue (quality over speed)
- Available on Business/Enterprise plans

---

## Part 5: Linear Best Practices (General)

### The Linear Method (Official Philosophy)

**Source:** [linear.app/method](https://linear.app/method)

- **Opinionated software** — guides toward defaults, don't over-customize
- **Break work into small parts** — complete several concrete tasks/week
- **Small scope = easier review** — avoid massive PRs
- **Everyone writes their own issues** — forces deep thinking
- **Quote user feedback directly** — don't summarize

### Issue Writing

- **Titles: action verbs** — "Fix calendar loading bug", "Add Stripe webhook"
- **Keep titles short and scannable** — read in context of boards/lists
- **Descriptions optional** — only what's needed to perform the task
- **Define clear outcomes** — clear deliverable (code, design, document, action)
- **Max 3 acceptance criteria** — or split the issue

### Issue Template Structure (AI-Optimized)

```markdown
## Summary
[1-2 sentences: what and why]

## Current → Expected Behavior
Current: [what happens now]
Expected: [what should happen]

## Acceptance Criteria
- [ ] [Specific, measurable condition 1]
- [ ] [Specific, measurable condition 2]
- [ ] [Specific, measurable condition 3]

## Scope
In: [what this issue covers]
Out: [explicit exclusions]

## Context
[Root cause analysis for bugs, or design rationale for features]
[Link to related issues/docs]
```

### Hierarchy

| Level | Use For | Scope |
|-------|---------|-------|
| **Initiatives** | OKRs, company goals | Quarters/halves |
| **Projects** | Time-bound deliverables | Weeks to months |
| **Cycles** | Sprint-like iterations | 1-2 weeks (2 weeks most common) |
| **Issues** | Individual work items | Hours to days |
| **Sub-issues** | Task breakdown | When issue is too big but not a project |

### Workflow States

Default: Backlog → Todo → In Progress → Done → Canceled

Recommended: **5-7 statuses max**. Start with defaults, customize only for real pain points.

Linear's own team uses: Backlog, Todo, In Progress, In Review, Ready to Merge, Done, Canceled, Won't Fix, Duplicate.

### Labels & Metadata

- Keep label sets small and meaningful
- Don't replicate statuses as labels
- Use priority for true urgency (0=None, 1=Urgent, 2=High, 3=Normal, 4=Low)
- Revisit quarterly — remove unused, consolidate duplicates

### Triage Process

- Triage = special inbox for issues from integrations / non-team members
- Review daily or weekly
- Actions: accept, escalate, merge, decline, snooze
- Archive untouched issues after 30+ days

### GitHub/GitLab Integration

- Auto-update issue status as PRs are drafted → opened → reviewed → merged
- Link PRs and commits automatically
- "Ready to merge" status when PRs pass all checks
- Branch naming: `feature/TEAM-123-description` pattern

---

## Part 6: FileScience-Specific Recommendations

Given the current workflow (Polylith monorepo, CLAUDE.md routing, memory-bank, plan/research/implement cycle):

### Recommended Integration Points

1. **Add Route G to CLAUDE.md** — "Execute Linear issue" route that fetches issue, implements, creates PR, updates status
2. **Create `/create-linear-issue` command** — issue creation template matching team conventions
3. **Create `/resolve-linear-issue` command** — full lifecycle: read issue → plan → implement → test → PR → update Linear
4. **Hook Linear into plan workflow** — when creating plans, optionally create corresponding Linear project + issues
5. **Status sync on `/implement_plan`** — update Linear issue status as plan phases complete

### Issue Structure for FileScience

```markdown
## Summary
[1-2 sentences]

## Acceptance Criteria
- [ ] Tests passing (ruff + ty + import-linter clean)
- [ ] [Specific behavioral criteria]

## Scope
In: [specific files/components]
Out: [what this doesn't touch]

## Context
Plan: [link to memory-bank plan if exists]
Research: [link to research if exists]
Component: [Polylith component/base affected]
```

### What's Already Working

The Anthropic-hosted Linear MCP is already connected with 40+ tools. No setup needed — you can start using `create_issue`, `list_issues`, `update_issue`, etc. immediately via the `mcp__claude_ai_Linear__*` tools.

---

## Key Sources

### Official Linear Docs
- [Linear MCP Server](https://linear.app/docs/mcp)
- [Linear for Agents](https://linear.app/agents)
- [Agent Interaction Guidelines](https://linear.app/developers/aig)
- [Agent Interaction SDK](https://linear.app/developers/agent-interaction)
- [Linear Method](https://linear.app/method)
- [Linear GraphQL API](https://linear.app/developers/graphql)

### Workflow Examples
- [Ship Daily System (AI Launchpad)](https://theailaunchpad.substack.com/p/my-ship-daily-system-with-claude)
- [Linear MCP + OpenSpec SDD](https://intent-driven.dev/blog/2026/01/11/linear-mcp-openspec-sdd-workflow/)
- [Cyrus: Claude Code + Linear Agent](https://github.com/ceedaragents/cyrus)
- [Damian Galarza's Claude Code Workflow](https://www.damiangalarza.com/posts/2025-11-25-how-i-use-claude-code/)

### Community Tools
- [wrsmith108/linear-claude-skill](https://github.com/wrsmith108/linear-claude-skill)
- [Linear Skill Pack (24 Skills)](https://claudecodeplugins.io/learn/linear/)
- [czottmann/using-linear](https://claude-plugins.dev/skills/@czottmann/claude-code-stuff/using-linear)
- [Valian/linear-cli-skill](https://github.com/Valian/linear-cli-skill)
- [tacticlaunch/mcp-linear](https://github.com/tacticlaunch/mcp-linear)

### Technical References
- [Linear Rate Limiting](https://linear.app/developers/rate-limiting)
- [How Linear Built Triage Intelligence](https://linear.app/now/how-we-built-triage-intelligence)
- [Why Linear Built an API for Agents](https://thenewstack.io/why-linear-built-an-api-for-agents/)
- [How Cursor Integrated with Linear](https://linear.app/now/how-cursor-integrated-with-linear-for-agents)
- [Builder.io Linear MCP Guide](https://www.builder.io/blog/linear-mcp-server)

### Best Practices Guides
- [Morgen: How to Use Linear](https://www.morgen.so/blog-posts/linear-project-management)
- [Everhour: Linear Task Management](https://everhour.com/blog/linear-task-management/)
- [Linear Dashboard Best Practices](https://linear.app/now/dashboards-best-practices)
- [Linear Continuous Planning](https://linear.app/now/continuous-planning-in-linear)

---

## Gaps / Open Questions

1. **No published "AI-optimized" issue template spec** — best practices are implicit, not codified
2. **Multi-agent coordination** — unclear how multiple agents coordinate on same project
3. **Cost management** — limited guidance on managing API costs at scale
4. **Metrics** — no public benchmarks on agent resolution rates or quality metrics
5. **Webhook → Claude Code** — no documented pattern for Linear webhooks triggering Claude Code sessions automatically
