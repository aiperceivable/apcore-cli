# Changelog

All notable changes to the apcore-cli specification will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/).


## [0.7.0] - 2026-04-21

### Added

- **FE-12: Module Exposure Filtering** — Declarative control over which discovered modules are exposed as CLI commands. Feature spec: `docs/features/exposure-filtering.md`.
  - `expose` section in `apcore.yaml` with three modes: `all` (default), `include` (whitelist), `exclude` (blacklist).
  - Glob-pattern matching on module IDs (e.g., `admin.*`, `webhooks.*`, `*.sse`).
  - `ExposureFilter` class with `is_exposed()` and `filter_modules()` methods.
  - `create_cli(expose=...)` parameter accepting `dict` or `ExposureFilter` instance.
  - `list --exposure {exposed,hidden,all}` filter flag.
  - 4-tier config precedence: `CliConfig.expose` > `--expose-mode` CLI flag > env var > `apcore.yaml`.
  - Hidden modules remain invocable via `exec <module_id>` (UX filter, not a security boundary).
- **FE-13: Built-in Command Group (`apcli`)** — All apcore-cli-provided built-in commands (`list`, `describe`, `exec`, `init`, `validate`, `health`, `usage`, `enable`, `disable`, `reload`, `config`, `completion`, `describe-pipeline`) relocate under a single reserved group named `apcli`. Feature spec: `docs/features/builtin-group.md`. New SRS requirement: FR-DISP-009.
  - Root level retains only `help` + `--help`/`--version`/`--verbose`/`--man`/`--log-level` + user business modules/groups.
  - `apcli` config accepts boolean shorthand (`true`/`false`) or object form (`mode`/`include`/`exclude`/`disable_env`). Mode ∈ {`all`, `none`, `include`, `exclude`}.
  - Auto-detected default: embedded mode (registry injected) → `none` (hidden); standalone mode → `all` (visible).
  - 4-tier precedence: `CliConfig.apcli` > `APCORE_CLI_APCLI` env var (when not disabled) > `apcore.yaml` > auto-detect.
  - `disable_env: true` opt-out severs the env-var override (config-only field; not exposed as a separate env var).
  - "Hidden ≠ unreachable": `mode: none` keeps `<cli> apcli <sub>` reachable for debugging / CI, only hides the group from root `--help`. For true lockdown use `{mode: include, include: [], disable_env: true}`.
  - `exec` subcommand always registered when the `apcli` group exists, preserving FE-12's hidden-module-invocation guarantee.
  - Discovery flags (`--extensions-dir`, `--commands-dir`, `--binding`) register only in standalone mode; embedded-mode CLIs produce "unknown option" if they are passed.
  - `RESERVED_GROUP_NAMES = {"apcli"}` rejects business modules whose group name, auto-grouped prefix, or top-level command name is `apcli`.
  - Normative cross-language contracts for Python / TypeScript / Rust / Go (Go SDK deferred, contract forward-looking).
- **Apache 2.0 license** added (`LICENSE` file).
- SRS: added requirement sections for FE-10 (Init Command), FE-11 (Usability Enhancements), FE-12 (Exposure Filtering) as TODO backfill items (B-004); added FR-DISP-009 (Built-in Command Group) for FE-13.
- **FE-13 conformance fixtures** (`conformance/fixtures/apcli-visibility/`) — cross-language fixture set covering the 4-tier apcli visibility decision chain (`CliConfig` > `APCORE_CLI_APCLI` > `apcore.yaml` > auto-detect). Five scenarios: `standalone-default`, `embedded-default`, `cli-override`, `env-override`, `yaml-include`. Each ships `create_cli.json` (snake_case caller options), `env.json` (env overlay), optional `input.yaml` (materialized as `apcore.yaml`), and a byte-match `expected_help.txt` golden.
- **Canonical help format** (normative, see `conformance/fixtures/apcli-visibility/README.md`) — SDKs must configure their underlying help renderer (Commander.js, Click, clap) to produce clap v4 / GNU-style output: description first, `Usage: <prog> [OPTIONS] [COMMAND]`, `Commands:` before `Options:`, uppercase `<PLACEHOLDER>`, `[default: VALUE]`, `-h, --help` → "Print help", `-V, --version` → "Print version" (short flag required), `-h`/`-V` rendered last, long-only options aligned under short+long rows (4-space slot), and no line wrapping. Reference implementation: `apcore-cli-typescript/src/canonical-help.ts` (Commander `configureHelp({ formatHelp })` override).

### Changed

- **Tech Design v2.0** — Major revision:
  - `BUILTIN_COMMANDS` canonicalized as a 14-entry alphabetically sorted constant (later retired by FE-13, see Removed).
  - `create_cli()` consolidated signature (§8.2.7) is now the authoritative reference; feature specs reference it incrementally.
  - `create_cli()` gains `expose` parameter for programmatic exposure filtering.
  - `create_cli()` gains `apcli` parameter (FE-13) for built-in command group visibility control.
- Root-level built-in commands are deprecated in standalone mode: `apcore-cli list`, `apcore-cli describe`, `apcore-cli exec`, `apcore-cli init`, `apcore-cli health`, `apcore-cli usage`, etc. emit a deprecation WARNING and delegate to the new `apcli` subcommand. Shims removed in v0.8.0.
- Embedded integrations (`create_cli(registry=...)`) now default to `apcli: none` — built-in commands hidden from end-user `--help`. Integrators who want the previous behavior pass `apcli: true` explicitly.
- `register_discovery_commands()` / `register_system_commands()` / `register_shell_commands()` registrars split into per-subcommand functions (`register_list_command`, `register_health_command`, etc.) to enable per-subcommand include/exclude filtering.
- SRS references updated from "Tech Design v1.0" to "Tech Design v2.0" throughout.
- SRS AC-1 for FR-DISP-001 updated: help output must list all 14 built-in commands from `BUILTIN_COMMANDS`.
- Conformance fixture `cli_parity.json` updated: `run` → `exec`, `--json` → `--format json`, `--inputs` → `--input`, exit code `127` → `44`, removed `is_perceivable` key.
- FE-11 version label corrected from "v0.7.0" to "v0.6.0" in feature overview.

### Removed

- **`BUILTIN_COMMANDS` constant retired** (FE-13). The 14-entry list in `apcore_cli/cli.py` and equivalents in other-language SDKs is replaced by `RESERVED_GROUP_NAMES = {"apcli"}`. External code that imported the constant will break at import time; impact assessed as low (internal constant, not public API per semver).
- Per-command collision check in `GroupedModuleGroup._build_group_map()` (was: warn + drop module on collision). Collision surface reduced to the single reserved group name `apcli` — checks are centralized in `RESERVED_GROUP_NAMES` enforcement.

### Migration

- Pre-v0.7 users invoking root-level built-ins see a deprecation warning pointing to the new `apcli`-prefixed path. Full migration table in `docs/features/builtin-group.md` §11. Shims removed in v0.8.0.

---

## [0.6.0] - 2026-04-06

### Changed

- **Dependency bump**: requires `apcore >= 0.17.1` (was `>= 0.15.1`). Incorporates three apcore releases:
  - **apcore 0.16.0**: Execution Pipeline Strategy (`ExecutionStrategy`, `PipelineEngine`, `PipelineTrace`, 11 built-in steps, preset strategies), `Executor.strategy` parameter, `call_with_trace()` / `call_async_with_trace()`, Config Bus enhancements (`env_style`, `max_depth`, `env_prefix` auto-derivation, `env_map`), `ContextKey<T>` typed accessors, `ModuleAnnotations.extra` field, ACL condition handlers (`register_condition()`, `$or`/`$not`, `async_check()`).
  - **apcore 0.17.0**: Pipeline v2 declarative step metadata (`match_modules`, `ignore_errors`, `pure`, `timeout_ms`), `PipelineContext.dry_run` / `version_hint` / `executed_middlewares`, `StepTrace.skip_reason`, `safety_check` → `call_chain_guard` rename, pipeline step reorder (middleware_before now executes before input_validation), YAML pipeline configuration.
  - **apcore 0.17.1**: `minimal` execution strategy preset (4-step pipeline), `requires`/`provides` step dependency metadata.
- Updated SRS, Tech Design, Project Manifest, and feature spec to reflect `apcore >= 0.17.1` dependency.
- Updated ADR-03 (Executor Integration) to document optional `strategy` parameter and `call_with_trace()` availability.
- Updated SRS Execution Constraint to reference the standard 11-step pipeline and custom `ExecutionStrategy` support via pre-configured Executor.

### Added

- **FE-11: Usability Enhancements** — 11 new capabilities implemented across Python, TypeScript, and Rust SDKs:
  - **`--dry-run` preflight mode** (§3.1) — Validates module call without executing via `Executor.validate()`. Standalone `validate` command also added.
  - **System management commands** (§3.2) — `health`, `usage`, `enable`, `disable`, `reload`, `config get`/`config set`. Delegates to `system.*` modules; graceful no-op when unavailable.
  - **Enhanced error output** (§3.3) — Structured JSON errors with `ai_guidance`, `suggestion`, `retryable`, `user_fixable`, `details`. TTY mode shows suggestion and retryable; JSON mode includes all fields.
  - **`--trace` pipeline visualization** (§3.4) — Displays per-step execution trace via `call_with_trace()`.
  - **ApprovalHandler integration** (§3.5) — `--approval-timeout` (configurable), `--approval-token` (async resume). `CliApprovalHandler` class wraps TTY prompt as standard `ApprovalHandler` protocol.
  - **`--stream` output** (§3.6) — JSONL streaming via `Executor.stream()` for modules with `annotations.streaming=true`.
  - **Enhanced `list` command** (§3.7) — `--search`, `--status`, `--annotation`, `--sort`, `--reverse`, `--deprecated`, `--deps` filters.
  - **`--strategy` selection** (§3.8) — Choose execution pipeline: `standard`, `internal`, `testing`, `performance`, `minimal`. New `describe-pipeline` command.
  - **Output format extensions** (§3.9) — `--format csv|yaml|jsonl` and `--fields` for dot-path field selection.
  - **Multi-level grouping** (§3.10) — `cli.group_depth` config (default: 1, max: 3).
  - **Custom command extension point** (§3.11) — `create_cli(extra_commands=[...])` for downstream projects.
- **New error code**: Handle `CONFIG_ENV_MAP_CONFLICT` from apcore 0.16.0.
- New config keys: `cli.approval_timeout` (60), `cli.strategy` ("standard"), `cli.group_depth` (1).
- New environment variables: `APCORE_CLI_APPROVAL_TIMEOUT`, `APCORE_CLI_STRATEGY`, `APCORE_CLI_GROUP_DEPTH`.

### Fixed

- **Schema parser**: Required schema properties now correctly enforced at CLI level (was silently optional).
- **Approval gate**: Fixed inverted logic in annotation type guard that could crash on malformed annotations.

---

## [0.5.1] - 2026-04-03

### Added

- **FR-DISP-008: Pre-populated registry support** — `create_cli()` accepts optional `registry` and `executor` parameters. When a pre-populated Registry is provided, filesystem discovery is skipped entirely. This enables frameworks that register modules at runtime (e.g. apflow's bridge) to generate CLI commands from their existing registry without requiring an extensions directory on disk.
- Cross-language API table in tech-design §8.2.7 and core-dispatcher §4.5: Python `create_cli(registry=)`, TypeScript `createCli({ registry })` via `CreateCliOptions`, Rust `CliConfig { registry }`.
- Verification tests T-DISP-18 through T-DISP-20 for pre-populated registry path.
- Passing `executor` without `registry` is a validation error (ValueError/throw).

---

## [0.5.0] - 2026-03-31

### Changed

- **Dependency bump**: requires `apcore >= 0.15.1` (was `>= 0.13.0`). Adds Config Bus namespace registration, canonical event type naming, Error Formatter Registry support, and simplified env var prefix convention.
- Updated SRS, Tech Design, and Project Manifest to reflect `apcore >= 0.15.1` dependency.

### Added

- **Config Bus integration** — `apcore-cli` registers namespace `apcore-cli` with env prefix `APCORE_CLI` at import time via `Config.register_namespace()`. Supports unified `apcore.yaml` configuration in namespace mode alongside legacy flat-key mode.
- **Canonical event types** — Spec updated to reference dot-namespaced canonical event names (`apcore.module.toggled`, `apcore.config.updated`, `apcore.module.reloaded`, `apcore.health.recovered`) from apcore 0.15.0. SDKs should adopt these names when implementing event subscriptions.
- **New error codes** — Handle `CONFIG_NAMESPACE_RESERVED`, `CONFIG_NAMESPACE_DUPLICATE`, `CONFIG_ENV_PREFIX_CONFLICT`, `CONFIG_MOUNT_ERROR`, `CONFIG_BIND_ERROR`, `ERROR_FORMATTER_DUPLICATE` from apcore 0.15.0.

## [0.4.0] - 2026-03-29

### Added

- **FR-DISP-007: Verbose help mode** — Built-in apcore options (`--input`, `--yes`, `--large-input`, `--format`, `--sandbox`) are now hidden from `--help` output by default. Pass `--help --verbose` to display the full option list. Added `verbose` to reserved flag names to prevent schema property collisions.
- **FR-SHELL-002: Universal man page generation** — `build_program_man_page()` generates a complete roff man page covering all registered commands (including downstream business commands). `configure_man_help()` adds `--help --man` support to any CLI program. Downstream projects get man pages with a single function call.
- **Documentation URL support** — `set_docs_url()` / `setDocsUrl()` sets a base URL for online documentation links in help footers and man pages. Documented in tech-design §8.7.6 and shell-integration §4.10.

### Changed

- `--sandbox` is now always hidden from help (not yet implemented). FR-DISP-007 updated from "five" to "four" toggled options.
- Improved built-in option descriptions across all three SDKs for clarity.
- Updated `core-dispatcher.md` feature spec: added FR-01-07 traceability entry and built-in option visibility note.
- Updated `tech-design.md`: documented verbose help behavior, `build_program_man_page` (§8.7.4), and `configure_man_help` (§8.7.5).
- Updated `srs.md`: added FR-DISP-007 requirement; updated FR-SHELL-002 with full-program mode and acceptance criteria.
- Updated `shell-integration.md` feature spec: added FR-06-03, §4.8 (`build_program_man_page`), §4.9 (`configure_man_help`), and verification tests T-SHELL-09 through T-SHELL-13.

## [0.3.0] - 2026-03-23

### Added

- **Display overlay routing (§5.13)** — `LazyModuleGroup` now reads `metadata["display"]["cli"]` for alias and description when building the command list and routing `get_command()`.
  - `_alias_map`: built from `metadata["display"]["cli"]["alias"]` (with module_id fallback), enabling invocation by alias.
  - `_descriptor_cache`: populated during alias map build to avoid double `registry.get_definition()` calls.
  - `_alias_map_built` flag only set on successful build, allowing retry after transient registry errors.
- **Display overlay in JSON output** — `format_module_list(..., "json")` reads `metadata["display"]["cli"]` for `id`, `description`, and `metadata["display"]["tags"]`.

### Changed

- `_ERROR_CODE_MAP.get(error_code, 1)`: guarded with `isinstance(error_code, str)` to prevent `None`-key lookup.
- Dependency bump: requires `apcore-toolkit >= 0.4.0` for `DisplayResolver`.
- Updated feature specs: `core-dispatcher.md` (alias map, descriptor cache), `output-formatter.md` (JSON branch display overlay).

### Tests

- `TestDisplayOverlayAliasRouting` (6 tests): `list_commands` uses CLI alias, `get_command` by alias, cache hit path, module_id fallback, `build_module_command` alias and description.
- `test_format_list_json_uses_display_overlay`: JSON output uses display overlay alias/description/tags.
- `test_format_list_json_falls_back_to_scanner_when_no_overlay`: JSON output falls back to scanner values.

### Added (Grouped Commands — FE-09)

- **Feature spec `grouped-commands.md`** (FE-09) — nested subcommand groups for CLI. Auto-groups by first `.` segment, with `display.cli.group` override. Includes 10 requirements (FR-09-01 through FR-09-10), boundary values, error handling table, and 18 verification test cases.
- **Tech Design v2.0** — full rewrite incorporating both §5.13 Display Overlay and Grouped CLI Commands. 3 alternative solutions with weighted comparison matrix, 8 ADRs, 5 sequence diagrams.
- **Updated `core-dispatcher.md`** — references FE-09, documents `GroupedModuleGroup` as v2.0 root group.
- **Updated `overview.md`** — added FE-09 row to feature table.

### Added (Convention Module Discovery — §5.14)

- **`apcore-cli init module <id>`** — scaffolding command with `--style` (decorator, convention, binding) and `--description` options. Generates module templates in the appropriate directory.
- **`--commands-dir` CLI option** — path to a convention commands directory. When set, `ConventionScanner` from `apcore-toolkit` scans for plain functions and registers them as modules.

### Tests (Convention Module Discovery)

- 6 new tests in `tests/test_init_cmd.py` covering all three styles and options.

---

## [0.2.2] - 2026-03-22

### Changed
- Rebrand: aipartnerup → aiperceivable

## [0.2.1] - 2026-03-19

### Changed
- **Schema Parser (FE-02)**: Help text truncation limit increased from 200 to 1000 characters (configurable via `cli.help_text_max_length`)
- **Schema Parser (FE-02)**: `_extract_help` signature updated — added `max_length: int = 1000` parameter
- **Schema Parser (FE-02)**: `schema_to_click_options` signature updated — added `max_help_length: int = 1000` parameter
- **Core Dispatcher (FE-01)**: `build_module_command` signature updated — added `help_text_max_length: int = 1000` parameter
- **Config Resolver (FE-07)**: DEFAULTS dict expanded to 6 keys — added `cli.stdin_buffer_limit`, `cli.auto_approve`, `cli.help_text_max_length`
- **Config Resolver (FE-07)**: `logging.level` default corrected from `"INFO"` to `"WARNING"` across all spec documents (tech-design, SRS, config-resolver, core-dispatcher)

### Added
- `cli.help_text_max_length` config key (default: 1000) with `APCORE_CLI_HELP_TEXT_MAX_LENGTH` env var support
- `APCORE_CLI_LOGGING_LEVEL` added to tech-design environment variables table (was documented in feature spec but missing from tech-design)

### Fixed
- **Spec chain contradiction**: `logging.level` default was `"INFO"` in 5 locations but `"WARNING"` in core-dispatcher — now consistently `"WARNING"` everywhere
- **Spec chain contradiction**: `check_approval` pseudocode passed 3 arguments but signature defined 2 — removed stale `ctx` argument from pseudocode
- `cli.auto_approve` was listed in config keys table but missing from DEFAULTS dict — added to both tech-design and config-resolver
- core-dispatcher SRS header now includes FR-DISP-005 and FR-DISP-006

## [0.2.0] - 2026-03-16

### Changed
- **Core Dispatcher (FE-01)**: Added 3-tier log level precedence spec — `--log-level` flag > `APCORE_CLI_LOGGING_LEVEL` > `APCORE_LOGGING_LEVEL` > `WARNING`; renumbered `create_cli` logic steps accordingly
- **Core Dispatcher (FE-01)**: `register_shell_commands()` call now passes `prog_name=prog_name` (FR-DISP-006 alignment)
- **Core Dispatcher (FE-01)**: `--log-level` accepted choices updated: `WARN` → `WARNING`
- **Core Dispatcher (FE-01)**: Added FR-DISP-006 requirement — CLI program name resolved from `argv[0]` basename with explicit `prog_name` parameter override
- **Shell Integration (FE-06)**: Added `4.2 _make_function_name` helper spec with POSIX identifier conversion example
- **Shell Integration (FE-06)**: Updated all generator function signatures to include `prog_name: str` parameter; documented `shlex.quote()` usage in all shell directive positions (not just embedded subshell commands)
- **Shell Integration (FE-06)**: Added `4.6 register_shell_commands` spec with `prog_name` parameter and closure capture semantics
- **Shell Integration (FE-06)**: Man page `.SH ENVIRONMENT` section now specifies 4 env vars including `APCORE_CLI_LOGGING_LEVEL`; updated `WARN` → `WARNING`
- **Approval Gate (FE-03)**: `check_approval` signature corrected — removed `ctx: click.Context` parameter (was not used in implementation)
- **Approval Gate (FE-03)**: `annotations` guard updated to support both dict access and attribute access
- **Security Manager (FE-05)**: `_hash_input` formula updated to include `secrets.token_bytes(16)` per-invocation salt (prevents cross-invocation correlation)
- **Security Manager (FE-05)**: `_get_user` extended with `pwd.getpwuid()` as second fallback step before env var lookup

### Added
- `APCORE_CLI_LOGGING_LEVEL` environment variable to CLI Reference and Environment Variables table in README (CLI-specific log level; takes priority over `APCORE_LOGGING_LEVEL`)

### Fixed
- README: `--log-level` default corrected to `WARNING` (was `INFO`); accepted values updated from `WARN` → `WARNING`

## [0.1.0] - 2026-03-15

### Added
- Initial specification and design documents
- Tech Design v0.4 and v1.0
- Software Requirements Specification (SRS) v0.1
- 8 feature specifications: Core Dispatcher, Schema Parser, Approval Gate, Discovery, Security Manager, Shell Integration, Config Resolver, Output Formatter
- Project manifest with feature table and implementation order
- Idea draft with problem validation and requirement IDs
- Beginner guide (Getting Started) section in README
- GitHub repository links and SDK reference table
- Related Projects section linking to ecosystem repos
- Full CLI Reference section (global options, commands, execution options, exit codes)
- Configuration section with 4-tier precedence and environment variables
- Architecture diagram and apcore-to-CLI mapping table
- CHANGELOG.md

[Unreleased]: https://github.com/aiperceivable/apcore-cli/compare/v0.7.0...HEAD
[0.7.0]: https://github.com/aiperceivable/apcore-cli/compare/v0.6.0...v0.7.0
[0.6.0]: https://github.com/aiperceivable/apcore-cli/compare/v0.5.1...v0.6.0
[0.5.1]: https://github.com/aiperceivable/apcore-cli/compare/v0.5.0...v0.5.1
[0.5.0]: https://github.com/aiperceivable/apcore-cli/compare/v0.4.0...v0.5.0
[0.4.0]: https://github.com/aiperceivable/apcore-cli/compare/v0.3.0...v0.4.0
[0.3.0]: https://github.com/aiperceivable/apcore-cli/compare/v0.2.2...v0.3.0
[0.2.2]: https://github.com/aiperceivable/apcore-cli/compare/v0.2.1...v0.2.2
[0.2.1]: https://github.com/aiperceivable/apcore-cli/compare/v0.2.0...v0.2.1
[0.2.0]: https://github.com/aiperceivable/apcore-cli/compare/v0.1.0...v0.2.0
[0.1.0]: https://github.com/aiperceivable/apcore-cli/releases/tag/v0.1.0
