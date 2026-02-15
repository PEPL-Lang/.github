# Language Structure

## Space Structure

Block ordering is enforced by the grammar. Blocks MUST appear in this order:

```pepl
space SpaceName {
  // 1. Type declarations (optional, any number)
  type Status = | Active | Paused | Done

  // 2. State block (required, exactly one)
  state {
    field_name: Type = default_value
  }

  // 3. Capabilities block (optional)
  capabilities {
    required: [http, storage]
    optional: [location]
  }

  // 4. Credentials block (optional)
  credentials {
    api_key: string,  // API key for the weather service
  }

  // 5. Derived state block (optional)
  derived {
    computed_field: Type = expression_using_state
  }

  // 6. Invariants (optional, any number)
  invariant name {
    boolean_expression
  }

  // 7. Actions (optional, any number)
  action action_name(param: Type) {
    set field = new_value
  }

  // 8. Views (required, at least one)
  view main() -> Surface {
    Column {
      Text { value: "hello", }
    }
  }

  // 9. Update loop (optional, for game/animation spaces)
  update(dt: number) {
    set field = field + velocity * dt
  }

  // 10. Event handler (optional, for game/interactive spaces)
  handleEvent(event: InputEvent) {
    match event {
      KeyDown(key) -> { /* handle key */ }
      _ -> { }
    }
  }
}

// 11. Tests (optional, outside space)
tests {
  test "description" {
    assert expression
  }
}
```

## Type System

**Primitive Types**:
- `number` — 64-bit float (IEEE 754). Also used for timestamps (milliseconds since epoch) and durations (milliseconds).
- `string` — UTF-8 text. Supports `${expr}` interpolation.
- `bool` — `true` / `false`
- `nil` — Absence of value
- `color` — Phase 0: hex string literals (`"#FF5733"`). Full language adds `"rgb()"`, named colors, and `color` module functions.

> **Resolution:** `timestamp` and `duration` are not separate types — they are `number` (milliseconds). This eliminates type conversion overhead and stdlib API complexity. The `time` module functions accept and return `number`.

**Composite Types**:
- `list<T>` — Ordered, immutable collections. `T` is a built-in type parameter, not user-definable.
- `{ field: Type, ... }` — Anonymous record types (structural typing)

> **Removed:** `record<{ field: Type }>` syntax and bare `record` type. Use `{ field: Type }` for all record types. This eliminates two syntaxes for the same concept.

**Sum Types**:
```pepl
type Shape =
  | Circle(radius: number)
  | Rectangle(width: number, height: number)
  | Triangle(a: number, b: number, c: number)
```

**Pattern Matching on Sum Types** (exhaustive — all variants required):
```pepl
let area = match shape {
  Circle(r) -> math.PI * r * r
  Rectangle(w, h) -> w * h
  Triangle(a, b, c) -> heron(a, b, c)
}

// Wildcard for catch-all:
let label = match priority {
  Critical -> "!!!"
  High -> "!"
  _ -> ""
}
```

The compiler enforces **exhaustiveness**: every variant must be covered, or a wildcard `_` must be present. Missing a variant produces compile error E210: "Non-exhaustive match: missing variant 'X'".

**Nil Narrowing**: In `if expr != nil { ... }`, the type of `expr` narrows from `T | nil` to `T` within the block body. This applies to variables bound via `list.get`, `list.find`, `list.first`, `list.last`, or any expression typed `T | nil`.

```pepl
let item = list.get(items, index)    // type: Item | nil
if item != nil {
  log(item.name)                     // type narrowed to Item — .name is valid
}
let fallback = item ?? default_item  // also valid — ?? unwraps nil
```

**Function Types**:
```pepl
type Predicate = (number) -> bool
```

> **Removed:** User-defined generics (`type Reducer<T, U> = ...`). Only built-in parameterized types exist: `list<T>`, `Result<T, E>`. This eliminates a complex feature that LLMs rarely generate correctly and that adds significant compiler complexity.

**Result Type** (built-in, used for all capability operations):
```pepl
// Result<T, E> is a built-in sum type — every capability call returns one
type Result<T, E> =
  | Ok(value: T)
  | Err(error: E)

// Usage: always match on Result to handle success/failure
action fetch_data() {
  let response = http.get("https://api.example.com/data")
  match response {
    Ok(r) -> {
      let parsed = json.parse(r.body)
      match parsed {
        Ok(data) -> { set items = data }
        Err(e) -> { set error = e.message }
      }
    }
    Err(e) -> { set error = e.message }
  }
}
```

`Result` is one of PEPL's core patterns — it replaces exceptions. There is no `try/catch`. All errors are values, handled explicitly via `match`.

**Dimension & Layout Types** (built-in, compiler-known):
```pepl
// These are built-in sum types used for component props.
// Users don't define these — they're part of the component type system.
type dimension = | Px(number) | Auto | Fill | Percent(number)
// Usage: width: 100,  (number literal → Px)
//        width: Auto,
//        height: Fill,

type edges = | Uniform(number) | Sides(top: number, bottom: number, start: number, end: number)
// Usage: padding: 16,                   (number literal → Uniform)
//        padding: Sides(8, 8, 16, 16),

type alignment = | Start | Center | End | Stretch | SpaceBetween | SpaceAround

type border_style = { width: number, color: color, style?: string }
type shadow_style = { offset_x: number, offset_y: number, blur: number, color: color }
```

> **Implementation note:** For LLM simplicity, number literals in dimension/edges positions are automatically coerced to `Px(n)` / `Uniform(n)` by the compiler. The LLM can write `width: 100` instead of `width: Px(100)`. String values like `"auto"` are NOT supported — use `Auto` variant.

## Phase 0 Subset

Phase 0 does **not** require the entire PEPL language. A **subset of ~140 language constructs** is sufficient to cover common application categories (trackers, dashboards, simple tools, games). The compiler approach for Phase 0 is a **thin transpiler** — not a from-scratch optimizing compiler.

### Phase 0 Subset Scope

| Category | Included Constructs | Count |
|----------|-------------------|-------|
| Core keywords | `space`, `state`, `action`, `view`, `set`, `if`/`else`, `for`, `let`, `match`, `invariant`, `capabilities`, `credentials`, `derived`, `update`, `handleEvent`, `tests`, `assert`, `return` | 19 |
| Types | `number`, `string`, `bool`, `nil`, `color`, `list` | 6 |

> Phase 0 `color` values are string hex literals (e.g., `"#FF5733"`). Programmatic `color` module functions (`color.rgb`, `color.lighten`, etc.) are deferred to the full language.
>
> **Contextual field names:** All keywords are valid as record field names (e.g., `{ color: string }`, `{ type: "widget" }`). Keywords remain reserved in all other positions.

| Operators | `+`, `-`, `*`, `/`, `%`, `==`, `!=`, `<`, `>`, `<=`, `>=`, `not`, `and`, `or`, `?` (Result unwrap), `??` (nil-coalescing), `...` (record spread) | 17 |
| Stdlib — Core | `core.log`, `core.assert`, `core.type_of`, `core.capability` | 4 |
| Stdlib — Math | `math.abs`, `math.min`, `math.max`, `math.floor`, `math.ceil`, `math.round`, `math.round_to`, `math.pow`, `math.clamp`, `math.sqrt`, `math.PI`, `math.E` | 12 |
| Stdlib — String | `string.length`, `string.concat`, `string.contains`, `string.slice`, `string.trim`, `string.split`, `string.to_upper`, `string.to_lower`, `string.starts_with`, `string.ends_with`, `string.replace`, `string.replace_all`, `string.pad_start`, `string.pad_end`, `string.repeat`, `string.join`, `string.format`, `string.from`, `string.is_empty`, `string.index_of` | 20 |
| Stdlib — List | `list.length`, `list.append`, `list.prepend`, `list.get`, `list.set`, `list.insert`, `list.remove`, `list.update`, `list.map`, `list.filter`, `list.sort`, `list.find`, `list.find_index`, `list.contains`, `list.reverse`, `list.slice`, `list.flatten`, `list.reduce`, `list.range`, `list.of`, `list.empty`, `list.repeat`, `list.concat`, `list.unique`, `list.take`, `list.drop`, `list.any`, `list.some`, `list.every`, `list.count`, `list.first`, `list.last`, `list.index_of`, `list.zip` | 34 |
| Stdlib — Record | `record.get`, `record.set`, `record.has`, `record.keys`, `record.values` | 5 |
| Stdlib — Time | `time.now`, `time.format`, `time.diff`, `time.day_of_week`, `time.start_of_day` | 5 |
| Stdlib — Convert | `convert.to_string`, `convert.to_number`, `convert.parse_int`, `convert.parse_float`, `convert.to_bool` | 5 |
| Stdlib — JSON | `json.parse`, `json.stringify` | 2 |
| Stdlib — Timer | `timer.start`, `timer.start_once`, `timer.stop`, `timer.stop_all` | 4 |
| Stdlib — HTTP | `http.get`, `http.post`, `http.put`, `http.patch`, `http.delete` | 5 |
| Stdlib — Storage | `storage.get`, `storage.set`, `storage.delete`, `storage.keys` | 4 |
| Stdlib — Location | `location.current` | 1 |
| Stdlib — Notifications | `notifications.send` | 1 |
| UI Components | `Text`, `Button`, `TextInput`, `Column`, `Row`, `Scroll`, `ScrollList`, `ProgressBar`, `Modal`, `Toast` | 10 |
| **Total** | | **~154** |
