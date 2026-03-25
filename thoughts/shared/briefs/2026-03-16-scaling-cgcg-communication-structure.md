---
title: "Scaling CGCG: Communication, Structure & Company Modeling"
type: state
status: active
created: 2026-03-16
tags: [organizational-design, slack, scaling, m&a-integration]
---

# Scaling CGCG: Communication, Structure & Company Modeling

## Context

Cyber Guardian Consulting Group (CGCG) has grown from ~15 to ~30 employees in two years through organic growth and MSP acquisitions (MCS, HRT). CEO projects 50-60 employees in the near term. The company operates as a multi-entity organization:

- **CGCG** — parent company; everyone is technically a CGCG employee
- **FileScience** — software product division
- **MCS** — acquired MSP
- **HRT** — acquired MSP

The current operational infrastructure — particularly communication tooling — was built for a 15-person FileScience team and hasn't been redesigned for the multi-entity, 30+ person reality. This brief frames the problem, explores structural options, and recommends a direction.

---

## The Problem: Three Scaling Walls

### Wall 1 — Communication Breakdown (15→30, hitting now)

At 15 people, communication is ambient. Everyone hears everything. At 30, this breaks:

- **Information asymmetry** — acquired teams (MCS, HRT) don't have the same context as FileScience veterans
- **Signal-to-noise collapse** — important announcements get buried in general chatter
- **No escalation path** — urgent cross-team issues travel through DMs and tribal knowledge
- **Onboarding friction** — new hires can't discover what channels matter or who owns what

**Evidence from current Slack:**
- 16 `fs-*` channels, 0 `mcs-*` channels, 1 `hrt-*` channel — acquired teams are invisible in the communication system
- No announcement layer — `#general` does everything
- No escalation channel — urgent issues have no dedicated home
- Alert channels mixed with discussion channels under the same `fs-` prefix

### Wall 2 — Structural Ambiguity (30→50, coming fast)

The org hasn't decided how entities relate to each other:

- Are MCS/HRT permanent brands or being absorbed into CGCG?
- Is there one service delivery operation or three?
- Who owns client relationships — the acquiring entity or CGCG?
- Is engineering centralized (FileScience) or distributed?

These aren't just org chart questions — they cascade into every operational decision: tooling, processes, hiring, client communication, and how people identify themselves.

### Wall 3 — Operating Model Gap (50→60+, must plan now)

Every ~2x in headcount breaks the previous operating model. The shift from 30 to 60 requires:

- **Middle management layer** — CEO can't directly manage everyone; first-time managers need support
- **Formalized departments** — sales, ops, engineering, support become real teams, not hats
- **Written processes** — what worked verbally at 15 must be documented at 50
- **Deliberate culture** — with multiple acquired cultures merging, "the way we do things" must be explicitly defined, not assumed

---

## The Deeper Question: How Do We Model Our Companies?

Before fixing Slack or any other tool, CGCG needs to answer a fundamental structural question: **What is the relationship between the parent company and its entities?**

### Model A — Holding Company (Federated)

Each entity keeps its identity, operations, and client relationships. CGCG provides shared services (finance, HR, compliance) but doesn't unify operations.

| Pros | Cons |
|------|------|
| Preserves acquired brand equity | Duplicate overhead across entities |
| Easier M&A integration (less change) | Harder to cross-sell or share clients |
| Teams keep their identity | "Us vs. them" culture risk |
| Simple when entities serve different markets | Doesn't scale cost-efficiently past 60-80 |

**Communication pattern:** Entity-prefixed channels (`fs-*`, `mcs-*`, `hrt-*`) with thin CGCG-wide overlay.

### Model B — Unified Operating Company

All entities merge into CGCG. One brand, one ops team, one client book, one engineering org. Acquired brands sunset or become product lines.

| Pros | Cons |
|------|------|
| One operating model, one culture | Disruptive to acquired teams and their clients |
| Efficient at scale (no duplication) | Risk of losing acquired talent who identified with their MSP |
| Clear career paths and org structure | Takes 12-18 months to truly unify |
| Clients see one company | Client contracts and relationships need re-homing |

**Communication pattern:** Function-prefixed channels (`team-*`, `ops-*`, `alert-*`) with no entity namespacing.

### Model C — Converging Platform (Recommended)

Start federated, converge intentionally over time. CGCG defines the shared platform (culture, processes, tools). Entities converge at their own pace based on operational readiness, not an arbitrary timeline.

| Pros | Cons |
|------|------|
| Realistic — meets people where they are | Requires active management of the convergence roadmap |
| Reduces integration risk | Temporary complexity of hybrid state |
| Allows learning from each entity's strengths | Need clear "converged" end-state to avoid permanent limbo |
| Can accelerate or slow per entity | |

**Communication pattern:** Hybrid — CGCG-wide channels for unified functions + entity channels that merge as operations unify.

**Convergence sequence:**
1. **Finance & compliance** unify first (already happening — billing, SOC2)
2. **Hiring & HR** unify next (one employer, one process)
3. **Sales** converge when go-to-market strategy aligns
4. **Service delivery** converges last (most client-facing, highest risk)

---

## Slack as the First Concrete Problem

Slack is the canary in the coal mine. It's the most visible symptom of the structural questions above. Here's the current state and recommended direction:

### Current State Audit

| Metric | Value | Assessment |
|--------|-------|------------|
| Total channels | ~53 | Reasonable for 30, needs pruning/restructuring for 50 |
| FileScience channels | 16 | Over-represented, doing too many jobs |
| MCS channels | 0 | Critical gap — acquired team has no home |
| HRT channels | 1 | Minimal presence |
| CGCG-wide channels | 1 | Severely under-represented |
| Announcement channels | 0 | `#general` doing double duty |
| Escalation channels | 0 | Urgent issues have no path |
| Naming conventions | 4-5 different patterns | Inconsistent, hard to discover |

### Key Naming Conventions Found

- `fs-*` (16 channels) — FileScience, but mixing alerts, ops, team, and business channels
- `proj-*` (6 channels) — client projects, time-bound
- `team-*` (3 channels) — team channels, underused
- `topic-*` (4 channels) — discussion topics
- `alert-*` pattern — not yet adopted, but needed
- Unprefixed (16 channels) — general, random, social, misc

### Recommended Slack Architecture (Converging Model)

**Layer 1 — CGCG-Wide (everyone, implement immediately)**
- `#announce-company` — restricted posting, leadership announcements
- `#general` — becomes watercooler/discussion (no longer announcements)
- `#escalations` — cross-team urgent issues
- `#team-leadership` — already exists, clarify scope
- `#ops-billing` — unified (finance is already converged)
- `#ops-hiring` — unified (one employer)
- `#cgcg-soc2` — already exists

**Layer 2 — Entity Channels (give every entity a home)**
- `#fs-general`, `#fs-engineering`, `#fs-ops`, `#fs-sales` — already exist
- `#mcs-general`, `#mcs-ops` — CREATE IMMEDIATELY
- `#hrt-general`, `#hrt-ops` — create (expand from `#hrt-billing`)

**Layer 3 — Functional Channels (converge over time)**
- As service delivery unifies → `#mcs-ops` + `#hrt-ops` → `#team-ops`
- As client books merge → entity-specific client channels → `#client-*`
- As tooling unifies → entity alert channels → `#alert-*`

**Layer 4 — Automated/Alert Channels (standardize prefix now)**
- All new automated channels use `#alert-*` prefix
- Existing `fs-infra-failures`, `fs-billing-alerts` etc. → rename to `#alert-*` during Phase 2

### Migration Phases

**Phase 1 — Now (no disruption, immediate value):**
- Create `#announce-company` (restrict posting to leadership)
- Create `#mcs-general` and `#hrt-general`
- Create `#escalations`
- All new automated channels use `#alert-` prefix
- Document the channel taxonomy and share with team

**Phase 2 — At ~35-40 people:**
- Rename `fs-` alert channels to `#alert-*`
- Consolidate client channels into `#client-*` convention
- Archive dead `#proj-*` channels (quarterly lifecycle)
- Stand up `#ask-ops` / `#ask-engineering` for cross-team Q&A

**Phase 3 — At ~50 people:**
- Full `#team-*` structure for every real team
- Merge entity ops channels as service delivery converges
- Formal channel lifecycle policy
- Sidebar section guidance for all employees

---

## Governance Principles

### Threading Discipline
Top-level messages are topics. Responses go in threads. "Also reply to channel" for conclusions only. This is the #1 habit that determines whether Slack scales.

### Channel Lifecycle
Every `#proj-*` channel gets an archive date in the topic. Quarterly audit: archive dead channels.

### DM Policy
Work decisions that affect more than one person must happen in channels, not DMs. DMs are for personal/sensitive matters only. This is critical for acquired teams who may default to DMs because they don't have channels yet.

### Default Channels on Join
New hires auto-join: `#announce-company`, `#general`, their `#team-*` channel, `#escalations`. NOT every client or alert channel.

---

## Decision Points for Leadership

Before implementing the full plan, leadership needs to align on these questions:

1. **Entity model** — Federated, unified, or converging? (Recommendation: converging)
2. **Brand strategy** — Do MCS and HRT keep their client-facing brands? This determines whether client channels are entity-scoped or unified.
3. **Service delivery structure** — One ops team or entity-specific ops teams? This determines the convergence timeline.
4. **Who owns this** — Communication architecture needs an owner. At 50 people, this is typically an ops/people lead, not ad hoc.

---

## Recommended Next Steps

1. **Immediate (this week):** Create `#announce-company`, `#mcs-general`, `#hrt-general`, `#escalations`
2. **Short-term (30 days):** Leadership aligns on entity model (A/B/C)
3. **Medium-term (60-90 days):** Implement Phase 2 channel restructuring based on model decision
4. **Ongoing:** Quarterly channel audits, update taxonomy as entities converge
