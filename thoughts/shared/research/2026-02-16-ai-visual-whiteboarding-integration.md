---
date: 2026-02-16
author: Claude Opus 4.6
topic: "AI + Visual Whiteboarding Integration — Landscape, Tools, and Approaches"
tags: [research, visualize, tldraw, excalidraw, diagrams, ai-canvas, mcp]
status: complete
---

# AI + Visual Whiteboarding Integration Research

## Context

Research to inform the `/visualize` skill design — how can Claude most effectively mesh with visual whiteboarding tools to enable bidirectional AI-human visual communication?

Prior art: [[2026-02-16_14-27-48_visualize-skill-design|2026-02-16_14-27-48_visualize-skill-design.md]]

---

## Executive Summary

**tldraw is the AI-canvas leader** with Agent Starter Kit, Make Real, and "computer" features — but uses a proprietary license (paid for production). **Excalidraw is MIT-licensed** with growing MCP support and an official Anthropic MCP server. The most effective approach is a **hybrid**: AI writes token-efficient intermediate text (Mermaid/D2) → auto-layout engine renders → optional visual polish via MCP. Bidirectional iteration (human edits → AI understands) remains a research frontier with no mature off-the-shelf solution.

---

## 1. tldraw vs Excalidraw for AI Integration

### tldraw: AI-First Platform

**AI Features:**
- **Make Real**: Sketch-to-working-HTML via GPT-4V/Claude/Gemini. Canvas becomes iterative design space — draw annotations, click "Make Real", refines output.
- **Agent Starter Kit**: Comprehensive toolkit for AI agents to manipulate canvas. Dual perception system (visual screenshots + structured shape data). Real-time streaming, multi-turn conversations.
- **"computer"**: Experimental natural language workflow platform with Gemini 2.0. Flow-based execution where output → input chains.

**SDK:**
- Full programmatic access via `Editor` instance (create/update/delete shapes, batch ops, viewport control)
- `meta` property on shapes accepts any JSON (equivalent to Excalidraw's `customData`)
- `.tldr` format is plain JSON but verbose (82% larger than minified)

**License:** Proprietary. Requires paid license for production use. Watermarks without license. Pricing not public (sales contact required).

### Excalidraw: Open Source Standard

**AI Features:**
- No built-in AI features
- **Official MCP Server** (Excalidraw + Anthropic, early 2026) — available in Claude Connectors, Claude Code
- Community MCP servers (yctimlin: 26 tools including `describe_scene`, `get_canvas_screenshot`)

**SDK:**
- Beta programmatic API (less mature than tldraw)
- `customData` property on elements for semantic metadata
- `.excalidraw` format is JSON, similar verbosity to tldraw

**License:** MIT. Free for all uses. No watermarks. 116k GitHub stars (2.5x tldraw's 45k).

### Comparison Matrix

| Aspect | tldraw | Excalidraw |
|--------|--------|------------|
| AI features | Agent SDK, Make Real, computer | None built-in (MCP only) |
| License | Proprietary (paid production) | MIT (free) |
| Metadata | `meta` property | `customData` property |
| SDK maturity | Production-ready | Beta |
| MCP servers | 3+ community | Official + community |
| Mermaid converter | None exists | Official (flowcharts only) |
| Stars | 45k | 116k |

### Verdict

**For our use case (Claude Code skill):** Excalidraw is the better fit. MIT license, official Anthropic MCP integration, existing Mermaid converter, and `customData` for semantic metadata. tldraw's AI features are impressive but the proprietary license and lack of Mermaid converter are blockers.

---

## 2. Generation Approaches — Token Efficiency

Text-based formats are **10-50x more token-efficient** than JSON visual formats.

### Rankings (most → least efficient)

| Format | Tokens (typical) | LLM Quality | Auto-Layout | Notes |
|--------|-----------------|-------------|-------------|-------|
| **Mermaid** | ~100-500 | Excellent | Yes (Dagre/ELK) | Best LLM support, Markdown-native |
| **D2** | ~100-500 | Good | Yes (ELK default) | Better aesthetics, simpler syntax |
| **GraphViz DOT** | ~100-500 | Good | Yes (dot/neato) | Industry standard for graphs |
| **Custom YAML DSL** | ~300-1000 | Variable | Needs converter | 30-40% more efficient than JSON |
| **Excalidraw/tldraw JSON** | ~1000-5000 | Poor | No (manual coords) | 10-50x larger than Mermaid |
| **SVG** | ~2000-20000+ | Poor | No | Prohibitively expensive |

### Cost Impact

- Simple flowchart in Mermaid: ~200 tokens = $0.001 (GPT-4o)
- Same flowchart in Excalidraw JSON: ~2000 tokens = $0.01 (10x cost)
- With prompt caching: 90% cost reduction on cache hits

### Recommendation

**Mermaid as primary intermediate format.** Best LLM support (Claude generates it naturally), markdown integration, GitHub-native rendering, official Excalidraw converter.

**D2 as secondary option** for architecture diagrams — uses ELK by default (better nested layout), cleaner aesthetics, multi-format export.

---

## 3. Layout Algorithms

### Algorithm Selection Guide

| Algorithm | Speed | Quality | Best For | Node Limit |
|-----------|-------|---------|----------|------------|
| **Dagre** | Fast | Good | Flowcharts, trees, DAGs | <1000 |
| **ELK** | Slow | Excellent | Nested architectures (C4 L3/L4) | <500 |
| **Cola.js** | Medium | Excellent | Constraint-based positioning | <100 |
| **D3-Force** | Medium | Good | Organic network graphs | <500 |

### By Diagram Type

| Diagram Type | Best Algorithm | Rationale |
|--------------|---------------|-----------|
| Flowcharts | Dagre | Fast hierarchical, process flow emphasis |
| C4 Architecture | ELK | Nested components, clear boundaries |
| Sequence Diagrams | Custom Timeline | Vertical time-based (Mermaid built-in) |
| State Machines | Dagre | Directed graph with clear transitions |
| ER Diagrams | Dagre or dot | Hierarchical with relationships |
| Component Diagrams | ELK | Nested subsystems |

### Mermaid's Internal Layout

- Default: Dagre for flowcharts, state diagrams
- Optional: ELK for complex hierarchies (configurable)
- Special: Tidy-Tree for mindmaps, custom timeline for sequences

---

## 4. Bidirectional AI-Human Iteration

### Current State: Mostly One-Way

**Forward (AI → Human) works well:**
- AI → Mermaid/D2 → Render → Human reviews (mature, reliable)
- AI → JSON → Excalidraw → Human edits (verbose but functional)

**Reverse (Human → AI) is a research frontier:**
- No mature off-the-shelf solution
- Commercial tools (Visual Paradigm, yWorks) implement partial round-trip
- Distinguishing layout changes vs structural changes is unsolved

### Key Challenges

1. **Semantic metadata preservation** — visual edits can destroy semantic layer
2. **Layout vs structure detection** — moving a box ≠ changing architecture
3. **Format conversion lossy** — Mermaid → Excalidraw drops semantic hierarchy
4. **Divergence management** — when text source and visual diverge, which wins?

### Existing Round-Trip Approaches

**Visual Paradigm (UML ↔ code):**
- Diagram edits → code updates, code changes → diagram updates
- Works for code-diagram pairs, not general visual editing
- Graph-based MOF models with transformation rules

**yWorks Interactive AI Diagrams:**
- Protocol linking text descriptions to visual components
- Maintains index between semantic and visual representations
- Both AI suggestions and manual edits coexist

**Miro/Lucidchart:**
- Generate → manual edit → regenerate loops
- No true round-trip: manual edits don't inform AI's semantic model
- Each regeneration may lose manual refinements

### Practical Workaround

For our skill, the most viable approach:
1. **Keep Mermaid as source of truth** (text-based, diffable, semantic)
2. **Excalidraw as presentation layer** (visual, editable, shareable)
3. **Track divergence** — version Mermaid source and Excalidraw visual separately
4. **Structural changes go through Mermaid** (AI understands text naturally)
5. **Layout/cosmetic changes stay in Excalidraw** (don't need AI understanding)
6. **Use `customData`/element IDs** to link Excalidraw elements back to Mermaid nodes

---

## 5. Off-the-Shelf Tools & MCP Servers

### Excalidraw MCP Servers

1. **Official Excalidraw MCP** (excalidraw/excalidraw-mcp) — released by Excalidraw + Anthropic. Works with Claude Code. Smooth viewport control, inline rendering.
2. **yctimlin/mcp_excalidraw** — most feature-complete community server (26 tools). Real-time WebSocket sync, `describe_scene`, `get_canvas_screenshot`.
3. **debu-sinha/excalidraw-mcp-server** — security-hardened, API auth, rate limiting.

### tldraw MCP Servers

1. **talhaorak/tldraw-mcp** — manages local `.tldr` files. 9 tools: read/write/create/list/search/get_shapes/add_shape/update_shape/delete_shape. MIT licensed.
2. **shahidhustles/tldraw-mcp** — natural language diagram instructions. 5 tools.
3. **Arsenic-01/Prompt2Sketch** — experimental "Excalidraw without limits". Real-time canvas sync, supports Ollama/local LLMs.

### Format-Agnostic Tools

1. **Kroki** (kroki.io) — unified API for 20+ diagram formats (Mermaid, D2, GraphViz, PlantUML, Excalidraw, BPMN). Self-hostable. MCP server available.
2. **Diagram Bridge MCP** — universal converter (9+ formats)
3. **ToDiagram** — JSON/YAML/XML/CSV → interactive diagrams

### Converters

1. **@excalidraw/mermaid-to-excalidraw** — official, maintained. Supports flowcharts. Other diagram types render as raster images. Playground at mermaid-to-excalidraw.vercel.app.
2. **excalidraw-converter** (sindrel/excalidraw-converter) — reverse: Excalidraw → Mermaid. Also supports Gliffy and draw.io.
3. **No Mermaid → tldraw converter exists** — feature request filed but not implemented.

---

## 6. Recommended Architecture for `/visualize` Skill

### Hybrid Approach: Intermediate Format + Excalidraw MCP

```
User prompt
  → Claude generates Mermaid (token-efficient, natural for LLM)
  → Validate syntax
  → Choose layout engine (Dagre default, ELK for nested)
  → Convert to Excalidraw via @excalidraw/mermaid-to-excalidraw
  → Optionally polish via Excalidraw MCP (customData, styling)
  → Store both Mermaid source + .excalidraw JSON
  → Present to user

Human edits in Excalidraw
  → Structural changes: update Mermaid source (Claude reads .excalidraw, regenerates Mermaid)
  → Layout changes: keep in Excalidraw only
  → Track divergence via version counters
```

### Why This Works

- AI-friendly: Claude generates Mermaid naturally (~200 tokens vs ~2000 for JSON)
- Auto-layout: No coordinate calculations needed
- Version control: Mermaid text is Git-friendly
- Visual refinement: Excalidraw MCP for polish and iteration
- Bidirectional: Mermaid as semantic truth, Excalidraw as visual layer
- Open source: All MIT, no licensing concerns
- Existing tooling: Official converter, official MCP server

### Storage Schema

```json
{
  "diagram_id": "uuid",
  "source_format": "mermaid",
  "source_content": "graph TD\n  A-->B",
  "layout_engine": "dagre",
  "visual_format": "excalidraw",
  "visual_content": "{...excalidraw json...}",
  "semantic_version": 1,
  "visual_version": 3,
  "diverged": false,
  "last_synced": "2026-02-16T10:00:00Z"
}
```

### Alternative: Direct Excalidraw MCP (No Intermediate)

For diagrams where Mermaid doesn't fit (freeform whiteboard, spatial brainstorming):
- Use Excalidraw MCP directly
- Claude writes shapes with `customData` for semantic tagging
- More expensive (more tokens) but more flexible
- Good for: decision trees, mind maps, spatial layouts, annotated screenshots

---

## 7. Key Insights for Implementation

### What tldraw Got Right (Learn From)

1. **Dual perception**: Agent sees both screenshot (spatial) AND structured data (semantic). Our skill should provide Claude with both Mermaid text AND visual context.
2. **Streaming generation**: Users see diagram form in real-time. For Claude Code, we can emit incremental Mermaid.
3. **Iterative refinement**: Make Real's "annotate and regenerate" pattern. Our skill should support "update this part" commands.
4. **Meta/customData**: Both tools support metadata on elements — critical for bidirectional round-trips.

### Critical Research Quotes

> "LLMs are not equipped to deal with reworking layout with each iteration for visual output" — simmering.dev

→ Validates intermediate format approach. Let layout engines handle positioning.

> "Users moved from passively consuming AI outputs → actively curating/editing them" — allo.io

→ The skill must support edit-then-refine loops, not just one-shot generation.

> "The equivalence problem: defining when two models are 'synchronized'" — round-trip engineering literature

→ We need a simple divergence model (version counters), not perfect sync.

### Open Questions

1. **Mermaid vs D2 as primary format?** Mermaid has wider LLM support and official Excalidraw converter. D2 has better aesthetics and ELK by default. Recommendation: Mermaid first, D2 as future option.

2. **Which Excalidraw MCP?** Official (simple, blessed) vs yctimlin (26 tools, more powerful). Could start official, migrate if needed.

3. **How to handle non-flowchart diagrams?** The official Mermaid-to-Excalidraw converter only supports flowcharts. For sequence/ER/state diagrams, options: (a) render as image in Excalidraw, (b) build custom converter, (c) use Kroki as universal renderer.

4. **Trigger model for the skill?** Options: explicit `/visualize` command only, proactive during planning, or both.

---

## 8. Sources

### tldraw
- [tldraw Agent Starter Kit](https://tldraw.dev/starter-kits/agent)
- [Make Real Demo](https://makereal.tldraw.com/)
- [tldraw SDK Docs](https://tldraw.dev/)
- [talhaorak/tldraw-mcp](https://github.com/talhaorak/tldraw-mcp)
- [Prompt2Sketch](https://github.com/Arsenic-01/Prompt2Sketch)
- [Gemini Powers tldraw](https://ai.google.dev/showcase/tldraw)
- [The Accidental AI Canvas (Latent Space)](https://www.latent.space/p/tldraw)

### Excalidraw
- [Excalidraw API Docs](https://docs.excalidraw.com/)
- [excalidraw/excalidraw-mcp](https://github.com/excalidraw/excalidraw-mcp)
- [yctimlin/mcp_excalidraw](https://github.com/yctimlin/mcp_excalidraw)
- [@excalidraw/mermaid-to-excalidraw](https://github.com/excalidraw/mermaid-to-excalidraw)
- [excalidraw-converter (reverse)](https://github.com/sindrel/excalidraw-converter)
- [mermaid-to-excalidraw playground](https://mermaid-to-excalidraw.vercel.app)

### Layout & Algorithms
- [Dagre GitHub](https://github.com/dagrejs/dagre)
- [ELK (Eclipse Layout Kernel)](https://www.eclipse.org/elk/)
- [Cola.js](https://ialab.it.monash.edu/webcola/)
- [D3-Force](https://d3js.org/d3-force)
- [Kroki.io](https://kroki.io/)

### Diagramming Research
- [simmering.dev — Diagrams and LLMs](https://simmering.dev/blog/diagrams/)
- [allo.io — Whiteboard Apps + AI Era](https://allo.io/blog/whiteboard-apps-are-dead-but-visual-collaboration-thrives-in-the-ai-era/)
- [infinitecanvas.tools — Spatial Thinking](https://infinitecanvas.tools/)
- [Visual Paradigm Round-Trip Engineering](https://www.visual-paradigm.com/)
- [yWorks AI Diagrams](https://www.yworks.com/)
