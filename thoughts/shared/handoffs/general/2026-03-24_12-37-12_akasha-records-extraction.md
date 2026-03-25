---
type: event
created: 2026-03-24
status: active
date: 2026-03-24T16:37:12Z
author: claude-opus-4-6
git_commit: 4bdb304f
branch: main
repository: filescience
topic: "Akasha Records Extraction + Scout Write Gateway Handoff"
tags: [handoff, akasha-records, scout, knowledge-architecture]
last_updated: 2026-03-24
last_updated_by: claude-opus-4-6
---

# Handoff: Akasha Records Extraction — Plans Rewrite In Progress

## Task(s)

| Task | Status |
|------|--------|
| Scout unified MCP (PR #183) | **completed** — merged to main, all 16 tools, 62 tests |
| Workspace container resilience (PR #186) | **completed** — merged to main, health protocol, staleness detection |
| Akasha-records v2 shaping | **completed** — two-wave approach with scout write gateway |
| Scout write gateway shaping | **completed** — put/delete/propose/commit_direct tools |
| Wave 1 plan rewrite | **in_progress** — first draft reviewed (3/10), rewriting with all fixes |
| Wave 2 plan rewrite | **in_progress** — first draft reviewed (3/10), rewriting with all fixes |
| Linear issues for v2/v3 vision | **completed** — ENG-2784 (intake agents), ENG-2785 (scout HTTP) |

## Critical References

1. **Brief (v2 extraction):** `memory-bank/thoughts/shared/briefs/2026-03-24-akasha-records-v2.md`
2. **Brief (scout write gateway):** `memory-bank/thoughts/shared/briefs/2026-03-24-scout-write-gateway.md`
3. **Current Wave 1 plan (NEEDS REWRITE):** `memory-bank/thoughts/shared/plans/2026-03-24-akasha-records-wave1.md`
4. **Current Wave 2 plan (NEEDS REWRITE):** `memory-bank/thoughts/shared/plans/2026-03-24-akasha-records-wave2.md`
5. **Blast radius report:** 84 files reference `memory-bank/` — full list in session context (agent output from `a58e4262211092cb2`)

## Recent changes

- `c02e0e0f` — PR #183 merged (scout unified MCP)
- `3ae22cd9` — PR #186 merged (workspace resilience)
- `68d63f69` — Fixed server.py resilience functions accidentally reverted by 44db856
- `ec679012` — Alignment cleanup (stale docstring, rules, tests, plan archival)
- `cc91257e` — Wave 1 + Wave 2 plans (first drafts, need rewrite)
- `4bdb304f` — Scout write gateway brief

## Learnings

### Architecture Decision: Scout as Knowledge Gateway
- **All knowledge access routes through scout MCP** — eliminates the 84-file path convention problem
- **Prompt files** (~50): use `mcp__scout__put("thoughts/shared/briefs/topic.md", content)` — vault-relative paths, no filesystem knowledge
- **Python hooks/scripts** (~10): use `AKASHA_PATH` env var with fallback `../akasha-records`
- **CI/Terraform** (~9): unchanged (S3 prefix stays `memory-bank/`)

### Scout Write Tools (v1, ~80 lines total)
- `put(path, content)` — write to vault working tree, validate frontmatter
- `delete(path)` — remove from vault working tree
- `propose(message)` — stage, branch, push, open PR to akasha-records
- `commit_direct(message, confirm)` — escape hatch, requires explicit `confirm=True`
- Frontmatter validation moves FROM `frontmatter.py` hook INTO scout's `put` tool

### Ad Review Findings (14 missing files across both plans)
Both plans scored 3/10. Key issues to fix in rewrite:

**Wave 1 blockers:**
- OIDC role is per-repo scoped — new role needed for akasha-records (Task 0 prerequisite)
- Disable `memory-sync.yml` FIRST in Phase 2 (S3 race window)
- `frontmatter.py` path resolution — moves into scout `put` tool
- `session_start.py:384,388` — hardcoded `memory-bank/topics/` in instruction string AND path
- `graph.py:233` — `removeprefix("memory-bank/")` breaks after vault repoint
- `docs/` directory must exist before `lint.yml` path update
- Delete scout LanceDB index after repoint
- `build_memory_index.py` — decide fate of filescience copy

**Wave 2 blockers:**
- `generate_scope.py:23` — `ALWAYS_ALLOWED` has stale `memory-bank/**`
- `settings.json:115` — Stop hook prompt hardcodes plan path
- Assembly-line `plan_graph.py` collision

**Missing from both plans' file lists:**
- `generate_scope.py`, `settings.json` Stop hook, `thoughts-locator.md` (11 refs), `wrapup/SKILL.md`, `durable_memory_size_checker.sh`, `memory_maintenance.py` + skill invocations, `cyrus-setup.sh:31` fallback scope, `agent-memory-conventions.md` S3 semantics, `complete_plan.md` topic note path

### Ground Results
- 5/6 assumptions hold. OIDC role doesn't hold (per-repo trust policy).
- `git mv --follow` works. Scout is path-agnostic. S3 sync is transparent. Frontmatter fix is trivial.

### Workspace Container Insights
- Stale Docker image was root cause of container deaths (entrypoint.sh guard missing)
- Image staleness now auto-detected via source SHA Docker label
- Health file protocol: `/workspace/.health` written on success/failure
- `--init` flag on all containers for zombie reaping
- `commit 44db856` accidentally reverted server.py resilience — caught by alignment audit, restored

## Artifacts

| Artifact | Path | Status |
|----------|------|--------|
| Scout unified MCP brief | `memory-bank/thoughts/shared/briefs/2026-03-23-scout-unified-mcp.md` | archived |
| Scout unified MCP plan | `memory-bank/thoughts/shared/plans/2026-03-23-scout-unified-mcp.md` | archived |
| Workspace resilience brief | `memory-bank/thoughts/shared/briefs/2026-03-23-workspace-container-resilience.md` | archived |
| Workspace resilience plan | `memory-bank/thoughts/shared/plans/2026-03-23-workspace-container-resilience.md` | archived |
| Akasha records v2 brief | `memory-bank/thoughts/shared/briefs/2026-03-24-akasha-records-v2.md` | active |
| Scout write gateway brief | `memory-bank/thoughts/shared/briefs/2026-03-24-scout-write-gateway.md` | active |
| Wave 1 plan (NEEDS REWRITE) | `memory-bank/thoughts/shared/plans/2026-03-24-akasha-records-wave1.md` | active |
| Wave 2 plan (NEEDS REWRITE) | `memory-bank/thoughts/shared/plans/2026-03-24-akasha-records-wave2.md` | active |
| Original akasha-records plan | `memory-bank/thoughts/shared/plans/2026-03-20-akasha-records.md` | superseded |
| Assembly-line ontology brief | `memory-bank/thoughts/shared/briefs/2026-03-23-assembly-line-work-ontology.md` | active |
| Assembly-line ontology plan | `memory-bank/thoughts/shared/plans/2026-03-23-assembly-line-work-ontology.md` | active |
| Linear: intake agents | ENG-2784 (backlog) | v2 vision |
| Linear: scout HTTP service | ENG-2785 (backlog) | v3 vision |
| Auto-memory | `~/.claude/projects/.../memory/project_akasha_records_v2.md` | active |

## Action Items & Next Steps

### Immediate (this handoff)
1. **Rewrite Wave 1 plan** — incorporate scout write gateway (put/delete/propose/commit_direct), all 14 missing files, OIDC prerequisite, path convention (prompts → scout tools, hooks → AKASHA_PATH). The plan structure changes:
   - Phase 0: Scout write tools (put/delete/propose/commit_direct, ~80 lines)
   - Phase 1: Create akasha-records repo + CI (S3 sync, OIDC role)
   - Phase 2: Filescience cutover (~50 prompt files → mcp__scout__put, ~10 hooks → AKASHA_PATH)
   - Phase 3: Verification + onboarding

2. **Rewrite Wave 2 plan** — move plans to `.work/plans/`, DISTILL step uses `mcp__scout__put`. Fix: generate_scope.py, settings.json Stop hook, assembly-line collision, all missing files.

3. **Run `/alignment` on both rewritten plans** — verify all ad review findings addressed

4. **If approved, implement Wave 1** on deep agents

### Deferred
- ENG-2784: Knowledge intake agents (v2) — after Wave 1 ships
- ENG-2785: Scout HTTP service (v3) — after v2
- Quartz static site — after repo exists, independent of scout

## Other Notes

### Key Architectural Diagram
```
Akasha Records (repo)                    Filescience (repo)
┌─────────────────────────┐             ┌──────────────────────┐
│ durable/identity.md     │             │ .work/plans/ (Wave 2)│
│ topics/*.md             │  ◄──scout── │ .work/templates/     │
│ thoughts/shared/        │  put/get/   │ code, infra, tools   │
│   briefs/               │  search/    │ docs/state.md        │
│   research/             │  propose    │ .claude/ (procedural)│
│   decisions/            │             │ tools/scout/ (MCP)   │
│   reference/            │             └──────────────────────┘
└──────────┬──────────────┘
           │ CI                     Intake Flow:
    ┌──────┴──────┐                 scout put → working tree
    ▼             ▼                 scout propose → PR
 S3 mirror    (Quartz)             intake CI → validate
 (triage)     (deferred)           merge → main
                                   (or commit_direct → main)
```

### Session Vital Stats
- Session ID: 8f8778af-e94b-4654-9f0f-2475cd5c4ed4
- PRs merged: #183 (scout), #186 (workspace resilience)
- Plans archived: scout-unified-mcp, workspace-container-resilience
- Plans superseded: original akasha-records (2026-03-20)
- Context is heavy — fresh session recommended for plan rewrites
