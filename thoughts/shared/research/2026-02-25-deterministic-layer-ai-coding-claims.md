# Research: Deterministic Layer for AI Coding Claims

**Date:** 2026-02-25  
**Prompt source:** Reddit comment thread (quoted in session)  
**Focus:** Validate feasibility of a "deterministic layer" that constrains AI coding to provably correct or contract-compliant changes.

## Executive Summary

The core idea is strong: place probabilistic model output inside deterministic verification and policy gates so reliability does not depend only on model intelligence.  

The strongest version of the claim is **partly true**:
- Deterministic gates can significantly improve structural correctness and reduce regressions.
- Deterministic, non-LLM refactors are practical and already proven in multiple ecosystems.
- Small/cheap models can work well when constrained by strong architecture.

The overstated part is absolute language:
- "100% fidelity mapping" and "provably correct" are only achievable in scoped subsets with explicit specifications and controlled language/runtime features.
- General software systems still face computability limits, dynamic behavior ambiguity, and external-boundary uncertainty.

## Claim-by-Claim Verdict

| Claim | Verdict | Why |
|---|---|---|
| We cannot brute-force 95% -> 100%; math does not allow it. | **Partly true** | For arbitrary programs, undecidability and path explosion block universal semantic certainty. Within restricted formalisms/specs, 100% proofs are possible. |
| We can map all internal function paths with 100% fidelity up to external boundaries. | **Overstated** | Dynamic dispatch, reflection, generated/runtime-loaded code, and metaprogramming reduce static certainty even inside a repo. |
| AI output can be deterministically challenged so only structurally/contractually valid changes land. | **Partly true** | Strongly true for syntax/types/contracts/policy boundaries; limited by quality/completeness of the encoded contracts. |
| Deterministic refactoring can split a monolith into many files with identical behavior (no LLMs). | **Partly true** | Feasible with AST-based codemods and transformation systems in bounded scopes; full-system equivalence remains hard with side effects/concurrency/I/O. |
| Deterministic constraints let us rely on cheaper models instead of chasing flagship power. | **Partly true** | Guardrails reduce model burden, but hard architecture and ambiguous intent still benefit from stronger models and human review. |

## Technical Interpretation

### What is feasible now

1. **Deterministic "conical docking" around AI output**
   - Constrain output shape (schemas/grammars/AST transforms).
   - Enforce architecture contracts (imports/dependency boundaries).
   - Gate changes with type checks, static analysis, and targeted tests.

2. **Deterministic refactor pipeline**
   - Parse -> transform -> render -> verify.
   - Use codemods for repeatable structural moves (file splits, API renames, call-site rewrites).
   - Validate behavior with differential/regression tests and mutation checks.

3. **Cheaper-model strategy under hard gates**
   - Use lower-cost models for candidate generation.
   - Escalate to stronger models only when constraints fail or design ambiguity is high.

### Boundary conditions (where certainty degrades)

- Dynamic language features: reflection, eval, monkey patching, runtime imports.
- Dispatch uncertainty: polymorphic call targets and plugin systems.
- Environmental/state effects: clocks, randomness, network, filesystem, DB transactions.
- Distributed semantics: retries, timeouts, eventual consistency, race conditions.
- Spec incompleteness: if intent is not formalized, "provable" only covers what is formalized.

## Practical Architecture Pattern

### Layer 1: Structural determinism
- AST parseability
- formatter/linter consistency
- import boundary contracts
- type checker clean

### Layer 2: Behavioral guardrails
- contract/unit tests
- differential tests (before/after transforms)
- mutation testing where feasible

### Layer 3: Policy and risk controls
- deny unsafe patterns (forbidden APIs, boundary violations, missing telemetry)
- confidence score + review routing
- required escalation for cross-boundary or high-risk changes

### Layer 4: Human intent preservation
- narrow, explicit change request format
- machine-checkable acceptance criteria
- semantic review only where formal checks are insufficient

## Relevance to This Repo (FileScience)

This repo already contains several deterministic controls aligned with the approach:
- Polylith import contracts via `import-linter`
- post-edit quality hooks (`ruff`, `ty`, import boundary checks)
- CI quality gates and increasing test rigor

High-value next increment, if pursued:
- Add a formal "change contract" format per task (inputs, allowed files, invariants, verification checks).
- Add deterministic codemod playbooks for common refactors.
- Add a fail-closed policy for risky changes (cross-component boundary moves, throttling logic, infra auth paths).

## Suggested Experiments

1. **Monolith split trial (deterministic only)**
   - Pick one medium-complexity module.
   - Execute codemod-driven split with no LLM code synthesis.
   - Require differential test parity.

2. **Cheap-model + hard-gates trial**
   - Run same task with lower-cost model vs higher-cost model.
   - Compare pass rate through deterministic gates, rework cycles, and total token cost.

3. **Intent-to-contract pilot**
   - Convert one feature request into machine-checkable acceptance criteria.
   - Measure how often AI proposals pass first-gate verification.

## Bottom Line

The concept is strategically sound when framed as **"deterministic guardrails over probabilistic generation"** rather than **"universal proof of correctness."**  
The best practical target is not absolute certainty; it is a measurable increase in reliability-per-token and lower day-2 maintenance risk.

## Sources

1. Rice's theorem lecture notes (MIT OCW):  
   https://ocw.mit.edu/courses/6-045j-automata-computability-and-complexity-spring-2011/7aead2c728dd3d5a737d832811ef97e6_MIT6_045JS11_lec09.pdf

2. Abstract Interpretation (Cousot & Cousot, 1977):  
   https://www.di.ens.fr/~cousot/COUSOTpapers/POPL77.shtml

3. CodeQL call graph guidance (dispatch and dynamic limits):  
   https://codeql.github.com/docs/codeql-language-guides/navigating-the-call-graph/

4. Missing edges in JavaScript call graphs (empirical evidence):  
   https://arxiv.org/abs/2205.06780

5. Dafny reference manual (machine-checked contracts/proofs):  
   https://dafny.org/latest/DafnyRef/DafnyRef

6. CompCert (semantics-preserving verified compiler):  
   https://compcert.org/

7. KLEE symbolic execution (power + path explosion constraints):  
   https://llvm.org/pubs/2008-12-OSDI-KLEE.pdf

8. Alive2 translation validation (bounded optimization correctness):  
   https://users.cs.utah.edu/~regehr/alive2-pldi21.pdf

9. Coccinelle semantic patches (deterministic refactors):  
   https://coccinelle.gitlabpages.inria.fr/website/rules/

10. OpenRewrite deterministic large-scale refactoring:  
    https://docs.openrewrite.org/

11. Structured constrained outputs (schema-constrained generation):  
    https://openai.com/index/introducing-structured-outputs-in-the-api/
