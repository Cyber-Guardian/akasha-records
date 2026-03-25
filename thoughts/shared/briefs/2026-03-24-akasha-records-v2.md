---
type: event
created: 2026-03-24
status: active
---
# Idea Brief: Akasha Records v2 — Two-Wave Knowledge Extraction

**Date:** 2026-03-24
**Status:** Shaped -> Planning

## Problem
`memory-bank/` mixes knowledge artifacts (briefs, research, decisions, topics, identity) with workflow artifacts (plans) inside a code repo. As the team grows and Akasha becomes the parent company, knowledge needs its own repo while plans stay close to the code they modify. Scout now replaces QMD + graph-memory, simplifying the infrastructure needed. The original akasha-records plan (2026-03-20) assumed FTS5-on-S3 + QMD + graph-memory — all obsolete.

## Constraints
- Scout is the unified MCP — just needs `--vault-path` repointed
- Triage agent reads from S3 (`memory-bank/` prefix) — must keep working
- Claude Code submodules don't work (issue #7852) — separate clone only
- Quartz + symlinks breaks (issue #1272) — CI must clone content, not symlink
- Assembly-line work ontology plan not shipped yet — plans shouldn't move until it does
- 84 files reference `memory-bank/` — ~60 for knowledge paths, ~20 for plan paths

## Options Considered

### Two-Wave Migration (chosen)
Wave 1: knowledge to akasha-records, S3 sync transfers, scout repointed. Wave 2: plans to `.work/plans/` when assembly-line ships, DISTILL step added.
- Gains: Decoupled, each wave self-contained, low risk per wave
- Costs: Partial state between waves (memory-bank/ has only plans)
- Complexity: Medium per wave

### Full Atomic Split
Everything moves at once — knowledge to akasha-records, plans to .work/plans/, all 84 references updated.
- Gains: Clean break, no intermediate state
- Costs: 84-file blast radius, depends on assembly-line readiness
- Complexity: High

### Git Subtree
Split memory-bank/ via git subtree, content lives in both repos.
- Gains: Zero-downtime, git-native
- Costs: Operationally complex, risk of permanence
- Complexity: Medium

## Chosen Approach
**Two-Wave Migration** — decouples the knowledge split (ready now) from the plan migration (blocked on assembly-line). Each wave is independently verifiable with manageable blast radius.

## Key Context Discovered During Shaping
- Scout replaced QMD + graph-memory — no FTS5 pipeline needed, no S3 sync for search index
- Triage agent S3 integration is independent from scout (Lambda reads S3 directly, scout is local MCP)
- S3 sync must continue from akasha-records CI — same bucket (`filescience-testing-artifacts`), same prefix (`memory-bank/`)
- `check_scope.py:35` hardcodes `PLANS_DIR = "memory-bank/thoughts/shared/plans"` — Wave 2 concern
- `.claude/settings.json:2` has `plansDirectory` — Wave 2 concern
- Separate clone at sibling path (`../akasha-records`) is the industry pattern for agent-accessible knowledge repos
- Quartz deployment needs clone-in-CI approach (not symlink) per jackyzha0/quartz#1272

## Related Artifacts
- [[2026-03-20-akasha-records|Original Akasha Records Plan]] — superseded by this brief
- [[2026-03-23-scout-unified-mcp|Scout Unified MCP Plan]] — delivered, eliminates QMD/graph-memory
- [[2026-03-23-assembly-line-work-ontology|Assembly-Line Work Ontology]] — Wave 2 depends on this
- [[2026-03-20-personal-life-ontology-agent-platform|Akasha Interface Substrate]] — parent vision

## Next Step
- [Plan] -> `/create_plan memory-bank/thoughts/shared/briefs/2026-03-24-akasha-records-v2.md` (both Wave 1 and Wave 2 plans)
