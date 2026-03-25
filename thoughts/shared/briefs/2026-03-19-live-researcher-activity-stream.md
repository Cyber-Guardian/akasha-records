---
type: event
created: 2026-03-19
status: active
---
# Idea Brief: Live Researcher Activity Stream

**Date:** 2026-03-19
**Status:** Shaped → Planning

## Problem
The autoresearch dashboard shows experiment progress notes (voluntarily reported by the researcher) and phase transitions, but not the researcher's actual tool calls, container commands, and file edits. You can't watch an experiment unfold in real-time — only see sparse notes and a final result.

## Constraints
- Python SDK streams `AssistantMessage` objects with `ToolUseBlock` content — currently silently ignored in `dispatch.py`
- Tool results come back as synthetic `UserMessage` entries — also ignored
- No standalone `ToolUseMessage` type — tool calls are inside `AssistantMessage.content` blocks
- Need to import `AssistantMessage` (and possibly `UserMessage`) from `claude_agent_sdk`
- `live_experiment.json` already polled at 2.5s by the dashboard — sufficient granularity
- Container exec returns structured `ExecResult(exit_code, stdout, stderr, truncated)` — data exists but not surfaced
- Researcher may make 30+ tool calls per experiment — noise control needed

## Options Considered

### Stream-and-filter in dispatch
Capture `AssistantMessage` and `UserMessage` in dispatch.py's `receive_response()` loop. Extract tool name + truncated args/results from content blocks. Append to `live_experiment.json`'s `tool_activity` array. Dashboard renders as scrolling activity feed.
- Gains: Full visibility — every tool call including file edits, container commands, web searches
- Costs: Noisy without filtering. Tighter coupling between dispatch and live file
- Complexity: Medium

### Container command sidecar
Have `run_in_container` in mcp_bridge.py append each command + result to `live_commands.jsonl`. Dashboard shows terminal-like view.
- Gains: Simple, focused on container execution. Minimal code change
- Costs: Only container commands — misses file edits, grep, web searches. Incomplete picture
- Complexity: Low

### Callback-based activity stream
Add callback parameter to `invoke_researcher` that gets called for every SDK message. Coordinator passes callback that writes to live file.
- Gains: Clean separation, testable, swappable
- Costs: More plumbing for single consumer. Over-engineered
- Complexity: Medium

## Chosen Approach
**Stream-and-filter in dispatch** — it's the only option giving full visibility. Container sidecar would immediately leave you wanting more context. Callback is over-engineered for one consumer.

Noise control: only capture `tool_use` blocks (not text blocks), truncate stdout/stderr to ~500 chars, skip Read tool results entirely (huge and uninteresting in a feed).

## Key Context Discovered During Shaping
- `dispatch.py:119-136` — only `ResultMessage` is processed from `receive_response()` stream; `AssistantMessage` with tool calls is silently dropped
- `claude_agent_sdk` Python SDK exports `AssistantMessage`, `UserMessage`, `SystemMessage`, `ResultMessage`, `StreamEvent`, `RateLimitEvent`
- `AssistantMessage.content` contains `ToolUseBlock` entries with tool name and input args
- `sandbox.py:75-94` — `exec()` returns `ExecResult(exit_code, stdout, stderr, truncated)`
- `mcp_bridge.py:48-56` — `run_in_container` JSON-encodes ExecResult fields into MCP response text

## Next Step
- [Plan] → `/create_plan memory-bank/thoughts/shared/briefs/2026-03-19-live-researcher-activity-stream.md`
