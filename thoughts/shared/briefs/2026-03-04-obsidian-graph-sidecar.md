# Idea Brief: Obsidian Vault + Graph Sidecar MCP for Memory-Bank

**Date:** 2026-03-04
**Status:** Shaped → Planning

## Problem
Our 218+ file memory-bank has excellent semantic search (QMD) but zero structural navigability. Agents can't follow provenance chains ("what research led to this plan?") without multiple search rounds. Humans have no visual way to browse relationships. Only 23 wikilinks exist across 11 files — the link graph is nearly empty, making structural queries impossible.

## Constraints
- QMD has no graph layer and no plugin architecture — graph capability must be external
- MCP best practices favor focused sidecar servers over monolithic mega-servers
- 193/218 files already have YAML frontmatter; 52 relative links across 18 files need converting
- Agent skills write to `memory-bank/thoughts/shared/` — wikilink conventions must be adopted in skills
- No external service dependencies (no Docker, no Postgres, no running Obsidian desktop)
- Must work with existing markdown files on disk — no separate storage layer

## Options Considered

### Standalone Graph Sidecar MCP
Build a new MCP server in `tools/graph-memory/`. On startup, scan `memory-bank/`, parse wikilinks via obsidiantools into NetworkX DiGraph, expose 4-5 graph traversal tools. Separate process alongside QMD.
- Gains: Clean separation, no QMD coupling, follows MCP best practice, obsidiantools handles parsing edge cases, NetworkX gives BFS/shortest-path/clustering for free
- Costs: New server to configure and maintain, second process alongside QMD
- Complexity: Low (~200-300 lines pure Python)

### QMD Wrapper MCP
Build an MCP server that wraps QMD search results with graph expansion — single interface replacing direct QMD access.
- Gains: Single agent interface, could rerank by graph proximity
- Costs: Couples to QMD internals, adds latency, breaks if QMD changes, violates focused-server best practice
- Complexity: Medium

### Adopt Community MCP (obsidian-graph-memory or similar)
Use an existing community MCP server for graph queries over Obsidian vaults.
- Gains: Zero build effort
- Costs: Requires Obsidian desktop running (GUI dependency), 0-star proof-of-concept quality, can't scope to subdirectory
- Complexity: Low build, high operational

## Chosen Approach
**Standalone Graph Sidecar MCP** — simplest build, follows MCP best practices, zero external deps beyond obsidiantools + networkx, keeps QMD untouched. Community options aren't mature enough. QMD wrapper adds complexity for marginal gain.

Two orthogonal workstreams: (1) the MCP server itself, (2) Obsidian vault setup + wikilink convention adoption in agent skills. The vault setup is just config — no code.

## Key Context Discovered During Shaping
- MCP best practices explicitly call monolithic "mega-servers" an anti-pattern — focused sidecars are the endorsed pattern
- obsidian-graph-memory (ghanithan) validates the tool surface (7 tools) but requires running Obsidian desktop — non-starter
- claude-graph-memory and memory-graph both manage their own storage — can't index existing markdown
- Hybrid graph+vector outperforms pure vector by 3.4x on multi-entity queries (FalkorDB/Diffbot benchmark)
- Sparse linking (2-3 per doc) is enough for useful backlink traversal — no need for dense linking
- Agent wikilink reliability degrades over long sessions (instruction-following drift) — convention should be lightweight
- `obsidiantools` builds a full NetworkX DiGraph from raw markdown files with zero Obsidian dependency
- Deep research: `memory-bank/thoughts/shared/research/2026-03-04-deep-obsidian-vaults-for-agentic.md`

## Next Step
- [Plan] → `/create_plan memory-bank/thoughts/shared/briefs/2026-03-04-obsidian-graph-sidecar.md`
