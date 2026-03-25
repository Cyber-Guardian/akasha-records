---
type: event
created: 2026-03-17
status: active
---
# Idea Brief: Agent Debug Toolkit

**Date:** 2026-03-17
**Status:** Shaped → Implementing

## Problem
Agents (debug-investigator, plan-implementer, codebase-analyzer, codebase-pattern-finder) operate blind to runtime state. The dev-harness MCP server exposes query_logs, query_traces, query_metrics, check_stack, and invoke_triage — but no agent definition included `mcpServers` or mentioned these tools in prompts. Debug-investigator's prompt referenced CloudWatch, Valkey CLI, and replay scripts it couldn't reach (no Bash tool).

## Constraints
- Agent `tools:` frontmatter is for built-in tools only — MCP access requires `mcpServers:` field
- Dev-harness tools require the observability stack to be running — agents must handle "stack is down" gracefully
- `mcpServers` grants access to all tools on a server (no per-tool filtering) — prompt guidance is the guardrail

## Options Considered

### Selective Toolkit (debug-investigator only)
Give mcpServers only to debug-investigator. Smallest change, lowest risk.
- Gains: Validates the pattern before expanding
- Costs: Plan-implementer still can't verify runtime health
- Complexity: Low

### Two-Tier Toolkit
Debug-investigator gets full query suite, plan-implementer gets just health-check guidance.
- Gains: Both key agents covered
- Costs: Prompt-level guardrail for plan-implementer usage scope
- Complexity: Medium

### Full Propagation
All investigation-capable agents get mcpServers. Prompt guidance tailored per role.
- Gains: Maximum utility. No artificial restrictions on read-only tools.
- Costs: More prompt work across agents
- Complexity: Medium

## Chosen Approach
**Full Propagation** — the downside of giving an agent access to a read-only tool it doesn't use is zero. The downside of withholding it when it could help is a missed investigation. Four agents get `mcpServers: [dev_harness]`, three get tailored prompt guidance.

## Key Context Discovered During Shaping
- No agent had ever used `mcpServers` field — this is the first usage in the repo
- Sub-agents inherit parent MCP servers by default, but explicit `mcpServers` in frontmatter is clearer and self-documenting
- `check_stack` as mandatory first step prevents agents from burning turns on unreachable services

## Next Step
- [Implementing] → no plan needed, proceeding directly (4 frontmatter additions + 3 prompt tweaks)
