# Snake-case kwargs conformance — Algorithm C-SNAKE

## Purpose

Verifies that schema properties named in `snake_case` (especially multi-word
fields like `has_solution`, `sort_by`) survive the round trip from CLI parsing
to the input dict that `Executor.execute(module_id, input)` receives.

Each SDK uses a different argument parser:

- Python — `click` (auto-derives parameter names by replacing `-` with `_`).
- Rust — `clap` (`Arg::new(prop_name)` keeps the original snake_case id).
- TypeScript — `commander` (auto-camelCases long flag names; the SDK must
  reverse-map the camelCase keys back to the schema's snake_case property
  names before forwarding to the module).

Without a conformance test, a single SDK can silently drop multi-word
snake_case fields while the others still work. This was the case for the
TypeScript SDK prior to v0.8.x — `--has-solution` parsed into
`options.hasSolution` and was forwarded as-is, so modules reading
`input["has_solution"]` always got `undefined`.

## Fixture format

`cases.json` contains:

| Field | Type | Description |
|---|---|---|
| `module_id` | string | Identifier for the synthetic test module the runner registers. |
| `input_schema` | object | JSON Schema for the module's input. The runner registers a fake module with this schema and a stub executor. |
| `test_cases` | array | List of `{id, args, expected_input}` cases. |

For each test case, the runner:

1. Builds the module's CLI command from `input_schema`.
2. Parses the CLI invocation `apcli exec <module_id> <args...>`.
3. Captures the `input` argument that the executor's `execute` callback
   receives.
4. Asserts the captured input contains every key/value pair in
   `expected_input` (partial match — extra keys in the actual input are
   allowed, but every key in `expected_input` must be present and equal).

The assertion is a **partial match** so that SDKs are free to include
default-valued fields that were not under test in this case.

## Adding a new case

Edit `cases.json` and add to `test_cases`. Every SDK runner reads this file
at test time, so a new case takes effect without per-SDK changes.
