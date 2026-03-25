---
type: state
created: 2026-03-24
status: active
---
# Plan Contract Guide

> How to write typed contracts that mechanically enforce plan quality.

## Why Typed Contracts Beat Prose Checkboxes

Prose checkboxes in plans ("Verify tests pass", "Check file exists") rely on the implementer's judgment and attention. They are:
- **Ambiguous**: "Tests pass" -- which tests? What counts as passing?
- **Forgettable**: Easy to skip when rushing to ship
- **Unverifiable**: A reviewer can't tell if they were actually checked
- **Non-deterministic**: Different implementers interpret them differently

Typed contracts are:
- **Precise**: `file_exists: "src/models.py"` has exactly one meaning
- **Automated**: `mcp__work__run_contracts` runs them mechanically
- **Gated**: `mcp__work__complete_task` refuses to mark tasks done if contracts fail
- **Schema-validated**: Malformed contracts are rejected at write time, not discovered at run time

## The 5 Assertion Types

### 1. `file_exists`
**When to use:** Verify an artifact was created.

```yaml
assertions:
  - type: file_exists
    path: "src/notifications/models.py"
```

**Good for:** New files, generated outputs, migration files.
**Not for:** Checking file content (use `grep_match`).

### 2. `file_lines`
**When to use:** Enforce complexity limits on a file.

```yaml
assertions:
  - type: file_lines
    path: "src/server.py"
    max_lines: 500
```

**Good for:** Keeping modules focused, preventing god files.
**Not for:** Checking that a file has a minimum number of lines.

### 3. `grep_match`
**When to use:** Verify specific content exists in a file.

```yaml
assertions:
  - type: grep_match
    path: "src/models.py"
    pattern: "class Notification"
```

The `pattern` field is a regex. You can use full regex syntax:

```yaml
assertions:
  - type: grep_match
    path: "pyproject.toml"
    pattern: "pyyaml"
  - type: grep_match
    path: ".mcp.json"
    pattern: "work-mcp"
```

**Good for:** Import validation, config presence, class/function definitions.
**Not for:** Complex structural checks (use `command_succeeds`).

### 4. `import_valid`
**When to use:** Verify a Python module is importable.

```yaml
assertions:
  - type: import_valid
    module: "work_protocol.graph"
```

This runs `python -c "import work_protocol.graph"` under the hood. It respects the MCP server's Python environment.

**Good for:** Verifying new modules are properly structured.
**Not for:** Non-Python imports or complex import chains.

### 5. `command_succeeds`
**When to use:** Escape hatch for anything not covered by the other types.

```yaml
assertions:
  - type: command_succeeds
    name: "lint passes"
    command: "ruff check src/"
    timeout: 30
```

**Good for:** Running test suites, linters, type checkers, custom validation scripts.
**Use sparingly:** Prefer the specific assertion types when possible -- they give better error messages.

### Shell Contracts
In addition to assertions, you can define named shell commands:

```yaml
shell:
  - name: "tests pass"
    command: "make test-fast"
    timeout: 60
  - name: "imports work"
    command: "uv run --project .work/lib python -c \"from work_protocol.manifest import PlanManifest; print('ok')\""
```

Shell contracts are similar to `command_succeeds` assertions but are listed separately in the contract set. Use shell contracts for complex multi-step verification commands.

## How to Write Good Contracts

### Prefer `make` targets
```yaml
# Good: uses existing infrastructure
shell:
  - name: "tests pass"
    command: "make test-fast"

# Avoid: fragile, duplicates existing targets
shell:
  - name: "tests pass"
    command: "cd .work/lib && uv run pytest tests/ -v"
```

### Test behaviors, not existence
```yaml
# Good: tests that the module actually works
assertions:
  - type: import_valid
    module: "work_protocol.graph"

# Less useful: only checks the file is there
assertions:
  - type: file_exists
    path: ".work/lib/work_protocol/graph.py"
```

Use `file_exists` for generated artifacts and config files. Use `import_valid` or `grep_match` for code.

### Be specific with grep patterns
```yaml
# Good: checks for the specific class
assertions:
  - type: grep_match
    path: "src/models.py"
    pattern: "class NotificationModel"

# Too broad: matches any mention of "Notification"
assertions:
  - type: grep_match
    path: "src/models.py"
    pattern: "Notification"
```

### Set realistic timeouts
```yaml
# Good: generous timeout for test suite
shell:
  - name: "full test suite"
    command: "make test"
    timeout: 120

# Bad: default 30s may not be enough
shell:
  - name: "full test suite"
    command: "make test"
```

## Good vs Bad Contract Examples

### Good: Comprehensive verification for a new module
```yaml
shell:
  - name: "graph module importable"
    command: "uv run --project .work/lib python -c \"from work_protocol.graph import build_plan_graph; print('ok')\""
  - name: "all tests pass"
    command: "uv run --project .work/lib pytest .work/lib/work_protocol/tests/ -v --tb=short"

assertions:
  - type: file_exists
    path: ".work/lib/work_protocol/graph.py"
  - type: file_exists
    path: ".work/lib/work_protocol/tests/test_graph.py"
  - type: grep_match
    path: ".work/lib/work_protocol/graph.py"
    pattern: "def build_plan_graph"
  - type: file_lines
    path: ".work/lib/work_protocol/graph.py"
    max_lines: 300
```

### Bad: Vague, incomplete, or redundant
```yaml
# Too vague -- what does "check" mean?
shell:
  - name: "check it works"
    command: "python check.py"

# Redundant -- file_exists is already implied by grep_match
assertions:
  - type: file_exists
    path: "src/models.py"
  - type: grep_match
    path: "src/models.py"
    pattern: "class Model"

# Missing timeout for slow command
shell:
  - name: "integration tests"
    command: "make test-integration"
    # timeout defaults to 30s -- probably too short
```

## The Circuit Breaker Pattern

If contracts cannot be satisfied after a genuine implementation effort, this is a signal that the plan needs reshaping:

1. **First attempt:** Implement the phase, run contracts, fix failures
2. **Second attempt:** If contracts still fail, check if the plan's assumptions are wrong
3. **Circuit breaker:** If the plan's assumptions are wrong, STOP. Do not force contracts to pass by weakening them. Instead:
   - Run `mcp__work__reassess(plan, phase, "Contracts cannot be satisfied because...")` to document the issue
   - Return to `/shape` to re-examine the approach
   - Create a new plan that supersedes the current one

The point of contracts is to catch problems early. Weakening contracts to make them pass defeats the purpose.

## How Graph Edges Work

Plan relationships are declared in `graph.yaml`:

- **`depends_on`**: "I need plan X to be done before I can start." Use when your plan consumes artifacts from another plan.
- **`enables`**: "Once I'm done, plan Y becomes possible." Use when your plan creates infrastructure or APIs that another plan will build on.
- **`supersedes`**: "I replace plan X." Use when you've reshaped or rewritten a previous plan.

Graph edges are validated during `freeze_plan` -- if you reference a plan that doesn't exist, the freeze will fail. This prevents dangling references.

Use `mcp__work__get_graph()` to visualize all plan relationships as a Mermaid diagram. Orphan plans (no edges) and unmet dependencies are highlighted in the summary.
