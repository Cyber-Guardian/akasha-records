---
date: 2026-03-05T12:00:00-05:00
researcher: Claude
git_commit: e58030f9f755d5153428ce061b647a2f4117a91b
branch: main
repository: filescience
topic: "Top 5 next priorities across product and agentic engineering"
tags: [deep-research, priorities, product, agentic-engineering, strategy]
status: complete
research_depth: standard
iterations_completed: 3
last_updated: 2026-03-05
last_updated_by: Claude
---

# Deep Research: Top 5 Next Priorities

## Research Question
Surface the top 5 next priorities covering both product work (dormant ~2 weeks) and meta/agentic engineering. Cross-reference git log, Linear issues, and memory-bank to find what's stalled, ready, and highest-leverage.

## Summary
Product work has been completely dormant for 2 weeks — every commit since Feb 20 is meta/agentic (triage agent, CI, extensions, research docs). The highest-leverage product item is clearing 36 poison DynamoDB stream records that block both the discover pipeline and VQP observability. The VQP Terraform stream trigger is now unblocked (ENG-2133 shipped) and can be wired. On the agentic side, Helm Phase 1 (`promote.py`) is tightly scoped and gates the entire automation pipeline. The DDD test investment plan is dual-purpose — it addresses the "5 components with zero tests" debt while being a hard prerequisite for the DDD restructuring. A quick Linear housekeeping pass would close 4 resolved-but-never-closed CI issues and improve backlog signal.

## Perspectives Explored
1. **Product backlog & stalled work** — Identified poison records as top blocker, VQP stream trigger as newly unblocked, DDD plans as ready-to-execute
2. **Agentic engineering momentum** — Mapped shipped (extension validation, triage agent) vs. ready (helm Phase 1, agent testing pyramid) vs. planned (memory v2, plan trees)
3. **Linear issue landscape** — Found 4 stale-but-resolved CI issues, Discover V2 cluster parked since Feb 27, Helm items all in Triage
4. **Technical debt & infrastructure** — 5 untested components, lint.yml duplicate job, VQP hardcoded values, stream trigger gap
5. **Strategic leverage** — Dependency chain analysis showing what unblocks what

## Top 5 Priorities

### 1. Fix 36 Poison DynamoDB Stream Records (Product — Unblocks Everything)

**Why #1:** This is the single highest-leverage item across both tracks. It depends on nothing and unblocks two services simultaneously:
- Discover pipeline Phase 3 debug test runs are deferred until these are fixed
- VQP batch processor silently fails with `KeyError: 'data'` on the same records

**What:** 36 records in the DynamoDB stream cause silent `batchItemFailures`. They need to be identified, understood (malformed? schema drift?), and either fixed or purged.

**Status:** No Linear issue exists yet. Known blocker documented in topic note.

**Effort:** Small-medium — investigation + cleanup. Could be a single session.

**References:**
- [[discover-service|Discover Service]] topic note (line 26-27)
- [[valkey-queue-processor|VQP]] topic note (line 25)

---

### 2. Wire VQP DynamoDB Stream Trigger in Terraform (Product — Now Unblocked)

**Why #2:** VQP is code-complete but has never processed a live event because the Terraform `stream_enabled` + event source mapping was never applied. The blocker (ENG-2133 environment strategy) shipped. This is the critical path to VQP being a real, running service.

**What:** Apply `stream_enabled = true` on the DynamoDB table + create Lambda event source mapping in Terraform/Terragrunt.

**Status:** Unblocked as of ENG-2133 completion. No dedicated Linear issue for the wiring itself.

**Effort:** Small — Terraform module update + `terragrunt apply`.

**References:**
- [[valkey-queue-processor|VQP]] topic note (line 16-17, 27)
- `infrastructure/modules/` and `infrastructure/live/`

---

### 3. DDD Test Investment Plan (Product + Tech Debt — Dual Purpose)

**Why #3:** This is the "two birds, one stone" item. It:
- Addresses the tech debt of 5 components with zero tests (cloud_api, models, config, domain_backup, entity_discovery_trigger)
- Is a hard prerequisite for the DDD micro-architecture restructuring (the biggest planned product refactor)
- Guards against regressions in the WorkQueue (710 LOC), the most complex module

**What:** 3-phase plan: WorkQueue unit tests, ephemeral Docker integration fixtures, integration smoke tests. Fully planned, all decisions resolved, ready to `/implement_plan`.

**Status:** Parked since Feb 27. Linear issues ENG-2188, ENG-2213-2215 in Investigation.

**Effort:** Medium — estimated 3 phases, each a session.

**References:**
- [[2026-02-27-ddd-refactor-test-investment|DDD Test Investment Plan]]
- [[2026-02-27-ddd-micro-architecture|DDD Micro-Architecture Plan]] (depends on this)

---

### 4. Helm Phase 1: `promote.py` (Agentic — Gates Automation Pipeline)

**Why #4:** Helm V1 is complete (9 commands, 17/19 MVP weaknesses resolved), but the "fire-and-forget" automation loop requires `helm promote` + `helm execute`. Phase 1 (`promote.py`) is the gate — every subsequent phase depends on it. It's tightly scoped and independently useful even without Phases 2-4.

**What:** One new module (`promote.py`), Linear client extension for combined state+assignee mutation, CLI registration, 5 unit tests. Detects phases with all `blocked_by` dependencies DONE and auto-assigns them.

**Status:** Plan complete, all decisions resolved, ready to `/implement_plan`. Linear items ENG-2259-2263 in Triage.

**Effort:** Small-medium — single session.

**References:**
- [[2026-03-04-helm-full-pipeline|Helm Full Pipeline Plan]] (lines 115-184 for Phase 1)
- [[helm-orchestration|Helm Orchestration]] topic note

---

### 5. Linear Housekeeping + Stale Issue Cleanup (Meta — Signal Quality)

**Why #5:** The Linear backlog has accumulated noise that degrades prioritization:
- 4 CI issues (ENG-2179, 2170, 2169, 2159) are resolved and archived but stuck in "In Progress" — never moved to Done
- ENG-2128 (engineering execution discipline) has unchecked sub-issues that may be done
- ENG-2097 (display ads) stale 14 days
- 2 Urgent Todos from Aug 2025 (ENG-1168, ENG-897) that should be triaged or killed
- No Linear issues exist for the poison records or stream trigger wiring — they should be created

**What:** Close resolved issues, triage ancient items, create missing issues for known blockers.

**Effort:** Small — 30 minutes of issue management.

---

## Honorable Mentions (Next Wave)

| Item | Track | Status | Notes |
|------|-------|--------|-------|
| Agent testing pyramid | Agentic | Shaped, needs planning | Motivated by triage agent's 4 prod failures |
| DDD micro-architecture (4 phases) | Product | Planned, blocked on #3 | Biggest refactor — splits WorkQueue, adds domain layers |
| CI local/CI parity fixes | Agentic | Gap analysis done | New `make lint`/`make check` targets recommended |
| lint.yml duplicate extensions job | Tech debt | Ready to fix | Lines 81/108 — one shadows the other |
| Memory system v2 | Agentic | Planned | 4-tier hot/warm/cold/procedural |

## Context: What Shipped Recently (Last 2 Weeks)

All recent work has been agentic/meta:
- Triage agent rewrite + deploy with test gating (PRs #69, #71)
- Extension validation pipeline — 5-layer (PR #70)
- Blacksmith runner migration (PR #68)
- Deep research docs: CI fix loops, context window optimization, routing reliability, agent architectural patterns
- CI complexity reports (auto-generated)

Zero product commits in this period.

## Key Sources
**Codebase:**
- `.github/workflows/lint.yml:81,108` (duplicate extensions job)
- `bases/filescience/valkey_queue_processor/handler.py:107` (cluster mode hardcoded)
- `bases/filescience/valkey_queue_processor/policies/registry.py:38` (license_count hardcoded)
- `components/filescience/models/core.py:63` (CloudService TODO)

**Memory-bank:**
- [[discover-service|Discover Service]] — poison records blocker
- [[valkey-queue-processor|VQP]] — stream trigger gap
- [[helm-orchestration|Helm Orchestration]] — V1 complete, pipeline plan ready
- [[agent-discipline|Agent Discipline]] — triage failures, testing pyramid
- [[ci-pipeline|CI Pipeline]] — local/CI parity gap
- [[2026-02-27-ddd-micro-architecture|DDD Micro-Architecture Plan]]
- [[2026-02-27-ddd-refactor-test-investment|DDD Test Investment Plan]]
- [[2026-03-04-helm-full-pipeline|Helm Full Pipeline Plan]]
- [[2026-03-05-legacy-discover-feature-gap-analysis|Discover Feature Gap Analysis]]

**Linear:**
- ENG-2190 (DDD restructuring parent), ENG-2188/2213-2215 (test investment)
- ENG-2259-2263 (Helm next items)
- ENG-2179/2170/2169/2159 (stale CI — resolved, need closing)
- ENG-2128 (engineering discipline — needs sub-issue audit)
- ENG-1168/ENG-897 (ancient Urgent Todos)

## Open Questions
- Are the 36 poison records a schema issue, a one-time data corruption, or a systemic bug? Investigation needed before fix approach is clear.
- Should Dashboard Facelift and Clio Marketing get shaped/planned, or are they blocked on product decisions outside engineering?
- Is ENG-1827 (async DynamoDB writer) worth picking up now, or is it premature optimization before VQP is live?
- Should the agent testing pyramid be prioritized before more helm development, given the triage agent's production failures?

## Research Metadata
- Depth mode: standard
- Iterations completed: 3 / 5
- Termination reason: gaps exhausted
- Manifest: `.claude/deep-research/2026-03-05-top-5-next-priorities.md`
