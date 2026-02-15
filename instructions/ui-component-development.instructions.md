---
applyTo: "**/ui/**,**/pepl-ui/**,**/components/**"
---

# UI Component Development — PEPL

The UI component model defines 10 Phase 0 components that compile to platform-abstract surfaces.

## Reference

- `docs/ui-components.md` — component props, types, behavior, accessibility

## Phase 0 Components

| Component | Category | Key Props |
|-----------|----------|-----------|
| `Text` | Content | `value`, `size`, `color`, `weight`, `align` |
| `Button` | Interactive | `label`, `on_tap`, `variant: "filled"\|"outlined"\|"text"`, `disabled` |
| `TextInput` | Interactive | `value`, `on_change`, `placeholder`, `keyboard: "text"\|"number"\|"email"\|"phone"\|"url"` |
| `Column` | Layout | `spacing`, `padding`, `align` |
| `Row` | Layout | `spacing`, `padding`, `align` |
| `Scroll` | Layout | `direction` |
| `ScrollList` | List | `items`, `render_item`, `key_fn` |
| `ProgressBar` | Feedback | `value` (0.0–1.0), `color` |
| `Modal` | Feedback | `visible`, children via second brace block |
| `Toast` | Feedback | `message`, `visible`, `duration` |

## Implementation Rules

- Components are **platform-abstract** — never reference DOM, native APIs, or rendering details
- Components produce `Surface` trees — the host's View Layer renders them
- Every component has built-in accessibility support (labels, hints, roles)
- Component children use second brace block: `Modal { props } { children }`
- Unknown component names produce compile error E402
- All components render in < 16ms (60fps budget)
- `color` values in Phase 0 are hex string literals only (e.g., `"#FF5733"`)

## Prop Type Enforcement

- Enum-like props use union string types, not bare `string`
- `Icon.name` follows Material Symbols naming (unknown names render placeholder square)
- `Image.source` accepts HTTPS URLs or data URIs only (no local file paths — sandboxed)
- `dimension` type is defined in the spec — use it for sizing props

## Action References in Props

Event handler props (`on_tap`, `on_change`) accept action references:

```
on_tap: action_name           — nullary action reference
on_tap: action_name(arg)      — action with bound argument
on_change: fn(v) { action(v) } — explicit lambda callback
```

Action references are NOT eagerly evaluated — they construct callbacks. The compiler distinguishes ActionRef from FunctionCall by checking if the identifier names an action.

## Testing

- Every component has tests for: required props, optional props, children, action binding
- Accessibility: every component test verifies accessibility attributes are present
- Invalid props: test that wrong types produce clear compile errors
