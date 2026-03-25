---
type: event
created: 2026-03-21
status: active
---
# Idea Brief: Workspace Container Resilience

**Date:** 2026-03-21
**Status:** Shaped → Implementing

## Problem
Workspace containers silently die when the MCP server process restarts. The in-memory pool is lost, and `cleanup_stale_containers()` force-removes every container not in the (now-empty) pool. This destroyed ~30 minutes of in-flight work twice in one session — including a completed 3-phase PR that had to be redone from scratch.

Secondary issues: `destroy_workspace` deletes temp dirs without checking for uncommitted work, and dead containers produce opaque Docker errors instead of actionable messages.

## Constraints
- MCP server is stdio-based, stateless by default — `_State` is a plain Python object
- `cleanup_stale_containers` runs on every `create_workspace` call (orphan cleanup)
- Host temp dirs (`/tmp/workspace-*`) survive container death — only cleaned by explicit `stop()`
- Docker containers persist across MCP restarts — only the registry is lost
- `WorkspaceContainer` can be reconstructed from `config` + `workspace_id`

## Options Considered

### Disk-persisted registry
JSON file written atomically on every create/destroy. On startup, load registry, verify containers via `docker inspect`, re-adopt live ones. `cleanup_stale_containers` only kills containers not in the persisted registry.
- Gains: Survives MCP restart. Simple. Exact reconstruction.
- Costs: Can go stale on crash (mitigated by atomic write).
- Complexity: Low

### Docker labels as registry
Encode workspace metadata as Docker container labels. On startup, `docker ps --filter label=workspace=true` + `docker inspect` reconstructs the pool. No file needed.
- Gains: Zero external state. Can't go stale. Docker IS the truth.
- Costs: Need re-adopt path in WorkspaceContainer. Label encoding for config.
- Complexity: Medium

### Dirty-check guard + health detection
`destroy_workspace` refuses if `git status --porcelain` shows uncommitted changes. `run_command` detects dead containers and returns clear error.
- Gains: Prevents destroy-with-unsaved-work footgun. Clear errors on death.
- Costs: ~50ms per command for health check. One extra exec on destroy.
- Complexity: Low

## Chosen Approach
**All three combined.** Disk registry is the fast path for MCP restart recovery. Docker labels are the fallback when the registry file is missing or corrupted. Dirty-check guard enforces the existing push-before-complete rule. They're complementary layers, not alternatives.

## Key Context Discovered During Shaping
- `_state.workspaces` at `server.py:69` is the single point of failure — pure in-memory dict
- `cleanup_stale_containers` at `container.py:363` is the kill mechanism — `docker rm -f` on anything not in the exclude set
- `create_workspace` at `server.py:145-148` triggers cleanup on every call
- `destroy_workspace` at `server.py:206` calls `stop()` with no dirty check
- `stop()` at `container.py:288-307` does `docker stop -t 2` + `docker rm -f` + `shutil.rmtree` — 2s grace, unconditional delete
- `WorkspaceContainer.__init__` takes `config` + `workspace_id` — re-adoption is structurally possible
- Host temp dirs survive container death — the git state is recoverable if we don't `shutil.rmtree`

## Next Step
Implementing directly — this is infrastructure that's actively breaking our workflow. Using workspace containers to fix workspace containers is circular, so editing directly on a branch.
