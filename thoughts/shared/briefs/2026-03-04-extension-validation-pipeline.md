# Idea Brief: Layered Validation Pipeline for Claude Code Extensions

**Date:** 2026-03-04
**Status:** Shaped → Planning

## Problem
After meta work (memory v2 migration, hook rewrites, skill updates), there is no way to validate that the Claude Code extension graph still works without manually starting a fresh session. Broken cross-references, stale context injection, and semantic drift in prompt templates go undetected until a human eyeballs a real session.

## Constraints
- `CLAUDECODE=1` blocks nested `claude -p` from within Claude Code — CI is the only viable e2e test environment
- No `agent_id` in hook events (upstream #14859) — subagent attribution not possible in real time
- Skills/rules/agents are markdown prompts — no programmatic behavioral test without an LLM
- 7 of 9 hooks have zero test coverage; only scope guards are tested
- Proven test pattern exists (scope_guard tests via importlib + stdin mock) — replicable to other hooks

## Options Considered

### Layered Validation Pipeline (Full Stack)
5 layers: cross-ref linter → hook unit tests → transcript validation → CI smoke tests → LLM-as-judge. Each catches a distinct failure class. Layers compound — a bug must evade all 5 to reach a user.
- Gains: Full testability spectrum coverage, each layer independently useful
- Costs: Non-trivial total build effort, ongoing maintenance
- Complexity: Medium-High (each layer is Low individually)

### Observability-First
Instrument runtime with hook-based telemetry, validate via queries against real session data. No synthetic test inputs needed.
- Gains: Tests real behavior, doubles as debugging tool
- Costs: No pre-merge gate for structural breaks
- Complexity: Medium

### Hybrid (Static Gate + Runtime Observability)
Layers 1-2 as CI gates, Layer 3 as always-on Stop hook. Skip CI smoke tests and promptfoo.
- Gains: Fast CI gate + runtime observability, less complexity
- Costs: No pre-merge behavioral validation
- Complexity: Medium

## Chosen Approach
**Layered Validation Pipeline (Full Stack)** — optimal coverage. Each layer catches failures the others miss. Cross-ref linter catches broken paths pre-merge. Hook unit tests catch logic regressions. Transcript validation catches runtime drift. CI smoke tests validate e2e composition. Promptfoo catches semantic drift in prompts.

## Key Context Discovered During Shaping
- CLAUDE.md routes `/debug` but only `investigate.md` exists in commands/ — cross-ref linter would catch this
- SessionStart hook builds `additionalContext` string that's snapshot-testable with mocked git + filesystem
- JSONL transcripts at `~/.claude/projects/<path>/<session-id>.jsonl` contain full tool call history parseable for assertions
- promptfoo supports `llm-rubric` assertions and has a dedicated coding agent evaluation guide
- Research doc: `memory-bank/thoughts/shared/research/2026-03-04-testing-claude-code-extensions.md`

## Next Step
- [Plan] → `/create_plan memory-bank/thoughts/shared/briefs/2026-03-04-extension-validation-pipeline.md`
