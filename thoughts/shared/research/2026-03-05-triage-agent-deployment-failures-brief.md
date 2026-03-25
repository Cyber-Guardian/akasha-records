---
date: 2026-03-05
source_research: memory-bank/thoughts/shared/research/2026-03-05-triage-agent-deployment-failures.md
last_generated: 2026-03-06T00:57:46.640018+00:00
---

# Research Brief: 2026-03-05-triage-agent-deployment-failures

## TL;DR

## Context
Deployed the rewritten Polylith triage agent (`tools/bases/tooling/triage_agent/`) to Lambda on 2026-03-05. Hit four sequential runtime errors, each requiring a rebuild + redeploy cycle. All were fixable in minutes but none were caught before deployment.
PR with fixes: #69 (`fix/triage-agent-deployment`)
## The Four Failures
### 1. ANTHROPIC_API_KEY not set
**Error:** `UserError: Set the ANTHROPIC_API_KEY environment variable`
**Root cause:** PydanticAI's `AnthropicProvider` reads `ANTHROPIC_API_KEY` from the environment at `Agent()` construction time. Our code fetches secrets from Secrets Manager at runtime, but the provider init happens first.
**Fix:** Fetch from Secrets Manager and inject into `os.environ` before constructing the Agent.
**Why tests missed it:** `test_handler_runs_agent` mocks `build_agent` entirely — never constructs a real `Agent()`.
### 2. Tool signatures wrong — bare TriageDeps instead of RunContext[TriageDeps]

## Key facts / constraints

None captured.

## Key recommendations

None captured.

## Open questions

None

## Links

- Source research: `memory-bank/thoughts/shared/research/2026-03-05-triage-agent-deployment-failures.md`
