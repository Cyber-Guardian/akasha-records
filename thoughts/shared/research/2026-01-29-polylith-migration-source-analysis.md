---
date: 2026-01-29T14:30:00-05:00
researcher: Claude
git_commit: a9c4eca99026b1a655109edac3a0383532c46aab
branch: main
repository: filescience
topic: "Polylith Migration Source Analysis - Discover, Bedrock, Cloudbridge, Valkey Throttle"
tags: [research, polylith, migration, discover, bedrock, cloudbridge, valkey, lambda, throttling]
status: complete
last_updated: 2026-01-29
last_updated_by: Claude
---

# Research: Polylith Migration Source Analysis

**Date**: 2026-01-29T14:30:00-05:00
**Researcher**: Claude
**Git Commit**: a9c4eca99026b1a655109edac3a0383532c46aab
**Branch**: main
**Repository**: filescience

## Research Question

Document the structure and dependencies of 5 codebases being migrated into the Polylith monorepo:
1. Discover (CLI application)
2. Bedrock (shared library)
3. Cloudbridge (cloud integrations library)
4. Valkey Lua Script (throttle service)
5. Valkey Queue Processor Lambda (with layer system)

## Summary

The migration involves 5 interconnected codebases that form a distributed backup discovery system:

| Component | Type | Purpose | Key Dependencies |
|-----------|------|---------|------------------|
| **Discover** | CLI App | Cloud resource traversal orchestrator | bedrock, cloudbridge, httpx, anyio, uvloop, glide |
| **Bedrock** | Library | Shared infrastructure (config, auth, persistence, throttling) | boto3, aioboto3, SQLAlchemy, pydantic, valkey-glide |
| **Cloudbridge** | Library | Cloud provider OAuth & API integrations | requests, jwt, boto3 (SSM) |
| **Valkey Lua** | Script | Distributed rate limiting with tree scheduling | Pure Lua for Redis/Valkey |
| **Lambda** | Function | DynamoDB stream processor for throttle context registration | bedrock.throttling, Lambda Powertools, glide |

### Dependency Graph
```
Discover (CLI)
    ├── bedrock (fsbedrock)
    │   ├── cloudbridge (fscloudbridge)
    │   └── valkey-glide
    └── cloudbridge (fscloudbridge)

Lambda (valkey_queue_processor)
    ├── bedrock.throttling
    ├── Lambda Layers
    │   ├── core (bedrock + cloudbridge)
    │   ├── valkey (glide)
    │   └── utils (powertools)
    └── Valkey Lua Script (deployed separately)
```

---

## Detailed Findings

### 1. Discover CLI Application

**Source**: `/Users/jordanmesches/Documents/work/filescience-discover-v2`

#### Project Structure
```
filescience-discover-v2/
├── pyproject.toml          # Poetry, Python >=3.13
├── poetry.toml             # in-project venv
├── src/
│   ├── aws.py              # SSM parameter fetching
│   ├── manager.py          # DiscoveryManager - main orchestrator
│   ├── dispatcherv2.py     # Entity dispatch & persistence
│   ├── queue.py            # WorkQueue with throttle context
│   ├── clouds/
│   │   ├── router.py       # CloudRouter, ServiceRouter base classes
│   │   ├── api/
│   │   │   ├── client.py   # Generic paginating API client
│   │   │   ├── oauth.py    # OAuth2 implementations
│   │   │   └── session.py  # SessionManager
│   │   └── microsoft/
│   │       ├── cloud.py    # O365Router instance
│   │       ├── api/client.py  # GraphClient
│   │       └── services/onedrive.py  # OneDrive handlers
├── sandbox.py              # Main entry point
└── test.py, test2.py       # Rate limiting tests
```

#### Dependencies (pyproject.toml)
```toml
python = ">=3.13,<3.14"
httpx[brotli,zstd] = "^0.28.1"
httpx-retries = "^0.4.5"
anyio = "^4.11.0"
uvloop = "^0.22.1"
boto3 = "^1.40.46"
pydantic = "^2.12.4"
orjson = "^3.11.5"
svix-ksuid = "^0.6.2"
python-ulid = "^3.1.0"

# Git dependencies
fscloudbridge = {git = "github.com/Cyber-Guardian/filescience-cloudbridge", branch = "v2"}
fsbedrock = {git = "github.com/Cyber-Guardian/filescience-bedrock", branch = "main"}
```

#### Core Components

**DiscoveryManager** (`src/manager.py:40-149`):
- Async context manager owning DynamoDB clients, WorkQueue, DispatcherV2
- Glide client connects to `localhost:6380` with TLS
- `run()` loop: leases work, spawns tasks, 10s timeout with in-flight tracking
- `wait_for_tree_ready()` polls until throttle tree has available contexts

**WorkQueue** (`src/queue.py:30-437`):
- `enqueue()`: SHA256 work_id, conditional PutItem for idempotency
- `lease_work()`: Gets optimal throttle context from Valkey, queries DynamoDB GSI
- `finish_work()`: Transaction to convert to tombstone, decrement ref_count
- `try_prune_throttle_context()`: Conditional delete with fencing token

**DispatcherV2** (`src/dispatcherv2.py:30-114`):
- Routes entities to cloud-specific routers
- Saves Resources/Artifacts/DeltaState to DynamoDB
- Enqueues child resources with throttle context

**Cloud Routers** (`src/clouds/router.py`):
- `CloudRouter`: Caches backup state, determines initial vs delta
- `ServiceRouter`: Registers handlers via decorator pattern
- Handlers yield Resource/Artifact/DeltaState items

#### Polylith Mapping
| Current | Polylith Target |
|---------|-----------------|
| `src/manager.py`, `dispatcherv2.py`, `queue.py` | `components/discover/` |
| `src/clouds/` | `components/clouds/` or `components/cloud_microsoft/` |
| `src/aws.py` | `components/aws_utils/` |
| `sandbox.py` | `projects/discover-cli/` |

---

### 2. Bedrock Library

**Source**: `/Users/jordanmesches/Documents/work/filescience/bedrock`

#### Project Structure
```
bedrock/
├── pyproject.toml
└── fsbedrock/
    ├── config.py           # Settings, Config singleton, AWS integration
    ├── auth.py             # AuthProvider for cloud sessions
    ├── domain/
    │   ├── base.py
    │   └── backup/models.py  # Node, Entity, Resource, Artifact, DeltaState
    ├── persistence/
    │   ├── repository.py   # Abstract Repository pattern
    │   ├── sqlalchemy/     # SQLAlchemy implementation
    │   └── dynamodb/       # DynamoDB with BiDirectionalQueryExecutor
    ├── services/
    │   ├── jobs.py         # Step Functions job orchestration
    │   ├── tenants.py      # Tenant management
    │   └── stripe.py       # Billing integration
    └── throttling/
        ├── models.py       # RateGateSpec, GateCharge, AdmissionResult, RequestPolicy
        ├── service.py      # ThrottleService (sync)
        ├── async_service.py
        └── valkey/
            ├── client.py   # create_valkey_client, create_async_valkey_client
            └── functions.py  # Valkey Lua function wrappers
```

#### Dependencies (pyproject.toml)
```toml
python = ">=3.12,<=3.15"
boto3, aioboto3 = "^15.5.0"
sqlalchemy, psycopg2-binary
pydantic, pydantic-settings
cryptography, requests
stripe
valkey-glide-sync, valkey-glide

# Internal
fscloudbridge = {git = "github.com/Cyber-Guardian/filescience-cloudbridge", branch = "v2"}
```

#### Key Exports

**Configuration**:
- `Config` singleton with database connection pooling
- `Settings` Pydantic model for environment variables
- `get_secret_from_aws()`, `discover_service_url()`

**Persistence - DynamoDB**:
- `query_table()`, `aquery_table()` - Sync/async iterators
- `BiDirectionalQueryExecutor` - 18% speedup on large tables
- `PydanticNodeSerializer` - Aggregates IDENTITY+LINK rows

**Throttling**:
- `ThrottleService.register_tree()`, `register_gate()`, `register_node()`
- `request_capacity_by_context()`, `penalize_by_context()`
- `M365ThrottleContext`, `OnedriveTraversalPolicy`

**Domain Models** (`domain/backup/models.py`):
- `Node` base with UUID5 id from `{tenant_id}:{cloud_external_id}`
- `Entity` (users/groups), `Resource` (containers), `Artifact` (files)
- `DeltaState` for delta sync tracking

#### Polylith Mapping
| Current | Polylith Target |
|---------|-----------------|
| `fsbedrock/config.py` | `components/config/` |
| `fsbedrock/auth.py` | `components/auth/` |
| `fsbedrock/persistence/dynamodb/` | `components/dynamodb/` |
| `fsbedrock/persistence/sqlalchemy/` | `components/sqlalchemy/` |
| `fsbedrock/throttling/` | `components/throttling/` |
| `fsbedrock/domain/backup/` | `components/domain_backup/` |
| `fsbedrock/services/` | `components/services/` |

---

### 3. Cloudbridge Library

**Source**: `/Users/jordanmesches/Documents/work/filescience/filescience-cloudbridge`

#### Project Structure
```
filescience-cloudbridge/
├── pyproject.toml
└── fscloudbridge/
    ├── model.py            # Cloud/CloudService definitions
    ├── auth.py             # CloudbridgeAuth wrapper
    ├── client.py           # CloudbridgeClient wrapper
    ├── registry.py         # IntegrationRegistry pattern
    ├── exceptions.py
    └── integrations/
        ├── __init__.py     # Auto-discovery
        ├── base/integration.py  # BaseIntegration ABC
        ├── office_365/     # Microsoft Graph integration
        ├── box/            # Box API
        ├── netdocs/        # NetDocs API
        ├── clio/           # Clio API
        ├── zoom/           # Zoom API
        ├── google/         # GSuite (incomplete)
        └── smartsheet/     # Smartsheet (incomplete)
```

#### Dependencies
```toml
jwt>=1.3.1
requests>=2.32.0
boto3  # For SSM parameter store
```

#### Key Patterns

**Cloud Model System** (`model.py`):
- `CloudService` with class-level registry
- `Cloud.OFFICE_365`, `Cloud.BOX`, etc. with service enums
- `Cloud.from_id(1)` returns Office365 cloud

**Registry Pattern** (`registry.py`):
- `@IntegrationRegistry.register(Cloud.BOX)` decorator
- `IntegrationRegistry.get(Cloud.BOX)` retrieves implementation

**BaseIntegration** (`integrations/base/integration.py`):
- OAuth state generation/validation
- AWS Parameter Store for credentials (`/clouds/{name}/keys`)
- Session management with retry adapter

**Office 365 Integration** (`integrations/office_365/`):
- Admin consent URL at `login.microsoftonline.com`
- Client credentials flow (not auth code)
- Entity retrieval: users, groups, sites from Graph API
- Usage reporting via CSV reports

#### Polylith Mapping
| Current | Polylith Target |
|---------|-----------------|
| `fscloudbridge/model.py` | `components/cloud_model/` |
| `fscloudbridge/registry.py` | `components/cloud_registry/` |
| `fscloudbridge/integrations/base/` | `components/cloud_base/` |
| `fscloudbridge/integrations/office_365/` | `components/cloud_office365/` |
| `fscloudbridge/integrations/box/` | `components/cloud_box/` |
| (etc. for each integration) | |

---

### 4. Valkey Throttle Service Lua Script

**Source**: `/Users/jordanmesches/Documents/work/filescience/filescience-infrastructure/aws/application/valkey/throttle_service.lua`

#### Registered Functions (14 total)
```lua
-- Limiter Operations
check_limiter                    -- Check/consume single limiter
register_limiter                 -- Register rate limiter config
get_limiter_usage_score          -- Get usage-based score
check_limiter_hierarchy          -- Check/consume limiter chain
ensure_and_check_limiter_hierarchy  -- Register missing + check
calculate_limiter_hierarchy_penalty -- Calculate penalty on deny

-- Tree Operations
register_tree                    -- Create throttle index tree
register_node                    -- Add node (branch or leaf)
deregister_node                  -- Remove node recursively
deregister_tree                  -- Remove entire tree

-- Context Operations
get_optimal_throttle_context     -- Find best leaf by score
get_leaf_penalty_score           -- Get penalty score
update_hierarchy_scores          -- Update scores after success
propagate_penalty_scores         -- Propagate penalty up tree
try_prune_throttle_context       -- Prune with fencing token
penalize_throttle_context        -- Penalize with fencing token
```

#### Key Data Structures

**Global Limiter Keys** (`{global}` hash tag):
- `{global}:limiter:{limiter_id}` (HASH) - rate_limit, window_size, counter_window_size
- `{global}:counters:{limiter_id}` (ZSET) - counter keys by start time
- `{global}:counter:{limiter_id}:{counter_id}` (HASH) - count, end time
- `{global}:penalty:{limiter_id}` (HASH) - backoff count, last_penalty

**Tree Keys** (`{tree_id}` hash tag):
- `{tree_id}:meta` (HASH) - created timestamp
- `{tree_id}:node:{node_id}` (HASH) - parent, throttle_context_id
- `{tree_id}:node:{node_id}:children` (SET) - child node IDs
- `{tree_id}:scores` (ZSET) - all node scores for traversal
- `{tree_id}:throttle_context:{id}` (HASH) - metadata including fencing_token

#### Algorithms

**Sliding Window Rate Limiting**:
- Counters with configurable window (e.g., counter_window=1min, window=10min)
- Weighted overlap calculation for partial counters
- Score = `now - (window_size - time_penalty)` (lower = available sooner)

**Exponential Backoff Penalty**:
- `penalty_duration = min(base * 2^(count-1), window_size)` where base = window_size/8
- Diminishing returns: `scale_factor = 1 / (1 + log(1 + existing_penalty / base_unit))`
- Count decays by half if last penalty > window_size ago

**Tree Traversal**:
- Scores represent "next available at" timestamps
- Lower score = higher priority
- Parent score = `max(own_score, best_child_score)`

#### Polylith Mapping
| Current | Polylith Target |
|---------|-----------------|
| `throttle_service.lua` | `infrastructure/modules/valkey/` or `components/throttle_lua/` (as resource) |

---

### 5. Valkey Queue Processor Lambda

**Source**: `/Users/jordanmesches/Documents/work/filescience/filescience-infrastructure/aws/application/lambdas/functions/standard/valkey_queue_processor`

#### Structure
```
valkey_queue_processor/
├── config.toml             # Lambda config (layers, triggers, env)
├── lambda_function.py      # Entry point with batch processing
├── config.py               # Environment variables
├── constants.py            # Key patterns, prefixes
├── keys.py                 # Key helpers, record type identification
├── utils.py
├── models/
│   ├── schema.py           # Pydantic models (SubmittedWorkItem, etc.)
│   └── policy_types.py     # RequestPolicyType enum
├── handlers/
│   ├── submitted_work_item_handler.py
│   └── throttle_context_metadata_handler.py
├── services/
│   ├── throttle_service.py    # ThrottleRegistrationService
│   ├── work_registration_service.py
│   └── pruning_service.py
├── policies/
│   └── registry.py         # build_throttle_context()
└── clients/
    └── dynamodb_client.py  # Singleton clients
```

#### Configuration (`config.toml`)
```toml
name = "valkey-queue-processor"
handler = "lambda_function.lambda_handler"
timeout = 120
memory_size = 512
runtime = "PYTHON_3_12"
layers = ["valkey", "utils", "core"]

[trigger]
type = "dynamodb"
table_name = "{infra_prefix}-queue"
batch_size = 10
report_batch_item_failures = true
```

#### Record Types Processed
1. **SUBMITTED_WORK_ITEM**: Has `request_policy_type`, needs throttle context registration
2. **THROTTLE_CONTEXT_METADATA**: Ref counting, triggers pruning when ref_count=0
3. **ENQUEUED_WORK_ITEM**: Already processed, skip
4. **TOMBSTONE**: TTL-only record, skip

#### Data Flow
```
DynamoDB Stream → Lambda Handler
  ├── SUBMITTED_WORK_ITEM:
  │   ├── Parse via Pydantic
  │   ├── Build throttle context (M365ThrottleContext)
  │   ├── Generate fencing token (ULID)
  │   ├── Register in Valkey (tree, limiters, nodes)
  │   └── Update DynamoDB (remove policy_type, add throttle_context_id, increment ref_count)
  │
  └── THROTTLE_CONTEXT_METADATA (ref_count=0):
      ├── Get tree_id from Valkey
      ├── Conditional delete from DynamoDB (validate fencing_token)
      └── Prune from Valkey tree
```

#### Dependencies
- `aws-lambda-powertools` (batch processing, idempotency, logging)
- `fsbedrock.throttling` (ThrottleService, M365ThrottleContext)
- `valkey-glide` (GlideClient)
- `pydantic` (validation)

---

### 6. Lambda Layer System

**Layer Bundler**: `/Users/jordanmesches/Documents/work/filescience/filescience-infrastructure/aws/cdk/domains/backup/compute/_lambda/layer/infrastructure.py`

**Layer Definitions**: `/Users/jordanmesches/Documents/work/filescience/filescience-infrastructure/aws/application/lambdas/layers/`

#### Available Layers
| Layer | Contents |
|-------|----------|
| `stripe` | stripe==11.6.0 |
| `fastapi` | fastapi, mangum |
| `firebase` | firebase-admin |
| `comms` | slack-sdk, jinja2 |
| `utils` | aws-lambda-powertools, aws-xray-sdk, pydantic, python-ulid |
| `xray` | aws-xray-sdk |
| `valkey` | valkey-glide-sync |
| `core` | SQLAlchemy, boto3, pydantic, fscloudbridge, fsbedrock, lambda_utils |

#### Bundling Process

**PythonDependencyLayer** (simple pip-based):
```bash
pip install --upgrade -r requirements.txt -t /asset-output/python
# Production: python -m compileall -q -j 0 /asset-output/python
```

**PythonCoreLibraryLayer** (git dependencies):
```bash
# Configure git with GitHub token
git config --global url."https://${GITHUB_TOKEN}@github.com/".insteadOf "https://github.com/"
# Copy source code
cp -r python /asset-output/
# Inject token and install
sed -i "s|{GITHUB_TOKEN}|$GITHUB_TOKEN|g" /tmp/requirements.txt
pip install --upgrade -r /tmp/requirements.txt -t /asset-output/python
```

#### Polylith Mapping
| Current | Polylith Target |
|---------|-----------------|
| `layers/core/` | Reference `components/` via workspace deps |
| `layers/utils/` | `components/lambda_utils/` |
| `layers/valkey/` | Already external (valkey-glide) |
| Lambda function code | `projects/valkey-queue-processor/` |

---

## Architecture Documentation

### Current Patterns

1. **Dependency Injection via Git URLs**: Both bedrock and cloudbridge are pulled as git dependencies
2. **Registry Pattern**: IntegrationRegistry for cloud providers, decorator-based handler registration
3. **Repository Pattern**: SQLAlchemy and DynamoDB abstractions in bedrock
4. **Fencing Token Pattern**: ULIDs prevent race conditions in distributed pruning
5. **Hierarchical Rate Limiting**: Tree structure with score propagation for optimal context selection

### Polylith Considerations

**Component Candidates** (reusable across projects):
- `throttling` - Rate limiting service
- `dynamodb` - DynamoDB persistence abstractions
- `cloud_model` - Cloud/service definitions
- `config` - Configuration management
- `domain_backup` - Backup domain models

**Base Candidates** (shared foundations):
- `cloud_base` - BaseIntegration for cloud providers

**Project Candidates** (deployable apps):
- `discover-cli` - CLI entry point with DiscoveryManager
- `valkey-queue-processor` - Lambda function

---

## Code References

### Discover
- Entry point: `/Users/jordanmesches/Documents/work/filescience-discover-v2/sandbox.py:44-103`
- DiscoveryManager: `/Users/jordanmesches/Documents/work/filescience-discover-v2/src/manager.py:40-149`
- WorkQueue: `/Users/jordanmesches/Documents/work/filescience-discover-v2/src/queue.py:30-437`

### Bedrock
- Config singleton: `/Users/jordanmesches/Documents/work/filescience/bedrock/fsbedrock/config.py:116-189`
- ThrottleService: `/Users/jordanmesches/Documents/work/filescience/bedrock/fsbedrock/throttling/service.py:47-977`
- Domain models: `/Users/jordanmesches/Documents/work/filescience/bedrock/fsbedrock/domain/backup/models.py:17-99`

### Cloudbridge
- Cloud model: `/Users/jordanmesches/Documents/work/filescience/filescience-cloudbridge/fscloudbridge/model.py:1-137`
- BaseIntegration: `/Users/jordanmesches/Documents/work/filescience/filescience-cloudbridge/fscloudbridge/integrations/base/integration.py:10-159`
- Office365: `/Users/jordanmesches/Documents/work/filescience/filescience-cloudbridge/fscloudbridge/integrations/office_365/integration.py:14-228`

### Valkey
- Lua script: `/Users/jordanmesches/Documents/work/filescience/filescience-infrastructure/aws/application/valkey/throttle_service.lua:1-1038`
- Lambda handler: `/Users/jordanmesches/Documents/work/filescience/filescience-infrastructure/aws/application/lambdas/functions/standard/valkey_queue_processor/lambda_function.py:185-325`

### Layers
- Bundler: `/Users/jordanmesches/Documents/work/filescience/filescience-infrastructure/aws/cdk/domains/backup/compute/_lambda/layer/infrastructure.py:36-176`
- Core layer: `/Users/jordanmesches/Documents/work/filescience/filescience-infrastructure/aws/application/lambdas/layers/core/`

---

## Open Questions

1. **Python Version Alignment**: Discover requires `>=3.13`, bedrock supports `>=3.12,<=3.15`. Monorepo should use 3.13.
2. **Git Dependency Migration**: How to convert git URL dependencies to Polylith workspace references?
3. **Lua Script Deployment**: Should the Lua script be a component resource or stay in infrastructure?
4. **Lambda Layer Strategy**: Replace CDK layer bundling with Polylith workspace packaging?
5. **Infrastructure Boundaries**: How much of the CDK/OpenTofu infrastructure should be in the monorepo vs separate?
