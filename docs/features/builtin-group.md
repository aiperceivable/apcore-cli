# Feature Spec: Built-in Command Group (`apcli`)

**Feature ID**: FE-13
**Status**: Draft — Ready for Review
**Priority**: P0
**Parent**: [Tech Design v2.0](../tech-design.md) Section 8.2
**SRS Requirements**: FR-DISP-001, FR-DISP-002, FR-DISC-001, NFR-USB-001
**Related Features**: [Core Dispatcher (FE-01)](core-dispatcher.md), [Grouped Commands (FE-09)](grouped-commands.md), [Discovery (FE-04)](discovery.md), [Init Command (FE-10)](init-command.md), [Exposure Filtering (FE-12)](exposure-filtering.md)
**Breaking Change**: Yes (see §11 Migration)

---

## 1. Description

The Built-in Command Group feature restructures the apcore-cli command surface by moving **all apcore-cli-provided commands** (`list`, `describe`, `exec`, `init`, `validate`, `health`, `usage`, `enable`, `disable`, `reload`, `config`, `completion`, `describe-pipeline`, etc.) under a single reserved group named **`apcli`**. The root level retains only universally recognized meta-commands and flags (`help`, `--help`, `--version`, `--verbose`, `--man`, `--log-level`), plus user business modules/groups discovered from the registry.

This addresses two problems with the pre-v0.8 design:

1. **Namespace pollution in branded CLIs.** When apcore-cli is embedded into a framework integration (e.g., `aisee`, a vision-domain CLI built on `createCli({ registry, executor })`), the built-in commands `list`, `init`, `describe` are common English verbs that collide with real business commands. A terminal user running `aisee list` may reasonably expect to list vision tasks, not apcore modules.
2. **Help-output noise.** Without filtering, `aisee --help` exposes developer-tooling commands (`init module`, `describe-pipeline`, `config set`) to end users who should never see them.

The feature introduces an `apcli` configuration key with three visibility modes — fully visible, fully hidden, or partially visible (subcommand whitelist/blacklist) — plus a default policy that auto-detects embedded vs. standalone usage. Crucially, **hidden ≠ unreachable**: when the `apcli` group is hidden, `aisee apcli list` still executes successfully; it simply does not appear in `--help` output or shell completion. This preserves debuggability, CI scripting, and the "UX filter, not security boundary" philosophy established by FE-12.

A secondary concern is **respecting integrator intent against user-environment overrides**. The feature supports the industry-standard precedence (parameters > env var > config file > default, per Terraform/AWS CLI/dbt/Claude Code conventions) but adds a `disableEnv` opt-out in the `apcli` configuration. Integrators with strong product opinions — who do not want end users to circumvent a hidden `apcli` via `APCORE_CLI_APCLI=show` — can set `disableEnv: true` to sever that mechanism at the library boundary.

As a net simplification, the `BUILTIN_COMMANDS` reserved-names list and the per-command `builtins: {...}` override surface are **retired**. Because every apcore-cli-provided command now lives under the `apcli.*` namespace, business modules at the root level can never collide with built-ins — the collision problem is eliminated at the structural level rather than at config-validation time.

---

## 2. Requirements Traceability

| Req ID | SRS Ref | Description |
|--------|---------|-------------|
| FR-13-01 | FR-DISP-001 | All apcore-cli-provided commands (except `help` and the five meta-flags) register under the `apcli` group. |
| FR-13-02 | FR-DISP-001 | Root `--help` shows only `help`, business modules/groups, and (conditionally) `apcli`. |
| FR-13-03 | FR-DISP-002 | Hidden `apcli` group remains invocable: `<cli> apcli list` executes successfully even when `apcli` is not listed in `--help`. |
| FR-13-04 | NFR-USB-001 | `apcli` config accepts: boolean shorthand (`true`/`false`), or object form with `mode`/`include`/`exclude`/`disableEnv` fields. |
| FR-13-05 | NFR-USB-001 | Default: embedded mode (registry injected via `create_cli()`) → `apcli` hidden; standalone mode (registry discovered from filesystem) → `apcli` visible. |
| FR-13-06 | FR-DISC-001 | `apcli list` / `apcli describe` preserve all behavior from pre-v0.8 `list` / `describe`. |
| FR-13-07 | NFR-USB-001 | Env var `APCORE_CLI_APCLI=show\|hide\|1\|0\|true\|false` forces visibility when env-var reading is not disabled. |
| FR-13-08 | FR-DISP-001 | Shell completion (bash/zsh/fish) respects `apcli` visibility: hidden group and its subcommands do not appear in completion when `apcli` is not visible. |
| FR-13-09 | FR-DISP-002 | Business module whose CLI alias, top-level command name, or group name is `apcli` is rejected with exit code 2 and clear error (the name is reserved). |
| FR-13-10 | NFR-USB-001 | 4-tier precedence for `apcli` visibility: `CliConfig.apcli` > `APCORE_CLI_APCLI` env var (when not disabled) > `apcore.yaml` > auto-detected default. |
| FR-13-11 | NFR-USB-001 | `apcli.disableEnv: true` severs the env-var override, making `CliConfig.apcli` / `apcore.yaml` / auto-detect the only resolution sources. `disableEnv` is configured via CliConfig or apcore.yaml only — it is not exposed as its own env var. |
| FR-13-12 | FR-DISP-002 | `exec` subcommand under `apcli` is always registered when the `apcli` group exists, regardless of `include`/`exclude` subcommand filtering — preserves FE-12's hidden-module-invocation guarantee. |
| FR-13-13 | NFR-USB-001 | Discovery-related root flags (`--extensions-dir`, `--commands-dir`, `--binding`) are registered only in standalone mode; in embedded mode (registry injected) they are not registered at all. |
| FR-13-14 | NFR-USB-001 | `--verbose` behavior is orthogonal to `apcli` visibility: `--verbose` continues to control per-command option density in help output and does not alter `apcli` group visibility. |

---

## 3. Module Path

- `apcore_cli/builtin_group.py` (new module — `ApcliGroup` class, visibility resolution)
- `apcore_cli/__main__.py` — `create_cli()` refactored to register under `apcli` group; discovery flags gated on standalone mode
- `apcore_cli/cli.py` — `GroupedModuleGroup` reserves `apcli` as a protected name; retires `BUILTIN_COMMANDS` constant; introduces `RESERVED_GROUP_NAMES`
- `apcore_cli/discovery.py` — `register_discovery_commands()` split into `register_list_command()`, `register_describe_command()`, `register_exec_command()`, `register_validate_command()` for per-subcommand registration control
- `apcore_cli/init_cmd.py` — `register_init_command()` attaches to `apcli` group
- `apcore_cli/system_cmd.py` — `register_system_commands()` split into `register_health_command()`, `register_usage_command()`, `register_enable_command()`, `register_disable_command()`, `register_reload_command()`, `register_config_command()` for per-subcommand registration control
- `apcore_cli/shell.py` — `register_shell_commands()` split into `register_completion_command()`; attaches to `apcli` group
- `apcore_cli/strategy.py` — `register_pipeline_command()` attaches to `apcli` group
- `apcore_cli/config.py` — `ConfigResolver` gains `apcli.*` keys

---

## 4. Implementation Details

### 4.1 Command Surface After Restructure

**Root level — always present:**

| Entry | Type | Description |
|-------|------|-------------|
| `help` | Command | Standard help dispatcher (Click/Commander built-in). |
| `--help` / `-h` | Flag | Per-command help. |
| `--version` | Flag | Print program version. |
| `--verbose` | Flag | Show built-in per-command options in `--help`. |
| `--man` | Flag | Render formatted man-page style help (used with `--help --man`). |
| `--log-level` | Flag | Set log level (DEBUG/INFO/WARNING/ERROR). |
| `<business-group>` | Group | User modules auto-grouped by namespace (see FE-09). |
| `<business-module>` | Command | Top-level user modules (no namespace prefix). |
| `apcli` | Group | Built-in group — visibility controlled by config. |

**Root level — standalone mode only (NOT registered when registry is injected):**

| Entry | Type | Description |
|-------|------|-------------|
| `--extensions-dir` | Flag | Path to extensions directory (filesystem discovery). |
| `--commands-dir` | Flag | Path to convention-based commands directory (apcore-toolkit). |
| `--binding` | Flag | Path to `binding.yaml` for display overlay (apcore-toolkit). |

Rationale: these three flags only have meaning when apcore-cli performs its own filesystem discovery. In embedded mode — where the integrator has pre-built the `registry` and passes it programmatically — the flags are inert (do nothing) and pollute `--help --verbose` output. Omitting them entirely in embedded mode results in cleaner help and an explicit "unknown option" error if a user attempts them.

**Under `apcli` group:**

| Subcommand | Source | Notes |
|------------|--------|-------|
| `apcli list` | `discovery.py` | Renamed from root `list`. |
| `apcli describe <module-id>` | `discovery.py` | Renamed from root `describe`. |
| `apcli exec <module-id>` | `discovery.py` | Renamed from root `exec`. **Always registered** (see §4.9). |
| `apcli validate <module-id>` | `discovery.py` | Renamed from root `validate`. |
| `apcli init module <id>` | `init_cmd.py` | Renamed from root `init`. |
| `apcli health` | `system_cmd.py` | Renamed from root `health`. |
| `apcli usage` | `system_cmd.py` | Renamed from root `usage`. |
| `apcli enable <module-id>` | `system_cmd.py` | Renamed from root `enable`. |
| `apcli disable <module-id>` | `system_cmd.py` | Renamed from root `disable`. |
| `apcli reload` | `system_cmd.py` | Renamed from root `reload`. |
| `apcli config get / set` | `system_cmd.py` | Renamed from root `config`. |
| `apcli completion <shell>` | `shell.py` | Renamed from root `completion`. |
| `apcli describe-pipeline` | `strategy.py` | Renamed from root `describe-pipeline`. |

### 4.2 Configuration Schema

**File**: `apcore.yaml`

```yaml
# Shorthand: hide the entire apcli group.
apcli: false

# Shorthand: show the entire apcli group.
apcli: true

# Object form: partial visibility via subcommand whitelist.
apcli:
  mode: include
  include:
    - list
    - describe

# Object form: partial visibility via subcommand blacklist.
apcli:
  mode: exclude
  exclude:
    - init
    - config
    - describe-pipeline

# Object form: lock down (hide + disable env-var override).
apcli:
  mode: none
  disable_env: true      # End users cannot use APCORE_CLI_APCLI to override.

# Object form: fully expanded.
apcli:
  mode: all              # "all" | "none" | "include" | "exclude". Default: auto-detect.
  include: []            # Used when mode is "include". Subcommand names.
  exclude: []            # Used when mode is "exclude". Subcommand names.
  disable_env: false     # Default: false (honor APCORE_CLI_APCLI env var).
```

**Semantic mapping (boolean shorthand → object form):**

| Shorthand | Equivalent Object Form |
|-----------|------------------------|
| `apcli: true` | `apcli: { mode: "all", disable_env: false }` |
| `apcli: false` | `apcli: { mode: "none", disable_env: false }` |
| `apcli: null` / omitted | Auto-detect visibility (see §4.5), `disable_env: false` |

**Validation rules:**
- `mode` must be one of `"all"`, `"none"`, `"include"`, `"exclude"`. Invalid value (including `"auto"` — which is an internal sentinel, not accepted from user config): exit 2.
- `mode: include` with empty `include` list: zero subcommands exposed (effectively equivalent to `mode: none`, but `exec` still special-cased per §4.9). No error.
- `mode: exclude` with empty `exclude` list: all subcommands exposed (equivalent to `mode: all`). No error.
- Unknown subcommand names in `include` / `exclude`: log WARNING, ignore. (Forward-compatible — allows config to reference subcommands added in future apcore-cli versions.)
- `disable_env` must be a boolean; default `false`. Non-boolean value: log WARNING, treat as `false`.
- `include` and `exclude` lists match against the **first-level apcli subcommand name only** (e.g., `list`, `config`). Nested paths (`config.set`) are not supported in v0.8 — if `config` is in the list, its entire subtree is controlled.

### 4.3 Class: `ApcliGroup`

**File**: `apcore_cli/builtin_group.py` (new)

```python
class ApcliGroup:
    """Built-in command group configuration for apcore-cli-provided commands.

    Encapsulates visibility resolution and subcommand filtering for the
    `apcli` group. Instantiated once by create_cli() and attached to the
    root Click group.
    """

    def __init__(
        self,
        mode: str = "auto",                  # internal sentinel; not accepted from user config
        include: list[str] | None = None,
        exclude: list[str] | None = None,
        disable_env: bool = False,
        registry_injected: bool = False,
    ) -> None:
        ...

    @classmethod
    def from_config(
        cls,
        config: dict | bool | None,
        *,
        registry_injected: bool,
    ) -> "ApcliGroup":
        """Construct from apcore.yaml 'apcli' value + context.

        Args:
            config: The raw 'apcli' value (None | bool | dict).
            registry_injected: True if create_cli() received a pre-populated
                registry (embedded mode).
        """
        ...

    def resolve_visibility(self) -> str:
        """Return effective mode after applying env-var override and auto-detect.

        Returns one of: "all", "none", "include", "exclude".
        """
        ...

    def is_subcommand_visible(self, subcommand: str) -> bool:
        """True if the named subcommand should appear in `apcli --help`."""
        ...

    def is_group_visible(self) -> bool:
        """True if the `apcli` group itself should appear in root `--help`."""
        ...
```

### 4.4 Method: `resolve_visibility`

**Signature**: `resolve_visibility(self) -> str`

Logic steps:
1. If `self._mode != "auto"` and visibility was explicitly set by caller (via `CliConfig` / `from_config` receiving a non-null value): this is Tier 1. Skip env-var check and return `self._mode` directly.
2. Check env var `APCORE_CLI_APCLI` (Tier 2):
   - Skip this step entirely if `self._disable_env` is True.
   - If value is `"show"` / `"1"` / `"true"` (case-insensitive): return `"all"`.
   - If value is `"hide"` / `"0"` / `"false"` (case-insensitive): return `"none"`.
   - Other non-empty values: log WARNING, ignore env var.
3. If `self._mode == "auto"` (no yaml, no CliConfig override, Tier 3 absent):
   - If `self._registry_injected` is True: return `"none"` (embedded default).
   - Else: return `"all"` (standalone default).
4. Return `self._mode` (Tier 3 — yaml value).

**Precedence encoding:** the distinction between Tier 1 (CliConfig) and Tier 3 (yaml) is captured by having `from_config` respect explicit values while `create_cli(apcli=None)` defers to yaml via ConfigResolver. A CliConfig value set to `None` means "not programmatically overridden"; a CliConfig value set to `False`/`True`/`dict`/`ApcliGroup` instance means "programmatic override, env var should not override this."

### 4.5 Auto-Detect Default Logic

The `mode: auto` internal sentinel applies when no explicit `apcli` config is provided anywhere (not in CliConfig, not in yaml):

| Condition | Default | Rationale |
|-----------|---------|-----------|
| `create_cli(registry=reg, ...)` — registry injected | `mode: none` | Embedded CLI. End users consume business modules, not developer tools. |
| `create_cli()` — no registry, filesystem discovery | `mode: all` | Standalone `apcore-cli` binary. Developer/module-author audience. |

The `registry_injected` flag is captured by `create_cli()` at construction time. It is **not** re-evaluated at command-dispatch time.

**Note:** `"auto"` is an internal sentinel only. Users writing `mode: auto` in `apcore.yaml` receive an exit-2 validation error per §4.2. The way to request auto-detect behavior is to omit the `apcli` key entirely or set `apcli: null`.

### 4.6 Method: `is_subcommand_visible`

**Signature**: `is_subcommand_visible(self, subcommand: str) -> bool`

Logic steps:
1. Let `mode = self.resolve_visibility()`.
2. If `mode == "none"`: return `False`.
3. If `mode == "all"`: return `True`.
4. If `mode == "include"`: return `subcommand in self._include`.
5. If `mode == "exclude"`: return `subcommand not in self._exclude`.

### 4.7 Method: `is_group_visible`

**Signature**: `is_group_visible(self) -> bool`

Logic steps:
1. Let `mode = self.resolve_visibility()`.
2. Return `mode != "none"`.

Note: when `mode == "include"` with an empty list, `is_group_visible()` returns True (the group itself appears), but only `exec` is registered under it (via the always-registered rule, §4.9). The group's `--help` will show just `exec`. Rare edge case; logged at DEBUG level on startup.

### 4.8 Integration: `create_cli()` Update

**File**: `apcore_cli/__main__.py`

The `create_cli()` signature gains an `apcli` parameter. See tech-design §8.2.7 for the full canonical signature; this feature adds:

```python
def create_cli(
    # ... canonical parameters, see tech-design §8.2.7 ...
    apcli: bool | dict | ApcliGroup | None = None,  # NEW
) -> click.Group:
```

Construction logic:

```python
registry_injected = registry is not None

# Build ApcliGroup from CliConfig + yaml + auto-detect
if isinstance(apcli, ApcliGroup):
    apcli_cfg = apcli
elif isinstance(apcli, (bool, dict)) or apcli is None:
    # Tier 1 (CliConfig) if apcli is not None; else fall through to yaml (Tier 3) via ConfigResolver
    if apcli is None:
        yaml_value = ConfigResolver().resolve("apcli")  # may return None, bool, or dict
        apcli_cfg = ApcliGroup.from_config(yaml_value, registry_injected=registry_injected)
    else:
        apcli_cfg = ApcliGroup.from_config(apcli, registry_injected=registry_injected)
else:
    raise TypeError(f"apcli: expected bool, dict, ApcliGroup, or None; got {type(apcli).__name__}")

# Build root Click group (see tech-design §8.2.7 for full setup)
root = click.Group(cls=GroupedModuleGroup, ...)

# Conditionally register discovery-related root flags (standalone mode only)
if not registry_injected:
    root.params.append(click.Option(["--extensions-dir"], ...))
    root.params.append(click.Option(["--commands-dir"], ...))
    root.params.append(click.Option(["--binding"], ...))

# Build apcli Click group — always created, hidden from help conditionally
apcli_click_group = click.Group(
    name="apcli",
    help="Built-in apcore-cli commands.",
    hidden=not apcli_cfg.is_group_visible(),
)

# Register subcommands into apcli, filtered by visibility (§4.9)
_register_apcli_subcommands(apcli_click_group, apcli_cfg, registry, executor)

root.add_command(apcli_click_group)
return root
```

### 4.9 Integration: Per-Subcommand Registration

**File**: Multiple (`discovery.py`, `system_cmd.py`, `shell.py`, `strategy.py`, `init_cmd.py`)

Pre-v0.8, registration was batched (`register_discovery_commands` registered four subcommands in one call). This is incompatible with per-subcommand `include`/`exclude` filtering. The registrar functions are split 1-to-1 with subcommands:

```python
# Before (pre-v0.8)
def register_discovery_commands(cli: click.Group, registry: Registry) -> None:
    cli.add_command(list_cmd)
    cli.add_command(describe_cmd)
    cli.add_command(exec_cmd)
    cli.add_command(validate_cmd)

# After (v0.8) — one function per subcommand
def register_list_command(apcli_group: click.Group, registry: Registry) -> None: ...
def register_describe_command(apcli_group: click.Group, registry: Registry) -> None: ...
def register_exec_command(apcli_group: click.Group, registry: Registry, executor: Executor) -> None: ...
def register_validate_command(apcli_group: click.Group, registry: Registry, executor: Executor) -> None: ...
```

Same pattern applies to `system_cmd.py` (split into 6 registrars: health, usage, enable, disable, reload, config) and `shell.py` (split into 1 registrar: completion).

Central dispatcher in `create_cli()`:

```python
# Subcommands that are always registered when the apcli group exists (FE-12 guarantee)
_ALWAYS_REGISTERED: frozenset[str] = frozenset({"exec"})

def _register_apcli_subcommands(
    apcli_group: click.Group,
    apcli_cfg: ApcliGroup,
    registry: Registry | None,
    executor: Executor | None,
) -> None:
    # (name, registrar_callable, requires_executor)
    registrars: list[tuple[str, Callable, bool]] = [
        ("list",              lambda: register_list_command(apcli_group, registry),             False),
        ("describe",          lambda: register_describe_command(apcli_group, registry),         False),
        ("exec",              lambda: register_exec_command(apcli_group, registry, executor),   True),
        ("validate",          lambda: register_validate_command(apcli_group, registry, executor), True),
        ("init",              lambda: register_init_command(apcli_group),                       False),
        ("health",            lambda: register_health_command(apcli_group, executor),           True),
        ("usage",             lambda: register_usage_command(apcli_group, executor),            True),
        ("enable",            lambda: register_enable_command(apcli_group, registry),           False),
        ("disable",           lambda: register_disable_command(apcli_group, registry),          False),
        ("reload",            lambda: register_reload_command(apcli_group, registry),           False),
        ("config",            lambda: register_config_command(apcli_group, executor),           True),
        ("completion",        lambda: register_completion_command(apcli_group),                 False),
        ("describe-pipeline", lambda: register_pipeline_command(apcli_group, executor),         True),
    ]

    for name, registrar, requires_exec in registrars:
        if requires_exec and executor is None:
            continue  # Skip commands that need an executor when none is available
        is_always = name in _ALWAYS_REGISTERED
        if is_always or apcli_cfg.is_subcommand_visible(name):
            registrar()
```

**Key invariants:**

1. **When `mode == "none"` (group hidden, but reachable):** all non-`exec` subcommands still register (because `is_subcommand_visible` returns False for `mode: none`, but `_ALWAYS_REGISTERED` ensures `exec` runs; the other subcommands are not registered at all). Wait — this creates an asymmetry: under `mode: none` the group is hidden but only `exec` is reachable. **Revised rule:** under `mode: none`, `is_subcommand_visible` is bypassed for ALL subcommands — they all register (hidden via the group's `hidden=True` attribute); only the group-level visibility flag is False. This matches the "output filter, not surface reduction" rule for group-level hiding.

Let me restate more precisely:

- **`mode == "none"` (group-level hide):** group is marked `hidden=True` but ALL subcommands register normally. `<cli> apcli list` works. `list` is listed in `<cli> apcli --help`. The group is only hidden from `<cli> --help`.
- **`mode == "include"` / `"exclude"` (subcommand-level filter):** only visible subcommands register. `exec` is always registered as an escape-hatch exception.
- **`mode == "all"`:** all subcommands register, group visible.

Dispatcher logic corrected:

```python
def _register_apcli_subcommands(
    apcli_group: click.Group,
    apcli_cfg: ApcliGroup,
    registry: Registry | None,
    executor: Executor | None,
) -> None:
    mode = apcli_cfg.resolve_visibility()
    for name, registrar, requires_exec in registrars:
        if requires_exec and executor is None:
            continue
        # Mode "all" and "none" → register every subcommand (group visibility handled separately)
        if mode in ("all", "none"):
            registrar()
            continue
        # Mode "include" / "exclude" → filter per subcommand, but always keep exec
        if name in _ALWAYS_REGISTERED or apcli_cfg.is_subcommand_visible(name):
            registrar()
```

**Exception for `exec`:** `exec` is always registered when the `apcli` group exists, regardless of filtering mode. This preserves FE-12's hidden-module-invocation guarantee: hidden business modules remain reachable via `<cli> apcli exec <module_id>`.

### 4.10 Reserved Name Enforcement

**File**: `apcore_cli/cli.py` — `GroupedModuleGroup._build_group_map()`

Extends the existing validation to reject `apcli` as any user-controlled command identifier:

```python
RESERVED_GROUP_NAMES = frozenset({"apcli"})

# In _build_group_map():
for module_id, descriptor in self._modules.items():
    group, cmd = self._resolve_group(module_id, descriptor)
    if group in RESERVED_GROUP_NAMES:
        raise click.UsageError(
            f"Module '{module_id}': group name '{group}' is reserved. "
            f"Use a different CLI alias or set display.cli.group to another value."
        )
    if group is None and cmd in RESERVED_GROUP_NAMES:
        raise click.UsageError(
            f"Module '{module_id}': top-level CLI name '{cmd}' is reserved. "
            f"Use a different CLI alias."
        )
```

Checks cover three cases:
1. Module with `display.cli.group: apcli` → rejected (explicit group name).
2. Module auto-grouped as `apcli` via dotted ID (e.g., `apcli.foo`) → rejected (auto-extracted group).
3. Top-level module (no dot) whose alias or module_id is literally `apcli` → rejected (would collide with the apcli group at root).

This replaces the retired `BUILTIN_COMMANDS` collision check. Since all apcore-cli commands now live under a single reserved namespace, the only collision surface is the `apcli` name itself.

### 4.11 Hidden-but-Reachable Semantics

| Mode | `apcli` in root `--help` | `<cli> apcli --help` subcommands | `<cli> apcli list` executes | `<cli> apcli init` executes |
|------|--------------------------|----------------------------------|------------------------------|------------------------------|
| `all` | ✅ visible | All subcommands | ✅ yes | ✅ yes |
| `none` | ❌ hidden | All subcommands (group `--help` still works) | ✅ yes | ✅ yes |
| `include: [list]` | ✅ visible | `list` + `exec` (always-registered) | ✅ yes | ❌ `No such command` |
| `exclude: [init]` | ✅ visible | All except `init` | ✅ yes | ❌ `No such command` |

The asymmetry between **group-level hide** (`mode: none`, reachable) and **subcommand-level hide** (`include`/`exclude`, unreachable for filtered subcommands) is deliberate:

- Group-level hide is an **output filter**: integrators with `apcli: false` want a clean `--help` but full debugging capability. This matches FE-12's "UX filter, not security boundary" rule.
- Subcommand-level hide is a **surface reduction**: an integrator listing which subcommands they want implies the others should be gone. Failing loudly is better than silent success — except for `exec`, which is universally guaranteed.

### 4.12 `disableEnv` Semantics

`disable_env` (YAML) / `disableEnv` (JS/TS/Go) / `disable_env` (Python/Rust) severs the env-var override channel.

| `disable_env` | `APCORE_CLI_APCLI` env var | Effect |
|---------------|----------------------------|--------|
| `false` (default) | unset | No effect; visibility from yaml/default. |
| `false` (default) | `show` / `hide` | Overrides yaml/default per Tier 2 in precedence. |
| `true` | unset | No effect. |
| `true` | `show` / `hide` | **Ignored.** Visibility from yaml/default only. |

**Key design decisions:**

1. `disableEnv` is **config-only** — not exposed as its own env var. Exposing an env var that disables env var reading produces circular semantics and undermines the lockdown purpose (end users could re-enable the override via env var).
2. Scope is limited to `APCORE_CLI_APCLI`. Other apcore-cli env vars (`APCORE_CLI_LOGGING_LEVEL`, `APCORE_EXTENSIONS_ROOT`, etc.) are unaffected — they serve distinct purposes (logging, discovery) and remain available.
3. The intended use case is **branded-CLI lockdown**: an integrator who set `apcli: false` does not want a user casually running `APCORE_CLI_APCLI=show aisee --help` to circumvent the product decision. `disable_env: true` seals that path.
4. Debugging an apcore-cli-powered branded CLI with `disable_env: true` requires either editing yaml locally or using standard Unix env manipulation (`env -u`, `unset`) — there is no runtime escape.

### 4.13 Integration: Shell Completion

Shell completion scripts (bash/zsh/fish) must respect both layers of visibility:

- `<cli> <TAB>` (root): does not suggest `apcli` when `mode: none`.
- `<cli> apcli <TAB>` (group): suggests only subcommands that `is_subcommand_visible()` returns True for (plus `exec`, always).
- `<cli> apcli list <TAB>` (subcommand): works identically to pre-v0.8 `<cli> list <TAB>`.

Completion generators consult `ApcliGroup` state baked into the generated script at `<cli> apcli completion <shell>` time. Runtime re-resolution is not supported (scripts are static); regenerating the script is the fix when config changes.

### 4.14 Cross-Language Normative Reference

| Language | API | Config type | Auto-detect trigger |
|----------|-----|-------------|---------------------|
| Python | `create_cli(apcli=False)` or `create_cli(apcli={"mode": "include", "include": ["list"], "disable_env": True})` | `bool \| dict \| ApcliGroup \| None` | `registry` is present |
| TypeScript | `createCli({ apcli: false })` or `createCli({ apcli: { mode: "include", include: ["list"], disableEnv: true } })` | `boolean \| ApcliConfig \| ApcliGroup \| undefined` | `registry` is present |
| Rust | `CliConfig { apcli: Some(ApcliVisibility::None { disable_env: true }), .. }` or `CliConfig { apcli: Some(ApcliVisibility::Include { subcommands: vec!["list".into()], disable_env: false }), .. }` | `Option<ApcliVisibility>` enum | `cli_config.registry.is_some()` |
| Go | `CliConfig{Apcli: &ApcliConfig{Mode: ApcliModeNone, DisableEnv: true}}` or `CliConfig{Apcli: &ApcliConfig{Mode: ApcliModeInclude, Include: []string{"list"}}}` | `*ApcliConfig` struct, pointer for nullability | `cfg.Registry != nil` |

**Rust enum (normative):**

```rust
pub enum ApcliVisibility {
    All { disable_env: bool },
    None { disable_env: bool },
    Include { subcommands: Vec<String>, disable_env: bool },
    Exclude { subcommands: Vec<String>, disable_env: bool },
}

pub struct CliConfig {
    pub registry: Option<Registry>,
    pub executor: Option<Executor>,
    pub apcli: Option<ApcliVisibility>,
    // ... other fields
}
```

**Go struct (normative):**

```go
type ApcliMode string

const (
    ApcliModeAll     ApcliMode = "all"
    ApcliModeNone    ApcliMode = "none"
    ApcliModeInclude ApcliMode = "include"
    ApcliModeExclude ApcliMode = "exclude"
)

type ApcliConfig struct {
    Mode       ApcliMode
    Include    []string
    Exclude    []string
    DisableEnv bool
}

type CliConfig struct {
    Registry *Registry
    Executor *Executor
    Apcli    *ApcliConfig // nil = auto-detect
    // ... other fields
}
```

**TypeScript interface (normative):**

```typescript
export type ApcliConfig =
  | boolean
  | {
      mode: "all" | "none" | "include" | "exclude";
      include?: string[];
      exclude?: string[];
      disableEnv?: boolean;
    };

export interface CreateCliOptions {
  registry?: Registry;
  executor?: Executor;
  apcli?: ApcliConfig;
  // ... other fields
}
```

All four implementations MUST produce identical output for equivalent inputs. The conformance suite (`conformance/fixtures/apcli-visibility/`) provides cross-language fixtures.

---

## 5. Configuration Precedence

The `apcli` visibility follows the industry-standard 4-tier precedence:

| Tier | Source | Notes |
|------|--------|-------|
| 1 (highest) | `CliConfig.apcli` / `create_cli(apcli=...)` | Framework integration passes visibility programmatically. Non-null value wins over env var and yaml. |
| 2 | `APCORE_CLI_APCLI` env var | **Skipped when `apcli.disable_env: true` is set by Tier 1 or Tier 3.** |
| 3 | `apcore.yaml` `apcli:` key | Declarative project-level config. |
| 4 (lowest) | Auto-detected default | `none` when registry injected, `all` otherwise. |

**Env var value semantics:**

| Value | Effect |
|-------|--------|
| `show`, `1`, `true` | Force `mode: all` |
| `hide`, `0`, `false` | Force `mode: none` |
| unset | No override |
| any other non-empty | WARNING, ignored |

The env var controls **group-level visibility only** (all-or-nothing). It cannot convey subcommand whitelists — complex filtering belongs in config files.

**`disable_env` resolution:** when `disable_env` is set in Tier 1 (CliConfig), it wins. When Tier 1 is null, Tier 3 (yaml) `disable_env` applies. Default is `false`. `disable_env` cannot be set via env var.

---

## 6. Boundary Values & Limits

| Parameter | Minimum | Maximum | Default | Reference |
|-----------|---------|---------|---------|-----------|
| Include list length | 0 | (same as total subcommand count) | — | §4.2 |
| Exclude list length | 0 | (same as total subcommand count) | — | §4.2 |
| Subcommand name length | 1 | 32 chars | — | Aligned with Click command-name conventions |
| Visibility resolution cost | O(1) env var read + O(len(list)) membership check | — | — | Negligible (<1µs) |
| `apcli` group registration cost | Single Click group creation + conditional subcommand adds | — | — | No measurable startup impact |

---

## 7. Error Handling

| Condition | Exit Code | Error Message | Reference |
|-----------|-----------|---------------|-----------|
| Invalid `mode` value in `apcore.yaml` (including `"auto"`) | 2 | `"Invalid apcli mode: '{mode}'. Must be one of: all, none, include, exclude."` | FR-13-04 |
| `apcli` is neither bool, dict, nor null | N/A (TypeError) | `"apcli: expected bool, dict, ApcliGroup, or None; got {type}"` | FR-13-04 |
| Non-boolean `disable_env` value | N/A (WARNING) | `"WARNING: apcli.disable_env must be boolean; got {type}. Treating as false."` | FR-13-11 |
| Unknown subcommand in `include`/`exclude` | N/A (WARNING) | `"WARNING: Unknown apcli subcommand '{name}' in include list — ignoring."` | FR-13-04 |
| Business module's CLI group name is `apcli` | 2 | `"Module '{module_id}': group name 'apcli' is reserved. Use a different CLI alias or set display.cli.group to another value."` | FR-13-09 |
| Business module's top-level CLI name is `apcli` | 2 | `"Module '{module_id}': top-level CLI name 'apcli' is reserved. Use a different CLI alias."` | FR-13-09 |
| Hidden group, user runs `<cli> apcli list` | 0 | (normal execution — intentional escape hatch) | FR-13-03 |
| Subcommand-level hidden, user runs `<cli> apcli init` when `init` not in include | 2 | Click's standard `"No such command 'init'"` | §4.11 |
| `APCORE_CLI_APCLI` env var with unknown value | N/A (WARNING) | `"WARNING: Unknown APCORE_CLI_APCLI value '{val}', ignoring. Expected: show, hide, 1, 0, true, false."` | FR-13-07 |
| Embedded mode, user passes `--extensions-dir` | 2 | Click's standard `"Error: No such option: --extensions-dir"` | FR-13-13 |

---

## 8. Verification

| Test ID | Description | Expected Result |
|---------|-------------|-----------------|
| T-APCLI-01 | Standalone `apcore-cli --help` (no registry injected) | `apcli` group visible in root help. |
| T-APCLI-02 | Embedded `createCli({ registry })` with no `apcli` config | `apcli` group hidden from root help. |
| T-APCLI-03 | `apcli: true` with embedded registry | `apcli` group visible, overriding auto-default. |
| T-APCLI-04 | `apcli: false` in standalone mode | `apcli` group hidden, overriding auto-default. |
| T-APCLI-05 | Hidden `apcli`, run `<cli> apcli list` | Executes successfully. |
| T-APCLI-06 | Hidden `apcli`, run `<cli> apcli --help` | Shows subcommand list (group itself is reachable). |
| T-APCLI-07 | `apcli: { mode: include, include: [list, describe] }` | `list`, `describe`, and `exec` visible in `<cli> apcli --help` (exec always registered). |
| T-APCLI-08 | Same config, run `<cli> apcli init` | Error: "No such command 'init'". |
| T-APCLI-09 | `apcli: { mode: exclude, exclude: [init] }` | All subcommands except `init` visible and reachable. |
| T-APCLI-10 | `APCORE_CLI_APCLI=show` with yaml `apcli: false` | Group visible (env overrides yaml, Tier 2 > Tier 3). |
| T-APCLI-11 | `APCORE_CLI_APCLI=hide` with yaml `apcli: true` | Group hidden (env overrides yaml). |
| T-APCLI-12 | `APCORE_CLI_APCLI=bogus` | WARNING logged; yaml/default value used. |
| T-APCLI-13 | Programmatic `createCli({ apcli: false })` + `APCORE_CLI_APCLI=show` | Group hidden (Tier 1 > Tier 2). |
| T-APCLI-14 | `apcli: { mode: none, disable_env: true }` + `APCORE_CLI_APCLI=show` | Group hidden; env var ignored. |
| T-APCLI-15 | `apcli: { mode: none, disable_env: true }` + `APCORE_CLI_APCLI` unset | Group hidden (no change vs. without disable_env). |
| T-APCLI-16 | Business module registered with CLI group `apcli` | Exit 2 with reserved-name error. |
| T-APCLI-17 | Business module registered with top-level CLI name `apcli` | Exit 2 with reserved-name error. |
| T-APCLI-18 | Root `--help` output | Shows `help`, `--help`, `--version`, `--verbose`, `--man`, `--log-level`, business modules/groups, `apcli` (if visible). No stray built-in commands. |
| T-APCLI-19 | `<cli> apcli list --help` | Identical output to pre-v0.8 `<cli> list --help` (behavioral parity). |
| T-APCLI-20 | `apcli: { mode: include, include: [] }` | Group visible; `<cli> apcli --help` shows only `exec` (always-registered). |
| T-APCLI-21 | `apcli: { mode: exclude, exclude: [] }` | Equivalent to `mode: all`. |
| T-APCLI-22 | Invalid `mode: whitelist` in apcore.yaml | Exit 2 with error. |
| T-APCLI-23 | yaml `mode: auto` | Exit 2 — `auto` not accepted in user config. |
| T-APCLI-24 | `mode: include, include: [list]`, run `<cli> apcli exec some.module` | Executes successfully (exec always registered, FE-12 guarantee). |
| T-APCLI-25 | Unknown subcommand `foo` in `include` list | WARNING logged; entry ignored. |
| T-APCLI-26 | Non-boolean `disable_env: "yes"` in yaml | WARNING logged; treated as `false`. |
| T-APCLI-27 | Embedded mode, `<cli> --extensions-dir foo` | Exit 2 — unknown option (flag not registered in embedded mode). |
| T-APCLI-28 | Standalone mode, `<cli> --extensions-dir foo` | Flag accepted; discovery path honored. |
| T-APCLI-29 | Shell completion `<cli> <TAB>` with `apcli: false` | `apcli` absent from completion candidates. |
| T-APCLI-30 | Shell completion `<cli> apcli <TAB>` with `include: [list]` | `list` and `exec` in completion candidates. |
| T-APCLI-31 | Cross-language parity: Python/TS/Rust/Go, same `apcli: false` | Identical `--help` outputs. |
| T-APCLI-32 | Startup time with `apcli` group | Within 5% of pre-v0.8 baseline (no regression). |
| T-APCLI-33 | `createCli({ apcli: new ApcliGroup({...}) })` | Programmatic `ApcliGroup` instance accepted. |
| T-APCLI-34 | `<cli> --verbose --help` with `apcli: false` | Per-command options shown for business commands; `apcli` group remains hidden (orthogonal behaviors, FR-13-14). |
| T-APCLI-35 | `env -u APCORE_CLI_APCLI <cli> --help` with `disable_env: true` set | Standard unset behavior; works regardless of disable_env. |

---

## 9. Impact on Existing Features

| Feature | Impact | Changes Required |
|---------|--------|-----------------|
| **Core Dispatcher (FE-01)** | `create_cli()` gains `apcli` parameter; all built-in command registrations move under a sub-group; discovery flags gated on standalone mode | Add parameter; refactor registration dispatch; conditional flag registration |
| **Schema Parser (FE-02)** | No impact | None |
| **Approval Gate (FE-03)** | No impact | None |
| **Discovery (FE-04)** | `list`/`describe`/`exec`/`validate` move under `apcli` group; registrars split per-subcommand | Split `register_discovery_commands` into four functions |
| **Security (FE-05)** | No impact | None |
| **Shell Integration (FE-06)** | `completion` subcommand moves under `apcli`; completion generators consume `ApcliGroup` visibility | Update generators; respect `is_subcommand_visible` |
| **Config Resolver (FE-07)** | New config key `apcli:` | Add to DEFAULTS schema, document |
| **Output Formatter (FE-08)** | No impact (apcli group renders through existing man/help pipeline) | None |
| **Grouped Commands (FE-09)** | Reserves `apcli` as a protected name; retires `BUILTIN_COMMANDS` collision check | Replace constant with `RESERVED_GROUP_NAMES = {"apcli"}`; check top-level command names too |
| **Init Command (FE-10)** | `init` moves under `apcli` group | Update `register_init_command` signature |
| **Usability Enhancements (FE-11)** | `health`/`usage`/`enable`/`disable`/`reload`/`config` move under `apcli`; `--verbose` behavior preserved and orthogonal to apcli visibility | Split `register_system_commands` into per-subcommand registrars |
| **Exposure Filtering (FE-12)** | Orthogonal — `expose` controls business-module visibility; `apcli` controls built-in group visibility. FE-12's `exec` guarantee preserved via `_ALWAYS_REGISTERED` | None. Both filters compose independently. |

---

## 10. Usage Examples

### 10.1 Branded CLI (embedded, clean surface)

```typescript
// aisee/src/cli.ts
import { createCli } from "apcore-cli";
import { app } from "./app.js";

const cli = createCli({
  registry: app.registry,
  executor: app.executor,
  // apcli defaults to hidden because registry is injected — no config needed
});
cli.parse(process.argv);
```

```bash
$ aisee --help
Usage: aisee [options] [command]

aisee — AI-powered vision CLI

Options:
  -h, --help        Display help for command
  --version         Show aisee version
  --verbose         Show all options in help output
  --log-level <lvl> Logging level (DEBUG|INFO|WARNING|ERROR)

Commands:
  vision <cmd>      Vision analysis commands
  search            Search indexed assets
  help [command]    Display help for command

$ aisee apcli list
# Still works — escape hatch for debugging and CI
ID              Description
vision.ocr      Extract text from images
vision.describe Describe image contents
search          Search indexed assets
```

### 10.2 Branded CLI With Partial apcli Exposure

A team wants `aisee apcli list` surfaced for operator convenience, but hides developer tools.

```yaml
# apcore.yaml in aisee repo
apcli:
  mode: include
  include:
    - list
    - describe
```

```bash
$ aisee --help
# ...
Commands:
  vision <cmd>    Vision analysis commands
  search          Search indexed assets
  apcli <cmd>     Built-in apcore-cli commands
  help [command]  Display help for command

$ aisee apcli --help
Commands:
  list            List available modules
  describe <id>   Show module details
  exec <id>       Execute module by ID  # always-registered
```

### 10.3 Branded CLI With Hard Lockdown

Product team decides end users should never see or trigger any apcli behavior, even via env var.

```yaml
# apcore.yaml
apcli:
  mode: none
  disable_env: true
```

```bash
$ APCORE_CLI_APCLI=show aisee --help
# apcli still hidden — disable_env severs the env-var override path
Commands:
  vision <cmd>    Vision analysis commands
  ...

$ aisee apcli list
# Still reachable — mode: none is group-level hide, not surface reduction.
# To truly block, combine with mode: include + empty list.
```

For **total** lockdown (nothing reachable under apcli except `exec` per FE-12):

```yaml
apcli:
  mode: include
  include: []          # only exec remains
  disable_env: true
```

### 10.4 Standalone `apcore-cli` Binary (unchanged developer UX)

```bash
$ apcore-cli --help
# Full surface available — apcli auto-defaults to visible in standalone mode.
# Discovery flags (--extensions-dir, --commands-dir, --binding) available with --verbose.
Commands:
  apcli <cmd>     Built-in apcore-cli commands
  help [command]  Display help for command
  <user groups from ./extensions>
```

### 10.5 Runtime Override for Unlocked Branded CLI

```bash
# Works when the branded CLI did NOT set disable_env:
$ APCORE_CLI_APCLI=show aisee --help
# apcli group now visible; can run `aisee apcli list` etc.
```

### 10.6 Rust

```rust
use apcore_cli::{create_cli, CliConfig, ApcliVisibility};

let cli = create_cli(CliConfig {
    registry: Some(app.registry),
    executor: Some(app.executor),
    apcli: Some(ApcliVisibility::Include {
        subcommands: vec!["list".into(), "describe".into()],
        disable_env: false,
    }),
    ..Default::default()
});
cli.run();
```

### 10.7 Go

```go
import "github.com/aipartnerup/apcore-cli-go"

cli := apcorecli.CreateCli(apcorecli.CliConfig{
    Registry: app.Registry,
    Executor: app.Executor,
    Apcli: &apcorecli.ApcliConfig{
        Mode:       apcorecli.ApcliModeExclude,
        Exclude:    []string{"init", "describe-pipeline"},
        DisableEnv: true,
    },
})
cli.Run()
```

---

## 11. Migration (Breaking Change)

This feature is a **breaking change** for standalone `apcore-cli` users who have scripts or documentation referencing root-level built-in commands.

### 11.1 What Breaks

| Pre-v0.8 invocation | v0.8+ invocation |
|---------------------|------------------|
| `apcore-cli list` | `apcore-cli apcli list` |
| `apcore-cli describe <id>` | `apcore-cli apcli describe <id>` |
| `apcore-cli exec <id>` | `apcore-cli apcli exec <id>` |
| `apcore-cli init module <id>` | `apcore-cli apcli init module <id>` |
| `apcore-cli health` | `apcore-cli apcli health` |
| (all other root built-ins) | (prefix with `apcli`) |

**What does NOT break:**
- Business module invocations (unaffected — modules remain at root level or in their own groups per FE-09).
- Embedded integrations (`create_cli(registry=...)`) — no script ever relied on built-in commands being exposed in that mode; the new default (hidden) is strictly better.
- `--help` / `--version` / `--verbose` / `--log-level` / `--man` flags.

### 11.2 Deprecation Window

For **v0.8.x**, apcore-cli emits a deprecation warning when a root-level built-in command is invoked (standalone mode only):

```
$ apcore-cli list
WARNING: 'list' as a root-level command is deprecated. Use 'apcore-cli apcli list' instead.
         Will be removed in v0.9. See: https://aiperceivable.github.io/apcore-cli/features/builtin-group/#11-migration
```

The command still executes. In **v0.9.0**, root-level aliases are removed entirely.

The deprecation wrapper is implemented by registering thin shim commands at the root for each former built-in, each of which prints the warning and invokes the corresponding `apcli` subcommand. The shim is active **only in standalone mode** (never in embedded mode — embedded integrators' end users should not see deprecation warnings for commands they were never meant to know about).

### 11.3 Version Rollout

| Version | Behavior |
|---------|----------|
| v0.7.x | Pre-existing flat surface (all built-ins at root) |
| v0.8.0 | `apcli` group introduced; standalone mode emits deprecation warnings on old flat invocations; embedded mode defaults to hidden (no deprecation shown); discovery flags gated on standalone mode |
| v0.9.0 | Flat-surface deprecation shims removed; `apcli` group is the only path |

### 11.4 Retiring `BUILTIN_COMMANDS`

The `BUILTIN_COMMANDS` constant in `apcore_cli/cli.py` (and equivalents in the other-language implementations) is **retired** in v0.8.0. It is replaced by:

- `RESERVED_GROUP_NAMES = frozenset({"apcli"})` — one-element set covering group names, auto-grouped prefixes, and top-level command names
- Removal of the per-command collision check in `GroupedModuleGroup._build_group_map()` (was: warn + drop module on collision)

External code that imported `BUILTIN_COMMANDS` will break at import time. Impact assessed as low (internal constant, not part of public API per semver).

---

## 12. Open Questions

1. **Should `help` also move under `apcli`?** Current design keeps `help` at root because it is the most universally recognized meta-command. Resolution: **root `help` preferred** — the English verb `help` has decades of meta-command consensus across `man`, `git`, `docker`, etc.
2. **Should `config get/set` stay nested under `apcli config <subcmd>` or flatten to `apcli config-get` / `apcli config-set`?** Current design preserves the pre-v0.8 nested structure (`apcli config get <key>`). Rationale: the config surface has distinct read vs. write semantics that subcommand grouping expresses well.
3. **Go SDK timing.** `apcore-cli-go` does not exist as of v0.7. The normative Go signature in §4.14 is a forward-looking contract; it becomes binding when the Go SDK is bootstrapped (currently planned for v0.9 per roadmap). Conformance tests for Go are deferred until then.
4. **Should `APCORE_CLI_APCLI=lock` force `disable_env: true` dynamically?** Rejected — creates circular semantics (an env var that disables env vars) and undermines the lockdown purpose. Standard Unix `env -u` / `unset` are the debugging escape hatches instead.
