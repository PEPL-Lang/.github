---
name: wasm-build
description: Use when building, testing, or debugging the WASM compilation targets — both the compiler-to-WASM build (Rust → WASM via wasm-pack) and the PEPL-to-WASM code generator (PEPL AST → wasm-encoder → .wasm binary). Triggers on "wasm", "wasm-pack", "wasm-encoder", "web worker", "browser build", "compilation target".
---

# WASM Build

## When to Use

- Building the PEPL compiler as WASM (for browser Web Worker)
- Working on the WASM code generator (PEPL → .wasm output)
- Debugging WASM output issues
- Setting up wasm-pack or wasm-encoder integration
- Testing WASM output determinism

## Two WASM Contexts

PEPL has two distinct WASM concerns — don't confuse them:

### 1. Compiler-as-WASM (Rust → WASM)

The PEPL compiler itself is compiled from Rust to WASM via `wasm-pack`, so it can run in a browser Web Worker.

```
Rust source (pepl compiler) → wasm-pack → pepl_compiler.wasm → Web Worker
```

- Built with `wasm-pack build --target web`
- Uses `wasm-bindgen` for JavaScript interop
- Must be small enough to load quickly in browser
- No `std::fs`, no `std::net` — browser-compatible APIs only

### 2. PEPL-to-WASM (Code Generator Output)

The compiler's code generator takes a PEPL AST and emits a `.wasm` binary using the `wasm-encoder` crate.

```
PEPL source → Parser → AST → ... → WASM Codegen → wasm-encoder → space.wasm → Sandbox
```

- Uses `wasm-encoder` crate for type-safe WASM module construction
- Injects gas metering at loop/call boundaries
- Output must be deterministic: same PEPL source → identical .wasm bytes
- NaN prevention: emit trap guards around division and sqrt

## wasm-encoder Usage

```rust
use wasm_encoder::{Module, CodeSection, Function, Instruction, ValType};

// Build module programmatically — no WAT text intermediary
let mut module = Module::new();
// Add types, functions, memory, exports...
let wasm_bytes: Vec<u8> = module.finish();
```

Key advantages over hand-rolled encoding:
- Type-safe API prevents encoding errors
- Debug with `wasmparser` validation + WAT disassembly
- Handles WASM binary format edge cases

## Testing WASM Output

```rust
#[test]
fn test_wasm_output_determinism() {
    let source = "space Counter { state { count: number = 0 } ... }";
    let first = compile_to_wasm(source).unwrap();
    for i in 0..100 {
        let result = compile_to_wasm(source).unwrap();
        assert_eq!(first, result, "WASM bytes differ at iteration {}", i);
    }
}

#[test]
fn test_wasm_validates() {
    let wasm_bytes = compile_to_wasm(source).unwrap();
    wasmparser::validate(&wasm_bytes).expect("Generated WASM must be valid");
}
```

## Rules

- **WASM output must be deterministic** — 100-iteration proof required
- **Always validate output** with `wasmparser` — never assume generated WASM is valid
- **Gas metering is mandatory** — every loop and call must be metered
- **NaN guards are mandatory** — division and sqrt must trap, not produce NaN
- **Compiler WASM build must work in browser** — no system APIs
- **Use wasm-encoder, not hand-rolled binary** — the spec explicitly chose this approach
