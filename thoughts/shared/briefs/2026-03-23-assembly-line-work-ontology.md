---
type: event
created: 2026-03-23
status: active
---
# Idea Brief: Assembly-Line Work Ontology for Agent-Driven Development

**Date:** 2026-03-23
**Status:** Shaped → Planning (v2 — supersedes earlier markdown-only approach)

## Problem

The work ontology — shape → plan → implement → commit — is designed for human developers but executed by agents. Agents have fundamentally different failure modes: they're additive by default (10.8:1 add/remove ratio), they don't self-simplify (1.4% refactor commits), they ship-then-patch (1.7 fixes per feat), they rewrite plans instead of executing them (78% of plans have 6+ touches, ~300% rewrite ratio), and god modules grow unchecked (coordinator.py: 2,143 lines, 105 touches). The ontology has no counter-pressure against these tendencies.

Current hooks are strong at the edit level (ruff, ty, import-linter) but completely absent at the workflow level. Plans are monolithic markdown files that serve both humans and agents poorly — too verbose for agents (context rot), too structured for humans (readability). Nothing mechanically enforces that contracts are satisfied.

## Constraints

- Must build on existing `.work/` infrastructure: `protocol.md`, `format.md`, `templates/plan.md`, `lib/work_protocol/` (models + parser)
- Must build on existing Claude Code hook + MCP infrastructure
- 5+ prior briefs explored parts of this problem — none shipped because they were partial solutions
- Agents can and should read human docs for context — the separation is about having dedicated machine artifacts for enforcement, not restricting access
- spec-workflow-mcp (Pimzino) validates the MCP-tools-for-plan-management pattern in production
- GSD-2's `.gsd/` validates the plan-as-directory pattern for agent work management

## Chosen Approach

**Plans-as-directories in `.work/plans/` with MCP tooling for deterministic lifecycle enforcement.** Typed contract YAML with 3-layer shift-left validation. Progressive disclosure: human README + agent manifest + per-phase contract files. Reflection station with multi-lens review and iterative fix loops.

### The Directory Structure

```
.work/plans/<plan-name>/
├── README.md              ← human: overview, reasoning, context, tradeoffs
├── manifest.yaml          ← agent: typed phases, tasks, files, dependencies
├── graph.yaml             ← edges: depends_on, enables, supersedes
├── phase-1/
│   ├── brief.md           ← human: phase context, design decisions
│   ├── contracts.yaml     ← enforcement: typed assertions (shell + structured)
│   └── state.yaml         ← progress: tasks done, contracts passed, reflection results
├── phase-2/
│   └── ...
└── history.jsonl          ← append-only decision register
```

### MCP Tools (deterministic enforcement)

```
mcp__work__create_plan         → creates directory structure
mcp__work__add_phase           → adds phase subdir with schema-validated files
mcp__work__add_contract        → appends to contracts.yaml (validated on write)
mcp__work__validate_plan       → checks references, completeness
mcp__work__freeze_plan         → locks plan, blocks further structural edits
mcp__work__get_phase           → returns typed phase data for agent consumption
mcp__work__run_contracts       → executes all assertions, returns pass/fail table
mcp__work__complete_task       → marks task done (only if contracts pass)
mcp__work__reassess            → logs observations to history.jsonl
mcp__work__get_graph           → renders plan dependency DAG
```

### 3-Layer Validation Chain (shift-left)

```
Layer 1 (write time):  add_contract validates YAML schema
Layer 2 (freeze time): freeze_plan validates all references exist
Layer 3 (impl time):   run_contracts executes assertions against codebase
```

### Contract Format (YAML)

```yaml
shell:
  - cmd: "make test-fast"
    proves: "all tests pass"
assertions:
  - type: file_exists
    path: .claude/rules/tdd-first.md
  - type: file_lines
    path: coordinator.py
    max: 500
  - type: net_growth
    scope: phase
    max: 50
  - type: import_valid
    module: "work_protocol.models"
```

### Assembly Line Flow

```
SHAPE → PLAN → FREEZE → [per phase:] → SHIP
                              │
                         TEST (red)
                              │
                        IMPLEMENT (green)
                              │
                        CONTRACTS (run_contracts)
                              │
                        REFLECT (multi-lens)
                         correctness
                          structure
                          alignment
                            cost
                          reassess
                              │
                        CIRCUIT BREAKER
                        (2x fail → kill & reshape)
```

### Key Design Elements

1. **Plans are directories** — progressive disclosure. Human reads README.md, agent reads manifest.yaml. Both available, only YAML mechanically enforced.
2. **MCP tools enforce lifecycle** — deterministic, not convention-based. Agent calls typed tools, not parses markdown.
3. **TDD ordering** — tests first (red), implement (green), contracts, reflect. Template ordering IS the enforcement.
4. **Typed contracts** — shell commands + structured assertions in YAML, schema-validated at write time.
5. **3-layer validation** — schema → references → execution. Shift-left: catch errors at cheapest point.
6. **Multi-lens reflection** — correctness, structure, alignment, cost, reassess. 3-pass cap. V1 is same-session structured self-review; V2 (ENG-2763) adds independent agent dispatch.
7. **Plan graph** — `graph.yaml` with depends_on/enables edges. MCP tool renders DAG.
8. **Reassess lens** — meta-lens checking whether the plan still makes sense. Outputs observations for human, not fix items.
9. **Circuit breaker** — plans that can't execute get killed and reshaped, not endlessly rewritten.

## Key Context Discovered During Shaping

### Git meta-analysis
- 1,135 commits over 8 weeks; 59% agent-authored, 10.8:1 additive ratio
- 78% of plan files have 6+ touches, ~300% rewrite ratio
- 1.7 fix-chain per feature, god modules never decomposed

### Grounding research
- **GSD-2**: `.gsd/` directory validates plan-as-directory pattern. Deleted 8,300 lines for 1,400 lines of linear loop.
- **spec-workflow-mcp**: Production MCP server for spec-driven workflow. Direct precedent for our mcp__work__* tools.
- **Shape Up**: Circuit breaker (kill don't extend), fixed appetite
- **Toyota TPS**: Jidoka (stop-the-line quality gates)
- **Shift-left**: 5x/10x/100x cost multiplier for late defect detection (industry-wide evidence)
- **HBS 2016**: Temporary architectural decisions "expedient in the short-term increase costs of maintaining and adapting in future"

### Existing infrastructure
- `.work/protocol.md` — GSD-2-inspired work ontology (Milestone→Phase→Task)
- `.work/format.md` — plan heading structure spec
- `.work/lib/work_protocol/` — Python models + parser (foundation for MCP server)
- `.work/templates/plan.md` — canonical plan template (to be evolved)

### Deferred to V2 (ENG-2761)
- ENG-2762: Automated TaskCompleted contract runner
- ENG-2763: Independent-agent reflection dispatch
- ENG-2764: CI contract staleness validation
- ENG-2765: File size gate hook with threshold tuning

## Next Step
- Plan → rewrite plan for the directory-based architecture
