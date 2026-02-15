---
applyTo: "**/stdlib/**,**/pepl-stdlib/**"
---

# Standard Library Development — PEPL

The stdlib provides 100 Phase 0 functions + 2 constants across 9 core modules + 4 capability modules.

## Reference

- `docs/phase-0-stdlib-reference.md` — every Phase 0 function signature and behavior

## Core Modules (Always Available, Deterministic)

| Module | Functions | Key Constraint |
|--------|-----------|----------------|
| `core` | 4 | `core.log` is no-op in production |
| `math` | 10 + 2 constants | All pure, no NaN output (trap instead) |
| `string` | 20 | All pure, UTF-8, no locale-dependent behavior |
| `list` | 34 | All return new lists (immutable), `list.of` is compiler-special-cased variadic |
| `record` | 5 | `record.get` is compile-time checked for known records |
| `time` | 5 | `time.now` is host-provided (deterministic on replay) |
| `convert` | 5 | Fallible conversions return `Result<T, ConvertError>` |
| `json` | 2 | Max depth: 32. Returns `Result<any, JsonError>` |
| `timer` | 4 | Host-managed scheduling, max 8 active timers |

## Capability Modules (Require Host Support)

| Module | Functions | Note |
|--------|-----------|------|
| `http` | 5 | All calls yield to host, return `Result<HttpResponse, HttpError>` |
| `storage` | 4 | Key-value, host-mediated |
| `location` | 1 | Geolocation, permission-gated |
| `notifications` | 1 | Permission-gated |

## Implementation Rules

- Every function executes in < 1ms
- All core functions are **pure** — no side effects, no host calls
- All list/record operations return **new values** — no mutation
- Module names are reserved keywords — cannot be shadowed by user code
- `list.sort` must use a stable, deterministic sorting algorithm
- `string.index_of` and `list.index_of` return -1 for not-found (not nil)
- `convert.to_number` and `convert.parse_*` return `Result` (never panic)
- `json.parse` depth limit is 32 levels

## Stability Guarantee

- Stdlib functions NEVER change behavior once released
- New functions MAY be added
- Existing functions are NEVER removed or modified
- Deprecated functions continue working but emit compiler warnings

## Testing

- Every function has unit tests covering: normal case, edge case, error case
- Determinism proof: 100 iterations producing identical output
- Sort functions: test stability with equal-key elements
- String functions: test with empty strings, Unicode, multi-byte characters
