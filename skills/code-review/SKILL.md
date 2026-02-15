---
name: code-review
description: Use when reviewing code changes, PRs, or validating that changes meet PEPL quality standards before committing. Covers determinism checks, spec compliance, error handling, and test coverage.
---

# Code Review

## When to Use

- Before committing any code changes
- Reviewing a PR or diff
- User asks to review code
- After implementing a feature, before marking it done

## Context

PEPL compiler output is deterministic — every bug in the compiler corrupts the determinism guarantee. The stdlib is immutable once released — every function signature is permanent. Code review is the last line of defense.

## 12-Point Checklist

1. **Spec compliance** — Does the code match the relevant spec file in `docs/`?
2. **Determinism** — Any source of non-determinism?
   - No `HashMap` (use `BTreeMap`)
   - No `rand` or randomness
   - No system time
   - No external I/O in compiler core
   - No float operations that could produce NaN without trap guards
3. **Error handling** — All `Result<T>` with proper error types? No `unwrap()` in library code?
4. **Error codes** — Compiler errors use correct E-code from the assigned range?
5. **Tests exist** — Unit test, error test, edge case test, determinism test (100 iterations)?
6. **Doc comments** — All public items documented with `///`?
7. **Naming** — Follows conventions? (PascalCase types, snake_case functions, SCREAMING_CASE constants)
8. **Minimal surface** — Is anything `pub` that shouldn't be? Prefer `pub(crate)`.
9. **No TODO without issue** — Every `// TODO:` has an associated tracking issue
10. **Clippy clean** — `cargo clippy -- -D warnings` passes?
11. **Formatted** — `cargo fmt --check` passes?
12. **ROADMAP updated** — If this completes a roadmap item, is the checkbox checked?

## Determinism Red Flags

These patterns are ALWAYS wrong in PEPL compiler/stdlib:

```rust
// WRONG — non-deterministic iteration
use std::collections::HashMap;

// RIGHT — deterministic iteration
use std::collections::BTreeMap;

// WRONG — NaN can enter state
let result = a / b;  // if b == 0.0

// RIGHT — trap on NaN-producing operations
if b == 0.0 { return Err(RuntimeTrap::DivisionByZero); }

// WRONG — panicking in library
let value = some_option.unwrap();

// RIGHT — error propagation
let value = some_option.ok_or(Error::MissingValue)?;
```

## Rules

- **All 12 checklist items must pass** before code is committed
- **Any determinism red flag = immediate rejection** — no exceptions
- **Tests are mandatory** — no test = no merge
- **Spec is authoritative** — if code contradicts the spec files in `docs/`, fix the code
