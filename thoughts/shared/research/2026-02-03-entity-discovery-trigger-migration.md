---
date: 2026-02-03T12:00:00-05:00
researcher: Claude
git_commit: da5a2d9b8a0f2784e0882581a39cddbb98a7c7b0
branch: main
repository: filescience
topic: "Entity Discovery Trigger Lambda Migration Analysis"
tags: [research, migration, lambda, entity-discovery, cloudbridge, dynamodb]
status: complete
last_updated: 2026-02-03
last_updated_by: Claude
---

# Research: Entity Discovery Trigger Lambda Migration Analysis

**Date**: 2026-02-03T12:00:00-05:00
**Researcher**: Claude
**Git Commit**: da5a2d9b8a0f2784e0882581a39cddbb98a7c7b0
**Branch**: main
**Repository**: filescience

## Research Question

Document the entity-discovery-trigger lambda from `filescience-infrastructure` and identify how it maps to patterns in the `filescience` monorepo for migration.

## Summary

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

## Source Lambda Analysis

### Location
```
/Users/jordanmesches/Documents/work/filescience/filescience-infrastructure/
  aws/application/lambdas/functions/standard/entity-discovery-trigger/
    ├── __init__.py
    ├── config.toml
    ├── inline-policy.json
    ├── lambda_function.py
    └── session.py
```

### Core Functionality (`lambda_function.py`)

**Purpose**: Sync entities from cloud providers to DynamoDB "resources" table.

**Event Schema**:
```python
{
    "CloudId": int,    # Cloud provider ID (e.g., 1 = Office 365)
    "TenantId": int,   # Tenant identifier
    "OrgId": int       # Organization identifier
}
```

**Data Models**:

1. `RemoteEntity` (line 22-26): Represents entity from cloud API
   - `name: str`, `type: str`, `cloud_external_id: str`

2. `Entity` (line 29-67): Full entity with persistence fields
   - Extends RemoteEntity fields
   - Adds: `active: bool`, `tenant_id: int`, `cloud_id: int`, `org_id: int`
   - `id` property: UUID5 from `{tenant_id}:{cloud_external_id}`
   - `pk` property: `f"TENANT#{self.tenant_id}"`
   - `sk` property: `f"ENTITY#{self.id}"`
   - `to_item()`: Returns dict with PK/SK for DynamoDB

**Core Operations**:

1. `fetch_remote(cloud_id)` (line 70-74): Get entities from cloud provider via CloudbridgeClient
2. `fetch_local(table, tenant_id)` (line 77-102): Paginated query for existing entities
3. `diff(remote, local, ...)` (line 105-123): Compute new/reactivated and removed entities
4. `check_active_status(cloud_id, entities)` (line 134-157): Verify entities still active in cloud
5. `update_active_status(table, statuses)` (line 160-174): Update active flags in DynamoDB
6. `save(table, entities)` (line 126-131): Batch write new/reactivated entities
7. `sync_entities(cloud_id, tenant_id, org_id)` (line 177-214): Main orchestration

**Async Pattern**: Uses `anyio.create_task_group()` for concurrent operations.

### Session Management (`session.py`)

**OAuth2TokenAuth Base Class** (line 17-48):
- Manages OAuth2 token lifecycle with 60s expiry buffer
- Abstract `url` and `oauth_body` properties for provider-specific config
- Injects Bearer token into request headers

**MicrosoftOAuth2TokenAuth** (line 51-63):
- Token URL: `https://login.microsoftonline.com/{tenant_id}/oauth2/v2.0/token`
- Scope: `https://graph.microsoft.com/.default`
- Grant type: `client_credentials`

**SessionManager** (line 68-96):
- `create_session(cloud: Cloud)`: Creates `requests.Session` with OAuth auth
- `_get_auth_config(cloud)`: Fetches credentials from SSM `/clouds/{cloud_name}/cloudsmith`

### Configuration (`config.toml`)

```toml
name = "entity-discovery-trigger"
handler = "lambda_function.lambda_handler"
timeout = 900
memory_size = 1024
runtime = "PYTHON_3_12"

kms_required = true
layers = ["core"]
managed_policies = [
    "service-role/AWSLambdaBasicExecutionRole",
    "SecretsManagerReadWrite",
    "AmazonDynamoDBFullAccess",
    "AWSCloudMapDiscoverInstanceAccess",
]
```

### IAM Permissions (`inline-policy.json`)

Allows:
- DynamoDB operations on `{infra_prefix}-cloud-users` and `{infra_prefix}-cloud-organizations` tables
- Secrets Manager read access
- Lambda invocation (for potential chaining)

## Target Repo Pattern Mapping

### 1. Cloud Models

**Source**:
```python
from fscloudbridge.model import Cloud
```

**Target**: `components/filescience/models/core.py`
```python
from filescience.models import Cloud, CloudService, CloudModel
```

The `Cloud` enum is preserved with same numeric IDs:
- `Cloud.OFFICE_365` (ID=1) maps to Microsoft 365
- Services defined as `CloudService` instances

### 2. Session Management

**Source**: Custom `SessionManager` in `session.py`

**Target**: `bases/filescience/discover/clouds/api/session.py`

```python
from filescience.discover.clouds.api.session import SessionManager
from filescience.discover.clouds.api.oauth import MicrosoftOAuth2TokenAuth
```

Key differences:
- Uses `httpx.AsyncClient` instead of `requests.Session`
- OAuth handlers in separate module at `bases/filescience/discover/clouds/api/oauth.py`
- Same SSM path pattern: `/clouds/{cloud_name}/cloudsmith`

### 3. DynamoDB Persistence

**Source**: Direct `aioboto3` operations with dataclass `to_item()` method

**Target**: Three-layer architecture in `components/filescience/dynamodb/`

```python
from filescience.dynamodb import (
    EntitySerializer,
    EntityDynamoDBModelMapper,
    atransactional_write_model_to_table,
    aget_models,
    Query,
)
```

**Existing EntityItem model** (`components/filescience/dynamodb/models.py:55-67`):
```python
class EntityItem(BaseItem):
    name: str
    type: str
    cloud_external_id: str
    active: bool
    cloud_id: int
    org_id: int
```

**PK/SK Pattern**: Same as source
- PK: `TENANT#{tenant_id}`
- SK: `ENTITY#{id}`

### 4. Domain Entity Model

**Source**: `Entity` dataclass in `lambda_function.py`

**Target**: `components/filescience/domain_backup/models.py:44-59`

```python
@dataclass
class Entity:
    name: str
    active: bool
    type: str
    tenant_id: int
    cloud_external_id: str
    org_id: int
    cloud: type[CloudModel]
    id: str = field(default_factory=...)
```

The target Entity stores `cloud: type[CloudModel]` instead of `cloud_id: int`. The mapper handles conversion.

### 5. Lambda Structure

**Source**: Single-file lambda in infrastructure repo

**Target**: Polylith architecture
- **Base**: `bases/filescience/entity_discovery_trigger/` (to be created)
- **Project**: `projects/entity-discovery-trigger/` (to be created)
- Handler exports via `__init__.py` chain

### 6. Build/Deploy

**Source**: `config.toml` with layer names and managed policies

**Target**:
- `Makefile` targets for building wheel and layers
- `infrastructure/modules/lambda/` Terraform module
- `infrastructure/live/non-prod/us-east-1/dev/entity-discovery-trigger/terragrunt.hcl` (to be created)

## Code References

| Source File | Target Equivalent |
|------------|------------------|
| `lambda_function.py:22-67` (Entity classes) | `components/filescience/domain_backup/models.py:44-59` |
| `session.py:17-63` (OAuth2) | `bases/filescience/discover/clouds/api/oauth.py:5-81` |
| `session.py:68-96` (SessionManager) | `bases/filescience/discover/clouds/api/session.py:9-45` |
| `config.toml` | `infrastructure/live/.../entity-discovery-trigger/terragrunt.hcl` |

## Architecture Documentation

### Existing Lambda Pattern (valkey-queue-processor)

```
projects/valkey-queue-processor/
├── pyproject.toml          # Polylith brick mapping
└── src/filescience_valkey_queue_processor/
    └── __init__.py         # Re-exports lambda_handler

bases/filescience/valkey_queue_processor/
├── __init__.py             # Exports lambda_handler
├── handler.py              # Lambda entry point
├── config.py               # Environment variable loading
├── handlers/               # Domain handlers
├── models/                 # Pydantic models
└── services/               # Business logic
```

### Proposed Migration Structure

```
bases/filescience/entity_discovery_trigger/
├── __init__.py             # Export lambda_handler
├── handler.py              # Main sync_entities logic
└── config.py               # Environment variables

projects/entity-discovery-trigger/
├── pyproject.toml          # Brick mappings
└── src/filescience_entity_discovery_trigger/
    └── __init__.py         # Re-export lambda_handler

infrastructure/live/non-prod/us-east-1/dev/entity-discovery-trigger/
└── terragrunt.hcl          # Lambda deployment config
```

### Component Reuse

The migration can reuse existing components:
- `components/filescience/models/` - Cloud enum
- `components/filescience/dynamodb/` - Persistence utilities
- `components/filescience/domain_backup/` - Entity model

New code needed in base:
- OAuth session creation (could extract from discover base)
- CloudbridgeClient equivalent (entity listing from Graph API)
- Sync orchestration logic

## Historical Context

Related research documents:
- `memory-bank/thoughts/shared/research/2026-01-30-dynamodb-utils-migration-analysis-brief.md` - DynamoDB persistence patterns
- `memory-bank/thoughts/shared/research/2026-01-30-valkey-queue-processor-dependency-analysis.md` - Lambda dependency analysis
- `memory-bank/thoughts/shared/plans/2026-01-29-polylith-migration.md` - Polylith migration strategy

## Open Questions

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

## Recommendations for Planning Phase

When creating the migration plan, consider:

1. **Phase 1**: Extract/verify CloudbridgeClient functionality
   - Identify fscloudbridge source
   - Map get_entities() to Graph API calls
   - Map check_entity_active() to appropriate endpoint

2. **Phase 2**: Create base with handler logic
   - Port sync_entities orchestration
   - Use existing DynamoDB utilities
   - Use existing SessionManager or extract to component

3. **Phase 3**: Create project and Terragrunt config
   - Follow valkey-queue-processor pattern
   - Add to Makefile build targets
   - Create terragrunt.hcl with dependencies

4. **Phase 4**: Testing
   - Unit tests for diff logic
   - Integration tests with moto
   - Manual verification in dev environment
