---
type: event
created: 2026-03-20
status: active
---
# Idea Brief: Akasha Records — Shared Knowledge Base for Unified Execution Substrate

**Date:** 2026-03-20
**Status:** Shaped -> Planning

## Problem
FileScience's memory-bank has outgrown its role as a subdirectory — 467 files, 6.9 MB, growing with every shaped idea and plan. It clutters the filescience repo's git history and PR diffs, but it's already infrastructure: CI syncs it to S3, the triage bot queries it there, QMD indexes it locally. If we're building a unified execution substrate where multiple agents (Claude Code, Helm-dispatched workers, nightly scheduled agents, future autonomous agents) all need shared knowledge, the knowledge layer should be a first-class system — not a subdirectory.

## Constraints
- The S3 sync pattern already works and is proven (triage bot: FTS5 index + raw markdown on S3)
- Lambda agents already query S3, not git — access pattern doesn't change for them
- Claude Code sessions (local) currently use QMD over the local filesystem
- Obsidian works fine as the human editing interface (single-writer model)
- Git repo of 500-2000 markdown files scales trivially (pain starts at ~100K+)
- S3 has strong read-after-write consistency since 2020
- SQLite FTS5 index must be built in DELETE journal mode (not WAL) before S3 upload to avoid sidecar file trap
- Quartz v4 global graph degrades at scale — disable `Component.Graph()`, use per-page backlinks
- CI sync pipelines need GitHub Actions `concurrency:` groups to prevent race conditions

## Options Considered

### Git Repo + S3 Mirror (extend existing pattern)
Akasha-records as its own GitHub repo. CI syncs to S3 on push (same as today). Lambda agents keep querying S3. Local Claude Code sessions clone akasha-records as sibling directory.
- Gains: Minimal new infra, proven pattern, clean separation
- Costs: Local dev needs two repos cloned; QMD config changes; CLAUDE.md paths all update
- Complexity: Low

### Git Repo + MCP Facade over S3
Same as above, but build a thin MCP server wrapping the S3-hosted index. All agents go through same MCP interface.
- Gains: Uniform access pattern everywhere; could add write-back capability
- Costs: New MCP server to build and maintain; overhead for what's currently a simple S3 read
- Complexity: Medium

### Git Repo + Quartz Publish + Read-Only Web
Same git repo, Quartz v4 publishes a static site on push. Agents get read access via S3. Humans get browsable web UI.
- Gains: Browsable knowledge base for team; zero-runtime; frontmatter gates what's public
- Costs: Quartz setup + GitHub Pages config; doesn't change agent access story
- Complexity: Low (additive, orthogonal)

## Chosen Approach
**Combined: Git Repo + S3 Mirror + Quartz + Three-Tier Search + Maintenance Agents**

The architecture:
- **Source of truth:** Separate GitHub repo ("akasha-records") containing Obsidian vault
- **Write path:** PRs — normal agents are additive (create/edit). Dedicated maintenance agents handle structural operations (renames, defrags, wikilink fixes) on a schedule.
- **Read path (deployed twin):** CI on merge to main triggers S3 sync (raw .md + FTS5 index + vector embeddings) + Quartz static site build
- **Three-tier search:** (1) FTS5 keyword search — instant, for exact lookups. (2) Vector semantic search — ~2s, for meaning-based queries. (3) Agentic exploration — seconds, an agent on AWS that reads/synthesizes/curates context for complex queries. These are tools, not dogma — callers pick the right tier.
- **Maintenance agents:** Nightly/scheduled agents that handle orphan detection, broken wikilinks, stale note detection, file reorganization. Maps to existing lifecycle brief and nightly agent plans.

## Key Context Discovered During Shaping

### Validated by prior art
- Letta Context Repositories: nearly identical git-backed agent memory with progressive disclosure
- GitAgent: mandatory PR governance for agent knowledge contributions
- Git Context Controller (arXiv): 48% on SWE-Bench using git-filesystem as agent memory
- Codified Context (arXiv): 3-tier hot/domain/cold memory with MCP retrieval — we're ahead (QMD has semantic search, theirs is keyword-only)
- GitHub Copilot memory: repo-scoped, just-in-time verification against cited code locations

### Existing patterns to extend
- Triage bot S3 pattern: `memory-bank/ (git) -> CI sync -> S3 (FTS5 + raw .md) -> agent tools`
- Memory system lifecycle brief: maintenance automation, orphan detection, wikilink fixes
- Nightly scheduled agents brief: memory-bank staleness detection, knowledge gap detection

### Pitfalls to design around
- SQLite WAL sidecar trap: build FTS5 in DELETE journal mode before S3 upload
- Concurrent CI sync races: GitHub Actions `concurrency:` groups
- Quartz global graph performance: disable at scale, use per-page backlinks
- Agent file renames breaking wikilinks: solved by dedicated maintenance agents with obsidiantools
- QMD needs local files: local clone for Claude Code sessions, S3 for cloud agents

## Related Artifacts
- [[2026-03-12-memory-system-lifecycle|Memory System Lifecycle Brief]]
- [[2026-02-26-nightly-scheduled-agents|Nightly Scheduled Agents Brief]]
- [[2026-03-04-obsidian-graph-sidecar|Obsidian Graph Sidecar Plan]]
- [[2026-03-03-memory-system-rearchitecture|Memory System Rearchitecture Brief]]

## Next Step
- [Plan] -> `/create_plan memory-bank/thoughts/shared/briefs/2026-03-20-akasha-records.md`
