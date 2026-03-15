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
1. If `shell == "bash"`: call `_generate_bash_completion()` and `click.echo()` the result.
2. If `shell == "zsh"`: call `_generate_zsh_completion()` and `click.echo()` the result.
3. If `shell == "fish"`: call `_generate_fish_completion()` and `click.echo()` the result.
4. Exit code 0.

### 4.2 Function: `_generate_bash_completion`

**Signature**: `_generate_bash_completion() -> str`

Logic steps:
1. Use Click's `_bashcomplete` module as the base template.
2. Override the completion function to include:
   a. Static subcommands: `exec`, `list`, `describe`, `completion`, `man`.
   b. Dynamic module IDs: invoke `apcore-cli list --format json` internally, extract `id` fields.
   c. Dynamic module flags: for `exec <module_id>`, invoke `apcore-cli exec <module_id> --help` internally, parse flag names.
3. Generate the Bash completion script body:
   ```bash
   _apcore_cli_completion() {
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
           local modules=$(apcore-cli list --format json 2>/dev/null | python3 -c "import sys,json;[print(m['id']) for m in json.load(sys.stdin)]" 2>/dev/null)
           COMPREPLY=( $(compgen -W "${modules}" -- ${cur}) )
           return 0
       fi
   }
   complete -F _apcore_cli_completion apcore-cli
   ```
4. Return the script as a string.

### 4.3 Function: `_generate_zsh_completion`

**Signature**: `_generate_zsh_completion() -> str`

Logic steps:
1. Generate Zsh completion function using `compdef` and `compadd`.
2. Include subcommand completion and dynamic module ID completion.
3. Return the script as a string.

### 4.4 Function: `_generate_fish_completion`

**Signature**: `_generate_fish_completion() -> str`

Logic steps:
1. Generate Fish completions using `complete -c apcore-cli` commands.
2. Include subcommand and dynamic module ID completion.
3. Return the script as a string.

### 4.5 Command: `man`

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
   a. `.TH "APCORE-CLI-{COMMAND}" "1" "{date}" "apcore-cli {version}" "apcore-cli Manual"`.
   b. `.SH NAME` section: `apcore-cli-{command} \\- {brief_description}`.
   c. `.SH SYNOPSIS` section: `\\fBapcore-cli {command}\\fR [OPTIONS] [ARGUMENTS]`.
   d. `.SH DESCRIPTION` section: full command description from Click help.
   e. `.SH OPTIONS` section: for each Click option:
      - `.TP` with `\\fB{flag_name}\\fR \\fI{type}\\fR`.
      - Help text on next line.
   f. `.SH EXIT CODES` section: table of all exit codes from error taxonomy.
   g. `.SH SEE ALSO` section: `\\fBapcore-cli\\fR(1), \\fBapcore-cli-list\\fR(1), \\fBapcore-cli-describe\\fR(1)`.
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
