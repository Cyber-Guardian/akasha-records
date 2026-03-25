# Serverless Agent Harness on AWS Lambda Durable Functions — Research Report

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

### What It Is

- Checkpoint-and-replay execution model for Lambda
- Total execution lifetime up to **1 year** (individual invocations still 15 min max)
- Zero compute charges during `wait_for_callback` suspension
- GA since Dec 2025, available in us-east-1 and 14 other regions
- Python 3.13/3.14 SDK available (`aws-durable-execution-sdk-python`)

### Core Primitives

| Primitive | What It Does |
|-----------|-------------|
| `context.step(fn, name)` | Execute `fn`, checkpoint result. On replay, return cached result without re-executing |
| `context.wait_for_callback(notify_fn, name)` | Suspend execution (zero cost). Resume when external system sends callback |
| `context.run_in_child_context(fn, name)` | Run `fn` in isolated checkpoint scope (own checkpoint log) |
| `context.parallel(fns)` | Run multiple steps concurrently |
| `context.map(fn, items)` | Fan-out with `maxConcurrency` control |
| `context.invoke(arn, payload)` | Call another Lambda with durability |

### Replay Model

1. Initial run: handler executes top-to-bottom, each `step()` persists its return value
2. On resume: handler reruns from the top, completed steps return cached results instantly
3. When SDK hits an un-checkpointed operation, it switches back to real execution
4. **Determinism required:** anything non-deterministic (`uuid()`, `datetime.now()`, `random()`) must be inside a `step()`

### Constraints

- **256 KB per checkpoint payload** — large LLM responses need S3 offloading or store-reference pattern
- **Step names must be deterministic** — SDK validates name + position against checkpoint log
- **boto3 >= 1.42.13 required** (default Lambda runtime boto3 doesn't include durable APIs)
- **Only AWS-owned KMS keys** for checkpoint encryption at launch
- **Version pinning** — if function code is updated during a suspended wait, replay runs new code against old checkpoint log. Use versioned aliases.

### Pricing

| Dimension | Cost |
|-----------|------|
| Durable operations | $8/million operations |
| Data written | Per-GB |
| Data retained | Per GB-month |
| Compute | Standard Lambda rates — **$0 during waits** |

~3x cheaper than Step Functions for equivalent workflows.

### Sources

- [AWS Lambda Durable Functions Docs](https://docs.aws.amazon.com/lambda/latest/dg/durable-functions.html)
- [Durable Execution SDK](https://docs.aws.amazon.com/lambda/latest/dg/durable-execution-sdk.html)
- [Best Practices](https://docs.aws.amazon.com/lambda/latest/dg/durable-best-practices.html)
- [AWS Blog — Build multi-step AI workflows](https://aws.amazon.com/blogs/aws/build-multi-step-applications-and-ai-workflows-with-aws-lambda-durable-functions/)
- [The Replay Model Explained](https://dev.to/aws/the-replay-model-how-aws-lambda-durable-functions-actually-work-2a79)
- [re:Invent 2025 Deep Dive](https://repost.aws/articles/ARc8wmu4l9TKywZCHX-_nn6w/re-invent-2025-deep-dive-on-aws-lambda-durable-functions)
- [AWS Pricing](https://aws.amazon.com/lambda/pricing/)

---

## Part 2: Why Not the Claude Agent SDK

### Architecture

The Agent SDK spawns the Claude Code CLI as a **child subprocess** (Node.js), communicating via JSON-lines over stdin/stdout. It is designed for long-running processes with filesystem access.

### Conflicts with Durable Lambda

| Constraint | Durable Lambda Reality | Agent SDK Requirement |
|------------|----------------------|----------------------|
| Cold start | Must be fast for replay | ~12s per `query()` (subprocess spawn) |
| State persistence | Checkpoint store (JSON) | Opaque session on local disk (ephemeral in Lambda) |
| Filesystem | Ephemeral `/tmp` | Expects persistent working directory |
| Runtime | Python only ideal | Requires Node.js + Python |
| Memory | Cost-sensitive | 2-3 GB minimum |
| Suspension model | `wait_for_callback` (zero cost) | No native suspend; subprocess stays alive |
| Checkpoint granularity | Per-step, per-tool | Per-session blob |

### What It Would Take

- Container image Lambda with Node.js + Python + Claude Code CLI
- Hybrid Session pattern: persist `session_id` in DynamoDB, `resume=session_id`
- Accept 12s overhead on every resume after a human wait
- Accept no fine-grained checkpointing (all-or-nothing session restore)
- Accept compute charges while subprocess is alive during tool execution

### Verdict

The SDK is excellent for Claude Code on developer machines and ECS containers. It fights the durable Lambda model at every turn. A custom harness using the Messages API directly is the right choice.

### Sources

- [Agent SDK Overview](https://platform.claude.com/docs/en/agent-sdk/overview)
- [Agent SDK Hosting Guide](https://platform.claude.com/docs/en/agent-sdk/hosting)
- [Agent SDK Python Reference](https://platform.claude.com/docs/en/agent-sdk/python)
- [12s overhead issue](https://github.com/anthropics/claude-agent-sdk-typescript/issues/34)
- [Inside the Claude Agent SDK](https://buildwithaws.substack.com/p/inside-the-claude-agent-sdk-from)

---

## Part 3: Custom Harness Architecture

### AWS Reference Implementation

AWS's `aws-samples/sample-ai-workflows-in-aws-lambda-durable-functions` repo contains a complete `agent.py` demonstrating:

- Agentic `while True` loop with tool dispatch
- Each LLM call wrapped in `context.step("converse", ...)` — checkpointed, not re-executed on replay
- Tool calls in `context.run_in_child_context(fn, f"tool:{name}")` — isolated checkpoint scope
- Human-in-the-loop via `wait_for_callback` inside a tool definition
- Conversation history rebuilt during replay from cached step return values

### Key Pattern: Agent Loop on Durable Lambda

```
@durable_execution
def handler(event, context):
    messages = [{"role": "user", "content": event["prompt"]}]

    while True:
        # LLM call — checkpointed
        response = context.step(
            lambda _: call_claude(messages),
            "converse"
        )
        messages.append(response.message)

        if response.stop_reason == "end_turn":
            return extract_result(response)

        # Tool dispatch — each in isolated child context
        for tool_call in response.tool_calls:
            result = context.run_in_child_context(
                lambda ctx: execute_tool(tool_call, ctx),
                f"tool:{tool_call.name}"
            )
            messages.append(tool_result(tool_call.id, result))
```

### What We'd Build On Top

Swap Bedrock converse for **Anthropic Messages API** (or Bedrock-hosted Claude). Define our own tool set:

| Tool | Purpose |
|------|---------|
| `search_linear_issues` | Find related/duplicate issues |
| `get_codebase_context` | Query repo structure, architecture, conventions |
| `get_recent_decisions` | Pull from memory-bank decision docs |
| `ask_human` | Post to Slack thread, suspend via `wait_for_callback` |
| `create_linear_issue` | Final output — well-shaped issue |

The **shaping logic** lives in the system prompt. The agent has context about the project and can run a multi-turn conversation in a Slack thread, asking clarifying questions and surfacing related work, before creating the issue.

### Execution Flow (Triage Bot Use Case)

```
Slack message arrives → SQS → Durable Lambda starts
  → step: classify intent, decide if this needs shaping or is clear enough
  → step: load project context (architecture, recent issues, conventions)
  → step: LLM generates initial framing + clarifying questions
  → step: post questions to Slack thread
  → wait_for_callback: suspend (zero cost, hours/days)

User replies in Slack → API Gateway → resume Lambda with reply
  → step: LLM refines understanding with new context
  → step: search Linear for related issues
  → either: ask more questions → wait again
  → or: converge → step: create Linear issue
  → step: post confirmation to Slack
```

### Harness vs. Agent Distinction

The **harness** is the reusable infrastructure:
- Durable Lambda execution with checkpoint/replay
- Tool registry and dispatch
- Human-in-the-loop via Slack + `wait_for_callback`
- Conversation state management (messages list rebuilt from checkpoints)
- Anthropic Messages API integration

The **agent** is the configuration on top:
- System prompt (shaping instructions, project context)
- Tool set (which tools this agent can use)
- Trigger (Slack channel, webhook, scheduled)
- Output target (Linear issue, Slack message, document)

Future agents on the same harness: backlog groomer, CI failure investigator, release notes generator, etc.

### Sources

- [aws-samples/sample-ai-workflows-in-aws-lambda-durable-functions](https://github.com/aws-samples/sample-ai-workflows-in-aws-lambda-durable-functions)
- [Hands-On with Durable Functions & Callback](https://dev.to/aws-heroes/hands-on-with-aws-lambda-durable-functions-callback-lets-build-series-4agd)
- [Building fault-tolerant applications with durable functions](https://aws.amazon.com/blogs/compute/building-fault-tolerant-long-running-application-with-aws-lambda-durable-functions/)

---

## Part 4: Gotchas and Open Questions

### Known Gotchas

1. **256 KB checkpoint limit** — LLM responses can be large. Need S3 offloading or trim responses before checkpointing. Conversation history is rebuilt from step caches, not stored as a single blob.

2. **Step naming in loops** — AWS sample uses static name `"converse"` every iteration; SDK uses positional ordering. Needs testing for long multi-turn conversations.

3. **Version pinning during waits** — if Lambda code is deployed while an execution is suspended for human reply, replay runs new code against old checkpoint log. Must use versioned aliases.

4. **Replay compute cost** — each resume replays from the top. For a 20-turn conversation, that's 20 cached step returns (fast but not free). Monitor replay latency.

5. **Determinism in tool dispatch** — if the LLM changes its mind about which tool to call (non-deterministic), but the step is already checkpointed, replay will use the cached response. This is actually correct behavior — it freezes the decision at checkpoint time.

6. **Slack thread timing** — user might reply multiple times before the bot responds. Need to batch or queue thread replies, not trigger a separate Lambda per message.

### Open Questions

1. **Anthropic API vs Bedrock-hosted Claude** — Bedrock is native to AWS, simpler IAM. Direct API gives latest models faster. Which do we prefer?

2. **Codebase context delivery** — how does the agent get project context? Options: (a) baked summary in system prompt, (b) tool that reads from S3-hosted repo snapshot, (c) MCP-style tool access to a running service, (d) embeddings in a vector store.

3. **Conversation timeout** — how long do we wait for a human reply before closing the session? 7 days? 30 days?

4. **Multi-agent on one harness** — is the harness a single Lambda with agent config passed as input, or separate Lambdas per agent type?

5. **Testing** — how do we test the durable execution locally? AWS SAM supports local invoke but durable checkpointing requires the actual service.

---

## Part 5: Related Prior Art in This Codebase

- **Existing triage bot** (`tools/triage-bot/`) — single-pass Slack → classify → Linear. Would be replaced/evolved by the first agent on the harness.
- **Helm orchestrator** (`tools/helm/`) — multi-agent decomposition over Linear issues. Different execution model (CLI, not serverless) but similar concept of orchestrating agent work.
- **Claude Code skills** (`/resolve-linear-issue`, `/create-linear-issue`) — Linear issue lifecycle management from Claude Code sessions.
- **CI failure routing** (`.github/workflows/ci-failure-linear.yml`) — could be a future agent on the harness.
- **Linear integration research** (`memory-bank/thoughts/shared/research/2026-02-16-linear-claude-code-integration.md`) — prior research on Linear MCP and agent patterns.
- **AI-enhanced Linear experience research** (`memory-bank/thoughts/shared/research/2026-03-04-ai-linear-experience-improvements.md`) — broader research that led to this direction.
