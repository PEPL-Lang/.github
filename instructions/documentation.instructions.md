---
applyTo: "**/*.md"
---

# Documentation — PEPL

## Honesty Policy

- Never document features that don't exist as if they work
- Status badges must reflect actual current phase
- Code examples must actually compile (or be clearly marked "planned")
- Install commands only shown when packages actually exist

## Cross-Repo Links

Always use absolute GitHub URLs for cross-repo references:
- `https://github.com/pepl-lang/pepl`
- `https://github.com/pepl-lang/pepl-stdlib`
- `https://github.com/pepl-lang/pepl-ui`

## Spec Reference

The language specification in `docs/` is the authoritative source. In documentation:
- Reference spec files by name: "See `docs/grammar.md`" or "See `docs/execution-semantics.md` § Action Execution Model"
- Never duplicate spec content into docs — link to it
- If docs contradict the spec, the spec wins

## After Any Change

- Verify `README.md` in all 3 repos stays in sync
- Update ROADMAP.md status if completing a phase item
- Ensure error code documentation matches implemented E-codes

## Doc Comment Standards (Rust)

- All public items documented with `///`
- Include `# Guarantees` section for determinism-critical functions
- Include `# Panics` section if function can panic (should be rare — prefer `Result`)
- Include `# Examples` with compilable doc tests where possible
