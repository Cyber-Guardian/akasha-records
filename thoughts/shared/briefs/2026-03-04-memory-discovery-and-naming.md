# Idea Brief: Memory Discovery, Naming, and Session Extraction

**Date:** 2026-03-04
**Status:** Shaped → Updating Plan

## Problem
Agents have no reliable way to discover relevant prior work across 200+ markdown files. The v2 plan's `active/` tier uses filename matching which breaks on vocabulary mismatch. Session learnings evaporate because nothing extracts journal findings into durable notes. The name "active" implies a lifecycle state rather than the living knowledge base it actually is.

## Constraints
- SessionStart hook has ~275 token budget, 15-second timeout
- Stop hook has 30-second timeout — cannot accommodate multi-file extraction
- Tags rot without enforcement (no vocabulary, no validation hook)
- LLM-based extraction is non-deterministic — promotes dead-end hypotheses as findings
- ~200 cold archive files have no frontmatter (mostly) — lexical search only

## Options Considered

### Conventions-only (tags + enriched SessionStart)
Add `tags:` frontmatter to wiki notes, inject enriched filename+tags in SessionStart.
- Gains: Zero dependencies, immediate
- Costs: Tag rot guaranteed without enforcement. Context budget ~doubled. Vocabulary mismatch unsolved.
- Complexity: Low
- Adversarial review: BLOCK — tag rot, phantom mechanism, context budget

### Automated closeout extraction (Stop hook)
Scan session log.md for findings, auto-update matching wiki notes on session stop.
- Gains: Systematic extraction without manual effort
- Costs: 30s timeout can't accommodate. LLM non-determinism promotes noise. Interrupted sessions leave partial writes.
- Complexity: High
- Adversarial review: BLOCK — timeout, non-determinism, no interrupt recovery

### QMD MCP + lightweight closeout prompt
Install QMD (local hybrid BM25+vector search), add MCP server. Closeout is a CLAUDE.md prompt, not automation.
- Gains: Solves vocabulary mismatch mechanically. Works across all tiers. No tag maintenance. Agent retains judgment.
- Costs: 2GB model download. Another MCP server. Cold query latency.
- Complexity: Medium

## Chosen Approach
**QMD MCP + `topics/` naming + lightweight closeout prompt** — QMD kills the discovery problem at the tool level (no tags needed). `topics/` is honest naming without lifecycle or wiki connotation. Closeout prompt nudges extraction without fragile automation.

## Key Context Discovered During Shaping
- Adversarial review found all three original proposals structurally flawed (2/10)
- QMD (github.com/tobi/qmd) provides local semantic search with MCP server — hybrid BM25+vector+reranking
- mgrep exists but sends files to cloud — privacy concern for architecture docs
- basic-memory requires reformatting all files — too invasive
- Tag rot is the dominant failure mode for any tag-based discovery system without enforcement

## Next Step
- [Updating plan] → modifying `memory-bank/thoughts/shared/plans/2026-03-04-memory-system-v2-rearchitecture.md`
