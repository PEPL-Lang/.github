# Project Metadata

## Future: CLI Targets

> **Phase 1+ — Not in Phase 0 scope. The compiler is consumed as a WASM module by host applications in Phase 0.**

| Command | What It Does | Exit Code |
|---|---|---|
| `pepl compile <file.pepl>` | Parse → type-check → emit `.wasm` binary | 0 success, 1 errors |
| `pepl validate <file.pepl>` | Parse → type-check only (no WASM emission) | 0 valid, 1 errors |
| `pepl test <file.pepl>` | Compile → execute `test` blocks in sandbox → report pass/fail | 0 all pass, 1 failures |

Not a separate repo. The CLI is an additional binary target of the compiler crate — same code, different entry point. Output uses the same structured error format defined in Compiler Error Message Format.

---

## Dependencies

PEPL is a standalone language specification with no upstream dependencies.

| Dependency | Relationship |
|---|---|
| WASM specification | Compilation target — PEPL compiles to standard WASM. No custom extensions. |
| Unicode standard | String encoding — PEPL strings are UTF-8. |
| IEEE 754 | Number representation — PEPL `number` is 64-bit double. |

All dependencies are stable international standards. None are software packages.

## Dependents

| Dependent | What It Uses From PEPL |
|---|---|
| Host applications | Embed PEPL compiler (WASM module), execute compiled spaces, render view trees |
| pepl-playground | Editor + live execution environment for PEPL spaces |
| Any third-party application | Can embed PEPL via the WASM import/export contract |
| LLM code generators | Generate PEPL source from natural language using the LLM Generation Contract |

---

## Success Criteria

### Language
- Type system catches >95% of errors at compile time
- Any space concept expressible in PEPL (games, trackers, dashboards, tools)
- Compilation < 2s for typical spaces
- Determinism verified through repeated execution

### Stdlib
- All Phase 0 stdlib functions (100 functions + 2 constants) implemented and tested
- Zero breaking changes after 1.0 release
- All functions execute in < 1ms

### UI
- 10 Phase 0 components render correctly on all target platforms
- Zero spaces need to reimplement stdlib functionality
- Full accessibility: screen reader, keyboard navigation, high contrast

### Transformation
- Given the canonical example prompts, LLMs generate compilable PEPL ≥12/13 on first attempt
- Diffs are meaningful (AST-level, not line-noise)
- Any space → Any space remains possible

---

## Definition of Done

This canvas is considered complete when:

- [x] Phase 0 PEPL subset (~140 constructs) fully defined with grammar
- [x] Thin transpiler compiles subset to valid WASM bytecode
- [x] Type checker catches all errors defined in E100–E699 taxonomy at compile time
- [x] Gas metering injected at all loop/call boundaries
- [x] Invariant checker verifies state postconditions
- [x] All stdlib functions in Phase 0 subset implemented and tested
- [x] UI components (10 core) render correctly via the host application's View Layer
- [x] Determinism verified: same source + same inputs = same output
- [ ] Code generators produce valid PEPL subset: ≥12/13 canonical examples compile on first attempt — *deferred to host integration*
- [ ] Compilation < 2s for typical spaces in browser Web Worker — *deferred to host integration*
- [ ] WASM import/export contract validated with at least one host implementation — *deferred to host integration*
- [ ] LLM Generation Contract tested: given canonical example prompts, LLM produces valid PEPL ≥12/13 on first attempt — *deferred to host integration*
- [x] All canonical examples compile and execute correctly
- [x] Credential management documented and testable
