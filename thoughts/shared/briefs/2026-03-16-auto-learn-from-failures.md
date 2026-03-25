---
type: event
created: 2026-03-16
status: active
---
# Idea Brief: Auto-Learn from Failures

**Date:** 2026-03-16
**Status:** Shaped → Planning

## Problem
When Claude encounters a failure (tool error, CLI crash, agent timeout, wrong approach) and then figures out how to fix it, that recovery knowledge evaporates. The next session hits the same wall and burns the same tokens rediscovering the fix. Today, the only paths to persist lessons are manual — /learn (user invokes about a specific thing) or /retrospective (planned but not built, session-end sweep). There is no automatic, failure-triggered extraction loop.

## Constraints
- PostToolUse hook chain already uses ~4-5s of ~25s budget
- PostToolUseFailure is a separate event (not competing with existing validator chain)
- Agent-type hooks exist and are proven (Stop hook closeout verifier)
- additionalContext injection available on all hook events
- Must filter noise from trivial failures (ruff/ty lint fixes)
- Existing JSONL logging pattern (routing_decisions.jsonl) as precedent

## Options Considered

### Failure Journal (structured event log)
PostToolUseFailure hook logs to JSONL. Stop hook scans journal + transcript for recovery arcs.
- Gains: Cheap, non-blocking (<50ms). Builds dataset. Decouples capture from analysis.
- Costs: No real-time benefit. Requires retrospective scanner.
- Complexity: Low (capture) + Medium (scanner)

### Live Failure Advisor (real-time context injection)
PostToolUseFailure hook logs + searches existing patterns. Injects additionalContext if match found.
- Gains: Real-time value — prevents re-discovery within same session. Two feedback loops.
- Costs: Search adds latency (~100-500ms). False matches could inject bad advice.
- Complexity: Medium

### Recovery Arc Extractor (transcript-aware agent)
Agent-type Stop hook reads transcript, identifies failure→recovery arcs, proposes learnings.
- Gains: Highest-quality extraction. Agent reasons about why recovery worked.
- Costs: 10-30s at session end. May propose low-value learnings from trivial failures.
- Complexity: Medium-High

## Chosen Approach
**Layered System** — all three as independent, composable layers:
- Layer 1 — Failure Journal (PostToolUseFailure, <50ms): Always-on JSONL capture
- Layer 2 — Live Advisor (PostToolUseFailure, ~200ms): Search known patterns, inject context if matched
- Layer 3 — Recovery Extractor (Stop, ~20s): Scan transcript for arcs, propose/write learnings

Each layer is independently useful and deployable. Progressive rollout: Layer 1 immediately, Layer 2 next, Layer 3 when /retrospective matures.

## Key Context Discovered During Shaping
- Two existing feedback_*.md files (AWS access, worktree isolation) prove the pattern works — both came from repeated human corrections
- "The skills that get referenced most aren't clean successes — they're the failures" (Claudeception creator)
- PostToolUseFailure is a distinct event from PostToolUse — fires on failure with error field
- Routing decisions already logged to .claude/hooks/routing_decisions.jsonl — precedent for structured event logging
- [[2026-03-13-retrospective-skill|Retrospective Skill]] is the manual sibling (shaped → planning)
- [[2026-03-16-custom-deterministic-agentic-backpressure|Backpressure Validators]] is the prevention layer — this is the learning layer

## Next Step
- [Plan] → `/create_plan memory-bank/thoughts/shared/briefs/2026-03-16-auto-learn-from-failures.md`
