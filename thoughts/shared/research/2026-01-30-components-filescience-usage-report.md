# `components/filescience/*` — Usage Report (2026-01-30)

This report answers: **which components under `components/filescience/` are actually imported by runtime code** (bases/projects/other components), and which appear **packaged-but-unused**.

## Component inventory

Components currently present under `components/filescience/`:

- `cloudbridge_base`
- `cloudbridge_box`
- `cloudbridge_clio`
- `cloudbridge_model`
- `cloudbridge_office365`
- `cloudbridge_registry`
- `domain_backup`
- `dynamodb` (placeholder)
- `throttling`
- `config` (directory exists but code deleted; no projects include it anymore)

## “Actually imported” map (static imports)

### Used by bases/

- **`cloudbridge_model`**
  - Imported by the `discover` base (multiple modules under `bases/filescience/discover/...`)
- **`domain_backup`**
  - Imported by the `discover` base (manager/dispatcher/queue/router + microsoft onedrive service)
- **`throttling`**
  - Imported by the `valkey_queue_processor` base (`handler.py`, `services/*`, `policies/registry.py`, `models/__init__.py`)

### Used only by other components/

These do have import usage, but **only within `components/`** (i.e. they are “internal library pieces” not referenced by bases/projects directly):

- **`cloudbridge_base`**
  - Used by `cloudbridge_box`, `cloudbridge_clio`, `cloudbridge_office365`
- **`cloudbridge_registry`**
  - Used by `cloudbridge_base`, `cloudbridge_box`, `cloudbridge_clio`, `cloudbridge_office365`

### Zero imports found outside their own component directory

These components are present, but **nothing outside their own folder imports them directly**:

- **`cloudbridge_box`**
- **`cloudbridge_clio`**
- **`cloudbridge_office365`**

Important nuance: this does *not* mean they’re useless overall — if you import `cloudbridge_registry` and expect it to discover/register provider integrations dynamically, you might still need these provider components present. In *this repo*, I didn’t see runtime code importing them yet.

### Appears unused (no imports found anywhere)

- **`dynamodb`**
  - No `from filescience.dynamodb ...` / `import filescience.dynamodb ...` usages found in runtime code. (There are TODO mentions about migrating DynamoDB utilities, but no imports.)

### `config` component status

- The `filescience.config` component **is not imported anywhere in runtime code**.
- It was previously included as a brick in both projects; we removed those brick mappings and deleted the component’s python files.
- The directory still exists, so Polylith still lists a `config` brick, but it’s **not included by any project**.

## Packaged-by-project vs actually imported

Polylith brick mappings can cause components to be packaged into the project wheel even if never imported.

### Project: `projects/discover`

Included bricks (current mapping): cloudbridge_* bricks + `cloudbridge_model`, `throttling`, `dynamodb`, `domain_backup`, plus base `discover`.

Actually imported (by static imports in `bases/filescience/discover/...`):
- ✅ `cloudbridge_model`
- ✅ `domain_backup`
- ❌ `cloudbridge_base`, `cloudbridge_registry`, `cloudbridge_office365`, `cloudbridge_box`, `cloudbridge_clio` (not imported)
- ⚠️ `throttling` (no imports; there are TODOs suggesting future integration)
- ⚠️ `dynamodb` (no imports; TODOs suggest future migration)

### Project: `projects/valkey-queue-processor`

Included bricks (current mapping): `cloudbridge_model`, `throttling`, `dynamodb`, `domain_backup`, plus base `valkey_queue_processor`.

Actually imported (by static imports in `bases/filescience/valkey_queue_processor/...`):
- ✅ `throttling`
- ❌ `cloudbridge_model`, `dynamodb`, `domain_backup` (not imported)

## Recommended “safe delete” candidates (in order)

These are candidates based on **current static import usage only**:

1. **`components/filescience/dynamodb`**
   - No imports anywhere; keep only if you’re actively planning migration into it.
2. **Provider components**: `cloudbridge_box`, `cloudbridge_clio`, `cloudbridge_office365`
   - No imports outside themselves; likely safe *only if* nothing relies on them via registry-style dynamic discovery.
3. **Over-included bricks in projects**
   - `projects/valkey-queue-processor`: `cloudbridge_model`, `dynamodb`, `domain_backup`
   - `projects/discover`: cloudbridge provider bricks and registry/base (unless you intend to wire them in soon)

## Verification checklist (recommended)

For each deletion/removal:

- Remove brick mapping(s) from all `projects/*/pyproject.toml`
- Delete the component directory
- Run:
  - `uv run poly info` (and `uv run poly check` if you use it)
  - `make build`
  - `uv build --wheel projects/discover -o build/wheels`
  - `uv build --wheel projects/valkey-queue-processor -o build/wheels`
  - Import smoke tests for real entrypoints

