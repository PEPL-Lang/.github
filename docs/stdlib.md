# Standard Library

> **Authority note:** This section is the complete standard library reference across all phases. For Phase 0, the Phase 0 Stdlib section (see `phase-0-stdlib-reference.md`) is authoritative — it defines the exact subset available and any Phase-0-specific behavior.

All stdlib modules are available without explicit imports. Functions are deterministic: same inputs → same output, always.

## `core` — Fundamental Operations

```pepl
// Boolean operators are keywords: not, and, or
// No stdlib function equivalents — use the operators directly

// Type checking
core.type_of(value: any) -> string   // "number", "string", "bool", "list", "record"

// Nil checking uses == operator: value == nil
// No is_nil() function — use direct comparison

// Comparison and equality use operators: ==, !=, <, <=, >, >=
// No stdlib duplicates (no core.eq, core.lt, etc.)

// Debug
core.log(value: any) -> nil          // Debug output, no-op in production

// Assertion (also available as assert statement)
core.assert(expr: bool, message?: string) -> nil
// Note: `?` in stdlib signatures means the parameter is optional.
// Callers may omit it: core.assert(x > 0) or core.assert(x > 0, "must be positive")
```

## `math` — Numeric Operations

```pepl
// Arithmetic uses operators: +, -, *, /, %
// No stdlib duplicates (no math.add, math.sub, etc.)
math.pow(a: number, exp: number) -> number

// Rounding
math.floor(n: number) -> number
math.ceil(n: number) -> number
math.round(n: number) -> number
math.round_to(n: number, decimals: number) -> number

// Bounds
math.min(a: number, b: number) -> number
math.max(a: number, b: number) -> number
math.clamp(value: number, min: number, max: number) -> number
math.abs(n: number) -> number
math.sqrt(n: number) -> number

// Constants
math.PI, math.E
math.MAX   // IEEE 754 f64 maximum: 1.7976931348623157e+308
math.MIN   // IEEE 754 f64 smallest positive: 5e-324

// Trigonometry
math.sin(rad: number) -> number
math.cos(rad: number) -> number
math.tan(rad: number) -> number
math.atan2(y: number, x: number) -> number
math.to_radians(deg: number) -> number
math.to_degrees(rad: number) -> number

// Random (deterministic PRNG seeded per-space, per-event)
math.random() -> number                          // [0.0, 1.0)
math.random_int(min: number, max: number) -> number  // [min, max] inclusive
```

## `string` — Text Operations

```pepl
// Construction
string.from(value: any) -> string
string.repeat(s, count) -> string
string.join(items: list<string>, separator) -> string

// Query
string.length(s), string.is_empty(s), string.contains(s, sub)
string.starts_with(s, prefix), string.ends_with(s, suffix)
string.index_of(s, sub) -> number   // -1 if not found

// Transformation
string.to_upper(s), string.to_lower(s), string.trim(s)
string.trim_start(s), string.trim_end(s)
string.replace(s, from, to), string.replace_all(s, from, to)

// Extraction
string.slice(s, start, end), string.char_at(s, index)
string.split(s, separator) -> list<string>

// Formatting
string.pad_start(s, length, pad), string.pad_end(s, length, pad)
string.format(template, values: record) -> string
  // string.format("Hello {name}, you have {count} items", {name: "Alex", count: 5})
  // Note: ${expr} string interpolation replaces most string.format use cases
```

## `list` — Ordered Collections

```pepl
// Construction
list.empty<T>(), list.of<T>(...items), list.range(start, end), list.repeat<T>(item, count)

// Query
list.length(l), list.is_empty(l), list.contains(l, item)
list.index_of(l, item), list.first(l), list.last(l), list.get(l, index)

// Transformation (all return NEW list, never mutate)
list.append(l, item), list.prepend(l, item), list.insert(l, index, item)
list.remove(l, index), list.remove_item(l, item), list.set(l, index, item)
list.concat(a, b), list.reverse(l), list.sort(l, comparator), list.sort_by(l, key)
list.unique(l), list.flatten(l), list.slice(l, start, end)
list.take(l, count), list.drop(l, count)

// Higher-order
list.map(l, f), list.filter(l, f), list.reduce(l, init, f)
list.find(l, f), list.any(l, f), list.every(l, f)
list.count(l, f), list.group_by(l, key: (T) -> string)

// Aggregation
list.sum(l), list.product(l), list.min(l), list.max(l), list.average(l)
```

## `record` — Key-Value Records

```pepl
// Construction
record.empty(), record.from_entries(entries: list<{key: string, value: any}>)

// Query
record.get(r, key), record.has(r, key), record.keys(r), record.values(r)
record.entries(r), record.size(r)

// Transformation (all return NEW record)
record.set(r, key, value), record.remove(r, key)
record.merge(a, b)  // b wins on conflict
record.map_values(r, f), record.filter(r, f: (string, any) -> bool)
```

## `time` — Temporal Operations

All times are event-sourced (deterministic). `time.now()` returns the current event's timestamp as a Unix millisecond number. All timestamps and durations are plain `number` values (milliseconds).

```pepl
// Current (deterministic — tied to event)
time.now() -> number                                  // Unix milliseconds

// Construction
time.from_iso(s: string) -> Result<number, TimeError> // ISO 8601 → ms
time.from_parts(year: number, month: number, day: number,
                hour: number, minute: number, second: number) -> number

// Extraction (t is Unix milliseconds)
time.year(t: number) -> number                        // 1-based
time.month(t: number) -> number                       // 1-based
time.day(t: number) -> number                         // 1-based
time.hour(t: number) -> number                        // 0-based
time.minute(t: number) -> number                      // 0-based
time.second(t: number) -> number                      // 0-based
time.day_of_week(t: number) -> number                 // 0=Sunday
time.day_of_year(t: number) -> number                 // 1-366

// Arithmetic (all values in milliseconds)
time.add_seconds(t: number, s: number) -> number
time.add_minutes(t: number, m: number) -> number
time.add_hours(t: number, h: number) -> number
time.add_days(t: number, d: number) -> number
time.diff_seconds(a: number, b: number) -> number
time.diff_minutes(a: number, b: number) -> number
time.diff_hours(a: number, b: number) -> number
time.diff_days(a: number, b: number) -> number
time.start_of_day(t: number) -> number

// Comparison
time.is_before(a: number, b: number) -> bool
time.is_after(a: number, b: number) -> bool
time.is_same_day(a: number, b: number) -> bool

// Formatting
time.format(t: number, pattern: string) -> string    // "YYYY-MM-DD", "HH:mm"
time.relative(t: number) -> string                    // "2 hours ago", "in 3 days" (Phase 1+ — locale-dependent)
time.duration_format(ms: number) -> string            // "2h 30m 15s"
```

## `convert` — Type Conversions & Unit Conversions

```pepl
// Type conversions (all return Result for fallible conversions)
convert.to_string(value: any) -> string                // always succeeds
convert.to_number(value: any) -> Result<number, ConvertError>
convert.to_bool(value: any) -> bool
convert.parse_int(s: string) -> Result<number, ConvertError>
convert.parse_float(s: string) -> Result<number, ConvertError>

// Unit conversions
// Temperature
convert.celsius_to_fahrenheit(c: number) -> number
convert.fahrenheit_to_celsius(f: number) -> number

// Distance
convert.km_to_miles(km: number) -> number
convert.miles_to_km(mi: number) -> number
convert.m_to_feet(m: number) -> number
convert.feet_to_m(ft: number) -> number

// Weight
convert.kg_to_lbs(kg: number) -> number
convert.lbs_to_kg(lbs: number) -> number

// Volume
convert.liters_to_gallons(l: number) -> number
convert.gallons_to_liters(g: number) -> number
convert.ml_to_oz(ml: number) -> number
convert.oz_to_ml(oz: number) -> number

// Data
convert.bytes_to_kb(b: number) -> number
convert.bytes_to_mb(b: number) -> number
convert.bytes_to_gb(b: number) -> number
```

## `json` — Serialization

```pepl
// Parsing
json.parse(s: string) -> Result<any, JsonError>       // JSON string → PEPL value
json.is_valid(s: string) -> bool                       // check without parsing

// Serialization
json.stringify(value: any) -> string                   // PEPL value → compact JSON
json.stringify_pretty(value: any) -> string            // PEPL value → indented JSON

// Type mapping:
//   JSON object  → record     JSON array  → list
//   JSON number  → number     JSON string → string
//   JSON boolean → bool       JSON null   → nil
//
// Max depth: 32 levels (parse traps beyond this)
```

## `timer` — Scheduled Actions

Timers allow spaces to schedule recurring or one-shot action dispatches. The host manages scheduling — PEPL declares intent.

```pepl
// Recurring timer
timer.start(action_name: string, interval_ms: number) -> string
  // Returns timer ID. Dispatches action_name every interval_ms milliseconds.

// One-shot timer
timer.start_once(action_name: string, delay_ms: number) -> string
  // Returns timer ID. Dispatches action_name once after delay_ms milliseconds.

// Stop a timer
timer.stop(timer_id: string) -> nil
  // No-op if timer already stopped or ID is invalid.

// Stop all timers
timer.stop_all() -> nil

// Timer rules:
// - Timer-dispatched actions execute like any other action (sequential, invariant-checked)
// - Timers are ephemeral — they do NOT survive space restarts
// - Maximum active timers per space: host-configurable (SHOULD default to 8)
// - Minimum interval: host-configurable (SHOULD default to 100ms)
// - Timer IDs are opaque strings (host-generated)
```

## `color` — Color Operations

```pepl
// Construction
color.rgb(r: number, g: number, b: number) -> color              // 0-255 per channel
color.rgba(r: number, g: number, b: number, a: number) -> color  // a: 0.0-1.0
color.parse(s: string) -> color                                   // "#FF5733", "rgb(255,87,51)", "red"
color.hex(s: string) -> color                                     // "#FF5733" or "FF5733"

// Transformation
color.lighten(c: color, amount: number) -> color     // amount: 0.0-1.0
color.darken(c: color, amount: number) -> color      // amount: 0.0-1.0
color.opacity(c: color, alpha: number) -> color      // alpha: 0.0-1.0
color.mix(a: color, b: color, ratio: number) -> color // ratio: 0.0 = all a, 1.0 = all b

// Query
color.to_hex(c: color) -> string                      // "#FF5733"
color.to_rgba(c: color) -> { r: number, g: number, b: number, a: number }
```
