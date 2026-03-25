# Idea Brief: Agent Scope Creep Prevention

**Date:** 2026-02-26
**Status:** Shaped → Implementing

## Problem
Cyrus (Linear's AI agent, running Claude Agent SDK) bundled 3 Linear issues (ENG-2174, ENG-2176, ENG-2178) into one PR when only assigned ENG-2176. Our CLAUDE.md has agent execution discipline but no explicit scope constraint telling agents to stay within their assigned issue. Cyrus reads CLAUDE.md and respects hooks — it just wasn't told to stay scoped.

## Constraints
- Cyrus runs Claude Agent SDK with `settingSources: ["user", "project", "local"]` — reads CLAUDE.md, loads `.claude/agents/*.md`, fires `.claude/settings.json` hooks
- All tools enabled including Task — Cyrus CAN use plan-implementer subagent
- Cyrus has its own prompt template from Linear (issue context injected via template vars) which may compete with CLAUDE.md instructions
- `appendInstruction` in Cyrus repo config is an additional lever for per-repo instructions
- Agent teams enabled (`CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS: "1"`)

## Options Considered

### A. CLAUDE.md scope constraint (soft guardrail)
Add explicit "stay within your assigned issue scope" to agent discipline section.
- Gains: Zero infrastructure, immediate, works for any CLAUDE.md-reading agent
- Costs: Soft — agent may still drift
- Complexity: Low

### B. PostToolUse scope-checking hook (hard guardrail)
Hook on Write/Edit that checks if modified file is "in scope" per a manifest or issue metadata.
- Gains: Hard enforcement regardless of agent behavior
- Costs: Scope definition is hard (manifest per branch? dynamic inference?), false positives on legitimate deps
- Complexity: Medium-High

### C. Agent dispatch via plan-implementer (structural guardrail)
CLAUDE.md instructions telling Cyrus to delegate to plan-implementer (which already has scope constraints).
- Gains: Leverages existing infrastructure, plan file = single source of truth for scope
- Costs: Chicken-and-egg (plan must exist before work), Cyrus's own prompt may fight with dispatch instructions
- Complexity: Medium

### D. Layered (A + C)
CLAUDE.md scope constraint as baseline, plus plan-implementer dispatch for multi-file changes.
- Gains: Soft guardrail for simple fixes, structural enforcement for complex work
- Costs: More CLAUDE.md text, threshold heuristic not a hard gate
- Complexity: Low-Medium

## Chosen Approach
**D (Layered)** — CLAUDE.md scope constraint + plan-implementer dispatch. A alone is too soft (Cyrus ignored explicit file lists in the issue description). B is over-engineered. C has chicken-and-egg. D layers immediate soft guardrail with structural enforcement using existing infrastructure.

## Key Context Discovered During Shaping
- Cyrus is open-source (`ceedaragents/cyrus`), runs Claude Agent SDK's `query()` with `settingSources: ["project"]` — confirmed CLAUDE.md + hooks + agents loaded
- `allowedTools` includes Task — Cyrus can spawn plan-implementer
- `CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS: "1"` enabled — agent teams available
- `.claude/agents/plan-implementer.md` already has "DO NOT modify files outside your phase's scope" (line 79)
- ENG-2176 incident: Cyrus saw parent issue's sibling sub-issues and tried to complete them all
- `appendInstruction` in Cyrus repo config can supplement CLAUDE.md

## Next Step
- Implementing directly — CLAUDE.md agent discipline section additions
