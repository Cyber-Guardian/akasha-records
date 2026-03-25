---
type: event
created: 2026-03-23
status: active
---
# Idea Brief: Assembly-Line V2 — Deferred Advanced Features

**Date:** 2026-03-23
**Status:** Shaped → Parked (backlog items for after v1 ships)

## Problem

The v1 assembly-line plan (see [[2026-03-23-assembly-line-work-ontology|v1 plan]]) deliberately defers several features that were identified during shaping and adversarial review. These are real gaps, not nice-to-haves — but they require either more infrastructure or more experience with v1 before implementing correctly.

## Deferred Features

### 1. Automated Contract Runner (TaskCompleted Integration)
**Why deferred:** The mechanism to identify which plan's contracts to run at TaskCompleted time is undefined. `PLAN_PATH` env var doesn't exist in Claude Code's hook protocol. Fallback (most recently modified active plan) is non-deterministic when multiple plans exist.
**What's needed:** Either (a) `implement_plan` injects `PLAN_PATH` into session env (requires understanding Claude Code's env propagation), (b) a `.claude/sessions/{id}/active_plan.txt` breadcrumb file written by `implement_plan` and read by the hook, or (c) Claude Code exposes plan context natively (upstream feature request).
**Value:** Closes the loop — contracts are checked automatically instead of relying on the agent to run them. Prevents the fix-chain pattern mechanically.
**Risk if skipped:** Contracts exist but enforcement is manual (agent runs them per `implement_plan` instructions). This is ~80% as effective as automated enforcement but still a massive improvement over no contracts at all.

### 2. Independent-Agent Reflection (True Adversarial Review)
**Why deferred:** v1's reflection runs in the same session context as the implementer. The adversarial-code-reviewer agent exists as an independent agent, but dispatching it mid-implementation requires: (a) constructing a synthetic diff from the current phase's changes, (b) passing the plan path + phase number, (c) marshaling findings back into the reflection loop. The agent's input format (PR diff + plan path) doesn't match what `/simplify` produces (changed files from git).
**What's needed:** Adapter between `/simplify`'s output format and `adversarial-code-reviewer`'s input format. Possibly a new agent definition purpose-built for the reflection station that takes changed files + plan context directly.
**Value:** True adversarial review — a fresh context reviews the work without anchoring to the implementer's decisions. The data shows agents pass their own reviews almost always; independent review catches what self-review misses.
**GSD-2 precedent:** GSD-2 dispatches each task in a fresh context window. The reflection pass could similarly get a fresh context with only the changed files + contracts, not the full implementation history.

### 3. Performance Lens
**Why deferred:** No profiling methodology defined. "Within timeout budgets" is vague. For a Python codebase, profiling hot paths requires either (a) `cProfile` integration, (b) benchmark test infrastructure, or (c) wall-clock timing of specific operations. None of this exists.
**What's needed:** Define when the performance lens applies (plan specifies it explicitly), what it measures (specific operations with timing thresholds), and how it measures (test fixtures with timing assertions, `pytest-benchmark`, or inline profiling).
**Value:** Prevents the autoresearch problem — code that works but is too slow for the eval timeout. Performance issues are currently caught by the evaluator, not by the development process.

### 4. Alignment Lens (Full Graph-Aware)
**Why deferred:** The alignment lens checks whether implementation serves the plan's goals and is consistent with dependent/downstream plans. This requires the plan graph (Phase 2) to exist and have edges populated. In v1, the alignment lens can check against the current plan's stated goals but not against neighbor plans.
**What's needed:** Graph traversal: given the current plan, load its `depends_on` and `enables` neighbors, read their interface expectations, and verify the current implementation is compatible.
**Value:** Macro coherence — the assembly line doesn't just produce correct individual phases, it produces phases that fit together across plans. This is the "factory" in "software factory."

### 5. Plan Graph Robustness
**Why deferred:** Several graph edge cases identified during review:
- **Bidirectionality:** If Plan A has `enables: [plan-b]` and Plan B has `depends_on: [plan-a]`, these should be the same edge. But if only one side is set, the graph has phantom edges. Need a union policy.
- **Archived dependencies:** A dependency on an archived plan is satisfied (work is done). A dependency on a non-existent plan stem is a broken reference. The graph script should distinguish these.
- **Stem collisions:** If two plans cover the same topic on different dates (superseded + new), the `depends_on` field needs to reference the correct stem. Need a convention: use full stem with date, or use a stable topic-slug alias.
**What's needed:** Graph validation logic, bidirectionality normalization, and a convention doc for plan stems.

### 6. Plan Self-Freeze Protection
**Why deferred:** If the freeze hook ships and someone freezes a plan mid-implementation, the hook blocks further edits to the plan itself (checking off phases, updating status). The executing plan needs an exemption.
**What's needed:** Either (a) the freeze hook exempts the currently-executing plan (requires `PLAN_PATH` mechanism from item 1), (b) the convention is "don't freeze until all phases complete" (enforced by docs, not hooks), or (c) the freeze hook only blocks content edits, not checkbox toggles (hard to distinguish mechanically).
**v1 approach:** Convention — plans are frozen after approval but before the implementing agent starts. The human freezes manually. If a plan needs unfreezing, the human sets `status: active`.

### 7. CI Contract Validation
**Why deferred:** Contract commands in active plans could become stale (paths change, make targets renamed). A CI job that periodically runs all contract commands in all active/frozen plans would catch this.
**What's needed:** A script that globs plans, extracts `### Contracts` commands, runs them, reports failures. Could run nightly or on PR.
**Value:** Prevents the "contracts are correct when written but rot over time" failure mode.

### 8. File Size Threshold Tuning
**Why deferred:** The 500/800 line thresholds are arbitrary. Current codebase has files well over 500 lines (coordinator.py: 2143). The hook would fire warnings on every edit to these files for unrelated changes, creating noise.
**What's needed:** Audit current file sizes, set thresholds based on the 90th percentile of current non-test Python files, create an initial exemptions list for known large files that are being actively decomposed.
**Prerequisite:** The git meta-analysis data from this session. Run `wc -l` on all `.py` files, set thresholds above the current state, tighten over time.

## Chosen Approach
**Park all 8 items as Linear backlog issues.** Revisit after v1 ships and we have experience with the contract format, plan freeze, and reflection station in practice. Items 1 and 2 are highest priority for v2.

## Next Step
- [Parked] → Linear backlog issues for each item
