# Idea Brief: Routing Reliability via UserPromptSubmit Hook

**Date:** 2026-03-04
**Status:** Shaped → Planning

## Problem
Claude Code's routing table in CLAUDE.md misses keyword triggers (~60-70% reliability) because instructions compete for attention in a 144-line document. When a user says "deep research", Claude may treat it as a generic descriptor rather than a skill trigger.

## Constraints
- UserPromptSubmit hook fires on every prompt (no matcher support) — must be fast (< 100ms)
- Hook can inject additionalContext (appears as system-reminder) but cannot modify the prompt
- Known issue #17550: JSON output may error on first message — plain stdout is safer fallback
- Existing hooks infrastructure in .claude/settings.json is well-structured
- CLAUDE.md routing table must remain as documentation + semantic fallback

## Options Considered

### UserPromptSubmit hook with keyword matching
Python script reads prompt from stdin JSON, matches against a config file of patterns → skills, emits additionalContext with routing directive. Config file allows adding routes without editing the script.
- Gains: ~84% activation (community data), high salience via system-reminder injection, idiomatic pattern
- Costs: Fires on every prompt (must be fast), another hook to maintain
- Complexity: Low

### Rules file extraction
Move routing triggers into a dedicated .claude/rules/ file for higher attention. No hook.
- Gains: Zero new code, leverages existing rules infrastructure
- Costs: Still relies on model attention (~60-70%), no deterministic matching
- Complexity: Very Low

### CLAUDE.md restructuring with XML tags
Wrap routing table in XML tags per Anthropic best practices. No hook.
- Gains: Improved salience within existing document
- Costs: Still probabilistic, marginal improvement
- Complexity: Very Low

## Chosen Approach
**UserPromptSubmit hook with keyword matching** — deterministic matching at the hook layer, with CLAUDE.md as semantic fallback. The layered approach (hook for exact triggers, CLAUDE.md for intent-based routing) gives the best of both worlds.

## Key Context Discovered During Shaping
- Deep research confirmed UserPromptSubmit is the idiomatic mechanism for this
- additionalContext appears as system-reminder block — highest salience injection point
- Community examples: ChrisWiles/claude-code-showcase, claudefa.st skill-activation-hook
- XML-tagged longer explanations outperform terse directives in additionalContext
- Research doc: memory-bank/thoughts/shared/research/2026-03-04-deep-routing-reliability-claude-code.md

## Next Step
- [Plan] → /create_plan memory-bank/thoughts/shared/briefs/2026-03-04-routing-reliability-hook.md
