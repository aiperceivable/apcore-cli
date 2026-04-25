# Contributing to apcore-cli

Thank you for your interest in contributing to apcore-cli.

## Schema Validation

The three SDK implementations use different runtime validators that all target JSON Schema Draft 7:

| SDK | Library | Notes |
|-----|---------|-------|
| Python | `jsonschema>=4.20` | Runtime validation, Draft 7/2019-09 |
| TypeScript | `@sinclair/typebox^0.32.0` | Schema builder + runtime validator |
| Rust | `jsonschema 0.28` | Runtime validation, Draft 7 |

### Cross-SDK compatibility
- Schemas must use JSON Schema Draft 7 features only; avoid Draft 2020-12 keywords.
- For behavioral parity testing, use the fixtures in `conformance/` which are validated against all three SDKs.
- Known divergence: typebox performs compile-time schema composition; jsonschema and jsonschema-rs validate at runtime. Edge-case keyword support may differ.
