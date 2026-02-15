# Execution Semantics

This section defines how PEPL programs execute. These semantics are normative — all conforming hosts MUST implement them.

## Action Execution Model

Actions are **atomic transactions**:

1. UI event or test dispatch triggers an action
2. `set` statements execute **sequentially** — each `set` immediately updates the field; subsequent statements see the updated value
3. After the last statement (or `return`), all invariants are checked
4. If all invariants pass → state commits, derived fields recompute, views re-render
5. If any invariant fails → **rollback**: state reverts to pre-action values, host reports the violation

```
User tap → dispatch_action("increment")
  → set count = count + 1    (count now reflects new value)
  → set total = total + 1    (sees updated count)
  → invariant check: count_bounded  ← PASS
  → commit state
  → recompute derived fields
  → re-render views
```

Invariant enforcement is **post-action, not per-statement**. Intermediate states may temporarily violate invariants as long as the final state after the action is valid.

**Invariant scope:** Invariants MUST NOT reference derived fields. Derived fields recompute only after invariants pass and state commits. Referencing a derived field in an invariant expression → compile error. Use the underlying state expressions directly in invariants instead.

**Concurrency:** Actions NEVER execute concurrently. The host processes one action at a time, in order. If a user taps rapidly, actions queue and execute sequentially.

## Derived Field Evaluation

Derived fields recompute after every committed action, in **declaration order**:

```pepl
derived {
  total: number = a + b         // evaluated first
  average: number = total / 2   // evaluated second (sees updated total)
}
```

- A derived field MAY reference state fields and **previously declared** derived fields
- Referencing a later-declared derived field → compile error
- Circular references between derived fields → compile error

## View Rendering

Views are **pure functions** of state + derived state. The host calls `render()` after every committed action to get an updated view tree. Views:

- MUST NOT contain `set` statements (compile error E501)
- MUST NOT call capability functions (no http, storage, etc. in views)
- MAY call pure stdlib functions (math, string, list, etc.)
- MAY use `let` bindings, `if`/`for` control flow, and `match`
- Return a `Surface` — the abstract UI tree

The host diffs the previous and current view trees and applies minimal updates to the platform UI.

## Capability Call Semantics

Capability calls (http, storage, location, etc.) appear **synchronous in PEPL** but execute asynchronously in the host:

```
PEPL: let response = http.get(url)     // execution suspends
  → Host performs async I/O
  → Host records result as event (deterministic replay)
  → Host resumes PEPL with Result value
PEPL: match response { ... }           // execution continues
```

PEPL code never sees concurrency, promises, or callbacks. From the language perspective, `http.get()` is a regular function that returns `Result<HttpResponse, HttpError>`. On replay, the host returns the recorded result — no live request. This preserves determinism.

## Equality Semantics

`==` and `!=` use **structural equality** (deep comparison):

| Type | Equality Rule |
|---|---|
| `number` | IEEE 754 comparison (`NaN != NaN`) |
| `string` | Byte-for-byte UTF-8 comparison |
| `bool` | Value equality |
| `nil` | `nil == nil` is `true` |
| Records | Recursive field-by-field comparison |
| Lists | Same length + element-by-element comparison |
| Sum type variants | Same variant + same payload values |
| Functions/lambdas | Always `false` (not comparable) |
| `color` | RGBA value comparison |

## `any` Type Restrictions

`any` exists **only in stdlib function signatures** — user code cannot declare state fields, action parameters, or `let` bindings with explicit `any` annotation:

| Stdlib Function | Uses `any` | User Handling |
|---|---|---|
| `json.parse()` | Returns `Result<any, JsonError>` | Access fields via `record.get()`, or assign to typed variable (runtime type check) |
| `core.type_of()` | Accepts `any` | Pass any value for type inspection |
| `convert.to_string()` | Accepts `any` | Pass any value for stringification |
| `core.log()` | Accepts `any` | Debug output |

When an `any` value is assigned to a typed state field (e.g., `set items = parsed_data`), the runtime checks that the actual value matches the declared type. Mismatch → runtime trap.

**Enforcement:** Compile error E200 ("Unknown type") if `any` appears in user-authored type annotations. The parser accepts `any` as a valid Type token; the type checker rejects it outside stdlib signatures.

## Nil Handling

Several stdlib functions return `nil` for "not found" cases:

| Function | Returns `nil` when |
|---|---|
| `list.get(l, index)` | Index out of bounds |
| `list.first(l)`, `list.last(l)` | List is empty |
| `list.find(l, f)` | No element matches predicate |
| `record.get(r, key)` | Key not present |

Use `??` (nil-coalescing) to provide defaults:

```pepl
let item = list.get(items, index) ?? default_item
```

Accessing fields on `nil` (e.g., `nil.name`) is a runtime trap.

## Memory Model

- All PEPL values live in WASM linear memory
- Each space has its own memory instance (no sharing between spaces)
- Memory is bounded by the host-configured WASM memory limit
- All values are immutable — `set` replaces the entire value, it does not mutate in place
- The compiled WASM module manages allocation internally (via `alloc`/`dealloc` exports)
- No manual memory management in PEPL — allocation and deallocation are transparent to PEPL code

## Test Execution Semantics

Tests execute outside the space in an isolated environment:

```pepl
tests {
  test "increment increases count" {
    // Each test starts with fresh state (defaults)
    assert count == 0
    // Actions are called as functions
    increment()
    assert count == 1
    increment()
    assert count == 2
  }
}
```

1. Each test case starts with a **fresh space** initialized to default state values
2. Actions are dispatched by calling them as functions: `increment()`, `add_item("task")`
3. After each action: invariants are checked, derived fields recompute
4. `assert expr` → if `false`, test fails with the expression as context. Optional message: `assert expr, "description"`
5. State fields are directly readable in test blocks
6. Tests run sequentially, each isolated from the others
7. Capability calls in tests use **recorded responses**: each test declares `with_responses { }` mapping capability calls to predetermined return values. The test framework replays these instead of calling the host. This preserves determinism and allows offline testing of http/storage-dependent spaces

## Game Loop Semantics

Spaces with `update` and/or `handleEvent` blocks operate as game loops:

```pepl
update(dt: number) {
  // dt = elapsed time since last frame in seconds (e.g., 0.016 for 60fps)
  set x = x + velocity * dt
}

handleEvent(event: InputEvent) {
  // Called once per input event
  match event {
    KeyDown(key) -> {
      if key == "ArrowLeft" { set vx = -3 }
      if key == "ArrowRight" { set vx = 3 }
    }
    _ -> { }
  }
}
```

- The host controls the frame rate (typically 60fps in Phase 0 browser)
- `dt` is delta time in **seconds** (fractional), not milliseconds
- Gas metering applies per `update` call
- Invariants are checked after each `update` call
- `handleEvent` is called for each input event before the next `update`
- Game loop spaces MAY also have regular actions (from UI Button taps, etc.)
- `update` and `handleEvent` are both optional — a space can have one, both, or neither
- `update` without `handleEvent` = animation/simulation only
- `handleEvent` without `update` = event-driven state changes only

## What's Deferred to Later Phases

| Construct | Phase | Reason |
|-----------|-------|--------|
| `canvas` (2D drawing) | Phase 0B+ | Needed for games, not MVP trackers |
| `chart` component | Phase 0B+ | Complex rendering, not launch-critical |
| Animation system (`animate`, `spring`, `ease`) | Phase 0C+ | Progressive enhancement |
| `record` operations (merge, project, rename) | Phase 0B+ | Convenience, not blocking |
| `convert` unit conversions (celsius, miles, etc.) | Phase 0B+ | Type conversions (`to_string`, `to_number`, `parse_int`, `parse_float`, `to_bool`) are Phase 0 |
| Resource annotations (`@resource`) | Phase 1+ | Multi-platform concern |
