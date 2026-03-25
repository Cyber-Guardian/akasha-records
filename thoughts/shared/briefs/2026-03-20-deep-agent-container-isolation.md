---
type: event
created: 2026-03-20
status: active
---
# Idea Brief: Deep Agent Container Isolation

**Date:** 2026-03-20
**Status:** Shaped → Planning

## Problem
Parallel Claude Code sessions corrupt each other's git state because worktrees share `.git` internals (branch pointers, reflog, core.bare). Current guardrails are convention-based (rule docs, prompt instructions) — the only hard git enforcement is blocking `git stash`. Sessions switch branches in the main worktree, commit directly to local main, create duplicate worktrees on the same branch, and leave orphan branches. This kills trust in autonomous agent execution.

## Constraints
- Everything that runs on ECS must run identically locally (Docker)
- Credentials unified via Secrets Manager — substrate (Docker vs Fargate) is the only variable
- Correctness over speed — fresh clone, not bind-mount
- Deep vs shallow boundary: anything vulnerable to write contention / mutation = deep agent
- Autoresearch MCP bridge pattern (`mcp_bridge.py`) already exists and proves the local model
- ECS agent platform (`runner.py`) already proves the fresh-clone + push + PR model

## Options Considered

### Persistent Container Workspace (docker exec)
Containers as live workspaces with `sleep infinity`. Parent uses raw `docker exec` to inspect, test, review. Push when satisfied.
- Gains: Simple, proven by autoresearch sandbox
- Costs: No structured interface, shell parsing, harder on ECS (ECS Exec latency)
- Complexity: Medium

### MCP Bridge Workspace
Same persistent container, but exposes an MCP server. Parent connects to structured tools: `run_command`, `read_file`, `write_file`, `git_status`, `run_tests`, `push_and_pr`. Sub-agent implements via MCP tools. Parent uses same MCP connection to inspect and test.
- Gains: Structured interaction, command allowlists, same interface local and ECS, machine-to-machine reliability, direct evolution of existing `mcp_bridge.py`
- Costs: Must build generalized MCP bridge, network plumbing for ECS
- Complexity: Medium-high

### Fire-and-Push (existing ECS model)
Container runs autonomously: clone → implement → push → PR → exit. No interactive inspection.
- Gains: Already exists on ECS, simplest
- Costs: No inspect-before-push, slow feedback loop, can't test divergent state from parent
- Complexity: Low

## Chosen Approach
**MCP Bridge Workspace** — build the structured MCP interface from the start rather than evolving from raw docker exec. The autoresearch bridge pattern generalizes naturally. The structured interface is required anyway for ECS parity (where raw exec is clunky). Sub-agents implement inside the container via MCP tools, then hand control back to the parent agent who can inspect state and run tests in the same container before approving the push.

## Key Context Discovered During Shaping
- `scope_guard_bash.py` blocks `git stash` but NOT `git checkout -b`, `git switch -c`, or branch switching — the biggest corruption vectors are unenforced
- 11 stale worktrees on disk, 3 on the same branch (`feat/agent-platform-infra`), local main diverged 15 commits from origin — all symptoms of shared `.git` corruption
- Autoresearch sandbox already uses `sleep infinity` + `docker exec` + MCP bridge with command allowlists (`sandbox.py`, `mcp_bridge.py`)
- ECS agent platform already uses fresh clone + `agent/{job_id}` branch naming + push + PR (`runner.py`, `git_ops.py`)
- Helm review/conflict-resolution agents run via `subprocess.run(["claude", "-p", ...])` in the main CWD with no isolation — another corruption vector
- Agent taxonomy: shallow (read-only, in-process) vs deep (write/mutate, containerized) based on operation set, not agent type

## Next Step
- Plan → `/create_plan memory-bank/thoughts/shared/briefs/2026-03-20-deep-agent-container-isolation.md`
