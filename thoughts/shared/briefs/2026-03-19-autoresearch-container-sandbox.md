---
type: event
created: 2026-03-19
status: active
---
# Idea Brief: Autoresearch Container Sandbox with Intelligent Compression

**Date:** 2026-03-19
**Status:** Shaped → Planning

## Problem

Researchers can write multi-file projects (ENG-2650 removed file gates) but can't run or test them before committing. Without execution, the researcher flies blind — writing code it can't verify, relying entirely on the evaluator to catch errors post-commit. This limits experiment quality and wastes budget on trivially broken experiments.

## Constraints

- Bash is fully blocked (RESEARCHER_DISALLOWED_TOOLS) — the existing security boundary
- MCP tool addition is straightforward: add to build_researcher_mcp_server, add to allowed_tools
- Container start/stop fits cleanly: after create_worktree (start), in finally block (stop)
- Target: Docker Desktop locally, ECS later — same image both places
- Industry standard is per-session container (OpenHands, SWE-agent, Devin)
- Shell injection is #1 MCP vulnerability (43% of CVEs) — must avoid shell=True, use allowlist

## Options Considered

### Per-Experiment Container + MCP Tool Proxy + Intelligent Compression
One Docker container per experiment. Worktree bind-mounted. MCP run_in_container tool executes allowlisted commands via docker exec. Haiku micro-agent compresses stdout/stderr with confidence scoring before returning to researcher. Threshold-based: short outputs pass through raw.
- Gains: Full isolation, token shaving, permission control, ECS-portable, confidence-scored outputs
- Costs: Docker overhead per experiment (~1-2s start), image maintenance, Haiku call latency (~1s)
- Complexity: Medium-High

### Shared Long-Running Container
One container for entire campaign, worktree swapped per experiment.
- Gains: No per-experiment startup cost, deps persist
- Costs: Deps leak between experiments (breaks isolation), state accumulates
- Complexity: Medium

### Host-Based Restricted Executor (no container)
MCP tool runs subprocess on host with allowlist + uv venv per worktree.
- Gains: Simplest, no Docker dependency
- Costs: No filesystem isolation, no resource limits, dead-end for ECS
- Complexity: Low

## Chosen Approach

**Per-Experiment Container + MCP Tool Proxy + Intelligent Compression** — best fit for constraints. Clean isolation per experiment, MCP proxy gives fine-grained permission + token control, same Dockerfile works locally and on ECS. The ~1-2s container startup is negligible vs 30-600s researcher runtime.

## Architecture (Three-Layer)

### Layer 1: Container Sandbox (ENG-2651)
- Base image: ghcr.io/astral-sh/uv:python3.13-bookworm-slim
- Per-experiment lifecycle: start after create_worktree, stop in finally block
- Worktree bind-mounted at /workspace
- MCP tool: run_in_container(command, timeout, compress) with command allowlist (python, uv, pytest, pip)
- docker exec for command execution — no shell=True, no raw TTY

### Layer 2: Haiku Edge Compressor (future issue)
- Haiku micro-agent at the MCP tool boundary
- Structured extraction (not summarization) for long outputs (>20 lines)
- Raw passthrough for short outputs
- Confidence scoring (0-1.0) on every compressed output
- Researcher system prompt rule: if confidence < 0.7, re-request with compress=False
- Hybrid confidence: self-score on every call, adversarial spot-check on sample

### Layer 3: Opus Filter Designer (future issue)
- Plugin architecture: registry of per-tool extraction programs
- Opus outloop agent observes raw vs compressed vs researcher behavior
- Designs specialized parsers (pytest regex, pip line filter, traceback extractor)
- Hybrid filters: symbolic parsing first, LLM fallback for novel formats
- Uses adversarial confidence scores as training signal
- Ships filters as Python functions in the plugin registry

## Key Context Discovered During Shaping

- OpenHands uses nikolaik/python-nodejs base; SWE-agent uses python:3.11-bullseye; uv recommends ghcr.io/astral-sh/uv:python3.13-bookworm-slim
- Per-session container is industry standard (OpenHands, SWE-agent, Devin, Modal, E2B)
- MCP tool proxy is established pattern (Anthropic engineering blog, OpenHands ActionExecutionServer)
- Shell injection accounts for 43% of MCP CVEs — command allowlist is mandatory
- Bind mount worktree into container (researcher modifies files we need back on host)
- Exclude .venv via anonymous volume to avoid platform corruption
- Docker containers share host kernel — adequate for trusted local dev, not for untrusted production
- Fargate does not support warm pools — EKS + Kubernetes Agent Sandbox for sub-second cold starts
- Same image runs locally and in production via SANDBOX_MODE env var switch

## Next Step

Plan → /create_plan for Layer 1 (container sandbox + MCP tool). Layers 2-3 as separate Linear issues.
