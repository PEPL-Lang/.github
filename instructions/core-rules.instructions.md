---
applyTo: "**"
---

# Core Rules — PEPL Development

## This Is a Development Workspace

This workspace builds the PEPL compiler, stdlib, and UI component model. Every output is working Rust code, tests, or documentation.

## The Spec Is Authoritative

The PEPL language specification lives in `docs/` (15 spec files). It is the **single source of truth** for language semantics, grammar, stdlib signatures, and component definitions.

- If code contradicts the spec, fix the code
- If a spec file is ambiguous, raise it before guessing
- Never silently deviate from the spec

## Determinism Is Non-Negotiable

- Same PEPL source + same inputs MUST produce identical outputs
- Same compiler invocation MUST produce identical WASM bytecode
- Use `BTreeMap`/`BTreeSet` — NEVER `HashMap`/`HashSet`
- No `rand`, no randomness, no system clock in compiler or stdlib
- All iteration order must be deterministic
- NaN cannot enter PEPL state — operations producing NaN trap instead

## Quality Standards

- Every public function has a unit test
- Every error path has a test
- 100-iteration determinism proof for all compiler outputs
- `cargo fmt --check` and `cargo clippy -- -D warnings` must pass
- No `unwrap()` in library code — use `?` or `.ok_or(Error::...)?`
- No `// TODO:` without a tracking issue

## Workspace Layout

```
PEPL/
├── .github/          — instructions + skills (this directory)
├── pepl/             — compiler (Cargo workspace)
├── pepl-stdlib/      — standard library
└── pepl-ui/          — UI component model
```

Each subdirectory is its own git repo under `pepl-lang` org.

## Cross-References

- Spec: 15 spec files in `docs/` (see `docs/overview.md` for the top-level view)
- Host integration context: see `docs/reference.md` for the public API surface
