---
date: 2026-03-04T12:00:00-05:00
researcher: Claude
git_commit: bf8dd50f2a584ab7f07d4bd73446e9a7fef16be4
branch: main
repository: filescience
topic: "Validate CI fix loop /create-pr skill — feasibility, risks, alternatives"
tags: [deep-research, ci, skills, claude-code, automation]
status: complete
research_depth: standard
iterations_completed: 3
last_updated: 2026-03-04
last_updated_by: Claude
---

# Deep Research: CI Fix Loop — /create-pr Skill Validation

## Research Question
Validate the idea in the CI Fix Loop brief — assess feasibility of a pure-skill /create-pr that creates a PR, watches CI via background Bash, and autonomously fixes failures in a bounded loop. Identify risks, explore alternatives, and validate the approach.

## Summary
The pure-skill approach is **feasible but must abandon `run_in_background`** as its core CI-watching primitive due to known bugs (notification regression #21048, infinite token drain #11716). The validated alternative is a **foreground polling loop** using `gh run list --commit <SHA> --json status,conclusion` with an extended Bash timeout — this costs zero tokens while waiting and avoids the 13K-line context overflow from `gh run watch`. The skill design should follow the existing helm skill pattern (multi-step conditional loop, 50-65 lines) with a hard 3-round fix cap matching both the cloud-side `agent-ci-feedback.yml` and industry consensus. Key risk: autonomous fix loops produce 75% more logic errors than human fixes — human checkpoint after each fix round is strongly recommended over fully autonomous operation.

## Perspectives Explored
1. **Existing CI Infrastructure** — The cloud-side system provides a proven design reference with sophisticated guardrails (SHA dedup, round tracking, escalation)
2. **Claude Code Primitives** — `run_in_background` is unreliable; foreground polling is the safe path; `gh run list --commit` is the right primitive
3. **Skill Authoring Feasibility** — The fix loop would be the most complex skill in the repo but stays within established patterns
4. **Existing Patterns & Prior Art** — pr_utils.py and external blog posts validate the polling approach
5. **Risk & Failure Modes** — Context degradation, fix oscillation, and wrong-fix-passes-CI are well-documented industry risks

## Detailed Findings

### Existing CI Infrastructure
The cloud-side `agent-ci-feedback.yml` (`.github/workflows/agent-ci-feedback.yml`) provides a mature reference implementation. It triggers on `workflow_run` completion for Test, Lint, and Semantic PR workflows (failure conclusions only). Key mechanisms:

- **SHA dedup**: Dual-track system ("code" track for Test/Lint, "semantic-pr" track for Semantic PR). Uses HTML comment markers (`<!-- agent-ci-feedback:{sha}:{track} -->`) in Linear comments to prevent duplicate feedback per commit per track.
- **Round counting**: Counts distinct committed SHAs that previously received non-escalation feedback. Hard cap at `MAX_ROUNDS = 3`.
- **Active session guard**: Detects open agent sessions via `lastSessionStart`/`lastSessionEnd` timestamps in Linear comment patterns — avoids sending feedback while the agent is already working.
- **Log parsing**: Extracts lines matching `##[error]`, `FAILED`, `Failure.`, `Missing lines \d`, appends 30-line log tail, truncates to 3000 characters. Builds job-specific remediation text distinguishing title-only, code-only, and combined failures.
- **Escalation**: After 3 rounds, switches from agent mention to human assignee mention.

**Relevance to local skill:** The log parsing patterns (error extraction + truncation) and 3-round cap are directly reusable. The SHA dedup and Linear integration are cloud-specific and not needed locally. The local skill inverts the push/pull model — the cloud system pushes feedback via Linear comments, while the local skill pulls via `gh` CLI.

### Claude Code Primitives

**`run_in_background` — UNRELIABLE, do not use as core primitive:**
- Notification regression in v2.1.19+ means completion notifications intermittently don't fire (anthropics/claude-code#21048)
- `KillShell` doesn't clear reminder state, causing infinite system-reminders that exhaust tokens at 11x normal cost — $10-18/session vs expected $0.90 (anthropics/claude-code#11716)
- Output retrieved via TaskOutput tool (poll model), no hard timeout for background tasks

**`gh run watch` — NOT recommended:**
- Takes a single run ID only (not all workflows per commit)
- Requires `--exit-status` flag for non-zero exit on failure
- Produces 13K+ lines of output that overflow context
- Works correctly locally (bug #8194 only affects hosted runners)

**`gh run view --log-failed` — USE WITH CAUTION:**
- Known bugs: silent empty output on some runner versions (#9153, #10551)
- Large logs can crash the command entirely
- Should be wrapped with a fallback (e.g., `gh run view --log` + grep for ERROR patterns)

**`gh run list --commit <SHA>` — RECOMMENDED polling primitive:**
- `gh run list --commit <SHA> --json databaseId,name,status,conclusion` finds all runs triggered by a specific push
- Each workflow gets its own run ID — one push can produce N runs queryable together
- Runs appear within seconds but no guaranteed SLA — short polling loop recommended

**Foreground Bash with extended timeout — RECOMMENDED waiting strategy:**
- Bash tool supports configurable timeout up to 600,000ms (10 minutes)
- Agent consumes ZERO inference tokens while waiting for a foreground Bash command
- Best used with a compact shell polling loop, not `gh run watch`

**Recommended pattern:**
```
# In foreground Bash with ~5min timeout:
SHA=$(git rev-parse HEAD)
while true; do
  RUNS=$(gh run list --commit $SHA --json status,conclusion,name)
  # Check if all runs completed
  # If all done, exit with pass/fail
  sleep 30
done
```

### Skill Authoring Feasibility

Current skill conventions:
- YAML frontmatter + Markdown body, 50-65 line target
- Only the helm skill has multi-step conditional logic (numbered sequence with if/branching and confirmation gates)
- No existing skill uses `run_in_background` or `gh` CLI commands
- plan-implementer subagent pattern exists for delegating multi-file changes

The CI fix loop skill would be the most complex skill in the repo, but it follows the established helm skill pattern. Architecture recommendation:

**Single skill with explicit phases** (not composable sub-skills):
1. Create PR phase: `gh pr create`, capture PR URL/number
2. Watch phase: foreground polling loop with extended timeout
3. Fix phase: parse failure logs, attempt fix, commit, push
4. Loop control: round counter, hard cap at 3, human checkpoint option

This fits within 50-65 lines if the fix-phase instructions are kept directive rather than prescriptive (i.e., "analyze the failure and fix it" rather than enumerating every possible failure type).

### Existing Patterns & Prior Art

**pr_utils.py** (`tools/helm/src/helm/pr_utils.py:36-84`): Uses `gh pr list --json number,headRefName,state,statusCheckRollup --limit 100`. Reduces `statusCheckRollup` to a single `checks_passing` boolean (all conclusions == "SUCCESS", empty array = False). This is a coarser signal than `gh run list` — it tells you pass/fail but not which workflow failed or why.

**Chris Dzombak blog** ("Ask Claude Code to fix CI"): Documents a real-world pattern of using Claude Code with polling to fix CI failures. Validates the general approach.

**Industry tools** (Devin, SWE-agent, Copilot Workspace): All implement bounded autonomy with human-in-the-loop review gates rather than unbounded self-healing loops. Devin specifically documents "closing the agent loop" for CI review comments.

### Risk & Failure Modes

1. **Context degradation**: Agents lose original intent as conversation history fills with failed CI output and fix attempts. Mitigation: keep CI output minimal (truncated error lines only, not full logs), use the 3000-char truncation pattern from agent-ci-feedback.yml.

2. **Fix oscillation**: Contradictory solutions cycle without convergence, especially with coupled test side effects. Mitigation: hard 3-round cap, require each fix to be a net improvement (fewer failures than before).

3. **Wrong fixes that pass CI**: AI PRs produce 75% more logic and correctness errors (194 per 100 PRs per CodeRabbit study). Subtle regressions — performance, control flow, hallucinated dependencies — routinely evade CI. Mitigation: human checkpoint after each fix round, not just at the end.

4. **Token cost**: Each fix round involves reading failure logs + analyzing code + writing fix + committing + pushing + waiting. At ~$0.10-0.30 per round, a 3-round loop costs $0.30-0.90 in addition to the initial PR creation.

5. **Blast radius**: An autonomous push to a PR branch is relatively safe (not pushing to main), but wrong fixes pollute git history and may confuse reviewers. Mitigation: squash merge at the PR level.

## Key Sources

### Codebase files
- `.github/workflows/agent-ci-feedback.yml` — cloud-side CI feedback system (round tracking, log parsing, escalation)
- `tools/helm/src/helm/pr_utils.py` — statusCheckRollup-based CI status checking
- `.claude/skills/helm/SKILL.md` — most complex existing skill (multi-step conditional loop)
- `.claude/agents/plan-implementer.md` — subagent delegation pattern
- `.claude/skills/extension-authoring/SKILL.md` — skill authoring conventions

### External references
- [gh run list manual](https://cli.github.com/manual/gh_run_list) — --commit flag, JSON output
- [gh run watch manual](https://cli.github.com/manual/gh_run_watch) — single-run-ID limitation
- [anthropics/claude-code#21048](https://github.com/anthropics/claude-code/issues/21048) — run_in_background notification regression
- [anthropics/claude-code#11716](https://github.com/anthropics/claude-code/issues/11716) — infinite system-reminder token drain
- [cli/cli#9153](https://github.com/cli/cli/issues/9153) — --log-failed silent empty output
- [Ask Claude Code to fix CI — Chris Dzombak](https://www.dzombak.com/blog/2025/10/ask-claude-code-to-fix-ci/)
- [AI vs Human Code Gen Report — CodeRabbit](https://www.coderabbit.ai/blog/state-of-ai-vs-human-code-generation-report)
- [Tips to Avoid AI Fix Loops — byldd.com](https://byldd.com/tips-to-avoid-ai-fix-loop/)

## Open Questions
1. **gh run view --log-failed reliability**: Should the skill have a fallback for when --log-failed returns empty? (e.g., `gh api` to fetch logs directly, or `gh run view --log` + grep)
2. **Human checkpoint granularity**: Should the skill pause after EVERY fix round for human review, or only after the final round? The brief's "chosen approach" implies fully autonomous, but research suggests per-round checkpoints are safer.
3. **Semantic PR failures**: The cloud system treats semantic PR failures differently (title-only fixes). Should the local skill handle this case explicitly, or just treat all failures uniformly?
4. **Polling interval tuning**: 30-second polling is a starting point. CI runs in this repo typically take 2-5 minutes — what's the optimal interval to balance responsiveness vs. API rate limits?

## Research Metadata
- Depth mode: standard
- Iterations completed: 3 / 5
- Termination reason: gaps exhausted
- Manifest: `.claude/deep-research/2026-03-04-ci-fix-loop-create-pr.md`
