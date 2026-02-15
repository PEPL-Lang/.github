---
applyTo: "**/ROADMAP.md"
---

# Roadmap Tracking — PEPL

## ORCHESTRATION.md Is the Master Sequence

`docs/ORCHESTRATION.md` defines the cross-repo execution order.
Each repo also has its own `ROADMAP.md` with detailed sub-phase checklists.

**Always check `docs/ORCHESTRATION.md` before starting any phase** to verify dependencies.

## ROADMAP.md Is the Detailed Source of Truth

Each repo has its own `ROADMAP.md`. It defines sequential phases with checkbox items.

## Rules

- Phases are **sequential** — complete Phase N before starting Phase N+1
- Check off items (`[x]`) only when code compiles, tests pass, and determinism is verified
- Never check off items that are partially done
- Never reorder phases without explicit discussion
- Each sub-phase gets its own git commit after completion

## Phase Format

```markdown
## Phase N: [Name]

### N.1 [Sub-phase]
- [ ] Item 1
- [ ] Item 2

### N.2 [Sub-phase]
- [ ] Item 3
```

## Verification Before Checking Off

Before marking any item `[x]`:
1. `source "$HOME/.cargo/env"` before any cargo command
2. `cargo build --workspace` succeeds
3. `cargo test --workspace` all pass
4. `cargo clippy -- -D warnings` clean
5. 100-iteration determinism test passes (if applicable)
6. Code reviewed against spec (see spec files in `docs/`)
7. Git commit and push from the specific repo directory (never from workspace root)
