# PEPL Reference

> Single-source cross-project reference for the PEPL language and compiler.
> Consumed by `pepl-playground` and any host embedding the PEPL compiler.
> Ground truth: source code in `pepl/`, `pepl-stdlib/`, `pepl-ui/`.

**Language version:** 0.1.0 (Phase 0)
**Current phase:** Phase 0 complete (27 ORCHESTRATION steps, milestones M1–M6 passed)

---

## §1 Identity

> Source: [docs/overview.md](docs/overview.md)

PEPL is a **deterministic, sandboxed, LLM-first language** that compiles to WebAssembly. It builds **Spaces** — self-contained interactive applications. Every syntax decision is optimized for LLM code generation accuracy: keyword-based operators (`not`/`and`/`or`), explicit `set` for mutation, mandatory type annotations, flat block structure, no imports, no filesystem.

### Repository Structure

| Repo | Crate(s) | Purpose |
|------|----------|---------|
| [`pepl/`](pepl/) | 7 crates: `pepl-lexer`, `pepl-types`, `pepl-compiler`, `pepl-codegen`, `pepl-eval`, `pepl-wasm`, `pepl-ast-diff` | Compiler: source → `.wasm` binary |
| [`pepl-stdlib/`](pepl-stdlib/) | 1 crate: `pepl-stdlib` | Standard library: 100 functions + 2 constants |
| [`pepl-ui/`](pepl-ui/) | 1 crate: `pepl-ui` | UI component model: 10 Phase 0 components |

### Test Counts

| Repo | Tests |
|------|-------|
| `pepl/` | ~577 |
| `pepl-stdlib/` | ~511 |
| `pepl-ui/` | ~354 |

---

## §2 Type System

> Source: [pepl/crates/pepl-types/src/lib.rs](pepl/crates/pepl-types/src/lib.rs) · [pepl/crates/pepl-compiler/src/checker.rs](pepl/crates/pepl-compiler/src/checker.rs) · [docs/language-structure.md](docs/language-structure.md)

### Primitive Types

| Type | Description |
|------|-------------|
| `number` | 64-bit IEEE 754 float (NaN prevented — traps instead) |
| `string` | UTF-8 immutable string |
| `bool` | `true` or `false` |
| `nil` | Absence of value |
| `color` | String hex literal (e.g., `"#FF5733"`) — no `color` module in Phase 0 |

### Compound Types

| Type | Description |
|------|-------------|
| `list<T>` | Ordered immutable collection. `T` is built-in only, not user-definable. |
| Records | Named or anonymous: `{ name: string, age: number }`. Spread: `{ ...base, field: val }` |
| `Result<T, E>` | Sum type: `Ok(value)` or `Err(error)`. Unwrap via `?` (traps on Err) or `match`. |
| `Surface` | View return type — abstract UI tree |
| `InputEvent` | Game loop event input type |

### Key Type Rules

- **Nil narrowing:** In `if expr != nil { ... }`, `expr` narrows from `T | nil` to `T` inside the block
- **Nil coalescing:** `expr ?? default` — use default if expr is nil
- **No user-defined generics** — only `list<T>` and `Result<T, E>` are parameterized
- **Structural equality** — `==`/`!=` compare by value (deep comparison for records/lists)

---

## §3 Reserved Words

> Source: [pepl/crates/pepl-lexer/src/token.rs](pepl/crates/pepl-lexer/src/token.rs)

39 keywords + 13 module names = **52 reserved identifiers** (cannot be shadowed by user code).

**Keywords (39):** `space`, `state`, `action`, `view`, `set`, `let`, `if`, `else`, `for`, `in`, `match`, `return`, `invariant`, `capabilities`, `required`, `optional`, `credentials`, `derived`, `tests`, `test`, `assert`, `fn`, `type`, `true`, `false`, `nil`, `not`, `and`, `or`, `number`, `string`, `bool`, `list`, `color`, `update`, `handleEvent`, `Result`, `Surface`, `InputEvent`

**Module names (13):** `core`, `math`, `string`, `list`, `record`, `time`, `convert`, `json`, `timer`, `http`, `storage`, `location`, `notifications`

> **Contextual field names:** All 52 keywords are valid as record field names — in record types (`{ color: string }`), record literals (`{ color: "#f00" }`), state/derived declarations, and `set` path segments after `.`. Keywords remain reserved in all other positions.

---

## §4 Standard Library

> Source: [pepl/crates/pepl-compiler/src/stdlib.rs](pepl/crates/pepl-compiler/src/stdlib.rs) · [docs/phase-0-stdlib-reference.md](docs/phase-0-stdlib-reference.md)

100 functions + 2 constants across 13 modules (9 pure + 4 capability).

### Module Summary

| Module | Count | Category | Key Constraint |
|--------|-------|----------|----------------|
| `core` | 4 | Pure | `core.log` is no-op in production |
| `math` | 10 + 2 constants | Pure | All pure, no NaN output (trap instead) |
| `string` | 20 | Pure | UTF-8, no locale-dependent behavior |
| `list` | 34 | Pure | All return new lists (immutable), `list.of` is variadic |
| `record` | 5 | Pure | `record.get` is compile-time checked for known records |
| `time` | 5 | Pure | `time.now` is host-provided (deterministic on replay) |
| `convert` | 5 | Pure | Fallible conversions return `Result<T, ConvertError>` |
| `json` | 2 | Pure | Max depth: 32. Returns `Result<any, JsonError>` |
| `timer` | 4 | Capability | Host-managed scheduling, max 8 active timers |
| `http` | 5 | Capability | All return `Result<string, HttpError>` |
| `storage` | 4 | Capability | Key-value, host-mediated |
| `location` | 1 | Capability | Geolocation, permission-gated |
| `notifications` | 1 | Capability | Permission-gated |

### `core` module (4 functions) — [source](pepl-stdlib/src/modules/core.rs)

| Function | Signature | Description |
|----------|-----------|-------------|
| `core.log` | `(value: any) -> nil` | Debug output (no-op in production) |
| `core.assert` | `(condition: bool, message?: string) -> nil` | Trap if false |
| `core.type_of` | `(value: any) -> string` | Returns type name: "number", "string", "bool", "nil", "list", "record" |
| `core.capability` | `(name: string) -> bool` | Check if optional capability is available |

### `math` module (10 functions + 2 constants) — [source](pepl-stdlib/src/modules/math.rs)

| Function | Signature | Description |
|----------|-----------|-------------|
| `math.abs` | `(a: number) -> number` | Absolute value |
| `math.min` | `(a: number, b: number) -> number` | Minimum of two numbers |
| `math.max` | `(a: number, b: number) -> number` | Maximum of two numbers |
| `math.floor` | `(a: number) -> number` | Round down |
| `math.ceil` | `(a: number) -> number` | Round up |
| `math.round` | `(a: number) -> number` | Round to nearest (0.5 rounds up) |
| `math.round_to` | `(a: number, decimals: number) -> number` | Round to N decimal places |
| `math.pow` | `(base: number, exp: number) -> number` | Exponentiation |
| `math.clamp` | `(value: number, min: number, max: number) -> number` | Clamp to range |
| `math.sqrt` | `(a: number) -> number` | Square root (traps on negative) |
| `math.PI` | `number` | 3.14159265358979... |
| `math.E` | `number` | 2.71828182845904... |

### `string` module (20 functions) — [source](pepl-stdlib/src/modules/string.rs)

| Function | Signature | Description |
|----------|-----------|-------------|
| `string.length` | `(s: string) -> number` | Character count |
| `string.concat` | `(a: string, b: string) -> string` | Concatenate |
| `string.contains` | `(s: string, sub: string) -> bool` | Check substring |
| `string.slice` | `(s: string, start: number, end: number) -> string` | Substring (start inclusive, end exclusive) |
| `string.trim` | `(s: string) -> string` | Remove leading/trailing whitespace |
| `string.split` | `(s: string, delimiter: string) -> list<string>` | Split by delimiter |
| `string.to_upper` | `(s: string) -> string` | Uppercase |
| `string.to_lower` | `(s: string) -> string` | Lowercase |
| `string.starts_with` | `(s: string, prefix: string) -> bool` | Check prefix |
| `string.ends_with` | `(s: string, suffix: string) -> bool` | Check suffix |
| `string.replace` | `(s: string, from: string, to: string) -> string` | Replace first occurrence |
| `string.replace_all` | `(s: string, from: string, to: string) -> string` | Replace all occurrences |
| `string.pad_start` | `(s: string, length: number, pad: string) -> string` | Pad from start |
| `string.pad_end` | `(s: string, length: number, pad: string) -> string` | Pad from end |
| `string.repeat` | `(s: string, count: number) -> string` | Repeat N times |
| `string.join` | `(items: list<string>, separator: string) -> string` | Join list with separator |
| `string.format` | `(template: string, values: record) -> string` | Template formatting |
| `string.from` | `(value: any) -> string` | Convert any value to string |
| `string.is_empty` | `(s: string) -> bool` | True if string has zero length |
| `string.index_of` | `(s: string, sub: string) -> number` | Index of first occurrence, or -1 |

### `list` module (34 functions) — [source](pepl-stdlib/src/modules/list.rs)

| Function | Signature | Description |
|----------|-----------|-------------|
| `list.length` | `(xs: list<T>) -> number` | Number of elements |
| `list.append` | `(xs: list<T>, item: T) -> list<T>` | New list with item added at end |
| `list.prepend` | `(xs: list<T>, item: T) -> list<T>` | New list with item added at start |
| `list.get` | `(xs: list<T>, index: number) -> T \| nil` | Item at index, nil if out of bounds |
| `list.set` | `(xs: list<T>, index: number, item: T) -> list<T>` | New list with item at index replaced |
| `list.insert` | `(xs: list<T>, index: number, item: T) -> list<T>` | New list with item inserted at index |
| `list.remove` | `(xs: list<T>, index: number) -> list<T>` | New list with item at index removed |
| `list.update` | `(xs: list<T>, index: number, item: T) -> list<T>` | Alias for `list.set` |
| `list.map` | `(xs: list<T>, fn: (T) -> U) -> list<U>` | Transform each element |
| `list.filter` | `(xs: list<T>, fn: (T) -> bool) -> list<T>` | Keep elements where fn returns true |
| `list.sort` | `(xs: list<T>, fn: (T, T) -> number) -> list<T>` | Sort by comparator (stable, deterministic) |
| `list.find` | `(xs: list<T>, fn: (T) -> bool) -> T \| nil` | First match, or nil |
| `list.find_index` | `(xs: list<T>, fn: (T) -> bool) -> number` | Index of first match, or -1 |
| `list.contains` | `(xs: list<T>, item: T) -> bool` | True if item in list |
| `list.reverse` | `(xs: list<T>) -> list<T>` | Reversed list |
| `list.slice` | `(xs: list<T>, start: number, end: number) -> list<T>` | Sublist (start inclusive, end exclusive) |
| `list.flatten` | `(xs: list<list<T>>) -> list<T>` | Flatten one level |
| `list.reduce` | `(xs: list<T>, init: U, fn: (U, T) -> U) -> U` | Left fold with initial value |
| `list.range` | `(start: number, end: number) -> list<number>` | Integers from start to end (exclusive) |
| `list.of` | `(...items: T) -> list<T>` | Create list from arguments (variadic, compiler-special-cased) |
| `list.empty` | `() -> list<T>` | Empty typed list |
| `list.repeat` | `(item: T, count: number) -> list<T>` | List of N copies |
| `list.concat` | `(a: list<T>, b: list<T>) -> list<T>` | Concatenate two lists |
| `list.unique` | `(xs: list<T>) -> list<T>` | Remove duplicates (preserves first occurrence) |
| `list.take` | `(xs: list<T>, count: number) -> list<T>` | First N elements |
| `list.drop` | `(xs: list<T>, count: number) -> list<T>` | All elements after first N |
| `list.any` | `(xs: list<T>, fn: (T) -> bool) -> bool` | True if fn returns true for any element |
| `list.some` | `(xs: list<T>, fn: (T) -> bool) -> bool` | Alias for `list.any` |
| `list.every` | `(xs: list<T>, fn: (T) -> bool) -> bool` | True if fn returns true for every element |
| `list.count` | `(xs: list<T>, fn: (T) -> bool) -> number` | Count matching elements |
| `list.first` | `(xs: list<T>) -> T \| nil` | First element, or nil |
| `list.last` | `(xs: list<T>) -> T \| nil` | Last element, or nil |
| `list.index_of` | `(xs: list<T>, item: T) -> number` | Index of first occurrence, or -1 |
| `list.zip` | `(a: list<T>, b: list<U>) -> list<list<any>>` | Pair elements from two lists |

### `record` module (5 functions) — [source](pepl-stdlib/src/modules/record.rs)

| Function | Signature | Description |
|----------|-----------|-------------|
| `record.get` | `(r: record, key: string) -> any` | Get field value by name |
| `record.set` | `(r: record, key: string, value: any) -> record` | New record with field updated |
| `record.has` | `(r: record, key: string) -> bool` | True if field exists |
| `record.keys` | `(r: record) -> list<string>` | Field names |
| `record.values` | `(r: record) -> list<any>` | Field values |

### `time` module (5 functions) — [source](pepl-stdlib/src/modules/time.rs)

| Function | Signature | Description |
|----------|-----------|-------------|
| `time.now` | `() -> number` | Current timestamp in ms (host-provided, deterministic) |
| `time.format` | `(timestamp: number, pattern: string) -> string` | Format: "YYYY-MM-DD", "HH:mm", etc. |
| `time.diff` | `(a: number, b: number) -> number` | Difference in ms (a - b) |
| `time.day_of_week` | `(timestamp: number) -> number` | 0=Sunday through 6=Saturday |
| `time.start_of_day` | `(timestamp: number) -> number` | Midnight timestamp for the given day |

### `convert` module (5 functions) — [source](pepl-stdlib/src/modules/convert.rs)

| Function | Signature | Description |
|----------|-----------|-------------|
| `convert.to_string` | `(value: any) -> string` | Always succeeds |
| `convert.to_number` | `(value: any) -> Result<number, ConvertError>` | Parse strings, bool→0/1 |
| `convert.parse_int` | `(s: string) -> Result<number, ConvertError>` | Parse integer |
| `convert.parse_float` | `(s: string) -> Result<number, ConvertError>` | Parse float |
| `convert.to_bool` | `(value: any) -> bool` | Truthy: 0/nil/"" → false |

### `json` module (2 functions) — [source](pepl-stdlib/src/modules/json.rs)

| Function | Signature | Description |
|----------|-----------|-------------|
| `json.parse` | `(s: string) -> Result<any, JsonError>` | Parse JSON to PEPL value (depth limit: 32) |
| `json.stringify` | `(value: any) -> string` | Serialize PEPL value to JSON |

### `timer` module (4 functions, capability) — [source](pepl-stdlib/src/modules/timer.rs)

| Function | Signature | Description |
|----------|-----------|-------------|
| `timer.start` | `(action_name: string, interval_ms: number) -> string` | Start recurring timer, returns ID |
| `timer.start_once` | `(action_name: string, delay_ms: number) -> string` | One-shot timer, returns ID |
| `timer.stop` | `(timer_id: string) -> nil` | Stop timer (no-op if invalid) |
| `timer.stop_all` | `() -> nil` | Stop all active timers |

### `http` module (5 functions, capability) — [source](pepl-stdlib/src/modules/http.rs)

| Function | Signature | Description |
|----------|-----------|-------------|
| `http.get` | `(url: string) -> Result<string, HttpError>` | GET request |
| `http.post` | `(url: string, body: string) -> Result<string, HttpError>` | POST request |
| `http.put` | `(url: string, body: string) -> Result<string, HttpError>` | PUT request |
| `http.patch` | `(url: string, body: string) -> Result<string, HttpError>` | PATCH request |
| `http.delete` | `(url: string) -> Result<string, HttpError>` | DELETE request |

### `storage` module (4 functions, capability) — [source](pepl-stdlib/src/modules/storage.rs)

| Function | Signature | Description |
|----------|-----------|-------------|
| `storage.get` | `(key: string) -> string \| nil` | Get value by key, nil if not found |
| `storage.set` | `(key: string, value: string) -> nil` | Store key-value pair |
| `storage.delete` | `(key: string) -> nil` | Remove key-value pair |
| `storage.keys` | `() -> list<string>` | List all stored keys |

### `location` module (1 function, capability) — [source](pepl-stdlib/src/modules/location.rs)

| Function | Signature | Description |
|----------|-----------|-------------|
| `location.current` | `() -> {lat: number, lon: number}` | Current geolocation |

### `notifications` module (1 function, capability) — [source](pepl-stdlib/src/modules/notifications.rs)

| Function | Signature | Description |
|----------|-----------|-------------|
| `notifications.send` | `(title: string, body: string) -> nil` | Send notification |

### Error Types

All capability/conversion error types share the shape `{ code: string, message: string }`:

| Error Type | Used By | Error Codes |
|---|---|---|
| `HttpError` | `http.*` | `"network_error"`, `"timeout"`, `"blocked_by_host"`, `"offline"`, `"cors_error"` |
| `JsonError` | `json.parse` | `"invalid_json"`, `"max_depth_exceeded"` |
| `StorageError` | `storage.*` | `"not_found"`, `"quota_exceeded"`, `"write_failed"`, `"permission_denied"` |
| `LocationError` | `location.*` | `"permission_denied"`, `"unavailable"`, `"timeout"` |
| `NotificationError` | `notifications.*` | `"permission_denied"`, `"blocked"` |
| `ConvertError` | `convert.to_number`, `convert.parse_*` | `"invalid_input"`, `"out_of_range"` |

---

## §5 CompileResult

> Source: [pepl/crates/pepl-compiler/src/lib.rs](pepl/crates/pepl-compiler/src/lib.rs) · [pepl/crates/pepl-compiler/src/reference.rs](pepl/crates/pepl-compiler/src/reference.rs)

The main compiler entry point is `compile_to_result(source, name) -> CompileResult`. The result is JSON-serializable for host consumption.

```
CompileResult {
    success: bool                           // Whether compilation succeeded
    wasm: Option<Vec<u8>>                   // .wasm bytes (base64 in JSON)
    errors: CompileErrors                   // Structured compile errors
    ast: Option<Program>                    // Full AST (JSON-serializable)
    source_hash: String                     // SHA-256 of source (hex)
    wasm_hash: Option<String>               // SHA-256 of WASM bytes (hex)
    state_fields: Vec<FieldInfo>            // [{ name, type }]
    actions: Vec<ActionInfo>                // [{ name, params: [{ name, type }] }]
    views: Vec<String>                      // View function names
    capabilities: Vec<String>               // Declared capabilities
    credentials: Vec<FieldInfo>             // [{ name, type }]
    language_version: String                // "0.1.0"
    compiler_version: String                // From Cargo.toml
    warnings: Vec<PeplError>                // Warnings (separate from errors)
    source_map: Option<SourceMap>           // WASM func index → PEPL source location
}
```

### Entry Points

| Function | Purpose |
|----------|---------|
| `compile_to_result(source, name) -> CompileResult` | Full pipeline with enriched metadata (main host entry) |
| `compile(source, name) -> Result<Vec<u8>, CompileErrors>` | Full pipeline, returns just WASM bytes |
| `type_check(source, name) -> CompileErrors` | Parse + type-check only (no codegen) |

### WASM Exports ([pepl-wasm](pepl/crates/pepl-wasm/src/lib.rs) crate)

| Export | Returns |
|--------|---------|
| `compile(source, filename) -> String` | JSON-serialized `CompileResult` |
| `type_check(source, filename) -> String` | JSON-serialized errors |
| `version() -> String` | Compiler version |
| `get_reference() -> String` | Compressed PEPL reference (~2K tokens, for LLM injection) |
| `get_stdlib_table() -> String` | JSON stdlib table (all functions/signatures/descriptions) |

---

## §6 WASM Import/Export Contract

> Source: [pepl/crates/pepl-codegen/src/lib.rs](pepl/crates/pepl-codegen/src/lib.rs) · [docs/host-integration.md](docs/host-integration.md)

Any host running compiled PEPL WASM must implement this interface.

### Imports (host provides)

| Import | Signature | Description |
|--------|-----------|-------------|
| `env.host_call` | `(cap_id: i32, fn_id: i32, payload_ptr: i32, payload_len: i32) -> i32` | All capability calls route through this. Returns response pointer. |
| `env.get_timestamp` | `() -> i64` | Deterministic event timestamp (ms). Host-controlled, never system clock. |
| `env.log` | `(ptr: i32, len: i32) -> void` | Debug logging. No-op in production. |
| `env.trap` | `(code: i32) -> void` | Runtime trap. Host terminates execution. |

### Exports (PEPL WASM provides)

| Export | Signature | Description |
|--------|-----------|-------------|
| `init` | `() -> void` | Initialize state to defaults |
| `dispatch_action` | `(action_id: i32, payload_ptr: i32, payload_len: i32) -> void` | Trigger named action |
| `render` | `() -> i32` | Return serialized view tree JSON pointer |
| `get_state` | `() -> i32` | Return serialized state pointer |
| `update` | `(dt: f64) -> void` | Game loop tick (if space declares `update`) |
| `handle_event` | `(event_ptr: i32, event_len: i32) -> void` | Game loop events (if space declares `handleEvent`) |
| `alloc` | `(size: i32) -> i32` | Allocate bytes in WASM linear memory |
| `dealloc` | `(ptr: i32, size: i32) -> void` | Free allocated memory |

### Memory Protocol

- **Host → WASM:** `alloc(n)` → get ptr → write payload at ptr → call `dispatch_action`/`handle_event`
- **WASM → Host:** `render()`/`get_state()` returns ptr → host reads UTF-8 JSON from WASM memory
- All payloads are **JSON-encoded UTF-8 strings**

### Capability IDs (for `env.host_call`)

| cap_id | Capability | fn_id values |
|--------|------------|--------------|
| 1 | http | 1=get, 2=post, 3=put, 4=patch, 5=delete |
| 2 | storage | 1=get, 2=set, 3=delete, 4=keys |
| 3 | location | 1=current |
| 4 | notifications | 1=send |
| 5 | credential | 1=get (internal) |

---

## §7 UI Components (Phase 0)

> Source: [pepl-ui/src/components/](pepl-ui/src/components/) · [docs/ui-components.md](docs/ui-components.md)

10 components available. Record-style syntax: `ComponentName { prop: value, ... }`.

| Component | Required Props | Optional Props | Children |
|-----------|---------------|----------------|----------|
| `Text` | `value: string` | `size`, `weight`, `align`, `color`, `overflow`, `max_lines` | No |
| `Button` | `label: string`, `on_tap: action` | `variant`, `icon`, `disabled`, `loading` | No |
| `TextInput` | `value: string`, `on_change: action` | `placeholder`, `label`, `keyboard`, `max_length`, `multiline` | No |
| `Column` | — | `padding`, `spacing`, `alignment`, `background`, `scroll` | Yes |
| `Row` | — | `padding`, `spacing`, `alignment`, `background` | Yes |
| `Scroll` | — | `direction`, `padding` | Yes |
| `ScrollList` | `items: list`, `render: fn` | `key`, `on_reorder`, `dividers` | No |
| `ProgressBar` | `value: number` (0.0–1.0) | `color`, `track_color`, `height` | No |
| `Modal` | `visible: bool`, `on_dismiss: action` | `title` | Yes |
| `Toast` | `message: string` | `duration`, `type` | No |

---

## §8 Error Codes

> Source: [pepl/crates/pepl-compiler/src/checker.rs](pepl/crates/pepl-compiler/src/checker.rs) · [docs/compiler.md](docs/compiler.md)

23 error codes across 6 categories:

| Range | Category | Codes |
|-------|----------|-------|
| E100–E199 | Syntax | E100: Unexpected token, E101: Unclosed brace, E102: Invalid keyword |
| E200–E299 | Type | E200: Unknown type, E201: Type mismatch, E202: Wrong argument count, E210: Non-exhaustive match |
| E300–E399 | Invariant | E300: Invariant unreachable, E301: Invariant references unknown field |
| E400–E499 | Capability | E400: Undeclared capability used, E401: Capability not available in sandbox, E402: Unknown component |
| E500–E599 | Scope | E500: Variable already declared, E501: State mutated outside action, E502: Recursion not allowed |
| E600–E699 | Structure | E600: Block ordering violated, E601: Derived field modified via set, E602: Expression-body lambda, E603: Block comment used, E604: Undeclared credential used, E605: Credential modified via set, E606: Empty state block, E607: Structural limit exceeded |

All errors include source line/column, message, and structured `suggestion` field.

---

## §9 Guarantees

> Source: [docs/execution-semantics.md](docs/execution-semantics.md)

| Guarantee | Description |
|-----------|-------------|
| **Determinism** | Same source + same inputs = identical outputs, byte-for-byte. Verified by 100-iteration compilation + execution determinism tests. |
| **Sandboxing** | No ambient state, no filesystem, no network unless declared as capability. All I/O goes through `env.host_call`. |
| **Immutability** | All list/record operations return new values. No mutation anywhere. |
| **State isolation** | State can only be modified via `set` inside `action` bodies. Views are pure (no side effects). |
| **Invariant enforcement** | Invariant expressions checked after every action. Violation → automatic rollback. |
| **Stdlib stability** | Functions never change behavior once released. New functions may be added. Existing functions are never removed or modified. |
| **NaN prevention** | Division by zero, `sqrt` of negative → trap. No NaN values can exist at runtime. |
| **Gas metering** | Loop iterations and function calls are budgeted. Infinite loops → deterministic trap. |

---

## §10 Canonical Examples

> Source: [docs/llm-generation-contract.md](docs/llm-generation-contract.md)

7 examples ship with the compiler and are used for integration testing:

1. **Counter** — minimal space: state, action, view, Button, Text
2. **TodoList** — list operations, for loops, TextInput, ScrollList, lambdas
3. **UnitConverter** — match expressions, derived state, convert module
4. **WeatherDashboard** — http capability, credentials, Result handling
5. **PomodoroTimer** — timer module, state machines, conditional UI
6. **HabitTracker** — complex state, list.map/filter/reduce, ProgressBar
7. **QuizApp** — sum types, match exhaustiveness, dynamic UI

---

## §11 Compilation Pipeline

> Source: [pepl/crates/pepl-lexer/src/lexer.rs](pepl/crates/pepl-lexer/src/lexer.rs) · [pepl/crates/pepl-compiler/src/checker.rs](pepl/crates/pepl-compiler/src/checker.rs) · [pepl/crates/pepl-codegen/src/lib.rs](pepl/crates/pepl-codegen/src/lib.rs) · [docs/compiler.md](docs/compiler.md)

```
Source → Lexer → Parser → Type Checker → Invariant Checker → WASM Codegen → .wasm
         tokens    AST     typed AST      verified AST        binary
```

Each stage produces structured errors. The pipeline short-circuits on error — no codegen if type errors exist.

**Evaluator** ([pepl-eval](pepl/crates/pepl-eval/src/evaluator.rs)): tree-walking interpreter that executes from the typed AST. Used as the golden reference for WASM correctness (eval↔codegen parity tests).

---

## §12 Deferred to Phase 1+

> Source: [phases.md](phases.md) · [docs/future.md](docs/future.md)

| Item | Tracking |
|------|----------|
| `any` type runtime checks on state assignment | F17 |
| Capability call suspension/resume in WASM | F20 |
| `with_responses` mock capability dispatch in WASM | Deferred — no WASM programs use capabilities yet |
| Keyword count reconciliation in LLM contract | F15 |
| `color` module (programmatic color functions) | Phase 2+ |
| User-defined generics | Out of scope |

---

*Last verified: 2026-02-13 against [`pepl/`](pepl/), [`pepl-stdlib/`](pepl-stdlib/), [`pepl-ui/`](pepl-ui/) source code.*
