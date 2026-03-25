---
type: event
created: 2026-03-24
status: active
---
# Idea Brief: Workspace Container Self-Sufficiency

**Date:** 2026-03-24
**Status:** Shaped -> Planning

## Problem

Workspace containers break the agent feedback loop at every stage. An agent dispatched into a container faces three systemic failures: (1) it can't diagnose what went wrong when things fail — every startup failure produces the identical "unknown -- no health file found" message, (2) it doesn't have the tools to verify its own work — `make` is in the allowlist but not installed, `node`/`npm` are installed but blocked by the allowlist, Chrome/ffmpeg are absent for Remotion rendering, and (3) errors throughout the lifecycle are silently swallowed in 7+ code paths rather than surfaced.

This was triggered by a video production session where a deep agent couldn't render-test Remotion components, couldn't diagnose why container startup failed, and left 8 unstaged files with 6 failing tests.

## Constraints

- Container runs as non-root `agent` user -- system packages must be in the Dockerfile
- Chrome Headless Shell is ~91MB compressed (~200-250MB on disk) -- meaningful but acceptable image size increase
- `set -euo pipefail` in entrypoint kills the script before health can be written -- must switch to explicit error handling in workspace mode
- Editable install hazard in workspace-mcp: must `git commit --no-verify` before any `uv run`
- Caroline ECS deployment and parallel research agents both depend on this image being solid
- MCP protocol supports `tools/list_changed` notification for schema hot-reload (stretch goal)

## Options Considered

### A. Fat Self-Sufficient Image (incremental fix)
Complete PR #186's health protocol, add Chrome/make/ffmpeg to image, fix tests, unify allowlists.
- Gains: Fixes all current pain points. Simple.
- Costs: Doesn't address future extensibility. Global allowlist stays.
- Complexity: Medium

### B. Layered Images by Workload
Multiple image tags (`:base`, `:video`, `:full`). Agent definitions specify which image.
- Gains: Smaller images for code-only agents.
- Costs: 3x maintenance. Image mismatch = new opaque failure mode.
- Complexity: High

### C. Capability Manifest + Runtime Install
Agent definitions declare required tools. `create_workspace` installs missing deps at start.
- Gains: Future-proof. No image rebuilds for new tools.
- Costs: Install cost on every start. New failure surface. Premature for 3 agent types.
- Complexity: High

### A+C Hybrid. Self-Sufficient Containers with Capability Contracts (chosen)
Fat image for current completeness + agent capability manifests for per-agent scoping and future extensibility. Plus upgraded command policy with dangerous-pattern deny list and argument-aware validation.
- Gains: Complete feedback loop now. Per-agent scoping. Extensible for future agents. Audit trail.
- Costs: ~1.2GB image. Manifest format to maintain. More validation logic.
- Complexity: Medium-High

## Chosen Approach

**A+C Hybrid: Self-Sufficient Containers with Capability Contracts** -- fat image provides everything agents need today, capability manifests in agent frontmatter provide per-agent scoping and the foundation for future extensibility. Five layers: (1) fat image with all deps, (2) agent capability manifests, (3) upgraded command policy enforcement, (4) health protocol + error surfacing completion, (5) test suite fixes.

The permissions/policy layer goes beyond the current 3-banned-flags approach to include dangerous-pattern deny lists, argument-aware validation, and audit logging -- without going to a full OPA/Cerbos policy engine (overkill for 3 agent types today).

## Key Context Discovered During Shaping

- Health file protocol has read side (`container.py:33-47`, `154-209`) but no write side -- `entrypoint.sh` never writes `.health`
- Working tree `entrypoint.sh` is a regression that REMOVES the health protocol (109 lines deleted)
- Two divergent allowlists: `validation.py:DEFAULT_ALLOWED_COMMANDS` (dead code) vs `models.py:WorkspaceConfig.allowed_commands` (enforced)
- 6 failing tests: 2 test bugs (patch nonexistent `_get_instance_type`, unmocked subprocess), 4 cascading
- 15 distinct failure scenarios across entrypoint/container/server, only 3 produce clear errors
- 7 code paths silently swallow errors (contextlib.suppress, bare except, etc.)
- Remotion needs 14 apt packages for Chrome system libs + `npx remotion browser ensure` at build time
- Remotion bundles its own ffmpeg -- no `apt install ffmpeg` needed
- Industry standard (OpenHands, SWE-agent, AIO Sandbox) is fat images, not layered
- MCP protocol has `tools/list_changed` notification for hot-reload (stretch goal)
- Cerbos for MCP is the right direction for enterprise-grade policy but premature for us

## Next Step

- [Plan] -> `/create_plan memory-bank/thoughts/shared/briefs/2026-03-24-workspace-container-self-sufficiency.md`
