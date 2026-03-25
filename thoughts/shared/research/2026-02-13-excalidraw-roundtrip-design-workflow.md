---
date: 2026-02-13T12:00:00-05:00
researcher: claude
git_commit: 97c176d
branch: main
repository: filescience
topic: "Excalidraw/tldraw round-trip design workflow feasibility"
tags: [research, excalidraw, tldraw, diagrams, design-workflow, architecture]
status: complete
last_updated: 2026-02-13
last_updated_by: claude
---

# Research: Excalidraw/tldraw Round-Trip Design Workflow

**Date**: 2026-02-13
**Researcher**: claude
**Git Commit**: 97c176d
**Branch**: main
**Repository**: filescience

## Research Question

Can we build a round-trip workflow where Claude dumps the current system architecture as an editable Excalidraw/tldraw diagram, the user modifies it to express design intent, and Claude reads back the modified diagram to implement or plan from it?

## Summary

**Verdict: Highly feasible with Excalidraw as the target format.**

The Excalidraw JSON format is simple, well-documented, and has a Python library (`Excalidraw-Interface`) for programmatic generation. tldraw can import Excalidraw files, so generating `.excalidraw` covers both tools. The codebase already has a comprehensive diagram generation pipeline (`scripts/generate_diagrams.py`) with tool-agnostic data extraction that can be extended to produce Excalidraw output alongside existing Structurizr/Mermaid output.

The key insight: our existing IR layer (dataclasses for `InfraStack`, `PolylithBrick`, `PolylithProject`, `ArchitectureModel`) already extracts all the structural data needed. A new renderer targeting Excalidraw JSON would reuse the same parsers.

## Detailed Findings

### 1. Excalidraw JSON Format

**Top-level structure:**
```json
{
  "type": "excalidraw",
  "version": 2,
  "source": "https://excalidraw.com",
  "elements": [...],
  "appState": { "gridSize": 20, "viewBackgroundColor": "#ffffff" },
  "files": {}
}
```

**Element types:** `rectangle`, `ellipse`, `diamond`, `text`, `arrow`, `line`, `freedraw`, `image`, `frame`

**Key mechanisms:**
- **Text inside shapes**: Separate `text` element with `containerId` pointing to parent shape. Parent shape has `boundElements: [{ id: "text-id", type: "text" }]`
- **Arrow bindings**: `startBinding`/`endBinding` with `{ elementId, focus, gap }`. Parent shapes list bound arrows in `boundElements`
- **Groups**: `groupIds` array on each element (supports nested groups)
- **Frames**: `frame` element type with `children` array; child elements have `frameId`
- **Sticky notes**: No native type — colored rectangle with bound text and `roundness: { type: 3, value: 32 }`

**Colors/styling:**
- `backgroundColor`/`strokeColor`: Hex strings (Open Colors palette)
- `fillStyle`: `"hachure"` | `"cross-hatch"` | `"solid"` | `"dotted"`
- `strokeStyle`: `"solid"` | `"dashed"` | `"dotted"`
- `roughness`: 0 (smooth) to 2 (sketchy)
- `fontFamily`: 1 (hand-drawn), 2 (normal), 3 (monospace)

**Sources:**
- [JSON Schema docs](https://docs.excalidraw.com/docs/codebase/json-schema)
- [Element Skeleton API](https://docs.excalidraw.com/docs/@excalidraw/excalidraw/api/excalidraw-element-skeleton)
- [Style customization](https://docs.excalidraw.com/docs/@excalidraw/excalidraw/customizing-styles)

### 2. Python Library: Excalidraw-Interface

**Package:** `Excalidraw-Interface` on PyPI (Python >= 3.10)

```python
from Excalidraw_Interface import SketchBuilder

sb = SketchBuilder(roughness=2)
rect = sb.Rectangle(x=100, y=100, width=200, height=100)
circle = sb.Ellipse(x=400, y=150, width=100, height=100)
text = sb.TextBox('Hello World', x=150, y=125)
sb.create_binding_arrows(rect, circle)
sb.export_to_file('my_diagram')  # Creates my_diagram.excalidraw
```

Supports: Rectangles, Diamonds, Ellipses, Text, TextBox, HeaderContentBox, Lines, Arrows, DoubleArrow, Groups, binding arrows, colors, roughness.

**Alternative:** `procXD` (GitHub: BardOfCodes/procXD) — original library, also supports NetworkX graph visualization.

**Fallback approach:** Generate raw JSON directly — the format is simple enough that a custom generator may be more flexible than using a library (especially for layout control).

### 3. tldraw Format Comparison

| Aspect | Excalidraw | tldraw v2 |
|--------|------------|-----------|
| Format | Flat element array | Versioned records with separate bindings |
| Bindings | Properties on arrow element | Separate `TLArrowBinding` records |
| Text | Plain text with containerId | TipTap/ProseMirror JSON (rich text) |
| Sticky notes | Rectangle + text hack | Native `TLNoteShape` type |
| Python libs | `Excalidraw-Interface` (PyPI) | `tldraw` (Jupyter only) |
| Schema complexity | Simple (flat) | Complex (migrations, versioning) |
| Import .excalidraw | N/A | Yes (official converter) |

**Recommendation:** Generate Excalidraw format. tldraw can import it. Excalidraw format is significantly simpler to generate programmatically.

### 4. Current Diagram Infrastructure

**`scripts/generate_diagrams.py`** (1383 lines) already implements:

**Tool-agnostic IR layer** (reusable for Excalidraw output):
- `InfraStack`: Terragrunt stack with name, module, dependencies, function_name, handler
- `PolylithBrick`: Component/base with name, kind, forbidden_imports, allowed_imports
- `PolylithProject`: Deployable project with name, bricks list
- `ArchitectureModel`: Complete model containing stacks, bricks, projects

**Parsers** (reusable as-is):
- `parse_terragrunt_stacks()`: Regex-based HCL parsing → `list[InfraStack]`
- `parse_polylith_workspace()`: TOML parsing → bricks, projects, dependencies
- `parse_all_bricks_code()`: AST parsing → classes, methods, fields per brick

**Current output targets:**
- L1-L3: Structurizr JSON + PlantUML/SVG via `buildzr`
- L4: Mermaid class diagrams in interactive HTML

**Makefile targets:** `make diagram`, `make diagram-svg`, `make diagram-l4`

### 5. Extractable Codebase Structure Data

**Component dependency graph** (from import-linter contracts):
```
models → (leaf)
throttling → (leaf)
lambda_utils → (leaf)
cloud_api → models
domain_backup → models
dynamodb → models, domain_backup
```

**Base → component dependencies** (from actual imports):
```
discover → cloud_api, domain_backup, dynamodb, models, throttling
valkey_queue_processor → lambda_utils, throttling
entity_discovery_trigger → lambda_utils, cloud_api, domain_backup, dynamodb, models
```

**Project assemblies** (from project pyproject.toml):
```
discover project: discover base + models, throttling, dynamodb, domain_backup, cloud_api
entity-discovery-trigger project: entity_discovery_trigger base + models, dynamodb, domain_backup, cloud_api, lambda_utils
valkey-queue-processor project: valkey_queue_processor base + models, throttling, dynamodb, domain_backup, lambda_utils
```

**Infrastructure graph** (from Terragrunt):
```
entity-discovery-trigger Lambda → artifact-bucket, dynamodb-resources, lambda-layers
valkey-queue-processor Lambda → artifact-bucket, vpc, valkey, dynamodb, idempotency, lambda-layers
work-queue DynamoDB --stream--> valkey-queue-processor Lambda
valkey → vpc, bastion
```

**Runtime routing** (from Python code):
```
DiscoveryManager → DispatcherV2 → CloudRouter (by cloud)
  O365Router → ServiceRouter (by service)
    onedrive.router → handlers (by node_type)
    outlook.router → handlers (by node_type)
```

### 6. Round-Trip Workflow Design

**Proposed workflow:**

```
make diagram-excalidraw          # Code → Excalidraw JSON
  ↓
User opens in excalidraw.com     # Or tldraw (imports .excalidraw)
  ↓
User modifies: adds boxes,       # Design thinking in visual form
  arrows, sticky notes, moves
  things around, annotates
  ↓
User saves .excalidraw file      # Modified JSON
  ↓
Claude reads .excalidraw JSON    # Diff: what was added/removed/annotated?
  ↓
Claude plans/implements           # Translates visual changes to code plan
```

**What Claude can extract from modified diagrams:**

| User Action | Excalidraw Signal | Claude Interpretation |
|-------------|------------------|----------------------|
| New rectangle with text | New element not in original | New component/service to create |
| New arrow between shapes | New arrow with bindings to existing elements | New dependency/data flow |
| Deleted element | `isDeleted: true` or missing from elements | Component to remove |
| Moved element to different frame | Changed `frameId` | Reorganization of module boundaries |
| Added sticky note (colored rect + text) | Rectangle with `backgroundColor` + bound text | Design annotation/constraint/question |
| Changed arrow style | `strokeStyle: "dashed"` | Optional/future dependency |
| Changed element color | `backgroundColor` changed | Status/category change |

**Convention system** (for Claude to interpret):
- Yellow sticky notes = questions/open issues
- Green elements = new additions
- Red/dashed = removals/deprecations
- Blue = existing (generated from code)
- Frame boundaries = module/service boundaries

### 7. Implementation Approach

**Option A: Extend generate_diagrams.py** (recommended)
- Add new renderer using existing IR layer
- New function: `render_excalidraw(model: ArchitectureModel) -> dict`
- New Makefile target: `make diagram-excalidraw`
- Reuses all existing parsers
- ~200-400 lines of new code

**Option B: Use Excalidraw-Interface library**
- `pip install Excalidraw-Interface`
- Simpler element creation API
- Less control over layout
- Additional dependency

**Option C: Raw JSON generation**
- No library dependency
- Full control over layout algorithm
- More verbose but more flexible
- Can embed custom metadata (e.g., `customData` field with component paths)

**Recommended: Option A + Option C hybrid** — extend the existing script with a raw JSON renderer that uses `customData` to tag elements with their source (file paths, component names). This enables Claude to diff original vs modified and understand which elements map to which code.

### 8. Layout Considerations

Excalidraw doesn't auto-layout — positions must be specified. Approaches:
- **Grid layout**: Components in a grid, bases in a row, infrastructure in another row
- **Hierarchical**: Top-to-bottom flow matching C4 levels
- **Force-directed**: Use NetworkX spring_layout for positioning, then convert to Excalidraw coordinates
- **Manual seed**: Generate once with reasonable positions, user can rearrange

The existing `generate_diagrams.py` doesn't need layout logic for Structurizr (it has auto-layout) or Mermaid (browser-rendered). Excalidraw will need explicit coordinates.

**Simplest viable approach**: Fixed grid with predictable spacing (e.g., infrastructure layer at top, components in middle, bases at bottom, 250px spacing).

## Code References

- `scripts/generate_diagrams.py` — Complete diagram pipeline (1383 lines)
  - Lines 31-72: Tool-agnostic IR dataclasses
  - Lines 960-1122: Terragrunt + Polylith parsers
  - Lines 1256-1280: Structurizr model builder
  - Lines 555-928: Mermaid renderer
- `pyproject.toml:175-270` — Import-linter contracts (component dependency source)
- `workspace.toml` — Polylith workspace config
- `projects/*/pyproject.toml` — Project brick assembly definitions
- `infrastructure/live/non-prod/us-east-1/dev/*/terragrunt.hcl` — Infrastructure definitions

## Architecture Documentation

The existing pipeline follows a 3-layer pattern:
1. **Parsers** → tool-agnostic IR (dataclasses)
2. **Model builders** → tool-specific (buildzr/Mermaid)
3. **CLI** → orchestration + file I/O

Adding Excalidraw output means adding a new model builder (layer 2) that consumes the same IR. No parser changes needed.

## Historical Context

- `memory-bank/thoughts/shared/decisions/2026-02-12-c4-diagram-methodology-and-tooling.md` — C4 adoption decision, chose buildzr
- `memory-bank/thoughts/shared/plans/2026-02-13-automatic-architecture-diagrams.md` — L1-L3 implementation plan (all phases complete)
- `memory-bank/thoughts/shared/plans/2026-02-13-l4-diagram-improvements.md` — L4 enhancement plan (phases 1-2 complete)
- `memory-bank/thoughts/shared/research/2026-02-12-architecture-diagram-automation.md` — Comprehensive tooling comparison (mentions Excalidraw + Structurizr pipeline)
- `memory-bank/thoughts/shared/research/2026-02-12-discover-diagram-generation.md` — Mermaid examples for discover flow

## Related Research

- [[2026-02-12-architecture-diagram-automation|2026-02-12-architecture-diagram-automation.md]] — Mentions Excalidraw + Claude Code skill approach and Structurizr → Excalidraw pipeline via Mermaid bridge
- [[2026-02-12-discover-diagram-generation|2026-02-12-discover-diagram-generation.md]] — Discover flow diagrams in Mermaid

## Open Questions

1. **Layout algorithm**: Grid vs force-directed vs hierarchical? Force-directed via NetworkX would look most natural but adds complexity.
2. **Granularity levels**: Should we generate separate Excalidraw files per C4 level (L1, L2, L3) or one combined diagram?
3. **customData convention**: What metadata to embed in elements for round-trip identification? Minimum: `{ "source": "generated", "component": "cloud_api", "file": "components/filescience/cloud_api/" }`.
4. **Diff strategy**: When Claude reads back a modified diagram, should it compare against the last generated version, or parse semantically?
5. **tldraw preference**: User currently designs in tldraw. Should we generate `.excalidraw` (simpler) and let tldraw import, or invest in direct `.tldr` generation?
6. **Excalidraw-Interface viability**: Need to verify the library works on Python 3.13 and generates valid output for our use case. May be simpler to generate raw JSON.
