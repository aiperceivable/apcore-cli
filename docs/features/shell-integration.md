# Feature Spec: Shell Integration

**Feature ID**: FE-06
**Status**: Ready for Implementation
**Priority**: P2
**Parent**: [Tech Design v1.0](../tech-design.md) Section 8.7
**SRS Requirements**: FR-SHELL-001, FR-SHELL-002

---

## 1. Description

Shell Integration provides two subcommands: `apcore-cli completion <shell>` for generating shell completion scripts (Bash, Zsh, Fish) and `apcore-cli man <command>` for generating man pages in roff format. Both commands leverage module metadata from the Registry for dynamic content generation.

---

## 2. Requirements Traceability

| Req ID | SRS Ref | Description |
|--------|---------|-------------|
| FR-06-01 | FR-SHELL-001 | Shell completion script generation for bash, zsh, fish. |
| FR-06-02 | FR-SHELL-002 | Man page generation in roff format. |

---

## 3. Module Path

`apcore_cli/shell.py`

---

## 4. Implementation Details

### 4.1 Command: `completion`

**Registration**: `@cli.command("completion")`

**Signature**: `completion(shell: str) -> None`

**Click decorators**:
```python
@click.argument("shell", type=click.Choice(["bash", "zsh", "fish"]))
```

Logic steps:
1. Resolve `resolved = ctx.find_root().info_name or prog_name`.
2. If `shell == "bash"`: call `_generate_bash_completion(resolved)` and `click.echo()` the result.
3. If `shell == "zsh"`: call `_generate_zsh_completion(resolved)` and `click.echo()` the result.
4. If `shell == "fish"`: call `_generate_fish_completion(resolved)` and `click.echo()` the result.
5. Exit code 0.


### 4.2 Helper: `_make_function_name`

**Signature**: `_make_function_name(prog_name: str) -> str`

Converts `prog_name` to a valid POSIX shell identifier by replacing any character that is not `[a-zA-Z0-9]` with `_` and prepending `_`. Example: `"my-tool"` → `"_my_tool"`.

### 4.3 Function: `_generate_bash_completion`

**Signature**: `_generate_bash_completion(prog_name: str) -> str`

Logic steps:
1. Compute shell function name via `_make_function_name(prog_name)`.
2. Build the inline module-list shell command as `"<prog_name>" list --format json 2>/dev/null | python3 -c "..."`. `prog_name` is double-quoted in the embedded command to handle names that contain spaces.
3. Generate the Bash completion script body (example for `prog_name="apcore-cli"`):
   ```bash
   _apcore_cli() {
       local cur prev opts
       COMPREPLY=()
       cur="${COMP_WORDS[COMP_CWORD]}"
       prev="${COMP_WORDS[COMP_CWORD-1]}"

       if [[ ${COMP_CWORD} -eq 1 ]]; then
           opts="exec list describe completion man"
           COMPREPLY=( $(compgen -W "${opts}" -- ${cur}) )
           return 0
       fi

       if [[ "${COMP_WORDS[1]}" == "exec" && ${COMP_CWORD} -eq 2 ]]; then
           local modules=$("apcore-cli" list --format json 2>/dev/null | python3 -c "import sys,json;[print(m['id']) for m in json.load(sys.stdin)]" 2>/dev/null)
           COMPREPLY=( $(compgen -W "${modules}" -- ${cur}) )
           return 0
       fi
   }
   complete -F _apcore_cli apcore-cli
   ```
4. Return the script as a string.

### 4.4 Function: `_generate_zsh_completion`

**Signature**: `_generate_zsh_completion(prog_name: str) -> str`

Logic steps:
1. Compute shell function name via `_make_function_name(prog_name)`.
2. Generate Zsh completion function using `compdef` and `compadd`. First-level completions: `exec`, `list`, `describe`, `completion`, `man`. Second-level (after `exec`): dynamic module IDs via inline command with `prog_name` double-quoted.
3. Emit `compdef {fn} {prog_name}` to register the function.
4. Return the script as a string.

### 4.5 Function: `_generate_fish_completion`

**Signature**: `_generate_fish_completion(prog_name: str) -> str`

Logic steps:
1. Generate Fish completions using `complete -c {prog_name}` commands. Subcommands: `exec`, `list`, `describe`, `completion`, `man`.
2. For `exec` second-level completion: inline command with `prog_name` double-quoted.
3. Return the script as a string.

### 4.6 Function: `register_shell_commands`

**Signature**: `register_shell_commands(cli: click.Group, prog_name: str = "apcore-cli") -> None`

Registers the `completion` and `man` commands on `cli`. The `prog_name` parameter is captured in closures so the completion generators and man page use the correct program name at runtime (resolved from `ctx.find_root().info_name` when available, otherwise from the `prog_name` argument).

### 4.7 Command: `man`

**Registration**: `@cli.command("man")`

**Signature**: `man_cmd(command: str) -> None`

**Click decorators**:
```python
@click.argument("command")
```

Logic steps:
1. Look up `command` in the CLI command registry (built-in subcommands and module IDs).
2. If not found: write to stderr "Error: Unknown command '{command}'." Exit 2.
3. Build roff document:
   a. `.TH "{PROG}-{COMMAND}" "1" "{date}" "{prog} {version}" "{prog} Manual"` — all names are dynamic from `prog_name`.
   b. `.SH NAME` section: `{prog}-{command} \\- {brief_description}` (first line of command help, trailing period stripped).
   c. `.SH SYNOPSIS` section: dynamically built from the command's actual Click parameters. For each `click.Option`: `[--flag \\fITYPE\\fR]` (required options omit brackets); for each `click.Argument`: `\\fIMETA\\fR` (optional arguments get brackets). Not a static `[OPTIONS] [ARGUMENTS]` placeholder.
   d. `.SH DESCRIPTION` section: full command description from Click help (if present).
   e. `.SH OPTIONS` section: for each Click option:
      - `.TP` with `\\fB{flag_name}\\fR \\fI{type}\\fR` (flags use `\\fB{flag_name}\\fR` only).
      - Help text on next line, followed by `Default: {value}.` if a non-flag default exists.
   f. `.SH ENVIRONMENT` section: four standard env vars:
      - `APCORE_EXTENSIONS_ROOT` — path to extensions directory, overrides default `./extensions`.
      - `APCORE_CLI_AUTO_APPROVE` — set to `1` to bypass approval prompts.
      - `APCORE_CLI_LOGGING_LEVEL` — CLI-specific log verbosity; takes priority over `APCORE_LOGGING_LEVEL`. One of: `DEBUG`, `INFO`, `WARNING`, `ERROR`. Default: `WARNING`.
      - `APCORE_LOGGING_LEVEL` — global apcore log verbosity; used as fallback when `APCORE_CLI_LOGGING_LEVEL` is not set.
   g. `.SH EXIT CODES` section: table of all exit codes:
      | Code | Meaning |
      |------|---------|
      | 0 | Success. |
      | 1 | Module execution error. |
      | 2 | Invalid CLI input or missing argument. |
      | 44 | Module not found, disabled, or failed to load. |
      | 45 | Input failed JSON Schema validation. |
      | 46 | Approval denied, timed out, or no interactive terminal available. |
      | 47 | Configuration error (extensions directory not found or unreadable). |
      | 48 | Schema contains a circular `$ref`. |
      | 77 | ACL denied — insufficient permissions for this module. |
      | 130 | Execution cancelled by user (SIGINT / Ctrl-C). |
   h. `.SH SEE ALSO` section: links to sibling man pages using `prog_name`.
4. Print the roff document to stdout.
5. Exit code 0.

**Usage for immediate display:**
```bash
apcore-cli man exec | man -l -
```

---

## 5. Parameter Validation

| Parameter | Type | Valid Values | Invalid Handling | SRS Reference |
|-----------|------|-------------|------------------|---------------|
| `shell` (completion) | `click.Choice` | `"bash"`, `"zsh"`, `"fish"` | Click rejects with "Invalid value". Exit 2. | FR-SHELL-001 AF-1 |
| `command` (man) | `str` | Any valid subcommand or module ID | "Error: Unknown command '{command}'." Exit 2. | FR-SHELL-002 AF-1 |

---

## 6. Error Handling

| Condition | Exit Code | Error Message | SRS Reference |
|-----------|-----------|---------------|---------------|
| Unsupported shell | 2 | Click auto: "Invalid value for 'SHELL'..." | FR-SHELL-001 AF-1 |
| Unknown command for man | 2 | "Error: Unknown command '{command}'." | FR-SHELL-002 AF-1 |

---

## 7. Verification

| Test ID | Description | Expected Result |
|---------|-------------|-----------------|
| T-SHELL-01 | `apcore-cli completion bash` | stdout: valid Bash completion script containing "exec", "list", "describe". Exit 0. |
| T-SHELL-02 | `apcore-cli completion zsh` | stdout: valid Zsh completion script. Exit 0. |
| T-SHELL-03 | `apcore-cli completion fish` | stdout: valid Fish completion script. Exit 0. |
| T-SHELL-04 | `apcore-cli completion invalid` | Click rejects. Exit 2. |
| T-SHELL-05 | Bash completion sourced, type `apcore-cli exec math.<TAB>` | `math.add` appears as completion candidate. |
| T-SHELL-06 | `apcore-cli man exec` | stdout: roff format with `.TH`, command name. Exit 0. |
| T-SHELL-07 | `apcore-cli man exec \| man -l -` | Man page renders correctly. |
| T-SHELL-08 | `apcore-cli man nonexistent` | stderr: "Unknown command". Exit 2. |
