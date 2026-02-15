# Complete Standard Library Reference (Phase 0)

Every function available in Phase 0. The LLM prompt injects this entire reference. If a function is not listed here, it does not exist.

## `core` module

| Function | Signature | Description |
|---|---|---|
| `core.log` | `(value: any) -> nil` | Debug logging (no-op in production, writes to console in dev) |
| `core.assert` | `(condition: bool, message?: string) -> nil` | Panics (WASM trap) if condition is false. Optional message provides context in test output |
| `core.type_of` | `(value: any) -> string` | Returns type name: "number", "string", "bool", "nil", "list", "record" |
| `core.capability` | `(name: string) -> bool` | Returns whether a declared optional capability is available at runtime |

## `math` module

Arithmetic uses operators (`+`, `-`, `*`, `/`, `%`) — no stdlib aliases. The math module provides functions that go beyond basic arithmetic:

| Function | Signature | Description |
|---|---|---|
| `math.abs` | `(a: number) -> number` | Absolute value |
| `math.min` | `(a: number, b: number) -> number` | Smaller of two values |
| `math.max` | `(a: number, b: number) -> number` | Larger of two values |
| `math.floor` | `(a: number) -> number` | Round down to nearest integer |
| `math.ceil` | `(a: number) -> number` | Round up to nearest integer |
| `math.round` | `(a: number) -> number` | Round to nearest integer (0.5 rounds up) |
| `math.round_to` | `(a: number, decimals: number) -> number` | Round to N decimal places |
| `math.pow` | `(base: number, exp: number) -> number` | Exponentiation |
| `math.clamp` | `(value: number, min: number, max: number) -> number` | Clamp value to [min, max] range |
| `math.sqrt` | `(a: number) -> number` | Square root |
| `math.PI` | `number` (constant) | Pi (3.14159265358979...) |
| `math.E` | `number` (constant) | Euler's number (2.71828182845904...) |

## `string` module

| Function | Signature | Description |
|---|---|---|
| `string.length` | `(s: string) -> number` | Number of characters |
| `string.concat` | `(a: string, b: string) -> string` | Concatenate two strings |
| `string.contains` | `(haystack: string, needle: string) -> bool` | True if needle found in haystack |
| `string.slice` | `(s: string, start: number, end: number) -> string` | Substring from start (inclusive) to end (exclusive) |
| `string.trim` | `(s: string) -> string` | Remove leading/trailing whitespace |
| `string.split` | `(s: string, delimiter: string) -> list<string>` | Split string by delimiter |
| `string.to_upper` | `(s: string) -> string` | Convert to uppercase |
| `string.to_lower` | `(s: string) -> string` | Convert to lowercase |
| `string.starts_with` | `(s: string, prefix: string) -> bool` | True if s starts with prefix |
| `string.ends_with` | `(s: string, suffix: string) -> bool` | True if s ends with suffix |
| `string.replace` | `(s: string, old: string, new: string) -> string` | Replace first occurrence of old with new |
| `string.replace_all` | `(s: string, old: string, new: string) -> string` | Replace all occurrences of old with new |
| `string.pad_start` | `(s: string, length: number, pad: string) -> string` | Pad string on the left to reach target length |
| `string.pad_end` | `(s: string, length: number, pad: string) -> string` | Pad string on the right to reach target length |
| `string.repeat` | `(s: string, count: number) -> string` | Repeat string `count` times |
| `string.join` | `(items: list<string>, separator: string) -> string` | Join list of strings with separator |
| `string.format` | `(template: string, values: record) -> string` | Template string with `{key}` placeholders replaced by record values |
| `string.from` | `(value: any) -> string` | Convert any value to its string representation |
| `string.is_empty` | `(s: string) -> bool` | True if string has zero length |
| `string.index_of` | `(s: string, sub: string) -> number` | Index of first occurrence of sub, or -1 if not found |

## `list` module

| Function | Signature | Description |
|---|---|---|
| `list.length` | `(xs: list<T>) -> number` | Number of elements |
| `list.append` | `(xs: list<T>, item: T) -> list<T>` | Return new list with item added at end |
| `list.prepend` | `(xs: list<T>, item: T) -> list<T>` | Return new list with item added at start |
| `list.get` | `(xs: list<T>, index: number) -> T \| nil` | Get item at index. Returns `nil` if index is out of bounds — use `??` for defaults |
| `list.set` | `(xs: list<T>, index: number, item: T) -> list<T>` | Return new list with item at index replaced |
| `list.insert` | `(xs: list<T>, index: number, item: T) -> list<T>` | Return new list with item inserted at index |
| `list.remove` | `(xs: list<T>, index: number) -> list<T>` | Return new list with item at index removed |
| `list.update` | `(xs: list<T>, index: number, item: T) -> list<T>` | Alias for `list.set` — return new list with item at index replaced |
| `list.map` | `(xs: list<T>, fn: (T) -> U) -> list<U>` | Transform each element |
| `list.filter` | `(xs: list<T>, fn: (T) -> bool) -> list<T>` | Keep elements where fn returns true |
| `list.sort` | `(xs: list<T>, fn: (T, T) -> number) -> list<T>` | Sort by comparator (negative=a first, positive=b first) |
| `list.find` | `(xs: list<T>, fn: (T) -> bool) -> T \| nil` | First element where fn returns true, or nil |
| `list.contains` | `(xs: list<T>, item: T) -> bool` | True if item is in list (equality check) |
| `list.reverse` | `(xs: list<T>) -> list<T>` | Return reversed list |
| `list.slice` | `(xs: list<T>, start: number, end: number) -> list<T>` | Sublist from start (inclusive) to end (exclusive) |
| `list.flatten` | `(xs: list<list<T>>) -> list<T>` | Flatten one level of nesting |
| `list.reduce` | `(xs: list<T>, init: U, fn: (U, T) -> U) -> U` | Left fold with initial value |
| `list.range` | `(start: number, end: number) -> list<number>` | Generate list of integers from start (inclusive) to end (exclusive) |

| `list.of` | `(...items: T) -> list<T>` | Create list from arguments: `list.of(1, 2, 3)`. Compiler-special-cased — PEPL has no general variadic function syntax |
| `list.empty` | `() -> list<T>` | Create empty typed list |
| `list.repeat` | `(item: T, count: number) -> list<T>` | Create list of `count` copies of item |
| `list.concat` | `(a: list<T>, b: list<T>) -> list<T>` | Concatenate two lists |
| `list.unique` | `(xs: list<T>) -> list<T>` | Remove duplicate elements (preserves first occurrence) |
| `list.take` | `(xs: list<T>, count: number) -> list<T>` | Return first `count` elements |
| `list.drop` | `(xs: list<T>, count: number) -> list<T>` | Return all elements after the first `count` |
| `list.any` | `(xs: list<T>, fn: (T) -> bool) -> bool` | True if fn returns true for any element |
| `list.some` | `(xs: list<T>, fn: (T) -> bool) -> bool` | Alias for `list.any` |
| `list.every` | `(xs: list<T>, fn: (T) -> bool) -> bool` | True if fn returns true for every element |
| `list.count` | `(xs: list<T>, fn: (T) -> bool) -> number` | Count elements where fn returns true |
| `list.first` | `(xs: list<T>) -> T \| nil` | First element, or nil if empty |
| `list.last` | `(xs: list<T>) -> T \| nil` | Last element, or nil if empty |
| `list.index_of` | `(xs: list<T>, item: T) -> number` | Index of first occurrence of item, or -1 if not found |
| `list.find_index` | `(xs: list<T>, fn: (T) -> bool) -> number` | Index of first element where fn returns true, or -1 |
| `list.zip` | `(a: list<T>, b: list<U>) -> list<list<any>>` | Pair elements from two lists |

## `record` module

| Function | Signature | Description |
|---|---|---|
| `record.get` | `(r: record, key: string) -> any` | Get field value by name (compile-time checked for known records) |
| `record.set` | `(r: record, key: string, value: any) -> record` | Return new record with field updated |
| `record.has` | `(r: record, key: string) -> bool` | True if record has the named field |
| `record.keys` | `(r: record) -> list<string>` | List of field names |
| `record.values` | `(r: record) -> list<any>` | List of field values |

## `time` module

| Function | Signature | Description |
|---|---|---|
| `time.now` | `() -> number` | Current timestamp in milliseconds (deterministic — provided by host, not system clock) |
| `time.format` | `(timestamp: number, pattern: string) -> string` | Format timestamp. Patterns: "YYYY-MM-DD", "HH:mm", "YYYY-MM-DD HH:mm:ss" |
| `time.diff` | `(a: number, b: number) -> number` | Difference in milliseconds (a - b) |
| `time.day_of_week` | `(timestamp: number) -> number` | 0=Sunday through 6=Saturday |
| `time.start_of_day` | `(timestamp: number) -> number` | Timestamp of midnight (00:00) for the given day |

## `convert` module

Consolidated type conversion functions. Fallible conversions return `Result`:

| Function | Signature | Description |
|---|---|---|
| `convert.to_string` | `(value: any) -> string` | Convert any value to string representation. Always succeeds. |
| `convert.to_number` | `(value: any) -> Result<number, ConvertError>` | Convert to number (parses strings, bool→0/1). Returns `Err` if value cannot be converted. |
| `convert.parse_int` | `(s: string) -> Result<number, ConvertError>` | Parse string to integer. Returns `Err` on invalid input. |
| `convert.parse_float` | `(s: string) -> Result<number, ConvertError>` | Parse string to float. Returns `Err` on invalid input. |
| `convert.to_bool` | `(value: any) -> bool` | Truthy conversion (0/nil/"" → false, everything else → true). Always succeeds. |

## `json` module

| Function | Signature | Description |
|---|---|---|
| `json.parse` | `(s: string) -> Result<any, JsonError>` | Parse JSON string to PEPL value. Objects become records, arrays become lists. |
| `json.stringify` | `(value: any) -> string` | Serialize PEPL value to JSON string. Records become objects, lists become arrays. |

```pepl
// JsonError type
// { code: string, message: string }
// Codes: "invalid_json", "max_depth_exceeded" (depth limit: 32)
```

> **Error type definitions used in capability modules:**
>
> All capability error types share the same shape: `{ code: string, message: string }`. Error codes are string constants specific to each module:
>
> | Error Type | Used By | Error Codes |
> |---|---|---|
> | `HttpError` | `http.*` | `"network_error"`, `"timeout"`, `"blocked_by_host"`, `"offline"`, `"cors_error"` |
> | `JsonError` | `json.parse` | `"invalid_json"`, `"max_depth_exceeded"` |
> | `StorageError` | `storage.*` | `"not_found"`, `"quota_exceeded"`, `"write_failed"`, `"permission_denied"` |
> | `LocationError` | `location.*` | `"permission_denied"`, `"unavailable"`, `"timeout"` |
> | `NotificationError` | `notifications.*` | `"permission_denied"`, `"blocked"` |
> | `TimeError` | `time.from_iso` *(Phase 1+)* | `"invalid_format"`, `"out_of_range"` |
> | `ConvertError` | `convert.to_number`, `convert.parse_int`, `convert.parse_float` | `"invalid_input"`, `"out_of_range"` |

## `timer` module

Timers allow spaces to schedule recurring or one-shot action dispatches. The host manages the actual scheduling — PEPL code declares intent, and the host fires the action at the specified interval.

| Function | Signature | Description |
|---|---|---|
| `timer.start` | `(action_name: string, interval_ms: number) -> string` | Start a recurring timer that dispatches the named action every `interval_ms` milliseconds. Returns a timer ID. |
| `timer.start_once` | `(action_name: string, delay_ms: number) -> string` | Schedule a one-shot timer that dispatches the named action after `delay_ms` milliseconds. Returns a timer ID. |
| `timer.stop` | `(timer_id: string) -> nil` | Stop a running timer. No-op if timer already stopped or ID is invalid. |
| `timer.stop_all` | `() -> nil` | Stop all active timers for this space. |

## `http` module (capability: `http`)

All HTTP functions yield to the host and return `Result<string, HttpError>`.

| Function | Signature | Description |
|---|---|---|
| `http.get` | `(url: string) -> Result<string, HttpError>` | Perform a GET request |
| `http.post` | `(url: string, body: string) -> Result<string, HttpError>` | Perform a POST request |
| `http.put` | `(url: string, body: string) -> Result<string, HttpError>` | Perform a PUT request |
| `http.patch` | `(url: string, body: string) -> Result<string, HttpError>` | Perform a PATCH request |
| `http.delete` | `(url: string) -> Result<string, HttpError>` | Perform a DELETE request |

## `storage` module (capability: `storage`)

Key-value storage, host-mediated.

| Function | Signature | Description |
|---|---|---|
| `storage.get` | `(key: string) -> string \| nil` | Get value by key, or nil if not found |
| `storage.set` | `(key: string, value: string) -> nil` | Store a key-value pair |
| `storage.delete` | `(key: string) -> nil` | Remove a key-value pair |
| `storage.keys` | `() -> list<string>` | List all stored keys |

## `location` module (capability: `location`)

Geolocation, permission-gated.

| Function | Signature | Description |
|---|---|---|
| `location.current` | `() -> {lat: number, lon: number}` | Get current geolocation coordinates |

## `notifications` module (capability: `notifications`)

Permission-gated push notifications.

| Function | Signature | Description |
|---|---|---|
| `notifications.send` | `(title: string, body: string) -> nil` | Send a notification to the user |

```pepl
// Timer semantics:
// - timer.start dispatches the named action repeatedly at the given interval
// - The host is responsible for scheduling — PEPL code does not block or sleep
// - Timer actions execute like any other action (sequential, with invariant checks)
// - Timers survive across actions but NOT across space restarts (ephemeral)
// - Maximum active timers per space: host-configurable (SHOULD default to 8)
// - Minimum interval: host-configurable (SHOULD default to 100ms)
//
// Usage in Pomodoro timer:
//   action start_work() {
//     set mode = "work"
//     set seconds_left = 1500
//     set timer_id = timer.start("tick", 1000)
//   }
//   action tick() {
//     if seconds_left > 0 {
//       set seconds_left = seconds_left - 1
//     } else {
//       timer.stop(timer_id)
//       set mode = "done_work"
//     }
//   }
```
