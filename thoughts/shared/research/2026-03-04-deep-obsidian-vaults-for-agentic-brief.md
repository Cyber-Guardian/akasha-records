---
date: 2026-03-04T00:00:00-05:00
source_research: memory-bank/thoughts/shared/research/2026-03-04-deep-obsidian-vaults-for-agentic.md
last_generated: 2026-03-05T03:13:47.587844+00:00
---

# Research Brief: 2026-03-04-deep-obsidian-vaults-for-agentic

## TL;DR

Obsidian's primary value for our system is as a **human interface layer** (backlinks, Dataview, unlinked mentions) — not as an agent access mechanism, since QMD already handles semantic search better than any Obsidian MCP server. The strongest architectural opportunity is a **hybrid graph+vector approach**: add a wikilink graph layer (via `obsidiantools` → NetworkX) that complements QMD for multi-hop and provenance queries, where evidence shows 3.4x improvement over pure vector retrieval. Migration is low-friction — 193/218 files already have YAML frontmatter, and making memory-bank an Obsidian vault requires only adding a `.obsidian/` config directory. The recommended path is incremental: vault setup → wikilink conventions for new notes → optional graph layer extension to QMD.

## Key facts / constraints

None captured.

## Key recommendations

None captured.

## Open questions

- Should wikilink conventions be enforced via a PostToolUse hook (like ruff/ty), or just documented?
- What's the right granularity for wikilinks — link every topic note mention, or only when the link aids navigation?
- Could QMD's reranker incorporate graph-proximity as a signal, or should graph expansion happen as a separate post-search step?
- Should the nightly consolidation agent (ENG-2183) also maintain the graph index, or should it be rebuilt on demand?
- Is there value in adopting basic-memory's typed relation syntax (`relation_type [[Entity]]`) for richer graph edges, or are untyped wikilinks sufficient?

## Links

- Source research: `memory-bank/thoughts/shared/research/2026-03-04-deep-obsidian-vaults-for-agentic.md`
