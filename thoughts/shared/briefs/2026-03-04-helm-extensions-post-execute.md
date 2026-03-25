# Idea Brief: Helm Extensions — Post-Execute Pipeline

**Date:** 2026-03-04
**Status:** Shaped → Planning (Telemetry + Rescoping), Shaped → Parked (others)

## Problem
The execute pipeline (helm full pipeline plan, 2026-03-04) gives helm a "fire and forget" happy path. But software doesn't follow happy paths — orchestrations fail in ways that need intelligence, not just retries. Additionally, every orchestration generates valuable data (review results, merge timing, conflict patterns, retry history) that is currently discarded as ephemeral terminal output. Without persistent telemetry, helm can't learn from its own history, can't detect when rescoping is needed, and can't improve future decompositions.

## Constraints
- One-person team — extensions must amplify, not add overhead
- Execute pipeline must ship first — all extensions build on top of it
- Linear remains the control plane for all orchestration state
- Token cost matters — can't add unbounded agent invocations per orchestration
- `retry_counts` field (from execute plan) is the first persistent failure data — a seed for telemetry
- Existing shaped briefs cover nightly agents (ENG-2183) and harness self-improvement (ENG-2225) — don't duplicate, link

## Extension Dependency Graph

```
                    ┌─────────────────────┐
                    │  1. Orchestration   │  ← FOUNDATIONAL
                    │     Telemetry       │     (everything else needs this)
                    └────────┬────────────┘
                             │
              ┌──────────────┼──────────────┐
              │              │              │
    ┌─────────▼──────┐ ┌────▼─────┐ ┌──────▼──────┐
    │ 2. Rescoping / │ │ 3. Quality│ │ 4. Nightly  │
    │    Escalation  │ │  Feedback │ │  Execution  │
    │    System      │ │  Loop     │ │  (ENG-2183) │
    └─────────┬──────┘ └────┬─────┘ └──────┬──────┘
              │              │              │
              │         ┌────▼─────┐  ┌─────▼──────┐
              │         │ Smarter  │  │  Harness   │
              │         │ Decomp   │  │  Self-Impr │
              │         └──────────┘  │  (ENG-2225)│
              │                       └────────────┘
              │
    ┌─────────▼──────────┐
    │ 5. Plan-from-Issue │
    │    Generation      │
    └─────────┬──────────┘
              │
    ┌─────────▼──────────┐
    │  Tournament Design │  ← V2 capstone (existing vision)
    │  Mode              │
    └────────────────────┘
```

## Extensions

### 1. Orchestration Telemetry (foundational)

**Problem:** Nothing persists after an orchestration beyond the manifest JSON (final state only). Review results, merge timing, conflict history, retry counts, promote actions — all ephemeral typer.echo output. No audit trail. No learning corpus.

**What it captures:**
- Timestamped event log per orchestration: state transitions, review verdicts (with full ReviewResult), merge outcomes (with MergeResult), conflict detections, promote actions, failure + retry events, agent timeout escalations
- Per-sub-issue timeline: PENDING -> IN_PROGRESS -> IN_REVIEW -> DONE with wall-clock timestamps at each transition
- Aggregate metrics: total orchestration duration, time-per-phase, retry rate, review block rate, conflict rate

**Implementation approach:**
- Event log as a list of typed Pydantic events in the manifest (or sidecar `.events.json` file)
- Events append-only during execution, archived alongside the manifest on completion
- `helm analytics` command aggregates across archived orchestrations: success rate, avg duration, common failure modes, cost estimates
- Structured JSON — queryable by scripts, agents, and future extensions

**Complexity:** Low-Medium — mostly plumbing existing data to a persistent store instead of typer.echo

**Dependencies:** Execute pipeline (source of events)

**Enables:** Extensions 2, 3, 5 all need historical data to function

---

### 2. Rescoping / Escalation System

**Problem:** The execute pipeline handles known failure modes with retries (max 2 per type). But some failures indicate the plan itself is wrong — the approach doesn't work, the decomposition created unsolvable conflicts, or the agent discovered a constraint the plan didn't anticipate. Today, the loop just stops after max retries with a generic error. No intelligence about why it failed or what to do next.

**Detection signals (deterministic, from telemetry):**
- Same sub-issue exhausts retries on multiple failure types (not just one — indicates fundamental blocker)
- Multiple sub-issues failing concurrently (decomposition was wrong, not individual phase issues)
- Agent PR comments/descriptions contain escalation keywords ("can't", "impossible", "blocked by", "need clarification", "assumption was wrong")
- Plan verification criteria consistently failing despite fix attempts (the criteria themselves may be wrong)
- Circular conflict pattern: phases A and B keep conflicting after resolution (decomposition overlap was unresolvable)

**Response spectrum:**

| Signal | Severity | Response |
|--------|----------|----------|
| Single retry fixes it | Normal | Continue (existing behavior) |
| Max retries on one failure type, one phase | Low | Pause that phase, continue others, log warning |
| Max retries on multiple failure types, one phase | Medium | Pause phase, invoke Claude diagnosis agent |
| Multiple phases failing concurrently | High | Full stop, generate rescope brief |
| Agent explicitly escalates in comments | High | Immediate pause + diagnosis |
| Circular conflict between phases | Critical | Decomposition was wrong — need replan |

**Diagnosis agent:** When severity >= Medium, helm invokes a Claude subprocess with:
- The failing sub-issue's PR diff, review results, retry history (from telemetry)
- The plan phase description and verification criteria
- The plan's assumptions section
- Prompt: "Analyze why this phase is failing. Is the failure fixable with a different approach, or does the plan need rescoping? Output structured JSON: {verdict: 'fixable'|'rescope_phase'|'rescope_plan', reason: str, suggestion: str}"

**Rescope brief generation:** When diagnosis recommends rescoping, helm auto-generates a rescope brief:
```markdown
# Rescope Brief: [Parent Issue] — [Failed Phase(s)]
## What was attempted
[From plan + telemetry]
## What failed and why
[From diagnosis agent + review results + retry history]
## What the agent discovered
[Extracted from PR comments/descriptions]
## Assumptions that were wrong
[Cross-referenced against plan assumptions section]
## Recommended next step
[Diagnosis agent suggestion: reshape approach / rewrite phase / split differently]
```

This brief becomes input to a `/shape` session — the user picks up from where the orchestration left off, with full context of what was tried and what didn't work.

**Complexity:** Medium — detection is deterministic (pattern matching on telemetry), diagnosis uses existing Claude subprocess pattern, brief generation is templated

**Dependencies:** Extension 1 (telemetry for detection signals), execute pipeline (retry_counts, failure data)

---

### 3. Quality Feedback Loop

**Problem:** Each orchestration generates implicit quality signals — how many retries, which phases conflict, how often reviews block — but these signals vanish after the orchestration completes. There's no mechanism to learn from past orchestrations to improve future ones.

**What it provides:**
- **Decomposition quality score**: % of phases that completed without retry / conflict / review block. Tracked per orchestration, trended across orchestrations.
- **Plan pattern analysis**: Correlate plan structure (number of phases, file overlap density, dependency depth) with orchestration success rate. Surface patterns like "plans with >5 phases have 2x the conflict rate" or "phases touching shared config files always need retry."
- **Agent effectiveness by task type**: Which kinds of phases (new file creation, refactoring existing, test writing) succeed most/least often. Feed back into plan authoring guidance.
- **Decomposition recommendations**: When `helm decompose` runs, compare the proposed decomposition against historical patterns. Warn: "Phase 3 and 4 share 4 files — historically this overlap pattern leads to conflicts in 70% of orchestrations. Consider merging these phases."

**Implementation approach:**
- `helm analytics` reads archived orchestrations + event logs
- Produces a summary report: success rates, common failure modes, plan structure correlations
- Optional: `helm decompose --check-history` compares proposed decomposition against historical patterns before creating sub-issues

**Complexity:** Medium — analytics is aggregation over structured JSON; decomposition recommendations need pattern matching

**Dependencies:** Extension 1 (telemetry data to aggregate)

**Enables:** Smarter decomposition (extension 3 provides the data, smarter decomposition acts on it)

---

### 4. Nightly Execution (ENG-2183 activation)

**Problem:** Nightly scheduled agents (shaped, parked as ENG-2183) have no execution infrastructure. Once `helm execute` exists, nightly agents become nearly trivial.

**What it provides:**
- GH Actions cron creates a Linear issue from a template (e.g., "Nightly: stale TODO scanner", "Nightly: CVE audit")
- Template includes a pre-written plan file path
- Cron invokes `helm execute <plan-file> --issue <new-issue>`
- Results land in Linear comments + repo (memory-bank/thoughts/)

**Implementation approach:**
- Nightly issue templates: YAML files in `.claude/nightly/` defining title, description, plan path, labels
- GH Action: `schedule: cron: '0 6 * * *'` → reads templates → creates issues → invokes helm execute
- Cost capping: `--max-budget` flag on execute (pass through to Claude subprocess `max_budget_usd`)

**Complexity:** Low — the execute pipeline does the hard work; this is just cron + templating

**Dependencies:** Execute pipeline (must be stable and battle-tested first)

**Enables:** Harness Self-Improvement (ENG-2225) — becomes a nightly task that uses this infrastructure

**Existing brief:** `memory-bank/thoughts/shared/briefs/2026-02-26-nightly-scheduled-agents.md`

---

### 5. Plan-from-Issue Generation

**Problem:** The current workflow has a gap between "I have a Linear issue" and "I have a decomposable plan." Today a human (or a `/create_plan` session) must write the plan. For well-scoped issues with clear acceptance criteria, this could be automated — closing the loop from issue creation to execution.

**What it provides:**
- `helm plan ENG-XXXX` reads the issue description and acceptance criteria from Linear
- Invokes Claude to generate a phased plan with file scopes (using codebase context)
- Human reviews/edits the plan (this is the gate — no fully autonomous plan generation)
- `helm execute` takes over from the approved plan

**Implementation approach:**
- Claude subprocess reads the issue + codebase structure + historical plan patterns (from telemetry/quality feedback)
- Outputs a plan file in the standard template format with `## Phase N`, `### Files` sections
- Plan is written to `memory-bank/thoughts/shared/plans/` for human review
- Only proceeds to execute after explicit human approval

**Complexity:** Medium-High — plan generation quality depends heavily on prompt engineering and codebase understanding

**Dependencies:** Execute pipeline (consumer), Extension 3 (historical patterns improve plan quality)

**Enables:** Tournament Design Mode (V2) — tournament generates N plans, evaluates them, executes the winner. Plan generation is the prerequisite.

---

## Relationship to Existing Work

| Extension | Existing Brief/Issue | Relationship |
|-----------|---------------------|--------------|
| 1. Telemetry | Agent Observability research (2026-03-03) | Implements the "structured event logging" recommendation from that research |
| 2. Rescoping | (new) | Builds on execute pipeline failure handlers |
| 3. Quality Feedback | Harness Self-Improvement (ENG-2225) | Quality feedback is the data layer; self-improvement is a consumer |
| 4. Nightly Execution | Nightly Agents (ENG-2183) | Execute pipeline makes the nightly brief's chosen approach (Option A) trivial |
| 5. Plan Generation | Experiment-Augmented Shaping (2026-03-03) | Plan generation could use experiments to validate approach before committing |
| V2 Tournament | Macro Agent Strategy (2026-02-28) | Plan generation + execute = tournament infrastructure |

## Priority Recommendation

1. **Telemetry** — build alongside or immediately after execute pipeline. Low cost, enables everything. Without it, extensions 2-5 are flying blind.
2. **Rescoping** — build after telemetry. Highest immediate value for execution reliability. Makes "walk away" actually safe.
3. **Nightly Execution** — build after execute is battle-tested. Nearly free, unlocks ENG-2183 and ENG-2225.
4. **Quality Feedback** — build after enough orchestrations have telemetry data to analyze (needs N>5 completed orchestrations).
5. **Plan Generation** — build last. Highest complexity, most dependent on quality signals from earlier extensions.

## Next Steps
- [Plan] Telemetry → create Linear issue, plan after execute pipeline ships
- [Plan] Rescoping → create Linear issue, plan after telemetry ships
- [Park] Nightly Execution → update ENG-2183 with execute pipeline dependency
- [Park] Quality Feedback → create Linear issue, park until telemetry data accumulates
- [Park] Plan Generation → create Linear issue, park until quality feedback exists
