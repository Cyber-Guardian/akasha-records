---
type: event
created: 2026-03-13
status: active
---
# Idea Brief: /retrospective Skill

**Date:** 2026-03-13
**Status:** Shaped → Planning

## Problem
Session friction patterns (repeated errors, human corrections, approach pivots) evaporate when a session ends. The memory system has five tiers designed for different durability, but the upward flow from session context to durable extensions is entirely manual. No structured step exists to extract and persist learnings.

## Constraints
- Manual command invocation (not automatic/hook-based)
- Can take up to ~20 minutes for thorough analysis
- Output goes to `.claude/` extensions + memory-bank (not just hidden auto-memory)
- Propose-confirm before writing anything — never write without approval
- Must check for overlap with existing artifacts before proposing new ones

## Options Considered

### Flat Extract (Claudeception-style)
Reflect → extract → write all as `.claude/skills/` or `.claude/rules/`. One output tier.
- Gains: Simple, proven by prior art (Claudeception, AccidentalRebel's session-retrospective)
- Costs: Misses memory-bank tiers (topics, decisions). Learnings that aren't rules/skills have nowhere to go.
- Complexity: Low

### Tiered Extract with Heuristic Router
Reflect → extract → classify by type → auto-route to the right tier → propose batch.
- Gains: Uses full memory architecture. Each learning lands where it'll be found next time.
- Costs: Routing adds complexity. Risk of over-deliberation on tier placement.
- Complexity: Medium

### Extract → Triage Table
Reflect → present triage table (learning + suggested tier + reasoning) → user picks/adjusts → batch-write approved items. Heuristic suggests, human decides.
- Gains: User controls routing. No wrong-tier risk. Skill does heavy lifting (identification), human does judgment (placement).
- Costs: More interactive per invocation.
- Complexity: Low-Medium

## Chosen Approach
**Extract → Triage Table** — the skill reflects on the session, identifies friction patterns, and presents a triage table with suggested tiers. User confirms/adjusts per item. Skill batch-writes approved items. The routing heuristic suggests but doesn't decide.

## Key Context Discovered During Shaping
- Prior art validates the core mechanics: Claudeception (skill extraction from sessions) and AccidentalRebel's session-retrospective (structured lessons-learned) both work via in-context reflection
- "The skills that get referenced most aren't clean successes — they're the failures" (Claudeception creator)
- This project already has two friction-born auto-memory files (`feedback_aws_access.md`, `feedback_worktree_isolation.md`) — both came from repeated human corrections
- Session JSONL history available at `~/.claude/projects/<project>/<session-id>.jsonl` but in-context reflection is sufficient (skill runs inside the active conversation)
- Existing `/learn` skill captures patterns but requires manual invocation about a specific thing — no session-wide sweep
- Session closeout verifier (Stop hook) checks journals exist but doesn't distill content
- Routing decisions logged to `.claude/hooks/routing_decisions.jsonl` — potential supplementary signal

## Prior Art
- [Claudeception](https://github.com/blader/Claudeception) — continuous learning, skills-only output
- [Session Retrospective](https://github.com/accidentalrebel/claude-skill-session-retrospective) — console output, no persistence
- [AccidentalRebel blog post](https://www.accidentalrebel.com/building-a-session-retrospective-skill-for-claude-code.html) — practical lessons

## Next Step
- Plan → `/create_plan memory-bank/thoughts/shared/briefs/2026-03-13-retrospective-skill.md`
