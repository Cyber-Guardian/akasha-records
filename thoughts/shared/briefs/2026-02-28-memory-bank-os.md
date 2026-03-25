# Idea Brief: Memory-Bank OS (Obsidian + Evergreen Notes)

**Date:** 2026-02-28
**Status:** Shaped → Parked

## Problem
The memory-bank is 158 flat markdown files organized by type (briefs/plans/research/decisions) and date. Connections between documents are manual relative-path links, sparse, and invisible without reading `current_work.md`. A human can't see the shape of what Claude has been thinking about. A fresh Claude session has to reconstruct relationships from bullet points every time. Plans are dead documents — they don't link to the brief that spawned them, the research that informed them, or the decisions that constrained them in a navigable way.

## Constraints
- Memory-bank serves two readers: Claude (follows links from known entry points) and humans (need visual overview + drill-down)
- 158 existing files can't all be migrated at once — must be additive
- CLAUDE.md startup reads reference specific file paths — backward compatibility needed
- Skills (shape, create_plan, etc.) write to specific paths (`briefs/YYYY-MM-DD-topic.md`) — convention must evolve, not break
- Must stay local-first (files on disk in the repo) — no cloud sync dependency
- Claude needs clear conventions for reading, writing, and linking

## Options Considered

### Evergreen-First (Matuschak)
Reorganize everything by concept, not lifecycle. "Valkey Queue Processor" is one evolving note, not 6 dated files. Notes are atomic, concept-titled, densely linked. Lifecycle is metadata, not folder structure.
- Gains: Notes compound over time. Claude finds everything about a topic in one place. Graph shows conceptual clusters.
- Costs: Major migration of 158 files. Breaks current skill conventions. Plans/briefs lose distinct identity.
- Complexity: High

### MOC-Layered (LYT / Nick Milo)
Keep current file structure, add Maps of Content as navigation hubs. MOCs curate links to related notes. Wikilinks everywhere. The macro plan IS a MOC.
- Gains: Minimal migration. Graph view works immediately. Additive to current system. Skills keep writing to existing paths.
- Costs: Two layers of organization (folders + MOCs) could drift. MOCs need maintenance.
- Complexity: Low-Medium

### Hybrid Evergreen + MOC
New notes are evergreen (concept-oriented, atomic, wikilinked). Existing dated files stay but get wikilinked into MOCs. Lightweight rules govern when to create new vs update existing.
- Gains: Best of both. No big-bang migration. Graph grows organically.
- Costs: Two conventions coexist — could be confusing.
- Complexity: Medium

## Chosen Approach
**MOC-Layered first, evolve toward Hybrid** — add Obsidian vault config + MOCs + wikilinks on top of existing structure. Write new notes following evergreen principles (atomic, concept-oriented). Let the dated archive be the historical record, MOC layer be the living navigation. Over time, important concepts consolidate from many dated notes into single evergreen notes.

## Key Context Discovered During Shaping

### Existing state
- 158 markdown files across `memory-bank/`
- Only 19 `[[wikilink]]` usages across 8 files — almost entirely relative-path links
- Folder structure: `durable/` (00-core, 01-active, 02-architecture) + `thoughts/shared/` (briefs, plans, research, decisions, handoffs, reference)
- CLAUDE.md startup reads: `project_brief.md`, `product_context.md`, `current_work.md`, `next_up.md`, `blockers.md`

### Note-taking methodology research
- Evergreen notes (Andy Matuschak): atomic, concept-oriented, densely linked, evolve over time
- LYT/MOCs (Nick Milo): Maps of Content as navigation hubs, hierarchical but flexible
- Zettelkasten: literature notes vs permanent notes — maps to "research" vs "decisions/briefs"
- Key insight: lifecycle type (brief/plan/research) can be metadata or frontmatter, not folder structure

### AI + Obsidian ecosystem
- ClawVault: structured memory for AI agents, markdown-native, graph-aware wikilinks
- agent-recall: SQLite-backed knowledge graph with MCP server, extracted from real multi-agent system
- obsidian-graph-memory MCP: exposes vault graph structure as queryable tools for AI agents
- Obsidian Vault MCP servers: AI autonomously searches/retrieves context from vault

### Implementation considerations
- Obsidian vault = just a folder with `.obsidian/` config dir — zero overhead to set up
- Scope to `memory-bank/` only (not whole repo)
- Obsidian MCP server could eventually replace hardcoded CLAUDE.md startup reads — Claude queries graph for relevant MOCs instead
- Graph view gives humans the visual overview they need without reading all 158 files

## Relationship to Other Work
- **Macro Agent Strategy** (2026-02-28) — the macro plan IS a MOC; this brief provides the infrastructure for that
- **Visualization in Shaping & Planning** (2026-02-27) — visual review of agent thinking; Obsidian graph is the meta-level version of this
- **Complexity-as-Moat** (2026-02-28) — well-structured knowledge compounds; agent-navigable memory-bank is part of the moat

## Next Step
- [Parked] — capture as reference; implement when ready to set up Obsidian vault and define the note-taking guidelines
