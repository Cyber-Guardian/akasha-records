# Idea Brief: Resolve Linear Issue — Tech Lead Orchestrator Pattern

**Date:** 2026-02-25
**Status:** Shaped → Planning

## Problem
The `/resolve-linear-issue` workflow has a fragile middle: understanding and closeout work well, but the plan → implement bridge is hand-wavy. For non-trivial issues, planning gets skipped or done sloppily. Context wipes between planning and implementation lose Linear state (issue ID, status, ACs), so the card never moves to "In Progress." The three skills involved (`resolve` → `create_plan` → `implement_plan`) don't share state and weren't designed to chain together.

## Constraints
- Skills are markdown prompts, not code — behavior is influenced by instruction, not enforced programmatically
- Subagent nesting is a hard platform constraint — subagents cannot spawn other subagents (Task tool not provisioned)
- Subagents share the parent's 200K token budget with 8192 output token limit per subagent
- Planning is inherently interactive (needs user conversation) — cannot be delegated to a subagent
- Shape is also inherently interactive — stays in main context
- Plan files in `memory-bank/thoughts/shared/plans/` are the established handoff medium
- Custom agents defined in `.claude/agents/` with frontmatter (model, tools, skills) — existing pattern via `adversarial-plan-reviewer.md`

## Options Considered

### Approach 1: "State File Only"
Persist Linear metadata in the plan file header. Each skill reads/writes it. No subagents — keep the existing wipe-and-resume workflow.
- Gains: Simple, low risk, works with current workflow
- Costs: Still burns context on implementation, still requires manual context wipes, Linear state survival depends on skills actually reading the metadata
- Complexity: Low

### Approach 2: "Pure Subagent Dispatcher"
Main agent is razor-thin glue — delegates everything (planning, implementation) to subagents.
- Gains: Maximum context savings
- Costs: Planning becomes non-interactive (worse plans), can't shape in a subagent, user talks through parent as relay (awkward)
- Complexity: Medium

### Approach 3: "Tech Lead Orchestrator" (Hybrid)
Main agent owns understanding + planning + closeout (interactive work). Implementation phases dispatched to a dedicated Opus subagent. Plan file carries Linear metadata as safety net.
- Gains: Context stays clean, Linear state survives naturally, no more context wipes for normal issues, interactive planning preserved, failure-return pattern for stuck implementers
- Costs: Implementation subagent can't nest explorers (mitigated by plan quality + parent pre-briefing + raw tools), new agent definition to maintain, subagent prompt engineering needed per phase
- Complexity: Medium

## Chosen Approach
**Tech Lead Orchestrator** — the main agent is the "tech lead" that thinks, plans, and reviews. A dedicated `plan-implementer` subagent (Opus) executes phases. The plan file is the handoff medium and carries Linear metadata.

## Key Context Discovered During Shaping
- The orchestrator-only pattern is the dominant architecture across the ecosystem (Anthropic's own multi-agent system, Open SWE, spec-driven development pattern)
- alexop.dev's spec-driven pattern achieved 71% context after 14 implementation tasks with this approach
- Cognition counter-argument ("Don't Build Multi-Agents") applies to decision-history-dependent work, not well-scoped implementation phases
- `skills:` frontmatter injects domain knowledge into subagents — mechanism for Polylith conventions, ty patterns
- Sequential subagent dispatch is safer than parallel for interdependent phases
- Parent can pre-brief each phase with targeted exploration results to compensate for no nested explore agents
- Failure-return pattern: if subagent hits unexpected state, it returns to parent with findings rather than guessing

## Research Sources
- Anthropic multi-agent research system: Opus lead + Sonnet subagents, external memory handoff
- alexop.dev spec-driven development: main agent writes spec, subagents implement tasks, atomic commits
- Open SWE (LangChain): Manager → Planner → Programmer → Reviewer pipeline
- Cognition "Don't Build Multi-Agents": counter-argument for single-agent with compression
- Claude Code docs: subagent nesting is hard wall, skills injection via frontmatter

## Next Step
- [Plan] → `/create_plan memory-bank/thoughts/shared/briefs/2026-02-25-resolve-linear-issue-orchestrator.md`
