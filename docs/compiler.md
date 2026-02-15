# Compiler

## Compiler Strategy

Phase 0 compiler is a **thin transpiler** that:
1. Parses PEPL subset into AST
2. Type-checks and validates invariants
3. Emits WASM bytecode for the subset constructs
4. Injects gas metering at loop/call boundaries

This is **not** a full optimizing compiler. Advanced optimizations (dead code elimination, constant folding, etc.) are deferred to post-Phase-0 when the language and usage patterns stabilize. The Phase 0 compiler prioritizes **correctness over performance**.

## Compiler Bootstrap Approach

Phase 0 uses a **transpiler bootstrap** — not direct WASM binary emission:

```
Phase 0 Compilation Path:
  PEPL Source
      ↓
  PEPL Parser (Rust) → PEPL AST
      ↓
  Type Checker + Invariant Checker → Verified AST
      ↓
  WASM Emitter (wasm-encoder crate) → .wasm binary
      ↓
  WebAssembly.instantiate() in sandbox
```

**Why use wasm-encoder instead of hand-rolling WASM binary?**

| Hand-rolled binary encoding | wasm-encoder crate |
|---|---|
| Write binary encoding by hand | Type-safe WASM module construction via Rust API |
| Debug by reading hex bytes | Debug with wasmparser validation + WAT disassembly |
| Must handle all WASM encoding edge cases | Crate handles encoding, you handle semantics |
| Estimated: 3-6 months for correct emitter | Estimated: 2-4 weeks for emitter |
| Optimal control over output | Standard, well-tested binary output |

Phase 0 priority is **validating the PEPL language design**, not optimizing compiler output. Using `wasm-encoder` lets the language grammar iterate rapidly without rewriting the code generator each time.

**The compiler itself runs as WASM** — compiled from Rust via `wasm-pack`, loaded in a Web Worker in Phase 0 (browser). This means the compilation pipeline is:

```
Browser tab → Web Worker → PEPL compiler (WASM) → wasm-encoder → .wasm binary → sandbox
```

No server needed for compilation. The entire pipeline runs client-side.

## Compilation Pipeline

```
PEPL Source Code
      ↓
┌──────────┐
│  Lexer   │ → Token stream
└────┬─────┘
     ↓
┌──────────┐
│  Parser  │ → AST (Abstract Syntax Tree)
└────┬─────┘
     ↓
┌───────────────┐
│ Type Checker  │ → Typed AST (all types resolved, errors caught)
└────┬──────────┘
     ↓
┌────────────────────┐
│ Invariant Checker  │ → Verified AST (bounds, termination, state validity)
└────┬───────────────┘
     ↓
┌──────────────┐
│ WASM Codegen │ → .wasm binary + source map + test exports
└──────────────┘
```

> Phase 1+ adds an **Optimizer** stage (dead code elimination, constant folding) between Invariant Checker and WASM Codegen.

**Compiler is Rust, compiled to WASM** — runs in a Web Worker in Phase 0 (browser).

### Additional Codegen Outputs

- **Source Map** — embedded as WASM custom section `pepl-source-map`, maps WASM byte offsets to PEPL source lines/columns
- **Test Exports** — test blocks compile to `__test_N()` WASM exports; `__test_count()` returns test count; asserts become WASM traps
- **LLM Reference** — `get_reference()` and `get_stdlib_table()` exports from `pepl-wasm` produce machine-generated language reference and stdlib table from the compiler's own type registry

### CompileResult

The full compilation pipeline returns a `CompileResult` (JSON-serializable) containing:
- WASM binary bytes (or null on failure)
- Structured errors/warnings
- AST (serialized)
- Source hash (SHA-256)
- State field list, action list, view list
- Required/optional capabilities, credentials
- Language and compiler version strings
- Source map (optional)

## Incremental Compilation

The compiler supports **incremental compilation** for the Evolve operation. When a space is edited via Evolve, only the changed AST nodes are re-type-checked and re-codegen'd:

1. Parse new source → AST_new
2. Diff AST_new against cached AST_old (from last successful compilation)
3. Identify changed subtrees (added/modified/removed declarations)
4. Re-run type checking only on changed subtrees + their dependents
5. Re-codegen only affected WASM functions; splice into cached module

This means a small edit to a large space does not require recompiling the entire space. The compiler caches the typed AST and WASM module between compilations.

> Phase 0 implements full recompilation. AST diff infrastructure (`ast_diff.rs`) is complete for Phase 1 incremental compilation.

## Bounds Checking

| Check | When | What |
|-------|------|------|
| Array bounds | Compile-time when possible, runtime otherwise | `list.get(l, -1)` → returns `nil` |
| Integer overflow | Runtime | Wraps (IEEE 754 float behavior) |
| Stack depth | Compile-time | Recursion depth limit enforced |
| Loop termination | Compile-time heuristic + runtime gas | Gas metering prevents infinite loops |
| Memory allocation | Runtime | WASM linear memory limit, `memory.grow` on exhaustion |

## Invariant Checking

```pepl
space Counter {
  state {
    count: number = 0
    max: number = 100
  }

  invariant count_bounded {
    count >= 0 and count <= max
  }

  action increment() {
    if count < max {
      set count = count + 1
    }
    // Invariant automatically verified after action
  }
}
```

## Compiler Error Message Format

Every compiler error is a structured object. The View Layer renders these — it must not parse free-form strings.

```json
{
  "errors": [
    {
      "file": "WaterTracker.pepl",
      "line": 12,
      "column": 5,
      "end_line": 12,
      "end_column": 22,
      "severity": "error",
      "code": "E201",
      "category": "type",
      "message": "Type mismatch: expected 'Number', found 'String'",
      "suggestion": "Use convert.to_int(value) to convert String to Number",
      "source_line": "  set state.count = \"hello\""
    }
  ],
  "warnings": [],
  "total_errors": 1,
  "total_warnings": 0
}
```

**Error Code Ranges:**

| Range | Category | Examples |
|---|---|---|
| E100–E199 | Syntax | E100: Unexpected token, E101: Unclosed brace, E102: Invalid keyword |
| E200–E299 | Type | E200: Unknown type, E201: Type mismatch, E202: Wrong argument count, E210: Non-exhaustive match |
| E300–E399 | Invariant | E300: Invariant unreachable, E301: Invariant references unknown field |
| E400–E499 | Capability | E400: Undeclared capability used, E401: Capability not available in sandbox |
| E500–E599 | Scope | E500: Variable already declared, E501: State mutated outside action, E502: Recursion not allowed |
| E600–E699 | Structure | E600: Block ordering violated, E601: Derived field modified via set, E602: Expression-body lambda, E603: Block comment used, E604: Undeclared credential used, E605: Credential modified via set, E606: Empty state block, E607: Structural limit exceeded |

**Severity Levels:** `error` (blocks compilation), `warning` (compiles but flagged — Phase 0 has no warnings; reserved for Phase 1+)

**Guarantees:**
- Every error includes the exact source line for context
- Every error includes a suggestion when one exists (LLM-generated code can be auto-fixed by re-prompting with the error)
- Error codes are stable — once assigned, a code never changes meaning
- Multiple errors are reported up to a limit of 20 per compilation (fail-fast after 20)

## Error Handling

### Compile-Time Errors

| Error Type | Example | Recovery |
|-----------|---------|----------|
| Syntax error | `set count =` (missing value) | Show line/col, suggest fix |
| Type error | `"hello" + 5` | Show expected/actual types |
| Undefined reference | `set unknown_field = 0` | Suggest similar names |
| Invariant violation | Action can provably violate invariant | Show counterexample |
| Capability error | Using `location` without declaring | Suggest adding to capabilities |

### Runtime Errors

| Error Type | Behavior | Recovery |
|-----------|----------|----------|
| Gas exhaustion | Execution halted | Show "space took too long", offer code review |
| Memory overflow | Execution halted | Show "space used too much memory", suggest optimization |
| Division by zero | WASM trap — space pauses, error overlay, rollback offered | Same as gas exhaustion |
| nil access | WASM trap — space pauses, error overlay, rollback offered | Same as gas exhaustion |
| Assertion failure | Action rolled back | Show which invariant failed |
