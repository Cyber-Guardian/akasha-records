---
type: event
created: 2026-03-23
status: active
---
# Idea Brief: Workspace Container Resilience — Full Overhaul

**Date:** 2026-03-23
**Status:** Shaped -> Planning

## Problem
The workspace container system has fragile error handling at every layer, and failures cascade into opaque 120s timeouts. Entrypoint uses `set -e` with no error reporting — containers die silently. The MCP server can't distinguish dead containers from slow ones. No staleness detection exists for the local Docker image. Dead container errors only surface in `run_command`, not in `git_status`/`read_file`/etc. Three different image names cause confusion. Today's incident: a stale image missing a branch guard fix wasted 30+ minutes debugging.

## Constraints
- Image build is ~30s locally (fast enough for on-demand)
- CI already auto-builds on push to main (path-filtered)
- Entrypoint serves both `workspace` and `runner` modes
- `set -euo pipefail` is valuable for runner mode
- MCP server runs as stdio process — can't do background health monitoring
- Must work for both local Docker and remote Docker (DOCKER_HOST)

## Options Considered

### Defensive Entrypoint Only
Harden entrypoint.sh — `git checkout -B`, trap errors, write failure to `/workspace/.entrypoint-error`, make `.workspace-init.sh` non-fatal.
- Gains: Containers stop dying silently
- Costs: Only fixes Layer 2. Doesn't address 120s timeout or staleness.
- Complexity: Low

### Smart Container Lifecycle Only
Fix `is_ready()` to detect dead containers immediately. Add `_handle_dead_container` to all tools. Image staleness check.
- Gains: Fast failure detection, stale images caught
- Costs: Entrypoint still dies silently — MCP server observes death but can't prevent it
- Complexity: Medium

### Combined (Entrypoint + Lifecycle)
Both above. Defense in depth.
- Gains: Entrypoint resilient AND MCP server observant
- Costs: Touches both layers
- Complexity: Medium

### Full Overhaul
Production-grade lifecycle: health check protocol (entrypoint writes structured status), fast dead-container detection in ALL tools, image version pinning (git SHA embedded in image metadata), automatic staleness detection + rebuild, structured error propagation from container to MCP caller, unified image naming.
- Gains: No more silent failures. Every failure has a clear error message within seconds.
- Costs: Touches entrypoint, Dockerfile, container.py, server.py, Makefile
- Complexity: Medium-High (but parallelizable with deep agents)

## Chosen Approach
**Full Overhaul** — this is infrastructure that every agent workflow depends on. Half-measures mean we'll be debugging the next opaque container death soon. With deep agents, the implementation cost is manageable.

## Key Context Discovered During Shaping

### Critical issues (entrypoint.sh)
- `set -e` kills container on any error — no error message reaches MCP server
- `git checkout -b` fails on existing branches (exit 128) — fixed in repo, stale in image
- `.workspace-init.sh` failure is fatal and opaque
- `generate_github_token` Python failure under `set -e` kills the container silently

### High issues (container.py + server.py)
- `is_ready()` can't distinguish dead container from slow clone — polls 120s every time: `container.py:101-116`
- `_handle_dead_container` only called in `run_command`, not `git_status`/`read_file`/`write_file`/`push_branch`/etc.: `server.py:725`
- `_is_container_running` suppresses TimeoutExpired and purges state: `server.py:223-235`
- `_restore_workspaces` at import time purges all state if Docker unavailable: `server.py:377`

### Medium issues (image build)
- No local staleness detection — manual `docker build` only
- 3 different image names: `filescience-agent-runtime` (CI), `filescience-agent-platform` (Makefile), `filescience-dev-agent-runtime` (older CI)
- SSM pull to compute host is fire-and-forget — no verification: `build-agent-runtime.yml:72-76`
- Compute host pull skipped silently if instance is stopped

### Low issues
- `_ensure_compute_host_running` failures silently suppressed: `server.py:533-534`
- `_auto_commit` runs `git add -A` — stages build artifacts: `server.py:500`

## Related Artifacts
- [[2026-03-23-scout-unified-mcp|Scout Unified MCP]] — the work that surfaced this bug
- `.claude/rules/container-self-sufficiency.md` — existing behavioral rule about missing tools
- `.claude/rules/agent-isolation.md` — workspace container usage conventions

## Next Step
- [Plan] -> `/create_plan memory-bank/thoughts/shared/briefs/2026-03-23-workspace-container-resilience.md`
