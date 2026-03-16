# Changelog

All notable changes to the apcore-cli specification will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/).

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
