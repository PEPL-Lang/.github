---
applyTo: "**"
---

# Git Workflow — PEPL

## SSH Remote

All repos use SSH host alias: `git@github-assetexpand2:pepl-lang/<repo>.git`

## Commit Message Format

```
type(scope): Brief description

- What changed
- Why it changed
```

Types: `feat`, `fix`, `refactor`, `chore`, `docs`, `test`
Scopes: `parser`, `lexer`, `type-checker`, `invariant-checker`, `codegen`, `gas`, `stdlib`, `ui`, `cli`

## Rules

1. **ALWAYS `cd` into the repo directory before any git command** — the workspace root `/home/opeworld/Documents/RobustBrains/PEPL-Lang/` is NOT a git repo. Use `cd /home/opeworld/Documents/RobustBrains/PEPL-Lang/pepl` (or `pepl-stdlib`, `pepl-ui`, `.github`).
2. `cargo build && cargo test` must pass before every commit (for Rust repos)
3. **ALWAYS stage files before committing**: `git add .` (or `git add -A`) — without this, `git commit` has nothing to commit and `git push` pushes nothing.
4. Full commit sequence: `git add . && git commit -m "type(scope): message" && git push`
5. Commit messages reference the phase: "Phase 1.1", "Phase 1.2", etc.
6. Never force-push to main

## 4 Repos

| Repo | Remote |
|------|--------|
| pepl | `git@github-assetexpand2:pepl-lang/pepl.git` |
| pepl-stdlib | `git@github-assetexpand2:pepl-lang/pepl-stdlib.git` |
| pepl-ui | `git@github-assetexpand2:pepl-lang/pepl-ui.git` |
| .github | `git@github-assetexpand2:pepl-lang/.github.git` |

## Environment

```bash
source "$HOME/.cargo/env"  # ALWAYS before any cargo command
```
