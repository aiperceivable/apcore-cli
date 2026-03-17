# Software Requirements Specification: apcore-cli

---

## 1. Document Information

| Field | Value |
|-------|-------|
| **Document ID** | SRS-APCORE-CLI-001 |
| **Version** | 0.1 |
| **Author** | Spec Forge |
| **Date** | 2026-03-14 |
| **Status** | Draft |

---

## 2. Revision History

| Version | Date | Author | Description |
|---------|------|--------|-------------|
| 0.1 | 2026-03-14 | Spec Forge | Initial draft |

---

## 3. Introduction

### 3.1 Purpose

This Software Requirements Specification defines the functional and non-functional requirements for `apcore-cli`, the CLI adapter for the apcore ecosystem. It provides a verifiable, traceable baseline for design, implementation, testing, and acceptance of the system.

The intended audience includes developers implementing apcore-cli, QA engineers writing test plans, and stakeholders reviewing feature completeness.

### 3.2 Scope

`apcore-cli` is a Python-based command-line interface that exposes apcore modules as terminal subcommands. The system shall:

- Provide an `exec` command that routes to and invokes apcore modules via the Executor.
- Auto-generate CLI flags from module JSON Schema definitions with zero manual configuration.
- Enforce Human-in-the-Loop (HITL) approval gates for modules annotated as requiring approval.
- Offer `list` and `describe` commands for terminal-native module discovery.
- Integrate with shell ecosystems (completion scripts, man pages).
- Authenticate, encrypt sensitive configuration, log all executions, and sandbox module execution.

The system shall NOT provide remote execution via apcore-a2a (deferred to Phase 4) or replace apcore-mcp (they are complementary adapters).

### 3.3 Definitions, Acronyms, and Abbreviations

| Term | Definition |
|------|-----------|
| **apcore** | The core protocol and framework for building AI-Perceivable modules. Provides Registry, Executor, and the 3-layer metadata model (Discovery, Capabilities, Execution). |
| **HITL** | Human-in-the-Loop. A safety pattern requiring explicit human confirmation before executing sensitive or destructive operations. |
| **Registry** | An apcore component that discovers and indexes module definitions from an extensions directory. Provides `list()`, `get_definition()`, and `get_schema()` APIs. |
| **Module** | An apcore module. A self-describing unit of business logic with metadata (description, input_schema, output_schema, annotations). |
| **Canonical ID** | A dot-separated, lowercase identifier for a module (e.g., `math.add`). Format: `^[a-z][a-z0-9_]*(\.[a-z][a-z0-9_]*)*$`. Maximum 128 characters. |
| **TTY** | Teletypewriter. In this context, refers to an interactive terminal session where `sys.stdin.isatty()` returns `True`. |
| **STDIN** | Standard Input. The default input stream for a process, used for piped JSON input. |
| **JSON Schema** | A vocabulary for annotating and validating JSON documents (RFC draft-bhutton-json-schema-01). Used by apcore modules to define `input_schema` and `output_schema`. |
| **PROTOCOL_SPEC** | The apcore Protocol Specification document defining module structure, error codes, environment variable conventions, and metadata standards. |
| **Executor** | The apcore execution engine that runs modules through a middleware chain (ACL, observability, approval handling). |
| **Click** | A Python package for creating CLI interfaces with composable commands and automatic help generation. |

### 3.4 References

| Ref ID | Document | Location |
|--------|----------|----------|
| REF-01 | Tech Design v1.0: apcore-cli | `docs/tech-design.md` |
| REF-02 | Feature Spec: Core Dispatcher | `docs/features/core-dispatcher.md` |
| REF-03 | Feature Spec: Schema Parser | `docs/features/schema-parser.md` |
| REF-04 | Feature Spec: Approval Gate | `docs/features/approval-gate.md` |
| REF-05 | Feature Spec: Discovery | `docs/features/discovery.md` |
| REF-06 | Feature Spec: Security Manager | `docs/features/security.md` |
| REF-07 | Feature Spec: Shell Integration | `docs/features/shell-integration.md` |
| REF-08 | Feature Spec: Config Resolver | `docs/features/config-resolver.md` |
| REF-09 | Feature Spec: Output Formatter | `docs/features/output-formatter.md` |
| REF-10 | apcore PROTOCOL_SPEC | `apcore/PROTOCOL_SPEC.md` |
| REF-11 | Idea Draft: apcore-cli | `ideas/draft.md` |
| REF-12 | Project Manifest: apcore-cli | `docs/project-apcore-cli.md` |

### 3.5 Overview

Section 4 describes the system context, user characteristics, and constraints. Section 5 enumerates all functional requirements organized by module. Section 6 specifies non-functional requirements with measurable targets. Section 7 defines data requirements. Section 8 covers external interface requirements. Section 9 documents requirement sources and traceability. Section 10 provides the appendix.

---

## 4. Overall Description

### 4.1 Product Perspective

`apcore-cli` is one adapter in the apcore ecosystem, alongside `apcore-mcp` (Model Context Protocol adapter) and `apcore-a2a` (Agent-to-Agent adapter). It occupies the CLI channel, translating terminal interactions into apcore module invocations.

```
                    +------------------+
                    |   Developer      |
                    |   (Terminal)     |
                    +--------+---------+
                             |
                             v
+------------------+   +----+-------------+   +------------------+
|   AI Agent       +-->|   apcore-cli     +-->|   apcore Module  |
|   (Shell Exec)   |   |   (The Adapter)  |   |   (Business      |
+------------------+   +----+-------------+   |    Logic)        |
                             |                 +------------------+
                             v
                    +--------+---------+
                    |   apcore Registry |
                    |   (Local/Remote)  |
                    +------------------+
```

apcore-cli is a standalone CLI process. It reads module metadata from the Registry, maps JSON Schema to CLI flags, enforces approval gates, and delegates execution to the apcore Executor.

### 4.2 Product Functions

The system provides eight feature groups:

1. **Core Dispatcher (FE-01, P0)**: CLI entry point (`apcore-cli`), extensions directory loading, module routing and execution via Executor, STDIN JSON piping.
2. **Schema Parser (FE-02, P0)**: Zero-config generation of CLI flags from module JSON Schema `input_schema`, including type mapping, boolean flag pairs, enum choices, required enforcement, help text extraction, and `$ref` resolution.
3. **Approval Gate (FE-03, P1)**: TTY-aware HITL approval prompting for modules with `annotations.requires_approval`, with bypass mechanisms (`--yes`, `APCORE_CLI_AUTO_APPROVE=1`) and timeout enforcement.
4. **Discovery (FE-04, P1)**: `list` and `describe` subcommands for browsing available modules, with tag filtering, format selection (table/json), and TTY-adaptive defaults.
5. **Security Manager (FE-05, P1/P2)**: API key authentication, encrypted config storage (keyring + AES-256-GCM), append-only audit logging, and subprocess sandboxing.
6. **Shell Integration (FE-06, P2)**: Shell completion scripts (bash/zsh/fish) and man page generation.
7. **Config Resolver (FE-07, P0)**: 4-tier configuration precedence (CLI flag > Environment variable > Config file > Default).
8. **Output Formatter (FE-08, P1)**: TTY-adaptive output formatting (JSON for pipes, rich tables for terminals).

### 4.3 User Characteristics

| User Type | Proficiency | Interaction Mode | Key Needs |
|-----------|-------------|------------------|-----------|
| **Developer** | High technical proficiency. Familiar with CLI tools, Unix pipes, JSON. | Interactive terminal (TTY). Types commands, reads formatted output. | Descriptive help text, readable error messages, tab completion, approval prompts. |
| **AI Agent** | Programmatic. Invokes commands via shell exec (`subprocess`, `os.system`). | Non-interactive (non-TTY). Parses stdout, interprets exit codes. | Deterministic exit codes, structured JSON output, no interactive prompts (requires `--yes` or auto-approve). |

### 4.4 Constraints

1. **Language Constraint**: The system shall be implemented in Python >= 3.11, aligned with apcore >= 0.13.0 requirements.
2. **Framework Constraint**: The system shall use the `click` library for CLI command construction (Tech Design ADR-01).
3. **Naming Constraint**: The default CLI program name (as shown in `--help` and `--version` output) shall be derived from the invoking entry-point script name (`argv[0]` basename). It shall NOT be hardcoded to `apcore-cli`. Downstream projects that install their own entry-point script shall receive their project name in all output automatically. An explicit `prog_name` parameter shall override this. Fallback when `argv[0]` is unavailable: `apcore-cli`. See FR-DISP-006 and Tech Design ADR-02.
4. **Execution Constraint**: The system shall use apcore's `Executor` for module invocation, preserving the full middleware chain (Tech Design ADR-03).
5. **Error Code Constraint**: All exit codes shall align with PROTOCOL_SPEC section 8.
6. **Environment Variable Constraint**: All environment variables shall follow the `APCORE_{SECTION}_{KEY}` naming convention (PROTOCOL_SPEC section 9.2).

### 4.5 Assumptions and Dependencies

**Assumptions:**

1. The apcore Registry is available on the local filesystem at a known path.
2. Modules conform to the apcore 3-layer metadata model and provide valid JSON Schema `input_schema`.
3. The host system has Python >= 3.11 installed with pip or poetry available.
4. Terminal emulators used by developers correctly report TTY status via `sys.stdin.isatty()`.

**Dependencies:**

| Dependency | Version | Purpose |
|-----------|---------|---------|
| `apcore` | >= 0.13.0 | Core protocol, Registry, Executor, error hierarchy |
| `click` | >= 8.1 | CLI framework for command construction |
| `jsonschema` | >= 4.20 | JSON Schema validation and parsing |
| `rich` | >= 13.0 | Terminal output formatting (tables, syntax highlighting) |

---

## 5. Functional Requirements

### 5.1 Core Dispatcher (DISP)

#### FR-DISP-001: Base Command Entry Point

| Field | Value |
|-------|-------|
| **ID** | FR-DISP-001 |
| **Title** | Base Command Entry Point |
| **Priority** | P0 |
| **Priority Rationale** | Foundation requirement. All other features depend on the base command existing. Without this, no CLI interaction is possible. |
| **Source** | Tech Design v1.0 ADR-02; Feature Spec FE-01 FR-01-01 |

**Description:** The system shall provide a base command `apcore-cli` that serves as the root entry point for all CLI interactions. The base command shall display a help message listing all available subcommands when invoked with `--help` or when invoked with no arguments.

**Actors:** Developer, AI Agent

**Preconditions:**
- `apcore-cli` is installed and available on the system PATH.

**Main Flow:**
1. The user invokes `apcore-cli` with no arguments or with `--help`.
2. The system shall load the apcore Registry from the configured extensions directory.
3. The system shall enumerate all registered modules via `Registry.list()`.
4. The system shall display a help message containing:
   - The program name (`apcore-cli`).
   - A brief description of the tool.
   - A list of built-in subcommands (`list`, `describe`, `completion`, `man`).
   - A list of available module Canonical IDs (as dynamically registered top-level subcommands).
5. The system shall exit with code 0.

**Alternative Flows:**

- **AF-1: Registry Load Failure.** If the extensions directory does not exist or is inaccessible, the system shall write an error message to stderr suggesting the user set `APCORE_EXTENSIONS_ROOT` or verify the default path (`./extensions`), and exit with code 47 (`CONFIG_NOT_FOUND`).
- **AF-2: Empty Registry.** If the Registry contains zero modules, the system shall display the help message with an informational note "No modules found in registry" and exit with code 0.
- **AF-3: Version Flag.** If the user invokes `apcore-cli --version`, the system shall print the installed version string in the format `apcore-cli, version X.Y.Z` and exit with code 0.

**Postconditions:**
- Help text is written to stdout.
- Exit code is 0 (success) or 47 (configuration error).

**Acceptance Criteria:**

- **AC-1:** Given `apcore-cli` is installed and a valid extensions directory exists, when the user runs `apcore-cli --help`, then the output shall contain the text "list", "describe", "completion", and "man" as available subcommands, along with any registered module IDs, and the process shall exit with code 0.
- **AC-2:** Given the configured extensions directory does not exist, when the user runs `apcore-cli --help`, then stderr shall contain a message including the text "extensions" and a suggestion to set `APCORE_EXTENSIONS_ROOT`, and the process shall exit with code 47.
- **AC-3:** Given the Registry contains modules `math.add` and `text.summarize`, when the user runs `apcore-cli --help`, then the output shall list both `math.add` and `text.summarize`.

---

#### FR-DISP-002: Module Execution

| Field | Value |
|-------|-------|
| **ID** | FR-DISP-002 |
| **Title** | Module Execution via Direct Subcommand |
| **Priority** | P0 |
| **Priority Rationale** | Core value proposition. Without execution, the CLI has no utility. |
| **Source** | Tech Design v1.0 ADR-03; Feature Spec FE-01 FR-01-02; ideas/draft.md FR-001 |

**Description:** The system shall expose each registered module as a direct top-level subcommand (e.g., `apcore-cli math.add`). When invoked, the system accepts a Canonical Module ID as the command name, resolves the module from the Registry, validates input arguments against the module's `input_schema`, and invokes the module via the apcore `Executor.call()` method. The system shall write the module's result to stdout and exit with code 0 on success.

**Actors:** Developer, AI Agent

**Preconditions:**
- The Registry is loaded and contains at least one module.
- The module's `input_schema` is a valid JSON Schema document.

**Main Flow:**
1. The user invokes `apcore-cli <module_id> [--flags...]`.
2. The system shall validate that `<module_id>` conforms to the Canonical ID format: `^[a-z][a-z0-9_]*(\.[a-z][a-z0-9_]*)*$`, with a maximum length of 128 characters.
3. The system shall look up `<module_id>` in the Registry via `Registry.get_definition()`.
4. The system shall parse the module's `input_schema` and generate CLI flags (delegated to Schema Parser, see section 5.2).
5. The system shall collect input values from CLI flags and/or STDIN (see FR-DISP-004).
6. The system shall validate the collected input against `input_schema` using `jsonschema.validate()`.
7. The system shall check `annotations.requires_approval` and invoke the Approval Gate if required (see section 5.3).
8. The system shall invoke the module via `Executor.call(module_id, validated_input)`.
9. The system shall write the module result to stdout.
10. The system shall exit with code 0.

**Alternative Flows:**

- **AF-1: Module Not Found.** If the Registry does not contain `<module_id>`, the system shall write to stderr: `Error: Module '<module_id>' not found in registry.` and exit with code 44 (`MODULE_NOT_FOUND`).
- **AF-2: Invalid Module ID Format.** If `<module_id>` does not match the Canonical ID regex or exceeds 128 characters, the system shall write to stderr: `Error: Invalid module ID format: '<module_id>'.` and exit with code 2 (`GENERAL_INVALID_INPUT`).
- **AF-3: Schema Validation Failure.** If the provided input fails JSON Schema validation, the system shall write to stderr a validation error message identifying the failing property and constraint, and exit with code 45 (`SCHEMA_VALIDATION_ERROR`).
- **AF-4: Module Execution Error.** If the Executor returns an error from the module, the system shall write the error message to stderr and exit with code 1 (`MODULE_EXECUTE_ERROR`).
- **AF-5: Module Disabled.** If the module exists but is marked as disabled, the system shall write to stderr: `Error: Module '<module_id>' is disabled.` and exit with code 44 (`MODULE_DISABLED`).
- **AF-6: User Interruption.** If the user sends SIGINT (Ctrl+C) during execution, the system shall write to stderr: `Execution cancelled.` and exit with code 130 (`EXECUTION_CANCELLED`).
- **AF-7: Module Load Error.** If the module is registered but fails to load (import error, missing entry_point), the system shall write to stderr a diagnostic message and exit with code 44 (`MODULE_LOAD_ERROR`).
- **AF-8: ACL Denied.** If the Executor's ACL middleware rejects the invocation, the system shall write to stderr: `Error: Permission denied for module '<module_id>'.` and exit with code 77 (`ACL_DENIED`).

**Postconditions:**
- On success: module result is written to stdout; exit code is 0.
- On failure: error message is written to stderr; exit code corresponds to the error category per PROTOCOL_SPEC section 8.

**Acceptance Criteria:**

- **AC-1:** Given a registered module `math.add` with `input_schema` requiring `a` (integer) and `b` (integer), when the user runs `apcore-cli math.add --a 5 --b 10`, then stdout shall contain the module result and the process shall exit with code 0.
- **AC-2:** Given no module `non.existent` in the Registry, when the user runs `apcore-cli non.existent`, then stderr shall contain "not found" and the process shall exit with code 44.
- **AC-3:** Given module `math.add` requires integer `a`, when the user runs `apcore-cli math.add --a "hello"`, then stderr shall contain a schema validation error and the process shall exit with code 45.
- **AC-4:** Given a module ID `INVALID!ID`, when the user runs `apcore-cli "INVALID!ID"`, then stderr shall contain "Invalid module ID format" and the process shall exit with code 2.

---

#### FR-DISP-003: Extensions Directory Loading

| Field | Value |
|-------|-------|
| **ID** | FR-DISP-003 |
| **Title** | Extensions Directory Loading |
| **Priority** | P0 |
| **Priority Rationale** | Required for module discovery. Without a loaded Registry, no commands can be generated or executed. |
| **Source** | Feature Spec FE-01 FR-01-03; ideas/draft.md NFR-002 |

**Description:** The system shall load the apcore extensions directory from a configurable path, instantiate a Registry via `Registry(path)`, and invoke `Registry.discover()` to index all available modules. The path shall be resolved using the configuration precedence defined in FR-DISP-005.

**Actors:** System (internal)

**Preconditions:**
- The system is starting up or a command requiring the Registry has been invoked.

**Main Flow:**
1. The system shall resolve the extensions directory path using the following precedence (highest to lowest): CLI flag `--extensions-dir`, environment variable `APCORE_EXTENSIONS_ROOT`, configuration file `apcore.yaml` key `extensions.root`, default value `./extensions`.
2. The system shall verify that the resolved path exists and is a readable directory.
3. The system shall instantiate `Registry(resolved_path)`.
4. The system shall invoke `Registry.discover()` to scan and index modules.
5. The system shall log at DEBUG level: `"Loading extensions from {resolved_path}"`.
6. The system shall log at INFO level: `"Initialized apcore-cli with {N} modules."` where N is the count of discovered modules.

**Alternative Flows:**

- **AF-1: Path Does Not Exist.** If the resolved path does not exist as a directory, the system shall write to stderr: `Error: Extensions directory not found: '{resolved_path}'. Set APCORE_EXTENSIONS_ROOT or verify the path.` and exit with code 47 (`CONFIG_NOT_FOUND`).
- **AF-2: Path Not Readable.** If the resolved path exists but is not readable (permission denied), the system shall write to stderr: `Error: Cannot read extensions directory: '{resolved_path}'. Check file permissions.` and exit with code 47 (`CONFIG_NOT_FOUND`).
- **AF-3: Corrupt Module in Directory.** If one or more modules in the directory fail to load (malformed metadata, import errors), the system shall log a WARNING for each failed module, skip it, and continue loading remaining modules. The system shall not exit with an error unless zero modules load successfully.
- **AF-4: Configuration File Malformed.** If `apcore.yaml` exists but is not valid YAML or lacks the `extensions.root` key, the system shall log a WARNING and fall through to the default path.

**Postconditions:**
- The Registry is instantiated and populated with discovered modules.
- The count of loaded modules is logged.

**Acceptance Criteria:**

- **AC-1:** Given `APCORE_EXTENSIONS_ROOT` is set to `/tmp/my-extensions` containing 3 modules, when the user runs any `apcore-cli` command, then the system shall load modules from `/tmp/my-extensions`.
- **AC-2:** Given `--extensions-dir /custom/path` is provided and `APCORE_EXTENSIONS_ROOT` is also set, when the user runs `apcore-cli --extensions-dir /custom/path list`, then the system shall load from `/custom/path` (CLI flag takes precedence).
- **AC-3:** Given no CLI flag, no env var, no config file, when the user runs `apcore-cli list`, then the system shall attempt to load from `./extensions`.
- **AC-4:** Given the extensions directory contains one valid module and one corrupt module, when the system loads, then the valid module shall be available and a warning shall be logged for the corrupt module.

---

#### FR-DISP-004: STDIN JSON Input

| Field | Value |
|-------|-------|
| **ID** | FR-DISP-004 |
| **Title** | STDIN JSON Input |
| **Priority** | P0 |
| **Priority Rationale** | Enables Unix pipe composability. Critical for AI Agent usage where structured input is piped from upstream processes. |
| **Source** | Feature Spec FE-01 FR-01-05; ideas/draft.md FR-003 |

**Description:** The system shall accept JSON input from STDIN when the user specifies `--input -` on the `exec` subcommand. The system shall read the entire STDIN stream, parse it as JSON, and merge the resulting key-value pairs with any CLI flags provided. CLI flags shall take precedence over STDIN values for the same key.

**Actors:** Developer, AI Agent

**Preconditions:**
- The `exec` subcommand is being invoked.
- STDIN contains a valid JSON object (not an array or primitive).

**Main Flow:**
1. The user invokes `apcore-cli exec <module_id> --input -` with JSON piped to STDIN.
2. The system shall read the STDIN stream up to the buffer limit (10 MB by default).
3. The system shall parse the stream content as a JSON object.
4. The system shall merge the parsed JSON keys with any CLI flag values, where CLI flags take precedence for duplicate keys.
5. The system shall pass the merged input to schema validation and subsequent execution.

**Alternative Flows:**

- **AF-1: STDIN Exceeds Buffer Limit.** If STDIN data exceeds 10 MB and `--large-input` is not specified, the system shall write to stderr: `Error: STDIN input exceeds 10MB limit. Use --large-input to override.` and exit with code 2 (`GENERAL_INVALID_INPUT`).
- **AF-2: Invalid JSON.** If the STDIN content is not valid JSON, the system shall write to stderr: `Error: STDIN does not contain valid JSON: {parse_error_detail}.` and exit with code 2 (`GENERAL_INVALID_INPUT`).
- **AF-3: JSON Not an Object.** If the STDIN content parses as a JSON array, string, number, boolean, or null instead of an object, the system shall write to stderr: `Error: STDIN JSON must be an object, got {actual_type}.` and exit with code 2 (`GENERAL_INVALID_INPUT`).
- **AF-4: Empty STDIN.** If STDIN is empty (0 bytes) when `--input -` is specified, the system shall treat the input as an empty object `{}` and proceed with CLI flag values only.
- **AF-5: STDIN Without --input Flag.** If STDIN has piped data but `--input -` is not specified, the system shall ignore STDIN and use only CLI flag values.
- **AF-6: Large Input Override.** If `--large-input` is specified, the system shall read STDIN without the 10 MB limit.

**Postconditions:**
- STDIN JSON is parsed, merged with CLI flags, and passed to schema validation.

**Acceptance Criteria:**

- **AC-1:** Given module `math.add` expects `a` and `b`, when the user runs `echo '{"a":5,"b":10}' | apcore-cli math.add --input -`, then the module shall receive `{a:5, b:10}` and execute successfully.
- **AC-2:** Given `echo '{"a":5}' | apcore-cli math.add --input - --b 20`, when executed, then the module shall receive `{a:5, b:20}` (CLI flag `--b` overrides nothing; STDIN provides `a`).
- **AC-3:** Given `echo '{"a":5,"b":10}' | apcore-cli math.add --input - --a 99`, when executed, then the module shall receive `{a:99, b:10}` (CLI flag `--a 99` takes precedence over STDIN `a:5`).
- **AC-4:** Given STDIN contains 15 MB of JSON and `--large-input` is not specified, when executed, then stderr shall contain "exceeds 10MB limit" and the process shall exit with code 2.

---

#### FR-DISP-005: Configuration Precedence

| Field | Value |
|-------|-------|
| **ID** | FR-DISP-005 |
| **Title** | Configuration Precedence |
| **Priority** | P0 |
| **Priority Rationale** | Foundational behavior required by PROTOCOL_SPEC section 9.2. Governs how all configurable values are resolved. |
| **Source** | Tech Design v1.0 section 8.3; PROTOCOL_SPEC section 9.2 |

**Description:** The system shall resolve all configurable values using a four-tier precedence hierarchy. For any given configuration key, the system shall use the value from the highest-priority source that provides a value.

**Actors:** System (internal)

**Preconditions:**
- The system is resolving a configuration value.

**Main Flow:**
1. The system shall check for a CLI flag value (e.g., `--extensions-dir`, `--yes`, `--log-level`). If present, use it.
2. If no CLI flag, the system shall check for an environment variable (e.g., `APCORE_EXTENSIONS_ROOT`, `APCORE_CLI_AUTO_APPROVE`, `APCORE_CLI_LOGGING_LEVEL`, `APCORE_LOGGING_LEVEL`). If set and non-empty, use it. For logging level, `APCORE_CLI_LOGGING_LEVEL` takes priority over `APCORE_LOGGING_LEVEL`.
3. If no environment variable, the system shall check for a value in the configuration file `apcore.yaml`. If the file exists, is valid YAML, and contains the relevant key, use it.
4. If no configuration file value, the system shall use the built-in default value.

**Alternative Flows:**

- **AF-1: Config File Not Found.** If `apcore.yaml` is not present at the expected location, the system shall skip tier 3 silently and fall through to the default.
- **AF-2: Config File Malformed.** If `apcore.yaml` exists but contains invalid YAML, the system shall log a WARNING: `"Configuration file 'apcore.yaml' is malformed, using defaults."` and fall through to the default.

**Postconditions:**
- The configuration value is resolved from the highest-priority available source.

**Acceptance Criteria:**

- **AC-1:** Given `--extensions-dir /cli-path` is provided, `APCORE_EXTENSIONS_ROOT=/env-path` is set, and `apcore.yaml` contains `extensions.root: /config-path`, when the system resolves the extensions directory, then it shall use `/cli-path`.
- **AC-2:** Given no CLI flag, `APCORE_EXTENSIONS_ROOT=/env-path` is set, and `apcore.yaml` contains `extensions.root: /config-path`, when the system resolves the extensions directory, then it shall use `/env-path`.
- **AC-3:** Given no CLI flag, no env var, and `apcore.yaml` contains `extensions.root: /config-path`, when the system resolves the extensions directory, then it shall use `/config-path`.
- **AC-4:** Given no CLI flag, no env var, and no config file, when the system resolves the extensions directory, then it shall use `./extensions`.

---

#### FR-DISP-006: CLI Program Name Customization

| Field | Value |
|-------|-------|
| **ID** | FR-DISP-006 |
| **Title** | CLI Program Name Customization |
| **Priority** | P1 |
| **Priority Rationale** | Required for library-mode deployments. Downstream projects that redistribute `apcore-cli` under their own brand receive a broken user experience if the program name is hardcoded. |
| **Source** | Tech Design ADR-02 (revised); Feature Spec FE-01 FR-01-06 |

**Description:** The system shall resolve the CLI program name (used in `--help` output, `--version` output, and error messages) dynamically rather than using a hardcoded value. The resolution shall follow a 2-tier precedence: an explicit parameter overrides automatic detection; otherwise the basename of `argv[0]` is used. This enables downstream projects to publish the CLI under their own project name with zero source-code changes.

**Actors:** Developer, AI Agent, Downstream Library User

**Preconditions:**
- The CLI adapter is installed either as a standalone package or as a dependency of a downstream project.

**Main Flow:**
1. At startup, the system shall resolve `prog_name` using the following precedence (highest to lowest):
   - **Tier 1 — Explicit parameter.** If `prog_name` is passed as an argument to `create_cli(prog_name=...)` or `main(prog_name=...)`, use it verbatim.
   - **Tier 2 — `argv[0]` basename.** Compute `os.path.basename(sys.argv[0])` and use it as `prog_name`.
2. The resolved `prog_name` shall be used as:
   - The Click group `name` parameter (controls the command name in `--help` output).
   - The `prog_name` parameter of `click.version_option()` (controls the version string prefix).
3. The version string format shall be: `{prog_name}, version {X.Y.Z}`.

**Alternative Flows:**

- **AF-1: `argv[0]` empty or unavailable.** If `os.path.basename(sys.argv[0])` returns an empty string or raises an exception, the system shall fall back to the string `apcore-cli`.
- **AF-2: Explicit parameter is `None`.** If `prog_name=None` is passed explicitly, the system shall treat it as "not provided" and proceed to Tier 2 (`argv[0]` basename).

**Postconditions:**
- All user-visible output (help, version, error messages referencing the command name) uses the resolved `prog_name`.

**Acceptance Criteria:**

- **AC-1:** Given a downstream project installs entry-point `myproject = "apcore_cli.__main__:main"`, when the user runs `myproject --help`, then the output shall contain `myproject` as the command name, not `apcore-cli`.
- **AC-2:** Given a downstream project installs entry-point `myproject = "apcore_cli.__main__:main"`, when the user runs `myproject --version`, then the output shall be `myproject, version X.Y.Z`.
- **AC-3:** Given `create_cli(prog_name="custom-name")` is called programmatically, when `--help` is invoked, then the output shall contain `custom-name` regardless of `argv[0]`.
- **AC-4:** Given the default `apcore-cli` entry point is used, when `apcore-cli --version` is run, then the output shall be `apcore-cli, version X.Y.Z`.
- **AC-5:** Given `prog_name=None` is passed to `create_cli()`, when `--help` is invoked, then the output shall contain the `argv[0]` basename (Tier 2 resolution applies).

---

### 5.2 Schema Parser (SCHEMA)

#### FR-SCHEMA-001: Property-to-Flag Mapping

| Field | Value |
|-------|-------|
| **ID** | FR-SCHEMA-001 |
| **Title** | Property-to-Flag Mapping |
| **Priority** | P0 |
| **Priority Rationale** | Core schema-to-CLI conversion. Without this, modules cannot receive input from the command line. |
| **Source** | Feature Spec FE-02 FR-02-01; ideas/draft.md FR-002 |

**Description:** The system shall map each property in a module's JSON Schema `input_schema.properties` to a CLI flag in the format `--property-name value`. Property names containing underscores shall be converted to kebab-case for the flag name (e.g., `input_file` becomes `--input-file`). The Click option type shall be determined by the JSON Schema `type` field.

**Actors:** System (internal)

**Preconditions:**
- A module has been resolved from the Registry.
- The module's `input_schema` is a valid JSON Schema object with a `properties` field.

**Main Flow:**
1. The system shall iterate over each key-value pair in `input_schema.properties`.
2. For each property, the system shall create a `click.Option` with:
   - Flag name: `--{property_name}` with underscores replaced by hyphens.
   - Type mapping:
     - `"string"` maps to `click.STRING`.
     - `"integer"` maps to `click.INT`.
     - `"number"` maps to `click.FLOAT`.
     - `"boolean"` maps to a flag pair (see FR-SCHEMA-002).
     - `"object"` maps to `click.STRING` (expects JSON string input).
     - `"array"` maps to `click.STRING` (expects JSON string input).
   - Default value: the JSON Schema `default` value if specified, otherwise `None`.
3. The system shall register all generated options on the `exec` Click command for the target module.

**Alternative Flows:**

- **AF-1: Unknown Type.** If a property has a `type` value not in the set `{string, integer, number, boolean, object, array}`, the system shall default to `click.STRING` and log a WARNING: `"Unknown schema type '{type}' for property '{name}', defaulting to string."`.
- **AF-2: No Type Specified.** If a property lacks a `type` field entirely, the system shall default to `click.STRING` and log a WARNING.
- **AF-3: Property Name Collision.** If converting underscores to hyphens causes two properties to map to the same flag name, the system shall write to stderr an error identifying the collision and exit with code 48 (`SCHEMA_CIRCULAR_REF` category, schema error).
- **AF-4: File Property Convention.** If a property name ends with `_file` or the property has an `x-cli-file: true` annotation, the system shall use `click.Path(exists=True)` as the type.

**Postconditions:**
- Each JSON Schema property is represented as a CLI flag on the exec command.

**Acceptance Criteria:**

- **AC-1:** Given a module with `input_schema.properties.name` of type `string`, when the system generates CLI flags, then `--name` shall accept a string value.
- **AC-2:** Given a module with `input_schema.properties.count` of type `integer`, when the user provides `--count 5`, then the value `5` shall be passed as an integer to the module.
- **AC-3:** Given a property named `input_file` of type `string`, when the system generates CLI flags, then the flag name shall be `--input-file`.
- **AC-4:** Given a property of type `object`, when the user provides `--data '{"key":"val"}'`, then the string shall be passed through for JSON parsing.

---

#### FR-SCHEMA-002: Boolean Flag Handling

| Field | Value |
|-------|-------|
| **ID** | FR-SCHEMA-002 |
| **Title** | Boolean Flag Handling |
| **Priority** | P0 |
| **Priority Rationale** | Boolean properties are common in module schemas. Incorrect mapping breaks module invocation for any boolean parameter. |
| **Source** | Feature Spec FE-02 FR-02-02 |

**Description:** The system shall map JSON Schema properties with `type: boolean` to Click boolean flag pairs. Each boolean property shall generate both a positive flag (`--property-name`) and a negative flag (`--no-property-name`). The default value shall be `False` unless the schema specifies `default: true`.

**Actors:** System (internal)

**Preconditions:**
- The Schema Parser is processing a property with `type: boolean`.

**Main Flow:**
1. The system shall create a `click.Option` with `is_flag=True` and flag declarations `--property-name/--no-property-name`.
2. If the schema specifies `default: true`, the default shall be `True`; otherwise `False`.
3. The flag value (`True` or `False`) shall be passed to the module input as the corresponding boolean.

**Alternative Flows:**

- **AF-1: Boolean with Enum.** If a boolean property also specifies an `enum` (e.g., `[true]`), the system shall treat it as a standard boolean flag and ignore the enum constraint (the flag inherently constrains to boolean values).

**Postconditions:**
- Boolean properties are represented as `--flag/--no-flag` pairs.

**Acceptance Criteria:**

- **AC-1:** Given a property `verbose` of type `boolean` with no default, when the user provides `--verbose`, then the module shall receive `verbose: true`.
- **AC-2:** Given a property `verbose` of type `boolean` with no default, when the user provides `--no-verbose`, then the module shall receive `verbose: false`.
- **AC-3:** Given a property `verbose` of type `boolean` with `default: true`, when the user provides no flag, then the module shall receive `verbose: true`.

---

#### FR-SCHEMA-003: Enum Choice Mapping

| Field | Value |
|-------|-------|
| **ID** | FR-SCHEMA-003 |
| **Title** | Enum Choice Mapping |
| **Priority** | P0 |
| **Priority Rationale** | Enum properties are frequent in module schemas. Click.Choice provides immediate validation and help text. |
| **Source** | Feature Spec FE-02 FR-02-03 |

**Description:** The system shall map JSON Schema properties with an `enum` array to `click.Choice` options. The allowed values shall be the string representations of each enum member. Click shall enforce validation at the CLI layer, rejecting values not in the enum.

**Actors:** System (internal)

**Preconditions:**
- The Schema Parser is processing a property with an `enum` field.

**Main Flow:**
1. The system shall extract the `enum` array from the property.
2. The system shall convert each enum value to its string representation.
3. The system shall create a `click.Option` with `type=click.Choice([str(v) for v in enum_values])`.
4. The help text shall list the allowed values.

**Alternative Flows:**

- **AF-1: Empty Enum Array.** If the `enum` array is empty (`[]`), the system shall log a WARNING: `"Empty enum for property '{name}', no values allowed."` and treat the property as a regular string.
- **AF-2: Enum with Mixed Types.** If enum values include non-string types (integers, booleans), the system shall convert all to strings for the Click.Choice and reconvert the selected value back to the original type before passing to the module.

**Postconditions:**
- The CLI rejects values not in the enum before module invocation.

**Acceptance Criteria:**

- **AC-1:** Given a property `format` with `enum: ["json", "csv", "xml"]`, when the user provides `--format json`, then the value `"json"` shall be passed to the module.
- **AC-2:** Given a property `format` with `enum: ["json", "csv", "xml"]`, when the user provides `--format yaml`, then Click shall reject the input with an error listing the allowed values, and the process shall exit with code 2.

---

#### FR-SCHEMA-004: Required Property Enforcement

| Field | Value |
|-------|-------|
| **ID** | FR-SCHEMA-004 |
| **Title** | Required Property Enforcement |
| **Priority** | P0 |
| **Priority Rationale** | Without required enforcement, modules receive incomplete input leading to runtime errors that are harder to diagnose than CLI-level validation. |
| **Source** | Feature Spec FE-02 FR-02-04 |

**Description:** The system shall mark CLI flags corresponding to properties listed in the JSON Schema `required` array as `required=True` in Click. Click shall enforce presence of these flags and display an error if they are omitted.

**Actors:** System (internal)

**Preconditions:**
- The Schema Parser is processing a module's `input_schema` that contains a `required` array.

**Main Flow:**
1. The system shall read the `required` array from the `input_schema`.
2. For each property name in the `required` array, the system shall set `required=True` on the corresponding `click.Option`.
3. If a required flag is not provided on the command line and not supplied via STDIN (`--input -`), Click shall display an error: `Error: Missing required option '--{flag-name}'.`

**Alternative Flows:**

- **AF-1: Required Property Not in Properties.** If a property name in `required` does not appear in `properties`, the system shall log a WARNING and ignore the entry.
- **AF-2: Required Satisfied by STDIN.** If a required property is not provided as a CLI flag but is present in the STDIN JSON input (when `--input -` is used), the requirement shall be considered satisfied after merge.

**Postconditions:**
- All required properties are either provided via CLI flags or STDIN, or the system exits with an error.

**Acceptance Criteria:**

- **AC-1:** Given a module with required property `name`, when the user runs `apcore-cli exec module.id` without `--name`, then Click shall display "Missing required option '--name'" and exit with code 2.
- **AC-2:** Given a module with required property `name`, when the user runs `echo '{"name":"test"}' | apcore-cli exec module.id --input -`, then the requirement shall be satisfied.

---

#### FR-SCHEMA-005: Help Text Generation

| Field | Value |
|-------|-------|
| **ID** | FR-SCHEMA-005 |
| **Title** | Help Text Generation |
| **Priority** | P0 |
| **Priority Rationale** | Help text is the primary documentation mechanism for both developers reading `--help` and AI agents parsing command metadata. |
| **Source** | Feature Spec FE-02 FR-02-05 |

**Description:** The system shall generate CLI help text for each flag from the JSON Schema property metadata. If the property contains an `x-llm-description` field, the system shall use its value as the help text (optimized for AI agent consumption). Otherwise, the system shall use the `description` field. If neither field is present, the system shall display no help text for that flag.

**Actors:** System (internal)

**Preconditions:**
- The Schema Parser is generating Click options for a module.

**Main Flow:**
1. For each property, the system shall check for `x-llm-description` first.
2. If `x-llm-description` is present and is a non-empty string, use it as the Click option `help` parameter.
3. If `x-llm-description` is absent or empty, check for `description`.
4. If `description` is present and is a non-empty string, use it as the Click option `help` parameter.
5. If neither field is present, set `help=None` (Click displays no help text for the option).

**Alternative Flows:**

- **AF-1: Description Exceeds Terminal Width.** If the description exceeds 200 characters, the system shall truncate it to 197 characters followed by `...` for the CLI help display. The full description shall remain available in `describe` output.

**Postconditions:**
- Each CLI flag has descriptive help text derived from the schema.

**Acceptance Criteria:**

- **AC-1:** Given a property with `description: "The user's full name"`, when the user runs `apcore-cli exec module.id --help`, then the `--name` flag help text shall contain "The user's full name".
- **AC-2:** Given a property with both `description: "Name"` and `x-llm-description: "Full legal name of the requesting user"`, when the system generates the flag, then the help text shall be "Full legal name of the requesting user".
- **AC-3:** Given a property with neither `description` nor `x-llm-description`, when the user runs `--help`, then the flag shall appear without help text.

---

#### FR-SCHEMA-006: Reference Resolution

| Field | Value |
|-------|-------|
| **ID** | FR-SCHEMA-006 |
| **Title** | JSON Schema Reference Resolution |
| **Priority** | P0 |
| **Priority Rationale** | Modules with composed schemas using `$ref` and `$defs` cannot be invoked without resolution. Blocking for any non-trivial module. |
| **Source** | Feature Spec FE-02 FR-02-06; Tech Design v1.0 section 8.2 |

**Description:** The system shall inline all `$ref` and `$defs` references in a module's `input_schema` before parsing properties into CLI flags. Resolution shall track visited references to detect circular references. The maximum `$ref` resolution depth shall be 32. Schema composition keywords (`allOf`, `anyOf`, `oneOf`) shall be flattened to top-level properties up to 3 levels of nesting.

**Actors:** System (internal)

**Preconditions:**
- The module's `input_schema` contains `$ref` or `$defs` references.

**Main Flow:**
1. The system shall walk the `input_schema` tree.
2. For each `$ref` encountered, the system shall:
   a. Increment the depth counter.
   b. Check if the depth exceeds 32. If so, abort with error.
   c. Check if the `$ref` target is in the visited set. If so, abort with circular reference error.
   d. Add the `$ref` target to the visited set.
   e. Resolve the reference and inline the target schema.
3. For `allOf` arrays, the system shall merge all sub-schemas into a single schema with combined `properties` and `required` arrays.
4. For `anyOf`/`oneOf` arrays, the system shall merge all sub-schema properties (union), marking only shared required properties as required.
5. After resolution, the system shall pass the fully inlined schema to the property-to-flag mapping (FR-SCHEMA-001).

**Alternative Flows:**

- **AF-1: Circular Reference Detected.** If a `$ref` target is already in the visited set, the system shall write to stderr: `Error: Circular $ref detected in schema for module '{module_id}' at path '{ref_path}'.` and exit with code 48 (`SCHEMA_CIRCULAR_REF`).
- **AF-2: Max Depth Exceeded.** If the resolution depth exceeds 32, the system shall write to stderr: `Error: $ref resolution depth exceeded maximum of 32 for module '{module_id}'.` and exit with code 48 (`SCHEMA_CIRCULAR_REF`).
- **AF-3: Unresolvable Reference.** If a `$ref` target path does not exist in the schema's `$defs`, the system shall write to stderr: `Error: Unresolvable $ref '{ref_value}' in schema for module '{module_id}'.` and exit with code 45 (`SCHEMA_VALIDATION_ERROR`).

**Postconditions:**
- The `input_schema` is fully inlined with no `$ref` or `$defs` references remaining.

**Acceptance Criteria:**

- **AC-1:** Given a schema with `$ref: "#/$defs/Address"` and a valid `$defs.Address` definition, when the system resolves references, then the Address properties shall appear as top-level CLI flags.
- **AC-2:** Given a schema with a circular reference (`A -> B -> A`), when the system resolves references, then it shall exit with code 48 and stderr shall contain "Circular $ref detected".
- **AC-3:** Given a schema with `$ref` nesting depth of 33, when the system resolves references, then it shall exit with code 48 and stderr shall contain "depth exceeded".

---

### 5.3 Approval Gate (APPR)

#### FR-APPR-001: Approval Check

| Field | Value |
|-------|-------|
| **ID** | FR-APPR-001 |
| **Title** | Approval Annotation Check |
| **Priority** | P1 |
| **Priority Rationale** | Safety mechanism. Required before executing any destructive or sensitive module. Lower than P0 because non-approval modules can function without this. |
| **Source** | Feature Spec FE-03 FR-03-01 |

**Description:** The system shall inspect the module's `annotations.requires_approval` field after schema validation and before execution. If the field is `true`, the system shall invoke the Approval Gate workflow. If the field is `false`, absent, or not a boolean, the system shall skip the Approval Gate and proceed directly to execution.

**Actors:** System (internal)

**Preconditions:**
- The module has been resolved from the Registry.
- Input arguments have been validated against the schema.

**Main Flow:**
1. The system shall read `annotations.requires_approval` from the module definition.
2. If the value is boolean `true`, the system shall proceed to the Approval Gate (FR-APPR-002 or FR-APPR-003 depending on TTY status).
3. If the value is `false`, absent, `null`, or any non-boolean type, the system shall skip the Approval Gate and proceed to execution.

**Alternative Flows:**

- **AF-1: Annotations Field Missing.** If the module definition has no `annotations` field, the system shall treat `requires_approval` as `false` and proceed to execution.

**Postconditions:**
- The system either enters the Approval Gate workflow or proceeds to execution.

**Acceptance Criteria:**

- **AC-1:** Given a module with `annotations.requires_approval: true`, when the user invokes `exec`, then the Approval Gate shall activate before execution.
- **AC-2:** Given a module with `annotations.requires_approval: false`, when the user invokes `exec`, then execution shall proceed without any approval prompt.
- **AC-3:** Given a module with no `annotations` field, when the user invokes `exec`, then execution shall proceed without any approval prompt.

---

#### FR-APPR-002: TTY Interactive Prompt

| Field | Value |
|-------|-------|
| **ID** | FR-APPR-002 |
| **Title** | TTY Interactive Approval Prompt |
| **Priority** | P1 |
| **Priority Rationale** | Primary interactive safety mechanism for human developers in terminal sessions. |
| **Source** | Feature Spec FE-03 FR-03-03 |

**Description:** When the Approval Gate is activated and the process is running in a TTY (i.e., `sys.stdin.isatty()` returns `True`), the system shall display the module's `ApprovalRequest.message` to the terminal and prompt the user for confirmation with `[y/N]`. The default response shall be `N` (deny). The system shall use `click.confirm()` for the prompt.

**Actors:** Developer

**Preconditions:**
- `annotations.requires_approval` is `true`.
- `sys.stdin.isatty()` returns `True`.
- No bypass mechanism is active (see FR-APPR-004).

**Main Flow:**
1. The system shall display the approval message from the module's metadata.
2. The system shall display the prompt: `"Proceed? [y/N]: "`.
3. The user types `y` or `Y` and presses Enter.
4. The system shall log at INFO level: `"User approved execution of module '{module_id}'."`.
5. The system shall proceed to module execution.

**Alternative Flows:**

- **AF-1: User Denies.** If the user types `n`, `N`, or presses Enter without input (default N), the system shall log at WARNING level: `"Approval rejected by user for module '{module_id}'."`, write to stderr: `Error: Approval denied.`, and exit with code 46 (`APPROVAL_DENIED`).
- **AF-2: Invalid Input.** If the user types any value other than `y`, `Y`, `n`, `N`, or empty, `click.confirm()` shall re-prompt. This is handled by Click's built-in behavior.
- **AF-3: Approval Message Missing.** If the module does not provide an `ApprovalRequest.message`, the system shall display a default message: `"Module '{module_id}' requires approval to execute."`.

**Postconditions:**
- User has approved (proceed to execution) or denied (exit with code 46).

**Acceptance Criteria:**

- **AC-1:** Given a TTY session and a module with `requires_approval: true` and message "This will delete data", when the Approval Gate activates, then the terminal shall display "This will delete data" followed by "Proceed? [y/N]: ".
- **AC-2:** Given the prompt is displayed and the user types `y`, then execution shall proceed.
- **AC-3:** Given the prompt is displayed and the user presses Enter (empty input), then the process shall exit with code 46.

---

#### FR-APPR-003: Non-TTY Behavior

| Field | Value |
|-------|-------|
| **ID** | FR-APPR-003 |
| **Title** | Non-TTY Approval Behavior |
| **Priority** | P1 |
| **Priority Rationale** | Critical for AI Agent usage. Agents invoke CLI via shell exec in non-TTY contexts and need deterministic failure behavior. |
| **Source** | Feature Spec FE-03 FR-03-04 |

**Description:** When the Approval Gate is activated and the process is NOT running in a TTY (i.e., `sys.stdin.isatty()` returns `False`), and no bypass mechanism is active, the system shall reject execution with exit code 46 (`APPROVAL_PENDING`). The system shall write to stderr a message instructing the caller to use `--yes` or `APCORE_CLI_AUTO_APPROVE=1`.

**Actors:** AI Agent

**Preconditions:**
- `annotations.requires_approval` is `true`.
- `sys.stdin.isatty()` returns `False`.
- Neither `--yes` flag nor `APCORE_CLI_AUTO_APPROVE=1` is active.

**Main Flow:**
1. The system shall detect the non-TTY environment via `sys.stdin.isatty() == False`.
2. The system shall write to stderr: `Error: Module '{module_id}' requires approval but no interactive terminal is available. Use --yes or set APCORE_CLI_AUTO_APPROVE=1 to bypass.`
3. The system shall log at ERROR level: `"Non-interactive environment, no bypass provided for module '{module_id}'."`.
4. The system shall exit with code 46 (`APPROVAL_PENDING`).

**Alternative Flows:**

- **AF-1: Bypass Active.** If `--yes` or `APCORE_CLI_AUTO_APPROVE=1` is active, this flow does not apply; see FR-APPR-004.

**Postconditions:**
- Execution is rejected. Exit code is 46.

**Acceptance Criteria:**

- **AC-1:** Given a non-TTY environment and a module with `requires_approval: true` and no bypass, when the agent invokes `apcore-cli exec module.id`, then stderr shall contain "requires approval but no interactive terminal" and the process shall exit with code 46.
- **AC-2:** Given the same conditions but `--yes` is provided, then execution shall proceed (FR-APPR-004 governs).

---

#### FR-APPR-004: Bypass Mechanisms

| Field | Value |
|-------|-------|
| **ID** | FR-APPR-004 |
| **Title** | Approval Bypass Mechanisms |
| **Priority** | P1 |
| **Priority Rationale** | Enables CI/CD pipelines, scripted workflows, and agent automation to run approval-required modules without interactive prompts. |
| **Source** | Feature Spec FE-03 FR-03-05; Tech Design v1.0 section 8.3 |

**Description:** The system shall provide two mechanisms to bypass the Approval Gate. Bypass evaluation shall follow a strict priority order: (1) `--yes` CLI flag (highest), (2) `APCORE_CLI_AUTO_APPROVE=1` environment variable. If either is active, the system shall skip the TTY prompt and non-TTY rejection, log the bypass, and proceed directly to execution.

**Actors:** Developer, AI Agent, CI/CD System

**Preconditions:**
- `annotations.requires_approval` is `true`.

**Main Flow:**
1. The system shall check for `--yes` CLI flag. If present, bypass approval.
2. If `--yes` is not present, the system shall check for `APCORE_CLI_AUTO_APPROVE` environment variable. If its value is exactly `"1"`, bypass approval.
3. When bypassing, the system shall log at INFO level: `"Approval bypassed via {mechanism} for module '{module_id}'."` where `{mechanism}` is `--yes flag` or `APCORE_CLI_AUTO_APPROVE`.
4. The system shall proceed to module execution.

**Alternative Flows:**

- **AF-1: APCORE_CLI_AUTO_APPROVE Invalid Value.** If `APCORE_CLI_AUTO_APPROVE` is set to any value other than `"1"` (e.g., `"true"`, `"yes"`, `"0"`), the system shall treat it as unset and not bypass. The system shall log a WARNING: `"APCORE_CLI_AUTO_APPROVE is set to '{value}', expected '1'. Ignoring."`.
- **AF-2: Both Mechanisms Active.** If both `--yes` and `APCORE_CLI_AUTO_APPROVE=1` are active, the system shall use `--yes` (higher priority) and log accordingly. No error.

**Postconditions:**
- Approval is bypassed. Module execution proceeds.

**Acceptance Criteria:**

- **AC-1:** Given `--yes` is provided, when invoking a module with `requires_approval: true`, then execution shall proceed without prompting and the log shall contain "bypassed via --yes flag".
- **AC-2:** Given `APCORE_CLI_AUTO_APPROVE=1` is set and no `--yes` flag, when invoking a module with `requires_approval: true` in a non-TTY, then execution shall proceed.
- **AC-3:** Given `APCORE_CLI_AUTO_APPROVE=true` (not `"1"`), when invoking a module with `requires_approval: true` in a non-TTY, then the process shall exit with code 46 and a warning shall be logged about the invalid value.

---

#### FR-APPR-005: Approval Timeout

| Field | Value |
|-------|-------|
| **ID** | FR-APPR-005 |
| **Title** | Approval Timeout |
| **Priority** | P1 |
| **Priority Rationale** | Prevents indefinite blocking in TTY sessions where the user steps away or forgets to respond. |
| **Source** | Feature Spec FE-03 section 3.2 |

**Description:** When the Approval Gate displays a TTY prompt (FR-APPR-002), the system shall enforce a timeout of 60 seconds. If the user does not respond within the timeout period, the system shall treat the prompt as denied due to timeout.

**Actors:** Developer

**Preconditions:**
- The TTY approval prompt (FR-APPR-002) is displayed and waiting for input.

**Main Flow:**
1. The system shall start a 60-second timer when the prompt is displayed.
2. If the user responds within 60 seconds, proceed per FR-APPR-002 (approved or denied).
3. If 60 seconds elapse without user input, proceed to AF-1.

**Alternative Flows:**

- **AF-1: Timeout Reached.** The system shall log at WARNING level: `"Approval timed out after 60s for module '{module_id}'."`. The system shall write to stderr: `Error: Approval prompt timed out after 60 seconds.` and exit with code 46 (`APPROVAL_TIMEOUT`).

**Postconditions:**
- User responded within timeout (proceed per FR-APPR-002) or timeout triggered (exit with code 46).

**Acceptance Criteria:**

- **AC-1:** Given a TTY approval prompt is displayed and 60 seconds elapse without input, then stderr shall contain "timed out after 60 seconds" and the process shall exit with code 46.
- **AC-2:** Given a TTY approval prompt is displayed and the user responds with `y` at 59 seconds, then execution shall proceed normally.

---

### 5.4 Discovery (DISC)

#### FR-DISC-001: Module Listing

| Field | Value |
|-------|-------|
| **ID** | FR-DISC-001 |
| **Title** | Module Listing |
| **Priority** | P1 |
| **Priority Rationale** | Enables developers and agents to discover available modules. Without listing, users must know module IDs a priori. |
| **Source** | Feature Spec FE-04 FR-04-01 |

**Description:** The system shall provide an `apcore-cli list` subcommand that displays all modules in the Registry as a formatted table. The table shall contain columns for Module ID, Description (first 80 characters), and Tags. The system shall invoke `Registry.list()` to retrieve the module set.

**Actors:** Developer, AI Agent

**Preconditions:**
- The Registry is loaded with at least zero modules.

**Main Flow:**
1. The user invokes `apcore-cli list`.
2. The system shall invoke `Registry.list()` to retrieve all module definitions.
3. The system shall format the results as a table with columns: `ID`, `Description`, `Tags`.
4. The Description column shall display the first 80 characters of the module's `description` field, appending `...` if truncated.
5. The Tags column shall display a comma-separated list of the module's tags.
6. The system shall render the table using `rich.table.Table` when output format is `table`.
7. The system shall write the table to stdout and exit with code 0.

**Alternative Flows:**

- **AF-1: Empty Registry.** If the Registry contains zero modules, the system shall display an empty table with headers and a note: `"No modules found."` and exit with code 0.
- **AF-2: JSON Output Format.** If `--format json` is specified (or the default for non-TTY per FR-DISC-004), the system shall output a JSON array of objects with keys `id`, `description`, `tags`.

**Postconditions:**
- Module list is written to stdout. Exit code is 0.

**Acceptance Criteria:**

- **AC-1:** Given a Registry with modules `math.add` (tags: `math, core`) and `text.summarize` (tags: `text`), when the user runs `apcore-cli list`, then stdout shall contain a table with both module IDs, their descriptions, and tags.
- **AC-2:** Given an empty Registry, when the user runs `apcore-cli list`, then stdout shall contain "No modules found" and the process shall exit with code 0.
- **AC-3:** Given a module with a 120-character description, when listed, then the Description column shall show 80 characters followed by `...`.

---

#### FR-DISC-002: Tag Filtering

| Field | Value |
|-------|-------|
| **ID** | FR-DISC-002 |
| **Title** | Tag Filtering |
| **Priority** | P1 |
| **Priority Rationale** | Enables focused discovery in large registries. AND logic prevents flooding results and matches developer expectations. |
| **Source** | Feature Spec FE-04 FR-04-03 |

**Description:** The system shall accept one or more `--tag` flags on the `list` subcommand. When tags are provided, the system shall filter the module list to include only modules whose tag set contains ALL specified tags (AND logic). The `--tag` flag shall be repeatable.

**Actors:** Developer, AI Agent

**Preconditions:**
- The `list` subcommand is invoked with at least one `--tag` flag.

**Main Flow:**
1. The user invokes `apcore-cli list --tag math --tag core`.
2. The system shall collect all `--tag` values into a set: `{math, core}`.
3. The system shall invoke `Registry.list(tags=["math", "core"])` or filter the full list by tag intersection.
4. The system shall include only modules where `module.tags` is a superset of the specified tags.
5. The system shall display the filtered results per FR-DISC-001.

**Alternative Flows:**

- **AF-1: No Matching Modules.** If no modules match all specified tags, the system shall display an empty table with the note: `"No modules found matching tags: {tag_list}."` and exit with code 0.
- **AF-2: Tag Does Not Exist.** If a specified tag does not exist on any module, this is not an error; the result is simply an empty set (per AF-1).

**Postconditions:**
- Only modules matching ALL specified tags are displayed.

**Acceptance Criteria:**

- **AC-1:** Given modules `math.add` (tags: `math, core`) and `text.summarize` (tags: `text`), when the user runs `apcore-cli list --tag math`, then only `math.add` shall appear.
- **AC-2:** Given the same modules, when the user runs `apcore-cli list --tag math --tag core`, then only `math.add` shall appear (it has both tags).
- **AC-3:** Given the same modules, when the user runs `apcore-cli list --tag math --tag text`, then no modules shall appear (no module has both tags).

---

#### FR-DISC-003: Module Description

| Field | Value |
|-------|-------|
| **ID** | FR-DISC-003 |
| **Title** | Module Description Display |
| **Priority** | P1 |
| **Priority Rationale** | Enables detailed module inspection before execution. Critical for both developer understanding and agent capability assessment. |
| **Source** | Feature Spec FE-04 FR-04-02, FR-04-04 |

**Description:** The system shall provide an `apcore-cli describe <module_id>` subcommand that displays the full metadata for a single module. The output shall include the module's core fields (description, input_schema, output_schema), annotations (readonly, destructive, idempotent, requires_approval), and extension metadata (x-when-to-use, x-preconditions). JSON schemas shall be rendered with syntax highlighting using `rich.syntax` when the output format is `table`.

**Actors:** Developer, AI Agent

**Preconditions:**
- The Registry is loaded.

**Main Flow:**
1. The user invokes `apcore-cli describe <module_id>`.
2. The system shall validate `<module_id>` conforms to the Canonical ID format.
3. The system shall look up `<module_id>` in the Registry via `Registry.get_definition()` and `Registry.get_schema()`.
4. The system shall format and display:
   - **Core**: Module ID, description (full text), input_schema (syntax-highlighted JSON), output_schema (syntax-highlighted JSON).
   - **Annotations**: readonly, destructive, idempotent, requires_approval, and any other annotation fields.
   - **Extension**: All `x-` prefixed metadata fields.
5. The system shall write the output to stdout and exit with code 0.

**Alternative Flows:**

- **AF-1: Module Not Found.** If `<module_id>` is not in the Registry, the system shall write to stderr: `Error: Module '{module_id}' not found.` and exit with code 44 (`MODULE_NOT_FOUND`).
- **AF-2: JSON Output Format.** If `--format json` is specified, the system shall output the module metadata as a single JSON object without syntax highlighting.
- **AF-3: Module Without Optional Fields.** If the module lacks `output_schema`, annotations, or extension metadata, the system shall omit those sections from the display rather than showing empty sections.

**Postconditions:**
- Full module metadata is displayed. Exit code is 0 or 44.

**Acceptance Criteria:**

- **AC-1:** Given module `math.add` with `input_schema: {properties: {a: {type: integer}, b: {type: integer}}}`, when the user runs `apcore-cli describe math.add`, then stdout shall contain the module ID, description, and the input schema.
- **AC-2:** Given module `math.add` with `annotations.requires_approval: true`, when described, then stdout shall display `requires_approval: true`.
- **AC-3:** Given module ID `non.existent`, when the user runs `apcore-cli describe non.existent`, then stderr shall contain "not found" and the process shall exit with code 44.

---

#### FR-DISC-004: Output Format Selection

| Field | Value |
|-------|-------|
| **ID** | FR-DISC-004 |
| **Title** | Output Format Selection |
| **Priority** | P1 |
| **Priority Rationale** | AI agents require structured JSON output. Developers prefer readable tables. TTY-adaptive defaults eliminate the need for explicit flags in common cases. |
| **Source** | Feature Spec FE-04 FR-04-05 |

**Description:** The system shall accept a `--format` flag on `list` and `describe` subcommands with allowed values `table` and `json`. The default shall be TTY-adaptive: `table` when `sys.stdout.isatty()` returns `True`, `json` when `sys.stdout.isatty()` returns `False`. The `--format` flag shall override the TTY-adaptive default.

**Actors:** Developer, AI Agent

**Preconditions:**
- The `list` or `describe` subcommand is invoked.

**Main Flow:**
1. The system shall check for a `--format` flag.
2. If `--format table` is specified, render output as a formatted table using `rich`.
3. If `--format json` is specified, render output as a JSON document (array for `list`, object for `describe`) with 2-space indentation.
4. If no `--format` is specified:
   a. If `sys.stdout.isatty()` is `True`, default to `table`.
   b. If `sys.stdout.isatty()` is `False`, default to `json`.

**Alternative Flows:**

- **AF-1: Invalid Format Value.** If the user provides a value other than `table` or `json`, Click shall reject the input with an error listing the allowed choices and exit with code 2.

**Postconditions:**
- Output is rendered in the selected or defaulted format.

**Acceptance Criteria:**

- **AC-1:** Given a TTY session, when the user runs `apcore-cli list` without `--format`, then the output shall be a formatted table.
- **AC-2:** Given a non-TTY context (e.g., piped output), when the agent runs `apcore-cli list` without `--format`, then the output shall be valid JSON.
- **AC-3:** Given a TTY session, when the user runs `apcore-cli list --format json`, then the output shall be valid JSON (overriding the TTY default).
- **AC-4:** Given `--format yaml` is provided, then Click shall reject the value and exit with code 2.

---

### 5.5 Security (SEC)

#### FR-SEC-001: API Key Authentication

| Field | Value |
|-------|-------|
| **ID** | FR-SEC-001 |
| **Title** | API Key Authentication |
| **Priority** | P1 |
| **Priority Rationale** | Required for remote registry access. Without authentication, remote registries cannot verify caller identity. |
| **Source** | User requirements (security stack) |

**Description:** The system shall authenticate with remote registries using API keys. The API key shall be resolved using the configuration precedence (FR-DISP-005): CLI flag `--api-key`, environment variable `APCORE_AUTH_API_KEY`, or configuration file `apcore.yaml` key `auth.api_key`. The system shall transmit the API key in the `Authorization` header as `Bearer {key}` when making remote registry requests.

**Actors:** Developer, AI Agent

**Preconditions:**
- A remote registry endpoint is configured.
- An API key is available through one of the configuration sources.

**Main Flow:**
1. The system shall resolve the API key using the configuration precedence order.
2. The system shall include the API key as `Authorization: Bearer {key}` in all HTTP requests to the remote registry.
3. The system shall proceed with the Registry operation (list, get_definition, etc.).

**Alternative Flows:**

- **AF-1: No API Key Configured.** If a remote registry is configured but no API key is available from any source, the system shall write to stderr: `Error: Remote registry requires authentication. Set --api-key, APCORE_AUTH_API_KEY, or auth.api_key in config.` and exit with code 77 (`ACL_DENIED`).
- **AF-2: Authentication Rejected.** If the remote registry responds with HTTP 401 or 403, the system shall write to stderr: `Error: Authentication failed. Verify your API key.` and exit with code 77 (`ACL_DENIED`).
- **AF-3: Local Registry Only.** If no remote registry is configured (local-only mode), the system shall not require or use an API key.

**Postconditions:**
- Remote registry requests include valid authentication credentials.

**Acceptance Criteria:**

- **AC-1:** Given a remote registry and `APCORE_AUTH_API_KEY=abc123`, when the system accesses the registry, then HTTP requests shall include `Authorization: Bearer abc123`.
- **AC-2:** Given a remote registry and no API key configured, when the system attempts access, then stderr shall contain "requires authentication" and the process shall exit with code 77.
- **AC-3:** Given a local-only registry, when the system operates, then no API key shall be required.

---

#### FR-SEC-002: Encrypted Configuration

| Field | Value |
|-------|-------|
| **ID** | FR-SEC-002 |
| **Title** | Encrypted Configuration Storage |
| **Priority** | P1 |
| **Priority Rationale** | Prevents credential exposure in plaintext config files on shared or compromised filesystems. |
| **Source** | User requirements (security stack) |

**Description:** The system shall encrypt sensitive values (API keys, tokens, secrets) when persisting them to the configuration file `apcore.yaml`. The system shall use the operating system's native keyring (via `keyring` library) as the primary storage mechanism. If the keyring is unavailable, the system shall encrypt values using AES-256-GCM with a machine-derived key and store the ciphertext in the config file with an `enc:` prefix.

**Actors:** System (internal)

**Preconditions:**
- A sensitive value is being written to the configuration file.

**Main Flow:**
1. The system shall detect if the OS keyring is available.
2. If available, the system shall store the sensitive value in the keyring under the service name `apcore-cli` and key name matching the config key (e.g., `auth.api_key`).
3. The config file shall contain a reference: `auth.api_key: "keyring:auth.api_key"`.
4. When reading the value, the system shall detect the `keyring:` prefix and retrieve from the OS keyring.

**Alternative Flows:**

- **AF-1: Keyring Unavailable.** If the OS keyring is not available (headless server, container), the system shall encrypt the value using AES-256-GCM with a machine-derived key (hostname + user + salt), store as `auth.api_key: "enc:base64ciphertext"`, and log a WARNING: `"OS keyring unavailable. Using file-based encryption."`.
- **AF-2: Decryption Failure.** If the system cannot decrypt a stored value (key mismatch, corrupted data), the system shall write to stderr: `Error: Failed to decrypt configuration value '{key}'. Re-configure with 'apcore-cli config set {key}'.` and exit with code 47 (`CONFIG_INVALID`).

**Postconditions:**
- Sensitive values are not stored in plaintext in the configuration file.

**Acceptance Criteria:**

- **AC-1:** Given a sensitive value `auth.api_key` is configured, when inspecting `apcore.yaml`, then the value shall NOT appear in plaintext; it shall be prefixed with `keyring:` or `enc:`.
- **AC-2:** Given the OS keyring is available, when the system reads `auth.api_key`, then it shall retrieve the value from the keyring and use it for authentication.
- **AC-3:** Given the system cannot decrypt a stored value, when the system reads the config, then stderr shall contain "Failed to decrypt" and the process shall exit with code 47.

---

#### FR-SEC-003: Audit Logging

| Field | Value |
|-------|-------|
| **ID** | FR-SEC-003 |
| **Title** | Audit Logging |
| **Priority** | P1 |
| **Priority Rationale** | Provides accountability and forensic capability for all module executions, essential for compliance and debugging. |
| **Source** | User requirements (security stack) |

**Description:** The system shall log every module execution to an append-only audit log file. Each audit entry shall contain: ISO 8601 timestamp, invoking user (from `os.getlogin()` or `USER` env var), module Canonical ID, input hash (SHA-256 of the serialized input, not the raw input), execution result status (`success` or `error`), exit code, and execution duration in milliseconds.

**Actors:** System (internal)

**Preconditions:**
- A module execution has been initiated (after approval, if applicable).

**Main Flow:**
1. The system shall record the start timestamp.
2. After module execution completes (success or failure), the system shall compute the SHA-256 hash of the serialized input JSON.
3. The system shall write an audit log entry in JSON Lines format to the audit log file (default: `~/.apcore-cli/audit.jsonl`).
4. Each entry shall contain: `{"timestamp": "...", "user": "...", "module_id": "...", "input_hash": "...", "status": "success|error", "exit_code": 0, "duration_ms": 42}`.

**Alternative Flows:**

- **AF-1: Audit Log Write Failure.** If the audit log file cannot be written (permissions, disk full), the system shall log a WARNING to stderr: `"Warning: Could not write audit log: {reason}."` and continue execution. Audit log failure shall NOT prevent module execution.
- **AF-2: User Cannot Be Determined.** If `os.getlogin()` raises an exception and `USER` env var is unset, the system shall record `"user": "unknown"`.

**Postconditions:**
- An audit entry is appended to the audit log file.

**Acceptance Criteria:**

- **AC-1:** Given a successful execution of `math.add`, when the audit log is inspected, then it shall contain a JSON line with `module_id: "math.add"`, `status: "success"`, `exit_code: 0`, and a valid ISO 8601 timestamp.
- **AC-2:** Given a failed execution, when the audit log is inspected, then it shall contain `status: "error"` and the corresponding exit code.
- **AC-3:** Given the audit log file is not writable, when a module executes, then the module shall still execute successfully and a warning shall appear on stderr.

---

#### FR-SEC-004: Execution Sandboxing

| Field | Value |
|-------|-------|
| **ID** | FR-SEC-004 |
| **Title** | Execution Sandboxing |
| **Priority** | P2 |
| **Priority Rationale** | Defense-in-depth measure. Important for running untrusted modules but not blocking for initial release with trusted local modules. |
| **Source** | User requirements (security stack) |

**Description:** The system shall isolate module execution from the host system by restricting the module's access to the filesystem, network, and environment variables. The system shall use process-level isolation (subprocess with restricted environment) for module execution when the `--sandbox` flag is provided or `APCORE_CLI_SANDBOX=1` is set.

**Actors:** System (internal)

**Preconditions:**
- Sandbox mode is activated via `--sandbox` flag or `APCORE_CLI_SANDBOX=1`.

**Main Flow:**
1. The system shall create a restricted subprocess environment containing only explicitly allowed environment variables.
2. The system shall set the working directory to a temporary directory.
3. The system shall invoke the module within this restricted environment.
4. After execution, the system shall clean up the temporary directory.

**Alternative Flows:**

- **AF-1: Sandbox Not Requested.** If neither `--sandbox` nor `APCORE_CLI_SANDBOX=1` is active, the system shall execute the module in the current process environment (no isolation).
- **AF-2: Platform Not Supported.** If the current platform does not support the sandboxing mechanism, the system shall log a WARNING: `"Sandboxing not available on this platform. Executing without isolation."` and proceed without sandbox.

**Postconditions:**
- Module execution is isolated (when sandbox is active) or runs in the normal environment.

**Acceptance Criteria:**

- **AC-1:** Given `--sandbox` is provided, when a module attempts to read `HOME` environment variable, then it shall not find it in the sandbox environment.
- **AC-2:** Given `--sandbox` is provided, when a module writes a file, then it shall be written to the temporary directory, not the user's working directory.
- **AC-3:** Given no sandbox flag, when a module executes, then it shall run with access to the normal environment.

---

### 5.6 Shell Integration (SHELL)

#### FR-SHELL-001: Shell Completion

| Field | Value |
|-------|-------|
| **ID** | FR-SHELL-001 |
| **Title** | Shell Completion Script Generation |
| **Priority** | P2 |
| **Priority Rationale** | Quality-of-life improvement for developers. Not blocking for MVP but significantly improves usability. |
| **Source** | User requirements (shell ecosystem integration) |

**Description:** The system shall generate shell completion scripts for Bash, Zsh, and Fish shells. The generated scripts shall provide completion for subcommands (`exec`, `list`, `describe`), module Canonical IDs (dynamic, from Registry), and module-specific flags (dynamic, from `input_schema`). The system shall provide an `apcore-cli completion <shell>` subcommand where `<shell>` is one of `bash`, `zsh`, `fish`.

**Actors:** Developer

**Preconditions:**
- The Registry is loaded.

**Main Flow:**
1. The user invokes `apcore-cli completion bash` (or `zsh` or `fish`).
2. The system shall generate a shell completion script appropriate for the specified shell.
3. The script shall include completions for:
   - Top-level subcommands: `exec`, `list`, `describe`, `completion`.
   - Module IDs: dynamically enumerated from the Registry.
   - Module flags: dynamically generated from each module's `input_schema`.
4. The system shall write the script to stdout and exit with code 0.
5. The user shall source the script in their shell profile.

**Alternative Flows:**

- **AF-1: Unsupported Shell.** If the user specifies a shell other than `bash`, `zsh`, or `fish`, the system shall write to stderr: `Error: Unsupported shell '{shell}'. Supported: bash, zsh, fish.` and exit with code 2.

**Postconditions:**
- A shell completion script is written to stdout.

**Acceptance Criteria:**

- **AC-1:** Given the user runs `apcore-cli completion bash`, then stdout shall contain a valid Bash completion script that includes `exec`, `list`, and `describe` as completable subcommands.
- **AC-2:** Given the user runs `apcore-cli completion invalid`, then stderr shall contain "Unsupported shell" and the process shall exit with code 2.
- **AC-3:** Given the Registry contains `math.add`, when the generated Bash completion is sourced and the user types `apcore-cli exec math.<TAB>`, then `math.add` shall appear as a completion candidate.

---

#### FR-SHELL-002: Man Page Generation

| Field | Value |
|-------|-------|
| **ID** | FR-SHELL-002 |
| **Title** | Man Page Generation |
| **Priority** | P2 |
| **Priority Rationale** | Standard Unix convention for CLI documentation. Not blocking but provides integration with `man` command. |
| **Source** | User requirements (shell ecosystem integration) |

**Description:** The system shall generate man pages in roff format from module metadata and CLI command definitions. The system shall provide an `apcore-cli man <command>` subcommand or a build-time script that generates man pages for installation. Man pages shall include the command synopsis, description, options, examples, and exit codes.

**Actors:** Developer

**Preconditions:**
- The Registry is loaded (for module-specific man pages).

**Main Flow:**
1. The user invokes `apcore-cli man exec` or `apcore-cli man list`.
2. The system shall generate a man page in roff format for the specified command.
3. The man page shall include: NAME, SYNOPSIS, DESCRIPTION, OPTIONS, EXIT CODES, SEE ALSO.
4. The system shall write the roff output to stdout.
5. The user may pipe to `man -l -` for immediate display.

**Alternative Flows:**

- **AF-1: Unknown Command.** If the specified command does not exist, the system shall write to stderr: `Error: Unknown command '{command}'.` and exit with code 2.

**Postconditions:**
- A man page in roff format is written to stdout.

**Acceptance Criteria:**

- **AC-1:** Given the user runs `apcore-cli man exec`, then stdout shall contain roff-formatted text including `.TH` (title heading) and the command name.
- **AC-2:** Given the user runs `apcore-cli man exec | man -l -`, then the man page shall render correctly in the terminal.

---

### 5.7 CRUD Matrix

| Entity | Create | Read | Update | Delete |
|--------|--------|------|--------|--------|
| **Module Registry** | FR-DISP-003 (discover/load from directory) | FR-DISP-002 (lookup module), FR-DISC-001 (list), FR-DISC-003 (describe) | N/A (read-only in CLI; managed externally) | N/A (managed externally) |
| **Module Definition** | N/A (defined by module authors) | FR-DISP-002 (resolve for exec), FR-DISC-003 (describe), FR-SCHEMA-001 (parse schema) | N/A (immutable at runtime) | N/A |
| **CLI Configuration** | FR-SEC-002 (initial config write) | FR-DISP-005 (precedence resolution), FR-SEC-001 (API key retrieval) | FR-SEC-002 (re-encryption) | N/A (manual file management) |
| **Audit Log** | FR-SEC-003 (append entry on each execution) | External tooling (not in CLI scope) | N/A (append-only) | N/A (retention managed externally) |
| **Approval Record** | FR-APPR-002 (user approves/denies), FR-APPR-004 (bypass logged) | FR-SEC-003 (recorded in audit log) | N/A (immutable) | N/A |

---

## 6. Non-Functional Requirements

### 6.1 Performance

#### NFR-PERF-001: CLI Startup Time

| Field | Value |
|-------|-------|
| **ID** | NFR-PERF-001 |
| **Title** | CLI Startup Time |
| **Metric** | Wall-clock time from process start to help text output |
| **Target** | < 100 milliseconds |
| **Measurement Method** | `time apcore-cli --help` averaged over 10 runs with a warm filesystem cache, using a Registry containing 100 modules |
| **Threshold Rationale** | Tech Design v1.0 section 3.2 Goal: "Minimal overhead (<100ms) for one-shot executions." ideas/draft.md NFR-001: "Execution overhead < 100ms." Sub-100ms ensures CLI feels instantaneous to developers and does not bottleneck agent workflows. |

#### NFR-PERF-002: Module Execution Overhead

| Field | Value |
|-------|-------|
| **ID** | NFR-PERF-002 |
| **Title** | Module Execution Overhead |
| **Metric** | Wall-clock time spent in apcore-cli adapter code (excluding module execution time) |
| **Target** | < 50 milliseconds |
| **Measurement Method** | Execute a no-op module (returns immediately) via `apcore-cli exec noop` and measure total time minus module execution time. Average over 10 runs. |
| **Threshold Rationale** | The adapter layer must not introduce perceptible latency. At 50ms overhead + <100ms startup, total CLI overhead stays under 150ms, ensuring the CLI remains faster than JSON-RPC/MCP alternatives for simple invocations (as stated in ideas/draft.md section 2: "Token Bloat" problem). |

#### NFR-PERF-003: Registry Capacity

| Field | Value |
|-------|-------|
| **ID** | NFR-PERF-003 |
| **Title** | Registry Capacity |
| **Metric** | Number of modules the Registry can index without degradation of startup time target (NFR-PERF-001) |
| **Target** | 1,000 modules |
| **Measurement Method** | Generate a synthetic Registry with 1,000 module definitions and measure startup time. Verify it remains < 100ms. |
| **Threshold Rationale** | Feature Spec FE-01 section 3.1: "Should handle up to 1,000 modules." This provides 10x headroom over the 100-module baseline, accommodating growth of enterprise module libraries. |

---

### 6.2 Security

#### NFR-SEC-001: Credential Storage

| Field | Value |
|-------|-------|
| **ID** | NFR-SEC-001 |
| **Title** | No Plaintext Credential Storage |
| **Metric** | Number of plaintext secrets found in configuration files |
| **Target** | 0 |
| **Measurement Method** | Automated scan of `apcore.yaml` and `~/.apcore-cli/` for known secret patterns (API keys, tokens) using a secret detection tool. Run as part of CI. |
| **Threshold Rationale** | User requirements specify "encrypted config." OWASP guidelines prohibit plaintext credential storage. Any plaintext secret is a critical vulnerability. |

#### NFR-SEC-002: Input Validation

| Field | Value |
|-------|-------|
| **ID** | NFR-SEC-002 |
| **Title** | Input Validation Against JSON Schema |
| **Metric** | Percentage of input fields validated against JSON Schema before module execution |
| **Target** | 100% |
| **Measurement Method** | Unit tests verifying that for every module execution path, `jsonschema.validate()` is invoked before `Executor.call()`. Code review of the execution pipeline. |
| **Threshold Rationale** | Unvalidated input bypasses the module's declared contract, potentially causing injection vulnerabilities or unexpected behavior. PROTOCOL_SPEC requires schema validation at each adapter boundary. |

#### NFR-SEC-003: STDIN Buffer Limit

| Field | Value |
|-------|-------|
| **ID** | NFR-SEC-003 |
| **Title** | STDIN Buffer Limit |
| **Metric** | Maximum STDIN size accepted without `--large-input` flag |
| **Target** | 10 MB |
| **Measurement Method** | Pipe input exceeding 10 MB to `apcore-cli exec module --input -` without `--large-input` and verify rejection with exit code 2. |
| **Threshold Rationale** | Tech Design v1.0 section 8.2: "STDIN > 10MB: Reject with GENERAL_INVALID_INPUT." The 10MB limit prevents out-of-memory conditions in memory-constrained environments while accommodating typical JSON payloads (99th percentile < 1MB). |

---

### 6.3 Reliability

#### NFR-REL-001: Deterministic Exit Codes

| Field | Value |
|-------|-------|
| **ID** | NFR-REL-001 |
| **Title** | Deterministic Exit Codes |
| **Metric** | Percentage of error conditions that produce the documented exit code |
| **Target** | 100% |
| **Measurement Method** | Integration test suite covering all error taxonomy entries (Tech Design section 8.1). Each test triggers a specific error condition and asserts the exit code. |
| **Threshold Rationale** | PROTOCOL_SPEC section 8 mandates specific error codes. AI agents rely on exit codes for programmatic error handling. Any non-deterministic exit code breaks agent workflows. |

#### NFR-REL-002: Graceful Degradation

| Field | Value |
|-------|-------|
| **ID** | NFR-REL-002 |
| **Title** | Graceful Degradation on Missing Extensions |
| **Metric** | System crash count when extensions directory is missing or corrupt |
| **Target** | 0 crashes (all failures produce a user-facing error message and a non-zero exit code) |
| **Measurement Method** | Test with: (1) missing directory, (2) empty directory, (3) directory with corrupt module files, (4) directory with permission denied. Verify no Python tracebacks reach stdout/stderr. |
| **Threshold Rationale** | Feature Spec FE-01 requires graceful handling of missing/corrupt extensions. Python tracebacks exposed to users degrade trust and are unusable by AI agents. |

---

### 6.4 Maintainability

#### NFR-MNT-001: Test Coverage

| Field | Value |
|-------|-------|
| **ID** | NFR-MNT-001 |
| **Title** | Unit Test Coverage |
| **Metric** | Line coverage percentage measured by `coverage.py` |
| **Target** | >= 80% |
| **Measurement Method** | Run `pytest --cov=apcore_cli --cov-report=term-missing` and verify the total coverage percentage. |
| **Threshold Rationale** | Industry standard for production CLI tools. 80% provides confidence in regression detection while allowing pragmatic exclusion of trivial code paths (main entry point, platform-specific branches). |

#### NFR-MNT-002: Structured Logging

| Field | Value |
|-------|-------|
| **ID** | NFR-MNT-002 |
| **Title** | Structured Logging |
| **Metric** | Percentage of components with dedicated logger namespace and configurable verbosity |
| **Target** | 100% of components (dispatcher, schema, approval, discovery) |
| **Measurement Method** | Code review verifying each component uses `logging.getLogger("apcore_cli.{component}")` and respects `APCORE_CLI_LOGGING_LEVEL` (CLI-specific, highest priority) / `APCORE_LOGGING_LEVEL` (global fallback) / `--log-level`. |
| **Threshold Rationale** | Tech Design v1.0 section 8.4 specifies structured logging with namespace `apcore_cli`. Consistent logging enables debugging in production and integration with centralized log aggregation. |

---

### 6.5 Portability

#### NFR-PRT-001: OS Compatibility

| Field | Value |
|-------|-------|
| **ID** | NFR-PRT-001 |
| **Title** | Cross-OS Compatibility |
| **Metric** | Number of supported operating systems where the full test suite passes |
| **Target** | 3 (Linux, macOS, Windows) with Python >= 3.11 |
| **Measurement Method** | CI matrix running the full test suite on Ubuntu latest, macOS latest, and Windows latest with Python 3.11 and 3.12. |
| **Threshold Rationale** | User requirements specify "Linux, macOS, Windows (Python 3.11+)." Python's cross-platform nature enables this with minimal platform-specific code. Requires 3.11+ to align with apcore >= 0.13.0. |

#### NFR-PRT-002: Terminal Compatibility

| Field | Value |
|-------|-------|
| **ID** | NFR-PRT-002 |
| **Title** | Terminal Emulator Compatibility |
| **Metric** | Number of terminal emulators where output renders correctly (no broken characters, correct colors) |
| **Target** | Correct rendering in: Terminal.app, iTerm2, GNOME Terminal, Windows Terminal, VS Code integrated terminal |
| **Measurement Method** | Manual verification of `list` (table output) and `describe` (syntax-highlighted JSON) in each terminal emulator. |
| **Threshold Rationale** | The `rich` library provides cross-terminal compatibility, but edge cases exist with color support and Unicode. The five listed terminals cover the majority of developer usage. |

---

### 6.6 Usability

#### NFR-USB-001: Help Text Quality

| Field | Value |
|-------|-------|
| **ID** | NFR-USB-001 |
| **Title** | Help Text Coverage |
| **Metric** | Percentage of commands and flags with non-empty help text |
| **Target** | 100% |
| **Measurement Method** | Parse the output of `apcore-cli --help`, `apcore-cli exec --help`, `apcore-cli list --help`, `apcore-cli describe --help` and verify every listed command/flag has associated descriptive text (non-empty string after the flag name). |
| **Threshold Rationale** | User requirements specify "every command and flag has descriptive help text." Help text is the primary discoverability mechanism for both human developers and AI agents introspecting CLI capabilities. |

#### NFR-USB-002: Error Message Quality

| Field | Value |
|-------|-------|
| **ID** | NFR-USB-002 |
| **Title** | Actionable Error Messages |
| **Metric** | Percentage of error messages that include (1) what went wrong, (2) a suggested fix or next step |
| **Target** | 100% of error exits (codes 1, 2, 44, 45, 46, 47, 48, 77, 130) |
| **Measurement Method** | Review all error message strings in the codebase. Each must contain a description of the problem AND at least one of: a CLI flag to try, an env var to set, a config to check, or an action to take. |
| **Threshold Rationale** | User requirements specify "user-friendly error messages with suggested fixes." Error messages without actionable guidance increase support burden and degrade developer experience. |

---

## 7. Data Requirements

### 7.1 Data Model

```
+-------------------+       +---------------------+       +------------------+
| Module Registry   |       | Module Definition   |       | Input Schema     |
|-------------------|       |---------------------|       |------------------|
| extensions_path   |1    *| canonical_id (PK)   |1    1| properties       |
| module_count      +-------+ description          +-------+ required[]       |
| discovered_at     |       | tags[]               |       | $defs            |
+-------------------+       | annotations          |       | type             |
                            |   .requires_approval |       +------------------+
                            |   .readonly          |
                            |   .destructive       |       +------------------+
                            |   .idempotent        |       | Output Schema    |
                            | extension_metadata   |1    1|------------------|
                            | input_schema (FK)    +-------+ properties       |
                            | output_schema (FK)   |       | type             |
                            | enabled              |       +------------------+
                            +---------------------+

+-------------------+       +---------------------+
| CLI Configuration |       | Audit Log Entry     |
|-------------------|       |---------------------|
| extensions_root   |       | timestamp (ISO 8601)|
| auth.api_key      |       | user                |
| logging_level     |       | module_id           |
| sandbox_enabled   |       | input_hash (SHA-256)|
+-------------------+       | status              |
                            | exit_code           |
                            | duration_ms         |
                            +---------------------+
```

### 7.2 Data Dictionary

| Field | Type | Constraints | Description |
|-------|------|-------------|-------------|
| `canonical_id` | String | Pattern: `^[a-z][a-z0-9_]*(\.[a-z][a-z0-9_]*)*$`. Max 128 chars. | Unique dot-separated identifier for a module. |
| `description` | String | Max 4,096 characters. | Human-readable description of the module's purpose. |
| `tags` | Array of String | Each tag: `^[a-z][a-z0-9_-]*$`. Max 32 tags per module. | Categorization labels for discovery filtering. |
| `requires_approval` | Boolean | Default: `false`. | Whether the module requires HITL approval before execution. |
| `readonly` | Boolean | Default: `false`. | Whether the module only reads data (no side effects). |
| `destructive` | Boolean | Default: `false`. | Whether the module performs destructive operations. |
| `idempotent` | Boolean | Default: `false`. | Whether repeated execution produces the same result. |
| `extensions_root` | String (path) | Valid filesystem path. Max 4,096 characters. | Path to the apcore extensions directory. |
| `auth.api_key` | String (secret) | Stored encrypted (see FR-SEC-002). Max 512 characters. | API key for remote registry authentication. |
| `logging_level` | Enum | `DEBUG`, `INFO`, `WARNING`, `ERROR`. Default: `WARNING`. | Verbosity level for structured logging. |
| `timestamp` | String (ISO 8601) | Format: `YYYY-MM-DDTHH:MM:SS.fffZ`. | When the audit event occurred. |
| `input_hash` | String (hex) | 64 characters (SHA-256). | Hash of the serialized input for auditability without storing raw input. |
| `status` | Enum | `success`, `error`. | Outcome of the module execution. |
| `exit_code` | Integer | 0-255. | Unix exit code of the CLI process. |
| `duration_ms` | Integer | >= 0. | Module execution duration in milliseconds. |

---

## 8. External Interface Requirements

### 8.1 User Interfaces

The system's user interface is the command-line terminal. All interaction occurs through:

- **Command invocation**: `apcore-cli <subcommand> [options] [arguments]`.
- **Help display**: Formatted text output via Click's built-in help rendering and `rich` formatting.
- **Table output**: Module listing rendered as formatted tables via `rich.table.Table`.
- **JSON output**: Machine-readable JSON written to stdout for non-TTY consumers.
- **Approval prompts**: Interactive `[y/N]` prompts rendered via `click.confirm()` in TTY mode.
- **Error output**: Error messages written to stderr, never to stdout (to preserve pipe integrity).

### 8.2 Hardware Interfaces

Not applicable. The system runs as a software process and does not interface directly with hardware.

### 8.3 Software Interfaces

| Interface | Version | Protocol | Direction | Description |
|-----------|---------|----------|-----------|-------------|
| `apcore` | >= 0.13.0 | Python API | Outbound | Registry discovery, module metadata retrieval, Executor invocation. The system imports `apcore.Registry`, `apcore.Executor`, and the error hierarchy. |
| `click` | >= 8.0 | Python API | Internal | CLI command tree construction, argument parsing, help generation, interactive prompts. |
| `jsonschema` | >= 4.0 | Python API | Internal | Input validation against module JSON Schema definitions. |
| `rich` | >= 13.0 | Python API | Internal | Terminal output formatting including tables, syntax highlighting, and styled text. |
| `keyring` | >= 24.0 | Python API | Outbound | OS-native credential storage for encrypted configuration (FR-SEC-002). |

### 8.4 Communication Interfaces

| Channel | Direction | Format | Description |
|---------|-----------|--------|-------------|
| **stdin** | Input | UTF-8 JSON | Accepts piped JSON input when `--input -` is specified. Buffer limit: 10 MB (configurable). |
| **stdout** | Output | UTF-8 text or JSON | Module execution results, table displays, JSON output, man pages, completion scripts. |
| **stderr** | Output | UTF-8 text | Error messages, warnings, diagnostic output. Never mixed with stdout to preserve pipe integrity. |
| **Exit codes** | Output | Integer (0-255) | Deterministic exit codes per PROTOCOL_SPEC section 8. See Tech Design section 8.1 for full taxonomy. |
| **Environment variables** | Input | String key-value | Configuration via `APCORE_*` prefixed variables (see Tech Design section 8.3). |
| **Filesystem** | Input/Output | Files | Extensions directory (input), configuration file (input/output), audit log (output). |

---

## 9. Requirements Source

This SRS was developed in **standalone mode** without an upstream Product Requirements Document (PRD). Requirements were derived from the following sources:

| Source | Location | Contribution |
|--------|----------|-------------|
| **Tech Design v1.0** | `docs/tech-design.md` | Architecture decisions (ADR-01 through ADR-03), error taxonomy, environment variable conventions, component design, performance targets. |
| **Feature Specs** | `docs/features/*.md` | Functional requirements for Core Dispatcher (FE-01), Schema Parser (FE-02), Approval Gate (FE-03), and Discovery (FE-04). Boundary values and verification criteria. |
| **User Clarification Interview** | N/A (inline) | Security requirements (auth, encryption, audit, sandbox), shell integration (completion, man pages), user characteristics (developer + AI agent), performance constraints. |
| **ideas/draft.md** | `ideas/draft.md` | Original requirement IDs (FR-001 through FR-004, NFR-001 through NFR-002), problem statement, and validation rationale. |

**Traceability Note:** *To establish full traceability from business objectives through product requirements to software requirements, consider running `/spec-forge:prd` first, then re-running `/spec-forge:srs`.*

---

## 10. Appendix

### 10.1 Open Questions

| ID | Question | Status | Owner |
|----|----------|--------|-------|
| OQ-01 | What is the maximum module execution timeout? Tech Design mentions `MODULE_TIMEOUT` but no default value is specified. | Open | Engineering |
| OQ-02 | Should shell completion (FR-SHELL-001) use Click's built-in completion or a custom implementation for dynamic module ID completion? | Open | Engineering |
| OQ-03 | What is the retention policy for audit log entries (FR-SEC-003)? Should the CLI provide a `log rotate` command? | Open | Product |
| OQ-04 | Should the STDIN buffer limit (10 MB) be configurable via `apcore.yaml` in addition to the `--large-input` flag? | Open | Engineering |
| OQ-05 | What sandbox mechanism shall be used for FR-SEC-004 on Windows, where Unix-style process isolation is not directly available? | Open | Engineering |
| OQ-06 | Should `apcore-cli exec` output default to `table` (TTY) / `json` (non-TTY) like Discovery, or always output raw module result? | Open | Product |
| OQ-07 | How should `anyOf`/`oneOf` schema composition be presented to users when multiple mutually exclusive property sets exist? | Open | Engineering |

### 10.2 Glossary

| Term | Definition |
|------|-----------|
| **Adapter** | A component that translates between the apcore module interface and a specific communication channel (CLI, MCP, A2A). |
| **Cold Start** | The first invocation of the CLI process, with no cached state from previous runs. |
| **Exit Code** | A numeric value (0-255) returned by a Unix process to indicate success (0) or a specific error category. |
| **JSON Lines** | A format where each line is a valid JSON object, commonly used for append-only log files. |
| **Kebab-case** | A naming convention where words are separated by hyphens (e.g., `input-file`). |
| **Middleware Chain** | A sequence of processing steps (ACL, observability, approval) that wrap module execution in the apcore Executor. |
| **Roff** | The formatting language used by Unix `man` pages. |
| **Snake_case** | A naming convention where words are separated by underscores (e.g., `input_file`). |

### 10.3 Change Request Log

| CR ID | Date | Requestor | Description | Status |
|-------|------|-----------|-------------|--------|
| _(none)_ | — | — | — | — |
