# PEPL Compliance Findings

> **ARCHIVED** — This document is historical. All Phase 0 findings (F1–F34) have been resolved across 27 ORCHESTRATION steps. The spec files referenced below have been reorganized into `docs/` and some merged (compiler.md + compilation-pipeline.md, language-structure.md + phase-0-subset.md, ui-components.md + accessibility.md). `pepl.md` references refer to an external project spec that has since been removed; equivalent information now lives in `reference.md` and the `docs/` specs. References below use original filenames from the audit date.

Post-build audit comparing the three PEPL repos (`pepl`, `pepl-stdlib`, `pepl-ui`) against the PEPL specification files. Findings only — no decisions or remediation plans. This document is appendable; further analysis will be added on subsequent passes.

**Audit date:** 2026-02-13
**Repos audited:** `pepl` (7 crates, 498 tests), `pepl-stdlib` (13 modules, 501 tests), `pepl-ui` (10 components, 379 tests)
**Spec files compared:** overview.md, grammar.md, language-structure.md, compiler.md, grammar-edge-cases.md, phase-0-subset.md, execution-semantics.md, capability-system.md, host-integration.md, compilation-pipeline.md, ui-components.md, stdlib.md, phase-0-stdlib-reference.md, llm-generation-contract.md, transformation-examples.md, project-metadata.md, future.md, accessibility.md

---

## Compliance Summary

| Area | Spec Compliance | Notes |
|------|:-:|-------|
| Grammar / Syntax | ~95% | All Phase 0 constructs parse correctly |
| Type System | ~90% | Core types, generics, sum types, Result/nil-coalescing all present |
| Stdlib (pure modules) | ~85% | Function coverage is high but name alignment has gaps |
| Stdlib (capability modules) | ~85% | All 5 capability modules present; minor discrepancies |
| WASM Codegen | ~70% | Core codegen works; lambda, record spread, Result unwrap are placeholders |
| UI Components | ~90% | All 10 Phase 0 components present with typed props |
| Error Codes | ~80% | 25 codes defined; some defined but not wired into checker |
| Host Integration | ~75% | Missing `env.get_timestamp` import; `dealloc` export absent |
| LLM Generation Contract | ~80% | Canonical examples compile; generation contract not automated |

---

## Finding 1: Stdlib Name Mismatches Between Type Checker and Spec

**Severity:** CRITICAL
**Spec source:** phase-0-stdlib-reference.md
**Code location:** `pepl/crates/pepl-compiler/src/stdlib.rs`

The type checker's stdlib registry uses function names that **match the spec** for most functions. However, cross-checking reveals the following discrepancies:

### 1a: `storage` module — extra `remove` function

| Type Checker (stdlib.rs) | Spec (phase-0-stdlib-reference.md) | Actual Stdlib (pepl-stdlib) |
|---|---|---|
| `storage.get`, `storage.set`, **`storage.remove`**, `storage.delete`, `storage.keys` | `storage.get`, `storage.set`, `storage.delete`, `storage.keys` | `storage.get`, `storage.set`, `storage.delete`, `storage.keys` |

The type checker registers **both** `storage.remove` and `storage.delete`. The spec and stdlib only define `storage.delete`. LLM-generated code using `storage.remove` would pass type checking but fail at runtime.

### 1b: `list` module — function count exceeds spec

| Count | Source |
|---|---|
| 31 functions | Spec (phase-0-stdlib-reference.md) |
| 33 functions | Type checker (stdlib.rs) |
| ~32 functions | Actual stdlib (pepl-stdlib) |

The type checker registers `list.update` and `list.set` which appear interchangeable — both take `(list, index, item) -> list`. The spec lists `list.set` only. The type checker also registers `list.insert` which is not in the spec's Phase 0 table.

Additional list functions in type checker but **not in Phase 0 spec**: `list.find_index`, `list.zip`, `list.flatten`.

### 1c: `list.any` vs `list.some` naming

| Spec Name | Type Checker Name | Stdlib Name |
|---|---|---|
| `list.any` | `list.some` | `list.some` |

The spec calls it `list.any`. Both the type checker and stdlib implementation call it `list.some`. LLM-generated code following the spec reference would use `list.any` and get a type error.

### 1d: `list.drop` missing

The spec defines `list.drop(xs, count) -> list<T>` (return all elements after first `count`). Neither the type checker nor the stdlib implements it.

---

## Finding 2: Lambda/Closure WASM Codegen Is a Placeholder

**Severity:** CRITICAL
**Spec source:** grammar.md (lambda syntax), execution-semantics.md
**Code location:** `pepl/crates/pepl-codegen/src/expr.rs` (lines 68–72)

The WASM code generator emits `nil` for all lambda expressions:

```rust
ExprKind::Lambda(_lambda) => {
    // Lambda closures are lowered in a later phase.
    // For now emit nil as a placeholder.
    emit_nil_lit(f)
}
```

**Impact:** Every stdlib function that takes a callback is broken in WASM output:
- `list.map`, `list.filter`, `list.sort`, `list.find`, `list.reduce`, `list.flat_map`, `list.every`, `list.some`, `list.count`, `list.sort_by`, `list.find_index`
- `ScrollList` component's `render` and `key` props
- Any `on_change` callback with inline lambda: `on_change: fn(v) { set_value(v) }`

The **interpreter** (`pepl-eval`) handles lambdas correctly — this is a codegen-only gap. Spaces that use any higher-order function compile to WASM that silently produces wrong results rather than trapping.

---

## Finding 3: Record Spread Codegen Is a No-Op

**Severity:** CRITICAL
**Spec source:** grammar.md (record spread syntax `{ ...base, field: val }`)
**Code location:** `pepl/crates/pepl-codegen/src/expr.rs` (lines 185–187)

Record spread entries are silently ignored in WASM codegen:

```rust
RecordEntry::Spread(_spread_expr) => {
    // Spread requires copying all fields from the source record
    // into the target. For now, emit nothing (TODO).
}
```

The `field_count` variable only counts `Field` entries, so spread fields don't contribute to the record at all. A record literal `{ ...old_record, name: "new" }` would produce a record with only the `name` field.

**Impact:** The LLM Generation Contract (llm-generation-contract.md) explicitly instructs LLMs to "use record spread for immutable updates — `{ ...old, field: new_value }`" (Generation Pattern #14). Every LLM-generated space using this pattern would produce silently incorrect WASM.

The interpreter handles record spread correctly.

---

## Finding 4: Result Unwrap (`?` operator) Codegen Is a Pass-Through

**Severity:** HIGH
**Spec source:** grammar.md, llm-generation-contract.md (Generation Pattern #15)
**Code location:** `pepl/crates/pepl-codegen/src/expr.rs` (lines 559–564)

```rust
fn emit_result_unwrap(...) {
    // `expr?` — if the result is an error variant, trap; otherwise unwrap.
    // For now, just pass through (the type checker validates this).
    emit_expr(inner, ctx, f)?;
    // TODO: check if variant tag is "Err" and trap
    Ok(())
}
```

The `?` operator, which the LLM Generation Contract tells LLMs to use for Result chaining (Pattern #15: `let data = http.get(url)?`), does not actually unwrap or trap in WASM. It passes the full Result value through, meaning subsequent code receives a Result where it expects an unwrapped value.

---

## Finding 5: Missing `env.get_timestamp` Host Import

**Severity:** HIGH
**Spec source:** host-integration.md
**Code location:** `pepl/crates/pepl-codegen/src/lib.rs`

The spec defines **4 host imports**:

| Spec Import | Implemented? |
|---|---|
| `env.host_call` | Yes |
| `env.get_timestamp` | **No** |
| `env.log` | Yes |
| `env.trap` | Yes |

`env.get_timestamp` is specified to return `i64` (milliseconds, deterministic, host-controlled, never system clock). The implementation routes `time.now()` through the generic `env.host_call` dispatch instead. This works functionally but diverges from the spec's explicit import contract.

---

## Finding 6: Missing `dealloc` Export

**Severity:** MEDIUM
**Spec source:** host-integration.md
**Code location:** `pepl/crates/pepl-codegen/src/lib.rs`

The spec defines **8 WASM exports**. The implementation provides 7:

| Spec Export | Implemented? |
|---|---|
| `init` | Yes |
| `dispatch_action` | Yes |
| `render` | Yes |
| `get_state` | Yes |
| `update` | Yes (conditional) |
| `handle_event` | Yes (conditional) |
| `alloc` | Yes |
| `dealloc` | **No** |

The implementation uses a bump allocator with no deallocation. The spec says the WASM module exports both `alloc` and `dealloc` for the host to manage memory. Without `dealloc`, the host cannot free memory it allocated via `alloc`, and long-running spaces will leak linear memory.

---

## Finding 7: Error Codes Defined But Not Wired

**Severity:** MEDIUM
**Spec source:** compiler.md (Error Code Ranges)
**Code location:** `pepl/crates/pepl-types/src/error.rs`, `pepl/crates/pepl-compiler/src/checker.rs`

25 error codes are defined in the error module. Cross-referencing against the type checker shows several are defined but not emitted by any checker pass:

| Code | Constant | Status |
|------|----------|--------|
| E301 | `INVARIANT_UNKNOWN_FIELD` | Defined, not emitted |
| E401 | `CAPABILITY_UNAVAILABLE` | Defined, not emitted |
| E600 | `BLOCK_ORDERING_VIOLATED` | Defined, not emitted |
| E602 | `EXPRESSION_BODY_LAMBDA` | Defined, not emitted |
| E603 | `BLOCK_COMMENT_USED` | Defined, not emitted |
| E604 | `UNDECLARED_CREDENTIAL` | Defined, not emitted |
| E606 | `EMPTY_STATE_BLOCK` | Defined, not emitted |
| E607 | `STRUCTURAL_LIMIT_EXCEEDED` | Defined, not emitted |

These represent spec-required checks that don't yet have checker logic. Notably:
- **E600 (block ordering)**: The spec mandates enforced ordering `types→state→capabilities→credentials→derived→invariants→actions→views`. If the parser enforces this structurally, no checker error is needed — but the spec assigns it a code.
- **E602 (expression-body lambda)**: `fn(x) -> x * 2` should be rejected (PEPL requires block body `fn(x) { x * 2 }`). If the parser rejects this syntax, no checker code is needed.
- **E603 (block comments)**: `/* */` should be rejected. If the lexer rejects this, no checker code is needed.

Whether these are actual gaps or correctly handled at an earlier pipeline stage (lexer/parser) needs per-code verification.

---

## Finding 8: Parser Strategy Divergence

**Severity:** LOW (non-blocking, design choice)
**Spec source:** grammar.md (Recommended Libraries)

The spec recommends **tree-sitter** as the parser generator:
- MIT licensed
- Incremental parsing with error recovery
- Generates WASM parser for browser + native parser from the same grammar

The implementation uses a **hand-written recursive-descent parser** in `pepl-parser`. This is a valid engineering choice (many production compilers use hand-written parsers for better error messages and control). However, the spec's incremental parsing benefit — re-parsing only changed subtrees during Evolve operations — is not available with the hand-written approach.

---

## Finding 9: Eval ↔ Codegen Parity Not Tested

**Severity:** MEDIUM
**Spec source:** (implicit — both backends must produce identical results)

The interpreter (`pepl-eval`, 498 tests) and the WASM code generator (`pepl-codegen`) are two execution backends for the same language. No test harness runs the same PEPL programs through both backends and compares results.

Given Finding 2 (lambda placeholder), Finding 3 (record spread no-op), and Finding 4 (Result unwrap pass-through), the two backends currently produce **divergent results** for any program using these features. A parity test harness would catch these and future divergences automatically.

---

## Finding 10: `dispatch_action` Signature Divergence

**Severity:** MEDIUM
**Spec source:** host-integration.md
**Code location:** `pepl/crates/pepl-codegen/src/lib.rs`

Spec signature: `dispatch_action(action_id: i32, payload_ptr: i32, payload_len: i32) -> void`
Implementation signature: `dispatch_action(action_id: i32, args_ptr: i32) -> result_ptr: i32`

Two differences:
1. Spec uses `payload_ptr` + `payload_len` (pointer + length pair). Implementation uses single `args_ptr` (presumably null-terminated or length-prefixed in memory).
2. Spec returns `void`. Implementation returns `result_ptr: i32` (pointer to action result).

---

## Finding 11: `init` Export Signature Divergence

**Severity:** LOW
**Spec source:** host-integration.md
**Code location:** `pepl/crates/pepl-codegen/src/lib.rs`

Spec signature: `init() -> void`
Implementation signature: `init(gas_limit: i32) -> void`

The implementation takes a `gas_limit` parameter. The spec defines `init` as parameterless. Gas limit configuration may need a separate mechanism or the spec needs updating.

---

## Finding 12: UI Component Prop Coverage

**Severity:** LOW (needs deeper audit)
**Spec source:** ui-components.md
**Code location:** `pepl-ui/src/lib.rs`, `pepl-ui/src/components/`

All 10 Phase 0 components are implemented: `Text`, `Button`, `TextInput`, `Column`, `Row`, `Scroll`, `ScrollList`, `ProgressBar`, `Modal`, `Toast`.

A detailed prop-by-prop audit against the spec's prop tables has not been performed. Known areas to verify:
- `ScrollList` `render` and `key` callback props (depends on lambda codegen — Finding 2)
- `TextInput` keyboard type variants (`"text"`, `"number"`, `"email"`, `"phone"`, `"url"`)
- `Button` variant prop (`"filled"`, `"outlined"`, `"text"`)
- `Toast` auto-dismiss timing (spec says 3 seconds default)
- Color type handling for theming props

---

## Finding 13: Accessibility Spec Compliance

**Severity:** LOW (needs deeper audit)
**Spec source:** accessibility.md
**Code location:** `pepl-ui/`

A separate `accessibility.md` spec exists defining requirements for PEPL UI components. The `pepl-ui` crate has `interactive_tests.rs` which tests some accessibility patterns. A full audit against `accessibility.md` requirements has not been performed.

Areas to verify:
- ARIA role mapping for each component
- Focus management and tab order
- Screen reader announcements for dynamic content (Toast, Modal)
- Keyboard navigation patterns
- Color contrast validation

---

## Finding 14: Spec-Defined Performance Targets Not Measured

**Severity:** LOW
**Spec source:** compiler.md, llm-generation-contract.md

The specs reference several performance expectations:
- Compilation should be fast enough to run in a Web Worker without blocking UI
- PEPL context budget: ~4K tokens for LLM prompt injection
- Gas metering prevents infinite loops

No benchmarks exist in any repo measuring:
- Compilation time for canonical examples
- WASM binary size for canonical examples
- Runtime memory usage for canonical examples
- Gas consumption for canonical examples

The `pepl-compiler` pipeline tests check that "compilation completes" but do not assert timing or size budgets.

---

## Finding 15: Keyword Count Discrepancy

**Severity:** INFORMATIONAL
**Spec source:** grammar.md, llm-generation-contract.md

| Source | Keyword Count |
|---|---|
| llm-generation-contract.md | "39 reserved words" |
| grammar.md keyword table | 37 keywords (not counting reserved module names) |
| grammar.md with module names | 37 + 13 = 50 "reserved" identifiers |

The claim of "39 reserved words" in the LLM Generation Contract doesn't match the grammar spec's keyword tables. The discrepancy depends on whether reserved module names (`core`, `math`, `string`, `list`, etc.) are counted as "reserved words" or not.

---

## Appendix: Compliance by Spec File

| Spec File | Status | Key Findings |
|---|---|---|
| overview.md | Aligned | High-level design principles implemented |
| grammar.md | ~95% | All Phase 0 syntax present; keyword count discrepancy (F15) |
| language-structure.md | ~95% | Block ordering, types, expressions all work |
| compiler.md | ~80% | Error codes partially wired (F7), codegen gaps (F2–F4) |
| grammar-edge-cases.md | ~90% | Edge cases tested in parser/lexer suites |
| phase-0-subset.md | ~85% | Core subset complete; list.drop missing (F1d) |
| execution-semantics.md | ~85% | Eval correct; codegen diverges for lambdas/spread/unwrap |
| capability-system.md | ~85% | All capability modules present; storage.remove extra (F1a) |
| host-integration.md | ~75% | Missing get_timestamp (F5), dealloc (F6), signature diffs (F10–F11) |
| compilation-pipeline.md | ~80% | Pipeline works end-to-end; codegen gaps in Features |
| ui-components.md | ~90% | All 10 Phase 0 components present (F12 — deep audit pending) |
| stdlib.md | ~85% | High coverage; name alignment gaps (F1) |
| phase-0-stdlib-reference.md | ~85% | Name mismatches (F1a–F1d) |
| llm-generation-contract.md | ~80% | Canonical examples compile; auto-validation not built |
| transformation-examples.md | ~85% | Examples covered in eval tests |
| project-metadata.md | Aligned | Metadata structure defined |
| future.md | N/A | Future phases — not built yet |
| accessibility.md | ~70% | Needs dedicated audit (F13) |

---

## Pass 2: Completeness Audit — PEPL for Web and Beyond

Second-pass findings focused on what is **missing or incomplete** in the three PEPL repos to make Phase 0 (Web) fully functional and the architecture scalable to all future platforms (desktop, mobile, wearables). All findings are scoped to PEPL as an open-source LLM-first language — host application concerns are excluded.

**Audit date:** 2026-02-13
**Additional sources compared:** pepl.md (PEPL spec), all 3 repo ROADMAPs, ORCHESTRATION.md, all codegen source files (expr.rs, stmt.rs, runtime.rs, space.rs, gas.rs, lib.rs)

---

### Finding 16: WASM Codegen Has 12 Known Stubs/Placeholders

**Severity:** CRITICAL (aggregate — multiple sub-findings)
**Location:** `pepl/crates/pepl-codegen/`

The WASM code generator has 12 documented TODO/placeholder implementations. Several were marked as "done" in the roadmap but are actually stubs. Complete inventory:

| # | File | What | Severity | Effect |
|---|---|---|---|---|
| 1 | `expr.rs` L59–61 | Lambda closures → emit `nil` | Critical | All higher-order stdlib calls broken (`list.map`, `list.filter`, `list.sort`, `list.reduce`, `list.find`, `list.every`, `list.some`, `list.count`, `list.find_index`, `list.flat_map`, `list.sort_by`) |
| 2 | `expr.rs` L192 | Record spread `...expr` → emit nothing | Critical | Immutable record updates silently drop all existing fields |
| 3 | `expr.rs` L548–550 | `?` operator → pass-through (no unwrap/trap) | High | Result values not unwrapped; downstream code receives Result instead of inner value |
| 4 | `stmt.rs` L164 | Nested `set a.b = x` → loses sibling fields | High | 2-level nested set creates inner record with only the target field |
| 5 | `stmt.rs` L191 | `set a.b.c = x` → falls back to root | High | 3+ level nesting loses all intermediate fields |
| 6 | `runtime.rs` L449 | String `==`/`!=` → pointer comparison | Critical | Dynamic strings with same content compare as not-equal |
| 7 | `runtime.rs` L875 | Record key lookup → pointer comparison | Critical | `record.get(r, "key")` fails for non-interned key strings |
| 8 | `runtime.rs` L145 | Bump allocator → no `memory.grow` | Medium | OOM trap on larger programs without memory expansion |
| 9 | `runtime.rs` L541 | `to_string` for records/lists → `"[value]"` | Low | Debug/display output is a placeholder |
| 10 | `space.rs` L612–621 | Derived recomputation → `NilLit` placeholder | Medium | Derived fields may not reflect computed values in WASM |
| 11 | — | No `dealloc` export (bump allocator only) | Medium | Long-running spaces leak linear memory |
| 12 | — | No `env.get_timestamp` import | Medium | `time.now()` routed through `host_call` instead of dedicated import |

Items 1, 2, 6, and 7 are **showstoppers** for runtime correctness — any space using lambdas, record spread, or dynamic string comparisons will produce silently wrong results in WASM.

---

### Finding 17: Roadmap Items Explicitly Left Unchecked

**Severity:** HIGH
**Location:** `pepl/ROADMAP.md`

6 items in the pepl repo roadmap were explicitly left unchecked (not done):

| Roadmap Item | Impact |
|---|---|
| `any` type runtime checks on state assignment (evaluator) | No runtime type validation when `any`-typed values (e.g., from `json.parse`) are assigned to typed state fields |
| Generate `?` postfix (Result unwrap, trap on Err) in WASM | `?` operator is dead in WASM output |
| Generate `any` type runtime checks in WASM codegen | Same as evaluator — no runtime type guards |
| Generate capability call suspension/resume (yield to host, resume with Result) | WASM cannot actually suspend/resume for async capability calls (`http.get`, `storage.get`, etc.) |
| Generate test execution codegen | Tests cannot be compiled to WASM — evaluator-only |
| Generate `with_responses` mock capability dispatch for test blocks | Test mocking not available in WASM |

---

### Finding 18: AST Diff Infrastructure Does Not Exist

**Severity:** HIGH (required for Evolve operations)
**Spec source:** compilation-pipeline.md (Incremental Compilation), pepl.md § Transformation guarantee
**Location:** `pepl/` (all crates searched)

Zero AST diff infrastructure exists. This is required for:

1. **Evolve operation scope validation**: Comparing old AST vs. new AST to ensure changes are scoped to the user's request. The host defines a complete AST diff algorithm (walk both ASTs in parallel, accept in-scope changes, reject out-of-scope changes). PEPL must provide the AST comparison primitives.

2. **Incremental compilation**: The spec describes re-type-checking and re-codegen of only changed subtrees for performance.

3. **Event storage optimization**: Storing AST diffs (~0.5–5 KB) instead of full source snapshots (~5–50 KB) per evolve event.

4. **PEPL's Transformation guarantee** (pepl.md): "Transformations produce meaningful diffs (AST-level, not line-level)."

The compiler currently does fresh full compilation every time. No AST caching, no diff computation, no scope validation.

---

### Finding 19: CompileResult Lacks Essential Metadata

**Severity:** HIGH
**Spec source:** pepl.md, host-integration.md
**Location:** `pepl/crates/pepl-compiler/src/lib.rs`

`CompileResult` currently contains only:

```
{ success: bool, wasm: Option<Vec<u8>>, errors: CompileErrors }
```

Missing fields needed by any host application:

| Missing Field | Why Needed |
|---|---|
| Full AST | Evolve scope validation, AST diff computation, semantic diff viewer |
| Source hash (SHA-256) | Determinism verification, event log integrity, `.space` file format |
| WASM hash (SHA-256) | Tamper detection, deployment verification, determinism proof |
| State field list (names + types) | Host needs to know space's state shape for persistence, event log, UI |
| Action list (names + parameter types) | Host needs to map UI events to action IDs, build dispatch table |
| View list (names) | Host needs to know which views exist for rendering |
| Declared capabilities | Host needs to know what capabilities to grant at instantiation |
| Declared credentials | Host needs to prompt user for credential values before space runs |
| PEPL language version | Version compatibility checks, migration detection |
| Compiler version | Debugging, reproducibility |
| Warnings | Spec reserves warnings for Phase 1+ but the field should exist (empty list) |

The compiler internally collects `state_field_names`, `action_names`, `view_names`, and `variant_ids` in its `Compiler` struct but does not surface them in `CompileResult`.

---

### Finding 20: Capability Call Suspension/Resume Not Implemented

**Severity:** HIGH
**Spec source:** execution-semantics.md, capability-system.md
**Location:** `pepl/crates/pepl-codegen/`

The spec defines capability calls as appearing synchronous in PEPL but executing asynchronously in the host:

```
PEPL calls http.get(url) → execution suspends
  → Host performs async I/O
  → Host resumes PEPL execution with Result value
```

The WASM code generator has no suspension/resume mechanism. Capability calls are dispatched via `env.host_call` but the WASM execution cannot actually pause and resume. This means:

- `http.get()`, `storage.get()`, `location.current()`, `notifications.send()` — all capability functions — cannot work as specified in WASM
- The evaluator handles this correctly (it can suspend Rust execution)
- WASM requires either: (a) Asyncify transform, (b) WASM component model await, (c) JavaScript wrapper that re-enters WASM, or (d) splitting execution at yield points

This is a fundamental architecture gap for any space that uses I/O capabilities in WASM.

---

### Finding 21: No Source Mapping (WASM Offset → PEPL Source)

**Severity:** MEDIUM
**Location:** `pepl/crates/pepl-codegen/`

No source map is generated during compilation. When a WASM trap occurs at runtime (gas exhaustion, assertion failure, division by zero), the host receives a WASM instruction offset but cannot map it back to a PEPL source line/column.

Impact:
- Runtime error messages cannot show "error at line 15, column 3 of your space"
- Debugging WASM execution is opaque
- The spec requires every error to include "exact source line for context" — this works for compile-time errors but not for runtime errors

---

### Finding 22: No Space Migration Infrastructure

**Severity:** MEDIUM (not needed until PEPL versions diverge)
**Spec source:** pepl.md § Versioning
**Location:** `pepl/` (all crates searched)

No migration logic exists for upgrading spaces from one PEPL version to another. The spec defines a 5-step migration protocol (DETECT → ANALYZE → GENERATE → PREVIEW → APPLY) with rollback. Since PEPL is at v0.1.0 with no prior versions, this is not blocking — but the infrastructure should be designed for scalability.

---

### Finding 23: Test Codegen Does Not Exist

**Severity:** MEDIUM
**Spec source:** execution-semantics.md § Test Execution Semantics
**Location:** `pepl/crates/pepl-codegen/`

PEPL `test` blocks can only be executed via the evaluator (`pepl-eval`). They cannot be compiled to WASM. For a fully web-based system where the compiler runs as WASM in a Web Worker, test execution requires either:

- Compiling tests to WASM (not implemented)
- Shipping the evaluator as WASM alongside the compiler (adds bundle size)
- Running tests server-side (breaks offline-first)

The spec says test execution should happen in the WASM sandbox with a 5-second timeout per test.

---

### Finding 24: `pepl-ui` Has No DOM Renderer Contract

**Severity:** MEDIUM (host responsibility, but contract needed)
**Spec source:** pepl.md § What It Builds, ui-components.md
**Location:** `pepl-ui/`

`pepl-ui` produces abstract `Surface` / `SurfaceNode` trees — platform-independent data structures with `component_type`, `props`, and `children`. This is correct by design ("Outside these repos: How UI components are rendered to DOM").

However, no formal **mapping specification** exists defining how each component type maps to DOM elements. The spec lists:

| PEPL Surface | Browser Element |
|---|---|
| `Button` | `<button>` with click handler |
| `TextInput` | `<input>` / `<textarea>` |
| `Scroll` | `overflow: auto` CSS |
| `Modal` | Fixed overlay `<div>` |
| `Toast` | Positioned notification `<div>` |
| `ProgressBar` | `<progress>` or styled `<div>` |

Defining a **reference DOM mapping** within `pepl-ui` or as a companion document would ensure consistent rendering across host implementations and enable LLMs to reason about the rendered output.

---

### Finding 25: Determinism Proof Infrastructure Missing

**Severity:** HIGH
**Location:** All repos

The spec requires a determinism proof: "Run space twice with identical inputs → assert byte-for-byte identical outputs. Failure is a critical bug."

No infrastructure exists to:
1. Execute a compiled space with known inputs
2. Capture the complete output (state + view tree)
3. Execute a second time with identical inputs
4. Compare outputs byte-for-byte

The eval↔codegen parity gap (Finding 9) makes this worse — even if a determinism test harness existed, the two backends would fail it for lambda/spread/unwrap cases.

---

### Finding 26: No `.space` File Format Implementation

**Severity:** LOW (host concern, but PEPL compiler should produce the data)
**Spec source:** pepl.md
**Location:** `pepl/crates/pepl-compiler/`

The spec defines a `.space` file format: JSON bundle containing PEPL source + source hash + compiled WASM + WASM hash + optional state snapshot + optional event history. The compiler should produce at minimum the source hash and WASM hash (see Finding 19). The bundling itself is host-side, but the hashes must come from the compiler.

---

### Finding 27: No Structured Error Suggestion System

**Severity:** MEDIUM
**Spec source:** compiler.md (Error Message Format)
**Location:** `pepl/crates/pepl-compiler/`

The spec requires every error to include a `suggestion` field: "Every error includes a suggestion when one exists (LLM-generated code can be auto-fixed by re-prompting with the error)."

Current error structs have `message` but need verification that all include actionable `suggestion` strings. Suggestions like "Use convert.to_int(value) to convert String to Number" or "Suggest adding to capabilities" are specified — but whether they are consistently generated for every error code has not been audited.

The LLM repair prompt template relies on these suggestions to guide fixes.

---

### Finding 28: No Mandatory WASM Validation Pass

**Severity:** MEDIUM
**Spec source:** compiler.md (Recommended Libraries: wasmparser)
**Location:** `pepl/crates/pepl-codegen/`

The spec recommends using `wasmparser` (Bytecode Alliance) to validate generated WASM before returning it. While `wasmparser` is a dependency, it is unclear whether every `compile()` call passes the output through WASM validation. Invalid WASM would cause `WebAssembly.instantiate()` to fail in the browser with an opaque error instead of a structured compiler error (code E702).

The pipeline integration tests (Step 22) do validate WASM output, but this should be a mandatory compiler pipeline step, not just a test.

---

### Finding 29: Stdlib Function Count Discrepancy

**Severity:** LOW
**Location:** `pepl/crates/pepl-compiler/src/stdlib.rs`, `pepl-stdlib/src/`

Multiple sources disagree on the stdlib function count:

| Source | Count |
|---|---|
| Spec header claim | 88 Phase 0 functions |
| Type checker (stdlib.rs) | 90 functions + 2 constants |
| pepl-stdlib actual | ~88 functions + 2 constants |
| Phase 0 stdlib reference tables (summed) | 4 + 12 + 20 + 31 + 5 + 5 + 5 + 2 + 4 = **88** + 2 constants |

The type checker registers extra functions not in the spec (`storage.remove`, `list.insert`, `list.update`, `list.find_index`, `list.zip`, `list.flatten`). These are valid utility functions but they diverge from the Phase 0 spec boundary.

---

### Finding 30: Gas Metering Granularity

**Severity:** LOW
**Spec source:** compiler.md
**Location:** `pepl/crates/pepl-codegen/src/gas.rs`

Gas metering is implemented at **per-loop-iteration + per-call** granularity. The spec mentions "gas metering at loop/call boundaries" which matches. However:

- No per-allocation metering (memory use isn't gas-counted)
- No per-WASM-instruction metering (optional `wasm-metering` library mentioned in spec but not used)
- Gas limit is hardcoded to 1,000,000 in the codegen; host can override via `init(gas_limit)` but there's no spec-defined default

Current implementation is adequate for Phase 0. Finer-grained metering is a Phase 1+ concern.

---

### Finding 31: `color` Module Not in Stdlib

**Severity:** LOW
**Spec source:** pepl.md § What It Builds
**Location:** `pepl-stdlib/src/modules/`

The pepl.md spec lists `color` as a core stdlib module: "Core modules (always available): `core`, `math`, `string`, `list`, `record`, `time`, `convert`, `json`, `color`". However:

- No `color` module exists in pepl-stdlib
- No `color` module is registered in the type checker
- Phase 0 uses `color` as a **type** (hex string literals like `"#FF5733"`) but has no programmatic color functions
- Programmatic `color` module functions (`color.rgb`, `color.lighten`, etc.) are deferred beyond Phase 0

The type exists, the module doesn't. This is consistent with Phase 0 scoping but diverges from pepl.md's "always available" claim.

---

### Finding 32: No Compressed PEPL Reference for LLM Prompts

**Severity:** MEDIUM (needed for LLM-first language)
**Spec source:** llm-generation-contract.md § LLM Context Injection Protocol
**Location:** All repos

The LLM Generation Contract specifies that every prompt must inject a "Compressed PEPL Reference (~2K tokens)" and a "Phase 0 Stdlib Table." These are **documentation artifacts** that should be:

1. Machine-generated from the compiler's type registry (so they stay in sync)
2. Versioned alongside the compiler
3. Available as an export from the `pepl-wasm` crate (e.g., `get_reference() -> String`, `get_stdlib_table() -> String`)

Currently, these references exist only as spec markdown files. If the stdlib changes (function added/renamed), the LLM reference would go stale unless manually updated. For an LLM-first language, the compiler should be the source of truth for its own reference material.

---

### Finding 33: No Static Invariant Checker

**Severity:** MEDIUM
**Spec source:** compilation-pipeline.md, compiler.md
**Location:** `pepl/crates/pepl-compiler/`

The compilation pipeline specifies 4 stages: Lexer → Parser → Type Checker → **Invariant Checker** → WASM Codegen. The type checker covers type validation, but the invariant checker — which should:

- Verify that actions cannot provably violate declared invariants
- Check bounds statically when possible
- Enforce recursion depth limits
- Perform termination analysis heuristics

— is not a separate pass. Invariant validation currently occurs as part of the type checker (checking that invariant expressions reference valid fields and have boolean type). The deeper analysis (can an action provably violate an invariant?) is not implemented.

Runtime invariant checking (post-action verification) IS implemented in both evaluator and codegen.

---

### Finding 34: Scalability Gaps — Features Needed for Future Phases

**Severity:** INFORMATIONAL (not Phase 0 blockers but architecture should not preclude them)

Features that PEPL's architecture must not preclude for future phases:

| Feature | Phase | What PEPL Needs |
|---|---|---|
| Native binary target (CLI, desktop) | 1 | Compiler already in Rust — add native binary build target. No PEPL language change needed. |
| CLI tool (`pepl compile/validate/test`) | 1 | Thin CLI wrapper over existing compiler API. No language change needed. |
| VS Code extension | 1 | tree-sitter grammar (currently using hand-written parser — F8). Would need separate grammar file or LSP server using existing parser. |
| Native SDK with FFI bindings | 2+ | Compiler API must be C-ABI-safe. Current Rust API is fine. |
| Canvas/Chart components | 0B+ | `pepl-ui` component definitions only — rendering is host-side. |
| Animation system | 0C+ | Need `animate()` function and interpolation primitives in stdlib. |
| Incremental compilation | 1+ | Needs AST caching + diff infrastructure (F18). |
| Space publishing (static site) | 1+ | Compiler needs to produce standalone WASM + thin runtime bundle. |

The current architecture (Rust compiler → WASM/native, abstract stdlib, abstract UI surfaces) is well-positioned for all of these. No fundamental redesign is needed — the gaps are additive features.

---

## Pass 2 Summary

| Category | Findings | Critical | High | Medium | Low |
|---|---|---|---|---|---|
| WASM Codegen Correctness | F16, F20 | 1 | 1 | — | — |
| Roadmap vs. Built | F17 | — | 1 | — | — |
| Missing Infrastructure | F18, F19, F21, F22, F23, F25 | — | 3 | 3 | — |
| Spec Compliance Gaps | F24, F27, F28, F29, F31, F33 | — | — | 4 | 2 |
| LLM-First Readiness | F32 | — | — | 1 | — |
| Scalability | F34 | — | — | — | 1 (info) |
| Performance | F30 | — | — | — | 1 |
| **Totals** | **19 findings** | **1** | **5** | **8** | **4 + 1 info** |

### Cumulative Finding Count: F1–F34 (15 from Pass 1 + 19 from Pass 2)

### Updated Compliance Summary (Pass 1 + Pass 2)

| Area | Spec Compliance | Web Readiness | Scalability |
|------|:-:|:-:|:-:|
| Grammar / Syntax | ~95% | Ready | Ready |
| Type System | ~90% | Ready | Ready |
| Stdlib (pure modules) | ~85% | Ready | Ready |
| Stdlib (capability modules) | ~85% | Blocked (F20) | Ready |
| WASM Codegen | ~50% | Blocked (F16) | Needs work |
| UI Components (abstract) | ~90% | Ready | Ready |
| Error Codes / Messages | ~75% | Partial (F27) | Ready |
| Host Integration | ~60% | Blocked (F5, F6, F19, F20) | Needs work |
| Compiler Metadata | ~40% | Blocked (F19) | Needs work |
| AST Infrastructure | ~10% | Missing (F18) | Missing |
| Testing Pipeline | ~60% | Partial (F23, F25) | Needs work |
| LLM-First Tooling | ~50% | Missing (F32) | Needs work |
