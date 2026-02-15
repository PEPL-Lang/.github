---
name: cross-repo-sync
description: Enforces docs/ORCHESTRATION.md sequencing across the pepl, pepl-stdlib, and pepl-ui repos. Use when starting any phase to verify dependencies are met, when checking milestone gates, or when deciding what to work on next. Prevents out-of-order work that creates integration failures.
---

# Cross-Repo Sync

The PEPL project spans 3 repos (`pepl/`, `pepl-stdlib/`, `pepl-ui/`). Work must follow the sequence defined in `docs/ORCHESTRATION.md`.

## Before Starting Any Phase

1. **Read `docs/ORCHESTRATION.md`** — find the step number for the phase you're about to start
2. **Check dependencies** — verify all listed dependencies are complete (checked off in their repo's `ROADMAP.md`)
3. **Check milestone gates** — if the phase is past a milestone boundary, verify all milestone criteria are met
4. **Only then proceed** — with the `phase-execution` skill workflow

## Dependency Lookup

| If starting... | Must be complete first |
|----------------|----------------------|
| C2 (Lexer) | C1 |
| S1 (Stdlib scaffolding) | nothing |
| U1 (UI scaffolding) | nothing |
| C3 (Parser) | C2 |
| S2, S3, S4, S5 (Stdlib modules) | S1 |
| S6 (Capability modules) | S1 |
| U2–U6 (UI components) | U1 |
| C4 (Type Checker) | C3 |
| C5 (Invariant Checker) | C4 |
| C6 (Evaluator) | C5, S1–S5, U1 |
| C7 (WASM Codegen) | C5, S6, U1, C6 |
| U7 (Accessibility) | U2–U6 |
| U8 (Final UI Validation) | U1–U7 |
| C8 (Integration) | C7, S1–S6, U1–U8 |
| C9 (Canonical Examples) | C8 |
| C10 (WASM Parity) | C9 |
| C11 (AST Diff + Source Map) | C10 |
| C12 (LLM Reference) | C11 |

> **Phase 0 is complete.** All steps C1–C12, S1–S6, U1–U8 are done (27 ORCHESTRATION steps).

## Milestone Gates

| Milestone | After | Criteria |
|-----------|-------|----------|
| **M1** | C3, S1–S3, U1–U2 | All 7 canonical examples parse into valid ASTs |
| **M2** | C5, S1–S5, U1–U4 | All examples pass type-check + invariant validation |
| **M3** | C6, U1–U6 | Evaluator executes all examples, golden reference captured |
| **M4** | C8, S1–S6, U1–U8 | Full pipeline, WASM matches evaluator reference |
| **M5** | C10 | WASM parity verified, canonical test suite passes |
| **M6** | C12 | LLM tooling complete, source maps, test codegen |

All milestones M1–M6 have been passed.

## Parallelizable Work

Some phases can be built simultaneously if their dependencies are met:

- S1, U1 can be built in parallel with C2
- S2, S3, U2 can be built in parallel with C3
- S4, U3 can be built in parallel with C4
- S5, U4 can be built in parallel with C5
- U5, U6 can be built in parallel with C6
- S6, U7, U8 can be built in parallel with C7

Parallel phases are **optional** — you can do them during or after the paired compiler phase.

## Commit Rules

Each repo has its own git repository. After completing any phase:

1. `cd` into the specific repo directory
2. **Stage all changes**: `git add .` — MANDATORY, without this nothing is committed or pushed
3. Commit with conventional format: `git commit -m "feat(scope): Phase X.Y — description"`
4. Push to remote: `git push`
5. Update the repo's `ROADMAP.md` checkboxes

**Never run git from the workspace root** (`PEPL-Lang/`) — it is not a git repo.

## What to Work On Next

When asked "what's next?" or "proceed":

1. Find the current step in `docs/ORCHESTRATION.md` (first unchecked step)
2. Verify its dependencies
3. If dependencies are met → start that phase
4. If dependencies are NOT met → start the blocking dependency instead
5. If multiple independent phases are available → suggest working on the critical path first
