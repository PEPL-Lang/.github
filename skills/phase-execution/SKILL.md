---
name: phase-execution
description: Use when starting, executing, or completing any ROADMAP.md phase or sub-phase. Enforces the repeating workflow for every phase — read spec, implement in batches, test with determinism proof, update ROADMAP checkboxes, git commit and push. Triggers on phrases like "start phase", "continue phase", "proceed", "begin 1.3", or any reference to executing a ROADMAP item.
---

# Phase Execution

Rigid workflow for executing any ROADMAP.md phase. Follow exactly — skipped steps cause rework.

## Before Starting

1. **Check docs/ORCHESTRATION.md** — verify the phase's dependencies are complete (use `cross-repo-sync` skill)
2. Read the phase items from the repo's `ROADMAP.md`
3. Read relevant spec files from `docs/` (grammar, semantics, stdlib, error codes)
4. Read existing code that the phase builds on
5. Present a brief plan to the user — what will be built, in how many batches

## Implementation Loop

For each batch:

1. **Announce** — Tell user what this batch covers
2. **Implement** — Write the code
3. **Build** — `source "$HOME/.cargo/env" && cargo build --workspace`
4. **Test** — `cargo test --workspace` (all existing + new tests must pass)
5. **Verify** — Re-read the code just written; confirm it matches the spec and fulfils the ROADMAP items for this batch. Fix any issues before moving on.
6. **Determinism** — If the phase involves compiler output, run 100-iteration determinism test
7. **Report** — Show results, ask user to type "continue" for next batch

## After All Batches Complete

1. **Full test suite** — `source "$HOME/.cargo/env" && cargo build --workspace && cargo test --workspace`
2. **Clippy** — `cargo clippy -- -D warnings`
3. **Determinism proof** — 100-iteration test on the completed component
4. **Verify against spec** — Re-read all code written in this phase and cross-check against:
   - The spec files in `docs/` (grammar.md, execution-semantics.md, stdlib.md, etc.)
   - The ROADMAP.md checklist items — confirm each item's requirements are fully met
   - Public API surface: correct types, field names, error variants, JSON serialization
   - Determinism constraints: BTreeMap (never HashMap), no rand, no system clock
   - Fix any discrepancies found before proceeding
5. **Update ROADMAP.md** — Check off all completed items with `[x]`
6. **Update docs/ORCHESTRATION.md** — update the completed step:
   - Mark the step heading with `✅ DONE`
   - Add a `Delivered:` line with test counts and key deliverables
   - Update the Compiler Phase Numbering table if it's a compiler phase
   - Update the dependency graph markers (add ✅ after completed nodes)
   - Check off any milestone gate criteria that are now satisfied
   - Commit changes in the `.github` repo
7. **Git commit & push** — ALWAYS `cd` into the specific repo directory first, ALWAYS `git add` before committing:
   ```bash
   cd <PEPL_WORKSPACE>/<repo>  # pepl, pepl-stdlib, or pepl-ui
   git add .  # MANDATORY — without this, nothing is staged and commit/push do nothing
   git commit -m "feat(scope): Phase X.Y — description"
   git push
   ```
   **NEVER run git commands from the PEPL workspace root** — it is NOT a git repo.
8. **Report** — Confirm phase complete, state what's next

## Environment

```bash
source "$HOME/.cargo/env"  # ALWAYS before any cargo command
```

SSH remotes use host alias `github-assetexpand2`, not `github.com`.

## Rules

- **Never skip the determinism test** — it's the core promise of PEPL
- **Never skip ROADMAP update** — it's the single source of truth for progress
- **Never skip git commit+push** — every sub-phase gets committed
- **Batch size** — user controls when to continue; don't auto-proceed
- **Build before test** — always `cargo build` before `cargo test`
- **Spec is authoritative** — if code contradicts the spec files in `docs/`, fix the code
- **Always `git add .` before `git commit`** — forgetting this means nothing is staged, committed, or pushed
