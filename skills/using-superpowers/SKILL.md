---
name: using-superpowers
description: Use when starting any conversation — establishes how to find and use skills, requiring skill invocation before ANY response including clarifying questions. If there is even a 1% chance a skill might apply, you MUST invoke it.
---

<EXTREMELY-IMPORTANT>
If you think there is even a 1% chance a skill might apply to what you are doing, you ABSOLUTELY MUST invoke the skill.

IF A SKILL APPLIES TO YOUR TASK, YOU DO NOT HAVE A CHOICE. YOU MUST USE IT.

This is not negotiable. This is not optional. You cannot rationalize your way out of this.
</EXTREMELY-IMPORTANT>

## How to Access Skills

Skills are in `.github/skills/`. When a skill applies, read its `SKILL.md` file IMMEDIATELY as your first action, BEFORE generating any response or taking action.

# Using Skills

## The Rule

**Invoke relevant skills BEFORE any response or action.** Even a 1% chance a skill might apply means you should read it to check. If it turns out to be wrong for the situation, you don't need to follow it.

## Available Skills

| Skill | When to Use |
|-------|-------------|
| `phase-execution` | Starting, executing, or completing any ROADMAP phase |
| `cross-repo-sync` | Starting any phase (verify deps), checking milestones, deciding what's next |
| `interpreter-development` | Building or using the pepl-eval tree-walking evaluator |
| `brainstorming` | Planning, exploring design decisions, comparing approaches |
| `code-review` | Before committing code, reviewing changes |
| `debugging` | Investigating bugs, test failures, unexpected behavior |
| `ci-cd-setup` | Creating GitHub Actions workflows, CI pipelines |
| `wasm-build` | WASM compilation, wasm-pack, wasm-encoder work |
| `skill-creator` | Creating or updating skills |
| `using-superpowers` | This skill — how to use skills |

## Red Flags

These thoughts mean STOP — you're rationalizing:

| Thought | Reality |
|---------|---------|
| "This is just a simple question" | Questions are tasks. Check for skills. |
| "I need more context first" | Skill check comes BEFORE clarifying questions. |
| "Let me explore the codebase first" | Skills tell you HOW to explore. Check first. |
| "This doesn't need a formal skill" | If a skill exists, use it. |
| "I remember this skill" | Skills evolve. Read current version. |
| "The skill is overkill" | Simple things become complex. Use it. |
| "I'll just do this one thing first" | Check BEFORE doing anything. |

## Skill Priority

When multiple skills could apply:

1. **Process skills first** (brainstorming, debugging) — these determine HOW to approach the task
2. **Implementation skills second** (wasm-build, ci-cd-setup) — these guide execution

"Let's build X" → brainstorming first, then implementation skills.
"Fix this bug" → debugging first, then domain-specific skills.

## Skill Types

**Rigid** (phase-execution, debugging): Follow exactly. Don't adapt away discipline.

**Flexible** (brainstorming): Adapt principles to context.

The skill itself tells you which.

## User Instructions

Instructions say WHAT, not HOW. "Add X" or "Fix Y" doesn't mean skip workflows.
