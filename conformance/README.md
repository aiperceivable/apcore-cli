# Conformance Suite

## Purpose

This directory contains cross-language parity test fixtures that all three SDK implementations — Python, TypeScript, and Rust — must pass. The fixtures define the canonical behavior of the `apcore-cli` command-line interface. Any SDK that wraps or re-implements the CLI must produce output that satisfies every test case in this suite.

The goal is to guarantee that a user switching between SDK implementations gets identical observable behavior: the same JSON fields, the same exit codes, and the same error messages.

---

## Fixture Format

Fixtures live in `fixtures/cli_parity.json`. The top-level object has the following shape:

```json
{
  "description": "CLI argument and output parity (Algorithm C01)",
  "version": "0.7.0",
  "conformance_spec": "Algorithm C01",
  "test_cases": [ ... ]
}
```

Each entry in `test_cases` is an object with the following fields:

| Field | Type | Required | Description |
|---|---|---|---|
| `id` | string | yes | Unique identifier for this test case. Used in failure messages and CI reports. |
| `args` | string[] | yes | CLI arguments array passed to `apcore-cli`. Each element is a separate argument token (already shell-split). |
| `expected_json_output` | object | no | Expected fields in the JSON object printed to stdout. This is a **partial match**: the actual output must contain at least these keys with matching values. Extra keys in the actual output are allowed. |
| `expected_output_type` | string | no | When set to `"array"`, asserts that stdout parses as a JSON array rather than an object. |
| `expected_json_keys` | string[] | no | Asserts that the top-level JSON object (or each element if the output is an array) contains at least these keys. Values are not checked. |
| `expected_exit_code` | integer | no | Expected process exit code. When omitted, the test does not assert the exit code. |
| `expected_stderr_contains` | string | no | Expected substring that must appear somewhere in stderr. Case-sensitive. |

---

## Algorithm C01

**Algorithm C01** is the CLI argument and output parity algorithm. It is the normative specification that all SDK implementations must conform to.

The algorithm states:

> For any valid set of CLI arguments, every conforming SDK implementation must produce byte-for-byte identical JSON output on stdout, identical exit codes, and stderr output that satisfies all `expected_stderr_contains` assertions.

Concretely, C01 covers:

- **Argument parsing** — flags, positional arguments, and short/long aliases must be accepted in the same form across all SDKs.
- **JSON output shape** — field names, nesting, and value types in JSON responses must be consistent.
- **Exit code semantics** — the numeric exit codes for success, validation errors, module-not-found, and usage errors must match across all SDKs (see exit code table below).
- **Error message content** — the stderr output for known error conditions must contain the required substrings so that callers can reliably parse error types.

### Exit Code Table

| Code | Meaning |
|---|---|
| 0 | Success |
| 2 | Unknown command / usage error |
| 44 | Module not found |
| 45 | Input validation failed |

---

## How to Use

SDK implementors should consume the fixtures as follows:

1. **Run as a subprocess.** For each test case, invoke `apcore-cli` as a subprocess, passing `test_case.args` as the argument list. Do not use a shell — pass the array directly to avoid quoting differences.

2. **Capture stdout and stderr separately.** Redirect stdout and stderr into distinct buffers. Do not merge them.

3. **Assert the exit code.** If `expected_exit_code` is present, assert that the process exit code equals that value.

4. **Parse stdout as JSON.** If either `expected_json_output` or `expected_json_keys` is present, parse stdout as JSON.
   - If `expected_output_type` is `"array"`, assert the parsed value is a JSON array.
   - For `expected_json_keys`, assert each listed key exists as a top-level key (or as a key in each array element).
   - For `expected_json_output`, perform a deep partial match: for every key-value pair in the expected object, assert the actual output contains a matching key with a compatible value.

5. **Check stderr.** If `expected_stderr_contains` is present, assert that the stderr buffer contains that substring.

### Example (Python)

```python
import subprocess, json

def run_case(test_case):
    result = subprocess.run(
        ["apcore-cli"] + test_case["args"],
        capture_output=True, text=True
    )

    if "expected_exit_code" in test_case:
        assert result.returncode == test_case["expected_exit_code"], (
            f"[{test_case['id']}] exit code: expected {test_case['expected_exit_code']}, "
            f"got {result.returncode}"
        )

    if "expected_json_output" in test_case or "expected_json_keys" in test_case:
        data = json.loads(result.stdout)

        if test_case.get("expected_output_type") == "array":
            assert isinstance(data, list), f"[{test_case['id']}] expected JSON array"

        for key in test_case.get("expected_json_keys", []):
            target = data[0] if isinstance(data, list) else data
            assert key in target, f"[{test_case['id']}] missing key: {key}"

        for key, value in test_case.get("expected_json_output", {}).items():
            assert data.get(key) == value, (
                f"[{test_case['id']}] field '{key}': expected {value!r}, got {data.get(key)!r}"
            )

    if "expected_stderr_contains" in test_case:
        assert test_case["expected_stderr_contains"] in result.stderr, (
            f"[{test_case['id']}] stderr missing: {test_case['expected_stderr_contains']!r}"
        )
```

---

## Contributing

To add a new test case:

1. Open `fixtures/cli_parity.json`.
2. Append a new object to the `test_cases` array.
3. Choose a unique, descriptive `id` that follows the existing snake_case naming convention (e.g., `exec_with_timeout`).
4. Include only the assertion fields that are meaningful for the case — omit fields you are not asserting.
5. If the new case covers a new exit code or a new command, update the exit code table and command description in this README.
6. Run the conformance suite against the Python, TypeScript, and Rust implementations before opening a pull request. All three must pass.

Do not modify existing test case `id` values — downstream CI pipelines reference them by name.
