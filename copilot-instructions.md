# PEPL Project — Copilot Instructions

You are working on the **PEPL (Persistent Executable Personal Language)** project — a deterministic, sandboxed, LLM-first language that compiles to WASM.

## Project Structure

- **pepl/** — Compiler (Rust → WASM): parser, type checker, invariant checker, WASM code generator
- **pepl-stdlib/** — Standard library (100 Phase 0 functions + 2 constants across 13 modules)
- **pepl-ui/** — UI component model (10 Phase 0 components, platform-abstract)

## Core Principles

1. **Determinism is non-negotiable** — same source + same inputs → identical outputs, always
2. **Spec is authoritative** — code must match the PEPL spec files in `docs/` (the PEPL language specification)
3. **LLM-first** — PEPL is designed for LLM code generation, not human authoring
4. **Safety** — no host memory access, no filesystem, all I/O host-mediated via capabilities
5. **Honesty** — docs reflect actual state, never aspirations

## Spec Reference

The authoritative PEPL specification lives in `docs/` (15 spec files). When code contradicts the spec, fix the code. Key spec files:

- `docs/grammar.md` — Phase 0 Formal Grammar (EBNF), the parser MUST accept exactly this grammar
- `docs/phase-0-stdlib-reference.md` — Complete Standard Library Reference, every Phase 0 function
- `docs/execution-semantics.md` — normative behavior all hosts MUST implement
- `docs/compiler.md` — Compiler Strategy, Pipeline, Error Message Format (E100–E699)
- `docs/grammar-edge-cases.md` — Edge Cases and Definitive Answers, authoritative rulings

## Cross-Project Reference

- `docs/reference.md` — Single-source cross-project reference for external consumers (`pepl-playground` and any host embedding the compiler). Contains: type system, full stdlib surface (100 functions + 2 constants), CompileResult struct, WASM import/export contract, UI components, error codes, and guarantees. Always verify against source code — this is a derived document.

## Progress & Audit Documents

These are not specs but **required reading** for context before starting any phase:

- `docs/phases.md` — High-level progress tracker: what's built (27 steps, Phase 0 complete), what findings revealed (F1–F34), what remains (Phase 1+)
- `docs/archive/findings.md` — Historical compliance audit: 34 numbered findings (F1–F34) with severity, code locations, and impact. All Phase 0 findings resolved.

## Compiler Pipeline

```
PEPL Source → Parser → AST → Type Checker → Invariant Checker → WASM Code Generator
                                                                    ↓
                                                              wasm-encoder → .wasm binary
```

The compiler itself compiles from Rust to WASM via `wasm-pack` and runs in a browser Web Worker.

## Git

- SSH alias: `github-assetexpand2` (not `github.com`)
- Org: `pepl-lang`
- After every sub-phase: **`git add . && git commit -m "message" && git push`** — always stage first!
- Remotes: `git@github-assetexpand2:pepl-lang/<repo>.git`
- **ALWAYS update `docs/ORCHESTRATION.md`** after completing any orchestration step — mark it ✅ DONE with test counts and deliverables, update the dependency graph and milestone gates. This file is the single source of truth for cross-repo progress. Commit changes in the `.github` repo, never in `pepl/`, `pepl-stdlib/`, or `pepl-ui/`.
- Always `cd` into the specific repo (`pepl/`, `pepl-stdlib/`, `pepl-ui/`) before any git command

## Rust Environment

- Always `source "$HOME/.cargo/env"` before cargo commands

## Key Constraints

- All numbers are 64-bit IEEE 754 floats
- No recursion (Phase 0)
- No concurrency — single-threaded execution
- No `eval`, no dynamic code loading
- Gas metering at loop/call boundaries prevents infinite execution
- NaN cannot enter state — operations that would produce NaN trap instead
