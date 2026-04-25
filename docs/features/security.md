# Feature Spec: Security Manager

**Feature ID**: FE-05
**Status**: Ready for Implementation
**Priority**: P1 (Auth, Encryption, Audit) / P2 (Sandbox)
**Parent**: [Tech Design v2.0](../tech-design.md) Section 8.6
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

## Contract: Sandbox._sandboxed_execute

### Inputs
- module_id: str, required — Canonical module identifier to execute in the sandbox.
- input_data: dict, required — JSON-serializable input dict passed to the module via stdin.

### Errors
- ModuleExecutionError(stderr_content) — when the subprocess exits non-zero
- ModuleExecutionError("sandbox output exceeds size limit") — when stdout exceeds 64 MiB
- ModuleExecutionError("Module '{id}' timed out in sandbox.") — when subprocess exceeds 300 s timeout

### Returns
- On success: Any — parsed JSON from subprocess stdout
- On failure: raises ModuleExecutionError (never returns None)

### Properties
- async: false
- thread_safe: false (spawns subprocess; each call creates a temp dir)
- pure: false (spawns subprocess, creates temporary directory, may read filesystem)

---

## Contract: Sandbox.execute

### Inputs
- module_id: str, required — Canonical module identifier.
- input_data: dict, required — Input dict for the module.
- executor: Executor, required — Executor instance used in non-sandbox path.

### Errors
- ModuleExecutionError — from `_sandboxed_execute` when sandbox is enabled and subprocess fails
- (any error from `executor.call` when sandbox is disabled)

### Returns
- On success: Any — module result (passed through from executor or sandboxed subprocess)
- On failure: raises ModuleExecutionError or executor-specific error

### Properties
- async: false
- thread_safe: false (sandbox path spawns subprocess; non-sandbox path inherits executor thread-safety)
- pure: false (I/O: calls executor or spawns subprocess)

---

## Contract: AuthProvider.authenticate_request

### Inputs
- headers: dict, required — HTTP headers dict to augment with the Authorization header.

### Errors
- AuthenticationError("Remote registry requires authentication. Set --api-key, APCORE_AUTH_API_KEY, or auth.api_key in config.") — when no API key is available (exit code 77)

### Returns
- On success: dict — the input headers dict with `"Authorization": "Bearer {key}"` added
- On no key: raises AuthenticationError

### Properties
- async: false
- thread_safe: true (reads config and env; no mutable shared state)
- pure: false (reads env vars and config file)

---

## Contract: ConfigEncryptor.aes_encrypt (internal: `_aes_encrypt`)

### Inputs
- plaintext: str, required — UTF-8 string to encrypt.

### Errors
- (cryptography library errors propagate — unexpected; input is always valid UTF-8 str)

### Returns
- On success: bytes — `16-byte salt || 12-byte nonce || 16-byte GCM tag || ciphertext` (enc:v2 wire format)

### Properties
- async: false
- thread_safe: false (uses OS random bytes; safe to call from single thread)
- pure: false (reads OS entropy for salt and nonce)

---

## Contract: ConfigEncryptor.store

### Inputs
- key: str, required — Config key name used as the keyring label (e.g., `"auth.api_key"`).
- value: str, required — Plaintext secret value to store.

### Errors
- keyring errors propagate if keyring.set_password fails unexpectedly

### Returns
- On success with keyring: str — `"keyring:{key}"` reference string
- On success without keyring: str — `"enc:v2:{base64}"` ciphertext reference string

### Properties
- async: false
- thread_safe: false (keyring calls are not guaranteed thread-safe)
- pure: false (writes to OS keyring or derives encryption key using hostname/username)

---

## Contract: ConfigEncryptor.retrieve

### Inputs
- config_value: str, required — The stored reference value (prefixed with `keyring:`, `enc:v2:`, or `enc:`).
- key: str, required — Config key name for error messages.
  validates: must start with `keyring:`, `enc:v2:`, or `enc:` to trigger decryption; otherwise returned as-is

### Errors
- ConfigDecryptionError(f"Keyring entry not found for '{ref_key}'.") — when keyring lookup returns None
- ConfigDecryptionError(f"Failed to decrypt configuration value '{key}'. Re-configure with 'apcore-cli config set {key}'.") — on decryption failure (wrong key, corrupted ciphertext, or `InvalidTag`)

### Returns
- On success: str — plaintext value
- On failure: raises ConfigDecryptionError (exit code 47)

### Properties
- async: false
- thread_safe: false (keyring and PBKDF2 calls are not thread-safe across instances)
- pure: false (reads OS keyring or derives key from hostname/username/PBKDF2)

---

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
   c. Return `f"enc:v2:{base64.b64encode(ciphertext).decode()}"`.

**Method: `retrieve(config_value: str, key: str) -> str`**

Logic steps:
1. If `config_value.startswith("keyring:")`:
   a. Extract `ref_key = config_value[len("keyring:"):]`.
   b. Call `result = keyring.get_password(self.SERVICE_NAME, ref_key)`.
   c. If `result is None`: raise `ConfigDecryptionError(f"Keyring entry not found for '{ref_key}'.")`.
   d. Return `result`.
2. If `config_value.startswith("enc:v2:")`:
   a. `data = base64.b64decode(config_value[len("enc:v2:"):])`
   b. Try `return self._aes_decrypt(data)` (v2: salt embedded in data).
   c. On `InvalidTag` or `ValueError`: raise `ConfigDecryptionError(f"Failed to decrypt configuration value '{key}'. Re-configure with 'apcore-cli config set {key}'.")`.
3. If `config_value.startswith("enc:")`:
   a. `ciphertext = base64.b64decode(config_value[len("enc:"):])`
   b. Try decryption using static salt `b"apcore-cli-config-v1"` + 600k iterations.
   c. If decryption fails, retry with 100k iterations (values written by SDK v0.6 and earlier).
   d. On both failures: raise `ConfigDecryptionError(f"Failed to decrypt configuration value '{key}'. Re-configure with 'apcore-cli config set {key}'.")`.
4. Otherwise: return `config_value` (plaintext, legacy or non-sensitive).

**Method: `_keyring_available() -> bool`**

Logic steps:
1. Try: `kr = keyring.get_keyring()`.
2. If `isinstance(kr, keyring.backends.fail.Keyring)`: return `False`.
3. Return `True`.
4. On any exception: return `False`.

**Method: `_derive_key() -> bytes`**

Logic steps:
1. `hostname = socket.gethostname()` (or OS equivalent).
2. `username = os.getenv("USER", os.getenv("USERNAME", "unknown"))` (or OS equivalent).
3. `salt = random 16-byte value` — generated per encryption and embedded in the ciphertext.
4. `material = f"{hostname}:{username}".encode()`.
5. Return PBKDF2-HMAC-SHA256(`material`, `salt`, iterations=**600_000**).

Output: 32-byte AES-256 key.

> **Security note**: 600,000 iterations follows OWASP 2024+ recommendations for PBKDF2-HMAC-SHA256. The salt is per-encryption and random (not static), removing the v1 machine-binding limitation.

**Method: `_aes_encrypt(plaintext: str) -> bytes`**

Logic steps:
1. `salt = random 16-byte value` — random per encryption.
2. `key = self._derive_key(salt)` — PBKDF2 with the per-encryption salt.
3. `nonce = random 12-byte value` — 96-bit nonce for GCM.
4. AES-256-GCM encrypt `plaintext.encode("utf-8")`.
5. Return `salt (16 bytes) + nonce (12 bytes) + GCM tag (16 bytes) + ciphertext`.

**Method: `_aes_decrypt(data: bytes) -> str`**

Logic steps:
1. `salt = data[:16]`, `nonce = data[16:28]`, `tag = data[28:44]`, `ct = data[44:]`.
2. `key = self._derive_key(salt)`.
3. AES-256-GCM decrypt with `(nonce, tag, ct)`.
4. Return decoded UTF-8 string.

**Wire formats:**

```
enc:v2:<base64 of (16-byte salt || 12-byte nonce || 16-byte GCM tag || ciphertext)>
```

The `enc:v2:` prefix identifies the per-encryption-salt format (written by all three SDKs).

Legacy format (read-only, written by SDK v0.6 and earlier):
```
enc:<base64 of (12-byte nonce || 16-byte GCM tag || ciphertext)>
```
When reading `enc:` (v1), implementations MUST try decryption with 600k iterations + static salt `b"apcore-cli-config-v1"` first; if decryption fails they MUST re-try with 100k iterations + same static salt (for values encrypted by very early SDK versions). On re-encrypt (e.g., `config set`), the implementation MUST write the v2 format.

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
   - `"input_hash"`: `hashlib.sha256(salt + json.dumps(input_data, sort_keys=True).encode()).hexdigest()` where `salt = secrets.token_bytes(16)`. A fresh 16-byte random salt is generated per invocation to prevent cross-invocation input correlation.
   - `"status"`: `status`.
   - `"exit_code"`: `exit_code`.
   - `"duration_ms"`: `duration_ms`.
2. Try: open `self._path` in append mode, write `json.dumps(entry) + "\n"`.
3. On `OSError as e`: log WARNING to stderr "Warning: Could not write audit log: {e}." Do NOT raise or exit.

**Method: `_get_user() -> str`**

Logic steps:
1. Try: return `os.getlogin()`.
2. On `OSError`: try `pwd.getpwuid(os.getuid()).pw_name` (Unix only).
3. On `ImportError` / `KeyError` / `AttributeError`: return `os.getenv("USER", os.getenv("USERNAME", "unknown"))`.

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
1. Build restricted environment dict using prefix-allow + deny strategy:
   - Allow: `PATH`, `LANG`, `LC_ALL` (locale / executable lookup).
   - Allow prefix: all `APCORE_*` variables from host.
   - Deny prefix: `APCORE_AUTH_*` (auth credentials must not cross trust boundary).
   - Python SDK additionally allows `PYTHONPATH` (required for module imports in Python envs).
   - Omit all other environment variables.
2. Create temporary directory (platform temp location; prefix `apcore_sandbox_`).
3. Set `HOME` and `TMPDIR` to temporary directory.
4. Serialize `input_data` to JSON string.
5. Spawn the **language-specific sandbox runner entry point** as a child process:
   - Python: `[sys.executable, "-m", "apcore_cli._sandbox_runner", module_id]`
   - TypeScript/Node: `[process.execPath, "<pkg_runner_script>", "--internal-sandbox-runner", module_id]`
   - Rust: `[std::env::current_exe(), "--internal-sandbox-runner", module_id]`
   - Pass serialized JSON to stdin.
   - Capture stdout (result) and stderr (error messages).
   - Timeout: **300 seconds** (5 minutes).
6. If process exits non-zero: raise `ModuleExecutionError(stderr_content)`.
7. If output exceeds 64 MiB: raise `ModuleExecutionError("sandbox output exceeds size limit")`.
8. Parse stdout as JSON and return.
9. Cleanup: temporary directory is auto-deleted.

**Language-specific sandbox runner** (invoked as child process):

Each language port ships a runner entry point that reads `module_id` from argv, deserializes JSON input from stdin, instantiates a fresh `Registry` + `Executor` in the restricted environment, calls the module, and writes JSON result to stdout. The runner MUST NOT read from the host filesystem beyond the extensions directory.

**Restricted environment contents:**

| Variable | Source | Purpose |
|----------|--------|---------|
| `PATH` | Host | Locate runtime executable |
| `PYTHONPATH` | Host (Python only) | Module imports in Python envs |
| `LANG` | Host | Locale |
| `LC_ALL` | Host | Locale |
| `APCORE_*` | Host (excluding `APCORE_AUTH_*`) | apcore configuration |
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
| T-SEC-05 | Store API key without keyring | `apcore.yaml` contains `enc:v2:base64...`. |
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
