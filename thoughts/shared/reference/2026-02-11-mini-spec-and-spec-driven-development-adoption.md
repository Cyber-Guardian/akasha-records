# Mini-Spec + Spec-Driven Development Adoption Notes (2026-02-11)

## Question
Should FileScience adopt the `zot/mini-spec` flow, and should specs be retained/updated over time?

## Short Answer
- **Adopt partially (recommended):** use a **spec-anchored** workflow for medium/large features and cross-service behavior, not for every small fix.
- **Yes, retain and update specs:** if specs are discarded after implementation, the workflow collapses back to plan-first coding and drift returns.
- Keep current implementation plans as execution artifacts, but promote stable behavior/contracts into durable specs and design artifacts.

## What Mini-Spec Actually Adds

From the repo and skill/tool docs:
- Three-level structure: `specs/` (intent) -> `design/` (interpreted design) -> `src/` (implementation).
- Explicit phase split: design first, then implementation.
- Drift control: artifact checkboxes + traceability comments + validation commands.
- Tooling support (`minispec`) for mechanical updates:
  - requirement coverage queries
  - artifact check/uncheck
  - gap tracking
  - phase validation

## Strengths
- Prevents hidden scope creep by forcing design review before code.
- Makes requirement/design/code traceability inspectable, not implicit.
- Gives a repeatable method to detect and correct drift when code changes.
- Fits AI-assisted workflows where generation can add unplanned behavior.

## Risks / Costs
- Can be heavy for small bugfixes and low-risk internal changes.
- Document overhead can become review overhead if applied indiscriminately.
- Requires team discipline: if checkboxes/specs are not maintained, process decays quickly.
- The full CRC/sequence/doc stack may be more than needed for many backend-only tasks.

## Research View: What "Spec-Driven" Means in Practice

A useful breakdown:
- **Spec-first:** write a spec before coding.
- **Spec-anchored:** keep and evolve specs after release.
- **Spec-as-source:** specs are the primary maintained artifact; code is derivative.

Most teams get value at **spec-anchored**. Going fully spec-as-source is high ceremony and still experimental.

## Answer To "Are We Supposed To Retain And Update Specs?"

**Yes.** In living-documentation/spec-by-example workflows, the core value comes from specs remaining aligned with behavior. If they are ephemeral, you lose:
- reliable intent history
- drift detection
- trustworthy onboarding docs
- regression guardrails at the requirement level

So: plans may be ephemeral; specs should be durable.

## Recommended Policy For This Repo

### 1) Two artifact types (explicit)
- **Execution Plans (ephemeral):** phase-by-phase implementation docs under `memory-bank/thoughts/shared/plans/`.
- **Feature Specs (durable):** retained artifacts under a stable spec/design location (either mini-spec folders or equivalent).

### 2) Trigger for durable specs
Create/maintain durable specs when any of these are true:
- behavior spans multiple components/services
- user-visible or compliance-relevant behavior
- non-trivial retry/throttle/error semantics
- feature expected to evolve over multiple iterations

### 3) Definition of Done addition
For work with durable specs:
- requirement changed -> update spec
- behavior/architecture changed -> update design artifact
- implementation changed independently -> mark drift and resolve before closeout

### 4) Keep mini-spec lightweight
- For small tasks: allow "spec-lite" (1 page requirements + acceptance checks) instead of full CRC suite.
- For large features: use full requirements + design + gap validation.

## Practical Adoption Path (Low Risk)

1. Pilot on one medium feature area for 2-3 cycles (not everywhere).
2. Measure:
   - time-to-implement
   - review clarity
   - post-merge drift incidents
   - "spec updated with code?" compliance
3. Keep if signal is positive; otherwise keep only the parts that help (likely requirement mapping + drift checks).

## Bottom Line
- Mini-spec concepts are strong for your current pain ("plans become stale after implementation").
- Full strict methodology on every change is likely too heavy.
- Best fit here is a **hybrid, spec-anchored model**: durable specs for important behavior, ephemeral plans for execution.

## Sources
- mini-spec repository and README  
  https://github.com/zot/mini-spec
- mini-spec skill definition  
  https://raw.githubusercontent.com/zot/mini-spec/main/.claude/skills/mini-spec/SKILL.md
- mini-spec user manual (CLI workflow and validations)  
  https://raw.githubusercontent.com/zot/mini-spec/main/tool/docs/user-manual.md
- Thoughtworks Technology Radar: Spec-driven development (Assess, Nov 2025)  
  https://www.thoughtworks.com/radar/techniques/spec-driven-development
- Martin Fowler / Thoughtworks analysis of SDD levels  
  https://martinfowler.com/articles/exploring-gen-ai/sdd-3-tools.html
- Cucumber BDD documentation (collaborative executable documentation)  
  https://cucumber.io/docs/bdd
- Serenity BDD living documentation model  
  https://serenity-bdd.github.io/docs/reporting/living_documentation
- IEEE/ISO/IEC 29148 standard entry (requirements engineering lifecycle management)  
  https://standards.ieee.org/standard/29148-2018.html
