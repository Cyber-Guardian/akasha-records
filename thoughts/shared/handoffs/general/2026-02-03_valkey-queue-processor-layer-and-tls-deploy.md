## Summary
- Deployed updates to `filescience-dev-valkey-queue-processor` in **us-east-1**.
- Fixed runtime connectivity assumptions for dev Valkey:
  - Dev Valkey replication group has **transit encryption disabled** (`transit_encryption_enabled = false`), so clients must use **plaintext**.
  - Lambda now sets `VALKEY_USE_TLS="false"` via Terragrunt.
- Updated Valkey layer build to include **`valkey-glide-sync`** so the Lambda can import `glide_sync`.
- Ensured the Lambda imports succeed (`glide_sync`, `aws_xray_sdk` present via layers). A direct invoke with `{}` still fails (expected) because the handler expects a DynamoDB Streams batch event.

## Changes made
- Code:
  - `bases/filescience/valkey_queue_processor/config.py`: added `VALKEY_USE_TLS` env parsing.
  - `bases/filescience/valkey_queue_processor/handler.py`: switched Valkey client to `glide_sync` and made TLS configurable.
  - `components/filescience/throttling/service.py`: `get_optimal_throttle_context()` now returns `None` if the tree doesn't exist (avoid noisy errors during startup).
- Build:
  - `Makefile`: Valkey layer now installs `valkey-glide` **and** `valkey-glide-sync`.
  - `docs/lambda-deployment.md`: updated layer contents description.
- Infra:
  - `infrastructure/live/non-prod/us-east-1/dev/valkey-queue-processor/terragrunt.hcl`: added `VALKEY_USE_TLS="false"`.
  - `infrastructure/modules/lambda-layer/main.tf`: added `skip_destroy = true` to layer resources (note: applying this can still force replacements).

## Deployment notes (dev)
- Upload artifacts:
  - `make clean && make build`
  - `make upload-latest ARTIFACT_BUCKET=filescience-dev-artifacts`
- Terragrunt applies:
  - `infrastructure/live/non-prod/us-east-1/dev/lambda-layers`: published new layer versions.
  - `infrastructure/live/non-prod/us-east-1/dev/valkey-queue-processor`: updated Lambda to new layer ARNs and env vars.

## Gotchas
- Uploading to a constant S3 key like `.../latest/layer.zip` does **not** automatically cause `aws_lambda_layer_version` / `aws_lambda_function` to update unless something forces a change.
  - In dev, publishing new layer versions required explicit replacement/recreation of the layer resources.
- When invoking/testing from CLI, ensure region is set:
  - Use `--region us-east-1` or set `AWS_REGION=us-east-1`.

