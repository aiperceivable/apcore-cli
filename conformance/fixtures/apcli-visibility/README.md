# apcli Visibility Fixtures

Cross-language fixtures for the v0.7.0 **apcli sub-group visibility
decision chain**. Every SDK (TypeScript, Python, Rust) consumes the same
inputs **and** asserts byte-equality against the same canonical
`expected_help.txt`. Each SDK is responsible for making its underlying
help renderer (Commander.js / Click / clap) emit the canonical format —
see "Canonical help format" below.

## Decision chain under test

An SDK MUST resolve apcli visibility using this 4-tier precedence:

1. `create_cli(apcli={...})` — explicit caller option (highest)
2. `APCORE_CLI_APCLI` env var — `show` / `hide` / `1` / `0` / `true` / `false`
3. `apcore.yaml` — `apcli.mode` / `apcli.include` / `apcli.exclude`
4. Auto-detect — `none` when `registry_injected=true`, else `all`

## Fixture files (per scenario)

| File | Purpose |
|---|---|
| `create_cli.json` | Caller options passed to `create_cli(...)`. Keys are **snake_case**; SDKs in camelCase languages map them at the test boundary. |
| `env.json` | Environment-variable overlay applied before `create_cli`. |
| `input.yaml` | Optional. `apcore.yaml` content materialized in the working directory. |
| `expected_help.txt` | Canonical `--help` output. SDK tests MUST byte-match. |

## Canonical help format

The help output follows **clap v4 / GNU** conventions. Every SDK must
configure its underlying framework to emit this exact shape:

- Description line first (then blank line).
- `Usage: <prog> [OPTIONS] [COMMAND]` — uppercase `OPTIONS`, `COMMAND`.
- `Commands:` section **before** `Options:`.
- Placeholders uppercase: `<LEVEL>`, `<PATH>`.
- Default values as `[default: VALUE]` — brackets, no quotes.
- `-h, --help` description: `Print help`.
- `-V, --version` description: `Print version` (short flag `-V` required).
- `help` subcommand description: `Print this message or the help of the given subcommand(s)`.
- `-h`/`-V` always rendered last in `Options:`.
- Long-only options indented 4 extra spaces so `--long` aligns under rows
  that have a short flag (`      --long` vs `  -s, --long`).
- Two-space gap between the term column and the description column.
- No line wrapping inside descriptions — each SDK must disable its
  framework's default terminal-width wrapping to keep the golden
  deterministic.

Reference TS implementation: `apcore-cli-typescript/src/canonical-help.ts`
(Commander.js `configureHelp({ formatHelp })` override).

## Scenarios

| Scenario | Covers |
|---|---|
| `standalone-default` | No injected registry → auto-detect resolves to `all` (apcli visible). |
| `embedded-default` | Injected registry → auto-detect resolves to `none` (apcli hidden). |
| `cli-override` | `create_cli(apcli={mode: all})` overrides `apcore.yaml mode: none`. |
| `env-override` | `APCORE_CLI_APCLI=show` overrides `apcore.yaml mode: none`. |
| `yaml-include` | `apcore.yaml apcli.include: [list, describe]` selects a subset. |

## create_cli.json schema

```json
{
  "prog_name": "apcore-cli",        // required
  "registry_injected": true,        // optional; pass a mock registry when true
  "apcli": { "mode": "all" }        // optional; same shape as apcore.yaml apcli.*
}
```

SDKs written in camelCase languages (TypeScript, etc.) map
`prog_name` → `progName`, `registry_injected` → `registryInjected` at the
test-loader boundary.
