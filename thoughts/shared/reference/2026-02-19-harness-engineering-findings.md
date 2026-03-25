# Harness Engineering: Findings & FileScience Flow Improvements

**Source:** [OpenAI — Harness engineering: leveraging Codex in an agent-first world](https://openai.com/index/harness-engineering/) (Feb 11, 2026)
**Author:** Ryan Lopopolo, Member of the Technical Staff
**Date extracted:** 2026-02-19

---

## Key Findings from the Article

### Scale & Results
- **0 lines of manually-written code** — entire internal product built by Codex agents in 5 months
- ~**1 million lines** of code (application logic, infra, tooling, docs, internal dev utilities)
- ~**1,500 PRs** merged by a team that grew from 3 → 7 engineers
- **3.5 PRs per engineer per day** average throughput (and throughput _increased_ as team grew)
- Estimated **10x speed** vs. manual coding
- Single Codex runs regularly last **6+ hours** (often overnight)

### Philosophy: Humans Steer, Agents Execute
- Humans prioritize work, translate user feedback into acceptance criteria, validate outcomes
- When the agent struggles → "what capability is missing, and how do we make it legible and enforceable?"
- The fix is almost never "try harder" — it's "what tool/guardrail/doc is missing?"

### AGENTS.md: Table of Contents, Not Encyclopedia
- Tried one big AGENTS.md — **failed** because:
  1. Context is scarce; bloated docs crowd out actual task context
  2. When everything is "important," nothing is — agents pattern-match locally
  3. Monolithic docs rot instantly; agents can't tell what's still true
  4. Single blobs resist mechanical verification (freshness, coverage, ownership)
- Solution: **~100-line AGENTS.md as a map** with pointers to deeper `docs/` sources of truth

### Repository Knowledge as System of Record
- Structured `docs/` directory layout:
  - `design-docs/` — catalogued, indexed, with verification status + "core beliefs"
  - `exec-plans/` — `active/`, `completed/`, `tech-debt-tracker.md`
  - `product-specs/` — indexed product specifications
  - `references/` — external `*-llms.txt` files for libraries/tools
  - `generated/` — auto-generated docs (e.g., `db-schema.md`)
  - Top-level: `DESIGN.md`, `FRONTEND.md`, `PLANS.md`, `PRODUCT_SENSE.md`, `QUALITY_SCORE.md`, `RELIABILITY.md`, `SECURITY.md`
- **Progressive disclosure**: agents start with small stable entry point, taught where to look next
- "What Codex can't see doesn't exist" — Slack threads, Google Docs, tacit knowledge must be encoded into the repo

### Mechanical Enforcement
- **Dedicated linters + CI** validate knowledge base is up-to-date, cross-linked, structured
- Custom linters with **remediation instructions baked into error messages** — inject fix guidance into agent context
- Structural tests enforce dependency direction (Types → Config → Repo → Service → Runtime → UI)
- Enforce **invariants, not implementations** — e.g., "parse data at the boundary" but not prescriptive on which library

### Doc-Gardening Agent
- **Recurring agent** scans for stale/obsolete documentation that doesn't reflect real code behavior
- Opens fix-up PRs automatically
- Most reviewable in under a minute, automerged

### Quality Score per Domain
- `QUALITY_SCORE.md` grades each product domain and architectural layer
- Tracks gaps over time

### Agent Legibility First
- Code optimized for **agent's legibility**, not human stylistic preferences
- "Boring" tech preferred — composable, stable APIs, well-represented in training data
- Sometimes cheaper to reimplement a subset than fight opaque upstream behavior (example: custom concurrency helper > `p-limit` because it's tighter with OTel, 100% tested, exact runtime semantics)

### Application Legibility (Chrome DevTools + Observability)
- App bootable per git worktree → Codex launches one instance per change
- **Chrome DevTools Protocol** wired into agent runtime — DOM snapshots, screenshots, navigation
- **Local observability stack per worktree** — ephemeral logs/metrics/traces, agents query LogQL/PromQL
- Prompts like "ensure startup < 800ms" or "no span exceeds 2s in these 4 journeys" become tractable

### Execution Plans as First-Class Artifacts
- Lightweight plans for small changes, full **execution plans** for complex work
- Progress + decision logs checked into the repo
- Active/completed/tech-debt all versioned and co-located

### Merge Philosophy
- **Minimal blocking merge gates** — corrections are cheap, waiting is expensive
- PRs are short-lived
- Test flakes addressed with follow-up runs rather than blocking indefinitely
- Agent-to-agent review — humans may review but aren't required to

### Entropy & Garbage Collection
- Initially spent **20% of the week** (every Friday) manually cleaning "AI slop" — didn't scale
- Replaced with **"golden principles"** encoded in repo + recurring automated cleanup:
  - Background Codex tasks scan for deviations on a regular cadence
  - Update quality grades
  - Open targeted refactoring PRs
- Technical debt treated like a high-interest loan — pay continuously in small increments

---

## What We Already Do Well (Validated)

| Their Practice | Our Equivalent | Status |
|---|---|---|
| AGENTS.md as TOC/router | CLAUDE.md route system (A-F) | Good — already router-style |
| Structured docs directory | `memory-bank/` (durable + archival split) | Good — similar progressive disclosure |
| Execution plans as artifacts | `memory-bank/thoughts/shared/plans/` | Good — dated, versioned |
| Mechanical enforcement | ruff + ty + import-linter PostToolUse hooks | Good — 7 contracts, auto on every edit |
| Enforce invariants not implementations | import-linter contracts for Polylith boundaries | Good — architectural fitness |
| Push context into repo | decisions/, reference/, research/ docs | Good — codified knowledge |
| Chrome DevTools available | MCP integration configured | Available but underutilized |
| Local observability | OTel pilot with Jaeger/Prometheus | Good — already invested |

## Improvement Opportunities (Gaps)

### 1. Doc-Gardening Automation (HIGH)
**Gap:** No automated freshness checking for `memory-bank/` artifacts.
**Their approach:** Recurring agent scans for stale docs, opens fix-up PRs.
**Our action:** Create a `/garden` skill or periodic task that:
- Scans `current_work.md` and `next_up.md` for entries older than 2 weeks
- Checks if linked plans/research artifacts still reflect code reality
- Flags stale entries for cleanup or archival

### 2. Quality Score Tracking (HIGH)
**Gap:** No per-domain quality grading.
**Their approach:** `QUALITY_SCORE.md` grades each domain/layer, tracks gaps over time.
**Our action:** Create `memory-bank/durable/01-active/quality_score.md` with grades for:
- Discover (orchestration, cloud services, throttling, queue)
- Entity Discovery Trigger
- Cloud API component
- Infrastructure (Terraform/Terragrunt)
- Observability coverage
- Test coverage (mutation score per area)

### 3. Linter Remediation Messages (MEDIUM)
**Gap:** Our ruff/ty/import-linter errors use default messages, not agent-optimized guidance.
**Their approach:** Custom lint error messages inject remediation instructions into agent context.
**Our action:** Add custom ruff rule messages in `pyproject.toml` where possible; for import-linter violations, add `description` fields with fix guidance. Consider custom linters for domain-specific rules (e.g., "all cloud service handlers must implement delta support").

### 4. Chrome DevTools for QA Loops (MEDIUM)
**Gap:** We have Chrome MCP but don't use it for systematic agent-driven validation.
**Their approach:** Agent boots app, drives UI via CDP, takes before/after screenshots, validates fixes.
**Our action:** Not directly applicable yet (backend-focused), but when Dashboard/Stargate work begins, wire CDP into the validation loop from day one.

### 5. Observability-Driven Agent Tasks (MEDIUM)
**Gap:** Our OTel pilot exists but agents can't yet query it for task validation.
**Their approach:** Agents query LogQL/PromQL to validate perf constraints.
**Our action:** When OTel is promoted beyond pilot:
- Make local observability queryable from skills (PromQL queries in validation steps)
- Add plan verification criteria like "no span > 2s" that agents can check

### 6. Golden Principles + Automated Cleanup (MEDIUM)
**Gap:** No recurring automated code-quality sweep.
**Their approach:** Background tasks scan for deviations, update quality grades, open refactoring PRs.
**Our action:** Define "golden principles" for our codebase (e.g., typed DTOs at boundaries, structured logging, no raw dict access for API responses) and create a periodic `/sweep` skill.

### 7. Agent-to-Agent Review (LOW — future)
**Gap:** No self-review or multi-agent review loops.
**Their approach:** Codex reviews its own changes, requests additional agent reviews, iterates until satisfied.
**Our action:** Not urgent for our team size, but worth exploring as a plan validation step — have the agent critique its own implementation before marking a plan phase complete.

### 8. Tech Debt Tracker (LOW)
**Gap:** No dedicated tech debt tracking file.
**Their approach:** `tech-debt-tracker.md` co-located with plans.
**Our action:** Consider adding `memory-bank/durable/01-active/tech_debt.md` to track known debt items alongside blockers.
