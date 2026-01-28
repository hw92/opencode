# OpenCode UI Package Deep Dive

[TOC]

A comprehensive study of the shared UI component library built on Solid.js and Kobalte.

---

## Overview

The UI package (`@opencode-ai/ui`) provides **45+ components** with a sophisticated theming system. Built on Solid.js with Kobalte for accessible primitives.

**Key Features:**
- Kobalte-based accessible components
- 16+ pre-built themes
- OKLCH color system
- Code/diff rendering via Pierre
- Zero runtime CSS framework dependency

---

## Component Inventory

### Primitives

| Component | Description |
|-----------|-------------|
| **Button** | Variants: primary/secondary/ghost, Sizes: small/normal/large |
| **TextField** | Label, description, error, copyable, multiline |
| **Checkbox** | Custom indicator with label |
| **RadioGroup** | Mutually exclusive selections |
| **Switch** | Toggle switch |
| **Select** | Dropdown with grouping |
| **Avatar** | Image/fallback with colors |

### Composites

| Component | Description |
|-----------|-------------|
| **Dialog** | Modal with title, description, actions |
| **Accordion** | Expandable sections |
| **Tabs** | Tabbed interface |
| **Card** | Container with variants |
| **DropdownMenu** | Full menu with submenus |
| **Tooltip** | Popover with keyboard hints |
| **Popover** | Generic positioning |
| **HoverCard** | Hover-triggered card |

### Content/Display

| Component | Description |
|-----------|-------------|
| **Code** | Syntax-highlighted code (Pierre) |
| **Diff** | Side-by-side diff viewer |
| **Markdown** | Rendered markdown with caching |
| **Icon** | 66+ embedded SVG icons |
| **FileIcon** | File type icons |
| **ProviderIcon** | AI provider logos |
| **Spinner** | Loading indicator |

### Specialized

| Component | Description |
|-----------|-------------|
| **SessionTurn** | User-assistant conversation turn |
| **SessionMessageRail** | Message navigation |
| **SessionReview** | Diff display |
| **List** | Virtualized list |
| **Toast** | Notifications |
| **Keybind** | Keyboard shortcut display |

---

## Theming System

### Color Architecture

```typescript
// OKLCH color space for perceptual accuracy
// Seed colors generate scales
const seeds = {
  neutral, primary, success, warning, error,
  info, interactive, diffAdd, diffDelete
}

// Each seed generates 12-step scale
generateScale(seed) // → [1..12] variants
generateAlphaScale(seed) // → transparency variants
```

### Pre-built Themes (16+)

- OC-1 (default)
- Tokyo Night
- Dracula
- Monokai
- Solarized (Light/Dark)
- Nord
- Catppuccin
- Ayu
- OneDark Pro
- Shades of Purple
- Nightowl
- Vesper
- And more...

### Theme Provider

```tsx
import { ThemeProvider, useTheme } from "@opencode-ai/ui/theme"

<ThemeProvider defaultTheme="oc-1">
  <App />
</ThemeProvider>

// In component
const theme = useTheme()
theme.setTheme("dracula")
theme.setColorScheme("dark")
```

### CSS Variables

```css
/* Typography */
--font-size-xs, --font-size-sm, --font-size-base, ...
--font-weight-normal, --font-weight-medium, --font-weight-bold

/* Spacing */
--spacing: 0.25rem  /* Base unit */

/* Colors (contextual) */
--color-background, --color-surface
--color-text-primary, --color-text-secondary
--color-border, --color-input-border
--color-button-primary, --color-button-secondary
```

---

## Component Patterns

### Kobalte Wrapper Pattern

```tsx
import { Button as Kobalte } from "@kobalte/core/button"

export function Button(props: ButtonProps) {
  const [split, rest] = splitProps(props, ["variant", "size"])

  return (
    <Kobalte
      {...rest}
      data-component="button"
      data-size={split.size || "normal"}
      data-variant={split.variant || "secondary"}
    >
      {props.children}
    </Kobalte>
  )
}
```

### Compound Components

```tsx
export const Accordion = Object.assign(AccordionRoot, {
  Item: AccordionItem,
  Header: AccordionHeader,
  Trigger: AccordionTrigger,
  Content: AccordionContent,
})

// Usage
<Accordion>
  <Accordion.Item>
    <Accordion.Header>
      <Accordion.Trigger>Title</Accordion.Trigger>
    </Accordion.Header>
    <Accordion.Content>Content</Accordion.Content>
  </Accordion.Item>
</Accordion>
```

### Data Attributes for Styling

```tsx
// Components use semantic data attributes
<button
  data-component="button"
  data-slot="button-label"
  data-variant="primary"
  data-size="normal"
  data-state="open"  // From Kobalte
>
```

```css
/* CSS targeting */
[data-component="button"][data-variant="primary"] {
  background: var(--color-primary);
}
```

---

## Styling Approach

### CSS Files (~1,881 lines)

```
src/styles/
├── base.css       # Reset/normalize (388 lines)
├── colors.css     # Color tokens (592 lines)
├── theme.css      # Component styles (588 lines)
├── utilities.css  # Helpers (131 lines)
├── animations.css # Keyframes (130 lines)
└── index.css      # Entry point
```

### No Tailwind in Core

- Uses semantic CSS variables
- Data attributes for variants
- Vite builds with Solid plugin

---

## Accessibility

### Built-in via Kobalte

- ARIA labels and descriptions
- Keyboard navigation (Tab, Enter, Escape, Arrows)
- Focus management
- Screen reader support

### Semantic HTML

```tsx
// Proper heading hierarchy
// Form labels with data-slot="input-label"
// Error messages with data-slot="input-error"
// Descriptions with data-slot="input-description"
```

### Visual Accessibility

```css
.sr-only { /* Screen reader only */ }

/* Custom focus rings */
[data-component]:focus-visible {
  outline: 2px solid var(--color-focus);
}

/* High contrast via light-dark() */
```

---

## Code/Diff Rendering (Pierre)

### Architecture

```
src/pierre/
├── index.ts      # Public API
└── worker.ts     # Web Worker processing
```

### Features

- Shadow DOM for isolated styling
- Web Workers for heavy computation
- Syntax highlighting via Shiki
- Markdown via Marked + KaTeX
- Line selection and annotation
- Mobile-responsive

### Usage

```tsx
import { Code, Diff } from "@opencode-ai/ui"

<Code
  file={{ path: "main.ts", contents: "..." }}
  language="typescript"
/>

<Diff
  before={oldContent}
  after={newContent}
  diffStyle="split"  // or "unified"
/>
```

---

## Package Exports

```typescript
// Components
import { Button, Dialog, TextField } from "@opencode-ai/ui"

// Specific component
import { Button } from "@opencode-ai/ui/button"

// Theme
import { ThemeProvider, useTheme } from "@opencode-ai/ui/theme"

// Pierre (code/diff)
import { Code, Diff } from "@opencode-ai/ui/pierre"

// Styles
import "@opencode-ai/ui/styles/index.css"

// Fonts
import "@opencode-ai/ui/fonts/inter.css"
```

---

## Dependencies

| Package | Purpose |
|---------|---------|
| `@kobalte/core` | Headless UI |
| `solid-js` | Reactive framework |
| `@solid-primitives/*` | Media queries, resize |
| `@pierre/diffs` | Diff rendering |
| `shiki` | Syntax highlighting |
| `marked` | Markdown parsing |
| `dompurify` | HTML sanitization |
| `katex` | Math rendering |

---

## Usage in App/Desktop

```tsx
import { Button, Dialog, TextField } from "@opencode-ai/ui"
import { ThemeProvider } from "@opencode-ai/ui/theme"
import "@opencode-ai/ui/styles/index.css"

function App() {
  return (
    <ThemeProvider defaultTheme="oc-1">
      <Dialog>
        <Dialog.Trigger>
          <Button variant="primary">Open</Button>
        </Dialog.Trigger>
        <Dialog.Content>
          <TextField label="Name" />
        </Dialog.Content>
      </Dialog>
    </ThemeProvider>
  )
}
```

---

## Key Design Patterns

| Pattern | Purpose |
|---------|---------|
| **Kobalte Wrapper** | Behavior from Kobalte, styling via data attrs |
| **Compound Components** | Flexible composition |
| **Data Attributes** | Semantic CSS targeting |
| **OKLCH Colors** | Perceptually uniform |
| **Shadow DOM** | Isolated code rendering |
| **Web Workers** | Non-blocking processing |

---

## Key Files

| File | Purpose |
|------|---------|
| `src/components/*.tsx` | 45+ components |
| `src/theme/` | Theme system |
| `src/pierre/` | Code/diff rendering |
| `src/styles/` | CSS files |
| `src/assets/` | Fonts, icons |

---

*Written by Claude (Opus 4.5) | 2026-01-22 17:00 PST*
