# Idea Brief: Enable Lambda Durable Execution + Re-invoke Fallback

**Date:** 2026-03-10
**Status:** Shaped → Planning
**Linear:** ENG-2399

## Problem
The triage agent's `ask_human` tool crashes in production because `PassthroughContext.wait_for_callback()` raises `NotImplementedError` — durable execution was never enabled. Additionally, when the agent returns clarifying text without using `ask_human`, the thread is marked `IGNORED` and follow-up replies are silently dropped. Both failures prevent multi-turn conversations.

## Constraints
- Triage-agent Terraform is on provider 5.100.0 (`~> 5.0`) — needs upgrade to 6.25.0+ for `durable_config` support. State is fully isolated from main infra (already on 6.30.0), so migration is low-risk.
- `@durable_execution` decorator required on the handler — SDK wraps context to provide `DurableContext`
- Processor execution role must switch from `AWSLambdaBasicExecutionRole` to `AWSLambdaBasicDurableExecutionRolePolicy`
- `boto3 >= 1.42.1` must be explicitly packaged — Python 3.13 runtime bundles pre-1.42
- `aws-durable-execution-sdk-python` 1.3.0 is the SDK package
- Callback API uses standard boto3 Lambda client: `send_durable_execution_callback_success(CallbackId=..., Result=bytes)` — Result is binary
- Callback handler role needs `lambda:SendDurableExecutionCallbackSuccess` + `SendDurableExecutionCallbackFailure` permissions on processor ARN
- Durable functions require invocation via alias (not `$LATEST`) — processor already has `aws_lambda_alias.processor_live`
- Terraform `durable_config` block takes `execution_timeout` and `retention_period`
- Re-invoke fallback should post a visible Slack message so users know the bot is reinitializing, not resuming

## Options Considered

### Durable Execution Only (strict scope)
Enable durable execution, rewrite callback handler, package SDK + boto3. `ask_human` works properly. Follow-ups to `IGNORED` threads still silently dropped.
- Gains: Smallest blast radius, cleanest implementation
- Costs: Leaves the IGNORED-thread follow-up gap open
- Complexity: Medium

### Durable Execution + Re-invoke Fallback (full fix)
Everything above, plus: re-invoke the agent with conversation history when a follow-up arrives for an IGNORED thread. Post a visible Slack message so the user knows it's a fresh invocation.
- Gains: Closes both failure modes, clear UX signal, resilient to future edge cases
- Costs: More code in _handle_thread_reply and dispatcher
- Complexity: Medium-High

### Durable Execution + Prompt-only Fix (middle ground)
Everything in option 1, plus strengthen the prompt to always use `ask_human`. No code change for IGNORED path.
- Gains: Simpler than option 2
- Costs: Fragile — model can still skip ask_human. No safety net.
- Complexity: Low

## Chosen Approach
**Durable Execution + Re-invoke Fallback** — Closes both failure modes. The re-invoke path is also valuable beyond this bug — makes the bot resilient to any scenario where a thread ends up IGNORED but the user keeps talking. The Slack message ("Reinitializing...") gives clear UX feedback.

## Key Context Discovered During Shaping
- `handler.py:280-297` (`_adapt_context`) already detects `DurableContext` via `hasattr(context, "step")` and wraps it with `DurableContextAdapter` — Python code is ready
- `callback_handler.py:60-74` uses `lambda_client.invoke()` — needs rewrite to `send_durable_execution_callback_success()`
- `handler.py:242-277` (`_handle_thread_reply`) only handles `ISSUE_CREATED` state — all other states silently dropped
- `dispatcher.py:67-73` only passes text metadata for file attachments — image blindness is a separate issue (not in scope)
- Terraform provider 5→6 migration is low-risk: isolated state, no breaking changes for resources used
- Investigation handoff: [[2026-03-10_09-59-53_triage-bot-follow-up-failure|Triage Bot Follow-Up Failure Investigation]]
- ENG-2397 plan: [[2026-03-09-ENG-2397-fix-triage-agent-crash|Fix Triage Agent Crash Plan]]

## Next Step
- Plan → `/create_plan memory-bank/thoughts/shared/briefs/2026-03-10-ENG-2399-durable-execution.md`
