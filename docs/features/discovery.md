# Feature Spec: Discovery

**Feature ID**: FE-04
**Status**: Ready for Implementation
**Priority**: P1
**Parent**: [Tech Design v2.0](../tech-design.md) Section 8.5
**SRS Requirements**: FR-DISC-001, FR-DISC-002, FR-DISC-003, FR-DISC-004

---

## 1. Description

The Discovery component provides `apcore-cli list` and `apcore-cli describe` subcommands for browsing available modules in the Registry. It supports tag-based filtering (AND logic), TTY-adaptive output format selection (table vs JSON), and rich terminal rendering with syntax-highlighted JSON schemas.

---

## 2. Requirements Traceability

| Req ID | SRS Ref | Description |
|--------|---------|-------------|
| FR-04-01 | FR-DISC-001 | `list` subcommand displaying modules as formatted table. |
| FR-04-02 | FR-DISC-003 | `describe` subcommand displaying full module metadata. |
| FR-04-03 | FR-DISC-002 | Tag filtering with AND logic on `list`. |
| FR-04-04 | FR-DISC-003 | Syntax-highlighted JSON schemas in `describe` output. |
| FR-04-05 | FR-DISC-004 | `--format` flag with TTY-adaptive defaults. |

---

## 3. Module Path

`apcore_cli/discovery.py`

---

## 4. Implementation Details

### 4.1 Command: `list_cmd`

**Registration**: `@cli.command("list")`

**Signature**: `list_cmd(tag: tuple[str, ...], format: str | None) -> None`

**Click decorators**:
```python
@click.option("--tag", multiple=True, help="Filter modules by tag (AND logic). Repeatable.")
@click.option("--format", "output_format", type=click.Choice(["table", "json"]),
              default=None, help="Output format. Default: table (TTY) or json (non-TTY).")
```

Logic steps:
1. Call `registry.list()` to get all module definitions.
2. If `tag` tuple is non-empty:
   a. Convert to set: `filter_tags = set(tag)`.
   b. Filter: `modules = [m for m in modules if filter_tags.issubset(set(m.tags))]`.
3. Resolve output format:
   a. If `output_format` is not None: use it.
   b. Else if `sys.stdout.isatty()`: use `"table"`.
   c. Else: use `"json"`.
4. If format is `"table"`:
   a. Create `rich.table.Table` with columns: "ID", "Description", "Tags".
   b. For each module:
      - `description`: truncate to 80 chars + `"..."` if longer.
      - `tags`: `", ".join(module.tags)`.
      - Add row.
   c. If no modules and tags were specified: display "No modules found matching tags: {tags}.".
   d. If no modules and no tags: display "No modules found.".
   e. Print table via `rich.console.Console().print(table)`.
5. If format is `"json"`:
   a. Build list of dicts: `[{"id": m.canonical_id, "description": m.description, "tags": m.tags} for m in modules]`.
   b. Print `json.dumps(result, indent=2)` to stdout.
6. Exit code 0.

### 4.2 Command: `describe_cmd`

**Registration**: `@cli.command("describe")`

**Signature**: `describe_cmd(module_id: str, output_format: str | None) -> None`

**Click decorators**:
```python
@click.argument("module_id")
@click.option("--format", "output_format", type=click.Choice(["table", "json"]),
              default=None, help="Output format. Default: table (TTY) or json (non-TTY).")
```

Logic steps:
1. Call `validate_module_id(module_id)`. On failure: exit 2.
2. Call `module_def = registry.get_definition(module_id)`.
3. If `module_def is None`: write to stderr "Error: Module '{module_id}' not found." Exit 44.
4. Access schemas from `module_def.input_schema` and `module_def.output_schema`.
5. Resolve output format (same logic as `list_cmd`).
6. If format is `"table"`:
   a. Print section header "Module: {module_id}" with `rich.panel.Panel`.
   b. Print "Description:" followed by full `module_def.description`.
   c. If `module_def.input_schema` is not None:
      - Print "Input Schema:" header.
      - Print `rich.syntax.Syntax(json.dumps(input_schema, indent=2), "json")`.
   d. If `module_def.output_schema` is not None:
      - Print "Output Schema:" header.
      - Print `rich.syntax.Syntax(json.dumps(output_schema, indent=2), "json")`.
   e. If `module_def.annotations` is not None and non-empty:
      - Print "Annotations:" header.
      - For each `(key, value)` in annotations: print `"  {key}: {value}"`.
   f. If any `x-` prefixed keys in module metadata:
      - Print "Extension Metadata:" header.
      - For each `x-` key: print `"  {key}: {value}"`.
   g. Print "Tags:" followed by `", ".join(module_def.tags)`.
7. If format is `"json"`:
   a. Build dict with all fields: `id`, `description`, `input_schema`, `output_schema`, `annotations`, `tags`, and all `x-` fields.
   b. Omit keys with `None` values.
   c. Print `json.dumps(result, indent=2)`.
8. Exit code 0.

### 4.3 Description Truncation Helper

**Function**: `_truncate(text: str, max_length: int = 80) -> str`

```python
def _truncate(text: str, max_length: int = 80) -> str:
    if len(text) <= max_length:
        return text
    return text[:max_length - 3] + "..."
```

---

## 5. Parameter Validation

| Parameter | Type | Valid Values | Invalid Handling | SRS Reference |
|-----------|------|-------------|------------------|---------------|
| `--tag` | `str` (multiple) | Each: `^[a-z][a-z0-9_-]*$` | Invalid format: exit 2. Non-existent tag: empty result (not error). | FR-DISC-002 |
| `--format` | `click.Choice` | `"table"`, `"json"` | Other values: Click rejects, exit 2. | FR-DISC-004 |
| `module_id` (describe) | `str` | Canonical ID regex, max 128 chars. | Invalid format: exit 2. Not found: exit 44. | FR-DISC-003 |

---

## 6. Output Format Examples

### 6.1 Table Output (`list`)

```
+----------------+--------------------------+------------+
| ID             | Description              | Tags       |
+----------------+--------------------------+------------+
| math.add       | Add two numbers.         | math, core |
| text.summarize | Summarize a text docum...| text       |
+----------------+--------------------------+------------+
```

### 6.2 JSON Output (`list`)

```json
[
  {
    "id": "math.add",
    "description": "Add two numbers.",
    "tags": ["math", "core"]
  },
  {
    "id": "text.summarize",
    "description": "Summarize a text document using extractive methods.",
    "tags": ["text"]
  }
]
```

### 6.3 Table Output (`describe`)

```
--- Module: math.add ---

Description:
  Add two numbers together and return the sum.

Input Schema:
  {
    "properties": {
      "a": {"type": "integer", "description": "First operand"},
      "b": {"type": "integer", "description": "Second operand"}
    },
    "required": ["a", "b"]
  }

Annotations:
  readonly: true
  requires_approval: false

Tags: math, core
```

---

## 7. Error Handling

| Condition | Exit Code | Error Message | SRS Reference |
|-----------|-----------|---------------|---------------|
| Module not found (describe) | 44 | "Error: Module '{id}' not found." | FR-DISC-003 AF-1 |
| Invalid format value | 2 | Click auto-generates: "Invalid value for '--format'..." | FR-DISC-004 AF-1 |
| Empty registry (list) | 0 | Table shows "No modules found." or JSON `[]`. | FR-DISC-001 AF-1 |
| No matching tags (list) | 0 | "No modules found matching tags: {tags}." or JSON `[]`. | FR-DISC-002 AF-1 |

---

## 8. Verification

| Test ID | Description | Expected Result |
|---------|-------------|-----------------|
| T-DISC-01 | `apcore-cli list` with 2 modules | Table with both module IDs, descriptions, tags. Exit 0. |
| T-DISC-02 | `apcore-cli list` with empty registry | "No modules found." Exit 0. |
| T-DISC-03 | `apcore-cli list --tag math` | Only modules with `math` tag shown. |
| T-DISC-04 | `apcore-cli list --tag math --tag core` | Only modules with both `math` AND `core` tags. |
| T-DISC-05 | `apcore-cli list --tag nonexistent` | "No modules found matching tags: nonexistent." Exit 0. |
| T-DISC-06 | `apcore-cli list --format json` | Valid JSON array output. |
| T-DISC-07 | `apcore-cli list` in non-TTY (piped) | JSON output by default. |
| T-DISC-08 | `apcore-cli list --format table` in non-TTY | Table output (flag overrides). |
| T-DISC-09 | `apcore-cli describe math.add` | Full metadata with syntax-highlighted schemas. Exit 0. |
| T-DISC-10 | `apcore-cli describe non.existent` | stderr: "not found". Exit 44. |
| T-DISC-11 | `apcore-cli describe math.add --format json` | JSON object output with all metadata. |
| T-DISC-12 | Module with 120-char description in `list` | Description truncated to 80 chars + "...". |
| T-DISC-13 | Module without `output_schema` in `describe` | Output Schema section omitted. |
| T-DISC-14 | Module without annotations in `describe` | Annotations section omitted. |
| T-DISC-15 | `apcore-cli list --format yaml` | Click rejects. Exit 2. |
