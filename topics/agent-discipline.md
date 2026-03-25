---
topic: Agent Discipline
status: active
touched: 2026-03-05
related:
  - thoughts/shared/briefs/2026-02-25-deterministic-guardrail-layer.md
  - thoughts/shared/plans/2026-03-05-agent-testing-pyramid.md
  - thoughts/shared/research/2026-03-05-triage-agent-deployment-failures.md
---

# Agent Discipline

## Current State
Rules documented in `.claude/reference/agent-discipline.md` and enforced by CI + hooks:
- Issues labeled `autonomous` or `co-dev` — 6 qualification criteria for autonomous
- Scope enforcement: only modify files listed in issue; discover work -> Linear comment, don't do it
- 3+ file changes: delegate to `plan-implementer` subagent via Task tool
- Commit format: `type(scope): phase N — description\n\nENG-XXXX`
- Agents read `identity.md` only — no writes to `memory-bank/durable/`
- PRs must use PR template and reference Linear issue in title
- `semantic-pr.yml` enforces conventional PR title and blocks durable/ modifications from agent PRs

## Key Decisions
- Agent PRs cannot modify `memory-bank/durable/` (CI-enforced)
- Plan-first approach: agents must have an approved plan before implementation
- Scope enforcement via `scope-check` CI job validates file changes against plan

## Testing Discipline (tools/ workspace)
- Triage agent rewrite hit 4 runtime failures in production — all integration wiring, all from "lasagna mocking"
- Plan created: [[2026-03-05-agent-testing-pyramid|Agent Testing Pyramid]] — 5 phases
- Key pattern: construction tests (build real agents with TestModel, no LLM calls) catch wiring bugs structurally
- PydanticAI-native: `ALLOW_MODEL_REQUESTS=False` + `TestModel` + `FunctionModel` + `Agent.override()`
- CI workflow planned for `tools/**` path-filtered lint + test + diff-coverage

## Open Questions
- Deterministic guardrail layer (brief shaped, not yet planned) — additional enforcement beyond CI
- Topic note write policy for agents: policy-only in agent-discipline.md, no CI enforcement

## Artifacts
- `.claude/reference/agent-discipline.md` — full rules
- `.github/workflows/semantic-pr.yml` — PR enforcement
- `.claude/rules/tools-testing.md` — testing conventions for tools/ (Phase 5 of testing pyramid plan)
