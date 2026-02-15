---
applyTo: "**/*test*,**/tests/**"
---

# Testing & Determinism — PEPL

## Every Test Must Prove

1. **Correctness** — output matches the spec (see spec files in `docs/`)
2. **Determinism** — 100 iterations produce identical output
3. **Idempotence** (where applicable) — `f(f(x)) == f(x)`

## Test Structure

```rust
#[test]
fn test_feature_works() {
    // Test the happy path
}

#[test]
fn test_feature_error_case() {
    // Test specific error conditions — each E-code needs at least one test
}

#[test]
fn test_feature_determinism_100_iterations() {
    let input = "space Counter { state { count: number = 0 } ... }";
    let first = compile(input).unwrap();
    for i in 0..100 {
        let result = compile(input).unwrap();
        assert_eq!(first, result, "Determinism failure at iteration {}", i);
    }
}
```

## PEPL-Specific Test Categories

### Parser Tests
- Valid PEPL source → correct AST structure
- Invalid PEPL source → correct error code (E100–E199)
- Edge cases from `docs/grammar-edge-cases.md` § "Edge Cases and Definitive Answers"
- Operator precedence from § "Expression Precedence — Worked Examples"

### Type Checker Tests
- Type errors → correct E-code (E200–E299)
- `any` in user annotations → E200
- Non-exhaustive match → E210
- `set` outside action → E501

### WASM Output Tests
- Same source → identical .wasm bytes (determinism)
- Gas metering present at loop/call boundaries
- NaN-producing operations → trap guards emitted
- Nested `set` desugaring produces correct code

### Stdlib Tests
- Every function: normal case, edge case, error case
- Sort stability: equal-key elements preserve order
- String operations: empty strings, Unicode, multi-byte
- `list.of` variadic: 0 args, 1 arg, many args

## Rules

- No `#[ignore]` without a tracking issue
- No `unwrap()` in test setup — use `.expect("description")`
- Test file names match module names: `parser.rs` tests in `parser::tests`
- Integration tests go in `tests/` directory
- **Never skip the 100-iteration determinism test** — it's the core promise
