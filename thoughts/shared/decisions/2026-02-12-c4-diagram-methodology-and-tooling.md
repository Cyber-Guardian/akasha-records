# Decision: C4 Diagram Methodology and Tooling Strategy

**Date:** 2026-02-12
**Status:** Decided
**Context:** [[2026-02-12-architecture-diagram-automation|2026-02-12-architecture-diagram-automation.md]]

---

## Decision

Adopt **C4 as a mental model** (not formal Structurizr DSL) with a **tiered implementation** that scales from current size (2-3 Lambdas) to target size (20 Lambdas, 10 tables, 6+ subsystems).

## Why C4

Polylith + Terraform already encode the exact abstraction layers C4 describes:
- **C4 Context** = FileScience + Tenants + Microsoft Graph API
- **C4 Container** = Terraform modules (Lambda functions, DynamoDB, Valkey, S3)
- **C4 Component** = Polylith bases + components, enforced by import-linter contracts
- **C4 Code** = Skip (low value for Python)

No other methodology (Arc42, 4+1) justifies its overhead for a small team.

## Tiered Scaling Plan

### Now (2-3 Lambdas): Option C — Code-Is-The-Model

Separate files per level, auto-generated where possible:

```
docs/architecture/
├── L1-system-context.excalidraw        # Manual, update quarterly
├── L2-containers.excalidraw            # make diagram-infra
├── L3-discover.excalidraw              # make diagram-deps-discover
├── L3-queue-processor.excalidraw       # make diagram-deps-processor
├── L3-entity-trigger.excalidraw        # make diagram-deps-trigger
└── README.md
```

Sources of truth:
- L2: Terraform/Terragrunt modules
- L3: import-linter contracts + pydeps
- L1: Manual (rarely changes)

### At 6+ Containers: Path 2 — Thin Python Generator

Custom script reads Terraform + import-linter → generates sliceable views:
- Subsystem views ("discover pipeline", "ingestion pipeline")
- Data-centric views ("everything touching work-queue table")
- Per-team views

Why not Structurizr at this stage:
- We'd have to build the Terraform→Structurizr and Polylith→Structurizr bridges ourselves (no existing tooling)
- A custom Python generator gives us the same slicing with less abstraction

### At 20 Lambdas: Path 3 — Auto-Populated Structurizr (if needed)

Migrate to Structurizr only if Path 2's rendering/layout becomes painful.
Use `buildzr` (best maintained Python Structurizr library, type-safe, Pythonic).
Auto-populate from Terraform + import-linter (same bridge scripts from Path 2).

## Tooling Stack

### Rendering

| Purpose | Tool | Why |
|---------|------|-----|
| **Architecture diagrams (freeform)** | Excalidraw via Claude Code skill | Best layout control, professional aesthetic, VS Code extension |
| **Inline diagrams (PRs, READMEs)** | Mermaid | Native GitHub rendering, zero setup |
| **Python dependency graphs** | pydeps | Auto-generated from bytecode |
| **Terraform dependency chain** | `terragrunt dag graph` | Built-in, zero setup |
| **Import boundary enforcement** | import-linter | Already active (7 contracts) |

### The Excalidraw Pipeline

**Now:** Claude Code Excalidraw skill generates `.excalidraw` directly from codebase analysis.

**Future (if model-based approach needed):**
```
buildzr (Python) → Structurizr JSON → CLI export → Mermaid → mermaid-to-excalidraw → .excalidraw
```

The bridge exists: `@excalidraw/mermaid-to-excalidraw` (699 stars, maintained by Excalidraw team).
Limitation: layout is regenerated (not preserved), so manual refinement needed.

**Pragmatic hybrid:** Use Structurizr→Mermaid for deterministic/repeatable diagrams, Claude Code skill for polished Excalidraw versions that need freeform layout.

## Key Gaps in Ecosystem (as of 2026-02)

| Gap | Status | Workaround |
|-----|--------|------------|
| Terraform → Structurizr auto-gen | No tooling exists | Custom Python script |
| Polylith → Structurizr auto-gen | No tooling exists | Parse `poly info` + import-linter contracts |
| Structurizr → Excalidraw direct | No converter | Structurizr → Mermaid → mermaid-to-excalidraw |
| D2 → Excalidraw | No converter | Separate rendering paths |
| `structurizr-python` | **Archived** | Use `buildzr` instead |
| Structurizr CLI | Archived (Feb 2026), consolidated into vNext | Monitor vNext, CLI still functional |

## D2 as Alternative to Structurizr (Research Update 2026-02-12)

D2 (by Terrastruct, 23k stars) was evaluated as an alternative to Structurizr for the "at scale" tiers.

### D2 Advantages Over Structurizr

- **Simpler syntax** — easier to generate programmatically via `py-d2`
- **Scenarios** — sliced views from a single source file (80% of Structurizr's model power, less ceremony)
- **Imports** — composable architecture across files (domain experts contribute separately)
- **Native C4 support** (v0.7+) — `suspend`/`unsuspend`, `c4-person` shape, C4 theme
- **AWS icons built-in** — Lambda, DynamoDB, S3 etc. via icons.terrastruct.com
- **Watch mode** — `d2 --watch` with live browser reload, fully offline
- **Active, not in transition** — Structurizr CLI archived Feb 2026, Cloud EOL Sept 2026

### D2 Disadvantages

- **No GitHub rendering** — must pre-render SVG/PNG (Mermaid renders natively)
- **No D2→Excalidraw converter** — can't feed into our Excalidraw skill pipeline
- **TALA is proprietary** ($20/mo) — free engines (Dagre/ELK) are worse for architecture
- **Small company** (2-10 employees, $2M funding) — TALA sustainability risk
- **Same bridge-script gap** — no Terraform→D2 or Polylith→D2 auto-gen tooling either

### Revised "At Scale" Assessment

At 20 Lambdas, the choice becomes **D2 vs Structurizr (via buildzr)** for the model layer:

| Factor | D2 | Structurizr |
|--------|-----|-------------|
| Syntax complexity | Lower | Higher |
| Sliced views | Scenarios + imports | Workspace views + filters |
| Model rigor | Hybrid (render-with-model-features) | True model |
| Excalidraw bridge | None | Via Mermaid export |
| Platform stability | Stable (OSS), TALA risk | Consolidating (vNext active) |
| Python library | `py-d2` (89 stars) | `buildzr` (8 stars) |

**Decision:** D2 is a viable alternative at scale, especially if we don't need the Excalidraw bridge. We defer this choice — evaluate when we actually hit 6+ containers. Both require custom bridge scripts from Terraform/Polylith, so the switching cost is primarily in the rendering/model layer, not the data extraction layer.

**Action:** Build the data extraction layer (Terraform parser + import-linter parser) in a tool-agnostic way, so it can target either D2 or Structurizr.

## What We're NOT Doing

- **Not adopting Structurizr DSL now** — overhead not justified at current scale
- **Not adopting D2 now** — same reasoning; evaluate at 6+ containers
- **Not using Structurizr Cloud/Lite** — Lite is EOL, Cloud is EOL Sept 2026
- **Not paying for TALA now** — evaluate if free ELK layout is sufficient first
- **Not using PlantUML** — adds Java dependency, Mermaid covers our needs
- **Not auto-generating class diagrams** — low value for Python

## Alternatives Rejected

| Alternative | Why Rejected |
|-------------|-------------|
| Arc42 | 12-section template, too heavy for small team |
| 4+1 View Model | Academic, largely historical |
| Informal (no methodology) | No completeness check, random diagrams that overlap or leave gaps |
| Structurizr DSL now | No Terraform/Polylith auto-gen tooling, overhead not justified yet |
| D2 now | Same auto-gen gap, plus no GitHub rendering and no Excalidraw bridge |
| Full formal C4 with Structurizr | Deferred to 20-Lambda scale |
