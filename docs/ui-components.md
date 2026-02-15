# UI Component Library

All components are platform-abstract. They compile to browser DOM elements in Phase 0, native representations on future platforms through the host application's platform abstraction layer.

## Layout Components

```pepl
// Vertical stack
Column { spacing?: number, align?: alignment, padding?: edges } { children }

// Horizontal stack
Row { spacing?: number, align?: alignment, padding?: edges } { children }

// Layered stack (z-axis, last child on top)
Stack { align?: alignment } { children }

// Scrollable container
Scroll { direction?: "vertical" | "horizontal" | "both" } { children }

// Grid layout
Grid { columns: number, spacing?: number } { children }

// Spacer (flexible empty space)
Spacer { size?: number }

// Divider line
Divider { thickness?: number, color?: color }

// Container with styling
Container {
  background?: color, border?: border_style, corner_radius?: number,
  padding?: edges, shadow?: shadow_style, width?: dimension, height?: dimension,
} { children }
```

## Content Components

```pepl
// Text display
Text {
  value: string, size?: "small"|"body"|"title"|"heading"|"display",
  weight?: "normal"|"medium"|"bold", color?: color,
  align?: "start"|"center"|"end", max_lines?: number,
  overflow?: "clip"|"ellipsis"|"wrap",
}

// Formatted number
NumberDisplay {
  value: number, format?: "decimal"|"integer"|"percentage"|"currency",
  decimals?: number, size?: "small"|"body"|"title"|"heading"|"display",
}

// Icon (built-in set — Material Symbols: "check", "close", "add",
//  "delete", "search", "settings", etc. Unknown names render placeholder square)
Icon { name: string, size?: number, color?: color }

// Image (source: HTTPS URL or data URI. No local file paths — sandboxed)
Image {
  source: string, fit?: "contain"|"cover"|"fill"|"none",
  width?: dimension, height?: dimension, alt?: string,
}

// Progress indicators
ProgressBar { value: number, color?: color, background?: color, height?: number }
// value is normalized: 0.0 = empty, 1.0 = full. Values outside 0.0-1.0 are clamped.
ProgressCircle { value: number, size?: number, stroke?: number, color?: color }
// Same normalized 0.0-1.0 semantics as ProgressBar.

// Badge/chip
Badge { label: string, color?: color, variant?: "filled"|"outlined"|"subtle" }
```

## Interactive Components

```pepl
// Button
Button {
  label: string, on_tap: () -> nil,
  variant?: "filled"|"outlined"|"text",
  icon?: string, disabled?: bool, loading?: bool,
}

// Text input
TextInput {
  value: string, on_change: (string) -> nil,
  placeholder?: string, label?: string,
  keyboard?: "text"|"number"|"email"|"phone"|"url",
  max_length?: number, multiline?: bool,
}

// Number input with increment/decrement
NumberInput {
  value: number, on_change: (number) -> nil,
  min?: number, max?: number, step?: number, label?: string,
}

// Toggle, checkbox, slider
Toggle { value: bool, on_change: (bool) -> nil, label?: string }
Checkbox { checked: bool, on_change: (bool) -> nil, label?: string }
Slider {
  value: number, on_change: (number) -> nil,
  min: number, max: number, step?: number, label?: string,
}

// Picker / dropdown
Picker {
  value: string, options: list<{label: string, value: string}>,
  on_change: (string) -> nil, label?: string,
}

// Date picker (values are Unix milliseconds)
DatePicker {
  value: number, on_change: (number) -> nil,
  min_date?: number, max_date?: number, label?: string,
}

// Color picker
ColorPicker { value: color, on_change: (color) -> nil, label?: string }
```

## List & Data Components

```pepl
// Scrolling list
ScrollList {
  items: list<T>, render: (T, number) -> Surface,
  key: (T) -> string, on_reorder?: (list<T>) -> nil, dividers?: bool,
}

// Swipeable list item
// swipe_action = { label: string, color: color, on_swipe: () -> nil }
Swipeable {
  leading_actions?: list<swipe_action>,
  trailing_actions?: list<swipe_action>,
} { child }

// Table (degrades to list on small screens)
Table {
  items: list<T>,
  columns: list<{header: string, render: (T) -> Surface,
                  width?: dimension, sortable?: bool}>,
}

// Card container
Card { on_tap?: () -> nil, elevation?: number } { children }
```

## Navigation Components

```pepl
// Tab bar
TabBar {
  selected: string,
  tabs: list<{id: string, label: string, icon?: string}>,
  on_select: (string) -> action,
}

// Bottom navigation (mobile) / sidebar (desktop)
Navigation {
  selected: string,
  items: list<{id: string, label: string, icon: string}>,
  on_select: (string) -> action,
}

// Top app bar
AppBar { title?: string, leading?: Surface, trailing?: list<Surface> }

// Modal/dialog overlay
Modal { visible: bool, on_dismiss: () -> nil, title?: string } { content }

// Bottom sheet / sidebar panel
Sheet {
  visible: bool, on_dismiss: () -> nil,
  height?: "auto"|"half"|"full",
} { content }
```

## Feedback Components

```pepl
Toast {
  message: string, duration?: number,
  type?: "info"|"success"|"warning"|"error",
}
Spinner { size?: number, color?: color }
EmptyState {
  icon?: string, title: string,
  description?: string, action?: {label: string, on_tap: () -> nil},
}
Skeleton { width?: dimension, height?: dimension, shape?: "rect"|"circle" }
```

## Canvas & Chart Components (Phase 0B+)

```pepl
// 2D drawing canvas
Canvas { width: number, height: number, on_draw: (context: canvas_context) -> nil }

// Canvas context operations
context.fill_rect(x, y, w, h, color)
context.stroke_rect(x, y, w, h, color, width)
context.fill_circle(cx, cy, radius, color)
context.stroke_circle(cx, cy, radius, color, width)
context.draw_line(x1, y1, x2, y2, color, width)
context.draw_text(text, x, y, size, color)
context.draw_path(points: list<{x, y}>, color, width, closed)

// Chart (built on canvas)
Chart {
  type: "bar"|"line"|"pie"|"donut",
  data: list<{label: string, value: number, color?: color}>,
  width?: dimension, height?: dimension,
  show_labels?: bool, show_legend?: bool,
}
```

---

## Animation System (Phase 0C+)

Animations are declarative and deterministic (no frame-based rendering):

```pepl
// Transition on state change
animate(property: string,         // "opacity", "offset_y", "scale", "color"
        duration: number,         // milliseconds
        curve?: "linear"|"ease_in"|"ease_out"|"ease_in_out"|"spring",
        delay?: number)

// Usage
Container {
  opacity: if(visible, 1.0, 0.0),
  animate: [animate("opacity", 200, curve: "ease_out")],
} { Text { value: "Hello" } }

// Staggered list animation
ScrollList {
  items: my_items,
  render: (item, index) -> {
    Container {
      offset_y: 0,
      animate: [animate("offset_y", 300, delay: index * 50, curve: "ease_out")],
    } { Text { value: item.name } }
  },
}
```

---

## Accessibility

All components have built-in accessibility support:

```pepl
accessible(
  label: string,          // Screen reader label
  hint?: string,          // Additional context
  role?: "button"|"checkbox"|"slider"|"heading"|"image"|"link",
  value?: string,         // Current value for screen readers
  live_region?: "polite"|"assertive"   // For dynamic content updates
)

// Usage
Button {
  label: "Add Water",
  on_tap: add_water(250),
  accessible: accessible(
    label: "Add 250 milliliters of water",
    hint: "Double tap to add water to today's intake",
  ),
}
```
