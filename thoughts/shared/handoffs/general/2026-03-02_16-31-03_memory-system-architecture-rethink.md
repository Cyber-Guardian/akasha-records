---
date: 2026-03-02T16:31:03-0500
author: claude-opus
git_commit: 342ee5b
branch: main
repository: filescience
topic: "Memory System Architecture Rethink — Shaping Session Handoff"
tags: [handoff, shaping, memory-system, architecture, sota-research]
status: in_progress
last_updated: 2026-03-02
last_updated_by: claude-opus
---

# Handoff: Memory System Architecture Rethink (Shaping)

## Task(s)

**Status: IN PROGRESS — shaping, approaching decision gate**

The user asked to "consider all thoughts and linear — lets start thinking about memory system improvements." This expanded into a full architecture rethink of the memory-bank system, not incremental improvements. The shaping session progressed through:

1. **Problem framing** — identified 3 structural gaps (discovery, compounding, lifecycle) — ALIGNED
2. **Initial approach brainstorm** — 4 approaches proposed (Evergreen, Tiered Cache, Knowledge Graph MCP, Hybrid Active+Manifest) — user liked the Hybrid+Graduation approach (D)
3. **SOTA research** — 3 parallel research agents produced comprehensive findings on production memory systems, academic papers, community patterns — COMPLETE
4. **SOTA-informed approach revision** — 4 revised approaches (Compound Memory, Indexed Vault, Memory MCP, Compound+Graduation) — user aligned on Approach D (Compound Memory + Graduation Path)
5. **Structural deep-dive** — analyzed current structure problems, proposed 5 design questions (hot tier, warm tier, cold tier, .claude/ fit, naming) with options for each — PRESENTED, AWAITING USER REACTION

**The session ran out of context before the user could react to the structural proposals.** The next session needs to pick up at the structural design questions and drive to a decision gate.

## Critical References

1. **SOTA Research Output (agent transcripts)** — the three research agents produced ~15,000 words of findings. These are NOT persisted as files — they existed only in this session's context. The key findings are synthesized below in this handoff.
2. **Existing parked brief:** `memory-bank/thoughts/shared/briefs/2026-02-28-memory-bank-os.md` — Memory-Bank OS (Obsidian + MOC). Parked. Partially superseded by this rethink but contains valid insights.
3. **Existing seed:** `memory-bank/thoughts/shared/briefs/2026-02-28-hive-mind-agent-communication.md` — Hive Mind inter-agent communication. Partially overlaps (event log = medium-term memory layer).
4. **Existing seed:** `memory-bank/thoughts/shared/briefs/2026-03-01-commit-lineage-indexing.md` — bridges "how did code get here" gap.

## SOTA Research Synthesis (CRITICAL — this is the main value of the session)

Three research agents ran in parallel. Below is the synthesized findings.

### Universal Pattern: Everyone Tiers

Every production memory system has converged on tiered memory:

| System | Hot (always loaded) | Warm (on-demand) | Cold (archive) | Consolidation |
|--------|-------------------|-------------------|-----------------|---------------|
| Letta/MemGPT | Core memory blocks (agent self-edits) | Recall (conversation search) | Archival (vector DB) | Agent self-manages |
| Codified Context (arXiv:2602.20478) | ~660 lines, every session | ~9,300 lines, per-agent-type | ~16,250 lines, MCP-served | Manual |
| Mastra | Observation prefix (stable for caching) | — | Full transcripts | Observer (30K trigger) + Reflector (40K trigger) |
| memory-mcp (yuvalsuede) | CLAUDE.md (~150 lines, budget-allocated) | .memory/state.json (MCP search) | — | Jaccard dedup + LLM consolidation every 10 extractions |
| GitHub Copilot | Citation-validated memory pool | — | — | JIT citation validation (self-healing) |
| agent-recall | LLM briefings (SessionStart) | Entity/slot/observation queries | Bitemporal archive | Adaptive cache invalidation |
| A-MEM (NeurIPS 2025) | — | Zettelkasten notes (evolving) | — | Memory evolution: existing notes auto-update |
| Ours (current) | 5 durable files (~436 lines) | — | 242 dated files (grep only) | Manual "distill" (inconsistent) |

### Six Innovations Worth Stealing

1. **GitHub Copilot: Citation-validated memory.** Each memory entry links to specific code locations. On retrieval, citations validated against current codebase. Stale memories self-heal. 7% increase in PR merge rates.

2. **Mastra: Observer/Reflector pattern.** No vector DB needed. Observer watches conversation, produces dated/priority-tagged observations (triggers at 30K tokens). Reflector restructures when observations grow too large (40K tokens). 94.87% on LongMemEval. Stable observation prefix enables prompt caching.

3. **A-MEM (NeurIPS 2025): Memory evolution.** Zettelkasten for agents — when new memories arrive, related existing memories auto-update their descriptions/tags. Knowledge compounds, not accumulates. 2x multi-hop reasoning, 85-93% fewer tokens.

4. **agent-recall: Scope chains + LLM briefings.** Hierarchical scope isolation (global → org → project) for 30+ concurrent agents. LLM generates structured briefings from hundreds of raw facts — not raw data dumps. Bitemporal storage.

5. **memory-mcp: Budget allocation with decay.** Fixed line budgets per CLAUDE.md section. Memories ranked by `confidence × accessCount`. Permanent vs 7-day vs 30-day half-life by memory type. ~$0.001/extraction.

6. **basic-memory: Markdown files + SQLite graph.** Files ARE the source of truth (human-readable, git-trackable). SQLite maintains a knowledge graph derived from wikilinks. Obsidian-compatible. Bidirectional human+LLM editing.

### Key Research Findings

- **CooperBench (arXiv:2601.13295):** Two coordinating agents achieve HALF the success rate of one. 42% expectation failures, 32% commitment failures, 26% communication failures. Agents should NOT coordinate — orchestrator coordinates.
- **"Anatomy of Agentic Memory" (arXiv:2602.19320, Feb 2026):** Many benchmarks saturated by 128K+ context. Complex graph schemas fail with weak models (30%+ format errors). Latency matters — MemoryOS 32+ seconds, AMem 15 hours to index.
- **Quality gate math (CEK framework):** Each ungated phase ~80% accuracy. 5 ungated phases = 0.8^5 = 33%. Validates shape→plan→implement lifecycle.
- **Codified Context (arXiv:2602.20478):** 283 real sessions, 108K-line codebase. Documentation was 24.2% of codebase. Critical failure: specification staleness — stale specs cause syntactically correct code conflicting with recent refactors.
- **Markdown vs DB tradeoff:** Markdown works up to ~5MB / ~1,000 files, then needs BM25. Our 242 files are well within this range.
- **Hybrid is winning:** basic-memory, ClawVault, memory-mcp all use files as source of truth + derived index for discovery.

### Production Systems Surveyed

**Coding agents:**
- Cursor: No native memory. Community memory-bank pattern (Cline-originated). `.cursor/rules/` for instructions.
- Windsurf/Cascade: Auto-generated workspace-scoped memories. Opaque retrieval algorithm. Background codebase analysis (~48h).
- GitHub Copilot: Citation-validated shared memory pool across 3 agents. Self-healing. 7% PR merge rate increase.
- Augment Code: Semantic Context Engine (MCP-exposed Feb 2026). Commit-aware "Context Lineage." Handles 400K+ file codebases.
- Cline: Original memory-bank pattern — 6 structured markdown files, mandatory full read every session.
- Aider: No native memory. File-context driven, human-controlled.

**Multi-agent frameworks:**
- CrewAI: 4-tier (short/long/entity/contextual) + Mem0 integration. Shared within crew.
- LangGraph/LangMem: Semantic/Episodic/Procedural types. Hot-path and background extraction. Namespace hierarchy.
- AutoGen: No native persistence. External integration required.
- OpenAI Agents SDK: Session-centric with pluggable backends (SQLite, Redis, PostgreSQL). Compaction wrapper.
- Claude Agent SDK: Session-isolated. Memory tool (beta). Subagent auto-memory. MEMORY.md 200-line limit.

**Knowledge graph systems:**
- Zep/Graphiti: Bi-temporal knowledge graph (most sophisticated). Episode → Semantic → Community subgraphs. Tri-modal search (cosine + BM25 + BFS) + RRF reranking. Neo4j backend.
- Mem0: Hybrid vector + graph. ADD/UPDATE/DELETE/NOOP decision process. 24+ storage backends.
- Letta/MemGPT: Agent-as-OS. Core memory blocks (self-editable in-context) + Recall + Archival. `memory_rethink` for wholesale rewrite.
- Cognee: ECL pipeline (Extract-Cognify-Load) + Memify post-processing (prunes stale, strengthens frequent, adds derived facts).
- A-MEM: Zettelkasten notes with 7 components. Dynamic linking via cosine similarity + LLM-reasoned connections. Memory evolution.
- agent-recall: SQLite knowledge graph. Scope chains. Bitemporal. LLM briefings. Obsidian export.
- basic-memory: Markdown + SQLite index. Wikilink graph. Obsidian-compatible.

**Academic surveys/papers (2025-2026):**
- arXiv:2512.13564 — "Memory in the Age of AI Agents" — taxonomy: factual/experiential/working
- arXiv:2602.19320 — "Anatomy of Agentic Memory" — 4 memory structures, critical empirical failures
- arXiv:2602.06052 — "Rethinking Memory Mechanisms" — 60-author survey, three-dimension framework
- arXiv:2602.20478 — "Codified Context Infrastructure" — 3-tier proven over 283 sessions
- arXiv:2507.03724 — "MemOS" — memory as first-class OS resource, MemCube abstraction
- arXiv:2502.12110 — "A-MEM" (NeurIPS 2025) — Zettelkasten for agents, 2x multi-hop reasoning
- arXiv:2501.13956 — "Zep: Temporal Knowledge Graph" — bi-temporal formalization
- ICLR 2026 MemAgents Workshop (April 2026) — first major venue dedicated to agent memory

## Approach Alignment (where we landed)

### Chosen Direction: Approach D — Compound Memory + Graduation Path

Three-tier system (hot/warm/cold) with concept-oriented active notes and automated consolidation, phased to work today (file-based) and graduate to MCP-based discovery later.

**Phase 1 (now):** Three-tier structure + active topic notes + skill-integrated consolidation + budget enforcement on durable. No new infrastructure.

**Phase 2 (with nightly agents / ENG-2183):** Add automated hygiene — staleness detection, budget enforcement, relationship extraction into lightweight index (YAML or SQLite).

**Phase 3 (with swarm harness):** Promote index into MCP server. Add LLM briefings, scope chains, semantic search. Replace hardcoded startup reads with `memory_briefing(context)`.

The user said "I like D as well, but I think we should do more consideration to memory-bank's structure."

## Structural Design (presented, awaiting reaction)

### Current Problems Identified

1. `current_work.md` at 249 lines (target 20-60) — it's 4 things in a trenchcoat: active work log, completed archive, resource reference, artifact index
2. `next_up.md` at 114 lines (target 10-40) — still has completed items
3. Numbered subdirs (`00-core`, `01-active`, `02-architecture`) are vestigial — CLAUDE.md explicitly names files
4. No separation between project identity and ephemeral state
5. Lifecycle-based archive folders optimize for writing not reading — "find everything about throttling" requires searching 6 folders
6. No middle tier — durable → archive is a cliff
7. `.claude/` directory IS procedural memory but not part of the memory-bank architecture explicitly

### Five Design Questions Posed (user has NOT yet responded)

**Q1: Hot Tier — What gets loaded every session?**
- Option A: Keep 5 files, enforce budgets
- Option B: Restructure into semantic sections (identity.md, state.md, focus.md, queue.md, blockers.md) — **RECOMMENDED**
- Option C: Single startup file with budget-allocated sections
- Rationale for B: Cleaner semantic boundaries, self-documenting file names, ~150-160 lines total

**Q2: Warm Tier — How to organize active notes?**
- Option: Flat list (throttling.md, helm-orchestration.md, etc.) — **RECOMMENDED**
- Alternative: Hierarchical (backend/, agent/, infra/ subdirectories)
- Rationale for flat: No classification decisions, simpler, let hierarchy emerge later if needed
- Expected ~15-20 active notes, ~30-80 lines each
- Proposed format: YAML frontmatter (topic, status, touched) + sections (Current State, Key Decisions, Open Questions, Artifacts)

**Q3: Cold Tier — Change archive structure?**
- Option A: Keep lifecycle folders (briefs/plans/research), add frontmatter — **RECOMMENDED**
- Option B: Reorganize by topic subdirectories
- Option C: Flatten entirely
- Rationale for A: Skills already write to lifecycle folders, frontmatter + wikilinks provide topic dimension, no migration needed

**Q4: How does `.claude/` fit?**
- Recommendation: Keep separate. `.claude/ = procedural memory`, `memory-bank/ = factual + working + experiential memory`
- `.claude/reference/` stays in `.claude/` because it loads via Claude Code's native platform mechanisms

**Q5: Naming and path conventions**
- Active notes: `active/{topic-slug}.md` (kebab-case, no dates)
- Archive files: Keep `YYYY-MM-DD-{topic}.md` (skills already write this way)
- Frontmatter: YAML with topic, type, status, related, touched
- Wikilinks: `[[YYYY-MM-DD-topic]]` for archive, `[[active/topic]]` for archive→active links

### Proposed Full Structure

```
memory-bank/
  durable/                    # HOT — always loaded (~150-200 lines total)
    identity.md                 # What this project is + product context (stable)
    state.md                    # What's deployed/true now (infra, components)
    focus.md                    # Active work: 3-5 items max
    queue.md                    # Ordered backlog: 5-10 items
    blockers.md                 # What's stuck

  active/                     # WARM — concept notes, loaded on demand (~15-20 notes)
    throttling.md
    helm-orchestration.md
    discover-service.md
    ci-pipeline.md
    observability.md
    agent-discipline.md
    ...

  thoughts/                   # COLD — raw dated archive (unchanged structure)
    shared/
      briefs/                   # Idea briefs (28 files)
      plans/                    # Implementation plans (45 files)
      research/                 # Research docs (79 files)
      decisions/                # Decision records (4 files)
      handoffs/                 # Session handoffs
      reference/                # Stable reference docs (8 files)

.claude/                      # PROCEDURAL — how the system behaves
  rules/                        # Path-scoped conventions
  skills/                       # Workflow procedures
  agents/                       # Agent configurations
  reference/                    # Stable reference docs
  hooks/                        # Automated reactions
```

**Cognitive type mapping:**
| Cognitive Type | Location | Purpose |
|---------------|----------|---------|
| Working memory | `durable/` | What's active right now |
| Factual memory | `durable/identity.md` + `active/` | What's true about the project and topics |
| Experiential memory | `thoughts/` | What happened, in chronological order |
| Procedural memory | `.claude/` | How to behave, what tools to use |

## Learnings

1. **Every production memory system has converged on tiers.** The debate is over tier count, boundaries, and automation — not whether to tier.
2. **Hybrid "files as source of truth + derived index" is the winning pattern.** basic-memory, ClawVault, memory-mcp all validate this. Files stay git-trackable and human-readable; the index enables discovery.
3. **Automated consolidation is the key differentiator between good and great systems.** Mastra Observer/Reflector, A-MEM memory evolution, memory-mcp's Jaccard+LLM consolidation — all automate what we do manually with "distill."
4. **GitHub Copilot's citation-validated memory is the most innovative pattern** — memories that link to specific code locations and self-heal when code changes. This is a Phase 3 idea for us.
5. **CooperBench killed agent-to-agent coordination** — two coordinating agents achieve half the success of one. Orchestrator coordinates; agents work independently.
6. **Markdown works at our scale** — 242 files, well under the ~1,000 file / ~5MB threshold where you need BM25 or vector search.
7. **The ICLR 2026 MemAgents workshop (April 2026)** signals agent memory is now a first-class research domain.

## Artifacts

- This handoff: `memory-bank/thoughts/shared/handoffs/general/2026-03-02_16-31-03_memory-system-architecture-rethink.md`
- Existing brief (partially superseded): `memory-bank/thoughts/shared/briefs/2026-02-28-memory-bank-os.md`
- Existing brief (overlapping): `memory-bank/thoughts/shared/briefs/2026-02-28-hive-mind-agent-communication.md`
- Existing brief (input): `memory-bank/thoughts/shared/briefs/2026-03-01-commit-lineage-indexing.md`
- Existing brief (input): `memory-bank/thoughts/shared/briefs/2026-02-28-macro-agent-strategy.md`
- Existing brief (input): `memory-bank/thoughts/shared/briefs/2026-02-28-agent-swarm-harness.md`
- Existing brief (input): `memory-bank/thoughts/shared/briefs/2026-02-28-complexity-as-moat.md`
- Existing brief (input): `memory-bank/thoughts/shared/briefs/2026-02-28-judge-taste-agents.md`
- Existing brief (input): `memory-bank/thoughts/shared/briefs/2026-03-02-harness-self-improvement.md`
- Memory rules reference: `.claude/reference/memory-rules.md`
- CLAUDE.md routing table: `CLAUDE.md` (section 2)

**No brief written yet** — shaping is not complete. The brief should be written after the user responds to the structural design questions and the decision gate is reached.

## Action Items & Next Steps

1. **Resume shaping at Q1-Q5 structural design questions.** The user said "we should do more consideration to memory-bank's structure" — they haven't yet reacted to the 5 design questions posed above. Start by presenting them again concisely and getting alignment.

2. **After structural alignment, present decision gate:**
   - Plan → `/create_plan` for the chosen structure
   - Park → Linear backlog
   - Implement directly → if simple enough
   - Kill → unlikely given alignment

3. **Write the idea brief** once the decision gate is reached. File: `memory-bank/thoughts/shared/briefs/2026-03-02-memory-system-architecture-v2.md`

4. **If proceeding to plan, key planning considerations:**
   - Phase 1 migration: how to restructure durable/ without breaking CLAUDE.md startup reads
   - Phase 1 skill updates: every skill needs an "update active note" step
   - Phase 1 initial active notes: which ~15-20 topics to seed from the 242 archival files
   - Phase 2: nightly agent design for automated hygiene
   - Phase 3: MCP server design (model after agent-recall or basic-memory)
   - CLAUDE.md routing table update for new file paths
   - `.claude/reference/memory-rules.md` rewrite for new architecture

5. **Linear issues to consider:**
   - ENG-2183 (nightly scheduled agents) — Phase 2 depends on this
   - ENG-2225 (harness self-improvement) — staleness detection is part of Phase 2
   - ENG-2227 (commit lineage indexing) — could become a new content type in the warm tier

## Other Notes

- The three SOTA research agent outputs were NOT saved to files — they only existed in conversation context. This handoff captures the synthesized findings. If deeper detail is needed on any specific system (e.g., Zep's bi-temporal model, Mastra's Observer/Reflector pattern, agent-recall's scope chains), the next session should run targeted web research.
- The user's reactions throughout the session were quick and directional: "scope is rethinking the arch" (not incremental), "everything you listed is a pain" (all 3 gaps equally important), "I like D" (compound + graduation), "we should do more consideration to memory-bank's structure" (deeper structural thinking needed). They trust the research and want to get the structure right before committing.
- Related Linear issues found during research: ENG-2134 (agent memory strategy, done), ENG-2128 (execution discipline, in progress), ENG-2152 (plan as handoff medium, done), ENG-2227 (commit lineage, investigation), ENG-2225 (harness self-improvement, investigation), ENG-2183 (nightly agents, investigation).
- The `.claude/reference/memory-rules.md` file has line targets that are wildly violated (current_work.md 249 vs target 20-60, next_up.md 114 vs target 10-40). This validates the need for structural change, not just discipline.
