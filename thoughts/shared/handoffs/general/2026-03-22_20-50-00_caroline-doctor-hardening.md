---
type: event
created: 2026-03-22
status: active
date: 2026-03-22T20:50:00-04:00
author: claude-opus-4-6
git_commit: 79d0c05b
branch: main
repository: filescience
topic: "Caroline Doctor System Hardening"
tags: [handoff, caroline, doctor, tunnel, linq, imessage]
last_updated: 2026-03-22
last_updated_by: claude-opus-4-6
---

# Handoff: Caroline doctor system needs hardening + QMD verification

## Task(s)

| Task | Status |
|------|--------|
| Caroline iMessage POC (Linq + Claude SDK) | In progress — functional but fragile |
| Interaction manifesto | Completed — committed, cross-referenced, deck generated |
| BaseAgent + ChannelAdapter component | Completed — Polylith brick, 12 contracts pass |
| Cognitive state choreography (decoupled frontend/backend) | Completed but needs testing |
| Doctor startup + liveness monitor | In progress — startup works, liveness unreliable |
| QMD search end-to-end verification | Not verified — agent returns fallback instead of search results |
| Dynamic model routing (haiku/sonnet/opus) | Completed |
| Interrupt + merge for follow-up messages | Completed |
| Session-based conversation temperature | Completed |

## Critical References

1. **Caroline source:** `tools/projects/caroline/src/caroline/` — all agent code
2. **Ethos:** `tools/projects/caroline/ETHOS.md` — "superhuman capability, human feel" design philosophy
3. **Alignment report (in session log):** Session identified 3 silent drops and plan/ethos contradictions. Plan says "narrate reasoning" but ethos says "no intermediate steps." Ethos wins.
4. **Interaction manifesto:** `memory-bank/thoughts/shared/reference/akasha-interaction-manifesto.md`
5. **Caroline plan (STALE):** `memory-bank/thoughts/shared/plans/2026-03-21-caroline-imessage-poc.md` — does not reflect current architecture (Claude SDK, cognitive states, decoupled frontend/backend)

## Recent changes

Key files in `tools/projects/caroline/src/caroline/`:

- **`sdk_agent.py`** — Claude SDK agent with persistent client, dynamic model routing (haiku/sonnet/opus), MCP servers (QMD + graph_memory). `run_sdk_agent()` returns `(response, tier)`. **BUG:** Response capture may still be broken — was only reading `ResultMessage.result` (often empty). Last fix added `AssistantMessage` content capture but untested.
- **`choreography.py`** — Decoupled frontend. `orchestrate(chat_id, backend_coroutine)` runs model in parallel with cognitive states (unread → read → thinking silence → compose with pause-resume → send). Target times are `min(profile_target, tier_max)`. Tier caps: haiku=8s, sonnet=20s, opus=45s.
- **`doctor.py`** — Startup health check + liveness monitor (30s interval). Auto-heals tunnel + re-registers Linq webhook. **PROBLEM:** Cloudflared free tier rotates URLs frequently. The liveness monitor detects it but the heal cycle (kill → restart → DNS propagation → webhook registration) takes 15-20s during which Caroline is unreachable.
- **`__main__.py`** — Entry point with interrupt+merge message loop, session-based temperature, liveness monitor task.
- **`state.py`** — In-memory state with `get_session_temp()` based on `seconds_since_last_message()`. HOT < 2min, WARM < 15min, COLD > 15min.
- **`linq_client.py`** — Linq V3 REST API client (send, typing, read receipts, webhooks).
- **`webhook.py`** — FastAPI webhook receiver with lenient signature verification.

## Learnings

1. **Cloudflared free tier is fundamentally unreliable** for a persistent service. URLs rotate without warning. Even with a liveness monitor, there's a 15-20s gap during heal. Options: (a) pay for Cloudflare Tunnel with fixed subdomain ($5/mo), (b) use bore with a VPS, (c) deploy Caroline to a server with a stable IP.

2. **Claude SDK spawns a subprocess** (`SubprocessCLITransport`). Each `ClaudeSDKClient` starts a full Claude Code CLI process. MCP servers are started inside that subprocess. `continue_conversation=True` keeps the client alive across queries but each query still goes through the subprocess IPC.

3. **`set_model()` is async** — forgetting to await it silently ignores model selection.

4. **`ResultMessage.result` is often empty** — the actual response text is in `AssistantMessage.content` blocks that stream before the `ResultMessage`. Must capture the last `AssistantMessage` text, not just `ResultMessage.result`.

5. **`setting_sources=[]` is critical** — without it, the SDK loads the repo's massive `CLAUDE.md` into context, causing "Prompt is too long" errors. Caroline's `cwd` must be her own project dir, not the repo root.

6. **The manifesto's P2 "Show the Work" contradicts the ethos** for conversational agents. The ethos says "absence communicates" — silence after Read IS the thinking. No narration. This is a deliberate divergence that needs to be documented in the manifesto.

7. **Linq API quirks:** Webhook subscription requires HTTPS. `create_chat` response nests under `chat.id` not `chat_id`. `send_message` needs `message.parts` wrapper. Typing indicator persists until stopped or message sent. No documented rate limits.

## Artifacts

- `tools/projects/caroline/` — entire Caroline project (11 Python modules)
- `tools/projects/caroline/ETHOS.md` — design philosophy
- `tools/projects/caroline/.env` — credentials (gitignored)
- `tools/projects/caroline/.gitignore`
- `tools/projects/caroline/pyproject.toml` — deps including filescience editable source
- `components/filescience/agent_runner/` — BaseAgent + ChannelAdapter + CLIChannel
- `memory-bank/thoughts/shared/reference/akasha-interaction-manifesto.md` — interaction manifesto
- `memory-bank/thoughts/shared/briefs/2026-03-21-akasha-interaction-design-manifesto.md`
- `memory-bank/thoughts/shared/briefs/2026-03-21-caroline-imessage-poc.md`
- `memory-bank/thoughts/shared/briefs/2026-03-21-generic-agent-runner.md`
- `memory-bank/thoughts/shared/plans/2026-03-21-caroline-imessage-poc.md` (STALE)
- `memory-bank/thoughts/shared/plans/2026-03-21-akasha-interaction-design-manifesto.md` (completed)
- `memory-bank/thoughts/shared/plans/2026-03-21-generic-agent-runner.md`
- `tools/dev-harness/src/dev_harness/adapters/caroline_runner.py` — dev harness adapter
- `/tmp/deck/akasha-interaction-manifesto-slides.pdf` — presentation deck

## Action Items & Next Steps

### Priority 1: Fix the tunnel reliability
The cloudflared free tier is the #1 source of failures. Options:
- **Cloudflare Tunnel with fixed subdomain** — `cloudflared tunnel create caroline` + DNS CNAME. $0 if using Cloudflare DNS, stable URL, no rotation. Requires Cloudflare account + domain.
- **bore + VPS** — `bore local 8080 --to your-vps:7835`. Stable, $5/mo DigitalOcean droplet.
- **Deploy Caroline to a server** — skip tunnels entirely. Run on the compute host or a small EC2 instance with a public IP.

### Priority 2: Fix response capture
`sdk_agent.py` response capture (`AssistantMessage.content` blocks) was just added but never verified. Test by texting Caroline a knowledge base question and checking logs for `Response: N chars` where N > 0.

### Priority 3: Verify QMD works through the SDK
The SDK has QMD connected as an MCP server (`MCP qmd: connected` in logs). But we never confirmed it actually searches and returns results. Test: text Caroline "what do we know about the effort heuristic?" — she should find the manifesto. Check logs for QMD tool calls.

### Priority 4: Update stale plan
`2026-03-21-caroline-imessage-poc.md` plan is stale — doesn't reflect Claude SDK migration, cognitive state choreography, or ethos. Either mark as `superseded` and write a new plan, or update in place.

### Priority 5: Reconcile manifesto P2 with ethos
Add a note to the interaction manifesto that P2 "Show the Work" applies to dashboard/generative UI surfaces but NOT to the conversational agent, where "absence communicates" takes precedence.

## Other Notes

- **Linq credentials:** API key in `tools/projects/caroline/.env`. Phone number: +12063967531 (Caroline), +18453033357 (Jordan). Chat ID: `f5a5993b-ea9b-4905-b58e-6c7ca8cd7b4b`.
- **To run Caroline:** `cd tools/projects/caroline && uv run python -m caroline` — doctor auto-starts tunnel + registers webhook.
- **To kill Caroline:** `pkill -f "python -m caroline"; pkill -f cloudflared`
- **The program (`.claude/program.md`) says autoresearch is the focus.** Caroline was a creative detour. The user chose to keep the program unchanged (option B). Caroline work is not in the program.
- **20+ commits on main** from this session covering manifesto, Caroline, agent runner, and dep fixes. All pushed to origin.
