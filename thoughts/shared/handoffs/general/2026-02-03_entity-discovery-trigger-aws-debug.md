#+#+#+#+ 1,25 @@
# Entity Discovery Trigger - AWS Debug Notes

**Date**: 2026-02-03  
**Scope**: `filescience-dev-entity-discovery-trigger` (us-east-1)

## What failed
- Lambda invocation reached Graph API fetch, then failed on DynamoDB query.
- CloudWatch error: `TypeError: cannot pickle '_thread.lock' object`
- Stack trace points to `boto3/dynamodb/transform.copy_dynamodb_params` during `table.query(...)`.

## Root cause
- `fetch_local_entities()` passed an **aioboto3 Table object** into `aget_models(...)`.
- `aget_models` expects a **table name** and internally calls `resource.Table(table_name)`.
- This leads to a `TableName` parameter that is a Table object, which fails `deepcopy()` inside DynamoDB request param handling (triggered when X-Ray/patching is enabled).

## Fix
- Use `aget_models_from_table(...)` when a table object is already available.
- Change in `bases/filescience/entity_discovery_trigger/handler.py`:
  - Import `aget_models_from_table`
  - Replace `aget_models(...)` call in `fetch_local_entities(...)`

## Follow-up failure + fix
- After deploy, Lambda failed with:
  - `ValidationException: Query condition missed key schema element: pk`
- Root cause: code used **PK/SK** while DynamoDB table schema uses **pk/sk**.
- Fixes:
  - Normalize DynamoDB serialization to **write pk/sk**, and accept pk/sk on read
  - Update queries/keys to use **pk/sk** in:
    - `bases/filescience/entity_discovery_trigger/handler.py`
    - `bases/filescience/discover/clouds/router.py`
    - `bases/filescience/discover/dispatcher.py`
- Result: Lambda invocation succeeds, logs show sync complete.

## Evidence
- Log group: `/aws/lambda/filescience-dev-entity-discovery-trigger`
- Log stream: `2026/02/04/[$LATEST]61bed55da416437dadc1f78d3a31b7c6`
