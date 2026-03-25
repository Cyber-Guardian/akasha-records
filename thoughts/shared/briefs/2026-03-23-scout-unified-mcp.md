---
type: event
created: 2026-03-23
status: active
---
# Idea Brief: Scout as Unified Knowledge Base MCP

**Date:** 2026-03-23
**Status:** Shaped -> Planning

## Problem
Knowledge base access is fragmented across three MCP servers (QMD, graph-memory, scout) with overlapping functionality. As memory-bank extracts into akasha-records and the company scales beyond one person generating artifacts, we need a single MCP that serves all tiers — fast search, graph traversal, and agent exploration — for both local Claude Code sessions and remote agents. The current stack won't scale: QMD is a Node.js binary that can't be embedded in Python, graph-memory duplicates scout's GraphAdapter, and none of them are built for 10k-100k+ documents.

## Constraints
- QMD is Node.js (node-llama-cpp + 3 GGUF models, ~2GB) — not embeddable as a Python library
- Scout already has a solid exploration engine (start/explore/reflect/curate) and four corpus adapters (QMD via subprocess, graph, codebase, web)
- Scout's GraphAdapter duplicates graph-memory's BFS logic (both use obsidiantools + NetworkX)
- Must scale to company-wide usage: 10k-100k+ documents, not just the current ~500
- At scale, scout runs as a service, not a single-file sync to Lambda
- Sub-100ms latency required for fast search tier (keyword + vector)
- Must serve local Claude Code (stdio) and remote agents (HTTP)

## Options Considered

### SQLite-Native Search (sqlite-vec + FTS5 + Model2Vec)
Build search layer using raw SQLite extensions. Single `.db` index file. Proven at 16k files / 49k chunks (23ms latency) in an Obsidian vault benchmark.
- Gains: Single file, minimal dependencies, full control, proven blueprint
- Costs: sqlite-vec is pre-v1 (no ANN/HNSW yet — brute-force only). Hits a wall at ~50-100k chunks. Manual RRF plumbing.
- Complexity: Medium

### LanceDB-Powered Search (Tantivy FTS + HNSW ANN)
Use LanceDB as embedded search layer. Built-in hybrid search with RRF, Tantivy FTS, HNSW vector index. Lance directory format (not SQLite).
- Gains: First-class hybrid search out of the box. HNSW ANN scales to millions of vectors. Less plumbing to maintain. Production-grade at company scale.
- Costs: Lance directory format (not single `.db` file). Tantivy FTS marked "legacy" by LanceDB team. New storage format diverges from existing SQLite patterns.
- Complexity: Medium

### QMD as Daemon Behind Scout (proxy pattern)
Keep QMD running as HTTP daemon. Scout proxies fast search calls to it. One MCP exposed, QMD is implementation detail.
- Gains: Zero search engine code. QMD's full sophistication available.
- Costs: Two processes. Node.js dependency stays. 2GB GGUF models in memory. Can't deploy without Node.js. Complex failure modes.
- Complexity: Low

### txtai All-in-One
Use txtai framework with sqlite backend. Hybrid search + local embeddings in one library.
- Gains: Single library, batteries included, well-documented.
- Costs: Heaviest dependency footprint (PyTorch/ONNX, Faiss). Framework-on-framework risk with scout's engine. Not single-file by default.
- Complexity: Medium

## Chosen Approach
**LanceDB-Powered Search** — scales to company-wide usage (HNSW ANN for 100k+ docs), built-in hybrid search with RRF eliminates manual plumbing, and the Lance directory format is a non-issue when scout runs as a service. At company scale, we're not syncing a `.db` file to Lambda — scout IS the service.

## Key Context Discovered During Shaping
- QMD runs 3 local GGUF models (~2GB) for query expansion, reranking, and embedding — overkill for markdown document search where BM25 + vector + RRF is sufficient
- An Obsidian vault benchmark (16,894 files, 49,746 chunks) achieves 23ms end-to-end with sqlite-vec + FTS5 + Model2Vec + RRF — proving the general approach works at scale
- sqlite-vec has no ANN index yet (brute-force only, tracked as pre-v1 milestone) — disqualifies it for company-scale growth
- Scout's GraphAdapter already duplicates all of graph-memory's functionality — graph-memory MCP can be fully absorbed
- Scout already has persistence (`ScoutRegistry`), state machine (`ExplorationState`), and corpus adapter ABC — the search engine slots in cleanly
- Model2Vec / static embeddings give ~1ms query embedding latency (500x faster than full transformers)
- `markdown-vault-mcp` is an existing open-source MCP doing FTS5 + FastEmbed + RRF over markdown vaults — reference implementation

## What Scout Becomes
Three tiers, one MCP:
1. **Fast search tools** — `search` (keyword/BM25), `vector_search` (semantic), `hybrid_search` (fused) — sub-100ms
2. **Direct access** — `get` (file by path), `list` (browse), `graph_neighbors` / `graph_hubs` (structure)
3. **Agent exploration** — `start` / `explore` / `reflect` / `curate` (existing engine, unchanged)

Replaces: QMD (removed), graph-memory (absorbed), current scout (extended)

## Related Artifacts
- [[2026-03-20-akasha-records|Akasha Records Repo Split]] — the repo extraction this serves
- [[2026-03-20-personal-life-ontology-agent-platform|Akasha Interface Substrate]] — parent vision
- [[2026-03-12-memory-system-lifecycle|Memory System Lifecycle]]

## Next Step
- [Plan] -> `/create_plan memory-bank/thoughts/shared/briefs/2026-03-23-scout-unified-mcp.md`
