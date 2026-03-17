# Feature Spec: Config Resolver

**Feature ID**: FE-07
**Status**: Ready for Implementation
**Priority**: P0
**Parent**: [Tech Design v1.0](../tech-design.md) Section 8.8
**SRS Requirements**: FR-DISP-005

---

## 1. Description

The Config Resolver implements the 4-tier configuration precedence hierarchy used throughout `apcore-cli`. For any configurable value, the resolver checks sources in order: CLI flag (highest priority), environment variable, config file (`apcore.yaml`), and built-in default (lowest priority). It handles malformed config files gracefully and supports flattened dot-notation key access.

---

## 2. Requirements Traceability

| Req ID | SRS Ref | Description |
|--------|---------|-------------|
| FR-07-01 | FR-DISP-005 | 4-tier configuration precedence: CLI > Env > File > Default. |
| FR-07-02 | FR-DISP-005 AF-1 | Config file not found: skip silently. |
| FR-07-03 | FR-DISP-005 AF-2 | Config file malformed: log WARNING, use defaults. |

---

## 3. Module Path

`apcore_cli/config.py`

---

## 4. Implementation Details

### 4.1 Class: `ConfigResolver`

**Constructor**: `__init__(self, cli_flags: dict[str, Any] = None, config_path: str = "apcore.yaml")`

Logic steps:
1. Store `self._cli_flags = cli_flags or {}`.
2. Store `self._config_path = config_path`.
3. Call `self._config_file = self._load_config_file()`.

### 4.2 Method: `resolve`

**Signature**: `resolve(key: str, cli_flag: str = None, env_var: str = None) -> Any`

Logic steps:
1. **Tier 1 — CLI flag**: If `cli_flag` is not None and `cli_flag` exists in `self._cli_flags`:
   a. Get `value = self._cli_flags[cli_flag]`.
   b. If `value is not None`: return `value`.
2. **Tier 2 — Environment variable**: If `env_var` is not None:
   a. Get `env_value = os.environ.get(env_var)`.
   b. If `env_value is not None` and `env_value != ""`: return `env_value`.
3. **Tier 3 — Config file**: If `self._config_file is not None` and `key in self._config_file`:
   a. Return `self._config_file[key]`.
4. **Tier 4 — Default**: Return `self.DEFAULTS.get(key)`.

### 4.3 Default Values

```python
DEFAULTS = {
    "extensions.root": "./extensions",
    "logging.level": "WARNING",
    "sandbox.enabled": False,
    "cli.stdin_buffer_limit": 10_485_760,  # 10 MB
}
```

### 4.4 Method: `_load_config_file`

**Signature**: `_load_config_file() -> dict | None`

Logic steps:
1. Try to open `self._config_path`.
2. On `FileNotFoundError`: return `None` (silent, per FR-DISP-005 AF-1).
3. Parse with `yaml.safe_load()`.
4. On `yaml.YAMLError`: log WARNING "Configuration file '{path}' is malformed, using defaults." Return `None`.
5. If parsed result is not a `dict`: log WARNING, return `None`.
6. Call `self._flatten_dict(config)` to convert nested YAML to dot-notation.
7. Return flattened dict.

### 4.5 Method: `_flatten_dict`

**Signature**: `_flatten_dict(d: dict, prefix: str = "") -> dict`

Logic steps:
1. Initialize `result = {}`.
2. For each `(key, value)` in `d.items()`:
   a. `full_key = f"{prefix}.{key}" if prefix else key`.
   b. If `isinstance(value, dict)`: recurse `result.update(self._flatten_dict(value, full_key))`.
   c. Else: `result[full_key] = value`.
3. Return `result`.

**Example:**
```yaml
# apcore.yaml
extensions:
  root: /custom/path
auth:
  api_key: "keyring:auth.api_key"
logging:
  level: DEBUG
```

Flattened to:
```python
{
    "extensions.root": "/custom/path",
    "auth.api_key": "keyring:auth.api_key",
    "logging.level": "DEBUG",
}
```

---

## 5. Configuration Keys

| Key | CLI Flag | Environment Variable | Default | Type | SRS Reference |
|-----|----------|---------------------|---------|------|---------------|
| `extensions.root` | `--extensions-dir` | `APCORE_EXTENSIONS_ROOT` | `./extensions` | `str` (path) | FR-DISP-003, FR-DISP-005 |
| `auth.api_key` | `--api-key` | `APCORE_AUTH_API_KEY` | `None` | `str` (secret) | FR-SEC-001 |
| `logging.level` | `--log-level` | `APCORE_LOGGING_LEVEL` | `WARNING` | `str` (enum) | NFR-MNT-002 |
| `sandbox.enabled` | `--sandbox` | `APCORE_CLI_SANDBOX` | `False` | `bool` | FR-SEC-004 |
| `cli.auto_approve` | `--yes` | `APCORE_CLI_AUTO_APPROVE` | `False` | `bool` | FR-APPR-004 |

---

## 6. Parameter Validation

| Parameter | Type | Valid Values | Invalid Handling |
|-----------|------|-------------|------------------|
| `config_path` | `str` | Valid file path or non-existent path. | Non-existent: skip silently. Invalid YAML: WARNING + use defaults. |
| `cli_flags` | `dict` | Keys are flag names, values are parsed types. | `None`: treated as empty dict. |
| `key` | `str` | Dot-notation config key. | Unknown key: returns `None` from DEFAULTS. |
| `env_var` | `str` | Valid env var name. | Unset or empty string: skip to next tier. |

---

## 7. Precedence Test Matrix

| Test | CLI Flag | Env Var | Config File | Default | Expected Result |
|------|----------|---------|-------------|---------|-----------------|
| All sources set | `/cli-path` | `/env-path` | `/config-path` | `./extensions` | `/cli-path` |
| No CLI flag | — | `/env-path` | `/config-path` | `./extensions` | `/env-path` |
| No CLI flag, no env | — | — | `/config-path` | `./extensions` | `/config-path` |
| No CLI flag, no env, no config | — | — | — | `./extensions` | `./extensions` |
| CLI flag is `None` | `None` | `/env-path` | — | `./extensions` | `/env-path` |
| Env var is empty string | — | `""` | `/config-path` | `./extensions` | `/config-path` |
| Config file not found | — | — | (missing) | `./extensions` | `./extensions` |
| Config file malformed | — | — | (invalid YAML) | `./extensions` | `./extensions` + WARNING |

---

## 8. Error Handling

| Condition | Behavior | SRS Reference |
|-----------|----------|---------------|
| Config file not found | Skip silently, fall through to default. | FR-DISP-005 AF-1 |
| Config file malformed YAML | Log WARNING, fall through to default. | FR-DISP-005 AF-2 |
| Config file not a dict at root | Log WARNING, fall through to default. | FR-DISP-005 AF-2 |
| Unknown config key | Return `None` (from DEFAULTS.get). | — |

---

## 9. Verification

| Test ID | Description | Expected Result |
|---------|-------------|-----------------|
| T-CFG-01 | CLI flag `--extensions-dir /cli` with env var set | Uses `/cli` (Tier 1 wins). |
| T-CFG-02 | No CLI flag, `APCORE_EXTENSIONS_ROOT=/env` set | Uses `/env` (Tier 2 wins). |
| T-CFG-03 | No CLI flag, no env, `apcore.yaml` has `extensions.root: /config` | Uses `/config` (Tier 3 wins). |
| T-CFG-04 | No CLI flag, no env, no config file | Uses `./extensions` (Tier 4 default). |
| T-CFG-05 | Config file does not exist | No error. Default used. |
| T-CFG-06 | Config file is invalid YAML | WARNING logged. Default used. |
| T-CFG-07 | Config file has nested keys | Flattened correctly: `extensions.root` accessible. |
| T-CFG-08 | Env var set to empty string | Skipped. Falls through to config file or default. |
| T-CFG-09 | CLI flag explicitly set to `None` | Skipped. Falls through to env var or lower. |
