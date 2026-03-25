# Idea Brief: Slack Relay Hook for Claude Code

**Date:** 2026-02-26
**Status:** Shaped -> Planning

## Problem
When Claude or Cyrus hits a decision point (`AskUserQuestion`), it blocks waiting for terminal input. If Jordan isn't watching the terminal, work stalls silently. For Cyrus (headless), there's no terminal at all — the session hangs until timeout. This kills the async agent execution model.

## Constraints
- `AskUserQuestion` is a tool call — `PreToolUse` hooks can intercept it
- Hooks can return `permissionDecision: "deny"` with `permissionDecisionReason` (shown to Claude) and `additionalContext` (injected into context) — this is the mechanism to feed answers back
- `additionalContext` on deny is underdocumented — `permissionDecisionReason` is the reliable channel
- Hook timeout default is 600s (10 min); configurable per hook, no documented hard cap
- Existing triage-bot has Slack bot token in AWS Secrets Manager + `slack-sdk>=3.33`
- Slack MCP tools (`slack_send_message`, `slack_read_thread`) are also available but polling in Claude's context is wasteful
- Hooks support `uv run --script` shebang for self-contained deps (pattern used by ruff/ty validators)

## Options Considered

### Hook Relay (PreToolUse intercept + Slack round-trip)
A PreToolUse hook intercepts `AskUserQuestion`, posts the question to a Slack channel, polls for a thread reply, and returns the answer via `permissionDecisionReason` — denying the original tool call (since the answer was already collected). Claude sees the denial reason containing the user's answer and continues without terminal input.
- Gains: Fully transparent to Claude/agents (no code changes), works for local + Cyrus, bidirectional
- Costs: Sync blocking hook (holds the agentic loop while waiting), needs Slack bot token accessible to hook, needs response parsing
- Complexity: Medium

### MCP Poll Loop (Claude posts to Slack and reads the response itself)
Claude uses `slack_send_message` + sleep + `slack_read_thread` in a loop. Enforced via CLAUDE.md convention.
- Gains: Zero new infrastructure
- Costs: Burns context, can't enforce usage, ugly polling, each iteration costs an API turn
- Complexity: Low but fragile

### Webhook Bridge (async service relays between Slack and file)
Hook fires async notification. Separate service watches Slack Events API, writes reply to a file. Hook polls the file.
- Gains: Clean separation, scalable to richer interactions
- Costs: Needs a running service, over-engineered for v1
- Complexity: High

## Chosen Approach
**Hook Relay** — it's the sweet spot. Zero changes to agent code, uses the established `uv run --script` hook pattern, and the only new artifact is a single Python script + config. The hook blocks synchronously while polling Slack, which is fine since the agentic loop can't do anything useful without the human's answer anyway.

## Key Context Discovered During Shaping
- PreToolUse hooks receive full `tool_input` on stdin including `questions` array with options — enough to format a rich Slack message
- `permissionDecision: "deny"` + `permissionDecisionReason` is the reliable feedback channel (shown to Claude)
- `additionalContext` was added in v2.1.9 and may also work alongside deny — worth trying as a secondary channel
- Bug in v2.0.76 where PreToolUse hooks on AskUserQuestion caused empty responses was fixed — safe to use now
- Existing hook scripts in `.claude/hooks/validators/` provide a clean template (uv script shebang, stdin JSON parsing, file logging, JSON output)
- Slack bot token is in AWS Secrets Manager (`slack-bot-token`) for triage-bot; hook should support both env var and secrets manager

## Next Step
- [Plan] -> `memory-bank/thoughts/shared/plans/2026-02-26-slack-relay-hook.md`
