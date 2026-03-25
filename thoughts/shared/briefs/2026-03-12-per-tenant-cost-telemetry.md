---
type: event
created: 2026-03-12
status: active
---
# Idea Brief: Per-Tenant Cost Telemetry

**Date:** 2026-03-12
**Status:** Shaped → Mapping to Linear

## Problem
FileScience has zero visibility into per-tenant cost. We can't answer "how much does tenant X cost us to serve?" — which means no margin analysis, no pricing optimization, no identification of unprofitable tenants. The OTel infrastructure exists (`components/filescience/telemetry/core.py`) but emits nothing cost-relevant. Previously too laborious to instrument manually — with AI agents and a ground-up v2 architecture, now feasible.

## Constraints
- Internal telemetry only (advise decisions, not customer-facing billing)
- Must work with multi-tenant batch processing (VQP: up to 100 mixed-tenant records per invocation)
- Must handle async interleaving in discover (multiple tenants' work items in the same event loop)
- `tenant_id` (int) already flows through every layer — DynamoDB PKs, throttle gates, session manager
- Polylith boundaries: telemetry is a component, can't import bases — instrumentation hooks must be generic

## Options Considered

### ContextVar + OTel Botocore Instrumentation (chosen)
Use Python `contextvars.ContextVar` to carry `tenant_id` through async call chains. Leverage existing `opentelemetry-instrumentation-botocore` library with custom `request_hook` to tag every AWS API call with tenant. Add `TenantWorkUnit` context manager for compute timing. Periodic jobs for storage metering.
- Gains: Minimal code changes. Extends existing telemetry component. asyncio-native. BotocoreInstrumentor already handles all boto3 plumbing.
- Costs: ContextVar must be set at every entry point. Periodic storage jobs are a separate pattern from inline metering.
- Complexity: Medium overall, Low per-issue

### Explicit Metering Passport
Pass a `TenantMeter` object through every call chain. Accumulates counts/durations, flushes to OTel.
- Gains: Explicit, testable, impossible to forget.
- Costs: Invasive refactor — every function touching AWS needs a meter parameter. Heavy for pass 1.
- Complexity: High refactor cost

### Derive from Existing Signals (post-hoc)
Pull CloudWatch metrics + throttle gate counters. Attribute via batch job, not inline.
- Gains: Zero code changes to services.
- Costs: Lossy — CloudWatch gives Lambda-level, not tenant-level. Valkey counters are transient. Approximate only.
- Complexity: Low

## Chosen Approach
**ContextVar + OTel Botocore Instrumentation** — lowest friction, extends existing infrastructure, asyncio-native. OpenMeter is the upgrade path if this ever needs to drive billing.

## Cost Taxonomy (three distinct patterns)

| Category | Driver | Method | Frequency |
|----------|--------|--------|-----------|
| External API calls | Graph/Clio/Box requests | ContextVar + httpx instrumentation at `cloud_api` `raw_request()` | Inline |
| AWS API calls | DynamoDB RCU/WCU, S3 ops | OTel `BotocoreInstrumentor` + tenant request_hook | Inline |
| Compute | Lambda wall-clock per tenant | `TenantWorkUnit` context manager | Inline |
| DynamoDB storage | GB at rest per tenant | Periodic scan of `TENANT#{id}` partitions | Daily/weekly |
| S3 storage | GB stored per tenant | S3 Inventory or prefix-scoped aggregation | Daily/weekly |

## Natural Decomposition (maps to Linear issues)

1. **Foundation: cost telemetry primitives** — `TenantWorkUnit` context manager, ContextVar, `install_cost_hooks()` in telemetry component. No service changes.
2. **AWS API metering** — `BotocoreInstrumentor` integration with tenant-aware hooks. Wired into `setup_telemetry()`.
3. **External API metering** — instrument `cloud_api.ApiClient.raw_request()` with tenant-scoped counters (httpx, not boto3).
4. **Compute attribution per service** — integrate `TenantWorkUnit` into EDT handler, VQP record_handler, discover work item loop. One issue per service or one issue total.
5. **DynamoDB storage metering** — periodic job scanning tenant partitions, emitting storage gauge metrics.
6. **S3 storage metering** — S3 Inventory setup or prefix-scoped scan job.
7. **Dashboard/reporting** — CloudWatch dashboard or Grafana board showing per-tenant cost breakdown.

## Key Context Discovered During Shaping
- `opentelemetry-instrumentation-botocore` already auto-instruments all boto3 calls and supports `request_hook`/`response_hook` for custom attributes — no need to build our own botocore hooks
- DynamoDB `ConsumedCapacity` is available on every response if `ReturnConsumedCapacity='TOTAL'` is passed — gives actual RCU/WCU per tenant, not just operation counts
- Python `contextvars` + asyncio: each Task inherits creator's context automatically — discover's async fan-out works without explicit propagation
- AWS SaaS Lens recommends tiered precision: start with approximation, add detail where it matters
- OpenMeter (Apache 2.0) has OTel collector — same events can flow to billing system later without re-instrumentation
- Three services have different attribution profiles: EDT (trivial, single-tenant), VQP (straightforward, sequential per-record), Discover (hardest, async interleaving — but contextvars handles it)

## Research Sources
- [[2026-02-12-opentelemetry-cloudwatch-prometheus-grafana-integration|OTel Integration Strategy]]
- AWS SaaS Lens: https://wa.aws.amazon.com/saas.question.COST_1.en.html
- OTel Botocore Instrumentation: https://opentelemetry-python-contrib.readthedocs.io/en/latest/instrumentation/botocore/botocore.html
- OpenMeter: https://openmeter.io/

## Next Step
Map to Linear as a project or milestone under V2 Platform. Issues decompose naturally into the 7 work items above, sequenced as a tiered rollout (inline metering first, storage metering later, dashboard last).
