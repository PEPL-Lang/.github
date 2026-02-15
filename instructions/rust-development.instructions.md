---
applyTo: "**/*.rs"
---

# Rust Development — PEPL Standards

## Determinism Rules (Non-Negotiable)

- Use `BTreeMap`/`BTreeSet` — NEVER `HashMap`/`HashSet`
- No `rand`, no randomness, no system time in compiler or stdlib
- IEEE 754 floating point — all numbers are f64, specify rounding where relevant
- No `unwrap()` in library code — use `?` or `.ok_or(Error::...)?`
- All iteration order must be deterministic
- NaN prevention: operations that would produce NaN (0.0/0.0, sqrt(-1)) must trap

## Error Handling

- All fallible operations return `crate::Result<T>`
- Use `thiserror` for error types
- Compiler errors include line:column and error code (E100–E699)
- Never swallow errors

## Naming

- Types: `PascalCase` (e.g., `SpaceNode`, `ActionDecl`)
- Functions: `snake_case` (e.g., `parse_space`, `check_types`)
- Constants: `SCREAMING_SNAKE_CASE` (e.g., `MAX_LAMBDA_DEPTH`)
- AST node names match grammar productions (e.g., `SetStatement`, `MatchExpr`)

## Testing

- Every public function has a unit test
- Every error path has a test (each E-code has at least one test)
- 100-iteration determinism proof: same input → identical output × 100
- Idempotence proof where applicable: `f(f(x)) == f(x)`

## Architecture

```
PEPL Source → Parser → AST → Type Checker → Invariant Checker → WASM Codegen
                                                                    ↓
                                                              wasm-encoder → .wasm
```

## WASM Target

The compiler itself compiles to WASM (via `wasm-pack`) to run in browser Web Workers. Code must be `no_std`-compatible where possible, or use `wasm-bindgen` for browser integration.

## Reference

- Spec: `docs/grammar.md` (grammar), `docs/execution-semantics.md` (semantics), `docs/compiler.md` (error codes)
- Error code ranges: E100–E199 (syntax), E200–E299 (type), E300–E399 (invariant), E400–E499 (capability), E500–E599 (scope), E600–E699 (structure)
