# Idea Brief: Agent Scope Enforcement — Defense-in-Depth

**Date:** 2026-02-26
**Status:** Shaped → Planning

## Problem
CLAUDE.md scope discipline instructions are soft guardrails — Cyrus (Linear's AI agent) ignored them and bundled 3 issues into 1 PR (ENG-2176 incident). No deterministic enforcement exists for file-scope boundaries. PreToolUse hooks are bypassable (agent can edit hook files), and settings.json deny rules have known bugs. A single enforcement point is insufficient.

## Constraints
- Cyrus runs Claude Agent SDK with `settingSources: ["project"]` — reads CLAUDE.md, fires hooks, loads agents
- Cyrus has NO path-level scope enforcement natively — only tool-type restrictions
- PreToolUse hooks are bypassable (Claude Code Issue #11226, closed "not planned")
- `settings.json` deny rules have bugs (#6699, #27040)
- `cyrus-setup.sh` runs before each session with `LINEAR_ISSUE_ID` env var (unused lever)
- Plan files already list files per phase with parseable `**File**: \`path\`` pattern
- Plan files committed to main before implementation — can be used as tamper-proof source of truth

## Options Considered

### Scope File + PreToolUse Hook (write-time only)
Immediate feedback when agent writes out-of-scope. Hook reads `.claude/scope.json` and blocks.
- Gains: Fastest feedback loop, agent self-corrects immediately
- Costs: Bypassable (agent can edit hook or scope file), needs Bash detection too
- Complexity: Low

### CI Scope Gate (merge-time only)
GitHub Action compares PR changed files against plan scope. Required status check.
- Gains: Hard merge gate, tamper-proof (server-side), catches everything
- Costs: Late enforcement (after work is done), agent wastes cycles on out-of-scope changes
- Complexity: Low

### Session-Startup Scoping via cyrus-setup.sh (pre-session only)
Script generates scope.json from plan file before agent starts working.
- Gains: Automatic per-issue, no human action needed
- Costs: Requires Linear API/plan file lookup, falls back to no enforcement without plan
- Complexity: Medium

### Defense-in-Depth Combo (all three + CODEOWNERS)
Three enforcement layers at different points: write-time → merge-time → session-start. Plus CODEOWNERS for shared components.
- Gains: No single point of failure. Each layer independently valuable. Matches industry consensus.
- Costs: Most upfront work (~1-1.5 days). More infrastructure to maintain.
- Complexity: Medium

## Chosen Approach
**Defense-in-Depth Combo** — The ENG-2176 incident proved a single enforcement point fails. Industry consensus is 2-3 independent enforcement layers. Each layer (hook, CI, setup script) is independently valuable and catches different failure modes. CODEOWNERS adds human review for the highest-risk shared components.

## Key Context Discovered During Shaping
- PreToolUse stdin JSON contains `tool_input.file_path` for Edit/Write — same format as PostToolUse
- Plan file regex for extracting files: `\*\*File\*\*:\s+\`([^`]+)\``
- Adding a job to `lint.yml` automatically triggers `agent-ci-feedback.yml` (Cyrus self-correction) and `ci-failure-linear.yml`
- No CODEOWNERS file exists in the repo
- `cyrus-setup.sh` doesn't exist yet — free lever for per-session setup

## Next Step
- Plan → `memory-bank/thoughts/shared/plans/2026-02-26-agent-scope-enforcement.md`
