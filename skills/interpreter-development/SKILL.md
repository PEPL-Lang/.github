---
name: interpreter-development
description: Guide for building and using the pepl-eval tree-walking evaluator (Phase C6). Triggers when working on the evaluator crate, implementing execution semantics, testing language behavior end-to-end, or generating golden reference output. Reference implementation that validates PEPL semantics before WASM codegen.
---

# Interpreter Development (pepl-eval)

The tree-walking evaluator is the **reference implementation** of PEPL execution semantics. It executes programs directly from the typed AST without WASM compilation. Its output becomes the golden reference for validating WASM codegen correctness.

## Architecture

```
Typed AST (from C5)
    ↓
pepl-eval (tree-walking evaluator)
    ├── State management (BTreeMap<String, EvalValue>)
    ├── Action dispatch (atomic transactions with rollback)
    ├── Expression evaluation (full precedence chain)
    ├── Stdlib dispatch (routes to pepl-stdlib implementations)
    ├── View rendering (constructs Surface trees)
    └── Test runner (fresh state per test, capability mocking)
```

## Core Types

```rust
// EvalValue — runtime representation of all PEPL values
enum EvalValue {
    Number(f64),       // IEEE 754, NaN traps
    String(String),
    Bool(bool),
    Nil,
    List(Vec<EvalValue>),             // ordered, immutable semantics
    Record(BTreeMap<String, EvalValue>), // BTreeMap for determinism
    SumVariant { tag: String, fields: Vec<EvalValue> },
    Function { params: Vec<String>, body: Block, env: Environment },
    ActionRef { name: String, bound_args: Vec<EvalValue> },
    Surface(SurfaceNode),
}
```

**CRITICAL:** Use `BTreeMap` everywhere. Never `HashMap`. This is non-negotiable for determinism.

## Execution Rules

Reference: `docs/execution-semantics.md` — these are normative.

### Action Atomicity
1. Save pre-action state snapshot
2. Execute `set` statements **sequentially** — each `set` immediately updates state
3. After last statement (or `return`): check ALL invariants
4. If invariants pass → commit state, recompute derived fields, re-render views
5. If any invariant fails → **rollback** to pre-action snapshot

### Derived Fields
- Recompute after every committed action, in **declaration order**
- May reference state fields and previously declared derived fields
- Circular or forward references → compile error (caught by C4/C5, not evaluator)

### Views Are Pure
- No `set` statements (compile error, caught earlier)
- No capability calls
- Walk AST to construct `Surface` tree
- Return JSON-serializable tree matching `docs/host-integration.md` format

### Capability Calls
- In normal execution: return `Err("unmocked_call")`
- In test `with_responses` context: match call signature → return recorded Result
- No actual I/O — evaluator is fully deterministic

### Equality
- Structural deep comparison for records, lists, sum variants
- Functions/lambdas: always `false`
- `NaN != NaN` (but NaN should never exist — it traps)

## Stdlib Integration

The evaluator routes `module.function()` calls to `pepl-stdlib` implementations:

```rust
// Dispatch qualified calls
fn eval_qualified_call(&mut self, module: &str, function: &str, args: Vec<EvalValue>) -> EvalResult {
    match module {
        "math" => self.stdlib.math.call(function, args),
        "string" => self.stdlib.string.call(function, args),
        "list" => self.stdlib.list.call(function, args),
        // ... etc
    }
}
```

**Dependency:** S1–S5 must be complete before C6 can execute all canonical examples.

## Golden Reference

After implementing the evaluator, generate golden reference fixtures:

1. Execute all 7 canonical examples (Counter, TodoList, UnitConverter, WeatherDashboard, PomodoroTimer, HabitTracker, QuizApp)
2. Capture state snapshots after each action dispatch
3. Capture Surface tree JSON after each render
4. Store as test fixtures in `tests/golden/`
5. WASM codegen (C7) later validates its output against these fixtures

## Testing

- Every evaluation path needs unit tests
- 100-iteration determinism test: same program → identical output × 100
- Test all edge cases from `docs/grammar-edge-cases.md` (division by zero, negative modulo, nil access, etc.)
- Test action rollback on invariant failure
- Test `with_responses` capability mocking
- Test game loop (`update` + `handleEvent`)

## Rules

- **BTreeMap only** — never HashMap, never IndexMap
- **No floating-point surprises** — trap on NaN, trap on division by zero
- **Spec is authoritative** — `docs/execution-semantics.md` defines behavior. If the evaluator disagrees, fix the evaluator.
- **No I/O** — the evaluator is pure. All capability calls are mocked.
- **Determinism non-negotiable** — 100-iteration proof required
