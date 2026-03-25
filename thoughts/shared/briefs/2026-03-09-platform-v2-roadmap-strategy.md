# Idea Brief: Platform V2 Roadmap Strategy

**Date:** 2026-03-09
**Status:** Shaped → Planning (Pass 1 pending)

## Problem

FileScience has a well-structured Linear workspace (5 initiatives, 15+ projects) and a detailed Macro Agent Strategy, but no coherent framework connecting "what to build" (product) with "how to build it" (agentic process). Jordan oscillates between product work, meta-process work, and strategic work — nothing gets sustained momentum. The existing initiatives are organized by product area, not by work mode, so there's no answer to: "What in-loop work do I do today that maximally unblocks out-loop product delivery tomorrow?"

## Core Model: Two Types of Intelligence

**Validated by research (see grounding section below).**

- **Humans are divergent thinkers** — architecture, taste, high-leverage decisions, novel problem framing
- **Agents are convergent thinkers** — pattern execution, extending established patterns, testing, bounded implementation
- **In-loop work (codev):** Human + AI together. High quality, slower. Two sub-paths:
  - **Foundation:** Architecture decisions, contracts, building blocks that agents implement against
  - **Meta:** Agent infrastructure, quality gates, swarm harness that makes agents reliable
- **Out-loop work (autonomous):** Agents following established patterns independently. For product delivery at scale.
- **The goal:** Progressive outsourcing — each in-loop investment unlocks more out-loop throughput. Humans focus increasingly on divergent chokepoints only.

## Constraints

- Team: Jordan (v2 monorepo), Niklas + Harsh (v1 legacy platform, transitioning later)
- Polylith monorepo with 7 import-linter contracts already enforcing boundaries
- 3 services operational (Discover, EDT, VQP), 0 services for Transfer/Recovery/Export/Dashboard
- Macro Agent Strategy exists (5-phase plan) but focused on meta — needs integration with product delivery
- Clio Launch target June 2026 (Niklas/Harsh on v1, not blocking this roadmap)
- Dashboard/API will live in this monorepo (Next.js frontend + OpenAPI contracts)

## Research Grounding

Three parallel research threads validated the core ideas (2026-03-09):

| Claim | Evidence Grade | Key Source |
|-------|---------------|------------|
| Agents are convergent, humans divergent | A (multiple RCTs) | CHI 2025 (n=1,100), Science Advances RCT, systematic review (137 studies) |
| In-loop → out-loop pipeline | A (convergent evidence) | Anthropic autonomy data, FeatureBench (74% → 11%), IBM bounded agents |
| Skeleton/spec-driven architecture | B+ (strong practice, thin controlled) | AWS Kiro, GitHub Spec Kit, MetaGPT benchmarks, hexagonal architecture tracked data |
| Factory model with quality gates | B+ (strong case studies) | Osmani (Google), OpenObserve 8-agent council, JM Family deployment |
| Testing-first as guardrails | A (production data) | Meta mutation testing, Simon Willison red/green TDD, OpenObserve case study |
| DDD bounded contexts = agent scope | B (benchmarks + practitioner) | MetaGPT ablation study, Event Storming for agent boundaries |
| Kubernetes for this team size | C (not recommended) | Industry consensus: 1-10 devs → modular monolith. Polylith already provides modularity. |
| Progressive outsourcing | A- (frameworks + warnings) | Shapiro 5 levels, ASDLC, but Anthropic skill degradation RCT (17% lower scores) |

Key warnings from research:
- AI-generated tests achieve only 20% mutation scores — high coverage =/= fault detection (mutation testing is critical)
- Experienced devs were 19% SLOWER with AI on complex tasks but believed they were faster (METR RCT)
- Repos adopting agents without preparation saw +39% code complexity (agent-induced complexity debt)
- Human-in-the-loop must be genuinely thinking, not rubber-stamping (Anthropic skill formation study)

## Options Considered

### Option A: Waterfall Layers
Build each layer to completion before starting the next. Understand → Test → Skeleton → Agent Infra → Product.
- Gains: Each layer is solid before building on it
- Costs: Extremely slow to first product output. Learnings from later layers can't feed back. Risk of over-engineering early layers.
- Complexity: Medium (simple sequencing)

### Option B: Spiral Passes
Execute all layers in iterative passes. Each pass refines all layers and shifts the in-loop/out-loop ratio toward more autonomy. Pass 1 is rough sketch; Pass 2 expands; Pass 3+ is production.
- Gains: Fast feedback loops. Each pass validates assumptions from the previous one. Learnings flow backward. Time-boxed.
- Costs: Requires discipline to not gold-plate early passes. Risk of incomplete layers if passes aren't scoped well.
- Complexity: Medium-High (need clear pass exit tests)

### Option C: Product-First with Retroactive Foundation
Just start building Transfer/Recovery/Dashboard and extract patterns afterward.
- Gains: Fastest path to product output
- Costs: Agent-induced complexity debt (+39% per research). No quality gates means unreliable agent output. Rebuilding foundations after the fact is more expensive than building them first (Fortune 500 auth module cautionary tale).
- Complexity: Low initially, High later (technical debt)

## Chosen Approach

**Option B: Spiral Passes** — because the research strongly supports "foundation must lead, but not by much." Each pass produces something testable. The spiral lets learnings flow backward without the rigidity of waterfall or the chaos of product-first.

### The Layer Model

```
Layer 0: UNDERSTAND — Domain model, V1 retro, bounded context map (pure divergent human work)
Layer 1: QUALITY MACHINERY — Mutation testing, E2E test contracts, quality gates
Layer 2: ARCHITECTURE SKELETON — Interface definitions, mock implementations, OpenAPI contracts
Layer 3A: REFERENCE IMPL — One service end-to-end (Discover V2) as the pattern exemplar
Layer 3B: AGENT FACTORY — Helm, quality gates, swarm harness, agent scope enforcement
Layer 4: PARALLEL PRODUCT DELIVERY — Agents fill in skeletons across services
Layer 5: PROGRESSIVE AUTONOMY — Nightly agents, swarm execution, humans at divergent chokepoints
```

### Pass Structure

**Pass 1 (Foundation Sketch, ~2-3 weeks):**
- L0: Solo domain model draft + V1 codebase analysis
- L1: Fix mutation testing, E2E test skeletons for Discover + Transfer
- L2: Skeleton interfaces for Discover (alignment) + Transfer (new shell)
- L3B: Minimal agent infra improvements
- Exit test: Can an agent pick up a bounded task, implement it, pass mutation testing gate?

**Pass 2 (Expand + Refine, ~3-4 weeks):**
- L0: Refine domain model, Event Storming with team
- L1: E2E contracts for all services
- L2: Skeleton for all v2 services + Dashboard + OpenAPI contracts
- L3A: Discover V2 pushed further as reference implementation
- L3B: Helm improvements, quality gates hardened
- Exit test: Can agents work on 2-3 services in parallel with quality gates catching real problems?

**Pass 3+ (Factory Production):**
- L4: Agents fill in implementations across all services
- Remaining in-loop: architecture decisions at boundaries, divergent thinking
- Ratio shifts from 80% in-loop to 80% out-loop

### Open Questions

- **Macro Agent Strategy integration:** The existing 5-phase plan (visualization, helm, tournaments, swarm, memory-bank OS) — does it fold into L3B within each pass, or does some of it (visualization for human review) become a parallel track? Deferred to Pass 1 learnings.
- **Pass 1 depth for L0:** Full Event Storming with team or lighter solo draft first? Currently leaning solo draft (Pass 1) → team session (Pass 2).
- **"Ready for out-loop" threshold:** What reliability metrics determine when a layer is solid enough? Mutation score > 70%? Agent PR merge rate > 80%?

## Key Context Discovered During Shaping

- Existing Linear has 5 initiatives organized by product area, not work mode — needs restructuring or augmentation
- FeatureBench shows same model goes 74% → 11% between bounded and unbounded tasks — the skeleton is what makes tasks bounded
- MetaGPT benchmarks: structured roles + bounded scope = 2x executability, 3x less human revision (peer-reviewed, causal)
- Polylith + import-linter already provides the "enforced boundaries" that research identifies as the key mechanism (compiler-enforced > documented)
- Osmani's factory model maps manufacturing principles (spec precision, quality gates, standardized interfaces, station isolation) to AI agent development
- Anthropic data: experienced users grant more autonomy but interrupt MORE (9% vs 5%) — the model is "human on the loop," not "human out of the loop"

## Related Artifacts

- [[2026-02-28-macro-agent-strategy|Macro Agent Strategy]] — 5-phase agentic capabilities plan (integrates at L3B/L5)
- [[2026-02-19-linear-workspace-restructure|Linear Workspace Restructure]] — current initiative/project structure
- [[2026-02-27-ddd-micro-architecture|DDD Micro-Architecture Plan]] — existing DDD plan (feeds into L0)
- [[2026-03-04-memory-system-v2-rearchitecture|Memory System v2]] — memory infrastructure (relates to L3B)

## Linear Restructure (Decided 2026-03-09)

### 3 Initiatives (down from 5)

| Initiative | Strategic Bet | Absorbs |
|---|---|---|
| **V2 Platform** | Build the next-gen product | Current V2 Platform + Dashboard Evolution |
| **Go to Market** | Acquire & retain customers on current platform | Clio Launch + Landing Site & Growth |
| **Engineering Leverage** | Build the factory that lets 3 engineers produce like 20 | DevOps & Quality + Agent Orchestration + Helm Orchestrator |

"Engineering Leverage" chosen over: "Development System", "Agentic Engineering", "The Software Factory" — grounded against industry naming (Block DEVIQ, Osmani "Agentic Engineering", Bessemer "Software 3.0", Factory.ai). Most investor-legible, implies compound returns.

### Projects per Initiative

**V2 Platform:** Domain Model & Architecture (NEW), V2 Services (NEW — services as milestones: Discover, Transfer, Recovery, Export), V2 Components (NEW — shared backbone library layer), Dashboard V2 (NEW, Paused until Pass 2)

**Go to Market:** Clio Marketing, Dashboard v1.5, Landing Site v2.2

**Engineering Leverage:** Testing Infrastructure (NEW), Helm Orchestrator

### Labels (Cross-Cutting)
- Work Mode: `in-loop`, `out-loop`
- Execution: `co-dev`, `autonomous`
- Path: `foundation`, `meta`, `product`
- Pass: `pass-1`, `pass-2`, `pass-3`

### Archive
9 Idea-state projects from 2024-2025 with no issues or recent activity.

### Visual
See `2026-03-09-roadmap-visual.html` for full interactive breakdown.

### Project Milestones (Decided 2026-03-09)

**V2 Platform — Domain Model & Architecture:**
1. Product & Market Discovery — customer feedback, competitive analysis, ICP validation, product hypotheses
2. V1 Retrospective — technical + product postmortem, informed by customer data
3. Bounded Context Map — domain model grounded in both technical reality and product priorities
4. Service Contracts — port definitions, OpenAPI specs, shared types, prioritized by product value
5. Skeleton Green — mock implementations pass E2E contracts, full system composes

**V2 Platform — Discover V2:**
1. DDD Layers — domain/application/infrastructure enforced by import-linter (ENG-2190)
2. Skeleton Aligned — implements skeleton spec Protocols/ports
3. Reference Complete — documented exemplar agents can replicate
4. Production Stable — full E2E, deployed, monitoring, delta sync all clouds

**V2 Platform — Transfer V2:**
1. Skeleton & Contracts — port definitions, mocks, E2E contracts
2. Agent Implementable — agent picks up bounded task, passes mutation testing (Pass 1 exit test)
3. Core Implementation — real implementations, unit tests
4. Production Ready — integration tests, deployed, monitoring

**V2 Platform — Dashboard V2 (Pass 2):**
1. Stack & Scaffold — Next.js app, component library, design system
2. API Contracts — OpenAPI specs bridging backend and dashboard
3. Core Views — overview, clouds, organization pages functional
4. Beta — auth, core flows, user-testable

**Engineering Leverage — Testing Infrastructure:**
1. Mutation Gate Live — pipeline fixed, hard gate at 70% score
2. E2E Contract Framework — pattern established, Discover + Transfer covered
3. Full Service Coverage — all active v2 services covered
4. Agent Validation Protocol — agents self-test, mutation score meets threshold

**Engineering Leverage — Helm Orchestrator:**
1. V1: Execution Layer (DONE)
2. V1.5: Production Hardening — telemetry, rescoping, quality feedback (ENG-2259/2260/2261)
3. V2: Design Layer — tournament ideation, intelligence allocation, plan-from-issue
4. V3: Custom Runtime — self-hosted agent runtime (deferred)

**Engineering Leverage — Agent Quality & Processes:**
1. Discipline Baseline — scope enforcement, CI feedback loop, conventions documented
2. Bounded Task Protocol — defined what "hand-offable" means, tested with real agent
3. Quality Scorecard — merge rate, mutation score, revision count tracked
4. Progressive Autonomy Framework — graduated trust levels, automated promotion

**Go to Market** — milestones defined by Niklas/Harsh for their projects.

## Next Steps

1. ~~Execute Linear restructure~~ — DONE (2026-03-10)
2. `/create_plan` for Pass 1 specifically
