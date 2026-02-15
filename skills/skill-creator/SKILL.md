---
name: skill-creator
description: Guide for creating effective skills. Use when users want to create a new skill (or update an existing skill) that extends capabilities with specialized knowledge, workflows, or tool integrations for the PEPL project.
---

# Skill Creator

This skill provides guidance for creating effective skills for the PEPL development workspace.

## About Skills

Skills are modular, self-contained packages that provide specialized knowledge, workflows, and tools. They transform a general-purpose agent into a specialized one equipped with procedural knowledge for specific domains.

### What Skills Provide

1. Specialized workflows — Multi-step procedures for specific domains
2. Tool integrations — Instructions for working with specific file formats or APIs
3. Domain expertise — PEPL-specific knowledge, compiler internals, WASM patterns
4. Bundled resources — Scripts, references, and assets for complex and repetitive tasks

## Core Principles

### Concise is Key

The context window is a shared resource. Only add context that isn't already known. Challenge each piece of information: "Is this truly needed?" and "Does this paragraph justify its token cost?"

Prefer concise examples over verbose explanations.

### Set Appropriate Degrees of Freedom

Match specificity to the task's fragility:

**High freedom (text-based instructions)**: Multiple valid approaches, context-dependent decisions.

**Medium freedom (pseudocode or scripts with parameters)**: Preferred pattern exists, some variation acceptable.

**Low freedom (specific scripts, few parameters)**: Fragile operations, consistency critical, specific sequence required.

### Anatomy of a Skill

```
skill-name/
├── SKILL.md (required)
│   ├── YAML frontmatter metadata (required)
│   │   ├── name: (required)
│   │   └── description: (required)
│   └── Markdown instructions (required)
└── Bundled Resources (optional)
    ├── scripts/          - Executable code
    ├── references/       - Documentation loaded into context as needed
    └── assets/           - Files used in output (templates, etc.)
```

#### SKILL.md (required)

- **Frontmatter** (YAML): `name` and `description` — these determine when the skill triggers
- **Body** (Markdown): Instructions loaded only when the skill triggers

#### Bundled Resources (optional)

- **Scripts** (`scripts/`): Deterministic, repeatedly-needed code. Token-efficient.
- **References** (`references/`): Documentation loaded on demand. Keep SKILL.md lean.
- **Assets** (`assets/`): Files used in output, not loaded into context.

#### What NOT to Include

No README.md, INSTALLATION_GUIDE.md, CHANGELOG.md, or auxiliary documentation. Only files needed by the agent to do the job.

### Progressive Disclosure

Three-level loading to manage context:

1. **Metadata (name + description)** — Always in context (~100 words)
2. **SKILL.md body** — When skill triggers (<5k words)
3. **Bundled resources** — As needed (unlimited)

Keep SKILL.md body under 500 lines. Split content into separate files when approaching this limit. Reference them clearly from SKILL.md so the reader knows they exist and when to use them.

## PEPL-Specific Skill Guidelines

When creating skills for this workspace:

- **Reference spec files in `docs/`** explicitly — skills should point to specific spec files (e.g., `docs/grammar.md`, `docs/execution-semantics.md`)
- **Include determinism constraints** — every skill touching compiler output must mention the 100-iteration proof
- **Match the pipeline** — skills should be aware of Parser → Type Checker → Invariant Checker → Codegen
- **Error codes** — if the skill involves compiler errors, reference the E-code ranges
- **WASM awareness** — distinguish between compiler-as-WASM and PEPL-to-WASM contexts
