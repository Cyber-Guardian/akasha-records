---
type: event
created: 2026-03-13
status: active
tags: [architecture, v2-platform, integrations, plugin, intermediate-format]
---
# Idea Brief: Plugin Integration Intermediate Format

**Date:** 2026-03-13
**Status:** Shaped → Parked (ENG-2535)

## Problem
Adding a new cloud integration to FileScience requires deep knowledge of platform internals. There is no deliberate contract between "the thing that understands a cloud" (the integration) and "the thing that stores and recovers data" (the platform). V1 had an implicit contract (JSON-serialized Artifact blobs over SQS). V2 has a slightly more formalized version (domain_backup models) but the same leak — an untyped `context: dict[str, Any]` carries cloud-specific routing data bound by convention, not schema. The integration and the platform are developed together, not independently.

## Constraints
- 9 hardcoded seams in the current discover base (session.py, dispatcher.py, queue.py, entities.py, policy registries)
- V1 transfer service consumed Artifacts from SQS with `cloud_specific_metadata: dict` — same untyped bag
- Clouds have fundamentally different data shapes: hierarchical files (OneDrive), threaded messages (Slack), flat records with cross-references (Salesforce), versioned config snapshots (Meraki)
- Recovery fidelity is bounded by cloud write APIs, not read APIs — format must capture more than what's needed for restore
- Pass 1 Foundation Sketch defines bounded contexts (Discovery, Transfer, Recovery, Catalog) that all need to share this contract
- DDD layering (domain/application/infrastructure) planned but not yet executed

## Options Considered

### Typed Tree Contract ("Node Protocol")
Evolve the v1 Resource/Artifact/ArtifactVersion tree into a deliberate contract with typed ContainerNode and ItemNode types, replacing the untyped context dict with explicit content descriptors.
- Gains: Familiar, natural for file-like clouds, proven by Restic/Borg/Kopia
- Costs: Tree assumption breaks for Salesforce cross-references, Gmail multi-label, Slack file cross-posts. Fixed type taxonomy becomes a bottleneck.
- Complexity: Low-Medium

### Self-Describing Catalog ("Declare + Emit")
Airbyte-inspired: integration declares a catalog of streams with JSON Schema, emits records conforming to its own declaration. Platform infers structure from declared relationships.
- Gains: Maximum flexibility, calendar events and Salesforce objects are first-class. Proven at 400+ Airbyte connectors. Schema validation catches bugs at boundary.
- Costs: Platform needs a generic schema interpreter. More upfront design per integration. Schema evolution is a real concern.
- Complexity: Medium-High

### Layered Envelope ("Core + Content + Cloud Facets")
Every piece of cloud data becomes a Record with three facets: core (platform-managed universal fields), content (platform-managed blob reference), cloud (integration-managed typed schema carrying all cloud-specific metadata including recovery fields). Plus a self-describing manifest per integration.
- Gains: Platform gets enough universal structure for browse/search/compliance. Integration gets full freedom in cloud facet. Merges best of Tree (universal browse) and Catalog (self-describing schemas). Proven decomposition — Restic, Borg, Kopia, Druva, Airbyte all converge on this split.
- Costs: Must get the core facet right (what's truly universal). Synthetic containers for non-tree clouds are pragmatic but lossy for primary browse view.
- Complexity: Medium

### Content-Addressed Archive ("Snapshot + Pack")
Restic/Kopia model directly: integration produces immutable snapshots of content-addressed tree objects + blob objects. Platform is a content-addressed object store.
- Gains: Deduplication is free. Immutability gives integrity guarantees. Storage-efficient for incremental backup.
- Costs: Most complex integration developer experience (content-addressing, tree building, blob chunking). Harder for real-time incremental updates. Search/browse requires reading tree objects.
- Complexity: High

## Chosen Approach
**Layered Envelope with Self-Describing Manifest** — three facets (core/content/cloud), integration manifest declaring streams and schemas, strict validation.

### The Record
```
Record
 +-- core      (platform-managed, universal, indexed)
 |    identity:   cloud, service, cloud_external_id, internal_id
 |    structure:  parent_ref, node_type (container|item|version)
 |    metadata:   name, created_at, modified_at, discovered_at, size, content_type, owner_id
 |    lifecycle:  change_type (discovered|modified|deleted)
 |
 +-- content   (platform-managed, optional)
 |    content_hash (SHA256), content_format, content_ref, original_size
 |
 +-- cloud     (integration-managed, typed schema, stored verbatim)
      all cloud-specific metadata + recovery fields in one place
      schema declared in manifest
```

### Key Design Decisions
- **Three facets, not four**: Cloud and recovery facets merged — the integration knows which of its fields are needed for recovery. Platform stores verbatim, returns verbatim.
- **Single parent_ref for browse tree**: Non-tree relationships in cloud facet. Every cloud gets the same browse UX. Richer views are cloud-aware UI features.
- **Synthetic containers for flat-record clouds**: Salesforce gets virtual "Accounts", "Contacts" containers. Platform doesn't know they're synthetic.
- **Items can parent items**: Email -> attachment, message -> thread reply. Relaxes v1 constraint.
- **Strict schema validation**: Reject malformed records. Backup domain: a stored-but-unrestorable record is worse than a rejected one.
- **Content = "the thing you'd restore"**: Files -> bytes, emails -> EML, SF records -> JSON, configs -> YAML. Uniform treatment.
- **Content-addressing internal to platform**: Integration produces records with content refs. Platform hashes, deduplicates, tiers internally.
- **Additive schema evolution**: New cloud facet fields don't require platform changes. Breaking changes require manifest major version bump.

### Integration Developer Experience
1. Read the Record spec + an existing manifest as reference
2. Write a manifest declaring streams, node types, relationships, cloud schemas
3. Implement discovery: call cloud APIs, yield Records with facets filled in
4. Test independently: validate against core schema + declared cloud schema. No platform needed.
5. Ship — platform consumes Records without knowing anything about the cloud

### Grounding Results
- Three-facet split: **Holds** — Restic, Borg, Kopia, Druva, Airbyte all converge on equivalent decomposition
- Single parent_ref: **Partly holds** — Veeam Salesforce preserves native relational model (different philosophy). Synthetic containers are pragmatic, acknowledge it's a UI tradeoff.
- Self-describing manifests: **Holds strongly** — Airbyte (400+ connectors), Terraform (4000+ providers), Grafana data plane contract, Keepit DSL, HYCU low-code platform
- Content-addressing internal: **Holds** — standard practice in Restic/Borg/Kopia, already how v1 transfer works
- Not tried and failed: **Unverified** — no negative evidence. Keepit's DSL connector framework + Merkle tree is conceptually aligned. Commercial vendors don't publish internals.

## Key Context Discovered During Shaping
- V1 discover repo used `BaseProvider` / `BaseService` abstract classes with `cloud_specific_metadata: dict` — same untyped bag pattern persists into v2 as `context: dict`
- V1 transfer stored `transformers` list per version for invertible recovery (COMPRESS -> DECOMPRESS) — this pattern should inform the content facet
- Industry convergence on content/metadata separation is universal (Restic, Borg, Kopia, Druva, Iceberg, OpenMetadata)
- Airbyte's "resilience over strictness" is wrong for backup — strict validation is the right tradeoff
- Keepit uses AI-assisted DSL connector generation — validates that a well-structured manifest enables agent-built integrations
- The Record is the shared language across Discovery, Transfer, Recovery, and Catalog bounded contexts — it IS the ubiquitous language in DDD terms

## Related
- [[2026-03-09-plugin-architecture-and-helm-intelligence|Plugin Architecture Brief (Parked)]] — ENG-2393, focused on pluggability mechanics
- [[2026-03-12-platform-v2-msp-breadth-strategy|MSP Breadth Strategy]] — integration factory concept, CGCG cloud stack priorities
- [[2026-03-12-brain-dump-validated-ideas|Brain Dump Validated Ideas]] — Integration Factory ranked #5
- [[2026-03-10-pass-1-foundation-sketch|Pass 1 Foundation Sketch]] — bounded contexts and port protocols
- [[2026-02-27-ddd-micro-architecture|DDD Micro-Architecture]] — internal layering prerequisite

## Next Step
- [Parked] -> ENG-2535: Design plugin integration intermediate format (Layered Envelope)
- When picked up: `/create_plan` from this brief for intermediate format design + M365 migration as reference implementation
