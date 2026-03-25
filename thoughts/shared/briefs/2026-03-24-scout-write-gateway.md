---
type: event
created: 2026-03-24
status: active
---
# Idea Brief: Scout as Read-Write Knowledge Gateway

**Date:** 2026-03-24
**Status:** Shaped -> Planning (v1 folded into akasha-records Wave 1)

## Problem
The akasha-records extraction has an 84-file blast radius because every skill, hook, and script hardcodes a filesystem path to the knowledge base. Centralizing access behind scout eliminates the path convention problem — consumers use vault-relative paths, scout resolves them. Additionally, raw agent writes to a shared knowledge base need a quality gate, not direct filesystem commits.

## Constraints
- Scout exists as read-only MCP with 16 tools (search, get, list, graph, exploration)
- Claude Code prompts can call MCP tools; Python hooks cannot (subprocesses)
- ~50 prompt files use `Write("memory-bank/...")` — can switch to `mcp__scout__put`
- ~10 hooks/scripts need filesystem access — use `AKASHA_PATH` env var
- Git operations require local clone
- Triage agent Lambda reads S3 — unchanged, separate path

## Options Considered

### Scout Write Gateway (minimal, no escape hatch)
put/delete/propose only. All writes go through PRs. No direct commits.
- Gains: Clean model, every write reviewed
- Costs: High friction for trivial updates (session closeout, topic notes)
- Complexity: Low

### Scout Write Gateway + Escape Hatch (chosen)
put/delete/propose + commit_direct with confirmation. Human-supervised → commit_direct. Autonomous → propose PR.
- Gains: Fast path for in-loop work, PR path for out-loop. Best of both.
- Costs: Escape hatch discipline needed
- Complexity: Low-Medium

### Full Knowledge Intake Pipeline (v2 vision, deferred)
Scout writes + PRs + dedicated intake agents (GitHub Actions or ECS). LLM reviewer, auto-merge, knowledge quality assessment.
- Gains: Self-maintaining knowledge base, autonomous contribution at scale
- Costs: Significant infrastructure, overkill for current team size
- Complexity: High

## Chosen Approach
**Scout Write Gateway + Escape Hatch** — maps to the in-loop/out-loop model. `commit_direct` for human-supervised sessions, `propose` for autonomous agent contributions. Intake CI (frontmatter + wikilinks) as lightweight GitHub Actions on akasha-records.

## Full Vision (v1 → v2 → v3)

### v1: Scout Write Gateway (Wave 1 deliverable)
- `put(path, content)` — write file to vault working tree, validate frontmatter
- `delete(path)` — remove file from vault working tree
- `propose(message, branch?)` — stage changes, push branch, open PR to akasha-records
- `commit_direct(message, confirm)` — commit + push to main (requires `confirm=True`, skills only pass after user approval)
- Intake CI: GitHub Actions on akasha-records PRs — frontmatter validation, wikilink integrity (obsidiantools), basic checks
- All 50+ prompt files updated to use `mcp__scout__put` instead of `Write("memory-bank/...")`
- ~10 hooks/scripts use `AKASHA_PATH` env var

### v2: Knowledge Intake Agents (follow-up plan)
- Dedicated intake agent reviews incoming PRs (LLM-powered)
- Checks: does this brief contradict existing decisions? Is the topic note update accurate? Are wikilinks semantically correct?
- Auto-merge if all checks pass (high-confidence contributions)
- Request changes with specific feedback if not
- Scheduled consolidation agent: merges related notes, detects stale topics, proposes archive

### v3: Scout as Remote Service (future)
- Scout deployed as HTTP service (not just local stdio MCP)
- Remote agents (triage, helm workers, ECS tasks) access knowledge via scout API
- Replaces S3 sync pipeline for triage agent
- Single knowledge API for all consumers, local and remote

## Key Context Discovered During Shaping
- The 84-file blast radius is caused by scattered filesystem path knowledge, not the repo split itself
- Scout already has the vault-path abstraction (`--vault-path` in `.mcp.json`) — adding write tools extends it naturally
- Python hooks can't call MCP tools (subprocess isolation) — they need `AKASHA_PATH` env var as fallback
- `propose` maps to git branch + push + `gh pr create` — ~50 lines of code
- `commit_direct` confirmation can be a simple parameter gate: skills only pass `confirm=True` after explicit user instruction
- Frontmatter validation currently in `frontmatter.py` hook (path-matching) — moves into scout's `put` tool (path-agnostic)
- The in-loop/out-loop distinction maps perfectly: human sessions → commit_direct, autonomous agents → propose

## Related Artifacts
- [[2026-03-24-akasha-records-v2|Akasha Records v2 Brief]] — the extraction this enables
- [[2026-03-23-scout-unified-mcp|Scout Unified MCP Plan]] — scout's current read-only state
- [[2026-03-23-assembly-line-work-ontology|Assembly-Line Work Ontology]] — DISTILL step uses scout write tools

## Next Step
- [Plan] → v1 folded into rewritten akasha-records Wave 1 plan. v2 and v3 captured as Linear backlog issues.
