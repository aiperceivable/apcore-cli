# Feature Spec: Output Formatter

**Feature ID**: FE-08
**Status**: Ready for Implementation
**Priority**: P1
**Parent**: [Tech Design v1.0](../tech-design.md) Section 8.5
**SRS Requirements**: FR-DISC-004

---

## 1. Description

The Output Formatter provides TTY-adaptive output rendering for `apcore-cli`. It detects whether stdout is connected to a terminal or a pipe, and defaults to rich table formatting for TTY sessions or JSON for non-TTY contexts. It is used by Discovery (`list`, `describe`) and can be used by `exec` output formatting.

---

## 2. Requirements Traceability

| Req ID | SRS Ref | Description |
|--------|---------|-------------|
| FR-08-01 | FR-DISC-004 | TTY-adaptive output format selection with `--format` override. |
| FR-08-02 | FR-DISC-001 | Table rendering for module lists using `rich.table.Table`. |
| FR-08-03 | FR-DISC-003 | Syntax-highlighted JSON rendering using `rich.syntax.Syntax`. |

---

## 3. Module Path

`apcore_cli/output.py`

---

## 4. Implementation Details

### 4.1 Function: `resolve_format`

**Signature**: `resolve_format(explicit_format: str | None) -> str`

Logic steps:
1. If `explicit_format is not None`: return `explicit_format`.
2. If `sys.stdout.isatty()`: return `"table"`.
3. Else: return `"json"`.

### 4.2 Function: `format_module_list`

**Signature**: `format_module_list(modules: list[ModuleDefinition], format: str, filter_tags: tuple[str, ...] = ()) -> None`

Logic steps:
1. If `format == "table"`:
   a. Create `table = rich.table.Table(title="Modules")`.
   b. Add columns: `"ID"`, `"Description"`, `"Tags"`.
   c. For each module:
      - `desc = _truncate(module.description, 80)`.
      - `tags = ", ".join(module.tags)`.
      - `table.add_row(module.canonical_id, desc, tags)`.
   d. If no modules and `filter_tags`:
      - Print `f"No modules found matching tags: {', '.join(filter_tags)}."`.
   e. Elif no modules:
      - Print `"No modules found."`.
   f. Print table via `Console().print(table)`.
2. Elif `format == "json"`:
   a. For each module:
      - `_meta = module.metadata or {}`
      - `_disp = _meta.get("display") or {}`
      - `_cli = _disp.get("cli") or {}`
      - `mid = _cli.get("alias") or _disp.get("alias") or module.canonical_id`
      - `desc = _cli.get("description") or module.description`
      - `tags = _disp.get("tags") or module.tags`
      - Append `{"id": mid, "description": desc, "tags": tags}` to result list.
   b. Print `json.dumps(result, indent=2)` via `click.echo`.

### 4.3 Function: `format_module_detail`

**Signature**: `format_module_detail(module_def: ModuleDefinition, format: str) -> None`

Logic steps:
1. If `format == "table"`:
   a. Print `rich.panel.Panel(f"Module: {module_def.canonical_id}")`.
   b. Print `"\nDescription:\n  {module_def.description}\n"`.
   c. If `module_def.input_schema`:
      - Print `"\nInput Schema:"`.
      - Print `rich.syntax.Syntax(json.dumps(module_def.input_schema, indent=2), "json", theme="monokai")`.
   d. If `module_def.output_schema`:
      - Print `"\nOutput Schema:"`.
      - Print `rich.syntax.Syntax(json.dumps(module_def.output_schema, indent=2), "json", theme="monokai")`.
   e. If `module_def.annotations` and non-empty:
      - Print `"\nAnnotations:"`.
      - For each `(k, v)`: print `f"  {k}: {v}"`.
   f. Collect `x_fields = {k: v for k, v in vars(module_def).items() if k.startswith("x_") or k.startswith("x-")}`.
   g. If `x_fields`:
      - Print `"\nExtension Metadata:"`.
      - For each `(k, v)`: print `f"  {k}: {v}"`.
   h. If `module_def.tags`:
      - Print `f"\nTags: {', '.join(module_def.tags)}"`.
2. If `format == "json"`:
   a. Build dict with all non-None fields.
   b. Print `json.dumps(result, indent=2)`.

### 4.4 Function: `format_exec_result`

**Signature**: `format_exec_result(result: Any, format: str | None = None) -> None`

Logic steps:
1. If result is a `dict` or `list`: print `json.dumps(result, indent=2, default=str)`.
2. If result is a `str`: print result directly.
3. If result is `None`: print nothing (empty stdout).
4. Otherwise: print `str(result)`.

### 4.5 Helper: `_truncate`

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
| `explicit_format` | `str \| None` | `"table"`, `"json"`, `None` | `None` triggers TTY detection. Invalid values should be caught by Click upstream. | FR-DISC-004 |
| `modules` | `list` | List of `ModuleDefinition` objects. | Empty list: display "No modules found." message. | FR-DISC-001 |
| `result` (exec) | `Any` | Any JSON-serializable type, string, or None. | Non-serializable: use `default=str` fallback. | — |

---

## 6. Error Handling

| Condition | Behavior | SRS Reference |
|-----------|----------|---------------|
| Empty module list | Display "No modules found." (table) or `[]` (JSON). Exit 0. | FR-DISC-001 AF-1 |
| Result not JSON-serializable | Use `default=str` in `json.dumps`. | — |
| Terminal without color support | `rich` auto-detects. Can be forced via `NO_COLOR=1` env var. | NFR-PRT-002 |

---

## 7. Verification

| Test ID | Description | Expected Result |
|---------|-------------|-----------------|
| T-OUT-01 | `resolve_format(None)` in TTY | Returns `"table"`. |
| T-OUT-02 | `resolve_format(None)` in non-TTY | Returns `"json"`. |
| T-OUT-03 | `resolve_format("json")` in TTY | Returns `"json"` (explicit overrides TTY). |
| T-OUT-04 | `format_module_list` with 2 modules, format="table" | Rich table with 2 rows. |
| T-OUT-05 | `format_module_list` with 0 modules, format="table" | "No modules found." message. |
| T-OUT-06 | `format_module_list` with modules, format="json" | Valid JSON array. |
| T-OUT-07 | `format_module_detail` with full metadata, format="table" | Syntax-highlighted schemas, annotations, tags. |
| T-OUT-08 | `format_module_detail` with minimal metadata (no output_schema) | Output Schema section omitted. |
| T-OUT-09 | `format_exec_result` with dict | JSON-formatted dict. |
| T-OUT-10 | `format_exec_result` with None | Empty stdout. |
| T-OUT-11 | Description truncation at 80 chars | 77 chars + "..." for text > 80 chars. |
