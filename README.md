<div align="center">
  <img src="./apcore-cli-logo.svg" alt="apcore-cli logo" width="200"/>
</div>

# apcore-cli

[![License](https://img.shields.io/badge/License-Apache_2.0-blue.svg)](https://opensource.org/licenses/Apache-2.0)
[![Python](https://img.shields.io/badge/python-3.11%2B-blue)](https://github.com/aiperceivable/apcore-cli-python)
[![TypeScript](https://img.shields.io/badge/typescript-node%2018%2B-blue)](https://github.com/aiperceivable/apcore-cli-typescript)
[![Rust](https://img.shields.io/badge/rust-2021%20edition-orange)](https://github.com/aiperceivable/apcore-cli-rust)

**The CLI Adapter for apcore — Expose modules as high-performance, AI-perceivable command-line tools.**

> **Build once, invoke by Code, AI, or Terminal.**

| | |
|---|---|
| **Spec repo** | [github.com/aiperceivable/apcore-cli](https://github.com/aiperceivable/apcore-cli) |
| **Python SDK** | [github.com/aiperceivable/apcore-cli-python](https://github.com/aiperceivable/apcore-cli-python) |
| **TypeScript SDK** | [github.com/aiperceivable/apcore-cli-typescript](https://github.com/aiperceivable/apcore-cli-typescript) |
| **Rust SDK** | [github.com/aiperceivable/apcore-cli-rust](https://github.com/aiperceivable/apcore-cli-rust) |
| **apcore core** | [github.com/aiperceivable/apcore](https://github.com/aiperceivable/apcore) |

---

## What is this?

`apcore-cli` is the "inside-out" adapter for the apcore ecosystem. It takes your **apcore modules** and automatically exposes them as **CLI subcommands** — with zero code changes.

It serves as the terminal-native counterpart to `apcore-mcp` (Model Context Protocol) and `apcore-a2a` (Agent-to-Agent).

```
    apcore Module Registry            apcore-cli Adapter (grouped)
  ━━━━━━━━━━━━━━━━━━━━━━━━━━        ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  math.add(a, b)             ──┐
  math.sub(a, b)             ──┤──→  apcore-cli math add --a 1 --b 2
  system.health.summary()    ──┤──→  apcore-cli system health.summary
  git.commit(msg)            ──┤──→  apcore-cli list --tag math
  db.query(sql)              ──┘──→  apcore-cli describe math.add
```

---

## Getting Started (Beginner Guide)

### Prerequisites

- Python 3.11+
- An existing [apcore](https://github.com/aiperceivable/apcore) project with an extensions directory, **OR** the example modules included in the Python SDK

### Step 1: Install

```bash
pip install apcore-cli
```

### Step 2: Try it with example modules

```bash
# Clone the Python SDK (includes 8 example modules)
git clone https://github.com/aiperceivable/apcore-cli-python.git
cd apcore-cli-python
pip install -e ".[dev]"

# Point to the example extensions
export APCORE_EXTENSIONS_ROOT=examples/extensions

# Run your first module
apcore-cli math.add --a 5 --b 10
# {"sum": 15}
```

### Step 3: Explore available modules

```bash
# List all discovered modules
apcore-cli list

# Filter by tag
apcore-cli list --tag math

# See full details for a module
apcore-cli describe math.add
```

### Step 4: Try STDIN piping

```bash
# Pipe JSON input
echo '{"a": 100, "b": 200}' | apcore-cli math.add --input -
# {"sum": 300}

# CLI flags override STDIN values
echo '{"a": 1, "b": 2}' | apcore-cli math.add --input - --a 999
# {"sum": 1001}

# Chain with other tools
apcore-cli sysutil.info | jq '.os, .hostname'
```

### Step 5: Use with your own project

If you already have an apcore project with an extensions directory:

```bash
# One-time setup
export APCORE_EXTENSIONS_ROOT=./extensions

# Or pass it directly
apcore-cli --extensions-dir ./extensions math.add --a 42 --b 58
```

That's it. No code changes, no configuration files, no new dependencies in your project.

### Step 6: Write your first module

Create a Python file in your extensions directory:

```python
# extensions/greet/hello.py
from pydantic import BaseModel

class Input(BaseModel):
    name: str
    greeting: str = "Hello"

class Output(BaseModel):
    message: str

class GreetHello:
    input_schema = Input
    output_schema = Output
    description = "Greet someone by name"

    def execute(self, inputs, context=None):
        return {"message": f"{inputs['greeting']}, {inputs['name']}!"}
```

Run it:

```bash
apcore-cli greet.hello --name World
# {"message": "Hello, World!"}
```

---

## Adding Custom Commands

### Fastest way (30 seconds)

```bash
apcore-cli init module ops.deploy -d "Deploy to environment"
# Edit the generated file, add your logic
```

### Zero-import way (convention discovery)

Drop a plain Python function into `commands/`:

```python
# commands/deploy.py
def deploy(env: str, tag: str = "latest") -> dict:
    """Deploy the app to the given environment."""
    return {"status": "deployed", "env": env}
```

Then run with `--commands-dir commands/`:

```bash
apcore-cli --commands-dir commands/ deploy deploy --env prod
```

The `init module` command supports three styles via `--style`:
- **decorator** (default) — generates a `@module`-decorated class in the extensions directory
- **convention** — generates a plain Python function in the commands directory
- **binding** — generates a `.binding.yaml` file

---

## Key Features

- **Grouped Commands** -- Modules with dots in their names are auto-grouped into nested subcommands (`apcore-cli product list` instead of `apcore-cli product.list`); `display.cli.group` in binding.yaml overrides the auto-detected group
- **Display Overlay** -- `metadata["display"]["cli"]` controls CLI command names, descriptions, and guidance per module (§5.13); set via `binding_path` in `create_cli()` / `fastapi-apcore`
- **Zero-Config Routing** -- Automatically maps module IDs (e.g., `math.add`) to CLI commands
- **Schema-Driven Args** -- Uses `input_schema` to generate CLI arguments, types, and validation
- **Boolean Flag Pairs** -- `--verbose` / `--no-verbose` from `"type": "boolean"` schema properties
- **Enum Choices** -- `"enum": ["json", "csv"]` becomes `--format json` with Click validation
- **STDIN Piping** -- `--input -` reads JSON from STDIN, CLI flags override for duplicate keys
- **TTY-Adaptive Output** -- Rich tables for terminals, JSON for pipes (configurable via `--format`)
- **Approval Gate** -- TTY-aware HITL prompts for modules with `requires_approval: true`, with `--yes` bypass and 60s timeout
- **Schema Validation** -- Inputs validated against JSON Schema before execution, with `$ref`/`allOf`/`anyOf`/`oneOf` resolution
- **Security** -- API key auth (keyring + AES-256-GCM), append-only audit logging, subprocess sandboxing
- **Shell Completions** -- `apcore-cli completion bash|zsh|fish` generates completion scripts with dynamic module ID completion
- **Man Pages** -- `apcore-cli man <command>` generates roff-formatted man pages
- **Audit Logging** -- All executions logged to `~/.apcore-cli/audit.jsonl` with SHA-256 input hashing

---

## CLI Reference

```
apcore-cli [OPTIONS] COMMAND [ARGS]
```

### Global Options

| Option | Default | Description |
|--------|---------|-------------|
| `--extensions-dir` | `./extensions` | Path to apcore extensions directory |
| `--log-level` | `WARNING` | Logging: `DEBUG`, `INFO`, `WARNING`, `ERROR` |
| `--version` | | Show version and exit |
| `--help` | | Show help and exit |

### Built-in Commands

| Command | Description |
|---------|-------------|
| `list` | List available modules with optional tag filtering |
| `describe <module_id>` | Show full module metadata and schemas |
| `exec <module_id>` | Internal routing alias for module execution (modules are also available as direct top-level subcommands) |
| `completion <shell>` | Generate shell completion script (bash/zsh/fish) |
| `man <command>` | Generate man page in roff format |

### Module Execution Options

When executing a module (e.g. `apcore-cli math.add`), these built-in options are always available:

| Option | Description |
|--------|-------------|
| `--input -` | Read JSON input from STDIN |
| `--yes` / `-y` | Bypass approval prompts |
| `--large-input` | Allow STDIN input larger than 10MB |
| `--format` | Output format: `json` or `table` |
| `--sandbox` | Run module in subprocess sandbox |

Schema-generated flags (e.g. `--a`, `--b`) are added automatically from the module's `input_schema`.

### Exit Codes

| Code | Meaning |
|------|---------|
| `0` | Success |
| `1` | Module execution error |
| `2` | Invalid CLI input |
| `44` | Module not found / disabled / load error |
| `45` | Schema validation error |
| `46` | Approval denied or timed out |
| `47` | Configuration error |
| `48` | Schema circular reference |
| `77` | ACL denied |
| `130` | Execution cancelled (Ctrl+C) |

---

## Configuration

apcore-cli uses a 4-tier configuration precedence:

1. **CLI flag** (highest): `--extensions-dir ./custom`
2. **Environment variable**: `APCORE_EXTENSIONS_ROOT=./custom`
3. **Config file**: `apcore.yaml`
4. **Default** (lowest): `./extensions`

### Environment Variables

| Variable | Description | Default |
|----------|-------------|---------|
| `APCORE_EXTENSIONS_ROOT` | Path to extensions directory | `./extensions` |
| `APCORE_CLI_AUTO_APPROVE` | Set to `1` to bypass all approval prompts | *(unset)* |
| `APCORE_CLI_LOGGING_LEVEL` | CLI-specific log level (takes priority over `APCORE_LOGGING_LEVEL`) | `WARNING` |
| `APCORE_LOGGING_LEVEL` | Global apcore log level (fallback when `APCORE_CLI_LOGGING_LEVEL` is unset) | `WARNING` |
| `APCORE_AUTH_API_KEY` | API key for remote registry authentication | *(unset)* |
| `APCORE_CLI_SANDBOX` | Set to `1` to enable subprocess sandboxing | *(unset)* |
| `APCORE_CLI_HELP_TEXT_MAX_LENGTH` | Maximum characters for CLI option help text before truncation | `1000` |

### Config File (`apcore.yaml`)

```yaml
extensions:
  root: ./extensions
logging:
  level: DEBUG
sandbox:
  enabled: false
cli:
  help_text_max_length: 1000
```

---

## How It Works

### Mapping: apcore to CLI

| apcore | CLI |
|--------|-----|
| `metadata["display"]["cli"]["alias"]` or `module_id` | Command name — auto-grouped by first `.` segment (`apcore-cli math add`) |
| `metadata["display"]["cli"]["description"]` or `description` | `--help` text |
| `input_schema.properties` | CLI flags (`--a`, `--b`) |
| `input_schema.required` | Required flag enforcement |
| `annotations.requires_approval` | HITL approval prompt |

### Architecture

```
User / AI Agent (terminal)
    |
    v
apcore-cli (the adapter)
    |
    +-- ConfigResolver       4-tier config precedence
    +-- GroupedModuleGroup   Auto-grouped nested Click commands (extends LazyModuleGroup)
    +-- SchemaParser         JSON Schema -> Click options
    +-- RefResolver          $ref / allOf / anyOf / oneOf
    +-- ApprovalGate         TTY-aware HITL approval
    +-- OutputFormatter      TTY-adaptive JSON/table output
    +-- AuditLogger          JSON Lines execution logging
    +-- Sandbox              Subprocess isolation
    |
    v
apcore Registry + Executor (your modules, unchanged)
```

---

## SDKs

| Language | Repository | Status |
|----------|-----------|--------|
| **Python** | [apcore-cli-python](https://github.com/aiperceivable/apcore-cli-python) | v0.3.0 -- 10 features, 329 tests |
| **TypeScript** | [apcore-cli-typescript](https://github.com/aiperceivable/apcore-cli-typescript) | v0.3.0 -- 10 features, 225 tests |
| **Rust** | [apcore-cli-rust](https://github.com/aiperceivable/apcore-cli-rust) | v0.3.0 -- 10 features, 503 tests |

---

## Project Structure

This repository contains the **specification and design documents**:

```
apcore-cli/
├── ideas/                     Design specifications and research notes
├── docs/
│   ├── project-apcore-cli.md  Project manifest (8 features)
│   ├── tech-design.md         Tech Design v1.0 (current)
│   ├── srs.md                 Software Requirements Specification
│   └── features/
│       ├── overview.md        Feature index & upstream references
│       ├── core-dispatcher.md     FE-01 (P0)
│       ├── schema-parser.md       FE-02 (P0)
│       ├── config-resolver.md     FE-07 (P0)
│       ├── approval-gate.md       FE-03 (P1)
│       ├── discovery.md           FE-04 (P1)
│       ├── output-formatter.md    FE-08 (P1)
│       ├── security.md            FE-05 (P1/P2)
│       └── shell-integration.md   FE-06 (P2)
├── CHANGELOG.md
└── README.md
```

For the **implementation**, see the [Python SDK](https://github.com/aiperceivable/apcore-cli-python).

---

## Related Projects

| Project | Description |
|---------|-------------|
| [apcore](https://github.com/aiperceivable/apcore) | Core protocol and framework for AI-Perceivable modules |
| [apcore-mcp](https://github.com/aiperceivable/apcore-mcp) | Model Context Protocol adapter |
| [apcore-a2a](https://github.com/aiperceivable/apcore-a2a) | Agent-to-Agent adapter |
| [django-apcore](https://github.com/aiperceivable/django-apcore) | Django integration |
| [flask-apcore](https://github.com/aiperceivable/flask-apcore) | Flask integration |

## License

Apache-2.0
