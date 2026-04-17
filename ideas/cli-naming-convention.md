# Solution Report: REST API to CLI Command Naming Convention

**Status**: Draft
**Author**: Spec Forge
**Date**: 2026-04-16
**Scope**: apcore ecosystem (apcore-toolkit, apcore-cli, framework scanners, PROTOCOL_SPEC)

---

## 1. Problem Statement

### 1.1 Current Behavior

When `fastapi-apcore` scans a FastAPI application, it generates module IDs that directly embed HTTP methods and URL path segments:

```
FastAPI Routes:                    Generated CLI Commands:
  POST   /tasks/user_data    →    tasks user_data.post
  GET    /tasks/user_data    →    tasks user_data.get
  DELETE /tasks/user_data    →    tasks user_data.delete
  GET    /tasks/user_data/{id} →  tasks user_data.get_by_id.get
```

### 1.2 Problems Identified

| # | Problem | Example | Industry Norm |
|---|---------|---------|---------------|
| P1 | HTTP method exposed as command suffix | `user_data.post` | `create` (semantic verb) |
| P2 | snake_case in command names | `user_data` | `user-data` (kebab-case) |
| P3 | Dot separator shown to user | `user_data.post` | Space-separated subcommands |

**P3 is already solved** by `GroupedModuleGroup` (ADR-08) which splits on `.` to produce space-separated subcommands. This report focuses on P1 and P2.

### 1.3 Desired End State

```bash
mycli tasks user-data create        # POST /tasks/user_data
mycli tasks user-data list          # GET  /tasks/user_data
mycli tasks user-data get --id 123  # GET  /tasks/user_data/{id}
mycli tasks user-data update        # PUT  /tasks/user_data/{id}
mycli tasks user-data delete        # DELETE /tasks/user_data/{id}
```

---

## 2. Industry Reference

### 2.1 Major CLI Tools Survey

| Tool | Framework | Language | Pattern | Example |
|------|-----------|----------|---------|---------|
| kubectl | Cobra | Go | `verb resource [name]` | `kubectl get pods`, `kubectl delete pod foo` |
| gh | Cobra | Go | `resource verb` | `gh pr create`, `gh issue list` |
| aws | custom | Python | `service action-name` | `aws ec2 describe-instances` |
| gcloud | custom | Python | `service resource verb` | `gcloud iam roles create` |
| docker | Cobra | Go | `resource verb` | `docker container ls`, `docker image pull` |
| az | Knack | Python | `group subgroup verb` | `az vm disk attach`, `az group delete` |
| heroku | oclif | TypeScript | `resource:verb` | `heroku apps:create`, `heroku pg:info` |
| stripe | Cobra | Go | `resource verb` | `stripe customers list`, `stripe charges create` |

### 2.2 Common Conventions

1. **Semantic verbs, not HTTP methods**: `create`/`list`/`get`/`update`/`delete`, never `post`/`get`/`put`/`patch`
2. **kebab-case**: all major CLIs use hyphen-separated words (`user-data`, not `user_data`)
3. **Noun-verb or verb-noun**: `resource action` (gh/docker/gcloud) or `action resource` (kubectl/aws)
4. **Hierarchical subcommands via spaces**: `cli group subgroup command`, not dot/colon separators

### 2.3 HTTP Method to Semantic Verb Mapping (Industry Consensus)

| HTTP Method | Path Pattern | Semantic Verb | Alternative |
|-------------|-------------|---------------|-------------|
| GET | `/resources` (collection) | `list` | `ls` |
| GET | `/resources/{id}` (single) | `get` | `show`, `describe` |
| POST | `/resources` | `create` | `add`, `new` |
| PUT | `/resources/{id}` | `update` | `set`, `replace` |
| PATCH | `/resources/{id}` | `patch` | `update` |
| DELETE | `/resources/{id}` | `delete` | `remove`, `rm` |

---

## 3. Architecture Analysis

### 3.1 Relevant Layers in the apcore Ecosystem

```
Layer 0: apcore (Core SDK)
         ├── apcore-python      v0.18.0
         ├── apcore-typescript  v0.18.0
         ├── apcore-rust        v0.18.0
         └── Responsibility: protocol runtime, Registry, Executor

Layer 1: apcore-toolkit (Shared Scanner Utilities)
         ├── apcore-toolkit-python      v0.4.2
         ├── apcore-toolkit-typescript  v0.4.1
         ├── apcore-toolkit-rust        v0.4.1
         └── Responsibility: shared scanner logic, schema extraction,
             SCANNER_VERB_MAP, generate_suggested_alias()  ← NEW

Layer 2: Framework Scanners (consume apcore-toolkit)
         ├── fastapi-apcore     (Python, depends on toolkit>=0.4.1)
         ├── django-apcore      (Python, depends on toolkit>=0.2.0)
         ├── flask-apcore       (Python, depends on toolkit>=0.2.0)
         ├── nestjs-apcore      (TypeScript, depends on toolkit>=0.4.0)
         ├── axum-apcore        (Rust, depends on toolkit=0.4)
         ├── express-apcore     (TypeScript, planned)
         └── Responsibility: framework-specific route extraction,
             call toolkit.generate_suggested_alias() per route

Layer 3: PROTOCOL_SPEC (Cross-Language Core)
         ├── module_id          (canonical, immutable)
         ├── suggested_alias    (scanner-generated, human-friendly)
         ├── annotations        (structured metadata)
         └── Responsibility: define module metadata schema

Layer 4: Display Overlay (binding.yaml)
         ├── display.cli.alias        (CLI surface alias)
         ├── display.cli.group        (CLI surface group override)
         ├── display.default.alias    (cross-surface default)
         └── Responsibility: per-project override of display names

Layer 5: Surface Adapters
         ├── apcore-cli         (Python/Click)    → CLI
         ├── apcore-mcp         (TypeScript)       → MCP tool names
         ├── apcore-a2a         (Go/Rust)          → A2A action names
         └── Responsibility: surface-specific rendering (_to_cli_name, etc.)
```

### 3.2 Current Alias Resolve Chain (ADR-07)

```
display.cli.alias → display.default.alias → suggested_alias → module_id
     (manual)            (manual)             (toolkit+scanner)  (canonical)
```

### 3.3 What Each Layer Knows

| Layer | Knows HTTP Semantics | Knows CLI Conventions | Knows MCP Conventions |
|-------|---------------------|----------------------|----------------------|
| apcore-toolkit | Yes (verb mapping table) | No | No |
| Framework Scanner | Yes (routes, methods, path params) | No | No |
| PROTOCOL_SPEC | No (protocol-agnostic) | No | No |
| binding.yaml | No | Partially (project-level) | Partially |
| apcore-cli | No | Yes (Click, kebab-case) | N/A |
| apcore-mcp | No | N/A | Yes (camelCase/snake_case) |

---

## 4. Solution Design

### 4.1 Core Principle: Separation of Concerns

The two transformations (P1 and P2) belong to different architectural layers because they require different domain knowledge:

| Transformation | Domain Knowledge Required | Correct Layer |
|---------------|--------------------------|---------------|
| P1: `post` → `create` | HTTP method semantics + path parameter awareness | **Scanner** (Layer 1) |
| P2: `user_data` → `user-data` | CLI surface naming convention | **Surface Adapter** (Layer 4) |

### 4.2 Solution Overview

```
┌─────────────────────────────────────────────────────────┐
│  apcore-toolkit (Layer 1 — shared across all scanners)  │
│                                                         │
│  Provides:                                              │
│    SCANNER_VERB_MAP          (normative verb mapping)   │
│    generate_suggested_alias()  (shared algorithm)       │
│    has_path_params()         (path parameter detection)  │
└──────────────────────┬──────────────────────────────────┘
                       │  imported by all framework scanners
                       │
┌──────────────────────▼──────────────────────────────────┐
│  fastapi-apcore Scanner (Layer 2)                       │
│                                                         │
│  Input:  POST /tasks/user_data                          │
│                                                         │
│  Calls:  toolkit.generate_suggested_alias(              │
│            path="/tasks/user_data",                     │
│            method="POST",                               │
│            has_path_params=False                         │
│          )                                              │
│                                                         │
│  Output:                                                │
│    module_id:       tasks.user_data.post    (canonical)  │
│    suggested_alias: tasks.user_data.create  (semantic)   │
│    x-http-method:   POST                   (metadata)    │
└──────────────────────┬──────────────────────────────────┘
                       │
                       │  suggested_alias flows through
                       │  PROTOCOL_SPEC metadata (no changes needed)
                       │
┌──────────────────────▼──────────────────────────────────┐
│  apcore-cli Adapter (Layer 5)                           │
│                                                         │
│  Resolve chain:                                         │
│    display.cli.alias  → (not set, skip)                 │
│    suggested_alias    → "tasks.user_data.create" (hit)  │
│                                                         │
│  CLI Normalization:                                     │
│    _to_cli_name(): user_data → user-data                │
│                                                         │
│  GroupedModuleGroup (existing):                         │
│    tasks.user-data.create → group: tasks                │
│                             subgroup: user-data         │
│                             command: create              │
└──────────────────────┬──────────────────────────────────┘
                       │
                       ▼
            CLI: mycli tasks user-data create
```

---

## 5. Detailed Specifications

### 5.1 Layer 1: apcore-toolkit — HTTP Verb Semantic Mapping

**Affected component**: `apcore-toolkit` (Python v0.4.2, TypeScript v0.4.1, Rust v0.4.1)

**Change**: Add `SCANNER_VERB_MAP` and `generate_suggested_alias()` as shared utilities. All framework scanners already depend on `apcore-toolkit`, so they import these directly — no duplication.

#### 5.1.1 Why apcore-toolkit, Not apcore Core or Each Scanner

| Placement | Pros | Cons |
|-----------|------|------|
| **apcore (core SDK)** | Maximum reach | Core is protocol-agnostic; HTTP verb mapping is HTTP-specific. Pollutes the core layer. |
| **Each scanner independently** | No shared dependency | 6 scanners × 3 languages = 18 copies. Guaranteed drift. |
| **apcore-toolkit** ✅ | All 6 scanners already depend on it. Exists in 3 languages. Its stated purpose is "shared scanner utilities". | None — this is exactly its design intent. |

Current dependency graph confirms the fit:

```
apcore-toolkit-python v0.4.2
    ↑ depended on by:
    ├── fastapi-apcore  (>=0.4.1)
    ├── django-apcore   (>=0.2.0)
    └── flask-apcore    (>=0.2.0)

apcore-toolkit-typescript v0.4.1
    ↑ depended on by:
    └── nestjs-apcore   (>=0.4.0)

apcore-toolkit-rust v0.4.1
    ↑ depended on by:
    └── axum-apcore     (=0.4)
```

#### 5.1.2 Verb Mapping Table

Placed in `apcore-toolkit` as the single source of truth. Each language SDK implements the same table:

**Python** (`apcore_toolkit/scanner_verb_map.py`):

```python
SCANNER_VERB_MAP: dict[str, str] = {
    "GET":     "list",     # collection (no path params)
    "GET_ID":  "get",      # single resource (has path params)
    "POST":    "create",
    "PUT":     "update",
    "PATCH":   "patch",
    "DELETE":  "delete",
    "HEAD":    "head",
    "OPTIONS": "options",
}


def resolve_http_verb(method: str, has_path_params: bool) -> str:
    """Map HTTP method to semantic CLI verb.

    Args:
        method: HTTP method (GET, POST, etc.)
        has_path_params: True if route has path parameters (e.g., {id})

    Returns:
        Semantic verb string (e.g., "create", "list", "get")
    """
    method_upper = method.upper()
    if method_upper == "GET":
        return SCANNER_VERB_MAP["GET_ID"] if has_path_params else SCANNER_VERB_MAP["GET"]
    return SCANNER_VERB_MAP.get(method_upper, method.lower())
```

**TypeScript** (`apcore-toolkit/src/scanner-verb-map.ts`):

```typescript
export const SCANNER_VERB_MAP: Record<string, string> = {
  GET:     "list",
  GET_ID:  "get",
  POST:    "create",
  PUT:     "update",
  PATCH:   "patch",
  DELETE:  "delete",
  HEAD:    "head",
  OPTIONS: "options",
};

export function resolveHttpVerb(method: string, hasPathParams: boolean): string {
  const upper = method.toUpperCase();
  if (upper === "GET") {
    return hasPathParams ? SCANNER_VERB_MAP.GET_ID : SCANNER_VERB_MAP.GET;
  }
  return SCANNER_VERB_MAP[upper] ?? method.toLowerCase();
}
```

**Rust** (`apcore-toolkit/src/scanner_verb_map.rs`):

```rust
pub fn resolve_http_verb(method: &str, has_path_params: bool) -> &'static str {
    match method.to_uppercase().as_str() {
        "GET" if has_path_params => "get",
        "GET" => "list",
        "POST" => "create",
        "PUT" => "update",
        "PATCH" => "patch",
        "DELETE" => "delete",
        "HEAD" => "head",
        "OPTIONS" => "options",
        _ => "unknown",
    }
}
```

#### 5.1.3 Path Parameter Detection

Also provided by `apcore-toolkit` — each language variant handles its framework's path param syntax:

```python
# apcore_toolkit/scanner_utils.py

# Regex covers all major frameworks:
#   FastAPI/OpenAPI: {param}
#   Express/NestJS:  :param
#   Gin:             :param
#   Axum:            :param
_PATH_PARAM_RE = re.compile(r'\{[^}]+\}|:[a-zA-Z_]\w*')


def has_path_params(path: str) -> bool:
    """Check if a URL path contains path parameters."""
    return bool(_PATH_PARAM_RE.search(path))
```

#### 5.1.4 Suggested Alias Generation

The main function that framework scanners call:

```python
# apcore_toolkit/scanner_utils.py

def generate_suggested_alias(path: str, method: str) -> str:
    """Generate a human-friendly suggested_alias from HTTP route info.

    The alias uses dot-separated snake_case segments (protocol convention).
    Surface adapters apply their own naming conventions (e.g., CLI → kebab-case).

    Examples:
      POST   /tasks/user_data         → tasks.user_data.create
      GET    /tasks/user_data         → tasks.user_data.list
      GET    /tasks/user_data/{id}    → tasks.user_data.get
      PUT    /tasks/user_data/{id}    → tasks.user_data.update
      DELETE /tasks/user_data/{id}    → tasks.user_data.delete

    Args:
        path: URL path (e.g., "/tasks/user_data/{id}")
        method: HTTP method (e.g., "POST")

    Returns:
        Dot-separated alias string (e.g., "tasks.user_data.create")
    """
    segments = [
        s for s in path.strip("/").split("/")
        if not _PATH_PARAM_RE.fullmatch(s)
    ]

    path_has_params = has_path_params(path)
    verb = resolve_http_verb(method, path_has_params)

    return ".".join(segments + [verb])
```

#### 5.1.5 Framework Scanner Integration

Each scanner calls the toolkit function — no verb mapping logic needed in the scanner itself:

**fastapi-apcore**:
```python
from apcore_toolkit.scanner_utils import generate_suggested_alias

class FastAPIScanner:
    def _scan_route(self, route: APIRoute) -> ModuleDefinition:
        module_id = self._generate_module_id(route)  # existing logic, unchanged
        suggested_alias = generate_suggested_alias(route.path, route.methods[0])

        return ModuleDefinition(
            module_id=module_id,
            suggested_alias=suggested_alias,
            metadata={
                "x-http-method": route.methods[0],
                "x-http-path": route.path,
            },
            # ... existing fields unchanged ...
        )
```

**django-apcore**:
```python
from apcore_toolkit.scanner_utils import generate_suggested_alias

class DjangoScanner:
    def _scan_view(self, url_pattern, view_func, method: str) -> ModuleDefinition:
        suggested_alias = generate_suggested_alias(url_pattern.pattern, method)
        # ...
```

**nestjs-apcore**:
```typescript
import { generateSuggestedAlias } from "apcore-toolkit";

class NestJSScanner {
    private scanRoute(route: RouteInfo): ModuleDefinition {
        const suggestedAlias = generateSuggestedAlias(route.path, route.method);
        // ...
    }
}
```

**axum-apcore**:
```rust
use apcore_toolkit::scanner_utils::generate_suggested_alias;

fn scan_route(route: &RouteInfo) -> ModuleDefinition {
    let suggested_alias = generate_suggested_alias(&route.path, &route.method);
    // ...
}
```

#### 5.1.6 Cross-Language Conformance

Each language's `apcore-toolkit` SHALL pass the same conformance test suite (input/output pairs):

```json
[
  {"path": "/tasks/user_data", "method": "POST", "expected": "tasks.user_data.create"},
  {"path": "/tasks/user_data", "method": "GET",  "expected": "tasks.user_data.list"},
  {"path": "/tasks/user_data/{id}", "method": "GET", "expected": "tasks.user_data.get"},
  {"path": "/tasks/user_data/{id}", "method": "PUT", "expected": "tasks.user_data.update"},
  {"path": "/tasks/user_data/{id}", "method": "DELETE", "expected": "tasks.user_data.delete"},
  {"path": "/tasks/{id}/archive", "method": "POST", "expected": "tasks.archive"},
  {"path": "/api/v2/users", "method": "GET", "expected": "api.v2.users.list"},
  {"path": "/health", "method": "GET", "expected": "health.list"}
]
```

This JSON fixture is checked into `apcore-toolkit-python/tests/fixtures/` and mirrored in the TypeScript and Rust packages. CI validates all three produce identical output.

### 5.2 Layer 2: PROTOCOL_SPEC — No Changes Required

The `suggested_alias` field already exists in the PROTOCOL_SPEC §5.13 Display Overlay system. The resolve chain (`display.cli.alias` > `display.default.alias` > `suggested_alias` > `module_id`) already supports this field.

**New metadata extensions** (`x-http-method`, `x-http-path`) use the existing `x-` extension mechanism and require no protocol changes.

### 5.3 Layer 3: binding.yaml — No Changes Required

The existing `display.cli.alias` override remains the escape hatch for cases where the scanner's `suggested_alias` is unsuitable:

```yaml
# Only needed for exceptions — most routes need no binding.yaml entry
bindings:
  # Override "list" to "search" for this specific route
  - module_id: "products.product_catalog.get"
    display:
      cli:
        alias: "products.product-catalog.search"
        description: "Search product catalog with filters"

  # Override "get" to "show" for detailed output
  - module_id: "tasks.user_data.get"
    display:
      cli:
        alias: "tasks.user-data.show"
        description: "Show detailed task user data"
```

### 5.4 Layer 5: CLI Adapter — CLI Name Normalization

**Affected component**: All CLI adapters (apcore-cli-python, apcore-cli-typescript, apcore-cli-rust)

**Change**: Apply `snake_case → kebab-case` normalization to group names and command names.

#### 5.4.1 Normalization Rule

```
CLI Surface Naming Convention (normative):

All CLI adapters SHALL convert underscore (_) to hyphen (-) in group names
and command names resolved from the alias chain. This applies uniformly
across all language implementations of the CLI surface.
```

#### 5.4.2 Implementation (apcore-cli, Python)

Location: `GroupedModuleGroup._resolve_group()` in `apcore_cli/cli.py`

```python
def _to_cli_name(name: str) -> str:
    """Normalize a protocol-level name to CLI convention (kebab-case)."""
    return name.replace("_", "-")


class GroupedModuleGroup(LazyModuleGroup):

    def _resolve_group(self, module_id: str, descriptor: Any) -> tuple[str | None, str]:
        # ... existing resolution logic (3-tier priority) ...
        # At the end, before returning:
        group = _to_cli_name(group) if group else None
        cmd = _to_cli_name(cmd)
        return (group, cmd)
```

#### 5.4.3 Implementation (apcore-cli-go, Go/Cobra)

```go
// cli/naming.go
func toCliName(name string) string {
    return strings.ReplaceAll(name, "_", "-")
}
```

#### 5.4.4 Implementation (apcore-cli-rs, Rust/Clap)

```rust
// src/naming.rs
fn to_cli_name(name: &str) -> String {
    name.replace('_', "-")
}
```

#### 5.4.5 Precedence: Schema Parser Consistency

The schema parser already applies this convention for CLI flags (SRS FR-SCHEMA-001):

```python
# schema-parser: property_name → CLI flag
flag_name = "--" + prop_name.replace("_", "-")
# input_file → --input-file
```

The new normalization extends the same convention to command and group names, ensuring consistency across the entire CLI surface.

#### 5.4.6 Why This Is Not a Scanner Responsibility

Different surfaces have different naming conventions:

| Surface | Naming Convention | `user_data` becomes |
|---------|-------------------|---------------------|
| CLI (all languages) | kebab-case | `user-data` |
| MCP (TypeScript) | snake_case or camelCase | `user_data` or `userData` |
| A2A (varies) | depends on protocol | `user_data` |
| REST API | snake_case (path) | `user_data` |

If the scanner emitted kebab-case in `suggested_alias`, non-CLI surfaces would need to reverse the transformation. Each surface should apply its own convention to the neutral snake_case form.

---

## 6. End-to-End Walkthrough

### 6.1 FastAPI Application

```python
# app.py
from fastapi import FastAPI
app = FastAPI()

@app.post("/tasks/user_data")
def create_user_data(payload: UserDataInput): ...

@app.get("/tasks/user_data")
def list_user_data(status: str = "active"): ...

@app.get("/tasks/user_data/{task_id}")
def get_user_data(task_id: int): ...

@app.put("/tasks/user_data/{task_id}")
def update_user_data(task_id: int, payload: UserDataInput): ...

@app.delete("/tasks/user_data/{task_id}")
def delete_user_data(task_id: int): ...
```

### 6.2 Scanner Output

```yaml
modules:
  - module_id: "tasks.user_data.post"
    suggested_alias: "tasks.user_data.create"
    metadata:
      x-http-method: POST
      x-http-path: "/tasks/user_data"

  - module_id: "tasks.user_data.list_user_data.get"
    suggested_alias: "tasks.user_data.list"
    metadata:
      x-http-method: GET
      x-http-path: "/tasks/user_data"

  - module_id: "tasks.user_data.get_user_data__task_id_.get"
    suggested_alias: "tasks.user_data.get"
    metadata:
      x-http-method: GET
      x-http-path: "/tasks/user_data/{task_id}"

  - module_id: "tasks.user_data.update_user_data__task_id_.put"
    suggested_alias: "tasks.user_data.update"
    metadata:
      x-http-method: PUT
      x-http-path: "/tasks/user_data/{task_id}"

  - module_id: "tasks.user_data.delete_user_data__task_id_.delete"
    suggested_alias: "tasks.user_data.delete"
    metadata:
      x-http-method: DELETE
      x-http-path: "/tasks/user_data/{task_id}"
```

### 6.3 Resolve Chain (apcore-cli)

```
For module_id "tasks.user_data.post":

  1. display.cli.alias?     → not set (no binding.yaml entry)
  2. display.default.alias? → not set
  3. suggested_alias?       → "tasks.user_data.create" ← HIT
  4. module_id (fallback)   → (not reached)

  resolved alias: "tasks.user_data.create"
```

### 6.4 CLI Normalization

```
resolved alias:  "tasks.user_data.create"
                       │
  _to_cli_name():      │  user_data → user-data
                       ▼
  CLI alias:     "tasks.user-data.create"
```

### 6.5 Group Resolution (GroupedModuleGroup, group_depth=2)

```
  CLI alias: "tasks.user-data.create"
  Split by ".": ["tasks", "user-data", "create"]
  group_depth=2: take min(2, 3-1)=2 segments as groups
    → group path: ["tasks", "user-data"]
    → command: "create"
```

### 6.6 Final CLI Commands

```bash
$ mycli --help
Usage: mycli [OPTIONS] COMMAND [ARGS]...

Commands:
  list       List available modules.
  describe   Show module details.
  exec       Execute a module by ID.

Groups:
  tasks      (5 commands)

$ mycli tasks --help
Usage: mycli tasks [OPTIONS] COMMAND [ARGS]...

Groups:
  user-data  (5 commands)

$ mycli tasks user-data --help
Usage: mycli tasks user-data [OPTIONS] COMMAND [ARGS]...

Commands:
  create   Create task user data
  list     List task user data
  get      Get task user data by ID
  update   Update task user data
  delete   Delete task user data

$ mycli tasks user-data create --name "report" --priority high
{ "id": 42, "name": "report", "priority": "high", "status": "created" }

$ mycli tasks user-data list --status active
┌────┬────────────┬──────────┬────────┐
│ ID │ Name       │ Priority │ Status │
├────┼────────────┼──────────┼────────┤
│ 42 │ report     │ high     │ active │
│ 17 │ quarterly  │ medium   │ active │
└────┴────────────┴──────────┴────────┘

$ mycli tasks user-data get --task-id 42
{ "id": 42, "name": "report", "priority": "high", "status": "active" }
```

### 6.7 MCP Surface (Same Modules, Different Convention)

```
apcore-mcp reads the same suggested_alias:
  "tasks.user_data.create"

MCP does NOT apply kebab-case. MCP tool names:
  tasks.user_data.create
  tasks.user_data.list
  tasks.user_data.get
  tasks.user_data.update
  tasks.user_data.delete
```

### 6.8 binding.yaml Override (Exception Handling)

```yaml
# Only needed for the minority of routes where auto-naming is insufficient
bindings:
  - module_id: "tasks.user_data.list_user_data.get"
    display:
      cli:
        alias: "tasks.user-data.search"       # override "list" → "search"
        description: "Search user data with advanced filters"
      mcp:
        alias: "tasks.user_data.search"        # MCP surface keeps snake_case
```

---

## 7. Impact Analysis

### 7.1 Changes by Component

| Component | Change | Effort | Breaking |
|-----------|--------|--------|----------|
| **apcore-toolkit** (Python/TS/Rust) | Add `SCANNER_VERB_MAP`, `generate_suggested_alias()`, `has_path_params()` | Medium | No (additive API) |
| **fastapi-apcore** | Call `toolkit.generate_suggested_alias()`, set `suggested_alias` in output | Small | No (additive field) |
| **django-apcore** | Same as fastapi-apcore | Small | No |
| **flask-apcore** | Same as fastapi-apcore | Small | No |
| **nestjs-apcore** | Same as fastapi-apcore (TypeScript toolkit) | Small | No |
| **axum-apcore** | Same as fastapi-apcore (Rust toolkit) | Small | No |
| **PROTOCOL_SPEC** | None | Zero | N/A |
| **binding.yaml schema** | None | Zero | N/A |
| **apcore-cli** (Python/TS/Rust) | Add `_to_cli_name()` in `_resolve_group()` | Small (3 lines per language) | Yes (see §7.2) |
| **apcore-mcp** | None (already uses `suggested_alias` if present) | Zero | N/A |
| **apcore-a2a** | None (already uses `suggested_alias` if present) | Zero | N/A |

### 7.2 Breaking Changes

**CLI command names change** when `_to_cli_name()` is applied:

```
Before:  mycli tasks user_data create
After:   mycli tasks user-data create
```

**Mitigation**:
- Acceptable if introduced in a minor version bump (pre-1.0 semver)
- Projects already using binding.yaml with explicit `display.cli.alias` are unaffected (highest priority in resolve chain)
- `suggested_alias` adoption is non-breaking (modules without it fall through to `module_id`)

### 7.3 Dependency Graph

```
                    apcore (core SDK)
                   ┌──────────────────┐
                   │  Protocol runtime │
                   │  Registry, Executor│
                   └────────┬─────────┘
                            │
                   ┌────────▼─────────┐
                   │  apcore-toolkit   │ ← NEW: SCANNER_VERB_MAP
                   │  (Python/TS/Rust) │        generate_suggested_alias()
                   │                   │        has_path_params()
                   └────────┬─────────┘
                            │ imports (existing dependency)
          ┌─────────┬───────┼───────┬──────────┐
          │         │       │       │          │
   fastapi-    django-  flask-  nestjs-    axum-
   apcore      apcore   apcore  apcore     apcore
   (Py)        (Py)     (Py)    (TS)       (Rust)
          │         │       │       │          │
          └─────────┴───────┼───────┴──────────┘
                            │ emits suggested_alias
                            │ in module metadata
                            │
              ┌─────────────┼─────────────┐
              │             │             │
        apcore-cli    apcore-mcp     apcore-a2a
        (kebab-case)  (snake_case)   (varies)
        _to_cli_name  (no transform) (surface-specific)
```

**Cross-language consistency**: Each language's `apcore-toolkit` (Python, TypeScript, Rust) implements the same `SCANNER_VERB_MAP` and validates against the shared conformance fixture (§5.1.6). No framework scanner implements verb mapping independently.

---

## 8. Edge Cases

### 8.1 Verb Collision

Two routes with different methods but the same semantic verb:

```python
@app.get("/tasks")        # → tasks.list
@app.get("/tasks/{id}")   # → tasks.get
# No collision: different verbs (list vs get)
```

### 8.2 Custom Action Routes

Routes that don't follow standard CRUD patterns:

```python
@app.post("/tasks/{id}/archive")    # Not a standard CRUD
@app.post("/tasks/{id}/reassign")   # Not a standard CRUD
```

**Handling**: Scanner uses the last path segment as the verb when it does not match standard CRUD patterns:

```python
def _generate_suggested_alias(path, method, route):
    segments = [s for s in path.strip("/").split("/") if not _is_param(s)]
    # If last segment is a known action word, use it directly
    last = segments[-1] if segments else method.lower()
    if _is_crud_verb(method, route):
        verb = SCANNER_VERB_MAP[method](route)
        return ".".join(segments + [verb])
    else:
        # Non-CRUD: last segment IS the verb (archive, reassign, etc.)
        # Don't append another verb
        return ".".join(segments)
```

Result:
```
POST /tasks/{id}/archive  → suggested_alias: "tasks.archive"
POST /tasks/{id}/reassign → suggested_alias: "tasks.reassign"

CLI: mycli tasks archive --id 42
CLI: mycli tasks reassign --id 42 --to user@example.com
```

### 8.3 Nested Resources

```python
@app.get("/organizations/{org_id}/teams/{team_id}/members")
```

```
module_id:        organizations.teams.members.get
suggested_alias:  organizations.teams.members.list

CLI (group_depth=3): mycli organizations teams members list
CLI (group_depth=2): mycli organizations teams members-list
CLI (group_depth=1): mycli organizations teams.members.list
```

Recommendation: `group_depth=2` (FR-11-10 default) handles most cases. Deeply nested resources (3+ levels) should use `display.cli.group` override.

### 8.4 API Versioned Routes

```python
@app.get("/api/v2/users")
```

```
suggested_alias: "api.v2.users.list"
CLI (group_depth=2): mycli api v2 users-list   # awkward

# Better: binding.yaml override
display.cli.alias: "users.list"                # skip api/v2 prefix
CLI: mycli users list
```

### 8.5 Hyphen in URL Path

```python
@app.get("/user-data/reports")    # path already uses hyphens
```

```
module_id:        user_data.reports.get     # canonical: underscores
suggested_alias:  user_data.reports.list

# _to_cli_name() converts _ → -
CLI: mycli user-data reports list           # correct
```

### 8.6 GET Collection vs GET Single: Ambiguous Routes

```python
@app.get("/tasks/user_data")          # collection → list
@app.get("/tasks/user_data/{id}")     # single → get
```

Both routes share the path prefix `tasks/user_data`. The scanner disambiguates via path parameter detection:

```
No path params  → list
Has path params → get
```

No collision because the verbs differ.

---

## 9. Migration Strategy

### 9.1 Phase 1: apcore-toolkit Update (Non-Breaking)

1. Add `SCANNER_VERB_MAP`, `generate_suggested_alias()`, `has_path_params()` to:
   - `apcore-toolkit-python` → v0.5.0
   - `apcore-toolkit-typescript` → v0.5.0
   - `apcore-toolkit-rust` → v0.5.0
2. Add shared conformance test fixture (`scanner_verb_map_fixture.json`)
3. All three language implementations pass the conformance test
4. No existing API changes — purely additive

### 9.2 Phase 2: Framework Scanner Integration (Non-Breaking)

1. Update each scanner to call `toolkit.generate_suggested_alias()`:
   - `fastapi-apcore` → calls toolkit, emits `suggested_alias`
   - `django-apcore` → same
   - `flask-apcore` → same
   - `nestjs-apcore` → same
   - `axum-apcore` → same
2. `suggested_alias` is an additive field — existing consumers are unaffected
3. CLI and MCP adapters automatically pick up `suggested_alias` via existing resolve chain

### 9.3 Phase 3: CLI Kebab Normalization (Minor Breaking)

1. Add `_to_cli_name()` to all CLI adapters:
   - `apcore-cli-python`
   - `apcore-cli-typescript`
   - `apcore-cli-rust`
2. Command names change from `user_data` → `user-data`
3. Release as part of the next minor version (pre-1.0)
4. Document the change in CHANGELOG

### 9.4 Phase 4: Multi-Level Grouping (Already Planned)

1. Implement FR-11-10 (multi-level nested grouping, `group_depth`)
2. Enables `tasks user-data create` instead of `tasks user-data.create`
3. Already in the roadmap at P2 priority

---

## 10. Verification

### 10.1 Scanner Tests

| ID | Input | Expected `suggested_alias` |
|----|-------|---------------------------|
| T-SCAN-01 | `POST /tasks/user_data` | `tasks.user_data.create` |
| T-SCAN-02 | `GET /tasks/user_data` | `tasks.user_data.list` |
| T-SCAN-03 | `GET /tasks/user_data/{id}` | `tasks.user_data.get` |
| T-SCAN-04 | `PUT /tasks/user_data/{id}` | `tasks.user_data.update` |
| T-SCAN-05 | `PATCH /tasks/user_data/{id}` | `tasks.user_data.patch` |
| T-SCAN-06 | `DELETE /tasks/user_data/{id}` | `tasks.user_data.delete` |
| T-SCAN-07 | `POST /tasks/{id}/archive` | `tasks.archive` |
| T-SCAN-08 | `GET /api/v2/users` | `api.v2.users.list` |
| T-SCAN-09 | `GET /health` | `health.list` or `health` (single-segment) |
| T-SCAN-10 | `POST /users` with no path params | `users.create` |

### 10.2 CLI Normalization Tests

| ID | Input (resolved alias) | Expected CLI group + command |
|----|----------------------|------------------------------|
| T-CLI-01 | `tasks.user_data.create` | group=`tasks`, subgroup=`user-data`, cmd=`create` |
| T-CLI-02 | `math.add` | group=`math`, cmd=`add` |
| T-CLI-03 | `admin.user_role.list` | group=`admin`, subgroup=`user-role`, cmd=`list` |
| T-CLI-04 | `standalone` | top-level cmd=`standalone` |
| T-CLI-05 | `health_check` (top-level) | top-level cmd=`health-check` |

### 10.3 End-to-End Tests

| ID | FastAPI Route | Expected CLI Invocation |
|----|--------------|------------------------|
| T-E2E-01 | `POST /tasks/user_data` | `mycli tasks user-data create` |
| T-E2E-02 | `GET /tasks/user_data` | `mycli tasks user-data list` |
| T-E2E-03 | `GET /tasks/user_data/{id}` | `mycli tasks user-data get --task-id 42` |
| T-E2E-04 | `PUT /tasks/user_data/{id}` | `mycli tasks user-data update --task-id 42 --name new` |
| T-E2E-05 | `DELETE /tasks/user_data/{id}` | `mycli tasks user-data delete --task-id 42` |
| T-E2E-06 | `POST /tasks/{id}/archive` | `mycli tasks archive --id 42` |

---

## 11. Decision Summary

| Decision | Choice | Rationale |
|----------|--------|-----------|
| Where does HTTP verb → semantic verb mapping live? | **apcore-toolkit** (Layer 1) | All 6 scanners already depend on it. Exists in 3 languages. Its stated purpose is "shared scanner utilities". |
| Where does snake_case → kebab-case live? | **CLI adapter layer** (Layer 5) | CLI-specific convention. Other surfaces (MCP, A2A) have different naming conventions. |
| How to ensure cross-scanner consistency? | **Shared conformance test fixture** in apcore-toolkit | JSON fixture checked into all 3 language toolkits. CI validates identical output. |
| How to handle exceptions? | **binding.yaml `display.cli.alias` override** (Layer 4) | Existing mechanism, no new features needed. |
| PROTOCOL_SPEC changes? | **None** | `suggested_alias` and `x-` extensions already supported. |
| Does `_` → `-` go in suggested_alias? | **No** | `suggested_alias` is cross-surface (toolkit output). Keep snake_case as neutral form. Each surface adapter applies its own convention. |
| Why not put verb map in each web SDK? | **Rejected** | 6 scanners × 3 languages = 18 copies. Guaranteed drift. apcore-toolkit exists precisely for this purpose. |
| Why not put verb map in apcore core? | **Rejected** | Core is protocol-agnostic. HTTP verb semantics is transport-layer knowledge, belongs in toolkit (scanner utilities layer). |
