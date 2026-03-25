---
date: 2026-03-03T13:14:44-0500
author: claude-opus
git_commit: d5b0536
branch: main
repository: filescience
topic: "Centralized Memory + MCP Server — Shaping & Planning Handoff"
tags: [handoff, shaping, planning, memory-system, mcp-server, session-isolation]
status: in_progress
last_updated: 2026-03-03
last_updated_by: claude-opus
---

# Handoff: Centralized Memory + MCP Server — Split into two plans

## Task(s)

**Status: IN PROGRESS — plan written, user wants it split into two + session isolation hook added**

This session continued the memory system architecture rethink from the [[2026-03-02_16-31-03_memory-system-architecture-rethink|prior handoff]]. Progression:

1. **Shaping** — User proposed centralizing memory as a dedicated MCP server in the monorepo (not a separate repo). Combined with prior V2 architecture work. Key evolution: user referenced the FS-1 harness plan's per-session journal concept and said active/warm memory should be per-session, not centralized. This eliminated the shared `active/` tier from the previous proposal. → Brief written.

2. **Planning** — Full 4-phase plan written covering durable restructure + MCP server + S3 sync + deploy. User reviewed and approved the phasing.

3. **Session isolation research** — User asked how to enforce per-session memory isolation (prevent cross-session leak). Research completed: `session_id` is in all hook inputs, `CLAUDE_ENV_FILE` pattern propagates it as an env var, PreToolUse hooks can enforce session-scoped writes.

4. **User requested split + handoff** — User wants the plan split into two separate plans: (A) memory restructure and (B) MCP server. Also wants a `SessionStart` hook for `$CLAUDE_SESSION_ID` propagation added. Context too heavy, creating handoff.

## Critical References

1. **Combined plan (needs splitting):** `memory-bank/thoughts/shared/plans/2026-03-03-centralized-memory-mcp.md` — 4 phases currently in one plan. User wants Phase 1 (durable restructure) + Phase 3 partial (frontmatter) as "memory rearchitecture" plan, and Phase 2 (MCP package) + Phase 3 partial (S3 sync) + Phase 4 (deploy) as "MCP server" plan.

2. **Brief:** `memory-bank/thoughts/shared/briefs/2026-03-03-centralized-memory-mcp.md` — shaped approach: two centralized tiers (durable + archive) + per-session journals (no centralized warm tier).

3. **Prior handoff (SOTA research):** `memory-bank/thoughts/shared/handoffs/general/2026-03-02_16-31-03_memory-system-architecture-rethink.md` — extensive SOTA research synthesis (15K+ words), Approach D alignment, 5 structural design questions. The structural questions from Q1-Q5 are NOW RESOLVED by the decisions in the combined plan.

4. **FS-1 harness plan:** `memory-bank/thoughts/shared/plans/2026-03-03-fs1-agent-harness.md` — per-session journal pattern that replaced the centralized warm tier.

## Key Decisions Made (carry forward)

These decisions are in the combined plan but must be preserved when splitting:

- **S3-backed reads (not bundled artifact)** — Lambda reads memory from S3. Push-to-main CI syncs `memory-bank/` to S3. Fresh within CI latency.
- **`awslabs.mcp_lambda_handler`** — Lambda-native MCP, no cold start overhead from web framework.
- **Lambda Function URL (not API Gateway)** — simpler, no extra infra for V1.
- **In-memory keyword search** — 181 files is trivially small. No vector DB or FTS5 needed.
- **Merge `project_brief.md` + `product_context.md` → `identity.md`** — single stable identity file.
- **Drop `decisions.md` from durable** — move template to `.claude/reference/`, index not pulling weight.
- **No centralized warm tier** — per-session journals (FS-1) replace shared active notes.

## Three-Layer Model (the architecture)

| Layer | Scope | What lives here | Access |
|-------|-------|----------------|--------|
| **Durable** (hot) | Centralized, shared | Project identity, stable state, focus queue, blockers | MCP-served + file reads |
| **Session journal** (warm) | Per-session, ephemeral | Working notes, decisions, findings, progress | Local files in session tmp dir (FS-1 journal) |
| **Archive** (cold) | Centralized, shared | All historical knowledge — briefs, plans, research, decisions | MCP-served (search, get_briefing) |

## Session Isolation Findings (NEW — not yet in plan)

Research on enforcing per-session memory isolation:

- **`session_id`** is a UUID available in every hook's JSON stdin input (SessionStart, PreToolUse, PostToolUse, Stop, etc.)
- **No native `CLAUDE_SESSION_ID` env var** — but `CLAUDE_ENV_FILE` pattern works: SessionStart hook extracts `session_id` from JSON, writes `export CLAUDE_SESSION_ID=<id>` to `$CLAUDE_ENV_FILE`, then it's available to all Bash commands for the session.
- **Agent SDK** exposes `session_id` on init message and `ResultMessage`.
- **Caveat:** `session_id` is NOT stable across `--resume` (new ID generated). `transcript_path` filename is more stable.
- **Enforcement layers:** (1) Natural isolation via `tmp/sessions/{session_id}/`, (2) SessionStart hook propagates ID, (3) PreToolUse hook validates session ID in write paths.

**Action needed:** Add a `SessionStart` hook to the memory rearchitecture plan that:
```bash
#!/bin/bash
SESSION_ID=$(cat | jq -r '.session_id')
if [ -n "$CLAUDE_ENV_FILE" ] && [ -n "$SESSION_ID" ]; then
  echo "export CLAUDE_SESSION_ID=$SESSION_ID" >> "$CLAUDE_ENV_FILE"
fi
exit 0
```

## Migration Scope (from research)

**Path references that change (durable restructure):** ~10 files
- `CLAUDE.md` lines 5-9 (startup reads)
- `.claude/settings.json` (Stop hook, plansDirectory unchanged)
- `.claude/agents/plan-implementer.md`
- `.claude/reference/agent-discipline.md`, `memory-rules.md`
- `.claude/commands/complete_plan.md`, `resolve-linear-issue.md`
- `.claude/hooks/durable_memory_size_checker.sh`
- `.github/workflows/semantic-pr.yml`

**Paths that DON'T change:** All `thoughts/shared/` paths (briefs, plans, research, decisions, handoffs, reference) — 20+ files reference these but no migration needed.

**Full migration scope audit:** Completed by codebase-analyzer agent. 29+ files total reference `memory-bank`, but only ~10 need updating for the durable restructure. See the combined plan Phase 1 section 4 for the complete table.

## MCP Server Architecture (from research)

- **Transport:** MCP Streamable HTTP (spec 2025-03-26) — single `POST /mcp` endpoint, JSON-RPC, fully stateless. Works perfectly with Lambda.
- **Library:** `awslabs.mcp_lambda_handler` — `pip install`, `@mcp.tool()` decorator, `mcp.handle_request(event, context)`.
- **Claude Code integration:** `type: "http"` in `.mcp.json` with Function URL. `claude mcp add --transport http` CLI command.
- **Agent SDK integration:** `mcp_servers: {"name": {"type": "http", "url": "..."}}` in `ClaudeAgentOptions`.
- **6 MCP tools:** `get_state`, `search_memory`, `get_plan`, `get_brief`, `list_plans`, `list_briefs`.
- **Storage:** S3-backed. Durable files read fresh every invocation. Archive files cached in `/tmp` for warm Lambda reuse.

## Artifacts

- Combined plan: `memory-bank/thoughts/shared/plans/2026-03-03-centralized-memory-mcp.md`
- Brief: `memory-bank/thoughts/shared/briefs/2026-03-03-centralized-memory-mcp.md`
- Prior handoff: `memory-bank/thoughts/shared/handoffs/general/2026-03-02_16-31-03_memory-system-architecture-rethink.md`
- FS-1 plan: `memory-bank/thoughts/shared/plans/2026-03-03-fs1-agent-harness.md`
- FS-1 brief: `memory-bank/thoughts/shared/briefs/2026-03-02-fs1-agent-harness.md`
- Updated: `memory-bank/durable/01-active/current_work.md` (added plan link)

## Action Items & Next Steps

1. **Split the combined plan into two:**
   - **Plan A: Memory Rearchitecture** — Phase 1 (durable restructure) + frontmatter script + SessionStart hook for `$CLAUDE_SESSION_ID` + memory-rules.md rewrite. Pure restructure, no new infrastructure.
   - **Plan B: Memory MCP Server** — MCP package (`tools/memory-mcp/`) + S3 sync pipeline + Terraform deploy + `.mcp.json` wiring. Depends on Plan A (needs target durable structure), but could start Phase 2 (MCP package) in parallel.

2. **Add SessionStart hook** to Plan A — `.claude/hooks/session-start.sh` that extracts `session_id` and writes `export CLAUDE_SESSION_ID=...` to `CLAUDE_ENV_FILE`. Register in `.claude/settings.json`.

3. **Commit plans and brief to main** — per plan commitment protocol, plans must be on main before implementation. The brief and combined plan are written but uncommitted.

4. **After splitting, update `current_work.md`** with links to both plans.

5. **Consider:** Should the two plans have separate Linear issues? The memory rearchitecture is internal tooling; the MCP server is new infrastructure.

## Other Notes

- The combined plan at `2026-03-03-centralized-memory-mcp.md` has full implementation detail for all 4 phases. When splitting, the content can be mostly copy-pasted — Phase 1 → Plan A, Phase 2/3/4 → Plan B, with cross-references.
- The `awslabs.mcp_lambda_handler` library research is thorough — see the web-search-researcher output in this session for installation, code examples, and compatibility notes. Key finding: it handles JSON-RPC parsing, type validation, and MCP docs generation automatically.
- Claude Code's `CLAUDE_ENV_FILE` pattern is the cleanest way to propagate session ID without waiting for the native `CLAUDE_SESSION_ID` env var (feature requested but not implemented as of March 2026).
- The prior handoff's Q1-Q5 structural design questions are all resolved: Q1 (hot tier) → 5 semantic files; Q2 (warm tier) → eliminated, per-session journals; Q3 (cold tier) → keep lifecycle folders + add frontmatter; Q4 (.claude/ fit) → separate, procedural memory; Q5 (naming) → flat durable, dated archive.
