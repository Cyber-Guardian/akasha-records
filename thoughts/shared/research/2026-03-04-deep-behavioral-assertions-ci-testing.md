---
date: 2026-03-04T16:30:00-05:00
researcher: Claude
git_commit: bf8dd50
branch: main
repository: filescience
topic: "What behavioral assertions are worth paying LLM cost for in CI when testing Claude Code extensions?"
tags: [deep-research, testing, extensions, ci, promptfoo, agent-eval, behavioral-testing]
status: complete
research_depth: standard
iterations_completed: 3
last_updated: 2026-03-04
last_updated_by: Claude
---

# Deep Research: Behavioral Assertions for CI Testing of Claude Code Extensions

## Research Question
What behavioral assertions are worth paying LLM cost for in CI when testing Claude Code extensions? Given Phases 1-2 cover structural checks at zero cost, what should Phases 4-5 actually test?

## Summary
The industry consensus is a two-tier assertion architecture: deterministic assertions on structured output (every commit) + LLM-as-judge only for genuinely semantic criteria that can't be decomposed (nightly/weekly). For agent routing specifically, if you can get structured output via `--json-schema`, a string-equality check is sufficient — LLM judges add no signal over deterministic checks for structural correctness. The highest-value live CI test is: send a routing trigger via `claude -p --output-format stream-json`, capture which Skill tool call fires, and assert deterministically. The current plan's Phase 5 using Haiku as judge is empirically unreliable (omega < 0.7); contrastive pairs with paired positive+negated assertions are the proven pattern for routing exclusivity. Trace-first (asserting on recorded session transcripts) is the most cost-efficient approach — our Phase 3 transcript validation is already halfway there.

## Perspectives Explored
1. **Industry eval frameworks** — Revealed promptfoo's `javascript` assertions as the key unlock: deterministic routing checks without LLM cost. `not-` prefix enables exclusivity.
2. **Real-world CI setups** — Anthropic advises outcome-based over sequence-based. AgentAssay's trace-first pattern achieves 78-100% cost reduction. No public evidence of Anthropic testing their own extensions.
3. **Signal-to-noise economics** — Haiku unreliable as judge (omega 0.42-0.71). LLM judges only pay for themselves on semantic criteria (tone, hallucination). No evidence supports per-commit LLM judge for stable routing.
4. **Our failure modes** — No production regressions documented. Five Phase 1-2 blind spots identified, but only two (routing mis-classification, additionalContext abandonment) are catchable by live CI tests.
5. **Assertion design patterns** — Contrastive pairs + paired assertions + confusion matrix framing. `stream-json` captures tool calls but is incompatible with `--json-schema` — use `tee` + `jq` to extract both.

## Detailed Findings

### What promptfoo can actually do (beyond what the plan uses)

The plan only uses `llm-rubric`, `contains`, and `not-contains`. Promptfoo has a much richer assertion surface:

- **`javascript` assertions**: Run arbitrary JS code on the output, returning pass/fail. Can parse JSON, check field values, validate routing logic — all deterministic, zero LLM cost. This is the key unlock for routing tests.
- **`not-` prefix**: Works on ANY assertion type (`not-contains`, `not-icontains`, `not-regex`). Enables exclusivity testing: `icontains: /investigate` + `not-icontains: /create_plan`.
- **`tool-call-f1`**: Measures precision/recall over expected tool sets. Catches both missing calls and extra calls.
- **`is-valid-function-call`**: Validates output against a function JSON schema.
- **`model-graded-closedqa`**: Binary yes/no rubric (more constrained than open `llm-rubric`, less flaky).
- **`cost` and `latency` guards**: Gate on token spend and response time.

**Implication for Phase 5**: Most `llm-rubric` assertions in the plan can be replaced with `javascript` assertions on structured output, eliminating LLM-judge cost and flakiness.

### When LLM-judge cost pays for itself

Research is unambiguous: LLM judges add value **only** for criteria that are fundamentally semantic and can't be decomposed into deterministic checks:

| Criterion | Deterministic? | LLM judge needed? |
|-----------|---------------|-------------------|
| Route selected = expected route | Yes (string equality) | No |
| Correct tool called | Yes (tool-call-f1) | No |
| JSON output matches schema | Yes (is-json) | No |
| Response tone is professional | No | Yes |
| Answer is faithful to context | No | Yes |
| No hallucinated file paths | Partially (regex) | Borderline |
| Routing decision is well-reasoned | No | Yes (but low value) |

**Decision rule**: If you can express the assertion as a function `f(output) -> bool`, don't use an LLM judge. Period.

**Frequency**: Deterministic assertions every commit. LLM-judge assertions at most on merge-to-main or nightly. No evidence supports per-commit LLM judge for stable routing.

**Cost**: Haiku ~$0.001-0.003/call, Sonnet ~$0.01-0.03/call, deterministic: $0.

### LLM-judge reliability data

- **McDonald's omega**: 0.42-0.71 for smaller models (Haiku-class), >0.95 for Claude Sonnet/GPT-4o on well-structured binary tasks
- **Known biases**: Position bias (response order affects verdict), self-preference bias (models prefer own output), verbosity bias (mostly resisted by modern models)
- **Temperature 0**: Near-perfect self-consistency but masks genuine uncertainty
- **Multi-judge ensemble**: ~5% accuracy gain over simple majority vote (ReasoningTree search pattern)
- **Critical for our plan**: Phase 5 uses Haiku as judge — this is empirically the wrong choice for nuanced routing rubrics. Either upgrade to Sonnet (5-10x cost) or replace with deterministic assertions.

### The highest-value live CI tests

Five failure modes exist that Phases 1-2 cannot catch:

1. **Routing mis-classification** — model selects wrong skill for a trigger phrase
2. **additionalContext abandonment** — hook output is correct but model ignores it
3. **Skill-CLAUDE.md semantic conflicts** — instruction contradictions
4. **scope_guard interaction** — cross-hook blocking effects
5. **Context-window saturation** — loaded context pushes instructions out of attention

Only #1 and #2 are catchable by live CI tests. #3-5 require full session transcript analysis or manual testing.

**For #1 (routing mis-classification)**, the highest-signal test:

```bash
# Use stream-json to capture actual tool calls
claude -p "debug this error in the auth module" \
  --output-format stream-json \
  --max-turns 2 \
  --max-budget-usd 0.30 \
  --no-session-persistence \
  --dangerously-skip-permissions \
  | tee /tmp/trace.jsonl \
  | jq -c 'select(.type == "content_block_start" and .content_block.type == "tool_use") | .content_block.name'
# Assert: "Skill" appears in tool calls (not just mentioned in text)
```

Then parse the Skill tool_use input to verify `skill: "investigate"` was the argument — not `/create_plan` or `/shape`.

**For #2 (additionalContext abandonment)**, test that identity.md content appears in Claude's first response:

```bash
claude -p "What project are you working on?" \
  --output-format json \
  --max-turns 1 \
  --max-budget-usd 0.20 \
  --no-session-persistence \
  --dangerously-skip-permissions \
  | jq -r '.result'
# Assert: mentions "filescience" or content from identity.md
```

### Assertion design patterns for routing

**Contrastive pairs** — the strongest routing test design:

Test pairs of similar inputs that should route differently. This surfaces confusion at decision boundaries:

| Input A | Expected Route A | Input B | Expected Route B |
|---------|-----------------|---------|-----------------|
| "shape this idea" | /shape | "plan this feature" | /create_plan |
| "debug this error" | /investigate | "what does this code do?" | /research_codebase |
| "ENG-2095" | /resolve-linear-issue | "create an issue for X" | /create-linear-issue |
| "research deeply" | /deep_research | "where is the throttling code?" | /research_codebase |

Each pair shares vocabulary but differs in the routing-critical signal. A regression that confuses Route B with Route C will fail one side of the pair.

**Paired assertions** (exclusivity):

For each test case, assert BOTH what should happen AND what should NOT happen:
- `contains: /investigate` AND `not-contains: /create_plan` AND `not-contains: /shape`
- This catches "partially correct" routing where the model mentions the right skill but also invokes a competing one.

**Confusion matrix gating**:

Treat the routing table as a multi-class classifier. Compute per-route precision/recall across the test suite. Gate CI on per-route thresholds (e.g., Route G must have >80% precision) rather than aggregate accuracy.

### Trace-first is the most cost-efficient pattern

AgentAssay (arxiv 2603.02601) demonstrates that recording real agent runs and asserting offline achieves 78-100% cost reduction vs live testing. Their approach:

1. Record traces during real sessions (we already have JSONL transcripts)
2. Run 4/6 test types at zero additional token cost: coverage, contracts, metamorphic relations, mutation
3. Use SPRT (Sequential Probability Ratio Testing) for the remaining test types, cutting trials 52-69%
4. Behavioral fingerprinting maps traces to compact vectors for multivariate regression detection (86% detection power)

**Our Phase 3 transcript validation is already the trace-first pattern.** The research suggests enriching it rather than building expensive live tests in Phases 4-5.

### Concrete recommendations for Phases 4-5 redesign

**Phase 4 — Replace routing coherence test, keep startup-reads test:**

- **Remove**: The routing coherence test (asks Claude to list commands, checks files exist) — Phase 1 already does this better statically.
- **Replace with**: Tool-call-trace routing test — send a routing trigger via `stream-json`, capture which Skill tool call fires, assert deterministically.
- **Keep**: The startup-reads test — it validates live behavior (identity.md actually loaded) that Phase 1 can't check.
- **Add**: Secret validation step — check `ANTHROPIC_API_KEY` exists before running LLM calls.

**Phase 5 — Restructure around deterministic assertions:**

- **Replace Haiku judge** with `javascript` assertions on structured output where possible.
- **Use contrastive pairs** for routing boundary testing (5-8 pairs covering the main ambiguity boundaries).
- **Use `not-contains`** on every test case for exclusivity.
- **Keep `llm-rubric`** (with Sonnet, not Haiku) only for the 2-3 genuinely semantic assertions that can't be decomposed.
- **Run weekly** (not on every push) — routing table changes rarely.

**Phase 3 — Enrich transcript validation (the highest-ROI investment):**

- Add richer property assertions to the Stop hook transcript parser
- Assert on actual Skill tool calls in transcripts (not just Read/Write)
- Track per-session routing decisions in a local log for aggregate analysis
- This is trace-first at zero additional LLM cost

## Key Sources

### Academic
- [AgentAssay: Token-Efficient Regression Testing (arxiv 2603.02601)](https://arxiv.org/html/2603.02601)
- [Can You Trust LLM Judgments? (arxiv 2412.12509)](https://arxiv.org/html/2412.12509v2)
- [Auditing Multi-Agent LLM Reasoning Trees (arxiv 2602.09341)](https://arxiv.org/html/2602.09341v1)
- [Intent Detection in the Age of LLMs (arxiv 2410.01627)](https://arxiv.org/html/2410.01627v1)
- [LLM-as-Judge on a Budget (arxiv 2602.15481)](https://arxiv.org/html/2602.15481)
- [Justice or Prejudice? Quantifying LLM Judge Biases](https://llm-judge-bias.github.io/)

### Industry
- [Anthropic: Demystifying Evals for AI Agents](https://www.anthropic.com/engineering/demystifying-evals-for-ai-agents)
- [Pragmatic Engineer: A Pragmatic Guide to LLM Evals](https://newsletter.pragmaticengineer.com/p/evals)
- [Evidently AI: LLM-as-a-Judge Complete Guide](https://www.evidentlyai.com/llm-guide/llm-as-a-judge)
- [Patronus AI: AI Agent Routing Testing](https://www.patronus.ai/ai-agent-development/ai-agent-routing)

### Promptfoo
- [Assertions and Metrics](https://www.promptfoo.dev/docs/configuration/expected-outputs/)
- [Deterministic Metrics](https://www.promptfoo.dev/docs/configuration/expected-outputs/deterministic/)
- [JavaScript Assertions](https://www.promptfoo.dev/docs/configuration/expected-outputs/javascript/)
- [Evaluate Coding Agents](https://www.promptfoo.dev/docs/guides/evaluate-coding-agents/)
- [CI/CD Integration](https://www.promptfoo.dev/docs/integrations/ci-cd/)

### Claude Code
- [CLI Reference](https://code.claude.com/docs/en/cli-reference)
- [Headless/Programmatic Usage](https://code.claude.com/docs/en/headless)
- [GitHub Actions](https://code.claude.com/docs/en/github-actions)
- [Streaming Output (Agent SDK)](https://platform.claude.com/docs/en/agent-sdk/streaming-output)
- [claude-code-action](https://github.com/anthropics/claude-code-action)

### Codebase
- `CLAUDE.md:17-30` — routing table
- `.claude/hooks/session_start.py:99-113` — additionalContext injection
- `.claude/hooks/scope_guard.py:109-119` — PreToolUse block logic
- `.claude/settings.json:96` — Stop hook agent prompt
- `memory-bank/thoughts/shared/plans/2026-03-04-extension-validation-pipeline.md` — the plan being redesigned

## Open Questions
1. **stream-json + Skill tool call parsing**: The exact JSON shape of a Skill tool_use `content_block_start` event in stream-json output hasn't been verified with a real run. Need a dry-run to confirm the tool input includes the `skill` field.
2. **Contrastive pair calibration**: How many contrastive pairs are needed for statistical confidence on routing boundaries? AgentAssay's SPRT approach could help determine minimum sample size.
3. **Transcript validation log aggregation**: If Phase 3 logs per-session routing decisions, what's the right aggregation window and alerting threshold for detecting drift?

## Research Metadata
- Depth mode: standard
- Iterations completed: 3 / 5
- Termination reason: gaps exhausted
- Manifest: `.claude/deep-research/2026-03-04-behavioral-assertions-ci-testing.md`
