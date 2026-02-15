# PEPL Phases — Progress & Roadmap

> Comprehensive tracking document for the PEPL compiler ecosystem.
> Three sections: what's built, what the audit found, and what remains.

**Last updated:** 2026-02-15
**Total tests passing:** 1,499 (588 pepl + 512 pepl-stdlib + 379 pepl-ui + 20 pepl-playground)

---

## What's Built (27 ORCHESTRATION steps, all complete)

### pepl/ (7 crates, 588 tests)

- [x] **Lexer** — full tokenization, string interpolation, all 39 keywords
- [x] **Parser** — hand-written recursive descent, all Phase 0 syntax
- [x] **Type system** — number, string, bool, nil, color, list\<T>, records, sum types, Result, Surface
- [x] **Type checker** — 23 error codes all wired, stdlib registry (100 functions + 2 constants), suggestions on all key errors
- [x] **Invariant checker** — E300 (derived field reference), E502 (recursion detection), structural limits
- [x] **Evaluator** — full interpreter with correct lambda, spread, capability, test execution
- [x] **WASM codegen** — compiles to `.wasm` via `wasm-encoder`, gas metering, NaN guards, memory.grow, 4 host imports, dealloc export
- [x] **WASM bindings** — `pepl-wasm` crate with `wasm-bindgen` (browser-ready)
- [x] **AST diff** — `ast_diff.rs` with AstDiff/AstChange types, JSON serialization, scope validation (14 tests)
- [x] **Determinism & parity** — wasmi-based harness, compilation + execution determinism, eval↔codegen state parity (11 tests)
- [x] **Test codegen** — `test_codegen.rs`, test blocks compile to `__test_N()` WASM exports, assert → trap (16 tests)
- [x] **Source mapping** — `source_map.rs`, WASM offset → PEPL source position, embedded in custom section + CompileResult
- [x] **LLM reference generator** — `reference.rs`, machine-generated compressed reference (~2K tokens) and JSON stdlib table from StdlibRegistry (14 tests)
- [x] **Contextual keywords** — keywords usable as identifiers in non-ambiguous positions, improved error recovery (11 new parser tests)

### pepl-stdlib/ (13 modules, 512 tests)

- [x] **9 pure modules:** core, math, string, list, record, time, convert, json, timer
- [x] **4 capability modules:** http, storage, location, notifications
- [x] **100 functions + 2 constants**, all tested

### pepl-ui/ (10 components, 379 tests)

- [x] **Text, Button, TextInput, Column, Row, Scroll, ScrollList, ProgressBar, Modal, Toast**
- [x] Typed props, builders, accessibility basics, abstract Surface tree output

### Milestones

- [x] **M1** — Parse all canonical examples (220 parser tests)
- [x] **M2** — Type-check & validate all canonical examples
- [x] **M3** — Evaluate all canonical examples (1,142 tests)
- [x] **M4** — Compile & execute all canonical examples end-to-end (1,378 tests)
- [x] **M5** — WASM runtime correctness (8 parity tests, determinism verified)
- [x] **M5.5** — C11 complete: AST diff, determinism proof, eval↔codegen parity, test codegen, source maps (563 pepl tests)
- [x] **M6** — Phase 0 complete: C12 LLM reference exports, all findings resolved/deferred, determinism+parity+test gates passed (577 pepl tests)
- [x] **M6.1** — Phase 13: contextual keywords as identifiers, error recovery improvements (588 pepl tests)
- [x] **M6.2** — All 9 crates published to crates.io at v0.1.2 with READMEs

---

## What the Findings Revealed (F1–F34)

Post-build compliance audit in 2 passes. 34 findings total.

### CRITICAL (must fix for runtime correctness)

- F1: Stdlib name mismatches (`list.any` vs `list.some`, extra `storage.remove`, missing `list.drop`)
- F2/F16.1: Lambda codegen emits `nil` — all higher-order functions broken in WASM
- F3/F16.2: Record spread codegen is a no-op — `{ ...old, field: val }` drops all old fields
- F16.6: String `==`/`!=` uses pointer comparison — dynamic strings broken
- F16.7: Record key lookup uses pointer comparison — `record.get` broken for non-interned keys

### HIGH (blocks web deployment)

- F4/F16.3: `?` operator is pass-through in WASM (no unwrap/trap)
- F5: Missing `env.get_timestamp` host import
- F16.4–5: Nested `set a.b = x` loses sibling fields
- F17: 6 roadmap items explicitly unchecked (Result unwrap, capability suspend, test codegen)
- F18: Zero AST diff infrastructure (needed for incremental compilation, host-driven transformations)
- F19: `CompileResult` missing AST, hashes, state/action/view lists, capabilities, credentials
- F20: Capability calls can't suspend/resume in WASM (http, storage, etc. broken)
- F25: No determinism proof harness

### MEDIUM

- F6: Missing `dealloc` export (memory leaks)
- F7: 8 error codes defined but not wired into checker
- F9: No eval↔codegen parity tests
- F16.8: No `memory.grow` in bump allocator
- F16.10: Derived recomputation is a placeholder
- F21: No source mapping (WASM offset → PEPL line)
- F22: No migration infrastructure
- F23: Test blocks can't compile to WASM
- F24: No DOM renderer contract in `pepl-ui`
- F27: Error `suggestion` field not consistently generated
- F28: WASM validation not mandatory in compile pipeline
- F32: No machine-generated PEPL reference for LLM prompts
- F33: No static invariant checker (only runtime checking)

### LOW / INFORMATIONAL

- F8: Hand-written parser vs spec-recommended tree-sitter
- F10–11: `dispatch_action` / `init` signature divergences
- F12–13: UI prop and accessibility deep audits pending
- F14–15: No performance benchmarks, keyword count discrepancy
- F26: No `.space` file format hashes
- F29: ~~Stdlib function count discrepancy (88 vs 90)~~ — resolved: actual count is 100 functions + 2 constants across 13 modules
- F30: Gas metering adequate but no per-allocation metering
- F31: `color` module missing (type exists, module doesn't)
- F34: Scalability gaps are additive, no redesign needed

---

## What Remains (by priority)

### Phase 0A — Fix Critical WASM Codegen

Must do before any web use. The WASM backend produces silently wrong results without these.

- [x] Implement lambda/closure codegen (function table + environment capture) (F2, F16.1) — Step 24
- [x] Implement record spread codegen (copy fields from source record) (F3, F16.2) — Step 24
- [x] Implement byte-by-byte string comparison (replace pointer comparison) (F16.6) — Step 24
- [x] Implement byte-by-byte record key comparison (replace pointer comparison) (F16.7) — Step 24
- [x] Implement `?` operator codegen (unwrap Ok, trap on Err) (F4, F16.3) — Step 24
- [x] Fix nested set to preserve sibling fields (F16.4–5) — Step 24
- [x] Align stdlib names (`list.any`↔`list.some` renamed, `list.drop` added, extra functions kept as extensions) (F1) — Step 23
- [x] Fix `to_string` for records/lists — proper debug output instead of `"[value]"` (F16.9) — Step 24
- [x] Resolve keyword count discrepancy in LLM Generation Contract (F15) — *deferred: LOW severity, cosmetic doc discrepancy only; grammar.md lists 37 keywords, contract says 35; does not affect compilation or correctness; Phase 1*

### Phase 0B — Complete Compiler Metadata & Host Integration

Required for any host application to actually use the compiler.

- [x] Enrich `CompileResult` (AST, hashes, state/action/view lists, capabilities, credentials, versions) (F19) — Step 25
- [x] Add `env.get_timestamp` import, `dealloc` export (F5, F6) — Step 25
- [x] Wire all 8 unused error codes into checker (F7) — Step 25 (E301, E401, E604 wired; remaining already emitted)
- [x] Add `suggestion` field to all error messages (F27) — Step 25
- [x] Make WASM validation (`wasmparser`) a mandatory pipeline step (F28) — already present in codegen pipeline
- [x] Fix derived field recomputation in codegen (F16.10) — Step 24
- [x] Add `memory.grow` to bump allocator (F16.8) — Step 25
- [x] Align `dispatch_action` export signature with spec (3 params, void return) (F10) — Step 25
- [x] Align `init` export signature with spec (parameterless) (F11) — Step 25
- [x] Implement capability call suspension/resume (Asyncify or split-execution) (F20) — *deferred: fundamental WASM limitation; requires JS wrapper or component model await; documented for Phase 1*
- [x] Implement `any` type runtime checks on state assignment — evaluator + codegen (F17) — *deferred: type checker prevents mismatches at compile time; all values already runtime-tagged; Phase 1*

### Phase 0C — Infrastructure for Incremental Compilation & Testing

Required for incremental compilation, host-driven code transformations, and PEPL's own test infrastructure.

- [x] Build AST diff infrastructure (compare, serialize diffs) (F18) — Step 26 (`ast_diff.rs`, 14 unit tests)
- [x] Build determinism proof harness (run twice, compare byte-for-byte) (F25) — Step 26 (wasmi-based, 100-iter compilation + execution determinism)
- [x] Build eval↔codegen parity test harness (F9) — Step 26 (5 canonical sources, state comparison via WASM memory reader)
- [x] Implement test codegen (compile test blocks to WASM) (F23) — Step 26 (`test_codegen.rs`, `__test_N()` exports, assert → trap)
- [x] Implement `with_responses` mock capability dispatch for test blocks in WASM (F17) — *deferred: no WASM programs use capabilities yet; evaluator mock runner fully functional*
- [x] Add source mapping (WASM offset → PEPL source position) (F21) — Step 26 (`source_map.rs`, embedded in WASM custom section + `CompileResult`)

### Phase 0D — LLM-First Tooling

Required for PEPL's identity as an LLM-first language.

- [x] Machine-generate compressed PEPL reference from type registry (F32) — Step 27 (`reference.rs`, 14 tests)
- [x] Export `get_reference()` / `get_stdlib_table()` from `pepl-wasm` (F32) — Step 27

### Phase 1 — Desktop & CLI

- [ ] Native binary build target (already Rust — add target)
- [ ] `pepl compile/validate/test` CLI
- [ ] VS Code extension (LSP server or tree-sitter grammar)
- [ ] Incremental compilation (depends on AST diff from Phase 0C) (F18)
- [ ] Source-to-source migration tool (`pepl migrate --from v1 --to v2`) (F22)
- [ ] Deep prop-by-prop audit of all 10 UI components against spec (F12)
- [ ] Full accessibility audit against accessibility.md (F13)
- [ ] Performance benchmarks (compile time, WASM size, memory, gas) (F14)
- [ ] Define reference DOM mapping for `pepl-ui` (non-normative guide) (F24)

### Phase 2+ — Multi-Platform

- [ ] Native SDK with FFI bindings (C-ABI)
- [ ] Canvas/Chart component definitions in `pepl-ui`
- [ ] Animation system (`animate()` + interpolation in stdlib)
- [ ] Static invariant analysis (prove actions can't violate invariants) (F33)
- [ ] Standalone WASM output for embedding (no host dependency)
- [ ] `color` stdlib module — programmatic color functions (`color.rgb`, `color.lighten`, etc.) (F31)

---

## Summary

| Phase | Items | Status |
|-------|------:|--------|
| What's Built | 27 steps, 6 milestones | ✅ Complete (1,477 tests) |
| Findings | 34 (5 critical, 8 high, 13 medium, 8 low) | Documented |
| Phase 0A (Critical WASM fixes) | 9 | ✅ Complete (F15 deferred to Phase 1) |
| Phase 0B (Metadata & host integration) | 11 | ✅ Complete (F20, F17 deferred to Phase 1) |
| Phase 0C (Incremental compilation & testing) | 6 | ✅ Complete (Step 26, 563 pepl tests) |
| Phase 0D (LLM-First Tooling) | 2 | ✅ Complete (Step 27, 577 pepl tests) |
| Phase 1 (Desktop & CLI) | 9 | 🔴 Not started |
| Phase 2+ (Multi-platform) | 6 | 🔴 Not started |
| **Total remaining (Phase 1+)** | **15** | |
