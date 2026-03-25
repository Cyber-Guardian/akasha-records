# Idea Brief: Commit Lineage Indexing

**Date:** 2026-03-01
**Status:** Seed (not yet shaped)

## Raw Thinking
- Memory-bank captures decisions and plans. Codebase captures current state. Neither captures *evolution*.
- "Why was this changed?" is unanswerable without reading raw git log
- Augment's Context Lineage feature: summarize every commit with a fast model → embed → searchable at query time
- We could do this locally without a hosted service
- Agents that understand code evolution make better decisions about what to change next

## Core Idea
Build a lightweight pipeline that summarizes git commit history into searchable, agent-consumable artifacts. Each commit (or logical batch) gets a short summary capturing: the goal, key files/functions touched, and technical terms for retrieval. These summaries live in the memory-bank as indexed markdown, queryable by skills like `/research_codebase` and `/shape`.

This bridges the gap between "what is the code now" (codebase) and "what did we decide" (memory-bank) by answering "how did the code get here" (lineage).

## Prior Art
- **Augment Context Lineage** — three-stage pipeline: commit harvesting → lightweight summarization (Gemini Flash) → embedding alongside code chunks. Summaries are ~a few sentences per commit, stored as embeddings. Enables "find a commit that did similar work" and "why was this changed?" queries.
- **vexp** (reviewed 2026-02-25) — local tree-sitter + SQLite graph. Focuses on current-state code structure, not history. Complementary.

## Possible Approaches

### A. Batch script + markdown archive
Periodically run a script: `git log --stat` → summarize N commits with Haiku → write to `memory-bank/thoughts/shared/lineage/YYYY-MM-week.md`. Skills search these files with Grep.
- Gains: Dead simple. No infrastructure. Works today.
- Costs: Not real-time. Grep isn't semantic search. Could grow large.

### B. Git hook + incremental summaries
Post-commit hook summarizes the commit → appends to a rolling lineage file. Agent reads on startup or when researching.
- Gains: Real-time. No batch job. Always current.
- Costs: Hook adds latency to commits. Needs API call per commit (cost).

### C. MCP server with embedded search
Index commit summaries into SQLite with embeddings. Expose via MCP `commit-search` tool. Any agent queries with natural language.
- Gains: Semantic search. Scales to large history. Clean integration.
- Costs: Most infrastructure. Embedding pipeline needed. Overkill for ~500 commits.

## Relationship to Existing Work
- **Memory-Bank OS** (2026-02-28) — lineage is a new content type for the knowledge graph. If/when Obsidian vault is set up, lineage summaries become nodes in the graph.
- **Hive Mind** (2026-02-28) — event log captures agent activity in real-time; lineage captures code evolution historically. Different time horizons, complementary.
- **Complexity-as-Moat** (2026-02-28) — agents that understand code evolution make higher-quality changes. Compounds the quality advantage.
- **vexp review** (2026-02-25) — vexp handles current-state code graph; lineage handles historical evolution. Orthogonal.

## Open Questions
- What granularity? Per-commit, per-PR (squash merge = 1 summary per feature), or per-day batch?
- How far back? Summarize full history once, then incremental? Or only from a cutoff?
- Storage: flat markdown (simple, grepable) vs structured (SQLite, queryable)?
- Cost: Haiku per commit is cheap (~$0.001), but full history of 500+ commits adds up for initial backfill. Worth it?
- Should this feed into `/shape` and `/create_plan` automatically, or only when explicitly researching?
