---
name: debugging
description: Use when investigating bugs, test failures, unexpected behavior, or compile errors in any PEPL component. Provides systematic debugging procedures for the compiler pipeline (parser, type checker, invariant checker, WASM codegen) and stdlib.
---

# Debugging

## When to Use

- A test fails unexpectedly
- `cargo build` or `cargo test` produces errors
- Compiler output doesn't match expected WASM
- Behavior doesn't match the spec `.md` files
- User reports a bug

## Context

PEPL's determinism guarantee means bugs are especially serious — non-determinism anywhere breaks the core promise. Debugging must be systematic, not guess-and-check.

## Procedure

1. **Reproduce** — Write the minimal PEPL source that triggers the bug
2. **Read the error** — Full error message, stack trace, line numbers
3. **Locate** — Which pipeline stage? Parser → Type Checker → Invariant Checker → Codegen?
4. **Spec check** — Read the relevant spec file in `docs/`. Is the code wrong or the expectation wrong?
5. **Isolate** — Write a failing unit test that captures exactly the bug
6. **Diagnose** — Use binary search:
   - Does the parser produce correct AST? → If no, parser bug
   - Does the type checker accept/reject correctly? → If no, type checker bug
   - Does the invariant checker validate correctly? → If no, invariant checker bug
   - Does the codegen produce correct WASM? → If no, codegen bug
   - Does the stdlib function compute correctly? → If no, stdlib bug
7. **Fix** — Make the minimal change that fixes the failing test
8. **Verify** — Run the full test suite, not just the fixed test
9. **Determinism** — Run 100-iteration determinism test on the fixed code
10. **Document** — Add a comment explaining WHY the bug existed and how the fix works

## Debugging Pipeline

```
Bug Report / Failure
       ↓
Reproduce (minimal PEPL source)
       ↓
Isolate (which pipeline stage?)
       ↓
    ┌─────────────────────────────────────────────┐
    │ Parser → Type Checker → Invariant → Codegen │
    │   ↓          ↓            ↓          ↓      │
    │ Wrong AST  Wrong types  Wrong check  Wrong WASM │
    └─────────────────────────────────────────────┘
       ↓
Write failing test
       ↓
Minimal fix
       ↓
Full test suite + determinism proof
```

## Common PEPL Bug Categories

| Symptom | Likely cause |
|---------|-------------|
| Parse error on valid PEPL | Lexer missing a keyword or operator |
| Wrong operator precedence | Parser binding power table incorrect |
| Type error on valid code | Type checker missing a rule from the spec (see `docs/execution-semantics.md`, `docs/language-structure.md`) |
| `set a.b.c = x` fails | Nested set desugaring bug in codegen |
| NaN in output | Missing trap guard in division/sqrt codegen |
| Different WASM on same source | Non-deterministic BTreeMap vs HashMap |
| Gas metering not applied | Loop/call boundary detection missed |
| Stdlib returns wrong value | Edge case in function implementation |

## Rules

- **Never fix without a failing test first** — write the test, then fix
- **Minimal fix only** — don't refactor while debugging
- **Check spec before assuming code is wrong** — maybe the test expectation is wrong
- **Determinism check after every fix** — a fix that breaks determinism is not a fix
- **No temporary workarounds without tracking issue**
