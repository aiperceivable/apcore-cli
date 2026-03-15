# Features Overview: apcore-cli

This directory contains the detailed feature specifications for `apcore-cli`. These features are designed to map `apcore` modules to a high-performance terminal interface.

---

## Feature List & Priority

| ID | Feature Name | Description | Priority | Specification |
|:---|:---|:---|:---|:---|
| **FE-01** | **Core Dispatcher** | CLI entry point, Registry loading, STDIN piping, module execution via Executor. | P0 | [core-dispatcher.md](core-dispatcher.md) |
| **FE-02** | **Schema Parser** | Automatic generation of flags/options from JSON Schema `input_schema`, including `$ref` resolution. | P0 | [schema-parser.md](schema-parser.md) |
| **FE-03** | **Approval Gate** | TTY-aware HITL logic for modules requiring approval, with bypass and timeout. | P1 | [approval-gate.md](approval-gate.md) |
| **FE-04** | **Discovery** | Terminal-optimized `list` and `describe` commands with tag filtering and format selection. | P1 | [discovery.md](discovery.md) |
| **FE-05** | **Security Manager** | API key auth, encrypted config (keyring + AES-256-GCM), audit logging, subprocess sandboxing. | P1/P2 | [security.md](security.md) |
| **FE-06** | **Shell Integration** | Shell completion scripts (bash/zsh/fish) and man page generation. | P2 | [shell-integration.md](shell-integration.md) |
| **FE-07** | **Config Resolver** | 4-tier configuration precedence (CLI > Env > File > Default). | P0 | [config-resolver.md](config-resolver.md) |
| **FE-08** | **Output Formatter** | TTY-adaptive output formatting (JSON for pipes, tables for terminals). | P1 | [output-formatter.md](output-formatter.md) |

---

## Implementation Order

1. **Config Resolver** (FE-07): Foundation for all configurable values.
2. **Core Dispatcher** (FE-01): Establishes the `apcore-cli` command structure and registry loading.
3. **Schema Parser** (FE-02): Connects module metadata to CLI arguments.
4. **Output Formatter** (FE-08): TTY-adaptive rendering used by Discovery and exec.
5. **Discovery** (FE-04): Provides tools for browsing available modules.
6. **Approval Gate** (FE-03): Adds governance and safety for sensitive module execution.
7. **Security Manager** (FE-05): Authentication, encryption, auditing, sandboxing.
8. **Shell Integration** (FE-06): Completion scripts and man pages.

---

## Dependency Graph

```
FE-07 Config Resolver (foundation)
  └── FE-01 Core Dispatcher (depends on FE-07)
        ├── FE-02 Schema Parser (depends on FE-01)
        │     └── FE-03 Approval Gate (depends on FE-02)
        ├── FE-08 Output Formatter (used by FE-01, FE-04)
        │     └── FE-04 Discovery (depends on FE-01, FE-08)
        ├── FE-05 Security Manager (depends on FE-07)
        └── FE-06 Shell Integration (depends on FE-01, FE-02)
```

---

## Upstream References

| Document | Location |
|----------|----------|
| Tech Design v1.0 | `docs/apcore-cli/tech-design.md` |
| SRS v0.1 | `docs/apcore-cli/srs.md` |
| Project Manifest | `docs/project-apcore-cli.md` |
