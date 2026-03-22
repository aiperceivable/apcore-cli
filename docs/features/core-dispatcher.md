# Feature Spec: Core Dispatcher

**Feature ID**: FE-01
**Status**: Ready for Implementation
**Priority**: P0
**Parent**: [Tech Design v1.0](../tech-design.md) Section 8.2
**SRS Requirements**: FR-DISP-001, FR-DISP-002, FR-DISP-003, FR-DISP-004, FR-DISP-006

---

## 1. Description

The Core Dispatcher is the primary entry point for `apcore-cli`. It provides the `LazyModuleGroup` Click Group subclass that lazily discovers modules from the apcore Registry, routes commands to built-in subcommands or dynamically generated module commands, handles STDIN JSON input, and delegates execution to the apcore Executor.

---

## 2. Requirements Traceability

| Req ID | SRS Ref | Description |
|--------|---------|-------------|
| FR-01-01 | FR-DISP-001 | Base command entry point with `--help` and `--version`. Program name is resolved dynamically (see FR-01-06). |
| FR-01-02 | FR-DISP-002 | `exec <module_id>` subcommand routing and execution via Executor. |
| FR-01-03 | FR-DISP-003 | Extensions directory loading from configurable path. |
| FR-01-04 | FR-DISP-001 AF-3 | Version flag: `{prog_name} --version` prints `{prog_name}, version X.Y.Z`. |
| FR-01-05 | FR-DISP-004 | STDIN JSON input when `--input -` is specified. |
| FR-01-06 | FR-DISP-006 | CLI program name resolved from `argv[0]` basename; explicit `prog_name` parameter overrides. |

---

## 3. Module Path

`apcore_cli/cli.py`

---

## 4. Implementation Details

### 4.1 Class: `LazyModuleGroup`

**File**: `apcore_cli/cli.py`

```python
class LazyModuleGroup(click.Group):
    """Custom Click Group that lazily loads apcore modules as subcommands."""

    def __init__(self, registry: Registry, executor: Executor, **kwargs):
        super().__init__(**kwargs)
        self._registry = registry
        self._executor = executor
        self._module_cache: dict[str, click.Command] = {}
```

**Method: `list_commands(ctx) -> list[str]`**

Logic steps:
1. Define built-in commands list: `["exec", "list", "describe", "completion", "man"]`.
2. Call `self._registry.list()` to get all module definitions.
3. Extract `canonical_id` from each module definition.
4. Return `sorted(set(builtin + module_ids))`.

Edge cases:
- Registry returns empty list: return only built-in commands.
- Registry raises exception during `list()`: catch, log WARNING, return only built-in commands.

**Method: `get_command(ctx, cmd_name) -> click.Command | None`**

Logic steps:
1. Check `self.commands` dict for built-in commands. If found, return it.
2. Check `self._module_cache` for previously resolved modules. If found, return it.
3. Call `self._registry.get_definition(cmd_name)`.
4. If `None`, return `None` (Click will show "command not found").
5. Call `build_module_command(module_def, self._executor)`.
6. Store result in `self._module_cache[cmd_name]`.
7. Return the command.

### 4.2 Function: `build_module_command`

**Signature**: `build_module_command(module_def: ModuleDefinition, executor: Executor, help_text_max_length: int = 1000) -> click.Command`

Logic steps:
1. Get `input_schema` from `module_def.input_schema`.
2. Call `resolve_refs(input_schema, max_depth=32)` to inline all `$ref` references.
3. Call `schema_to_click_options(resolved_schema)` to generate Click options.
4. Create a Click command with:
   - `name`: `module_def.canonical_id`
   - `help`: `module_def.description`
   - Built-in options: `--input`, `--yes`, `--large-input`, `--format`, `--sandbox`
5. The command callback:
   a. Call `collect_input(stdin_input, kwargs, large_input)` to merge STDIN + CLI flags.
   b. Call `jsonschema.validate(merged, resolved_schema)`. On failure: exit 45 with validation error detail.
   c. Call `check_approval(module_def, auto_approve)`. On denial/timeout: exit 46.
   d. Record `audit_start = time.monotonic()`.
   e. Call `executor.call(module_def.canonical_id, merged)`.
   f. Compute `duration_ms = int((time.monotonic() - audit_start) * 1000)`.
   g. Call `audit_logger.log_execution(module_id, merged, "success", 0, duration_ms)`.
   h. Call `format_output(result, ctx)` to write to stdout.
   i. Exit with code 0.
6. On `Executor` error: catch, write error to stderr, audit log with `"error"`, exit with mapped error code.
7. On `KeyboardInterrupt`: write "Execution cancelled." to stderr, exit 130.
8. Append all generated Click options to `command.params`.
9. Return the command.

### 4.3 Function: `collect_input`

**Signature**: `collect_input(stdin_flag: str | None, cli_kwargs: dict, large_input: bool) -> dict`

Logic steps:
1. If `stdin_flag` is None or empty string: return `cli_kwargs` (with None values removed).
2. If `stdin_flag == "-"`:
   a. Read `sys.stdin.read()` into `raw`.
   b. Check `len(raw.encode('utf-8'))`:
      - If `> 10_485_760` and `large_input is False`: exit code 2, message "STDIN input exceeds 10MB limit. Use --large-input to override."
   c. If `raw` is empty (0 bytes): set `stdin_data = {}`.
   d. Else: parse `json.loads(raw)` into `stdin_data`.
      - On `json.JSONDecodeError`: exit code 2, message "STDIN does not contain valid JSON: {e.msg}."
   e. If `stdin_data` is not a `dict`: exit code 2, message "STDIN JSON must be an object, got {type(stdin_data).__name__}."
   f. Merge: `result = {**stdin_data, **cli_kwargs_non_none}` (CLI flags override STDIN for duplicate keys).
3. Return `result`.

### 4.4 Function: `validate_module_id`

**Signature**: `validate_module_id(module_id: str) -> None`

Logic steps:
1. Check `len(module_id) > 128`: exit code 2, message "Invalid module ID format: '{module_id}'. Maximum length is 128 characters."
2. Check `re.fullmatch(r'^[a-z][a-z0-9_]*(\.[a-z][a-z0-9_]*)*$', module_id)`: if None, exit code 2, message "Invalid module ID format: '{module_id}'."

### 4.5 Function: `create_cli`

**Signature**: `create_cli(extensions_dir: str | None = None, prog_name: str | None = None) -> click.Group`

**File**: `apcore_cli/__main__.py`

**Purpose**: Factory function that assembles and returns the fully configured Click group. Separating construction from invocation enables library users to embed the CLI in a larger application and override settings (including the program name) without touching entry-point code.

Logic steps:
1. Resolve `prog_name` (FR-DISP-006):
   a. If `prog_name` is not `None`, use it (Tier 1 — explicit parameter).
   b. Otherwise, compute `os.path.basename(sys.argv[0])`.
   c. If the result is empty, fall back to `"apcore-cli"`.
2. Resolve and apply initial log level (before Click runs). Three-tier precedence:
   - Tier 1 (highest): `--log-level` CLI flag — applied at runtime in the group callback.
   - Tier 2: `APCORE_CLI_LOGGING_LEVEL` env var — CLI-specific; takes priority over the global var.
   - Tier 3: `APCORE_LOGGING_LEVEL` env var — global apcore setting; used as fallback when the CLI-specific var is absent.
   - Default: `logging.WARNING`.

   Steps:
   a. Read `APCORE_CLI_LOGGING_LEVEL`; if empty, read `APCORE_LOGGING_LEVEL`. Map to a `logging` constant; default to `logging.WARNING` if absent or unrecognised.
   b. Call `logging.basicConfig(level=..., format="%(levelname)s: %(message)s")` once (no-op on subsequent calls).
   c. Set `logging.getLogger("apcore").setLevel(ERROR)` when the resolved level is above INFO; set to the resolved level when INFO or lower. This suppresses noisy upstream apcore output unless the user explicitly requests verbose output.
   d. At runtime the `--log-level` flag overrides: call `logging.getLogger().setLevel(level)` (not `basicConfig`) and apply the same `apcore` level logic. Accepted choices: `DEBUG`, `INFO`, `WARNING`, `ERROR` (case-insensitive).
3. Resolve `extensions_dir`:
   a. If `extensions_dir` is not `None`, use it.
   b. Otherwise, instantiate `ConfigResolver()` and call `config.resolve("extensions.root", cli_flag="--extensions-dir", env_var="APCORE_EXTENSIONS_ROOT")`.
4. Verify `extensions_dir` path exists and is a readable directory:
   - Not exists: exit 47, message "Extensions directory not found: '{path}'. Set APCORE_EXTENSIONS_ROOT or verify the path."
   - Not readable: exit 47, message "Cannot read extensions directory: '{path}'. Check file permissions."
5. Instantiate `Registry(extensions_dir)`.
6. Call `registry.discover()`. Log DEBUG: `"Loading extensions from {extensions_dir}"`.
7. Log INFO: `"Initialized {prog_name} with {N} modules."`.
8. Instantiate `Executor(registry)`.
9. Initialize `AuditLogger()`. Set as module-level global via `set_audit_logger()`.
10. Build `click.Group` using `cls=LazyModuleGroup`, `name=prog_name`, `registry=registry`, `executor=executor`.
11. Add `click.version_option(version=__version__, prog_name=prog_name)`.
12. Add `--extensions-dir` and `--log-level` options to the root group.
13. Register built-in commands via `register_discovery_commands()` and `register_shell_commands(cli, prog_name=prog_name)`.
14. Return the assembled `click.Group`.

### 4.6 Function: `main`

**Signature**: `main(prog_name: str | None = None) -> None`

**File**: `apcore_cli/__main__.py`

**Purpose**: Standard entry-point shim. Pre-extracts `--extensions-dir` from `argv` before Click parses (required because the registry must be instantiated before Click runs), then delegates to `create_cli()`.

Logic steps:
1. Call `_extract_extensions_dir(sys.argv[1:])` to extract `--extensions-dir` value from raw `argv` before Click processes it.
2. Call `create_cli(extensions_dir=ext_dir, prog_name=prog_name)` to obtain the configured CLI group.
3. Invoke `cli(standalone_mode=True)`.

**Library integration patterns (cross-language normative reference):**

| Integration Pattern | How to specify program name |
|--------------------|-----------------------------|
| Default entry point | No action needed. `argv[0]` basename is used automatically. |
| Custom entry point in `pyproject.toml` | Add `myproject = "apcore_cli.__main__:main"` under `[project.scripts]`. The name `myproject` is used automatically. |
| Programmatic override | Call `main(prog_name="myproject")` or `create_cli(prog_name="myproject")`. |
| Other languages (Go, TypeScript, Rust) | Pass the desired name to the equivalent `createCLI(progName: string)` factory; fall back to `os.Args[0]` basename when not provided. |

---

## 5. Boundary Values & Limits

| Parameter | Minimum | Maximum | Default | SRS Reference |
|-----------|---------|---------|---------|---------------|
| Module ID length | 1 char | 128 chars | — | FR-DISP-002 |
| STDIN buffer | 0 bytes | 10 MB (configurable) | 10 MB | FR-DISP-004 |
| Registry module count | 0 | 1,000 (design target) | — | NFR-PERF-003 |
| Startup time | — | 100 ms | — | NFR-PERF-001 |
| Adapter overhead | — | 50 ms | — | NFR-PERF-002 |

---

## 6. Error Handling

| Condition | Exit Code | Error Message | SRS Reference |
|-----------|-----------|---------------|---------------|
| Invalid module ID format | 2 | "Error: Invalid module ID format: '{id}'." | FR-DISP-002 AF-2 |
| Module not found | 44 | "Error: Module '{id}' not found in registry." | FR-DISP-002 AF-1 |
| Module disabled | 44 | "Error: Module '{id}' is disabled." | FR-DISP-002 AF-5 |
| Module load error | 44 | "Error: Module '{id}' failed to load: {detail}." | FR-DISP-002 AF-7 |
| Schema validation failure | 45 | "Error: Validation failed for '{prop}': {constraint}." | FR-DISP-002 AF-3 |
| Module execution error | 1 | "Error: Module '{id}' execution failed: {detail}." | FR-DISP-002 AF-4 |
| STDIN exceeds buffer | 2 | "Error: STDIN input exceeds 10MB limit. Use --large-input to override." | FR-DISP-004 AF-1 |
| STDIN invalid JSON | 2 | "Error: STDIN does not contain valid JSON: {detail}." | FR-DISP-004 AF-2 |
| STDIN not object | 2 | "Error: STDIN JSON must be an object, got {type}." | FR-DISP-004 AF-3 |
| Extensions dir missing | 47 | "Error: Extensions directory not found: '{path}'. Set APCORE_EXTENSIONS_ROOT or verify the path." | FR-DISP-003 AF-1 |
| Extensions dir unreadable | 47 | "Error: Cannot read extensions directory: '{path}'. Check file permissions." | FR-DISP-003 AF-2 |
| ACL denied | 77 | "Error: Permission denied for module '{id}'." | FR-DISP-002 AF-8 |
| User interruption | 130 | "Execution cancelled." | FR-DISP-002 AF-6 |

---

## 7. Verification

| Test ID | Description | Expected Result |
|---------|-------------|-----------------|
| T-DISP-01 | Run `apcore-cli --help` with valid extensions | Output contains "exec", "list", "describe" and module IDs. Exit 0. |
| T-DISP-02 | Run `apcore-cli --version` | Output: "apcore-cli, version X.Y.Z". Exit 0. |
| T-DISP-03 | Run `apcore-cli exec non.existent` | stderr: "not found". Exit 44. |
| T-DISP-04 | Run `apcore-cli exec "INVALID!ID"` | stderr: "Invalid module ID format". Exit 2. |
| T-DISP-05 | Run `apcore-cli exec math.add --a 5 --b 10` | stdout: module result. Exit 0. |
| T-DISP-06 | Pipe `echo '{"a":5,"b":10}' \| apcore-cli exec math.add --input -` | Module receives `{a:5, b:10}`. Exit 0. |
| T-DISP-07 | Pipe `echo '{"a":5}' \| apcore-cli exec math.add --input - --a 99` | Module receives `{a:99}` (CLI flag overrides STDIN). |
| T-DISP-08 | Pipe 15MB JSON without `--large-input` | stderr: "exceeds 10MB limit". Exit 2. |
| T-DISP-09 | Pipe invalid JSON | stderr: "does not contain valid JSON". Exit 2. |
| T-DISP-10 | Set `APCORE_EXTENSIONS_ROOT=/tmp/test` | Modules loaded from `/tmp/test`. |
| T-DISP-11 | Extensions dir does not exist | stderr: "not found". Exit 47. |
| T-DISP-12 | Extensions dir with one corrupt module | Corrupt module skipped with WARNING. Valid modules available. |
| T-DISP-13 | `--extensions-dir` overrides `APCORE_EXTENSIONS_ROOT` | CLI flag path is used. |
| T-DISP-14 | Default entry point `apcore-cli --version` | Output: `apcore-cli, version X.Y.Z`. Exit 0. |
| T-DISP-15 | Downstream entry point `myproject --version` (installed as `myproject = "apcore_cli.__main__:main"`) | Output: `myproject, version X.Y.Z`. Exit 0. |
| T-DISP-16 | `create_cli(prog_name="custom-name")` invoked with `--help` | Help output contains `custom-name`. Does not contain `apcore-cli`. |
| T-DISP-17 | `create_cli(prog_name=None)` invoked when `argv[0]` is `pytest` | Help output contains `pytest` (argv[0] basename). Falls back gracefully. |
