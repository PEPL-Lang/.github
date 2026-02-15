# Transformation Examples

## Water Tracker → Enhanced Water Tracker (Incremental)

```pepl
// Before
space WaterTracker {
  state { current_intake: number = 0, daily_goal: number = 2000 }
  action add_water(amount: number) { set current_intake = current_intake + amount }
  view main() -> Surface {
    Column {
      Text { value: "Water: ${current_intake}ml / ${daily_goal}ml" }
      Button { label: "+ 250ml", on_tap: add_water(250) }
    }
  }
}

// After (code generator adds reset + progress bar + derived)
space WaterTracker {
  state { current_intake: number = 0, daily_goal: number = 2000 }

  derived {
    progress: number = current_intake / daily_goal
  }

  action add_water(amount: number) { set current_intake = current_intake + amount }
  action reset_intake() { set current_intake = 0 }  // NEW
  view main() -> Surface {
    Column {
      Text { value: "Water: ${current_intake}ml / ${daily_goal}ml" }
      ProgressBar { value: progress }                  // NEW (uses derived)
      Row {
        Button { label: "+ 250ml", on_tap: add_water(250) }
        Button { label: "+ 500ml", on_tap: add_water(500) }    // NEW
        Button { label: "Reset", on_tap: reset_intake }         // NEW
      }
    }
  }
}
```

## Game Loop Pattern

```pepl
space BouncingBall {
  state {
    x: number = 160, y: number = 100
    vx: number = 3, vy: number = 2
  }

  update(dt: number) {
    set x = x + vx * dt
    set y = y + vy * dt
    if x <= 0 or x >= 320 { set vx = -vx }
    if y <= 0 or y >= 240 { set vy = -vy }
  }

  view main() -> Surface {
    Canvas {
      width: 320,
      height: 240,
      on_draw: fn(ctx) {
        ctx.fill_rect(0, 0, 320, 240, "#000000")
        ctx.fill_circle(x, y, 10, "#FF0000")
      },
    }
  }
}
```

---

## Performance Characteristics

| Operation | Target | Notes |
|-----------|--------|-------|
| Stdlib function call | < 1ms | Pure computation |
| UI component render | < 16ms | 60fps budget |
| PEPL compilation (small space, < 200 lines) | < 500ms | In Web Worker |
| PEPL compilation (medium space, 200–1000 lines) | < 2s | In Web Worker |
| PEPL compilation (large space, 1000+ lines) | < 5s | In Web Worker; incremental compilation reduces this for Evolve |
| State action execution | < 50ms | WASM sandbox, gas-metered |
| Memory per space (stdlib) | < 100KB | Stdlib overhead |

No artificial construct or resource limits — performance scales with hardware capabilities, not with counts of declarations or hardcoded caps. Resource limits (memory, CPU quotas) are host-configured.
