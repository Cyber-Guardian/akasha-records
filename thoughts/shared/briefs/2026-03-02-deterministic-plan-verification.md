# Idea Brief: Deterministic Plan Verification via Z3 State Machines

**Date:** 2026-03-02
**Status:** Shaped → Parked

## Problem
AI coding agents execute implementation plans by reasoning probabilistically about phase ordering, preconditions, and architectural impact. Plans encode state machines implicitly — phases are transitions, verification criteria are postconditions, file scopes are state boundaries — but none of it is formally checkable. The agent guesses whether phase 2 depends on phase 1's output, whether a change preserves architectural invariants, and whether the plan is internally consistent. These questions have deterministic answers.

## Constraints
- Python-only monorepo, ~81 source files — small enough for full-repo static analysis
- Plan template already has machine-parseable elements: file paths per phase (regex), phase numbers, completion checkboxes, automated verification commands
- Preconditions, postconditions, and inter-phase dependencies are currently free-text prose — need formalization
- 9 import-linter contracts in `pyproject.toml` fully specify the architectural dependency graph
- `generate_scope.py` already parses plans and computes per-phase allowed file sets
- Z3 Python API is mature (`pip install z3-solver`), agent-friendly, no external toolchain

## Options Considered

### Z3 Phase-Chain Verifier (Sequential, Python-native)
Model each phase as a state transition with Boolean preconditions/effects. Z3 checks that every phase's preconditions are met by cumulative effects of prior phases, and architectural invariants hold in every intermediate state. Plan template gets a structured `### State:` block with `produces:`, `requires:`, `preserves:` fields. A CLI tool (`scripts/verify_plan.py`) reads the plan, emits Z3 constraints, and reports counterexamples.
- Gains: Python-native, no external toolchain, agent-callable CLI, oracle loop pattern (propose → verify → fix)
- Costs: Vocabulary design needed (what are valid state predicates?), plan template change required
- Complexity: Medium

### TLA+ Plan Specification (Concurrent-aware, temporal properties)
Model plans as TLA+ specs. TLC model-checker explores all states, verifies invariants and temporal properties. Handles parallel plan streams (like macro-agent-strategy's Stream A/B). Verifies concurrent execution doesn't violate shared invariants.
- Gains: Handles concurrency natively, exhaustive state exploration, liveness properties, AWS-proven at scale
- Costs: Separate language (Java-based TLC), steeper learning curve, plan-to-TLA+ translator needed
- Complexity: Medium-High

### Alloy Architectural Invariant Checker
Model Polylith architecture in Alloy — components, bases, import relationships, 9 contracts. For each plan phase, model proposed changes as relation mutations and verify every intermediate state satisfies architectural constraints.
- Gains: Architectural invariants are exactly Alloy's strength, relational model maps 1:1 to import graph, exhaustive bounded checking
- Costs: Java-based, Eclipse-era UI, separate translator needed
- Complexity: Medium

### Hybrid (Z3 + Alloy + PDDL/VAL)
Use each tool where strongest: Z3 for phase logic, Alloy for architecture, VAL for plan validation as a planning problem.
- Gains: Covers all gaps
- Costs: Three tools, three translators, three output formats
- Complexity: High

## Chosen Approach
**Z3 Phase-Chain Verifier** — Python-native, simplest path, builds on existing plan parsing (`generate_scope.py`), directly callable by agents. Layer on TLA+ for concurrency or Alloy for architecture later if needed.

## Key Context Discovered During Shaping

### Landscape (nobody is doing this for software plans)
- IBM "Bridging LLM Planning Agents and Formal Methods" (ASE 2025) — closest proof of concept, NL→Kripke+LTL+NuSMV, GPT-5 hits F1=96.3%. No code released.
- PAT-Agent (IEEE 2025) — most runnable end-to-end pipeline, NL→CSP→PAT model checker→repair loop. Open-source.
- VeriPlan (CHI 2025) — exact architecture we want (rules→model checker→LLM refinement). No code, targets personal planning.
- Formal-LLM (Rutgers, open-source) — constrains plan generation via CFG/PDA, different angle
- AgentSpec (ICSE 2026) — runtime enforcement DSL for LLM agents, most production-ready design
- SafeAgentBench — LLMs only self-reject 10% of hazardous plans, empirical proof external verification is needed

### Existing tooling in this repo
- 17 deterministic tools already exist (guards, extractors), but none are solvers
- `generate_scope.py` parses plan file paths per phase — the extraction infrastructure exists
- `pyproject.toml:188-296` fully specifies the architectural dependency graph (9 contracts)
- `scripts/diagrams/parsers/polylith.py` computes `allowed_imports` from contracts
- `scripts/diagrams/parsers/python_ast.py` parses code structure via AST

### Key design decisions for when we build
- Start with a finite vocabulary of state predicates derived from what we already parse: file/directory existence, component state, test state, phase completion
- Add `### State:` structured block to plan template with `produces:`, `requires:`, `preserves:` fields
- Build as CLI: `scripts/verify_plan.py --plan <path>` → JSON pass/fail + counterexamples
- Integrate into `create_plan` skill as a verification step before plan is committed

### Research sources
- [Bridging LLM Planning Agents and Formal Methods](https://arxiv.org/abs/2510.03469)
- [VeriPlan: Formal Verification + LLMs for Planning](https://arxiv.org/abs/2502.17898)
- [Formal-LLM: CFG-Constrained Agent Planning](https://arxiv.org/abs/2402.00798)
- [PAT-Agent: Autoformalization for Model Checking](https://arxiv.org/abs/2509.23675)
- [AgentSpec: Runtime Enforcement for LLM Agents](https://arxiv.org/abs/2503.18666)
- [Martin Kleppmann: AI will make formal verification go mainstream](https://martin.kleppmann.com/2025/12/08/ai-formal-verification.html)
- [MCP-Solver: Z3/MiniZinc/PySAT as MCP tools](https://github.com/szeider/mcp-solver)
- [Z3 Python API tutorial](https://theory.stanford.edu/~nikolaj/programmingz3.html)
- [FizzBee: TLA+ with Python syntax](https://fizzbee.io/)

### Related work in this repo
- Deterministic Guardrail Layer brief: `memory-bank/thoughts/shared/briefs/2026-02-25-deterministic-guardrail-layer.md` — defensive (constrain bad output). This brief is offensive (produce correct answers).
- Deterministic claims research: `memory-bank/thoughts/shared/research/2026-02-25-deterministic-layer-ai-coding-claims.md`
- Macro Agent Strategy plan: `memory-bank/thoughts/shared/plans/2026-02-28-macro-agent-strategy.md`

## Next Step
- [Parked] → Linear backlog issue with this brief linked
- When ready: `/create_plan memory-bank/thoughts/shared/briefs/2026-03-02-deterministic-plan-verification.md`
