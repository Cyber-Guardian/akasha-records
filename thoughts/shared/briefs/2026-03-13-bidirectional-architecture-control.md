---
type: event
created: 2026-03-13
status: active
---
# Idea Brief: Bidirectional Architecture Diagram ↔ Code

**Date:** 2026-03-13
**Status:** Shaped → Parked

## Problem
The existing architecture pipeline is read-only (code → diagrams via `scripts/diagrams/`). Import-linter contracts and the `ArchitectureModel` IR make the architecture machine-readable, but the diagram is a dead artifact. The vision is to make the architecture diagram a **control surface** where visual mutations translate back into code changes — closing the loop into a live bidirectional system.

## Constraints
- Python-only stack (rope, libCST — no ts-morph/OpenRewrite needed)
- Polylith brick boundaries map 1:1 to diagram nodes — each node is a filesystem directory
- Import-linter contracts express forbidden edges only — allowed edges are computed as complements via `apply_contracts()`
- The `config` component currently has no import-linter contract (gap)
- Tooling/product boundary contracts are commented out (ENG-2484 Phase 2)
- Must validate rope handles Polylith namespace packages (`filescience.*` across `components/` and `bases/`)
- Must compose with existing agent infrastructure (helm, plan-implementer, triage-agent)

## Research Context
Three parallel research threads validated this idea:

**Bidirectional diagram↔code tools:** Nobody has built general-purpose diagram→code. 30 years of UML round-trip engineering failed due to the behavioral completeness problem (diagrams capture structure, not implementation). The one success is Stately/XState — works because state machines are semantically complete. Key principle: bidirectionality works when the visual model captures everything that matters at its abstraction level.

**AST rewriting:** rope (Python) is the most complete cross-file refactoring engine — rename, move, extract across entire projects. libCST is best for codemod-style transforms with lossless round-trips. tree-sitter provides fast syntax parsing for building component graphs but has no semantic layer. No tool is designed for the Polylith pattern specifically.

**Microagent architectures:** The field converges on: deterministic ops via LSP/AST (Kiro's insight), scoped agents for judgment calls, dependency graph as the scoping medium, spec-first intent articulation before code changes.

## Options Considered

### Architecture DSL + Reconciler ("Desired State")
Define target architecture in a YAML/TOML DSL. Reconciler diffs DSL against actual codebase state via `ArchitectureModel` IR, generates migration plan — rope for structural changes, agents for semantic ones, contract rewrites for boundaries. Diagram is a rendered view of the DSL.
- Gains: Declarative, diffable, version-controlled. "Architecture as code" proven paradigm. Plan/apply review step.
- Costs: Another config format alongside import-linter. No visual editing.
- Complexity: Medium

### Diagram-as-Control-Surface ("Visual Mutations")
Interactive Excalidraw diagram where visual mutations (rename, drag, split, connect/disconnect) are captured as structured events, classified (structural vs semantic), dispatched to appropriate handlers.
- Gains: Ultimate UX — see it, change it. Zero abstraction gap. Composes with [[2026-02-27-visualization-in-shaping-planning|Visualization Brief]].
- Costs: Hardest to build. Correspondence problem must be solved. Layout preservation on round-trip.
- Complexity: High

### MCP Architecture Server ("Claude as Bridge")
MCP server exposes architecture-aware tools (`rename_brick`, `move_symbol`, `split_component`, `add_dependency`, `remove_dependency`). Claude interprets natural language directives and calls tools. Tools execute rope/libCST operations + contract rewrites internally.
- Gains: Fastest to ship. Natural language already the interface. Each tool independently testable. Composable with helm and out-loop agents.
- Costs: No visual feedback until diagram regeneration. Relies on Claude interpreting intent.
- Complexity: Low-Medium

### Layered Composition (C → A → B)
Ship MCP tools first (pass-1), extract into DSL reconciler (pass-2), wire visual diagram as frontend to DSL (pass-3). Each layer independently valuable.
- Gains: Incremental validation. Discovers rope/Polylith issues at pass-1, not after building visual editor. MCP tools become stable API for all layers above.
- Costs: Three investments. Risk of DSL feeling redundant.
- Complexity: Low → Medium → High (incremental)

## Chosen Approach
**Layered Composition** — Aligns with the spiral pass model. Each pass delivers value and informs the next. MCP tools are foundation, DSL is leverage, visual is product.

### Pass-1: MCP Architecture Tools (foundation)
- `rename_brick` — rope rename across project, update import-linter contracts
- `move_symbol` — rope move function/class between bricks, rewire imports
- `split_component` — create new brick, move selected symbols, add re-export stubs, write new contracts
- `add_dependency` / `remove_dependency` — update import-linter forbidden contracts
- `show_architecture` — render current `ArchitectureModel` as structured output
- Contract rewriting engine — programmatic `pyproject.toml` edits for import-linter sections

### Pass-2: Architecture DSL + Reconciler (leverage)
- Declarative desired-state format (superset of import-linter contracts)
- Diff engine: desired state vs actual state (via `ArchitectureModel`)
- Migration plan generator: sequence of MCP tool calls to reach desired state
- Plan/apply model with review step

### Pass-3: Visual Control Surface (product)
- Excalidraw diagram with stable element↔brick correspondence
- Visual mutation capture → DSL diff generation
- DSL diff → reconciler → code changes
- Layout preservation on round-trip

## Key Context Discovered During Shaping

### Existing substrate is stronger than expected
- `scripts/diagrams/ir.py` already synthesizes contracts + filesystem + projects into `ArchitectureModel`
- `apply_contracts()` at `scripts/diagrams/parsers/polylith.py:78-83` computes allowed imports from forbidden contracts
- `scripts/diagrams/parsers/python_ast.py` extracts L4 code model (classes, functions, fields per brick)
- The [[2026-02-27-visualization-in-shaping-planning|Visualization Brief]] provides the Excalidraw substrate for pass-3
- The [[2026-02-28-macro-agent-strategy|Macro Agent Strategy]] envisions "human reviews visually" — this is that interface

### Research validates the approach
- Stately/XState is the only successful bidirectional tool — works because the model is semantically complete
- Polylith bricks may have this property: brick boundaries, dependencies, and imports ARE fully expressible as a graph
- Kiro's key insight: use deterministic LSP/AST for structural ops, LLMs only for judgment calls
- The "correspondence problem" (mapping diagram elements to code entities) is solved by Polylith — each brick IS a directory

### Open risks
- rope may not handle Polylith namespace packages cleanly (needs validation)
- Contract rewriting via programmatic TOML edits may have edge cases with comment preservation
- "Split component" is the hardest operation — deciding which symbols go where is semantic, not structural

## Related Artifacts
- [[2026-02-27-visualization-in-shaping-planning|Visualization in Shaping & Planning Brief]] — the Excalidraw substrate for pass-3
- [[thoughts/shared/plans/2026-02-27-visualization-in-shaping-planning|Visualization Plan]] — 7-phase implementation
- [[2026-02-28-macro-agent-strategy|Macro Agent Strategy]] — the visual review vision this enables
- [[thoughts/shared/plans/2026-02-16-decompose-generate-diagrams|Decompose Generate Diagrams Plan]] — the existing diagram pipeline

## Linear Issues

**Project:** [Architecture Studio](https://linear.app/filescience/project/architecture-studio-6467d47b90ce) (Engineering Leverage)

- **ENG-2519** — Build MCP architecture refactoring tools (pass-1 foundation) — parent
  - **ENG-2520** — Spike: validate rope on Polylith namespace packages (risk gate)
  - **ENG-2521** — Evaluate CodeBoarding: adopt, integrate, or feature-steal
  - **ENG-2522** — Build contract rewriting engine for pyproject.toml
  - **ENG-2523** — Add MCP tools: show_architecture, rename_brick, move_symbol (blocked by 2520, 2522)
  - **ENG-2524** — Add MCP tools: add_dependency, remove_dependency (blocked by 2522)
  - **ENG-2525** — Add MCP tool: split_component (blocked by 2520, 2522)

## Next Step
- [Parked] → All issues in Backlog. Rope spike (ENG-2520) and CodeBoarding evaluation (ENG-2521) are the entry points.
