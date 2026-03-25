---
date: 2026-02-03T12:00:00-05:00
source_research: memory-bank/thoughts/shared/research/2026-02-03-entity-discovery-trigger-migration.md
last_generated: 2026-02-03T18:16:23.588458+00:00
---

# Research Brief: 2026-02-03-entity-discovery-trigger-migration

## TL;DR

The entity-discovery-trigger lambda syncs entities (users/groups) from cloud providers to DynamoDB. It uses:
- `fscloudbridge` for cloud API access (CloudbridgeClient, Cloud enum, SessionManager)
- `aioboto3` for async DynamoDB operations
- Dataclass-based Entity model with PK/SK patterns
- `anyio` for async orchestration

The target monorepo has equivalent patterns:
- Cloud models in `components/filescience/models/`
- Session management in `bases/filescience/discover/clouds/api/`
- DynamoDB persistence in `components/filescience/dynamodb/`
- Entity model in `components/filescience/domain_backup/models.py`

## Key facts / constraints

None captured.

## Key recommendations

None captured.

## Open questions

1. **CloudbridgeClient Equivalent**: The source uses `CloudbridgeClient.get_entities()` and `CloudbridgeClient.check_entity_active()`. Need to determine:
   - Where is `fscloudbridge` source code?
   - Should these methods be added to GraphClient in discover base?
   - Or create separate component for entity discovery?

2. **Session Sharing**: The discover base already has SessionManager. Should entity-discovery-trigger:
   - Use the existing SessionManager from discover?
   - Have its own copy (for independence)?
   - Extract SessionManager to a shared component?

3. **Trigger Mechanism**: Source lambda expects event with CloudId/TenantId/OrgId. What triggers this?
   - Scheduled EventBridge rule per tenant?
   - SNS/SQS message?
   - Step Functions orchestration?

4. **Table Name**: Source uses hardcoded `TABLE = "resources"`. Target should use environment variable like `RESOURCES_TABLE_NAME`.

5. **Idempotency**: Source has no idempotency layer. Should migration add PowerTools idempotency?

## Links

- Source research: `memory-bank/thoughts/shared/research/2026-02-03-entity-discovery-trigger-migration.md`
