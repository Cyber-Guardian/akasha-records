---
date: 2026-03-05T12:00:00-05:00
source_research: memory-bank/thoughts/shared/research/2026-03-05-legacy-discover-feature-gap-analysis.md
last_generated: 2026-03-05T18:45:36.797149+00:00
---

# Research Brief: 2026-03-05-legacy-discover-feature-gap-analysis

## TL;DR

The current monorepo discover service supports **2 of 7 Microsoft services** (OneDrive, Outlook) and **0 of 6 non-Microsoft cloud providers** that exist in the legacy. Beyond cloud coverage, there are significant architectural differences: the legacy uses multi-threaded synchronous processing with SQS output, while the current uses async processing with a DynamoDB work queue and Valkey throttle tree. Several legacy internal features (version backfill, stall detection, Step Functions integration, org stats) have no equivalent in the current codebase.

## Key facts / constraints

None captured.

## Key recommendations

None captured.

## Open questions

1. **Prioritization**: Clio and Box are listed as supported clouds in identity.md. What's the timeline for adding their discover handlers?
2. **ArtifactVersion handling**: The current OneDrive handler yields versions, but the dispatcher skips them. Is this intentional (deferred) or a bug?
3. **Missing M365 services**: Models and graph client methods exist for Teams, Calendar, Contacts, Todo, and Planner. Are router handlers planned?
4. **SQS vs DynamoDB**: Was the shift from SQS output to direct DynamoDB writes an intentional architectural decision, or is SQS integration planned?
5. **Stall detection**: Is the 10s idle timeout sufficient, or does the Valkey throttle tree provide equivalent protection?

## Links

- Source research: `memory-bank/thoughts/shared/research/2026-03-05-legacy-discover-feature-gap-analysis.md`
