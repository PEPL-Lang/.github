# .github — PEPL Organization Repository

Central hub for the PEPL language specification, project coordination, and development tooling.

## Contents

### `profile/`

GitHub organization profile page — renders at [github.com/pepl-lang](https://github.com/pepl-lang).

### `docs/`

The authoritative PEPL language specification (15 spec files) and project coordination documents.

| File | Description |
|------|-------------|
| [overview.md](docs/overview.md) | Top-level language overview |
| [grammar.md](docs/grammar.md) | Phase 0 formal grammar (EBNF) |
| [grammar-edge-cases.md](docs/grammar-edge-cases.md) | Definitive rulings on ambiguous grammar cases |
| [execution-semantics.md](docs/execution-semantics.md) | Normative runtime behavior |
| [language-structure.md](docs/language-structure.md) | Types, expressions, statements |
| [stdlib.md](docs/stdlib.md) | Standard library design |
| [phase-0-stdlib-reference.md](docs/phase-0-stdlib-reference.md) | Complete Phase 0 function reference (100 functions + 2 constants) |
| [compiler.md](docs/compiler.md) | Compiler pipeline strategy and error codes (E100–E699) |
| [capability-system.md](docs/capability-system.md) | Capability-based security model |
| [host-integration.md](docs/host-integration.md) | WASM embedding contract |
| [ui-components.md](docs/ui-components.md) | 10 Phase 0 UI components |
| [llm-generation-contract.md](docs/llm-generation-contract.md) | LLM code generation rules |
| [transformation-examples.md](docs/transformation-examples.md) | Canonical PEPL programs |
| [project-metadata.md](docs/project-metadata.md) | Project metadata format |
| [future.md](docs/future.md) | Phase 1+ planned features |
| [reference.md](docs/reference.md) | Cross-project API reference for host embedders |
| [ORCHESTRATION.md](docs/ORCHESTRATION.md) | Cross-repo build sequencing (27 completed steps) |
| [phases.md](docs/phases.md) | High-level progress tracker |
| [archive/findings.md](docs/archive/findings.md) | Historical compliance audit (F1–F34, all Phase 0 resolved) |

### `instructions/`

GitHub Copilot instruction files — automatically loaded based on file context.

| File | Applies To |
|------|------------|
| core-rules.instructions.md | All files |
| git-workflow.instructions.md | All files |
| progress-tracking.instructions.md | All files |
| roadmap-tracking.instructions.md | ROADMAP.md files |
| documentation.instructions.md | Markdown files |
| rust-development.instructions.md | Rust files |
| compiler-pipeline.instructions.md | Type checker, invariant checker, codegen, gas |
| parser-development.instructions.md | Parser files |
| stdlib-development.instructions.md | Stdlib files |
| ui-component-development.instructions.md | UI component files |
| testing-determinism.instructions.md | Test files |

### `skills/`

Specialized Copilot skill modules for domain-specific workflows.

| Skill | Purpose |
|-------|---------|
| phase-execution | Workflow for executing ROADMAP phases |
| cross-repo-sync | Enforces ORCHESTRATION.md sequencing across repos |
| brainstorming | Structured design decision evaluation |
| code-review | Quality validation against PEPL standards |
| debugging | Systematic debugging for the compiler pipeline |
| interpreter-development | Tree-walking evaluator (pepl-eval) development |
| wasm-build | WASM compilation targets and wasm-pack builds |
| ci-cd-setup | GitHub Actions workflows |
| skill-creator | Guide for creating new skills |
| using-superpowers | Skill discovery and invocation rules |

## Related Repos

| Repo | Description |
|------|-------------|
| [pepl](https://github.com/pepl-lang/pepl) | Compiler — lexer, parser, type checker, evaluator, WASM codegen (8 crates) |
| [pepl-stdlib](https://github.com/pepl-lang/pepl-stdlib) | Standard library — 100 deterministic functions + 2 constants |
| [pepl-ui](https://github.com/pepl-lang/pepl-ui) | UI component model — 10 accessible, deterministic components |

## License

MIT
