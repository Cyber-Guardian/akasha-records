# Idea Brief: Durable Lambda Invocation Topology

**Date:** 2026-03-04
**Status:** Shaped → Planning

## Problem
The serverless agent harness plan assumes SQS ESM → durable processor Lambda, but SQS ESM imposes a hard 15-minute cap on total durable execution time (enforced at `CreateEventSourceMapping` creation). The triage agent needs `wait_for_callback` for human-in-the-loop patterns (duplicate confirmation, clarification) where responses come in minutes to hours, making SQS ESM fundamentally incompatible with long-running durable executions.

## Constraints
- SQS ESM → durable Lambda: hard 15-minute cap, enforced at ESM creation time
- Async direct invoke (`InvocationType: Event`): up to 1 year — the only path supporting long `wait_for_callback`
- AWS-recommended pattern: SQS ESM → standard Lambda → async invoke → durable Lambda
- Lambda Destinations don't work with durable functions (only DLQs supported)
- Current receiver already does `sqs.send_message()` — infrastructure exists
- Button clicks and thread replies currently share the same SQS queue
- Callback handler needs `callback_id` to call `SendDurableExecutionCallbackSuccess`

## Options Considered

### A: Receiver-as-Intermediary (Direct Invoke)
Receiver Lambda replaces `sqs.send_message()` with `lambda.invoke(InvocationType='Event')` directly. Removes the events queue for the durable path entirely.
- Gains: Simplest topology, one fewer infra component, lowest latency
- Costs: Loses SQS FIFO dedup/DLQ/backpressure — must reimplement via Lambda Powertools idempotency + DynamoDB. Receiver changes required. Async invoke failures harder to observe.
- Complexity: Low-Medium

### B: Keep SQS as Buffer, Add Dispatcher Lambda
Keep SQS FIFO between receiver and processor. Add a thin dispatcher Lambda (~20 LOC) triggered by SQS ESM that async-invokes the durable processor and returns immediately. Receiver stays unchanged except for adding emoji reaction.
- Gains: SQS FIFO dedup/DLQ/backpressure all preserved for free. Receiver untouched. Clean separation of concerns. SQS retries dispatcher up to 3x before DLQ.
- Costs: One extra Lambda (~20 lines). One extra hop of latency (~50-100ms).
- Complexity: Low

## Chosen Approach
**B: SQS + Dispatcher** — the dispatcher is trivial boilerplate but buys SQS's built-in dedup, DLQ, backpressure, and retry semantics for free. These would all need to be reimplemented manually in option A. The receiver adds an emoji reaction (`:eyes:`) for instant user feedback before enqueue.

Additional decisions:
- Callback handler is a simple standard Lambda (not durable) — reads `callback_id` from DynamoDB, calls `SendDurableExecutionCallbackSuccess`
- Lambda Powertools idempotency on the durable processor as a safety net (SQS FIFO is primary dedup layer)
- Emoji reaction happens in the receiver inline, before SQS send

## Key Context Discovered During Shaping
- SQS ESM hard 15-min cap: enforced at `CreateEventSourceMapping` creation, not at runtime — `ExecutionTimeout` > 15 min causes API error
- Async direct invoke supports up to 1 year execution — the only viable trigger for long `wait_for_callback`
- `ReportBatchItemFailures` does work with durable Lambdas (reviewer B5 was wrong), but irrelevant since we're adding a dispatcher
- Current receiver does not read DynamoDB at all — it's a pure fan-in/enqueue layer (reviewer B3's TOCTOU premise was wrong about current state)
- Current processor timeout is 60s, receiver is 10s
- Durable processor deployed as versioned alias to prevent code drift during suspended waits

## Next Step
- Plan → update Phase 5 of `memory-bank/thoughts/shared/plans/2026-03-04-serverless-agent-harness.md` with the new invocation topology, incorporating all validated blocker fixes from adversarial review
