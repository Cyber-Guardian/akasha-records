# OpenTelemetry Integration Strategy for FileScience (CloudWatch + Prometheus + Grafana)

Date: 2026-02-12
Type: Reference
Scope: Monorepo-wide observability strategy (projects, bases, components, infra)

## Why this document

Define an optimal, practical path to introduce OpenTelemetry (OTel) across all current FileScience runtimes while keeping AWS-first operability and preserving a clean path to Prometheus/Grafana.

## What exists today (repo-specific baseline)

- Runtimes:
  - Lambda: `entity-discovery-trigger`
  - Lambda: `valkey-queue-processor`
  - Local/CLI runner: `discover`
- Logging:
  - Lambda handlers use AWS Lambda Powertools logger (structured JSON).
  - Discover flow mostly uses stdlib logging.
- Tracing:
  - `valkey-queue-processor` has Powertools tracer + X-Ray SDK usage.
  - `entity-discovery-trigger` has no tracing instrumentation.
- Metrics:
  - No backend metrics pipeline for app SLO metrics yet.
  - Discover has in-process run metrics logged to stdout (not exported).
- Infra:
  - CloudWatch log groups exist.
  - No explicit Lambda `tracing_config` in shared Terraform module.
  - No dashboards/alarms for app-level golden signals.
  - No ADOT collector config, no AMP, no Grafana resources in infra.

## Design goals

1. Keep AWS operations simple for current Lambda-heavy topology.
2. Adopt vendor-neutral OTel semantic conventions and context propagation now.
3. Avoid lock-in while minimizing migration risk from existing Powertools/X-Ray usage.
4. Establish one telemetry contract for all projects (service naming, attributes, span/metric naming, trace correlation in logs).
5. Make Prometheus/Grafana optional add-ons, not a forced day-1 dependency.

## Recommended target architecture

Recommended long-term shape: **Hybrid OTel with AWS-first default**.

- Instrumentation in code: OTel SDK APIs and semantic conventions.
- Export path:
  - Primary (AWS ops): traces -> X-Ray path, metrics -> CloudWatch/EMF-compatible path, logs -> CloudWatch Logs.
  - Optional analytics/advanced observability: metrics -> Prometheus remote write (AMP/self-hosted), traces -> Tempo/OTLP backend, logs -> Loki (if adopted).
- Visualization:
  - Short-term: CloudWatch dashboards + alarms.
  - Optional: Grafana dashboards reading from CloudWatch, AMP/Prometheus, Tempo, Loki.

This lets teams start with AWS-native reliability while keeping open standards and multi-backend flexibility.

## Backend choice analysis

### Option A: AWS-native only (CloudWatch + X-Ray)

Pros:
- Lowest operational overhead in current Lambda-centric environment.
- Native IAM, retention, alarms, and incident workflow.
- Fastest rollout with least infra change.

Cons:
- Less backend portability.
- Trace/metric analysis flexibility is lower than dedicated Prom/Grafana stacks.

Best when:
- Team wants fast, low-risk adoption first.

### Option B: Prometheus/Grafana first (OTLP -> Collector/Alloy -> Prom/Tempo/Loki)

Pros:
- Strong unified visualization and cross-signal correlation.
- Portable and backend-agnostic from day 1.

Cons:
- Higher ops complexity (collector pipelines, auth, storage, cardinality management).
- More moving parts before immediate business value.

Best when:
- Observability team maturity and dedicated platform ownership already exist.

### Option C (recommended): Hybrid staged model

Pros:
- Delivers immediate AWS value with a clean path to Grafana ecosystem.
- Supports both cost/operational control and future portability.
- Enables phased rollout without high migration risk.

Cons:
- Requires discipline to prevent drift between AWS and Grafana dashboards.

## OpenTelemetry standards to define once (applies to all projects)

- Resource attributes:
  - `service.name`
  - `service.namespace=filescience`
  - `deployment.environment` (`dev`, `staging`, `prod`)
  - `cloud.provider=aws`
  - `cloud.region`
- Semantic conventions:
  - HTTP, AWS SDK, messaging/db semantics where applicable.
- Trace propagation:
  - W3C `traceparent` + `baggage`.
- Log correlation:
  - Every structured log includes trace/span identifiers.
- Metric guardrails:
  - Hard rules for low-cardinality dimensions.
  - Business dimensions by whitelist only (tenant-level cardinality must be controlled).

## Integration blueprint by project

### 1) `entity-discovery-trigger` (Lambda)

Near-term:
- Add tracing instrumentation (currently absent).
- Ensure explicit service/resource attributes.
- Emit core custom metrics (`invocations`, `errors`, `duration`, queue enqueue outcomes).
- Preserve CloudWatch structured logs with trace correlation.

Mid-term:
- Transition from mixed Powertools-only tracing to OTel-aligned tracing API usage.

### 2) `valkey-queue-processor` (Lambda)

Near-term:
- Normalize service naming/env attributes.
- Add explicit golden-signal metrics:
  - processed records
  - failed records
  - retry attempts
  - age/latency histograms where practical
- Keep existing failure-capture path; add trace IDs in error records.

Mid-term:
- Replace or wrap current tracer usage with OTel-consistent spans.
- Add span links from producer -> consumer boundaries where context exists.

### 3) `discover` (local/sandbox worker)

Near-term:
- Add OTel SDK for local execution spans and metrics export toggle.
- Correlate queue operations and service calls with trace context.
- Keep stdout logging; attach trace fields for local debugging and CI runs.

Mid-term:
- Optional OTLP export to shared collector/gateway for integration tests.

### 4) Shared components/bases

- Introduce telemetry helper module (single import surface) for:
  - tracer/meter/logger acquisition
  - resource attribute defaults
  - safe metric wrappers
  - propagation helpers
- Enforce conventions in one place to avoid drift.

## CloudWatch / Prometheus / Grafana tie-in model

### CloudWatch + X-Ray path

- Logs:
  - Lambda structured logs remain in CloudWatch Logs.
- Traces:
  - Use X-Ray-compatible export path for AWS operations and service map continuity.
- Metrics:
  - CloudWatch metrics for operational alarms and SLO guardrails.
- Alerting:
  - CloudWatch alarms + SNS/Pager integration.

### Prometheus path

- Metrics exported via OTel Collector pipeline using Prometheus remote write.
- Prefer AMP for managed ingestion in AWS.
- Enforce strict label cardinality policy before enabling broad export.

### Grafana path

- Grafana as visualization/control plane over:
  - CloudWatch datasource (immediate value)
  - AMP/Prometheus datasource (metric analytics)
  - Tempo datasource (distributed tracing)
  - Loki datasource (if centralized log pipeline is adopted)
- Grafana Alloy can act as an OTel collector distribution for OTLP intake and routing to Tempo/Prometheus/Loki.

## Recommended phased rollout (optimal for current repo)

### Phase 0: Foundation (1-2 days)

- Define telemetry convention document (resource attrs, metric names, span names, cardinality policy).
- Add shared telemetry helper in a component (no-op safe if exporter disabled).
- Enable explicit Lambda tracing configuration in Terraform module.

Exit criteria:
- Both Lambda projects have consistent service naming + environment/resource tags.

### Phase 1: AWS-first instrumentation (2-4 days)

- Instrument `entity-discovery-trigger` tracing + metrics.
- Normalize and complete `valkey-queue-processor` metrics + trace correlation.
- Add CloudWatch dashboards and alarms for:
  - errors
  - throttles
  - latency/duration
  - stream iterator age / backlog proxies

Exit criteria:
- On-call can diagnose failures from AWS-native dashboards alone.

### Phase 2: OTel portability hardening (2-5 days)

- Introduce OTel exporter configuration abstraction (env-driven).
- Validate OTLP-compatible pipeline in dev (without replacing AWS path yet).
- Add integration checks for trace propagation across async boundaries.

Exit criteria:
- Same telemetry contract works whether backend is AWS-native or OTLP endpoint.

### Phase 3: Prometheus/Grafana expansion (optional)

- Add collector/alloy pipeline:
  - OTLP receiver
  - batch + memory limiter processors
  - exporters to Prometheus remote write and tracing backend
- Bring up Grafana dashboards for cross-signal views.

Exit criteria:
- Grafana dashboards match AWS dashboards for core golden signals.

## Critical implementation details and guardrails

- Keep logs in CloudWatch during early phases even if metrics/traces route elsewhere.
- Avoid high-cardinality labels:
  - tenant IDs, resource IDs, user IDs should not be default metric labels.
- Use sampling strategy explicitly:
  - low in high-volume steady-state
  - elevated in incidents/debug windows.
- Ensure async context handling:
  - queue handoff should preserve trace relationship (propagation or span links).
- Treat telemetry schema as a versioned contract:
  - changes require review and dashboard updates.

## What to avoid

- Big-bang migration from Powertools/X-Ray straight to full LGTM stack.
- Shipping metrics without cardinality policy.
- Enabling multiple exporters in prod before cost/volume baselines are known.
- Creating separate telemetry patterns per project.

## Minimal decision set needed now

1. Confirm **hybrid staged strategy** as default.
2. Decide whether Phase 1 remains fully AWS-native for prod while OTLP path is validated only in dev.
3. Approve canonical metric/trace naming and dimension policy.
4. Choose Grafana target model (AWS Managed Grafana vs self-managed Grafana/Cloud).

## Source basis

- Codebase observability map (repo scan, 2026-02-12).
- OpenTelemetry documentation via Context7 (`/websites/opentelemetry_io`).
- OpenTelemetry Collector Contrib documentation via Context7 (`/open-telemetry/opentelemetry-collector-contrib`).
- Grafana Alloy documentation via Context7 (`/grafana/alloy`).
