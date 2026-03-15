# Idea Draft: apcore-cli

**Status**: VALIDATED (High Demand for CLI-first AI Agent architectures)
**Owner**: Spec Forge
**Date**: 2026-03-14

---

## 1. The Core Idea
`apcore-cli` is an **Adapter** that maps `apcore` modules directly to terminal commands. It treats the CLI as a "Cognitive Interface," enabling AI Agents and developers to invoke complex business logic with the efficiency and composability of Unix pipes, bypassing the token bloat of JSON-RPC/MCP where appropriate.

## 2. Problem & Validation
### The "Why now?"
- **Token Bloat**: MCP and deep JSON-RPC layers consume excessive tokens for simple tool calls.
- **DX Gap**: Developers lack a standard way to test `apcore` modules without a full UI or MCP host.
- **Agent Native**: LLMs are trained on terminal interactions; shell commands are often more reliable than complex schema-based tool calling.

### Validation Evidence
- Industry shift (e.g., Perplexity) back to CLI/API-first patterns.
- Success of the "Discovery/Capabilities/Execution" 3-layer metadata model in `apcore`.

## 3. Scope & Constraints
### In-Scope
- **Mapping Logic**: Automatic conversion of `Module` metadata (Schema) to CLI options/flags.
- **Core Commands**: `apcore-cli exec`, `apcore-cli list`, `apcore-cli describe`.
- **Governance**: Interactive HITL (Human-in-the-Loop) terminal prompts for `requires_approval=true`.
- **Piping**: Support for JSON input via `stdin` and output formatting (Table/JSON).

### Out-of-Scope (for MVP)
- Remote execution via `apcore-a2a` (Phase 4).
- Complex shell completion (Zsh/Bash/Fish) generation (deferred to Phase 2/3).

## 4. Requirement IDs (Initial)
- **FR-001**: Mapping of `Canonical ID` to subcommands.
- **FR-002**: Auto-generation of `--key value` flags from JSON Schema `input_schema`.
- **FR-003**: Support for `stdin` piping (`-`).
- **FR-004**: TTY-aware approval requests for destructive or sensitive operations.
- **NFR-001**: Execution overhead < 100ms (excluding module logic).
- **NFR-002**: Zero-config: Should work by just pointing to an `apcore` Registry.

---

## 5. "What if we don't build this?"
Developers will continue writing redundant "glue code" for every module they want to expose to the terminal. Agents will remain tethered to high-latency protocols for simple tasks, increasing cost and reducing reliability.
