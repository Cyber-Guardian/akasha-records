---
type: event
created: 2026-03-27
status: active
touched: 2026-03-27
tags: [claude-code, plugins, team-enablement, developer-experience, marketplace]
---
# Idea Brief: Claude Code Plugin Marketplace — Team-Wide AI Operating System

**Date:** 2026-03-27
**Status:** Shaped → Planning

## Problem
We've built a comprehensive Claude Code operating system in the filescience monorepo: a CLAUDE.md routing table, shape→plan→implement lifecycle, skill router hook, Scout knowledge layer, assembly-line work system, adversarial review agents, quality hooks, and session discipline. But it all lives in `.claude/` inside filescience. The rest of the engineering team (Niklas on dashboard, Harsh on landing/studio, Sungmin/Abhinav on RPA) gets none of it. They're using vanilla Claude Code or not using it at all.

Eng department meeting next week (week of 2026-03-31) is the moment to change that.

## Constraints
- Claude Code has an official **Plugin + Marketplace** system (shipped, v1.0.33+) — this is the right distribution mechanism
- A plugin bundles commands, skills, agents, hooks, MCP servers, and settings into one installable unit
- Marketplace = a private GitHub repo with `marketplace.json` listing plugins
- Team installs via `/plugin marketplace add Cyber-Guardian/claude-plugins` + `/plugin install`
- Skills get namespaced: `/cgcg-dev-system:shape` (not `/shape`)
- Other repos are Python and TypeScript/HTML — need stack-aware rules
- Must include the full methodology: routing table, shape→plan→implement, Scout knowledge layer, Work MCP, deep agents — not just lint rules
- MCP servers can bundle inside plugins via `.mcp.json` + `${CLAUDE_PLUGIN_ROOT}` for paths
- Scout MCP can run in local-vault mode (`--vault-path`) — no S3/GitHub API needed per repo
- Work MCP is pure local filesystem — zero external deps

## Options Considered

### Full Plugin Marketplace with Scout + Work MCP
Create `Cyber-Guardian/claude-plugins` repo. Ship one plugin (`cgcg-dev-system`) that bundles the complete operating system: routing table, all methodology skills, adversarial agents, quality hooks, Scout in local-vault mode, Work MCP. Each repo gets its own local knowledge vault and `.work/plans/` directory.
- Gains: Full operating system. Every repo works the same way. Knowledge stays local per repo. Typed plan contracts via Work MCP. Semantic search via Scout.
- Costs: ~1-2 days. Strip filescience refs from ~15 artifacts. Parameterize Scout vault path and Work workspace root. Write plugin.json manifest.
- Complexity: Medium

### Lite Plugin (no MCP servers)
Same marketplace, but skills fall back to local file writes. No Scout search, no typed contracts.
- Gains: Ships in half a day.
- Costs: Loses semantic search and typed plan contracts — the two highest-value primitives.
- Complexity: Low

### Staged (Lite Now, Full Later)
Ship Lite for the meeting, add MCP servers in follow-up.
- Gains: Something in hands by meeting.
- Costs: Two install rounds. First impression is degraded version.
- Complexity: Low → Medium

## Chosen Approach
**Full Plugin Marketplace with Scout + Work MCP** — the meeting story is stronger ("install one plugin, get the whole system"), and the MCP servers are straightforward to bundle (Work is pure local, Scout just needs `--vault-path` instead of `--s3-bucket`).

## Key Context Discovered During Shaping

### Artifact portability audit
- **Portable (zero changes):** 6 rules, 2 agents — safe-git-staging, recurring-errors, dogfood-iterate, no-backwards-compat, tdd-first, agent-authoring, codebase-locator, web-search-researcher
- **Partially portable (strip filescience deps):** ~15 skills/agents/rules/hooks — alignment, ground, adversarial-code-reviewer, ruff, ty, model-based-testing, etc.
- **Repo-specific (stays in filescience):** polylith boundaries, import-linter, terraform, helm, lab-retrospective, debug-investigator

### MCP server portability
- **Work MCP** (`tools/work-mcp/`): Pure local. Reads/writes `.work/plans/`. `WORK_WORKSPACE_ROOT` env var (defaults to CWD). Depends on `work-protocol` library at `../lib`. Zero external services.
- **Scout MCP** (`tools/scout/`): Can run in local-vault mode (`--vault-path ./knowledge/`). Needs `lancedb`, `model2vec`, `tantivy` for search. GitHub write ops (`git_ops.py`) hardcode `DEFAULT_REPO = "Cyber-Guardian/akasha-records"` — needs parameterization. `rg` (ripgrep) runtime dependency.

### Skill MCP dependencies
- 8 of 10 core skills work with zero Scout/Work MCP
- Only `create_plan` and `implement_plan` are deeply coupled to Work MCP (create_plan, add_phase, add_contract, validate_plan, get_phase, freeze_plan, run_contracts, complete_task)
- Scout is always indirect — via optional `knowledge-explorer` subagent dispatch, never direct tool calls in skill files

### Plugin system details (from official docs)
- `plugin.json` manifest in `.claude-plugin/` directory
- `${CLAUDE_PLUGIN_ROOT}` variable for hook/MCP paths within the plugin's cached install directory
- `${CLAUDE_PLUGIN_DATA}` for persistent data that survives plugin updates
- `hooks/hooks.json` — same format as settings.json hooks section
- `/plugin validate .` for pre-publish validation
- `extraKnownMarketplaces` + `enabledPlugins` in repo's `.claude/settings.json` for auto-prompting
- Private repos work via existing `gh auth` credentials
- `GITHUB_TOKEN` env var needed for background auto-updates

## Next Step
- [Plan] → `/create_plan thoughts/shared/briefs/2026-03-27-claude-code-plugin-marketplace.md`
