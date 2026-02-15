---
applyTo: "**"
---

# Progress Tracking — PEPL

## Context Documents

Three documents provide essential context for ongoing work. **Always read them before starting any phase or debugging any issue.**

### `docs/reference.md` — Cross-Project API Reference

**What it is:** The single-source cross-project reference for the PEPL language and compiler. Contains the type system, full stdlib surface (100 functions + 2 constants), CompileResult struct, WASM import/export contract, UI components, error codes, and guarantees.

**When to read it:**
- Before starting any phase — it has the definitive Phase 0 function list, type system, and spec tables
- When checking whether something is in scope or out of scope
- When verifying stdlib function names/signatures
- When working on UI components — it has the full component spec, props, and rendering rules

**Location:** `docs/reference.md`

### `docs/phases.md` — High-Level Progress & Roadmap

**What it is:** A comprehensive tracking document showing what's built, what the compliance audit found, and what remains across all phases (0A through 2+).

**When to read it:**
- Before starting any new phase — understand where it fits in the overall picture
- When prioritizing work — Phase 0A items are critical and block everything else
- When a user asks about project status or progress

**What it contains:**
- "What's Built" — all 27 completed ORCHESTRATION steps with checkboxes (Phase 0 complete)
- "What the Findings Revealed" — summary of all 34 audit findings (F1–F34) grouped by severity
- "What Remains" — Phase 1+ items

### `docs/archive/findings.md` — Historical Compliance Audit

**What it is:** The historical compliance audit comparing the three PEPL repos against all spec files. Contains 34 numbered findings (F1–F34) across 2 passes with exact code locations, severity ratings, and impact analysis. All Phase 0 findings have been resolved.

**When to read it:**
- When understanding the history of a particular finding (F-numbered item)
- When working on Phase 1+ items that reference findings
- For context on past decisions and resolutions

**What it contains:**
- Pass 1 (F1–F15): Core compliance — grammar, type system, stdlib names, codegen gaps, host integration
- Pass 2 (F16–F34): Completeness — WASM stubs inventory, metadata gaps, infrastructure needs, scalability

> **Note:** All Phase 0 findings are resolved. This document is archived for historical reference.

## Workflow

1. Read `docs/phases.md` to understand what phase you're in and what's next
2. Read the relevant findings in `docs/archive/findings.md` for historical root cause
3. Execute work from `docs/ORCHESTRATION.md` (cross-repo sequencing) and the repo's `ROADMAP.md` (detailed checkboxes)
4. `docs/phases.md` and `docs/archive/findings.md` are **context documents** — the authoritative execution sequence is always `docs/ORCHESTRATION.md` + each repo's `ROADMAP.md`
