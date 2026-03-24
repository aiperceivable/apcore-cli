# Changelog

All notable changes to the apcore-cli specification will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/).

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
