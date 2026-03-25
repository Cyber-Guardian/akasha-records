# Memory & Document Storage System — Deep Research Synthesis

**Date:** 2026-03-11
**Sources:** 9 parallel research agents covering: current system audit, memory system artifact analysis, AI agent memory architectures, knowledge management at scale, semantic search scaling, AI agent document storage, Claude Code community patterns, and competitive tool analysis.

---

## The Problem

FileScience's memory-bank (362 markdown files, 4.9MB) is about to grow significantly. The bizdev operating system plan adds 3 recurring files per week (~156/year). The current system has no lifecycle management — no rollup, no staleness detection, no recency-aware retrieval, no distinction between one-time research and recurring reports. Without changes, the research/ directory (already 160 files) will double in a year, QMD search quality will degrade on temporal queries, and the sales director will get stale competitive intel because vector search has no concept of "latest."

---

## Current System Audit

### What's Built (v2 four-tier architecture — fully operational)

| Tier | Location | Contents | Access |
|------|----------|----------|--------|
| Hot/Durable | `memory-bank/durable/` | 3 files: `identity.md` (49 lines), `state.md` (24 lines, CI-generated), `scorecard-template.md` (263 lines — **shouldn't be here**) | Loaded every session |
| Warm/Topics | `memory-bank/topics/` | 13 notes, all active, 25-43 lines each, YAML frontmatter | On-demand via QMD |
| Session | `.claude/sessions/{id}/` | `todo.md` + `log.md` per session. 40+ dirs accumulated, no cleanup | Per-session, not committed |
| Cold/Archive | `memory-bank/thoughts/shared/` | 6 subdirs: research (160), briefs (68), plans (80), decisions (5), reference (8), handoffs (15) | On-demand via QMD |

### File Distribution

| Directory | Count | Notes |
|-----------|-------|-------|
| research/ | 160 | Largest by 2x. ~80 are research+brief pairs. Includes scorecard evaluations. |
| plans/ | 80 | ~10 have random-word names from Helm (e.g., `fuzzy-hopping-boole.md`) |
| briefs/ | 68 | Standalone shaped-idea docs from `/shape` route |
| handoffs/ | 15 | One outlier placed at root instead of `general/` |
| reference/ | 8 | Stable reference documents |
| decisions/ | 5 | Architectural Decision Records |
| topics/ | 13 | All active, all within size targets |
| durable/ | 3 | `identity.md` + `state.md` + stray `scorecard-template.md` |

### Live Bugs

1. **`scorecard-template.md` in durable/ (263 lines)** — loads into every session context window. It's a template, not identity. Should be in `.claude/reference/` or `thoughts/shared/reference/`.
2. **CLAUDE.md Route K references deleted `queue.md`** — Route K says "Update: `queue.md`" but `queue.md` was deleted in v2 migration. Linear is the correct target. Any session following Route K will attempt to update a non-existent file.
3. **`state.md` CI health shows "not found"** — workflow names in `generate_state.py` don't match actual `.github/workflows/` names. Every session sees broken CI health data.
4. **40+ session dirs with no cleanup** — planned cleanup (delete >7 days) was never implemented.

### Metadata Coverage

- **19% of archive files have YAML frontmatter** (~69 of 362). Only files created after the v2 migration (2026-03-04+) consistently carry frontmatter.
- **Wikilinks**: 295 occurrences across 83 files — concentrated in recent files. Pre-Feb 2026 files use plain paths.
- **`related:` YAML field**: present in all 13 topic notes, 4 archive files. Uses plain paths, not wikilinks.
- **Research+brief pairs**: `-brief.md` companion files have `source_research:` frontmatter linking to parent. This is the one well-maintained cross-reference pattern.

---

## What the State of the Art Says

### The Universal Pattern: Every Production Memory System Converges on Tiers

Every system studied — Mem0, Zep/Graphiti, Letta/MemGPT, LangGraph, CrewAI, Claude Diary, Cline Memory Bank, Windsurf Memories, AWS AgentCore, Google ADK — implements tiered memory. The tiers vary in naming but the structure is identical:

| Tier | What | Lifecycle | FileScience Equivalent |
|------|------|-----------|----------------------|
| Always-loaded | Identity, constraints, active instructions | Human-maintained, rarely changes | `durable/identity.md` |
| On-demand warm | Topic knowledge, living documents | Agent-updated, reviewed by human | `topics/*.md` |
| Searchable cold | Historical artifacts, completed work | Append-only, rolled up periodically | `thoughts/shared/**` |
| Ephemeral | Session state, working memory | Per-session, discarded after | `.claude/sessions/` |

FileScience already has this structure. The gap is not architecture — it's **lifecycle management within the cold tier**.

### Finding 1: Two-Class Content Model (Most Important)

Every mature system distinguishes between two fundamentally different content types with different lifecycle rules:

**Event notes** (dated, immutable, append-only):
- Session logs, research investigations, debug notes, weekly intelligence reports
- Created with a date prefix, never modified after creation
- Accumulate over time → roll up into summaries → originals deleted after rollup
- The rollup IS the archive — deleting originals after rollup is not information loss

**State notes** (evergreen, edited in place):
- Topic notes, identity, architectural decisions, competitive landscape summary
- One canonical version that gets updated, never duplicated
- An agent touching a state note should UPDATE it, not create a sibling

FileScience implicitly has this (topic notes = state, research = event) but it's not enforced. The key enforcement mechanism from Mem0: **before creating a new file, vector-search existing files. If similarity > 0.85, UPDATE the existing note instead of creating a new one.**

### Finding 2: Temporal Knowledge Is the #1 Unsolved Problem

**The core failure (from Zep, confirmed across all research):** Pure vector search has no temporal model. When a fact changes (competitor repositions, regulation updates), vector search returns the most *semantically similar* fact — which is the OLD one, not the current one.

This is not a theoretical concern. With 52 weekly competitive-intel files all discussing Veeam's positioning, QMD vector search will surface whichever file's language best matches the query — likely not the most recent one.

**Solutions (ordered by implementation effort):**

1. **"Latest" pointer files** (simplest): A stable file in `durable/` that contains the path to the most recent instance of each recurring report. Skills read the pointer instead of searching.

2. **`is_current: true` metadata flag**: Add to frontmatter of the latest instance of each recurring report. Pre-filter QMD results to `is_current` docs. Requires maintaining the flag (set on new, unset on previous).

3. **Recency-weighted scoring**: `final = semantic × (1-β) + recency × β`, where β = 0.15-0.25. Post-process QMD results with date-weighted scoring. QMD doesn't natively support this.

4. **Bi-temporal model** (Zep/Graphiti): Track when a fact was true AND when we learned it. Old facts get end-dated, not deleted. Full implementation requires a knowledge graph — overkill for current scale.

### Finding 3: Fractal Rollup for Event Notes

From Obsidian power users (Steph Ango's fractal journaling), Confluence best practices, enterprise data archiving, and Mem0/SimpleMem:

```
Weekly files (hot, current)
    ↓ after 4 weeks
Monthly summaries (warm, synthesized)
    ↓ after 6 months
Quarterly summaries (cold, permanent)
```

- The rollup can be **deterministic extraction** (pull specific frontmatter fields, key findings, change flags) or **LLM-generated** (MapReduce summarization). Deterministic is safer — LLM summaries are probabilistic.
- **SimpleMem** (arxiv, Jan 2026) achieves 89-95% compression, 30x token reduction by clustering related memory units when affinity > 0.85 and synthesizing into higher-level abstractions.
- **Mem0's pipeline**: For each new fact, retrieve top 10 similar existing memories. LLM classifies as ADD (new), UPDATE (augment existing), DELETE (contradiction), or NOOP. This prevents duplication at creation time rather than cleaning up after.

### Finding 4: Progressive Disclosure for Retrieval

From claude-mem (most sophisticated Claude Code memory hook):

- **3-layer retrieval**: compact index (~50-100 tokens) → timeline layer → full detail (~500-1000 tokens)
- Only fetch depth on demand — claims 10x token savings
- Applied to bizdev brief: load the 1-page digest first, drill into full competitive-intel only if needed

From Anthropic's own context engineering guidance:
- "Just in time" context > upfront loading
- Agents maintain lightweight identifiers (paths, queries) and dynamically load data via tools
- Sub-agents return condensed summaries (1,000-2,000 tokens) to coordinators

### Finding 5: Category-Aware Search Beats Unified Search

From semantic search scaling research:

- At <200 docs per category, brute-force within a namespace beats HNSW across the full corpus
- Pre-filtering by category before vector search eliminates recall degradation
- **At 362 docs total, QMD's search quality won't degrade from collection size** — the real problem is temporal, not volumetric
- HNSW recall degradation starts at ~10K vectors. With section-level chunking (3-8 chunks per doc), the collection would need 1,000+ docs before this matters
- **QMD already uses SQLite FTS5 + vector + RRF fusion** — this is the right architecture for current scale

### Finding 6: Frontmatter as Lifecycle Metadata

From every knowledge management system studied, the minimum viable metadata for lifecycle management:

```yaml
type: event | state          # lifecycle class
created: YYYY-MM-DD          # immutable creation date
status: active | superseded | archived  # lifecycle state
series: competitive-intel    # for recurring reports (optional)
```

This makes staleness detectable, rollup automatable, and retrieval filterable. Currently only 19% of files have any frontmatter at all.

### Finding 7: Maintenance Scripts Are Table Stakes

From obsidiantools (Python package for vault analytics):

- `vault.isolated_notes` — zero-backlink notes (orphans)
- `vault.nonexistent_notes` — broken wikilinks
- `vault.get_note_metadata()` — DataFrames with backlink counts, last-modified, wikilink stats

A weekly script that reports: orphaned notes (zero backlinks + old), broken wikilinks, state notes past their review date, and event notes older than the rollup threshold would prevent the collection from silently degrading.

The `graph-memory` MCP server already in the stack could power this.

---

## How Others Handle the Same Problem

### Claude Code Community Patterns

| Pattern | Author | Key Idea | Relevance |
|---------|--------|----------|-----------|
| Claude Diary | rlancemartin (LangChain) | `/diary` captures session artifacts → `/reflect` synthesizes patterns → proposes CLAUDE.md updates (human reviews before applying) | Direct model for bizdev brief rollup |
| claude-mem | thedotmack | PostToolUse hooks capture tool execution → two-stage compression → 3-layer progressive retrieval | Architecture for efficient retrieval |
| Modular CLAUDE.md | centminmod | Split into topic files (`-activeContext.md`, `-patterns.md`, `-decisions.md`) with index | Already close to FileScience's approach |
| Memory Bank Synchronizer | centminmod | Agent diffs codebase state against memory docs — catches staleness | Pattern for maintenance scripts |
| PreCompact hook | Community (Jan 2026) | Fires before autocompact, generates handover summary before context lost | 30% reduction in info loss during compaction |

### Competitor Tool Memory

| Tool | Approach | Strength | Weakness |
|------|----------|----------|----------|
| Cline | 6 structured markdown files, methodology enforced by custom instructions | Clean separation of concerns, explicit update triggers | No auto-generation; relies on user discipline |
| Windsurf | Auto-generated memories + human-authored rules | Two-class model: auto memories are supplementary, rules are reliable | Auto-memory retrieval is opaque ("when Cascade believes relevant") |
| Cursor | Per-file `.cursor/rules/*.mdc`, no write-back | Static rules are reliable | No memory consolidation, no learning from sessions |

**The competitive consensus**: Static rules (reliable, version-controlled, human-authored) + session-specific memory (less reliable, automatic). None have solved automatic knowledge distillation from sessions into durable rules.

### Agent Framework Memory

| System | Temporal Reasoning | Conflict Resolution | Retrieval Cost | Build Complexity |
|--------|-------------------|---------------------|----------------|-----------------|
| Letta/MemGPT | Weak | Manual (LLM edits core) | Low (tiered) | Medium |
| LangGraph Store | None built-in | None built-in | Low (vector) | Low framework, high custom |
| CrewAI | Weak | LLM-based | Medium (RAG) | Low |
| Zep/Graphiti | Strong (bi-temporal) | Automatic invalidation | High (graph traversal) | High |
| Mem0 | Moderate | LLM-driven CRUD | Low (0.20s p50) | Low (managed SaaS) |
| SimpleMem | Moderate | Recursive consolidation | Low | Medium |

### Production Artifact Management

| System | Pattern | Key Feature |
|--------|---------|-------------|
| Google ADK | Append-versioned named artifacts | `save_artifact` always creates new version; `load_artifact` returns latest by default |
| ReMe (AgentScope) | `MEMORY.md` + `memory/YYYY-MM-DD.md` journals | Compactor generates structured summaries with explicit fields (Goal, Constraints, Progress, Key Decisions, Next Steps) |
| Anthropic Memory Tool | Client-side filesystem abstraction | Agent decides file structure; `view`, `create`, `str_replace` operations on `/memories` |

---

## What Fails (Anti-Patterns)

1. **RAG for dynamic memory without temporal model** — vector search returns the most similar fact, not the most current. Every system that tried pure vector search for evolving facts failed on "what's the CURRENT state?" queries. (Zep, Mem0, multiple independent analyses)

2. **Auto-saved memory without human review** — Cursor's memory feature: 73% of memories auto-converted to temporary context, vanished within 24 hours. Produces stores full of hallucinations and duplicate facts. (Cursor forum, community analysis)

3. **Monolithic memory files at startup** — Cline power users found 6 verbose memory files consumed so much context the model's working window shrank. (Cline community)

4. **Over-engineering for current model limitations** — MemGPT's tool-based heartbeat architecture was abandoned by Letta when frontier models made it unnecessary. Design for model trajectory, not current limitations. (Letta v1 blog)

5. **Memory without staleness detection** — Documentation that doesn't know it's outdated is worse than no documentation. Provides confident wrong answers. (centminmod's memory-bank-synchronizer pattern exists to address this)

6. **More context ≠ better outcomes** — Simply enlarging context windows leads to "context rot" — degraded performance without deliberate context management. The "lost in the middle" problem causes >30% performance degradation when relevant info sits in the middle of a large context. (Factory.ai, RAGFlow 2025 review)

7. **Building vector-store memory that recreates RAG failure modes** — Teams spend months building custom systems that have the same decontextualization, retrieval timing, and staleness problems — but now it's their problem to debug. (Oracle dev blog, Composio analysis: "If Anthropic and OpenAI are still treating this as an active research problem, your internal team is not going to solve it.")

---

## Semantic Search at FileScience's Scale

### QMD Is Fine for Now

- QMD uses SQLite FTS5 (BM25) + vector embeddings + optional LLM reranking, all local
- At 362 docs (even growing to 500+), this is well within the "small collection" regime where exact/flat search gives 100% recall
- HNSW recall degradation starts at ~10K vectors — with section-level chunking, FileScience would need 1,000+ docs to approach this
- The real problem is **temporal, not volumetric** — QMD has no recency signal

### The Recency Gap

QMD's three search modes (`search` ~30ms, `vector_search` ~2s, `deep_search` ~10s) all rank by relevance, not recency. For "find the latest competitive-intel brief," this is the wrong ranking.

**Options (from semantic search research):**
1. **Post-process QMD results with date weighting** — extract date from filename, apply decay function, re-sort
2. **Pre-filter by date range** — restrict QMD search to documents from last N days
3. **Stable "latest" pointer files** — bypass search entirely for known recurring reports

### When to Upgrade (Not Now)

| Threshold | Action |
|-----------|--------|
| 500+ docs | Add frontmatter metadata to enable filtered search |
| 1,000+ docs | Consider LanceDB (4MB idle, native hybrid search, no index needed under 100K vectors) |
| 2,000+ docs | Consider category-based collection sharding |
| 10,000+ vectors | HNSW becomes necessary; tune `ef_search` parameter |

---

## Recommendations (Ordered by ROI)

### Fix Now (30 min)

1. Move `scorecard-template.md` from `durable/` to `.claude/reference/`
2. Fix CLAUDE.md Route K: `queue.md` → "add to Linear backlog"
3. Delete session dirs older than 7 days

### Add to Bizdev Plan as Phase 0 (2-4 hours)

4. Create `memory-bank/thoughts/shared/intelligence/` for recurring bizdev files
5. Add `series` frontmatter to recurring report templates
6. Implement "latest" pointer pattern: `memory-bank/durable/latest-{series}.md`
7. Document monthly rollup convention

### Shape/Plan Separately (Bigger Projects)

8. **Frontmatter lifecycle metadata on all new files** — `type: event|state`, `created`, `status`. Don't backfill 362 files; enforce going forward.
9. **QMD recency enhancement** — post-process results with date-weighted scoring, or contribute a date filter to QMD upstream
10. **Maintenance script** — weekly scan for orphaned notes, broken wikilinks, stale state notes, un-rolled-up event notes
11. **Mem0-style dedup check** — before any skill creates a new research file, vector-search existing files; UPDATE if similarity > 0.85
12. **Category-aware QMD collections** — namespace by document type (plans, briefs, research, intelligence, topics)

### Defer (Not Needed at Current Scale)

13. Knowledge graph / Graphiti integration — overkill under 1,000 docs
14. LanceDB migration — QMD + SQLite FTS5 is sufficient for years
15. Full bi-temporal model — "latest" pointers solve the practical problem
16. RAPTOR-style hierarchical indexing — two-level (summary + sections) is enough

---

## Key Sources

### Agent Memory Architectures
- [Zep/Graphiti — temporal knowledge graph (arxiv 2501.13956)](https://arxiv.org/abs/2501.13956)
- [Mem0 — production memory with CRUD pipeline (arxiv 2504.19413)](https://arxiv.org/abs/2504.19413)
- [SimpleMem — recursive consolidation, 30x token reduction (arxiv 2601.02553)](https://arxiv.org/html/2601.02553v1)
- [Letta v1 — lessons from MemGPT rearchitecture](https://www.letta.com/blog/letta-v1-agent)
- [Stop Using RAG for Agent Memory — Zep](https://blog.getzep.com/stop-using-rag-for-agent-memory/)

### Claude Code Community
- [Claude Diary — diary + reflect pattern](https://rlancemartin.github.io/2025/12/01/claude_diary/)
- [claude-mem — hook-based progressive memory](https://github.com/thedotmack/claude-mem)
- [mcp-memory-service — SQLite-vec + knowledge graph](https://github.com/doobidoo/mcp-memory-service)
- [basic-memory — dual-layer markdown + SQLite](https://github.com/basicmachines-co/basic-memory)

### Knowledge Management
- [Steph Ango — How I use Obsidian (fractal journaling)](https://stephango.com/vault)
- [obsidiantools — Python vault analytics](https://github.com/mfarragher/obsidiantools)
- [Andy Matuschak — Evergreen Notes](https://notes.andymatuschak.org/Evergreen_notes)
- [Mem0 ADD/UPDATE/DELETE taxonomy](https://arxiv.org/html/2504.19413v1)

### Semantic Search
- [HNSW at Scale — recall degradation analysis](https://towardsdatascience.com/hnsw-at-scale-why-your-rag-system-gets-worse-as-the-vector-database-grows/)
- [VersionRAG — 58% → 90% accuracy on version-sensitive queries](https://arxiv.org/abs/2510.08109)
- [QMD source — architecture details](https://github.com/tobi/qmd)
- [Hybrid search with temporal filtering](https://www.tigerdata.com/blog/hybrid-search-timescaledb-vector-keyword-temporal-filtering)

### Context Engineering
- [Anthropic — Effective Context Engineering for AI Agents](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents)
- [Factory.ai — The Context Window Problem](https://factory.ai/news/context-window-problem)
- [Google ADK — Artifact versioning](https://google.github.io/adk-docs/artifacts/)
- [ReMe — Memory Management Kit for Agents](https://github.com/agentscope-ai/ReMe)

### Competitive Tool Analysis
- [Cline Memory Bank docs](https://docs.cline.bot/features/memory-bank)
- [Windsurf Cascade Memories docs](https://docs.windsurf.com/windsurf/cascade/memories)
- [Cursor Memories forum thread](https://forum.cursor.com/t/0-51-memories-feature/98509)
- [Cursor rules empirical study (401 repos)](https://arxiv.org/html/2512.18925v3)
