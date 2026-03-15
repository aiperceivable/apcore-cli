# Feature Spec: Security Manager

**Feature ID**: FE-05
**Status**: Ready for Implementation
**Priority**: P1 (Auth, Encryption, Audit) / P2 (Sandbox)
**Parent**: [Tech Design v1.0](../tech-design.md) Section 8.6
**SRS Requirements**: FR-SEC-001, FR-SEC-002, FR-SEC-003, FR-SEC-004

---

## 1. Description

The Security Manager is a sub-package providing four security components: API key authentication for remote registries, encrypted configuration storage using OS keyring with AES-256-GCM fallback, append-only JSON Lines audit logging of all module executions, and subprocess-based execution sandboxing for untrusted modules.

---

## 2. Requirements Traceability

| Req ID | SRS Ref | Description |
|--------|---------|-------------|
| FR-05-01 | FR-SEC-001 | API key authentication for remote registries. |
| FR-05-02 | FR-SEC-002 | Encrypted configuration storage (keyring + AES-256-GCM). |
| FR-05-03 | FR-SEC-003 | Audit logging in JSON Lines format. |
| FR-05-04 | FR-SEC-004 | Subprocess-based execution sandboxing. |

---

## 3. Module Paths

```
apcore_cli/security/
├── __init__.py            # Exports: AuthProvider, ConfigEncryptor, AuditLogger, Sandbox
├── auth.py                # API key authentication
├── config_encryptor.py    # Keyring + AES-256-GCM fallback
├── audit.py               # JSON Lines audit logging
└── sandbox.py             # Subprocess isolation
```

---

## 4. Implementation Details

### 4.1 AuthProvider (`auth.py`)

**Class**: `AuthProvider`

**Constructor**: `__init__(self, config: ConfigResolver)`

**Method: `get_api_key() -> str | None`**

Logic steps:
1. Call `self._config.resolve("auth.api_key", cli_flag="--api-key", env_var="APCORE_AUTH_API_KEY")`.
2. If result is not None and starts with `keyring:` or `enc:`: call `self._config.encryptor.retrieve(result, "auth.api_key")`.
3. Return the resolved (and possibly decrypted) API key, or `None`.

**Method: `authenticate_request(headers: dict) -> dict`**

Logic steps:
1. Call `key = self.get_api_key()`.
2. If `key is None`: raise `AuthenticationError("Remote registry requires authentication. Set --api-key, APCORE_AUTH_API_KEY, or auth.api_key in config.")`.
3. Set `headers["Authorization"] = f"Bearer {key}"`.
4. Return `headers`.

**Method: `handle_response(status_code: int) -> None`**

Logic steps:
1. If `status_code == 401` or `status_code == 403`: raise `AuthenticationError("Authentication failed. Verify your API key.")`.
2. Otherwise: return (no action).

| Scenario | Behavior | Exit Code |
|----------|----------|-----------|
| API key from `--api-key abc123` | `Authorization: Bearer abc123` | — |
| API key from `APCORE_AUTH_API_KEY=abc123` | `Authorization: Bearer abc123` | — |
| API key from config file (encrypted) | Decrypted, used as Bearer token | — |
| No API key, remote registry configured | `AuthenticationError` | 77 |
| HTTP 401 response | `AuthenticationError("Authentication failed")` | 77 |
| HTTP 403 response | `AuthenticationError("Authentication failed")` | 77 |
| Local-only registry | `get_api_key()` returns None, but not called | — |

### 4.2 ConfigEncryptor (`config_encryptor.py`)

**Class**: `ConfigEncryptor`

**Constant**: `SERVICE_NAME = "apcore-cli"`

**Method: `store(key: str, value: str) -> str`**

Logic steps:
1. Call `self._keyring_available()`.
2. If `True`:
   a. Call `keyring.set_password(self.SERVICE_NAME, key, value)`.
   b. Return `f"keyring:{key}"`.
3. If `False`:
   a. Log WARNING: "OS keyring unavailable. Using file-based encryption."
   b. Call `ciphertext = self._aes_encrypt(value)`.
   c. Return `f"enc:{base64.b64encode(ciphertext).decode()}"`.

**Method: `retrieve(config_value: str, key: str) -> str`**

Logic steps:
1. If `config_value.startswith("keyring:")`:
   a. Extract `ref_key = config_value[len("keyring:"):]`.
   b. Call `result = keyring.get_password(self.SERVICE_NAME, ref_key)`.
   c. If `result is None`: raise `ConfigDecryptionError(f"Keyring entry not found for '{ref_key}'.")`.
   d. Return `result`.
2. If `config_value.startswith("enc:")`:
   a. `ciphertext = base64.b64decode(config_value[len("enc:"):])`
   b. Try `return self._aes_decrypt(ciphertext)`.
   c. On `InvalidTag` or `ValueError`: raise `ConfigDecryptionError(f"Failed to decrypt configuration value '{key}'. Re-configure with 'apcore-cli config set {key}'.")`.
3. Otherwise: return `config_value` (plaintext, legacy or non-sensitive).

**Method: `_keyring_available() -> bool`**

Logic steps:
1. Try: `kr = keyring.get_keyring()`.
2. If `isinstance(kr, keyring.backends.fail.Keyring)`: return `False`.
3. Return `True`.
4. On any exception: return `False`.

**Method: `_derive_key() -> bytes`**

Logic steps:
1. `hostname = socket.gethostname()`.
2. `username = os.getenv("USER", os.getenv("USERNAME", "unknown"))`.
3. `salt = b"apcore-cli-config-v1"`.
4. `material = f"{hostname}:{username}".encode()`.
5. Return `hashlib.pbkdf2_hmac("sha256", material, salt, iterations=100_000)`.

Output: 32-byte AES-256 key.

**Method: `_aes_encrypt(plaintext: str) -> bytes`**

Logic steps:
1. `key = self._derive_key()`.
2. `nonce = os.urandom(12)` — 96-bit nonce for GCM.
3. Create `Cipher(algorithms.AES(key), modes.GCM(nonce))`.
4. `encryptor = cipher.encryptor()`.
5. `ct = encryptor.update(plaintext.encode("utf-8")) + encryptor.finalize()`.
6. Return `nonce (12 bytes) + encryptor.tag (16 bytes) + ct`.

**Method: `_aes_decrypt(data: bytes) -> str`**

Logic steps:
1. `key = self._derive_key()`.
2. `nonce = data[:12]`, `tag = data[12:28]`, `ct = data[28:]`.
3. Create `Cipher(algorithms.AES(key), modes.GCM(nonce, tag))`.
4. `decryptor = cipher.decryptor()`.
5. Return `(decryptor.update(ct) + decryptor.finalize()).decode("utf-8")`.

**Wire format for encrypted values:**
```
enc:<base64 of (12-byte nonce || 16-byte GCM tag || ciphertext)>
```

**Dependencies**: `keyring >= 24.0`, `cryptography >= 41.0`.

### 4.3 AuditLogger (`audit.py`)

**Class**: `AuditLogger`

**Constant**: `DEFAULT_PATH = Path.home() / ".apcore-cli" / "audit.jsonl"`

**Constructor**: `__init__(self, path: Path | None = None)`

Logic steps:
1. `self._path = path or self.DEFAULT_PATH`.
2. Call `self._ensure_directory()`: `self._path.parent.mkdir(parents=True, exist_ok=True)`.

**Method: `log_execution(module_id, input_data, status, exit_code, duration_ms) -> None`**

| Parameter | Type | Validation |
|-----------|------|------------|
| `module_id` | `str` | Canonical ID format (already validated upstream). |
| `input_data` | `dict` | Any dict. Serialized for hashing only (not stored). |
| `status` | `Literal["success", "error"]` | Must be one of two values. |
| `exit_code` | `int` | 0-255. |
| `duration_ms` | `int` | >= 0. |

Logic steps:
1. Build `entry` dict:
   - `"timestamp"`: `datetime.now(UTC).isoformat(timespec="milliseconds").replace("+00:00", "Z")`.
   - `"user"`: call `self._get_user()`.
   - `"module_id"`: `module_id`.
   - `"input_hash"`: `hashlib.sha256(json.dumps(input_data, sort_keys=True).encode()).hexdigest()`.
   - `"status"`: `status`.
   - `"exit_code"`: `exit_code`.
   - `"duration_ms"`: `duration_ms`.
2. Try: open `self._path` in append mode, write `json.dumps(entry) + "\n"`.
3. On `OSError as e`: log WARNING to stderr "Warning: Could not write audit log: {e}." Do NOT raise or exit.

**Method: `_get_user() -> str`**

Logic steps:
1. Try: return `os.getlogin()`.
2. On `OSError`: return `os.getenv("USER", os.getenv("USERNAME", "unknown"))`.

**Audit entry example:**
```json
{"timestamp":"2026-03-14T10:30:45.123Z","user":"tercelyi","module_id":"math.add","input_hash":"a1b2c3d4...","status":"success","exit_code":0,"duration_ms":42}
```

### 4.4 Sandbox (`sandbox.py`)

**Class**: `Sandbox`

**Constructor**: `__init__(self, enabled: bool = False)`

**Method: `execute(module_id: str, input_data: dict, executor: Executor) -> Any`**

Logic steps:
1. If `not self._enabled`: return `executor.call(module_id, input_data)` (no sandbox).
2. If `self._enabled`: call `self._sandboxed_execute(module_id, input_data)`.

**Method: `_sandboxed_execute(module_id: str, input_data: dict) -> Any`**

Logic steps:
1. Build restricted environment dict:
   a. Copy `PATH` from host (required for Python executable).
   b. Copy `PYTHONPATH` from host (required for module imports).
   c. Copy `LANG`, `LC_ALL` from host (locale).
   d. Copy all `APCORE_*` variables from host.
   e. Omit all other environment variables.
2. Create temporary directory: `tempfile.TemporaryDirectory(prefix="apcore_sandbox_")`.
3. Set `HOME` and `TMPDIR` to temporary directory.
4. Serialize `input_data` to JSON string.
5. Call `subprocess.run()`:
   - `cmd`: `[sys.executable, "-m", "apcore_cli._sandbox_runner", module_id]`
   - `input`: serialized JSON.
   - `capture_output`: `True`.
   - `text`: `True`.
   - `env`: restricted environment.
   - `cwd`: temporary directory.
   - `timeout`: 300 seconds (5 minutes).
6. If `result.returncode != 0`: raise `ModuleExecutionError(result.stderr)`.
7. Parse `result.stdout` as JSON and return.
8. Cleanup: temporary directory is auto-deleted by context manager.

**Sandbox Runner Module**: `apcore_cli/_sandbox_runner.py`

```python
"""Entry point for sandboxed module execution."""
import json
import sys
from apcore import Registry, Executor

def main():
    module_id = sys.argv[1]
    input_data = json.loads(sys.stdin.read())
    extensions_root = os.environ.get("APCORE_EXTENSIONS_ROOT", "./extensions")
    registry = Registry(extensions_root)
    registry.discover()
    executor = Executor(registry)
    result = executor.call(module_id, input_data)
    json.dump(result, sys.stdout)

if __name__ == "__main__":
    main()
```

**Restricted environment contents:**

| Variable | Source | Purpose |
|----------|--------|---------|
| `PATH` | Host | Locate Python executable |
| `PYTHONPATH` | Host | Module imports |
| `LANG` | Host | Locale |
| `LC_ALL` | Host | Locale |
| `APCORE_*` | Host | apcore configuration |
| `HOME` | Temp dir | Prevent access to real home |
| `TMPDIR` | Temp dir | Isolate temp files |

---

## 5. Error Handling

| Condition | Exit Code | Error Message | SRS Reference |
|-----------|-----------|---------------|---------------|
| No API key for remote registry | 77 | "Error: Remote registry requires authentication..." | FR-SEC-001 AF-1 |
| Authentication failed (401/403) | 77 | "Error: Authentication failed. Verify your API key." | FR-SEC-001 AF-2 |
| Config decryption failure | 47 | "Error: Failed to decrypt configuration value '{key}'..." | FR-SEC-002 AF-2 |
| Audit log write failure | — (WARNING) | "Warning: Could not write audit log: {reason}." | FR-SEC-003 AF-1 |
| Sandbox subprocess failure | 1 | "Error: Module '{id}' execution failed: {stderr}." | FR-SEC-004 |
| Sandbox timeout (5 min) | 1 | "Error: Module '{id}' timed out in sandbox." | FR-SEC-004 |

---

## 6. Verification

| Test ID | Description | Expected Result |
|---------|-------------|-----------------|
| T-SEC-01 | Set `APCORE_AUTH_API_KEY=abc123`, access remote registry | HTTP request includes `Authorization: Bearer abc123`. |
| T-SEC-02 | No API key, remote registry configured | Exit 77. stderr: "requires authentication". |
| T-SEC-03 | Remote registry returns 401 | Exit 77. stderr: "Authentication failed". |
| T-SEC-04 | Store API key with keyring available | `apcore.yaml` contains `keyring:auth.api_key`. |
| T-SEC-05 | Store API key without keyring | `apcore.yaml` contains `enc:base64...`. |
| T-SEC-06 | Retrieve keyring-stored value | Correct value returned from OS keyring. |
| T-SEC-07 | Retrieve AES-encrypted value | Correct value decrypted and returned. |
| T-SEC-08 | Corrupted ciphertext | Exit 47. stderr: "Failed to decrypt". |
| T-SEC-09 | Machine change (different hostname) | Exit 47. AES key derivation produces different key. |
| T-SEC-10 | Successful module execution | Audit log contains entry with `status: "success"`, `exit_code: 0`. |
| T-SEC-11 | Failed module execution | Audit log contains entry with `status: "error"`, correct exit code. |
| T-SEC-12 | Audit log not writable | Module executes successfully. WARNING on stderr. |
| T-SEC-13 | `os.getlogin()` fails | Audit entry has `user` from `USER` env var or `"unknown"`. |
| T-SEC-14 | `--sandbox` flag provided | Module runs in restricted subprocess. `HOME` env var is temp dir. |
| T-SEC-15 | Sandbox: module writes file | File written to temp dir, not user's working directory. |
| T-SEC-16 | No sandbox flag | Module runs in current process. Normal environment. |
| T-SEC-17 | Local-only registry | No API key required. No authentication headers. |
| T-SEC-18 | Inspect `apcore.yaml` for stored secret | Value is NOT plaintext. Prefixed with `keyring:` or `enc:`. |
