# Host Integration Model

PEPL is designed to be consumed by any host application that needs deterministic, sandboxed interactive programs. The host is responsible for:

| Host Responsibility | What It Does |
|---|---|
| **WASM Sandbox** | Executes compiled PEPL WASM, provides deterministic PRNG seed + event timestamp |
| **View Layer** | Renders abstract UI surfaces to platform-native elements (DOM, native widgets, etc.) |
| **Intent Layer** (optional) | Generates PEPL code from structured intent (e.g., via LLM) |
| **Validation Layer** (optional) | References PEPL spec + stdlib to validate generated code |
| **Lifecycle Manager** | Manages space creation, versioning, and state persistence |

PEPL itself requires only a standard WASM runtime (`WebAssembly.instantiate`, `WebAssembly.validate`).

## What Consumes PEPL

| Consumer | Relationship |
|---|---|
| Host applications | Orchestrate space lifecycle, render UI, manage execution |
| WASM sandboxes | Execute compiled PEPL WASM with capability enforcement |
| Every space | Written in PEPL, compiled to WASM |
| pepl-playground | Editor + live execution environment |
| Third-party apps | Embed PEPL via the WASM import/export contract below |

## WASM Import/Export Contract

Any host that runs compiled PEPL MUST implement this WASM interface. This is the embedding contract — the only thing a host needs to know to execute PEPL spaces.

**Imports (host provides to PEPL WASM module):**

| Import | Signature | Description |
|---|---|---|
| `env.host_call` | `(cap_id: i32, fn_id: i32, payload_ptr: i32, payload_len: i32) -> i32` | All capability calls (http, storage, etc.) route through this single function. Returns pointer to response. |
| `env.get_timestamp` | `() -> i64` | Deterministic event timestamp (milliseconds). Host controls this — never the system clock. |
| `env.log` | `(ptr: i32, len: i32) -> void` | Debug logging. No-op in production. |
| `env.trap` | `(code: i32) -> void` | Runtime trap (assertion failure, division by zero, out of bounds). Host terminates execution. |

**Exports (PEPL WASM module provides to host):**

| Export | Signature | Description |
|---|---|---|
| `init` | `() -> void` | Initialize state to defaults. Called once on space creation. |
| `dispatch_action` | `(action_id: i32, payload_ptr: i32, payload_len: i32) -> void` | Trigger a named action. Host maps UI events → action IDs. |
| `render` | `() -> i32` | Return view tree as serialized JSON pointer. Host reads from WASM memory. |
| `get_state` | `() -> i32` | Return serialized state pointer. Host reads for persistence/event logging. |
| `update` | `(dt: f64) -> void` | Game loop tick (only exported if space declares `update`). |
| `handle_event` | `(event_ptr: i32, event_len: i32) -> void` | Game loop events (only exported if space declares `handleEvent`). |
| `alloc` | `(size: i32) -> i32` | Allocate bytes in WASM linear memory. Host uses this to write payloads. |
| `dealloc` | `(ptr: i32, size: i32) -> void` | Free allocated memory. |

**Memory Protocol:**

```
Host → WASM:  host calls alloc(n) → gets ptr → writes payload at ptr → calls dispatch_action/handle_event
WASM → Host:  render()/get_state() returns ptr → host reads UTF-8 JSON from WASM memory at ptr
```

All payloads are **JSON-encoded UTF-8 strings**. The view tree JSON schema:

```json
{
  "type": "Column",
  "props": {},
  "children": [
    { "type": "Text", "props": { "value": "Hello" }, "children": [] },
    { "type": "Button", "props": { "label": "Click", "on_tap": "increment" }, "children": [] }
  ]
}
```

**Capability ID mapping** (for `env.host_call`):

| cap_id | Capability | fn_id values |
|---|---|---|
| 1 | http | 1=get, 2=post, 3=put, 4=patch, 5=delete |
| 2 | storage | 1=get, 2=set, 3=delete, 4=keys |
| 3 | location | 1=current |
| 4 | notifications | 1=send |
| 5 | credential | 1=get |

> **Note:** Capability ID 5 (`credential`) is used internally by the host to resolve `credentials {}` block bindings. PEPL code does not call it directly.

**Embedding Quickstart:**

A minimal host needs to:
1. Compile PEPL source to WASM (using the PEPL compiler, itself a WASM module)
2. Instantiate the WASM module providing the 4 imports above
3. Call `init()` to initialize state
4. Call `render()` to get the view tree → render to platform UI
5. When user interacts → call `dispatch_action()` → call `render()` again → update UI
6. Implement `env.host_call` for any capabilities the space requires

## Error Format

`compile()` returns structured errors in `errors.errors[]`. Each error is a JSON object:

| Field | Type | Description |
|---|---|---|
| `file` | string | Source filename |
| `line` | number | Error start line |
| `column` | number | Error start column |
| `end_line` | number | Error end line |
| `end_column` | number | Error end column |
| `severity` | `"error"` or `"warning"` | Error severity |
| `code` | string (e.g. `"E201"`) | PEPL error code |
| `category` | string | Error category for grouping |
| `message` | string | Human-readable error description |
| `suggestion` | string (optional) | Suggested fix |
| `source_line` | string | Source line for context display |

**Limits:**
- Max **20 errors** per compilation (compiler caps at 20, drops further errors but tracks `total_errors`)
- Phase 0 has no warnings (reserved for Phase 1+)

Error codes are documented in the PEPL-Lang error reference. Hosts can use error codes for automated repair prompts (e.g., E201 = type mismatch, E402 = unknown component).

## Test Discovery

Compiled PEPL WASM modules export test functions using a numbering convention:

```
__test_count() → i32     // number of test cases
__test_0() → void        // run test 0 (traps on assertion failure)
__test_1() → void        // run test 1
...
__test_{N-1}() → void
```

Hosts discover tests by calling `__test_count()`, then run each `__test_N()`. Success = void return. Failure = WASM trap (assertion failed).

Test names are available via the source map custom section (the compiler records `FuncKind::Test` entries with descriptions).

### `with_responses` (Capability Mocking)

Tests that mock capability calls (`with_responses { http.get("...") -> "..." }`) work in the **evaluator** path but are **not compiled to WASM**. Hosts should distinguish:

- **Pure tests** (no capabilities): run via WASM `__test_N()` exports
- **Capability tests** (with `with_responses`): run via the evaluator (tree-walking interpreter), which fully supports mock response injection

The evaluator is the golden reference for correctness — eval↔codegen parity is tested.

## Structural Limits

The PEPL compiler enforces these structural limits:

| Limit | Value |
|---|---|
| Max compiler errors | 20 per compilation |
| Max lambda nesting | 3 |
| Max record nesting | 4 |
| Max expression depth | 16 |
| Max `for` nesting | 3 |
| Max params per action/fn | 8 |
| Max active timers | 8 per Space |
| Min timer interval | 100ms |
| Max JSON parse depth | 32 |

## Recommended Libraries

The PEPL compiler uses a **hand-written lexer and recursive-descent parser** — no parser generator.

| Component | Library | License | Why |
|---|---|---|---|
| Lexer/Parser | **Hand-written** (Rust) | — | Full control over error recovery, error messages, and LLM-friendly diagnostics |
| WASM emission | **wasm-encoder** 0.225 (Rust, bytecodealliance) | Apache 2.0 | Official Bytecode Alliance crate, type-safe WASM module construction |
| WASM validation | **wasmparser** 0.225 (Rust, bytecodealliance) | Apache 2.0 | Validates WASM binary structure and types before execution |
| WASM-JS bridge | **wasm-bindgen** 0.2 | MIT/Apache 2.0 | Rust↔JavaScript interop for browser compilation |
