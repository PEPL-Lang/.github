---
name: brainstorming
description: Use when planning new features, exploring design decisions, or thinking through an approach before implementing. Helps evaluate trade-offs for architecture decisions in the PEPL project. Triggers on "let's think about", "how should we", "what's the best way", "explore", "trade-offs", "pros and cons".
---

# Brainstorming

## When to Use

- User says "let's think about...", "how should we...", "what's the best way to..."
- Before starting a new phase from ROADMAP.md
- Designing a new compiler feature or stdlib function
- Choosing between multiple implementation approaches
- Architecture decisions that affect the compilation pipeline

## Context

PEPL is a language with strict determinism guarantees. Design decisions in the compiler, stdlib, and component model are hard to reverse once the language is in use. Think before building.

Reference files:
- Spec files in `docs/` — language specification (authoritative)
- `ROADMAP.md` — what's committed vs planned (per repo)

## Procedure

1. **Define the problem** — What exactly are we deciding? Write it as a single question.
2. **List constraints** — What do the spec `.md` files require? What does the ROADMAP commit to?
3. **Generate options** — At least 2-3 approaches. Don't anchor on the first idea.
4. **Evaluate trade-offs** — For each option:
   - Does it preserve determinism?
   - Does it fit the compiler pipeline (Parser → Type Checker → Invariant Checker → Codegen)?
   - Does it affect WASM output?
   - How complex? How maintainable?
   - Does it align with the LLM-first design (error messages, token efficiency)?
5. **Recommend** — Pick one and explain why.
6. **Get user confirmation** — Present the recommendation before implementing.

## Decision Template

```markdown
### Decision: [Title]

**Question:** [What are we deciding?]

**Constraints:**
- [Constraint from spec `.md` files]
- [Constraint from pipeline]

**Options:**
1. **[Option A]** — [Brief description]
   - Pro: ...
   - Con: ...
2. **[Option B]** — [Brief description]
   - Pro: ...
   - Con: ...

**Recommendation:** Option [X] because [reason].
```

## Rules

- **Spec is a hard constraint** — no option that violates the spec files in `docs/`
- **Determinism is a hard constraint** — no option that introduces non-determinism
- **Don't over-design** — solve today's problem, not hypothetical future problems
- **Document the decision** — significant decisions go in a comment or PLAN.md

## Anti-Patterns

- Implementing before thinking
- Considering only one option
- Ignoring spec constraints in design
- Over-engineering for hypothetical future needs
- Not documenting significant decisions
