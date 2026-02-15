---
applyTo: "**/parser/**"
---

# Parser Development — PEPL

The parser converts PEPL source into an AST matching the Phase 0 Formal Grammar (EBNF) in `docs/grammar.md` exactly.

## Pipeline

`PEPL source → Lexer → Token stream → Parser → AST (SpaceNode)`

## Reference

- `docs/grammar.md` — authoritative grammar (EBNF, Operator Precedence, Reserved Keywords)
- `docs/grammar-edge-cases.md` — Expression Precedence Worked Examples, Edge Cases, Structural Limits

## Checklist

1. Read the EBNF production rule you're implementing from `docs/grammar.md`
2. Write the AST type first (what the parser produces)
3. Write the parser function (recursive descent)
4. Write unit tests including edge cases from `docs/grammar-edge-cases.md`
5. Error messages must include line:column and error code
6. 100-iteration determinism test
7. Test against cases from `docs/grammar-edge-cases.md` § "Edge Cases and Definitive Answers"

## Key Grammar Constraints

- Statements separated by newlines (no semicolons)
- Block ordering enforced: types → state → capabilities → credentials → derived → invariants → actions → views → update → handleEvent
- Tests live outside `space {}` declaration
- Only `//` single-line comments
- Lambdas use block body only: `fn(x) { x + 1 }` (not `fn(x) x + 1`)
- No user-defined generics — only built-in `list<T>` and `Result<T, E>`
- String interpolation: `${expr}` (dollar-brace)
- `+` is numbers-only — string concatenation uses `${}`
- `for` iterates lists only — `list.range()` for counted loops
- `?` postfix unwraps Result (traps on Err)
- `??` infix nil-coalescing
- Trailing commas allowed everywhere

## Rules

- Parser is pure: no side effects, no I/O, no randomness
- All state is in the token stream position
- Error recovery: report multiple errors (up to 20), don't stop at first
- AST nodes carry Span for error reporting
- Expression-statement: bare expression whose value becomes block return if last
