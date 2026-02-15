---
name: ci-cd-setup
description: Use when creating or modifying GitHub Actions workflows, CI pipelines, or automated publishing for any PEPL repo. Covers testing, formatting, linting, determinism proofs, and WASM builds.
---

# CI/CD Setup

## When to Use

- Creating GitHub Actions workflows
- Setting up automated testing
- Configuring automated package publishing (crates.io, npm for WASM)
- Setting up WASM build pipeline in CI

## Context

PEPL has 3 repos that need CI. The compiler repo has the most complex pipeline (build Rust + compile to WASM + test determinism). CI must enforce determinism — it's the last line of defense.

## Procedure

1. Identify which repo needs CI (pepl, pepl-stdlib, or pepl-ui)
2. Create `.github/workflows/` directory
3. Write workflow YAML following the templates below
4. Ensure the workflow tests determinism (100-iteration tests)
5. Set up caching for Rust dependencies
6. Test the workflow on a branch before merging to main

## Workflow Templates

### pepl: CI (compiler)

```yaml
name: CI
on: [push, pull_request]
jobs:
  build-and-test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: dtolnay/rust-toolchain@stable
      - uses: Swatinem/rust-cache@v2
      - run: cargo fmt --check
      - run: cargo clippy -- -D warnings
      - run: cargo build --workspace
      - run: cargo test --workspace

  wasm-build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: dtolnay/rust-toolchain@stable
      - run: cargo install wasm-pack
      - run: wasm-pack build --target web
```

### pepl-stdlib: CI

```yaml
name: CI
on: [push, pull_request]
jobs:
  build-and-test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: dtolnay/rust-toolchain@stable
      - uses: Swatinem/rust-cache@v2
      - run: cargo fmt --check
      - run: cargo clippy -- -D warnings
      - run: cargo build --workspace
      - run: cargo test --workspace
```

## Rules

- **CI must run determinism tests** — not just `cargo test`, specifically 100-iteration tests
- **CI must check formatting** — `cargo fmt --check`
- **CI must run clippy** — `cargo clippy -- -D warnings`
- **CI must build WASM target** — the compiler runs as WASM in browser
- **Publishing requires a tag** — never auto-publish from main
- **Cache dependencies** — CI should be fast (< 5 minutes for quick checks)

## Anti-Patterns

- CI that doesn't test determinism
- Auto-publishing on every push to main
- No caching (slow builds discourage frequent commits)
- Skipping the WASM build target
- Secrets in workflow files (use GitHub Encrypted Secrets)
