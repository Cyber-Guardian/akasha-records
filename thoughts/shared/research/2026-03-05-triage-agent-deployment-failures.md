---
title: Triage Agent Deployment Failures — Post-Mortem
date: 2026-03-05
type: research
status: actionable
tags: [triage-agent, testing, deployment]
related:
  - memory-bank/thoughts/shared/plans/2026-03-04-serverless-agent-harness.md
touched: 2026-03-05
---

# Triage Agent Deployment Failures

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

**Error:** `UserError: First parameter of tools that take context must be annotated with RunContext[...]`
**Root cause:** Tools were defined as `async def search_linear_issues(query: str, deps: TriageDeps)` but PydanticAI's `agent.tool()` requires `ctx: RunContext[TriageDeps]` as the first parameter.
**Fix:** Changed all tool signatures to use `RunContext[TriageDeps]` and access deps via `ctx.deps`.
**Why tests missed it:** Tool tests call functions directly from the registry via `get_tool("search_linear_issues")` — they never go through `agent.tool()` registration. The `build_agent` path in handler tests is mocked.

### 3. LambdaContext has no .step() — durable execution not enabled

**Error:** `AttributeError: 'LambdaContext' object has no attribute 'step'`
**Root cause:** `_adapt_context()` wrapped everything non-CheckpointContext in `DurableContextAdapter`, which delegates to `.step()`. But durable execution isn't enabled on the Lambda yet (commented out in Terraform), so the raw LambdaContext has no `.step()`.
**Fix:** Added `PassthroughContext` that executes functions directly. `_adapt_context` checks `hasattr(context, "step")` to pick the right wrapper.
**Why tests missed it:** `test_wraps_unknown_context` uses `MagicMock()` which **auto-creates any attribute** including `.step()`. The exact attribute that was missing in production was silently provided by the mock.

### 4. deps not forwarded to agent.run()

**Error:** `AttributeError: 'NoneType' object has no attribute 'linear_api_key'` (ctx.deps was None)
**Root cause:** `run_agent()` calls `agent.run(user_prompt, ...)` but never passes `deps=`. PydanticAI needs `deps=` for tools using `RunContext[TriageDeps]`.
**Fix:** Added `deps` parameter to `run_agent`, `_run_agent_turn_sync`, and `_run_agent_turn`. Handler passes `deps=deps`.
**Why tests missed it:** `test_handler_runs_agent` mocks `run_agent` — never exercises the real `run_agent -> agent.run(deps=)` path. Tool tests call functions with `deps=` directly, bypassing RunContext.

## Root Pattern: Lasagna Mocking

All four failures share the same testing anti-pattern: **every layer mocks the layer below it**, so integration wiring is never tested.

```
handler tests    -> mock build_agent, mock run_agent
tool tests       -> call get_tool() directly, pass deps as kwarg
adapter tests    -> MagicMock() auto-creates missing attributes
runner tests     -> (none exist)
```

Each unit works in isolation. The connections between them — where all four bugs lived — are never exercised.

Additionally, `make deploy` depends on `build` only, not `test`. Even the existing tests (which would have caught failure #2 via signature mismatch) were never run.

## Action Items

### P0: Gate deployment on tests

Update `tools/triage-bot/Makefile`:
```makefile
deploy: test build
    cd terraform && tofu apply -auto-approve
```

### P1: Fix the 4 broken tool tests

The tool tests in `test/bases/triage_agent/test_tools.py` still call `fn(deps=_make_deps())` but signatures now use `RunContext[TriageDeps]`. They need to either:
- Create a real `RunContext` with deps, or
- Call tools through a real PydanticAI Agent with `FunctionModel`

### P2: Add integration smoke test

A single test that exercises the real path without mocking construction/wiring:

```python
def test_agent_construction_and_tool_registration():
    """Verify build_agent succeeds and tools are callable via PydanticAI."""
    os.environ["ANTHROPIC_API_KEY"] = "test-key"  # pragma: allowlist secret
    config = AgentConfig(
        system_prompt="You are a test agent.",
        tools=["search_linear_issues", "create_linear_issue", "ask_human"],
        model="test",  # PydanticAI's TestModel
    )
    deps = _make_deps()
    agent = build_agent(config, deps)

    # Verify tools are registered and callable
    assert agent is not None
    # Run one turn with FunctionModel to verify deps flow
```

This single test would have caught failures #1, #2, and #4.

### P3: Fix MagicMock auto-attribute problem

In `test_handler.py`, change:
```python
# BAD — MagicMock auto-creates .step(), hiding the real bug
mock_ctx = MagicMock()

# GOOD — spec constrains mock to the real interface
mock_ctx = MagicMock(spec=object)  # plain object has no .step()
```

Add explicit test for non-durable Lambda:
```python
def test_adapt_context_without_durable_execution():
    """LambdaContext without .step() gets PassthroughContext."""
    lambda_ctx = type("LambdaContext", (), {"function_name": "test"})()
    result = _adapt_context(lambda_ctx)
    assert isinstance(result, PassthroughContext)
```

### P4: Add runner tests

`tools/components/tooling/durable_harness/runner.py` has zero test coverage. At minimum:
- `run_agent` with `PassthroughContext` + `FunctionModel` + real deps
- Verify deps arrive in tool calls (ctx.deps is not None)
- Verify multi-turn loop with deferred tools

## Takeaway

The deployment pipeline had no test gate, and the test suite tested units in isolation without ever exercising the integration seams. Four failures, four rebuild+deploy cycles, all preventable with one integration test and `deploy: test build`.
