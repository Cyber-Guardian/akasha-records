---
date: 2026-01-29T16:50:00-05:00
author: claude
git_commit: a9c4eca99026b1a655109edac3a0383532c46aab
branch: main
repository: filescience
topic: "Polylith Migration Phase 4 Handoff"
tags: [handoff, implementation, polylith, discover, migration]
status: complete
last_updated: 2026-01-29
last_updated_by: claude
---

# Handoff: Polylith Migration - Phase 4 Complete, Ready for Phase 5

## Task(s)
- **Phase 4: Discover Base & Project** - COMPLETED
  - Migrated discover CLI from `/Users/jordanmesches/Documents/work/filescience-discover-v2/src/` to `bases/filescience/discover/`
  - Updated all imports from `fsbedrock.*` → `filescience.*` and `fscloudbridge.*` → `filescience.cloudbridge_*`
  - Created project entry point at `projects/discover/src/filescience_discover/`
  - Fixed project pyproject.toml brick paths to include `filescience/` namespace

- **Phase 5: Valkey Lambda Base & Project** - NOT STARTED
- **Phase 6: Build System** - NOT STARTED

## Critical References
1. **Implementation Plan**: `memory-bank/thoughts/shared/plans/2026-01-29-polylith-migration.md`
2. **Source Analysis**: `memory-bank/thoughts/shared/research/2026-01-29-polylith-migration-source-analysis.md`

## Recent changes

### Files Created in Phase 4:
- `bases/filescience/discover/__init__.py` - Exports DiscoveryManager, DispatcherV2, WorkQueue, WorkItem
- `bases/filescience/discover/manager.py` - DiscoveryManager with optional glide_sync imports
- `bases/filescience/discover/dispatcher.py` - DispatcherV2 with DynamoDB stubs
- `bases/filescience/discover/queue.py` - WorkQueue with throttle context management
- `bases/filescience/discover/session.py` - aioboto3 optimized session factory
- `bases/filescience/discover/aws.py` - SSM parameter utilities
- `bases/filescience/discover/clouds/` - Cloud router infrastructure:
  - `router.py` - CloudRouter, ServiceRouter, Context, BackupState
  - `models.py` - ServiceEntity
  - `api/client.py` - ApiClient, ResponseIterator, pagination strategies
  - `api/oauth.py` - OAuth2TokenAuth implementations (Microsoft, Box, Clio, NetDocs)
  - `api/session.py` - SessionManager with auth binding
  - `api/suppression.py` - Error suppression rules
  - `microsoft/cloud.py` - O365Router setup
  - `microsoft/api/client.py` - GraphClient, GraphResponse, all Graph components
  - `microsoft/api/models.py` - All Microsoft Graph API Pydantic models
  - `microsoft/services/onedrive.py` - OneDrive service router handlers

### Files Modified:
- `projects/discover/pyproject.toml:27-39` - Fixed brick paths to include `filescience/` namespace
- `projects/valkey-queue-processor/pyproject.toml:18-23` - Fixed brick paths to include `filescience/` namespace
- `memory-bank/thoughts/shared/plans/2026-01-29-polylith-migration.md:627-630` - Updated Phase 4 checkmarks

## Learnings

1. **Polylith "loose" theme namespace structure**: Components/bases must be at `components/filescience/<name>/` and `bases/filescience/<name>/` for the `filescience.*` namespace to work. Project brick paths need to reference `../../components/filescience/<name>` not `../../components/<name>`.

2. **glide_sync import handling**: The glide_sync package may not be available at import time. Made imports optional with `GLIDE_AVAILABLE` flag and graceful fallback.

3. **DynamoDB persistence gap**: The discover dispatcher uses several utilities from `fsbedrock.persistence.dynamodb` that haven't been migrated yet:
   - `aget_model_from_table`, `atransactional_write_model_to_table`, `awrite_model_to_table`
   - `ResourceSerializer`, `ArtifactSerializer`, `DeltaStateSerializer`
   - `ResourceDynamoDBModelMapper`, `ArtifactDynamoDBModelMapper`, `DeltaStateDynamoDBModelMapper`
   These are stubbed in `dispatcher.py` with TODO comments.

4. **Cloud model adaptation**: The original code used `model.Cloud.from_id()` and `model.CloudService.from_id()`. The new code uses `Cloud.from_id()` and `Cloud.service_from_id()` - these methods need to exist on the Cloud class. The CloudModel implementation may need adjustment.

## Artifacts

### Implementation Plan
- `memory-bank/thoughts/shared/plans/2026-01-29-polylith-migration.md` (Phases 1-3 complete, Phase 4 complete)

### Research Documents
- `memory-bank/thoughts/shared/research/2026-01-29-polylith-migration-source-analysis.md`
- `memory-bank/thoughts/shared/research/2026-01-29-polylith-migration-source-analysis-brief.md`

### Migrated Code
- `bases/filescience/discover/` - Complete discover base (20 Python files)
- `projects/discover/src/filescience_discover/` - Project entry point

## Action Items & Next Steps

### Immediate (Phase 5)
1. Read Phase 5 section of the plan
2. Copy valkey_queue_processor Lambda from `/Users/jordanmesches/Documents/work/filescience/filescience-infrastructure/aws/application/lambdas/functions/standard/valkey_queue_processor/`
3. Migrate to `bases/filescience/valkey_queue_processor/`
4. Update imports from `fsbedrock.throttling` → `filescience.throttling`
5. Create project entry point at `projects/valkey-queue-processor/src/filescience_valkey_queue_processor/`
6. Copy Valkey Lua script to infrastructure module

### Later (Phase 6)
1. Create Makefile with build-lambda, build-layers targets
2. Create Terragrunt modules for artifact-bucket, lambda, lambda-layer
3. Create live configurations for dev environment

### Known Issues to Address
1. `Cloud.from_id()` and `Cloud.service_from_id()` methods may not exist on CloudModel - verify or implement
2. DynamoDB persistence utilities need full migration from bedrock (currently stubbed)
3. ThrottleService integration needs glide_sync package installation

## Other Notes

### Verification Commands
```bash
# Check workspace structure
uv run poly info

# Test imports
uv run python -c "from filescience.discover import DiscoveryManager"

# Build wheel
uv build --wheel projects/discover -o build/wheels

# Run module (will error without infrastructure)
uv run --project projects/discover python -m filescience_discover
```

### Original Source Locations
- Discover: `/Users/jordanmesches/Documents/work/filescience-discover-v2/src/`
- Bedrock: `/Users/jordanmesches/Documents/work/filescience/bedrock/fsbedrock/`
- Cloudbridge: `/Users/jordanmesches/Documents/work/filescience/filescience-cloudbridge/fscloudbridge/`
- Valkey Lambda: `/Users/jordanmesches/Documents/work/filescience/filescience-infrastructure/aws/application/lambdas/functions/standard/valkey_queue_processor/`
