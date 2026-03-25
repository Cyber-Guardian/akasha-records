# Valkey Queue Processor — Dependency Analysis (2026-01-30)

Scope: `bases/filescience/valkey_queue_processor` (the Lambda “function code” base) and how it’s packaged via `projects/valkey-queue-processor`.

## Executive summary

- The base has **one cross-brick (components/) dependency**: `filescience.throttling`.
- The `valkey-queue-processor` project **packages additional components** (e.g. `cloudbridge_model`, `config`, `dynamodb`, `domain_backup`) into the function wheel/zip **even if not imported** by the base. This is driven by Polylith brick mapping in the project’s `pyproject.toml`.
- There is a **potential runtime mismatch** between:
  - the Lambda handler’s Valkey client import (`glide.GlideClient`), and
  - the throttling component’s expected client type (`glide_sync.GlideClient`).
- Lambda layer build currently installs `valkey-glide` but **not** `valkey-glide-sync`; if the throttling component is used (it is), the layer may need to include `valkey-glide-sync` too.

## Cross-brick dependencies (components/)

Only `filescience.throttling` is imported from within the base:

- `bases/filescience/valkey_queue_processor/handler.py`
  - `from filescience.throttling import ThrottleService`
- `bases/filescience/valkey_queue_processor/services/throttle_service.py`
  - `from filescience.throttling import ThrottleContext, RateGateSpec, ThrottleService`
- `bases/filescience/valkey_queue_processor/services/pruning_service.py`
  - `from filescience.throttling import ThrottleService`
- `bases/filescience/valkey_queue_processor/models/__init__.py`
  - re-exports `RequestPolicy`, `ThrottleContext`, `RateGateSpec` from `filescience.throttling`
- `bases/filescience/valkey_queue_processor/policies/registry.py`
  - `from filescience.throttling import M365ThrottleContext, OnedriveTraversalPolicy`

## Internal module graph (high level)

Entrypoint:

- `filescience.valkey_queue_processor.handler.lambda_handler`

Main internal flow:

- `handler.lambda_handler`
  - batch processing + idempotency via Powertools
  - routes DynamoDB stream records to:
    - `handlers/submitted_work_item_handler.py`
    - `handlers/throttle_context_metadata_handler.py`
  - both paths use `filescience.throttling.ThrottleService` (directly or via services)

No circular imports detected in the base.

## Third-party dependencies (runtime)

The base directly imports/uses:

- `aws-lambda-powertools` (Logger/Tracer, DynamoDB streams batch, idempotency)
- `boto3` (+ `botocore.exceptions.ClientError`) for DynamoDB
- `pydantic` for schema validation
- `python-ulid` for fencing tokens
- `valkey-glide` / `valkey-glide-sync` via `glide` / `glide_sync` modules (Valkey connectivity)

## Packaging + “why do we have components we don’t import?”

The project `projects/valkey-queue-processor/pyproject.toml` uses `hatch-polylith-bricks` and explicitly maps multiple components + the base into the wheel. This means the resulting function zip contains those bricks regardless of whether the base imports them today.

## Potential mismatch: `glide` vs `glide_sync`

- `bases/.../handler.py` creates a Valkey client via `glide.GlideClient`.
- `components/filescience/throttling/service.py` expects a `glide_sync.GlideClient` for `ThrottleService`.

This could be harmless if both clients are compatible, but it is a **sharp edge**:

- If `glide_sync` is not present in the Lambda runtime, importing `ThrottleService` will fail.
- If the client objects are not interchangeable, calls inside throttling may fail at runtime.

Mitigations to consider:

- Align handler to use `glide_sync` (sync client) if the throttling component is sync-first, and ensure the Valkey layer installs `valkey-glide-sync`.
- Or switch base to use `AsyncThrottleService` consistently (bigger change).

