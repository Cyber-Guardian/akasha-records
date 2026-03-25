---
date: 2026-02-13T14:53:25.950021+00:00
source_research: memory-bank/thoughts/shared/research/2026-02-12-architecture-diagram-automation.md
last_generated: 2026-02-13T14:53:25.950021+00:00
---

# Research Brief: 2026-02-12-architecture-diagram-automation

## TL;DR

**Date:** 2026-02-12
**Scope:** Diagram types, technologies, and automation strategies for Python/Polylith/AWS monorepo
---
## Part 1: Diagram Types — What Exists and When to Use Each
### C4 Model (Modern Standard)
The C4 model provides four hierarchical abstraction levels. Best-fit for our Polylith + AWS Lambda architecture.
| Level | Shows | Auto-generable? | Our Use Case |
|-------|-------|-----------------|--------------|
| **1 — System Context** | System as box + users + external systems | No (requires human boundary judgment) | FileScience + Microsoft Graph API + tenants |
| **2 — Container** | Independently deployable units | Partial (from Terraform) | Lambda functions, DynamoDB, Valkey, S3 |

## Key facts / constraints

None captured.

## Key recommendations

None captured.

## Open questions

None

## Links

- Source research: `memory-bank/thoughts/shared/research/2026-02-12-architecture-diagram-automation.md`
