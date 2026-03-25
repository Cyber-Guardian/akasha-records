---
title: "Brain Dump Distillation — Top 5 Ideas Validated"
created: 2026-03-12
type: brief
status: validated
tags: [strategy, architecture, product, pmf, brainstorm]
---

# Brain Dump Distillation — Top 5 Ideas Validated

Filtered ~100 raw ideas from Slack brain dump into 5 high-signal themes based on repetition (conviction signal), leverage, and novelty. Each validated via parallel deep research.

---

## Verdict Summary

| # | Idea | Verdict | Confidence | Priority |
|---|------|---------|------------|----------|
| 1 | **PMF Diagnosis** | Distribution problem, not product problem | High | **Do first** |
| 2 | **Contract-First Parallel Dev** | Walking skeleton + Polylith fake impls = natural fit | High | Shape next |
| 3 | **FSM-Constrained Agents** | Established pattern, your framing is cutting-edge consensus | High | Build into triage agent + Helm |
| 4 | **Event-Based Backbone** | Natural fit for backup domain. DynamoDB event store + thin DIY layer | Medium-High | Shape after PMF clarity |
| 5 | **Integration Factory** | Validated in ETL, unproven in backup. 4-6mo MVF | Medium | Defer until v2 architecture settles |

---

## 1. PMF Diagnosis — "Why Isn't It Selling?"

**Verdict: It's channel + positioning, not product.**

### The structural problem
- 74% of MSPs consolidating vendors → backup is bundled into Datto/Kaseya MSP contracts
- Law firms believe they already have backup (through their MSP or Clio itself)
- No organic search intent for "cloud backup for law firms" → CPC fights structural reality
- Purchase triggers are reactive: ransomware, cyber insurance renewal, malpractice insurer, peer firm data loss
- No legal-specific backup competitor exists — either opportunity or the economics don't work standalone

### Root causes (ranked by likelihood)
1. **Wrong channel for the ICP** — SMB law firms are reached through MSPs, Clio Certified Consultants, bar association tech committees, peer referral. Not Google ads.
2. **ICP too broad** — "Clio users" is 150K+ firms. Need trigger-based ICP: firms with upcoming cyber insurance renewal, firms in states with new data protection ethics guidance, firms whose MSP doesn't cover Clio backup.
3. **Positioning mismatch** — "Set it and forget it" convenience positioning vs. fear-based "your practice is one sync error away from losing everything" positioning for triggered buyers.
4. **Integration anchor may be partial** — If attorney's real risk is in M365 (email, docs) but FileScience is protecting Clio (cases, contacts), the perceived value is "nice to have" not "must have."

### 6-step diagnostic protocol
1. **Sean Ellis test** on existing customers (or structured interviews if <40 active users)
2. **Interview 10 lost deals** — "What did you do instead?" + "What would have had to be true?"
3. **Map trigger events** in closed-won customers — what happened that made them look?
4. **Test MSP channel seriously** — 5-10 legal MSPs, 30-40% ARR margin, 90-day pilot
5. **Test trigger-based outbound** — cyber insurance renewals, bar ethics guidance updates, breach notifications
6. **Evaluate integration breadth** — is Clio-only creating a "partial protection" perception?

### Key competitive insight
The market consolidated through MSP-centric acquisitions (Datto/Kaseya own Spanning + Backupify). FileScience's defensible angle is Clio-native + legal vertical positioning — no one else has it. Risk: space may be too small, or Clio builds it natively.

---

## 2. Contract-First Parallel Development (Walking Skeleton)

**Verdict: Strongly validated. Polylith natively supports this.**

### What it is
Build executable shells for all v2 services (discover, transfer, recovery, dashboard) with defined interfaces. Write e2e tests against contracts first. Fill in implementations in parallel. For debugging: compose system with dummy components, swap in real ones to isolate faults.

### Key findings
- **Walking skeleton** is the established name (Cockburn, 2000). Must be executable end-to-end, not just interface files.
- **Polylith natively supports fake implementations** — `cloud_api_fake` alongside `cloud_api`, wired at project-build time. First-class feature, not a workaround.
- **Pact** for consumer-driven contract testing. Async Lambda messages use Pact message contracts. pact-python v4 uses Rust FFI core.
- **Ports and Adapters** — split Lambda entrypoint (base) from domain logic (component). Pact tests at the component interface boundary.
- **AWS Hexagonal Architecture** prescriptive guidance confirms the pattern for Lambda specifically.

### Failure modes to watch
- Schemas are not contracts (PactFlow) — must be consumer-expressed, not provider-defined
- Skeleton must be deployable and produce observable results from external entry point
- Fake component maintenance cost — keeping `cloud_api_fake` aligned with `cloud_api` over time
- Contract tests pass but integration still fails on IAM, throttling, cold starts → need both contracts + integration tests

### How it maps to your codebase

| Layer | Contract mechanism | Parallelism enabler |
|---|---|---|
| `components/` | `interface.py` per brick | Fake implementations (`cloud_api_fake`) |
| `bases/` | Event schema (SQS/SNS message structure) | Pact message contracts |
| `projects/` | Integration tests against real AWS | Ephemeral stacks per feature |
| CI | `can-i-deploy` gate | Pact Broker as coordination hub |

---

## 3. FSM-Constrained Agent Architecture

**Verdict: Strongly validated. You're arriving at the emerging production consensus independently.**

### What it is
State machines gating AI agent decisions. At any state, agent can only choose from valid transitions. Symbolic reasoning determines WHICH transitions are valid. LLM chooses WHICH valid transition to take and HOW. Structured subagents, open main agent.

### Key findings
- **StateFlow** (Microsoft/AutoGen): 13-28% higher task success, 3-5x lower cost than ReAct
- **MetaAgent** (ICML 2025): auto-generates FSM-controlled multi-agent systems from task descriptions
- **FARS**: FSM-constrained API selection eliminates 90%+ of API hallucinations
- **The "Hybrid Loop" pattern** is what practitioners are converging on: open Planner (frontier LLM) + deterministic Supervisor (FSM owns state/transitions) + constrained Worker agents
- **This is different from guardrails** — guardrails are post-hoc filtering; FSM grounding is pre-hoc structural constraint. The agent never considers invalid actions.

### Tradeoffs
- Helps: deterministic domains, safety-critical, cost reduction (5x), auditability
- Hurts: open-ended creative tasks, over-specification, context limits from complex FSM descriptions

### How to apply
- Triage agent: FSM already implicit in state transitions → make explicit
- Helm subagents: structured workers with FSM constraints, open orchestrator
- Production agents: use the "intelligent deterministic behavior" pattern

### Key frameworks
LangGraph (graph nodes = states), AutoGen/StateFlow (FSM as first-class primitive), Guidance/Outlines (token-level constraints), Temporal.io (durable state machines)

---

## 4. Event-Based System Backbone

**Verdict: Validated. Event sourcing is one of backup's strongest use cases.**

### What it is
Proper event-sourced backbone under discover, transfer, recovery. DynamoDB as event store, Streams + fanout Lambda + EventBridge as publication bus. Existing Valkey queue becomes a materialized projection.

### Key findings
- Backup domain = "what changed and when" → natural event sourcing alignment
- **Architecture**: DynamoDB event ledger (PK=aggregateId, SK=version) → DynamoDB Streams → fanout Lambda → EventBridge → projector Lambdas / Valkey queue / S3 archive
- **Skip the `eventsourcing` Python library** — designed for long-lived processes, not Lambda cold starts. Use thin DIY layer: `EventStore` class with `append_event`, `load_events`, `save_snapshot`, `load_snapshot`.
- **"Python first, Rust later"** is well-established via PyO3 + maturin. 81x cost reduction on CPU-bound Lambda. Best candidates: snapshot computation, integrity hashing, event serialization. Keep orchestration in Python.
- **Separate event classes**: system events (what your platform did) vs source events (what happened in customer's cloud). Both in event store, keyed differently.

### Critical prerequisite
**Estimate event volume before committing.** Write amplification matters. 200GB/day case study (IoT) is cautionary. Calculate events-per-entity-per-day → project DynamoDB write capacity and storage costs.

### Migration path
Strangler fig: run both architectures in parallel, migrate consumers one at a time. Valkey queue becomes projection artifact, not source of truth.

---

## 5. Integration Factory

**Verdict: Validated in ETL/analytics, unproven in backup. You'd be first.**

### What it is
Automated pipeline for building new cloud integrations. Standardized connector framework where new integrations take days instead of months.

### Key findings
- **Airbyte/Fivetran/Nango** prove the factory pattern at scale. Airbyte went from months to <30min per connector with their Low-Code CDK.
- **The abstraction**: `Requester` (HTTP) + `RecordSelector` (extraction) + `Paginator` (strategy) covers ~80% of every connector. 20% needs custom escape hatches.
- **No backup company has built a factory** — all build integrations custom (Veeam, Spanning, Backupify). Either opportunity or there's a reason.
- **Backup-specific challenges**: deletion tracking (mandatory, provider windows: Dropbox 120d, M365 90d, Google 30d, Box 14d), version history depth, change detection APIs differ radically per provider, OAuth scopes require elevated permissions.
- **AI-assisted building** works for the 80% (auth, pagination) but not the backup-specific 20%.
- **Gold/Silver/Bronze tiering** is validated (Airbyte certified/community, Databricks medallion). Risk: customers using Silver for critical backup, discovering quality gap at restore time.
- **Realistic MVF timeline**: 4-6 months.

### Recommended sequence
1. Extract abstraction from existing 3 integrations (2-4 months)
2. Migrate existing connectors onto framework (1-2 months)
3. First net-new connector on framework (1-2 months)
4. Tooling, observability, partner SDK (ongoing)

---

## What Got Filtered Out (Noise)

**Personal tasks**: email biz dev guy, print forms, workout, NYC apt, London trip
**Already done**: ground command, deep research, Hypothesis RBSM, CI improvements, Linear restructure, triage agent
**Too far out**: digital twin for meetings, gameification, AI incubator rev share model, search engine from ground up
**Too vague**: Kubernetes, Supabase, frontend-to-DB direct
**Subsumed by top 5**: R&D lab (subsumed by event backbone), Rust backbone (subsumed by Python-first-Rust-later), URL testing (subsumed by contract testing), cost telemetry (good idea but tactical, not strategic)

---

## Recommended Sequence

1. **PMF Diagnosis** — Before building more, understand why existing product isn't selling. 2-3 weeks of customer interviews + lost deal analysis. This determines everything else.
2. **Walking Skeleton for v2** — Shape the contract-first architecture. Leverage Polylith's native fake-implementation support. Write e2e tests first.
3. **FSM-Constrained Agents** — Apply to triage agent and Helm subagents now (incremental, no big rewrite).
4. **Event Backbone** — Shape after PMF diagnosis gives clarity on what v2 needs to optimize for.
5. **Integration Factory** — Defer. Depends on v2 architecture settling + knowing which integrations matter most from PMF work.

---

## Related Artifacts
- [[2026-03-11-shapes-brainstorm-best-ideas|Shapes Brainstorm Synthesis]] — product ideas scored and tiered
- [[2026-03-11-bizdev-operating-system|BizDev Operating System Brief]] — shaped, ready for planning
- [[2026-02-28-macro-agent-strategy|Macro Agent Strategy]] — agent orchestration roadmap
- [[2026-02-27-ddd-refactor-test-investment|DDD Refactor Test Investment Plan]]
