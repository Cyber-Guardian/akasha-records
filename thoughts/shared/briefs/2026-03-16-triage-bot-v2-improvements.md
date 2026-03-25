---
type: event
created: 2026-03-16
status: active
---
# Idea Brief: Triage Bot V2 Improvements

**Date:** 2026-03-16
**Status:** Shaped -> Planning

## Problem
The triage bot has bugs that undermine trust (phantom project/label assignment, PAT identity attribution), UX patterns that frustrate daily users (over-questioning with no override, gatekeeping valid requests), and lacks product context to be genuinely useful without hand-holding. Real Slack usage from Niklas and Harsh reveals the bot interrogates instead of shaping, claims metadata it never sets, and misattributes follow-ups.

## Constraints
- Bot authenticates with Jordan's PAT -- every issue shows as created by Jordan
- `create_linear_issue` tool never passes `project_id` despite the client supporting it
- Label resolution fails silently (returns `[]` -> `None`) with no logging
- Follow-up replies in threads are always treated as comments on the existing issue, even when they're unrelated new requests
- "Reinitializing" messages leak internal durable execution machinery to users
- The interview/shaping approach is the RIGHT default -- users want it. The problem is no override, no round cap, and answers aren't incorporated back into issues
- The prompt tells the bot to ignore non-engineering messages, which causes it to gatekeep valid requests in the triage channel
- Memory bank tools exist but are underutilized -- bot lacks product glossary and team context

## Options Considered

### A. Wire It Up (plumbing fixes only)
Fix bugs: PAT identity, wire project assignment, label verification, follow-up misattribution, suppress "reinitializing", dedup.
- Gains: Stops the bot from lying. Immediate trust improvement.
- Costs: Doesn't fix UX or intelligence gaps.
- Complexity: Low-Medium

### B. Better Judgment (prompt + UX only)
Smarter interview with override mechanism, max question rounds, update-after-clarify, stop gatekeeping, self-awareness.
- Gains: Dramatically better UX for daily users.
- Costs: Bot still can't route to projects or assign people.
- Complexity: Medium

### C. Product Brain (context + routing only)
Product glossary, assignee routing, better memory usage, project resolution via GraphQL.
- Gains: Bot becomes genuinely knowledgeable.
- Costs: Maintenance burden for context freshness.
- Complexity: Medium

### D. Full Stack, Phased (chosen)
All three layers, shipped incrementally: Phase 1 (A), Phase 2 (B), Phase 3 (C).
- Gains: Each phase independently valuable, complete solution.
- Costs: Largest total investment, but each phase is 1-2 days.
- Complexity: Medium overall

## Chosen Approach
**Full Stack, Phased** -- ship A (trust) first because the bot lying about metadata undermines everything else. Then B (smarter interview + override) for the biggest UX win. Then C (product brain) as the long-term differentiator.

Key refinement from shaping: the interview approach is the RIGHT default (like /shape). The fix is not "less questioning" but "smarter questioning + override + incorporate answers."

## Key Context Discovered During Shaping
- `tools.py:79-85` -- `create_linear_issue` never passes `project_id`, despite `client.py:257-286` fully supporting it
- `tools.py:286-299` -- `_resolve_label_ids` fails silently, returns `[]` when category doesn't match config
- `prompts.py:18-93` -- system prompt includes project routing as text but no mechanism to execute it
- `queries.py:63-80` -- `CREATE_ISSUE` GraphQL mutation already accepts `$projectId`, `$labelIds`, `$assigneeId`
- `linear-config.yaml:7-31` -- project routing patterns exist but are prompt-only
- Slack thread analysis: bot averaged 2-3 rounds of questions, worst case 3+ rounds with no issue update (backup button pipeline thread)
- Harsh explicitly requested override mechanism ("kill switch" message 2026-03-10)
- Niklas explicitly requested project assignment ("triage dropbox assign to projects" message 2026-03-10)
- Bot misidentified itself when Niklas gave it feedback -- thought "triage dropbox" meant "Dropbox integration"
- Bot pushed back on Harsh's logo request as "not for engineering" -- gatekeeping a valid request in the triage channel
- Bot misattributed Niklas's separate request as a follow-up to an unrelated issue (peek thread)
- Prior research: [[2026-03-04-ai-linear-experience-improvements|AI Linear Experience Improvements]] flagged single-pass classification and no learning from corrections
- Prior plan: [[2026-03-13-triage-bot-structural-hardening|Triage Bot Structural Hardening]] covers some infrastructure

## Next Step
- [Plan] -> `/create_plan memory-bank/thoughts/shared/briefs/2026-03-16-triage-bot-v2-improvements.md`
