# Future Platform UI Targets (Compressed)

> Phase 0 renders all UI to browser DOM. Future phases add platform-native rendering.

## Surface Rendering Rules by Platform

| Surface Element | DOM (Phase 0) | Desktop Native | Mobile Native | Glasses | Gadget |
|----------------|---------------|----------------|---------------|---------|--------|
| `Button` | `<button>` | Native button | Tappable 44pt | Gaze-selectable | Physical button map |
| `TextInput` | `<input>` | Native field | Virtual keyboard | Voice dictation | Voice input / unavailable |
| `Scroll` | CSS overflow | Native scroll | Touch rubber-band | Head tilt | Button scroll |
| `Modal` | Fixed overlay | Centered window | Bottom sheet | Fixed HUD | Audio announcement |
| `Chart` | `<canvas>` | Full interactive | Touch-zoomable | Simplified 2D | Text summary |

## UI Downgrading Chain

When platform cannot render full UI: **Full UI → Simplified UI → Text-only → Audio-only → Status indicators**. Each step has defined rules for which components degrade and how.

**Rendering guarantee:** Every valid `Surface` MUST be renderable on every supported platform, possibly in degraded form. If a component has no visual representation on a platform (e.g., `TextInput` on a minimal gadget), the host either degrades it to an available modality (voice input, physical button) or omits it with an audio/status indicator. A space is never rejected because its target platform cannot render a component — the host always finds a degradation path.

## Resource Annotations (Future)

```pepl
// Future: Per-component resource hints for constrained platforms
@resource(render_cost: "high", min_memory: "4MB")
view complex_chart() -> Surface { ... }

@resource(render_cost: "low")
view simple_summary() -> Surface { ... }
```

Constrained platforms (glasses, gadgets) can select lower-cost views automatically.

## PEPL Language Evolution

**Versioning:** PEPL uses semantic versioning `MAJOR.MINOR.PATCH`:
- **Phase 0** = `0.1.0` (pre-1.0, breaking changes allowed between minor versions)
- **MAJOR** bumps = grammar-breaking changes (new keywords, changed syntax)
- **MINOR** bumps = new constructs/components (backwards compatible)
- **PATCH** bumps = bug fixes, clarifications

Every compiled WASM module embeds the PEPL version it was compiled with. Hosts check version compatibility before execution. Pre-1.0 modules (`0.x.y`) are not guaranteed cross-version compatible.

```
Phase 1: Proposal (RFC document)
  → Describe syntax, semantics, motivation, examples
  → Must include migration path from existing spaces

Phase 2: Experimental (behind flag)
  → Available as @experimental("feature_name")
  → Not guaranteed stable
  → Feedback collected from actual usage

Phase 3: Stable (default available)
  → No more breaking changes to this feature
  → Migration tools provided if syntax changed from experimental

Phase 4: Mature (widely used)
  → Performance optimizations allowed
  → Used as building block for newer features
```

## Deprecation Lifecycle

```
Version N:   Feature works normally
Version N+1: Compiler warning — "X is deprecated, use Y instead"
Version N+2: Compiler error — "X removed, use Y instead"
             Old compiled WASM continues running (binary compat)
```
