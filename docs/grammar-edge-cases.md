# Grammar Edge Cases

## Expression Precedence — Worked Examples

These examples resolve ambiguity. The parser MUST produce these exact parse trees.

```
Expression                          Parsed As                             Rule
───────────                         ─────────                             ────
a + b * c                           a + (b * c)                           * binds tighter than +
a + b * c - d                       (a + (b * c)) - d                     Left-assoc at same level
a > b and c < d                     (a > b) and (c < d)                   Comparison binds tighter than and
a or b and c                        a or (b and c)                        and binds tighter than or
not a and b                          (not a) and b                         Unary binds tightest
-a + b                              (-a) + b                              Unary - binds tightest
a == b == c                          COMPILE ERROR                         No comparison chaining
a + b > c * d or e                   ((a + b) > (c * d)) or e              Full precedence chain
result?                             (result?)                              Postfix ? unwraps Result
result?.field                       ((result?).field)                      ? then field access
value ?? fallback                   (value ?? fallback)                    Nil-coalescing
a ?? b and c                        (a ?? b) and c                         ?? binds tighter than and
http.get(url)?.body                 ((http.get(url))?).body                Call → unwrap → access
```

## Edge Cases and Definitive Answers

| Edge Case | Allowed? | Behavior | Rationale |
|---|---|---|---|
| Empty action body: `action reset() {}` | **Yes** | No-op action, compiles to empty handler | Valid for stubs and placeholder actions |
| Empty state block: `state {}` | **No** | Compile error: "State must have at least one field" | A space with no state has no purpose |
| Variable shadowing: `let x = 1; let x = 2;` | **No** | Compile error: "Variable 'x' already declared in this scope" | Prevents confusion, especially for LLM-generated code |
| Nested lambdas: `fn(x) { fn(y) { x + y } }` | **Yes (Phase 0)** | Closure captures outer variable. Max nesting depth: 3 | Needed for `list.map` callbacks that reference state |
| Nested records: `{ a: { b: { c: 1 } } }` | **Yes** | Max nesting depth: 4 levels | Sufficient for real spaces, prevents pathological structures |
| Sum types in collections: `list<Active(n) \| Done>` | **Yes** | List elements are the sum type, matched via pattern match | Core use case: todo items with status variants |
| String interpolation: `"Count: ${count}"` | **Yes** | Native syntax. Compiler lowers to `string.concat("Count: ", convert.to_string(count))` internally. Zero runtime cost — pure syntax sugar. `\$` escapes literal dollar sign. | Reduces verbosity for common string operations |
| Record spread: `{ ...record, field: value }` | **Yes** | Creates new record with all fields from source, overriding specified fields | Replaces verbose `record.set` chains for single-field updates |
| Result unwrap: `result?` | **Yes** | Returns `Ok` value. On `Err`, WASM trap with error message. | Use `match` for graceful error handling; `?` is for "expect success" paths |
| Nil-coalescing: `value ?? default` | **Yes** | Returns `value` if not nil, else `default` | Prevents nil access traps without verbose `if` checks |
| Derived field: `derived { total: number = x + y }` | **Yes** | Recomputes after every action; read-only (cannot be `set`) | Eliminates redundant state for computed values |
| Credentials block: `credentials { key: string }` | **Yes** | Compile-time declaration; credential names become read-only bindings within the space | Host prompts user to configure before space runs |
| Return in action: `return` | **Yes** | Exits action early; all prior `set` statements are applied | No return values — actions are void |
| Tests outside space: `tests { test "name" { ... } }` | **Yes** | Tests run against the space but are not part of its runtime | Separates test code from production code |
| Division by zero: `5 / 0` | **Runtime trap** | WASM trap → space pauses → error overlay → rollback offered | Deterministic language must not produce undefined results |
| `0.0 / 0.0` | **Runtime trap** | Same as division by zero. NaN values cannot enter state — any operation that would produce NaN (including `math.sqrt(-1)`) traps instead | Determinism: NaN breaks equality and comparison guarantees |
| Negative modulo: `-7 % 3` | **`-1`** | IEEE 754 remainder (truncation toward zero). `7 % -3` is also `1`. | Consistent with WASM `f64.rem` semantics |
| Integer overflow | **IEEE 754 float wrapping** | All numbers are 64-bit floats — overflow follows IEEE 754 | Consistent with `number` being a float type |
| Accessing nil field: `record.field` on nil | **Runtime trap** | WASM trap → same recovery as division by zero | Null safety is explicit via optional handling |
| Recursive functions | **Not allowed (Phase 0)** | Compile error: "Recursion is not supported" | Gas metering complexity; use `for` loops instead |
| Mutating state outside action | **Compile error** | "State can only be modified inside an action via 'set'" | Core invariant of PEPL |
| Self-referencing state: `state { x: number = x + 1 }` | **Compile error** | "State initializers cannot reference other state fields" | State initializers MAY call pure stdlib functions (`list.empty()`, `list.of(1,2,3)`, `math.PI`). They MUST NOT call capability functions or reference other state fields |
| Setting derived field: `set total = 100` | **Compile error** | "Derived fields are read-only — they recompute automatically" | Derived state is computed, not mutable |
| Blocks out of order: action before state | **Compile error** | "Block ordering violated: 'action' must come after 'state'" | Grammar enforces deterministic structure |
| Expression-body lambda: `fn(x) x + 1` | **Compile error** | "Lambdas require block body: fn(x) { x + 1 }" | One syntax rule — no ambiguity |
| Block comment: `/* ... */` | **Compile error** | "Only single-line comments (//) are supported" | One comment style — reduces LLM confusion |
| `?` on non-Result: `count?` | **Compile error** | "Operator '?' requires Result type, got number" | Type mismatch — `?` is only valid on Result values |
| `??` on non-nullable: `count ?? 0` | **Compile warning** | "Operator '??' has no effect — left side is never nil" | Allowed (no-op) but flagged as unnecessary |

## Structural Limits

PEPL imposes **no artificial limits** on the number of state fields, actions, views, derived fields, sum type variants, or credentials a space can have. A space can be as complex as the user needs it to be.

The only limits are **structural sanity constraints** that prevent pathological nesting depths (which harm readability and compilability regardless of space complexity):

| Limit | Value | Rationale |
|---|---|---|
| Max lambda nesting depth | 3 | `fn(item) { fn(subitem) { fn(x) { ... } } }` |
| Max record nesting depth | 4 | `{ a: { b: { c: { d: val } } } }` |
| Max expression depth | 16 | Prevents absurd nesting from code generators |
| Max `for` nesting depth | 3 | Nested loops beyond 3 suggest wrong approach |
| Max parameters per action/fn | 8 | Practical limit |

These limits are enforced at compile time. Exceeding them produces a clear error message naming the limit and the current count.

The true boundaries of a space are **resource-based**, not construct-based. Resource limits (memory, CPU, storage) are host-configured and hardware-adaptive — they are not part of the PEPL language specification. See the host's documentation for enforcement details.
