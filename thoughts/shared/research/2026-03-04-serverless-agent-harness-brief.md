---
date: 2026-03-04T17:37:15.608753+00:00
source_research: memory-bank/thoughts/shared/research/2026-03-04-serverless-agent-harness.md
last_generated: 2026-03-04T17:37:15.608753+00:00
---

# Research Brief: 2026-03-04-serverless-agent-harness

## TL;DR

**Date:** 2026-03-04
**Status:** RESEARCH COMPLETE
**Scope:** Architecture for a lightweight agentic runtime on durable Lambda, with triage bot as first use case
---
## Executive Summary
AWS Lambda Durable Functions (GA Dec 2025) provide a checkpoint-and-replay execution model that is a natural fit for multi-turn AI agents that need to suspend for human input. AWS has already published a reference implementation (`agent.py` in `aws-samples/sample-ai-workflows-in-aws-lambda-durable-functions`) demonstrating an agentic tool-calling loop with human-in-the-loop via `wait_for_callback`.
We evaluated two paths: (1) embedding the Claude Agent SDK inside durable Lambda, and (2) building a lightweight custom harness using the Anthropic Messages API directly. The Agent SDK is a poor fit — it spawns a Node.js subprocess with 12s cold starts, stores session state on ephemeral disk, and is designed for long-running container processes. The custom harness path is native to durable Lambda's execution model and gives fine-grained checkpointing, zero-cost suspension, and full control over the tool set.
The harness is a reusable piece of infrastructure. The triage bot (Slack → multi-turn shaping → Linear issue) is the first agent running on it, but any multi-turn agentic workflow can be built on the same foundation.
---
## Part 1: Lambda Durable Functions — Key Facts

## Key facts / constraints

None captured.

## Key recommendations

None captured.

## Open questions

None

## Links

- Source research: `memory-bank/thoughts/shared/research/2026-03-04-serverless-agent-harness.md`
