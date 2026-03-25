# Idea Brief: Meta Skills — `/learn` and `/extend`

**Date:** 2026-03-02
**Status:** Shaped → Planning

## Problem

When Claude encounters a recurring workflow or the user says "remember how to do X", there's no idiomatic way to persist that as a reusable capability. The only option is MEMORY.md (flat text, no dispatch semantics) or the user manually authoring SKILL.md/agent/hook files. The extension surface (.claude/skills, agents, hooks, rules) is well-structured but has no on-ramp for growing organically.

## Constraints

- Skills budget capped at 2% of context window — can't create dozens of bulky skills
- Each artifact type has a distinct file format (SKILL.md YAML frontmatter, agent .md frontmatter, hooks JSON in settings.json, rules .md with globs)
- Our existing convention: skills = static reference (~65 lines), commands = procedural orchestration, agents = tool-scoped workers
- Artifacts in `.claude/` are git-tracked — must be committable and reviewable
- Agent Skills Open Standard exists (cross-platform SKILL.md spec) — should align with it
- Anthropic's official `skill-creator` meta-skill exists but only creates skills, not hooks/agents/rules

## Options Considered

### Single `/extend` Command
One command that classifies the need (skill vs hook vs agent vs rule), interviews, proposes, and writes. Handles both lightweight "learn this" and heavier "build this extension" flows.
- Gains: Single entry point, Claude handles classification
- Costs: One command covering all artifact types gets long. Lightweight "learn" flow carries overhead of heavier "build" interview steps
- Complexity: Medium

### Per-Type Commands (`/create-skill`, `/create-agent`, `/create-hook`, `/create-rule`)
Separate focused commands for each artifact type, following `create-X` naming convention.
- Gains: Each command focused and short. Direct access when you know the type.
- Costs: 4 commands to maintain. Doesn't handle "not sure what type" case. Goes from 2 `create-X` commands to 6.
- Complexity: Low per-command, medium total

### Two Flows + Shared Reference Skill (chosen)
`/learn` for lightweight reflection (recurring pattern → skill/rule). `/extend` for constructive authoring (new capability → hook/agent/skill). One `extension-authoring` skill (`user-invocable: false`) carries schemas, templates, and project conventions for all artifact types.
- Gains: Matches the two distinct workflows (reflective vs constructive). `/learn` stays lightweight (~2 min). `/extend` gets full interview/design treatment. Shared skill prevents duplication of schema knowledge.
- Costs: 2 commands + 1 skill to author. Boundary between "learn" and "extend" could occasionally be fuzzy.
- Complexity: Medium

## Chosen Approach

**Two Flows + Shared Reference Skill** — separate `/learn` (reflective, quick) and `/extend` (constructive, thorough) commands, both loading an `extension-authoring` skill that carries artifact schemas and project conventions.

Rationale: The research showed that merging lightweight reflection with heavyweight construction kills the simplicity that makes reflection actually get used (cf. `claude-meta`'s success through simplicity). Meanwhile, constructive authoring needs the full interview/propose/confirm flow (cf. Skill Factory's multi-artifact coverage). Two entry points, one shared knowledge base.

Scope: V1 covers skills, agents, rules, and hooks. Commands are excluded from auto-generation — they're too complex (200-400 lines of orchestration prose) and rare enough to author manually.

## Key Context Discovered During Shaping

- Anthropic has an official `skill-creator` in `anthropics/skills` repo — 5-stage loop with eval framework, but skills-only
- `alirezarezvani/claude-code-skill-factory` is the only multi-artifact generator found — uses interactive Q&A, covers skills + agents + hooks + commands
- `aviadr1/claude-meta` achieves compounding improvement through simple reflection → write rule loop — 10.87% accuracy improvement reported by Arize
- `rebelytics/one-skill-to-rule-them-all` (Task Observer) uses passive observation → propose-only pattern
- Agent Skills Open Standard (agentskills.io) is cross-platform — validation via `skills-ref` Python library
- Our existing skills are tight: 3 skills, all `user-invocable: false`, ~50-65 lines, minimal frontmatter (`name`, `description`, `user-invocable`), static reference content
- Our existing agents: 9 definitions with `name`, `description`, `tools`, optional `skills`, `model`, `maxTurns`
- Our existing hooks: 4 event types (PreToolUse, PostToolUse, Stop, TaskCompleted) with Python scripts + inline agents
- Description quality is critical — descriptions are the routing mechanism for skill selection (pure LLM reasoning, not algorithmic)

## Next Step

- [Plan] → `/create_plan memory-bank/thoughts/shared/briefs/2026-03-02-meta-skills-learn-extend.md`
