---
applyTo: "**/type_checker/**,**/invariant_checker/**,**/codegen/**,**/gas/**"
---

# Compiler Pipeline — PEPL

The PEPL compiler has 5 stages after parsing. Each stage is pure and deterministic.

## Stages

```
AST → Type Checker → Typed AST → Invariant Checker → Verified AST → WASM Codegen → .wasm
                                                                         ↓
                                                                    Gas Metering
```

### Type Checker

- Validates all expressions against PEPL's type system
- Ensures `set` only appears inside actions
- Ensures views are pure (no `set`, no capability calls)
- Validates `derived` field references (only previously declared fields)
- Rejects `any` in user-authored type annotations (E200)
- Validates sum type exhaustiveness in `match` (E210)
- Checks action parameter types, return statements, let bindings
- Reference: `docs/overview.md` § "Type Safety" (Guarantee), `docs/execution-semantics.md` § "any Type Restrictions"

### Invariant Checker

- Validates invariant expressions are valid boolean expressions
- Ensures invariants don't reference derived fields (compile error)
- Validates structural limits (lambda depth ≤ 3, record depth ≤ 4, expression depth ≤ 16, for depth ≤ 3, params ≤ 8)
- Checks state initializer restrictions (pure stdlib only, no capability calls, no cross-field refs)
- Reference: `docs/overview.md` § "Invariants", `docs/grammar-edge-cases.md` § "Structural Limits"

### WASM Code Generator

- Uses `wasm-encoder` crate (not hand-rolled binary encoding)
- Emits WASM binary for each space
- Injects gas metering at loop boundaries and function call sites
- Handles `set` desugaring for nested fields: `set a.b.c = x` → `set a = { ...a, b: { ...a.b, c: x } }`
- NaN prevention: division and sqrt emit trap-on-NaN guards
- **Source Map**: embedded as WASM custom section `pepl-source-map`, maps WASM byte offsets to PEPL source lines/columns
- **Test Exports**: test blocks compile to `__test_N()` WASM exports; `__test_count()` returns count; asserts become WASM traps
- **LLM Reference**: `get_reference()` and `get_stdlib_table()` exports from `pepl-wasm` produce machine-generated language reference from the compiler's type registry
- Reference: `docs/compiler.md` § "Compiler Strategy", "Compiler Bootstrap Approach"

### Gas Metering

- Injected at `for` loop boundaries
- Injected at function/action call sites
- Injected at `update()` call boundaries (game loop)
- Host configures gas limit per action/update invocation
- Gas exhaustion → WASM trap → space pauses

## Error Output

Compiler errors are structured JSON objects with:
- `code`: E-code string (e.g., "E201")
- `message`: Human-readable description
- `line`, `column`, `length`: Source location
- `source_line`: The exact source text
- `suggestion`: Optional fix hint (for LLM re-prompting)
- `severity`: "error" (Phase 0 has no warnings)

Maximum 20 errors per compilation (fail-fast after 20).

## Rules

- Each stage is pure — no I/O, no randomness, no system state
- Stages compose: AST → TypedAST → VerifiedAST → WASM
- Every error code (E100–E699) must have at least one test
- 100-iteration determinism proof: same source → identical .wasm bytes × 100
