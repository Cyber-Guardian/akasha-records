---
date: 2026-02-05T12:00:00-05:00
source_research: memory-bank/thoughts/shared/research/2026-02-05-outlook-graph-concurrency-gates.md
last_generated: 2026-02-05T16:45:34.049804+00:00
---

# Research Brief: 2026-02-05-outlook-graph-concurrency-gates

## TL;DR

Microsoft Graph imposes **two distinct types of limits** for Outlook/Exchange endpoints:

1. **Rate Gates** (already supported): 10,000 requests per 10 minutes per app-mailbox combination
2. **Concurrency Gates** (NOT yet supported): **Maximum 4 concurrent requests per mailbox** (MailboxConcurrency)

The existing throttle service handles rate gates well but has **no support for concurrency/slot-based gates**. Adding concurrency gates requires new Lua functions for slot acquisition/release and new Python abstractions to manage slot lifecycle.

## Key facts / constraints

None captured.

## Key recommendations

None captured.

## Open questions

1. **Slot TTL duration**: What's the appropriate TTL for Outlook operation slots? (Suggested: 30-60 seconds with refresh)
2. **Slot acquisition order**: FIFO vs priority-based slot allocation?
3. **Batch request handling**: Should a JSON batch of 4 requests acquire 4 slots or 1?
4. **Cross-service isolation**: Outlook concurrency limits are per-mailbox across all apps - do we need to coordinate with other services?
5. **Upload bandwidth tracking**: Is 150 MB/5 min worth implementing as a separate gate type?

## Links

- Source research: `memory-bank/thoughts/shared/research/2026-02-05-outlook-graph-concurrency-gates.md`
