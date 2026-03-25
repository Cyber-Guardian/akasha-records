---
type: event
created: 2026-03-24
status: active
date: 2026-03-24T10:48:01-04:00
author: claude
git_commit: b1c8cbc62d65f4af6c2803e325962bafb5d1bfb2
branch: main
repository: filescience
topic: "Workspace MCP Improvements Handoff"
tags: [handoff, workspace-mcp, reliability, container]
last_updated: 2026-03-24
last_updated_by: claude
---

# Handoff: Workspace MCP Improvements

## Task(s)
- Workspace MCP reliability issues identified during video production session — **not yet tracked in Linear**
- 8 unstaged workspace-mcp files sitting in working tree with 5 failing tests — **in_progress, needs completion**

## Critical References
- `tools/workspace-mcp/src/workspace_mcp/server.py` — MCP server implementation
- `infrastructure/agent-platform/Dockerfile` — container image definition
- `.claude/rules/agent-isolation.md` — workspace usage documentation

## Recent changes
- `0d42852a` — Default `repo_url` in `create_workspace` to `https://github.com/Cyber-Guardian/filescience.git` (`server.py:507`)
- `f5abaa39` — Allowed `git stash` in scope guard (`.claude/hooks/scope_guard_bash.py`)
- PR #186 merged — container resilience: health protocol, fast dead-detection, staleness auto-rebuild

## Learnings
- **Wrong repo URL caused opaque 120s timeout.** `create_workspace` was called with `filescience/filescience` instead of `Cyber-Guardian/filescience`. Clone failed silently, container never became ready. Error message gave no hint — had to `docker logs` manually to find "repository not found."
- **MCP server caches tool schemas at startup.** After changing `create_workspace` signature (adding default repo_url), the running server kept the old schema requiring the param. Had to pass it explicitly until server restart.
- **Container image has no Chrome Headless Shell.** Deep agents working on Remotion video code can write components but can't render-test. Chrome is ~90MB and needs system libs (`libnspr4`, `libnss3`, etc.) or `npx remotion browser ensure`.
- **Resilience PR #186 shipped health file protocol.** `.health` JSON written on success/failure. But the clone-failure case may not be surfacing the error through the MCP server's ready check — needs verification.

## Artifacts
- `tools/workspace-mcp/src/workspace_mcp/server.py` — default repo_url change committed
- `.claude/hooks/scope_guard_bash.py` — stash unblocked
- `.claude/rules/agent-isolation.md` — repo URL documented

## Action Items & Next Steps

### Issue 1: Clone failure produces opaque 120s timeout
**Symptom:** `create_workspace` with wrong repo URL → "not ready after 120.0s" with no hint about clone failure.
**Root cause:** Clone fails in entrypoint, container stays alive (sleep infinity per resilience PR), but the MCP server's ready poll doesn't read the health file's error reason — it just times out.
**Fix:** In the MCP server's ready-check loop, read `/workspace/.health` on each poll iteration. If health file exists with `status: failed`, return immediately with the `error` field from the health JSON instead of waiting for timeout.
**Verify:** Check if PR #186's `is_ready()` already does this. If not, add it.

### Issue 2: Chrome Headless Shell not in container image
**Symptom:** `npx remotion still` and `npx remotion render` fail in containers with missing `libnspr4.so` and other system libraries.
**Fix:** Update `infrastructure/agent-platform/Dockerfile` — either:
  - (a) Install system deps: `apt-get install -y libnspr4 libnss3 libatk1.0-0 libatk-bridge2.0-0 libcups2 libdrm2 libxkbcommon0 libxcomposite1 libxdamage1 libxrandr2 libgbm1 libpango-1.0-0 libcairo2 libasound2`
  - (b) Run `npx remotion browser ensure` during image build (downloads ~90MB Chrome Headless Shell)
**Impact:** Enables video deep agents to render-test components, not just write code.

### Issue 3: MCP server doesn't hot-reload schema changes
**Symptom:** Changed `create_workspace` signature (added default), but running MCP server kept old schema requiring the param.
**Fix options:**
  - (a) Document that MCP server restart is needed after source changes (lowest effort)
  - (b) Watch source files and restart automatically (medium effort)
  - (c) Read defaults from a config file at call time instead of baking into function signature (avoids the problem)

### Issue 4: Unstaged workspace-mcp reliability changes
**Symptom:** 8 files sitting as unstaged changes in working tree with 5 failing tests.
**Files:**
  - `components/filescience/workspace_runtime/models.py`
  - `infrastructure/agent-platform/Dockerfile`
  - `infrastructure/agent-platform/entrypoint.sh`
  - `tools/workspace-mcp/src/workspace_mcp/server.py` (additional changes beyond the default repo_url fix)
  - `tools/dev-harness/uv.lock`, `tools/projects/autoresearch/uv.lock`, `tools/workspace-mcp/uv.lock`
  - `memory-bank/thoughts/shared/plans/2026-03-21-parallel-research-agents.md`
**Failing tests:** 5 in `test/tools/workspace_mcp/` — `test_container_resilience.py` (2), `test_registry.py` (1), `test_server.py` (2)
**Action:** Either complete the reliability changes and fix the tests, or stash them cleanly with a note about what they contain.

## Other Notes
- The `filescience-agent-runtime:latest` Docker image is the base for all workspace containers
- Workspace containers run as `agent` user (not root) — system package installation requires changes to the Dockerfile, not runtime commands
- The resilience PR (#186) added `_check_alive` to all 12 MCP tools — this is a significant improvement but the clone-failure UX path wasn't tested
