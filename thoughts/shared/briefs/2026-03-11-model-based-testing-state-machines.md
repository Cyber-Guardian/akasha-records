# Idea Brief: Model-Based Testing with Spec-First State Machines

**Date:** 2026-03-11
**Status:** Shaped → Planning

## Problem
The triage agent (and future stateful services) keeps hitting edge-case bugs that live in unexplored paths through the state space — field erasure on state transitions, silent failures on unexpected event sequences, degrading context across reinvocations. Example-based tests only cover paths humans think of. The combinatorial space of states × events × field presence is too large for manual test authoring.

Motivating bugs:
- ENG-2435: `put_item` erases `original_text` on PROCESSING → IGNORED transition
- ENG-2436: ISSUE_CREATED + thread reply = silent comment with no user feedback
- IGNORED → reinvoke → IGNORED loop with degrading context

## Constraints
- Must work with Python 3.13, pytest, existing Polylith test structure
- Hypothesis already a dependency — RBSM is available without new deps
- External state in DynamoDB (moto available for test doubles), Slack/Linear as injectable boundaries
- Async Lambda handlers — Hypothesis rules are synchronous, need `asyncio.run()` wrappers
- Small team — pattern must be low-overhead to adopt and maintain
- First application is triage agent (6 states, ~8 implicit fields, ~15 decision points), but pattern must generalize

## Options Considered

### Spec-Table + Hypothesis Oracle (separate markdown table → hand-translate to RBSM)
Write state machine spec as markdown table, manually translate to Hypothesis RuleBasedStateMachine with in-memory reference model.
- Gains: Human-readable design doc, decoupled from test framework
- Costs: Two artifacts that can drift, translation overhead
- Complexity: Medium

### Executable DSL (Python DSL auto-generates RBSM)
Define state machine in a lightweight Python DSL that auto-generates the RuleBasedStateMachine class.
- Gains: Zero drift between spec and tests, reusable framework
- Costs: High upfront investment building the DSL before knowing real requirements
- Complexity: High — premature abstraction

### TLA+ Design + Manual Bridge
Write state machine in TLA+, model-check it, manually translate to Hypothesis rules.
- Gains: Strongest design guarantees, exhaustive spec-level verification
- Costs: TLA+ learning curve, manual bridge to Python, overkill for current scale
- Complexity: High

### Mermaid Thinking Tool + Direct RBSM Co-Development (chosen)
Claude generates Mermaid state diagrams during in-loop shaping as a visual thinking tool. Once aligned, Claude converts the diagram to a Hypothesis RBSM scaffold. The RBSM is the living artifact — the Mermaid diagram is disposable after conversion.
- Gains: Visual design aid without tooling investment, RBSM is both spec and test, no sync problem, lightweight adoption
- Costs: Mermaid has no guard/action semantics (Claude bridges this gap via conversation context), no auto-sync from code back to diagram
- Complexity: Low-Medium

## Chosen Approach
**Mermaid Thinking Tool + Direct RBSM Co-Development** — Visual diagrams during design, RBSM as the executable spec and test oracle. Applied first to triage agent, designed as a reusable pattern.

The visual diagram is for human thinking during shaping. The RBSM is the machine-verifiable contract between human intent and agent execution. No new tooling dependencies, no persistent sync problem.

## Key Context Discovered During Shaping

**Tool choice grounded:**
- Hypothesis RBSM is the right tool — actively maintained (v6.151.9), Python 3.13 supported, no credible Python alternative for stateful sequence testing
- Parsec.cloud is the best production case study — oracle/reference model pattern, narrowly scoped machines, external deps at fixture level

**MBT evidence is strong for event-driven systems:**
- Poolboy: 5 bugs at 85% coverage. LevelDB: 17/31-step bugs. Klarna: 5 race conditions. AWS DynamoDB internal: 35-step error trace.
- All required specific operation *sequences* — exactly what RBSM generates

**Agentic dimension is ahead of evidence but promising:**
- LLMs write simple Hypothesis tests at 21-56% success rate, but zero published evidence of RBSM generation
- LLMs can iterate on state-machine specs with model-checker feedback (PAT Model Checking Agent)
- The spec-as-oracle pattern composes well with agent workflows (PGS framework)
- Agent-extending-spec direction is undemonstrated but theoretically coherent

**Visual tooling grounded:**
- Stately Studio and StateSmith exist but add overhead for a 6-state machine
- No tool generates RBSM from visual models — the bridge is always manual/Claude
- Mermaid as disposable thinking tool during shaping is the right weight

**Process insight:**
- RBSM sits at integration level in the testing pyramid (real handler logic, injected boundaries)
- Spec comes before code — state machine designed during planning, RBSM is the TDD red step
- In-loop: human authors states/invariants. Out-loop: agent implements under oracle

## Related
- [[2026-03-11-debug-triage-bot-reinvoke-followup|Triage Bot Debug: Reinvoke + Follow-up Bugs]]
- [[2026-02-11-mini-spec-and-spec-driven-development-adoption|Spec-Driven Development Adoption Notes]]
- [[2026-02-27-visualization-in-shaping-planning|Visualization in Shaping & Planning Plan]]
- ENG-2435: original_text erasure bug
- ENG-2436: silent comment follow-up bug

## Next Step
- Plan → `/create_plan memory-bank/thoughts/shared/briefs/2026-03-11-model-based-testing-state-machines.md`
