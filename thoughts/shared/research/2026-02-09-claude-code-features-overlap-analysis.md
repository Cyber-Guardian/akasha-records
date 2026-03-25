# Claude Code Features & Overlap Analysis — What NOT to Build

**Date:** 2026-02-09
**Purpose:** Map Claude Code's current/recent features against our planned workflow improvements to avoid wasting effort on things Anthropic already ships natively.
**Context:** Comparison against our [[2026-02-09-optimal-workflow-frameworks-analysis|14 prioritized improvements]] and [[2026-02-06-ai-agent-backpressure-tools-brief|backpressure tools research]].

---

## Key Claude Code Features (as of Feb 2026)

### 1. Agent Teams (Experimental — Feb 5, 2026)

**What it is:** Native multi-agent orchestration. One session acts as team lead; teammates work independently in their own context windows with inter-agent messaging.

**Enable:** `CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1` in settings.json or environment.

**Key capabilities:**
- Lead/teammate model with shared task list and self-coordination
- Teammates message each other directly (not just report to lead)
- Split pane display (tmux/iTerm2) or in-process mode
- `TeammateIdle` hook — quality gate before teammate stops working
- `TaskCompleted` hook — quality gate before task is marked complete
- Delegate mode — restricts lead to coordination only (Shift+Tab)
- Plan approval for teammates — require lead approval before implementation
- File locking for task claiming (prevents race conditions)

**Best use cases:** Research/review, new modules, debugging with competing hypotheses, cross-layer coordination.

**Limitations:** Experimental, no session resumption with in-process teammates, one team per session, no nested teams, high token usage.

**Source:** [Agent Teams docs](https://code.claude.com/docs/en/agent-teams)

### 2. Comprehensive Hooks System

**All hook events (14 total):**

| Event | When | Can Block? | New? |
|-------|------|-----------|------|
| `SessionStart` | Session begins/resumes | No | No |
| `UserPromptSubmit` | User submits prompt | Yes | No |
| `PreToolUse` | Before tool execution | Yes | No |
| `PermissionRequest` | Permission dialog shown | Yes | Newer |
| `PostToolUse` | After successful tool | No | No |
| `PostToolUseFailure` | After failed tool | No | Newer |
| `Notification` | Notification sent | No | Newer |
| `SubagentStart` | Subagent spawned | No | Newer |
| `SubagentStop` | Subagent finished | Yes | Newer |
| `Stop` | Claude finishes responding | Yes | No |
| `TeammateIdle` | Agent team teammate idle | Yes | **NEW** |
| `TaskCompleted` | Task marked complete | Yes | **NEW** |
| `PreCompact` | Before compaction | No | **NEW** |
| `SessionEnd` | Session terminates | No | **NEW** |

**Three hook types:**
1. **Command hooks** (`type: "command"`) — shell scripts, our current approach
2. **Prompt hooks** (`type: "prompt"`) — **NEW** — single-turn LLM evaluation. Send prompt + context to Claude (Haiku by default), get `{ok: true/false, reason}` back. No code needed!
3. **Agent hooks** (`type: "agent"`) — **NEW** — multi-turn subagent with tool access (Read, Grep, Glob). Can investigate files before making a decision. Up to 50 turns.

**Async hooks:** `"async": true` on command hooks — runs in background, reports back on next turn.

**Hooks in skills/agents:** Hooks can be defined in skill/agent YAML frontmatter, scoped to component lifetime.

**Source:** [Hooks reference](https://code.claude.com/docs/en/hooks)

### 3. Opus 4.6 Model Capabilities

- **1M token context window** (beta) — up from 200K
- **128K output tokens** — doubled from 64K
- **Adaptive thinking** — `thinking: {type: "adaptive"}` replaces manual budget_tokens
- **Effort parameter GA** — `max` level for highest capability
- **Fast mode** — 2.5x faster output at premium pricing ($30/$150 per MTok)
- **Compaction API** (beta) — server-side context summarization for infinite conversations

**Source:** [What's new in Claude 4.6](https://platform.claude.com/docs/en/about-claude/models/whats-new-claude-4-6)

### 4. Built-in Task Tracking

- `TaskCreate`, `TaskUpdate`, `TaskGet`, `TaskList` tools
- Tasks with status (pending/in_progress/completed), dependencies (blocks/blockedBy)
- Shared across agent teams
- `TaskCompleted` hook for quality gates

### 5. Built-in Plan Mode

- `EnterPlanMode` / `ExitPlanMode` tools
- Read-only exploration + plan approval workflow
- Can be required for agent team teammates before they implement

### 6. Skills & Plugins System

- Skills with YAML frontmatter defining hooks, description, etc.
- Plugin marketplace (`claude plugin marketplace add`)
- Skills invocable via `/skill-name`

---

## Impact on Our 14 Planned Improvements

### Improvements That Are NOW REDUNDANT (don't build)

| # | Improvement | Why Redundant | Native Feature |
|---|------------|---------------|----------------|
| 8 | Wave-based parallel execution | Agent Teams does this natively with shared task lists, self-coordination, and dependency management | Agent Teams |
| 12 | Codebase mapping before major work | Can spawn an agent team with 4 teammates analyzing different dimensions simultaneously | Agent Teams |

### Improvements That Should USE NATIVE FEATURES (build differently)

| # | Improvement | Original Plan | Better Approach |
|---|------------|--------------|-----------------|
| 4 | Proactive compact at 60-70% | `strategic_compact_suggester.sh` command hook | **Keep as-is** — `PreCompact` hook can supplement (save context before compaction). Our proactive suggestion is still useful since Compaction API is server-side auto. |
| 5 | Durable memory size limits | PostToolUse command hook | **Use prompt hook** instead of shell script. `type: "prompt"` can evaluate whether a durable file exceeds targets without custom code. |
| 6 | Two-stage validation | Custom validator subagent between plan phases | **Use agent hooks** on `TaskCompleted` and/or `Stop`. An `agent` hook can Read files, run checks, and block completion if criteria aren't met. No custom skill modification needed. |

### Improvements That Are STILL VALUABLE (build as planned)

| # | Improvement | Why Still Needed | Notes |
|---|------------|-----------------|-------|
| 1 | Verification criteria in plans | No native support for plan phase verification criteria | Update `/create_plan` template |
| 2 | Rewrite routes as triggers | CLAUDE.md optimization, not a tool feature | CSO "Use when..." patterns |
| 3 | Mandatory skill enforcement | Behavioral guidance, not a tool feature | CLAUDE.md language |
| 7 | Structured implementer prompts | Agent team spawn prompts benefit from structure | Applies to both subagents and agent team teammates |
| 9 | /debug skill | Systematic debugging methodology | No native equivalent |
| 10 | Plan header with execution instructions | Plan template improvement | Helps handoff between sessions |
| 11 | TDD for process docs | Methodology, not tooling | Superpowers pattern |
| 13 | ADI cycle for decisions | Decision-making methodology | Route D enhancement |
| 14 | Quick mode | Skip full planning for small tasks | No native equivalent |

---

## New Opportunities (things we SHOULD build using new features)

### 1. Agent-based Stop Hook for Session Closeout

**Currently:** CLAUDE.md says "Before ending, ensure X, Y, Z" — easily forgotten.
**New approach:** `Stop` hook with `type: "agent"` that reads durable memory files and verifies:
- `current_work.md` has at least one `memory-bank/thoughts/` artifact link
- `next_up.md` reflects concrete next actions
- `blockers.md` has any new blockers captured

```json
{
  "hooks": {
    "Stop": [
      {
        "hooks": [
          {
            "type": "agent",
            "prompt": "Verify session closeout requirements. Read memory-bank/durable/01-active/current_work.md and check it has at least one link to memory-bank/thoughts/. Read next_up.md and verify it has actionable items. If either check fails, respond {\"ok\": false, \"reason\": \"Missing: [specifics]\"}. Otherwise respond {\"ok\": true}. Context: $ARGUMENTS",
            "timeout": 60
          }
        ]
      }
    ]
  }
}
```

### 2. TaskCompleted Quality Gate

**Use `TaskCompleted` hook** to verify work before marking tasks done:

```json
{
  "hooks": {
    "TaskCompleted": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "make test && make lint-imports"
          }
        ]
      }
    ]
  }
}
```

### 3. PreCompact Context Saver

**Use `PreCompact` hook** to save critical context before compaction destroys it:

```bash
#!/bin/bash
# Save current task state before compaction
INPUT=$(cat)
SESSION_ID=$(echo "$INPUT" | jq -r '.session_id')
echo "Compaction triggered for session $SESSION_ID" >> ~/.claude/compaction-log.txt
# Could also auto-create a minimal handoff doc
```

### 4. SessionEnd Auto-Handoff

**Use `SessionEnd` hook** to auto-create minimal handoff when session ends:

```json
{
  "hooks": {
    "SessionEnd": [
      {
        "hooks": [
          {
            "type": "command",
            "command": ".claude/hooks/auto-handoff.sh"
          }
        ]
      }
    ]
  }
}
```

### 5. Agent Teams for Research Tasks

Instead of Route A spawning a single research subagent, complex research could use agent teams:
- Teammate 1: Code structure analysis
- Teammate 2: Test coverage analysis
- Teammate 3: Dependency/architecture analysis

### 6. Prompt Hook for Durable Memory Size

Replace shell script complexity with a prompt hook:

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Write|Edit",
        "hooks": [
          {
            "type": "prompt",
            "prompt": "A file was just written/edited. Check if the file path contains 'memory-bank/durable'. If so, check if the content appears to exceed these targets: project_brief.md: 80 lines, current_work.md: 60 lines, next_up.md: 40 lines, blockers.md: 30 lines. Context: $ARGUMENTS. Respond {\"ok\": false, \"reason\": \"File exceeds size target\"} if over limit, {\"ok\": true} otherwise.",
            "model": "haiku"
          }
        ]
      }
    ]
  }
}
```

---

## taches-cc-resources Analysis

**Repo:** [glittercowboy/taches-cc-resources](https://github.com/glittercowboy/taches-cc-resources)

**What it is:** A collection of Claude Code skills and commands designed for powerful development workflows. Installable via plugin marketplace or manual copy.

**Key skills offered:**
- `create-plans` — Plan creation with structured templates
- `create-mcp-server-python` / `create-mcp-server-typescript` — MCP server scaffolding
- `create-hooks` — Expert guidance for hook creation/configuration
- Various other development workflow skills

**Philosophy:** "Everything is possible" — assumes Claude Code can do anything with the right prompting.

**Relevance to us:**
- Their `create-hooks` skill could be useful reference for our hook improvements
- Their plan creation patterns may overlap with our `/create_plan`
- MCP server skills are not relevant to our current needs
- The "adaptive intake" pattern (analyzing what the user needs before acting) aligns with our route selection in CLAUDE.md

**Full inventory:** 27 commands, 9 skills, 3 auditor subagents.

**Notable patterns worth cherry-picking:**

1. **Mental model commands** (`/consider:pareto`, `/consider:first-principles`, `/consider:inversion`, `/consider:second-order`) — on-demand thinking frameworks for architecture decisions. Low effort to add (~1 hour each).

2. **Domain expertise skills** — 5k-10k line exhaustive knowledge bases at `~/.claude/skills/expertise/` (e.g., `polylith-python.md`, `aws-serverless.md`). Auto-loaded by create-plans to make specs framework-specific. We could create `polylith-python.md` and `aws-lambda-python.md`.

3. **Skill self-healing** — `/heal-skill` diagnoses failures and updates the skill. Meta-quality improvement.

4. **Auditor subagents** — `skill-auditor`, `slash-command-auditor`, `subagent-auditor`. Meta-skills that review other tools for best practices. Could apply to our custom skills.

5. **Brief → Roadmap → PLAN hierarchy** — More granular than our single-plan approach. Brief (1-2 sentences) → Roadmap (phases) → Research → PLAN.md (executable) → SUMMARY.md.

6. **Fresh context loops (Ralph)** — Stateless autonomous coding where progress lives in files/git, not LLM context. Same prompt repeatedly; state persists externally.

7. **"PLAN.md IS the prompt"** — Plans written to be directly executable by agents, not just human-readable.

**Their strengths vs ours:**
- Mental model commands, skill self-healing, auditor subagents, domain expertise, fresh context loops
- More granular planning hierarchy (Brief → Roadmap → PLAN)

**Our strengths vs theirs:**
- Architectural fitness enforcement (import-linter contracts)
- Type safety gates (ty in PostToolUse)
- Mutation testing (mutmut + noise filtering)
- Structured memory (explicit durable vs archival with size targets)
- Explicit route system (5 clear routes A-E)

**Assessment:** Our setup is more mature for code quality enforcement. Their setup has better meta-workflow tooling (self-healing, auditing, mental models). Worth borrowing: mental model commands, domain expertise skills, auditor pattern.

---

## Revised Priority List

Given native feature overlap, here's the updated action list:

### Do NOW (Tier 1 — still valuable, quick wins)

1. **Add verification criteria to plan template** — Update `/create_plan` skill
2. **Rewrite CLAUDE.md routes as triggers** — "Use when..." patterns
3. **Add mandatory skill enforcement language** — CLAUDE.md update
4. **Add agent-based Stop hook for session closeout** — NEW, uses native agent hooks
5. **Add TaskCompleted quality gate hook** — NEW, uses native hook

### Do NEXT (Tier 2 — leverage new native features)

6. **Replace strategic_compact_suggester.sh** — Consider prompt hook or keep + supplement with PreCompact
7. **Add prompt hook for durable memory size limits** — Use prompt hook instead of shell script
8. **Add PreCompact context saver** — Save context before compaction
9. **Add SessionEnd auto-handoff hook** — Auto-create minimal handoff
10. **Structured implementer prompts** — Apply to agent team spawn prompts too

### SKIP (redundant with native features)

- ~~Wave-based parallel execution~~ → Agent Teams
- ~~Codebase mapping~~ → Agent Teams with specialized teammates

### DEFER (methodology, not tooling)

- TDD for process docs
- ADI cycle for decisions
- Quick mode for lightweight tasks
- /debug skill

---

## Sources

### Official Documentation
- [Agent Teams](https://code.claude.com/docs/en/agent-teams)
- [Hooks Reference](https://code.claude.com/docs/en/hooks)
- [What's New in Claude 4.6](https://platform.claude.com/docs/en/about-claude/models/whats-new-claude-4-6)
- [Anthropic: Introducing Claude Opus 4.6](https://www.anthropic.com/news/claude-opus-4-6)

### Community/Press
- [TechCrunch: Anthropic releases Opus 4.6 with agent teams](https://techcrunch.com/2026/02/05/anthropic-releases-opus-4-6-with-new-agent-teams/)
- [Addy Osmani: Claude Code Swarms](https://addyosmani.com/blog/claude-code-agent-teams/)
- [taches-cc-resources](https://github.com/glittercowboy/taches-cc-resources)

### Internal References
- [[2026-02-09-optimal-workflow-frameworks-analysis|Optimal Workflow Frameworks Analysis]]
- [[2026-02-06-ai-agent-backpressure-tools-brief|AI Agent Backpressure Tools Brief]]
