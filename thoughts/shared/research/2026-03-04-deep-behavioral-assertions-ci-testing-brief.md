---
date: 2026-03-04T16:30:00-05:00
source_research: memory-bank/thoughts/shared/research/2026-03-04-deep-behavioral-assertions-ci-testing.md
last_generated: 2026-03-04T20:07:27.529050+00:00
---

# Research Brief: 2026-03-04-deep-behavioral-assertions-ci-testing

## TL;DR

The industry consensus is a two-tier assertion architecture: deterministic assertions on structured output (every commit) + LLM-as-judge only for genuinely semantic criteria that can't be decomposed (nightly/weekly). For agent routing specifically, if you can get structured output via `--json-schema`, a string-equality check is sufficient — LLM judges add no signal over deterministic checks for structural correctness. The highest-value live CI test is: send a routing trigger via `claude -p --output-format stream-json`, capture which Skill tool call fires, and assert deterministically. The current plan's Phase 5 using Haiku as judge is empirically unreliable (omega < 0.7); contrastive pairs with paired positive+negated assertions are the proven pattern for routing exclusivity. Trace-first (asserting on recorded session transcripts) is the most cost-efficient approach — our Phase 3 transcript validation is already halfway there.

## Key facts / constraints

None captured.

## Key recommendations

None captured.

## Open questions

1. **stream-json + Skill tool call parsing**: The exact JSON shape of a Skill tool_use `content_block_start` event in stream-json output hasn't been verified with a real run. Need a dry-run to confirm the tool input includes the `skill` field.
2. **Contrastive pair calibration**: How many contrastive pairs are needed for statistical confidence on routing boundaries? AgentAssay's SPRT approach could help determine minimum sample size.
3. **Transcript validation log aggregation**: If Phase 3 logs per-session routing decisions, what's the right aggregation window and alerting threshold for detecting drift?

## Links

- Source research: `memory-bank/thoughts/shared/research/2026-03-04-deep-behavioral-assertions-ci-testing.md`
