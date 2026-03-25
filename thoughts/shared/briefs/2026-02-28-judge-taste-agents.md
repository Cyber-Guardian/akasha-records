# Idea Brief: Judge / Taste Agents

**Date:** 2026-02-28
**Status:** Seed (not yet shaped)

## Raw Thinking
- Should make multiple autonomous agents that build taste and opinion
- These are used as judges for the general worker agents
- Separate from worker agents — they evaluate, they don't build

## Core Idea
Create a class of autonomous agents whose job is NOT to write code, but to develop and enforce quality standards. These "judge agents" build taste over time — they review worker agent output, assess architectural fit, evaluate code quality beyond what linters catch, and flag drift from product principles. They act as a compounding quality layer: the more they review, the sharper their taste gets.

Think of it as: worker agents are junior engineers, judge agents are senior engineers / tech leads who do code review and maintain the bar.

## Possible Judge Roles
- **Architecture judge**: Does this change respect Polylith boundaries, component dependency rules, and established patterns?
- **Product judge**: Does this feature align with product principles (simple > complex, few-click, fast UX)?
- **Quality judge**: Beyond linting — is this code idiomatic, maintainable, does it follow existing conventions?
- **Cost judge**: Is this approach cost-efficient? Could we achieve the same with fewer resources/tokens/API calls?

## Relationship to Existing Work
- **Deterministic Guardrail Layer** — the deterministic layer catches structural violations; judge agents catch subjective/taste violations that rules can't express
- **Agent Execution Discipline** — judge agents could replace/augment the manual review step after agent PRs
- **Agent Helm Orchestration** — the helm could invoke judge agents as a gate before merge

## Open Questions
- How do judge agents "build taste"? Persistent memory? Fine-tuning? Curated examples?
- Are these synchronous (block merge until approved) or async (flag issues for human review)?
- How do you prevent judge agents from becoming a bottleneck or going stale?
- What's the minimum viable judge? One agent reviewing PRs against CLAUDE.md + product principles?
