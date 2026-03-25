# Idea Brief: Serverless Agent Harness on AWS Lambda Durable Functions

**Date:** 2026-03-04
**Status:** Shaped → Planning

## Problem

Our triage bot is a single-pass classifier with no project context, no learning loop, and no ability to run multi-turn conversations. More broadly, we have no reusable infrastructure for running agentic AI workflows serverlessly — each bot/automation is bespoke. Lambda Durable Functions (GA Dec 2025) provide a checkpoint-and-replay execution model purpose-built for this: fine-grained durability, zero-cost suspension during human waits, up to 1-year execution lifetime.

## Constraints

- Must be **model-agnostic** — not locked to Anthropic. Should support Claude, GPT, Bedrock-hosted models, etc.
- Should integrate an existing **agentic development SDK** if one can operate at a low enough level to work with durable Lambda's checkpoint/replay model (i.e., we can wrap individual LLM calls and tool executions in `context.step()`)
- Claude Agent SDK is ruled out — subprocess model (12s cold start), ephemeral FS, designed for long-running container processes
- Must run on AWS Lambda Durable Functions with Python 3.13
- Must support human-in-the-loop via Slack threads + `wait_for_callback`
- First agent is the triage bot (Slack → multi-turn shaping → Linear issue)
- Harness must be reusable for future agents (backlog groomer, CI failure investigator, etc.)
- 256 KB per checkpoint payload limit — need S3 offloading or store-reference pattern for large LLM responses

## Options Considered

### Claude Agent SDK on Durable Lambda
Embed Anthropic's official Agent SDK inside durable Lambda. Gets full Claude Code toolset out of the box.
- Gains: Rich built-in tools, session management, MCP support
- Costs: 12s cold start per query, opaque session state, Node.js subprocess dependency, 2-3GB memory, fights durable Lambda's execution model
- Complexity: High (impedance mismatch)

### Custom Harness with Direct API Calls
Build our own agent loop using LLM provider APIs directly. Each call is a `context.step()`, tools in `run_in_child_context()`, human waits via `wait_for_callback()`.
- Gains: Native durable Lambda fit, fine-grained checkpointing, zero-cost suspension, full control, model-agnostic
- Costs: Must build tool dispatch, conversation management, and provider abstractions ourselves
- Complexity: Medium

### Custom Harness with Agentic SDK Integration
Same as above but integrate an existing agentic SDK (e.g., Strands, LangGraph, or similar) at a low level — wrapping its LLM calls and tool dispatch in durable checkpoints.
- Gains: SDK handles model abstraction, tool schemas, conversation management; durable Lambda handles durability and suspension
- Costs: Must find an SDK whose internals are hookable enough to wrap in `context.step()`; potential abstraction leaks
- Complexity: Medium-High (depends on SDK choice)

## Chosen Approach

**Custom Harness with Agentic SDK Integration** — build the durable Lambda harness ourselves, but integrate an existing model-agnostic agentic SDK for the LLM abstraction and tool dispatch layer. This avoids reinventing model provider interfaces while keeping full control over the durability and suspension mechanics.

Key research needed during planning: which agentic SDKs (Strands, LangGraph, LiteLLM + custom loop, Mastra, etc.) can be decomposed enough to wrap individual operations in durable checkpoints.

## Key Context Discovered During Shaping

- AWS has a reference implementation: `aws-samples/sample-ai-workflows-in-aws-lambda-durable-functions` — `agent.py` with complete agentic loop + human-in-the-loop
- AWS also ships `durable_strands_agent.py` wrapping Strands as a single opaque step — simpler but loses mid-loop resume capability
- Lambda Durable Functions: $8/million operations, zero cost during waits, 256 KB checkpoint limit, Python 3.13 SDK
- Existing triage bot (`tools/triage-bot/`) provides the Slack integration, Linear client, and classifier to build on
- Full research: `memory-bank/thoughts/shared/research/2026-03-04-serverless-agent-harness.md`
- Broader AI+Linear research: `memory-bank/thoughts/shared/research/2026-03-04-ai-linear-experience-improvements.md`

## Next Step

- [Plan] → `/create_plan memory-bank/thoughts/shared/briefs/2026-03-04-serverless-agent-harness.md`
