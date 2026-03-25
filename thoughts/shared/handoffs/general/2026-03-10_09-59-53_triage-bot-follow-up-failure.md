---
date: 2026-03-10T09:59:53-0400
author: claude
git_commit: 636f79e
branch: fix/ENG-2397-receiver-dedup-s3-memory
repository: filescience
topic: "Triage Bot Follow-Up Failure Investigation Handoff"
tags: [handoff, investigation, triage-agent, ask-human, image-blindness]
status: complete
last_updated: 2026-03-10
last_updated_by: claude
---

# Handoff: Triage bot silently drops follow-up replies after asking clarifying questions

## Task(s)
- **Investigation of triage bot failure** — status: **complete** (root causes identified, no code changes made)
- Harsh posted a message with a PostHog screenshot in `#triage-dropbox` (Slack channel `C0AG1N398P5`, ts `1773150152.310879`). The bot asked for clarification because it couldn't see the image. Harsh replied with details. The bot never responded to the follow-up.

## Critical References
- **Receiver:** `tools/bases/tooling/triage_agent/receiver.py` — event filtering, dedup, routing logic
- **Handler:** `tools/bases/tooling/triage_agent/handler.py` — agent execution, thread reply handling, PassthroughContext
- **Runner:** `tools/components/tooling/durable_harness/runner.py` — multi-turn agent loop, `wait_for_callback` call site
- **Tools:** `tools/bases/tooling/triage_agent/tools.py` — `ask_human` tool (deferred, raises `CallDeferred`)
- **Agent authoring rule:** `.claude/rules/agent-authoring.md` — prompt-capability alignment (violated)
- **ENG-2397 plan:** `memory-bank/thoughts/shared/plans/2026-03-09-ENG-2397-fix-triage-agent-crash.md`

## Root Causes (two cascading issues)

### RC1: Image blindness (why the bot asked for clarification at all)
- `dispatcher.py:67-73` only appends text metadata about file attachments: `[Attached file: image.png (image/png)]`
- Actual image content/URL is never passed to the model
- Slack app lacks `files:read` OAuth scope (`slack-manifest.yaml:29-33`) — can't download images even if it tried
- Claude Haiku CAN process images but never receives them

### RC2: Follow-up replies silently dropped (the structural failure)
This is the critical bug affecting ALL multi-turn conversations:

1. `ask_human` tool raises `CallDeferred` → runner calls `ctx.wait_for_callback()` (`runner.py:152`)
2. `PassthroughContext.wait_for_callback()` raises `NotImplementedError` (`handler.py:306-308`) — durable execution not enabled
3. Agent's only option: return clarifying text as final output
4. Agent completes without calling `create_linear_issue` → thread marked `IGNORED` (`handler.py:235-237`)
5. Human reply arrives → receiver routes as `thread_reply` (`receiver.py:206-210`, thread_ts != ts)
6. `_handle_thread_reply()` checks parent thread state (`handler.py:249`)
7. State is `IGNORED` (not `ISSUE_CREATED`) → reply silently dropped: `"No completed conversation for this thread"`

**Causal chain:**
```
Bot can't see image → asks for clarification as text
  → ask_human broken (PassthroughContext) → agent returns text instead
    → thread marked IGNORED → follow-up reply silently dropped
```

**Violated rule:** `.claude/rules/agent-authoring.md` — `ask_human` is in the default tool list (`handler.py:54`) but crashes in the production context. The agent either uses it (crash with NotImplementedError) or doesn't (follow-ups dropped).

## Learnings
- `PassthroughContext` (`handler.py:300-308`) is the non-durable fallback. It handles `step()` (just executes the function) but raises `NotImplementedError` on `wait_for_callback()`. This means ALL deferred tools are broken in the current deployment.
- The receiver's routing logic (`receiver.py:195-229`) correctly identifies thread replies and checks DynamoDB for `AWAITING_CALLBACK` state. But the agent can never reach that state because `wait_for_callback` crashes before `_on_human_wait` is called.
- Thread state lifecycle: `PROCESSING` → agent runs → either `ISSUE_CREATED`/`DUPLICATE_FOUND`/`IGNORED`/`TIMED_OUT`. Only `ISSUE_CREATED` gets follow-up handling (comment added to Linear issue). All other states silently drop follow-ups.
- The `_handle_thread_reply` function (`handler.py:242-277`) is a "comment-only fallback" — it was designed to add follow-up comments to already-created Linear issues, NOT to handle multi-turn clarification flows.

## Artifacts
- This handoff document
- No code changes were made — investigation only

## Action Items & Next Steps

### Fix 1 (High priority): Make follow-up replies work without durable execution
Since `ask_human`/`wait_for_callback` can't work until Lambda Durable Execution is enabled, the bot needs a non-durable path for multi-turn conversations:

**Option A — Re-invoke agent on follow-ups:** When a thread reply arrives for a thread in `IGNORED` or `PROCESSING` state, re-invoke the agent with the full conversation history (original message + follow-up) instead of silently dropping it. This makes the agent stateless but multi-turn.

**Option B — Remove `ask_human` from tool list:** Remove `ask_human` from the default tools (`handler.py:54`) and update the prompt to never ask clarifying questions — just create a best-effort issue with open questions in the body. This avoids the problem entirely but reduces issue quality.

**Option C — Implement non-durable ask_human:** Replace `CallDeferred` with a direct Slack post + state save, then have the receiver re-invoke the full agent with conversation history when the reply comes. Emulates durable execution without the SDK.

**Recommendation:** Option A is the simplest and most robust. Option C is the most complete but requires more work.

### Fix 2 (Medium priority): Pass image content to the model
- Add `files:read` scope to `slack-manifest.yaml`
- In `dispatcher.py:67-73`, download file content via Slack API and include the image URL (or base64) in the payload sent to the handler
- Update the agent to use multimodal input (Claude Haiku supports vision)

### Fix 3 (Low priority): Remove or gate `ask_human` tool
Per `.claude/rules/agent-authoring.md`, remove `ask_human` from the default tool list until durable execution is enabled. It's currently a trap: the model may try to use it and crash.

## Other Notes
- The current branch `fix/ENG-2397-receiver-dedup-s3-memory` already has event dedup + S3 memory sync work. The follow-up failure is a separate issue discovered during this investigation.
- Slack thread for reference: `https://cyberguardiangroup.slack.com/archives/C0AG1N398P5/p1773150152310879`
- The bot DID add an :eyes: reaction to Harsh's original message (receiver.py:189), so the receiver processed the event. The drop happens downstream in the handler.
- All four Lambda functions in the chain: receiver → dispatcher (events queue) → handler → (callback_handler never triggered because state was never AWAITING_CALLBACK)
