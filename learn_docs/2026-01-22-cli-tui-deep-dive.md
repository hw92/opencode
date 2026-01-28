# OpenCode CLI/TUI Architecture Deep Dive

[TOC]

A comprehensive guide to building terminal user interfaces using OpenCode's architecture - featuring **SolidJS** + **OpenTUI** (not Ink React!).

---

## Important Correction: It's NOT Ink React

OpenCode's TUI uses:
- **SolidJS** - A reactive UI framework (similar to React but with fine-grained reactivity)
- **@opentui/solid** - A custom terminal renderer that renders SolidJS to the terminal
- **@opentui/core** - Core primitives for terminal rendering

**Why not Ink React?**
- Ink uses React's reconciler which can be slow for rapid terminal updates
- SolidJS has fine-grained reactivity (no virtual DOM diffing)
- OpenTUI is optimized specifically for terminal rendering with 60fps target

---

## Architecture Overview

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        CLI/TUI ARCHITECTURE                                 │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  src/index.ts                                                               │
│       │                                                                     │
│       ▼                                                                     │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │  YARGS CLI                                                          │   │
│  │  • Command parsing                                                  │   │
│  │  • Option handling                                                  │   │
│  │  • Middleware (logging, env setup)                                  │   │
│  └─────────────────────┬───────────────────────────────────────────────┘   │
│                        │                                                   │
│    ┌───────────────────┼───────────────────┬───────────────────┐           │
│    ▼                   ▼                   ▼                   ▼           │
│ ┌──────────┐    ┌──────────┐        ┌──────────┐        ┌──────────┐      │
│ │   auth   │    │   run    │        │  serve   │        │   tui    │      │
│ │ (prompts)│    │ (SDK)    │        │ (server) │        │ (OpenTUI)│      │
│ └──────────┘    └──────────┘        └──────────┘        └────┬─────┘      │
│                                                              │             │
│                                     ┌────────────────────────┘             │
│                                     ▼                                      │
│                       ┌─────────────────────────────────────┐              │
│                       │      SOLID-JS + OPENTUI             │              │
│                       ├─────────────────────────────────────┤              │
│                       │  Contexts (state management)        │              │
│                       │  Components (reusable UI)           │              │
│                       │  Routes (screen-level)              │              │
│                       │  UI Primitives (dialog, toast)      │              │
│                       └─────────────────────────────────────┘              │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## Part 1: CLI Layer (Yargs)

### Main Entry Point

**File:** `packages/opencode/src/index.ts`

```typescript
import yargs from "yargs"
import { hideBin } from "yargs/helpers"

const cli = yargs(hideBin(process.argv))
  .parserConfiguration({ "populate--": true })  // Pass remaining args via "--"
  .scriptName("opencode")
  .wrap(100)
  .help("help", "show help")
  .alias("help", "h")
  .version("version", "show version number", Installation.VERSION)
  .middleware(async (opts) => {
    // Initialize logging, set env vars
    await Log.init({ print: process.argv.includes("--print-logs") })
    process.env.AGENT = "1"
    process.env.OPENCODE = "1"
  })
  .usage("\n" + UI.logo())
  .completion("completion", "generate shell completion script")
  // Register all commands
  .command(AuthCommand)
  .command(RunCommand)
  .command(ServeCommand)
  .command(McpCommand)
  // ... more commands
  .strict()

await cli.parse()
```

### Command Definition Pattern

**File:** `packages/opencode/src/cli/cmd/cmd.ts`

```typescript
import type { CommandModule } from "yargs"

type WithDoubleDash<T> = T & { "--"?: string[] }

export function cmd<T, U>(input: CommandModule<T, WithDoubleDash<U>>) {
  return input
}
```

This helper ensures consistent command typing with support for `--` passthrough.

### Example Command: Auth

**File:** `packages/opencode/src/cli/cmd/auth.ts`

```typescript
import { cmd } from "./cmd"
import * as prompts from "@clack/prompts"  // Beautiful CLI prompts

export const AuthCommand = cmd({
  command: "auth",
  describe: "manage credentials",
  builder: (yargs) =>
    yargs
      .command(AuthLoginCommand)
      .command(AuthLogoutCommand)
      .command(AuthListCommand)
      .demandCommand(),
  async handler() {},
})

export const AuthLoginCommand = cmd({
  command: "login [url]",
  describe: "log in to a provider",
  builder: (yargs) =>
    yargs.positional("url", {
      describe: "opencode auth provider",
      type: "string",
    }),
  async handler(args) {
    prompts.intro("Add credential")

    const provider = await prompts.autocomplete({
      message: "Select provider",
      maxItems: 8,
      options: [
        { label: "OpenCode", value: "opencode", hint: "recommended" },
        { label: "Anthropic", value: "anthropic" },
        // ...
      ],
    })

    if (prompts.isCancel(provider)) throw new UI.CancelledError()

    const key = await prompts.password({
      message: "Enter your API key",
      validate: (x) => (x && x.length > 0 ? undefined : "Required"),
    })

    await Auth.set(provider, { type: "api", key })
    prompts.outro("Done")
  },
})
```

### CLI-Only UI Helpers

**File:** `packages/opencode/src/cli/ui.ts`

```typescript
export namespace UI {
  // ANSI color codes for terminal styling
  export const Style = {
    TEXT_HIGHLIGHT: "\x1b[96m",
    TEXT_DIM: "\x1b[90m",
    TEXT_NORMAL: "\x1b[0m",
    TEXT_WARNING: "\x1b[93m",
    TEXT_DANGER: "\x1b[91m",
    TEXT_SUCCESS: "\x1b[92m",
    // ... bold variants
  }

  export function println(...message: string[]) {
    Bun.stderr.write(message.join(" ") + EOL)
  }

  export function error(message: string) {
    println(Style.TEXT_DANGER_BOLD + "Error: " + Style.TEXT_NORMAL + message)
  }
}
```

---

## Part 2: TUI Framework (SolidJS + OpenTUI)

### The TUI Entry Point

**File:** `packages/opencode/src/cli/cmd/tui/app.tsx`

```typescript
import { render, useKeyboard, useRenderer, useTerminalDimensions } from "@opentui/solid"
import { Switch, Match, createEffect, ErrorBoundary, createSignal } from "solid-js"

// The main entry function
export function tui(input: {
  url: string
  args: Args
  directory?: string
  fetch?: typeof fetch
  events?: EventSource
  onExit?: () => Promise<void>
}) {
  return new Promise<void>(async (resolve) => {
    const mode = await getTerminalBackgroundColor()  // Detect dark/light mode!

    render(
      () => (
        <ErrorBoundary fallback={(error, reset) => <ErrorComponent error={error} reset={reset} />}>
          {/* Provider tree - state management */}
          <ArgsProvider {...input.args}>
            <ExitProvider onExit={onExit}>
              <KVProvider>
                <ToastProvider>
                  <RouteProvider>
                    <SDKProvider url={input.url}>
                      <SyncProvider>
                        <ThemeProvider mode={mode}>
                          <LocalProvider>
                            <KeybindProvider>
                              <DialogProvider>
                                <CommandProvider>
                                  <App />  {/* Main app component */}
                                </CommandProvider>
                              </DialogProvider>
                            </KeybindProvider>
                          </LocalProvider>
                        </ThemeProvider>
                      </SyncProvider>
                    </SDKProvider>
                  </RouteProvider>
                </ToastProvider>
              </KVProvider>
            </ExitProvider>
          </ArgsProvider>
        </ErrorBoundary>
      ),
      {
        targetFps: 60,
        exitOnCtrlC: false,
        useKittyKeyboard: {},  // Enhanced keyboard support
      },
    )
  })
}
```

### Key Insight: Terminal Background Detection

OpenTUI can detect if the terminal is dark or light mode:

```typescript
async function getTerminalBackgroundColor(): Promise<"dark" | "light"> {
  if (!process.stdin.isTTY) return "dark"

  return new Promise((resolve) => {
    const handler = (data: Buffer) => {
      const str = data.toString()
      const match = str.match(/\x1b]11;([^\x07\x1b]+)/)
      if (match) {
        // Parse RGB from color string and calculate luminance
        const luminance = (0.299 * r + 0.587 * g + 0.114 * b) / 255
        resolve(luminance > 0.5 ? "light" : "dark")
      }
    }

    process.stdin.setRawMode(true)
    process.stdin.on("data", handler)
    process.stdout.write("\x1b]11;?\x07")  // Query background color

    setTimeout(() => resolve("dark"), 1000)  // Fallback
  })
}
```

---

## Part 3: Context System (State Management)

### Context Factory Pattern

**File:** `packages/opencode/src/cli/cmd/tui/context/helper.tsx`

```typescript
import { createContext, Show, useContext, type ParentProps } from "solid-js"

export function createSimpleContext<T, Props extends Record<string, any>>(input: {
  name: string
  init: ((input: Props) => T) | (() => T)
}) {
  const ctx = createContext<T>()

  return {
    provider: (props: ParentProps<Props>) => {
      const init = input.init(props)
      return (
        <Show when={init.ready === undefined || init.ready === true}>
          <ctx.Provider value={init}>{props.children}</ctx.Provider>
        </Show>
      )
    },
    use() {
      const value = useContext(ctx)
      if (!value) throw new Error(`${input.name} context must be used within a provider`)
      return value
    },
  }
}
```

### Example: Route Context

**File:** `packages/opencode/src/cli/cmd/tui/context/route.tsx`

```typescript
import { createStore } from "solid-js/store"
import { createSimpleContext } from "./helper"

export type HomeRoute = { type: "home"; initialPrompt?: PromptInfo }
export type SessionRoute = { type: "session"; sessionID: string }
export type Route = HomeRoute | SessionRoute

export const { use: useRoute, provider: RouteProvider } = createSimpleContext({
  name: "Route",
  init: () => {
    const [store, setStore] = createStore<Route>({ type: "home" })

    return {
      get data() { return store },
      navigate(route: Route) {
        console.log("navigate", route)
        setStore(route)
      },
    }
  },
})
```

**Usage:**
```typescript
function MyComponent() {
  const route = useRoute()

  // Navigate programmatically
  route.navigate({ type: "session", sessionID: "abc123" })

  // Access current route
  if (route.data.type === "home") { /* ... */ }
}
```

### SDK Context (Backend Communication)

**File:** `packages/opencode/src/cli/cmd/tui/context/sdk.tsx`

```typescript
import { createOpencodeClient } from "@opencode-ai/sdk/v2"
import { createGlobalEmitter } from "@solid-primitives/event-bus"

export const { use: useSDK, provider: SDKProvider } = createSimpleContext({
  name: "SDK",
  init: (props: { url: string; fetch?: typeof fetch }) => {
    const sdk = createOpencodeClient({
      baseUrl: props.url,
      fetch: props.fetch,
    })

    // Event emitter for SSE events
    const emitter = createGlobalEmitter<{
      [key in Event["type"]]: Extract<Event, { type: key }>
    }>()

    // Batched event processing (16ms batches for 60fps)
    let queue: Event[] = []
    const handleEvent = (event: Event) => {
      queue.push(event)
      if (queue.length === 1) {
        setTimeout(() => {
          batch(() => {
            for (const event of queue) {
              emitter.emit(event.type, event)
            }
          })
          queue = []
        }, 16)
      }
    }

    onMount(async () => {
      const events = await sdk.event.subscribe()
      for await (const event of events.stream) {
        handleEvent(event)
      }
    })

    return { client: sdk, event: emitter, url: props.url }
  },
})
```

### Keybind Context (Vim-like Leader Key)

**File:** `packages/opencode/src/cli/cmd/tui/context/keybind.tsx`

```typescript
export const { use: useKeybind, provider: KeybindProvider } = createSimpleContext({
  name: "Keybind",
  init: () => {
    const sync = useSync()
    const [store, setStore] = createStore({ leader: false })
    const renderer = useRenderer()

    let focus: Renderable | null
    let timeout: NodeJS.Timeout

    function leader(active: boolean) {
      if (active) {
        setStore("leader", true)
        focus = renderer.currentFocusedRenderable
        focus?.blur()

        // Auto-exit leader mode after 2 seconds
        timeout = setTimeout(() => {
          if (store.leader) leader(false)
        }, 2000)
        return
      }

      if (focus) focus.focus()
      setStore("leader", false)
    }

    // Global keyboard handler
    useKeyboard(async (evt) => {
      if (!store.leader && result.match("leader", evt)) {
        leader(true)
        return
      }
      if (store.leader && evt.name) {
        setImmediate(() => leader(false))
      }
    })

    return {
      get leader() { return store.leader },
      match(key: keyof KeybindsConfig, evt: ParsedKey) {
        const keybind = keybinds()[key]
        if (!keybind) return false
        // Check if event matches configured keybind
        return Keybind.match(keybind[0], Keybind.fromParsedKey(evt, store.leader))
      },
      print(key: keyof KeybindsConfig) {
        // Format keybind for display (e.g., "ctrl+x a")
        return Keybind.toString(keybinds()[key]?.[0])
      },
    }
  },
})
```

---

## Part 4: UI Components

### OpenTUI Primitives

OpenTUI provides terminal-native components:

| Primitive | Description |
|-----------|-------------|
| `<box>` | Flexbox container with full layout support |
| `<text>` | Text with styling (colors, bold, underline) |
| `<textarea>` | Multi-line text input with extmarks |
| `<scrollbox>` | Scrollable container |
| `<spinner>` | Loading indicator |

### Dialog Component

**File:** `packages/opencode/src/cli/cmd/tui/ui/dialog.tsx`

```typescript
export function Dialog(props: ParentProps<{
  size?: "medium" | "large"
  onClose: () => void
}>) {
  const dimensions = useTerminalDimensions()
  const { theme } = useTheme()

  return (
    <box
      onMouseUp={() => props.onClose?.()}  // Click outside to close
      width={dimensions().width}
      height={dimensions().height}
      alignItems="center"
      position="absolute"
      paddingTop={dimensions().height / 4}
      left={0}
      top={0}
      backgroundColor={RGBA.fromInts(0, 0, 0, 150)}  // Semi-transparent overlay
    >
      <box
        onMouseUp={(e) => e.stopPropagation()}  // Prevent close on dialog click
        width={props.size === "large" ? 80 : 60}
        maxWidth={dimensions().width - 2}
        backgroundColor={theme.backgroundPanel}
        paddingTop={1}
      >
        {props.children}
      </box>
    </box>
  )
}

// Dialog context for programmatic control
function init() {
  const [store, setStore] = createStore({
    stack: [] as { element: JSX.Element; onClose?: () => void }[],
  })

  useKeyboard((evt) => {
    if (evt.name === "escape" && store.stack.length > 0) {
      const current = store.stack.at(-1)!
      current.onClose?.()
      setStore("stack", store.stack.slice(0, -1))
      evt.preventDefault()
    }
  })

  return {
    clear() {
      for (const item of store.stack) item.onClose?.()
      setStore("stack", [])
    },
    replace(element: JSX.Element, onClose?: () => void) {
      for (const item of store.stack) item.onClose?.()
      setStore("stack", [{ element, onClose }])
    },
    get stack() { return store.stack },
  }
}

export const DialogProvider = ...
export const useDialog = ...
```

### Toast Component

**File:** `packages/opencode/src/cli/cmd/tui/ui/toast.tsx`

```typescript
export function Toast() {
  const toast = useToast()
  const { theme } = useTheme()
  const dimensions = useTerminalDimensions()

  return (
    <Show when={toast.currentToast}>
      {(current) => (
        <box
          position="absolute"
          top={2}
          right={2}
          maxWidth={Math.min(60, dimensions().width - 6)}
          paddingLeft={2}
          paddingRight={2}
          paddingTop={1}
          paddingBottom={1}
          backgroundColor={theme.backgroundPanel}
          borderColor={theme[current().variant]}  // Colored border by variant
          border={["left", "right"]}
        >
          <Show when={current().title}>
            <text attributes={TextAttributes.BOLD} fg={theme.text}>
              {current().title}
            </text>
          </Show>
          <text fg={theme.text} wrapMode="word" width="100%">
            {current().message}
          </text>
        </box>
      )}
    </Show>
  )
}

function init() {
  const [store, setStore] = createStore({ currentToast: null as ToastOptions | null })
  let timeoutHandle: NodeJS.Timeout | null = null

  return {
    show(options: ToastOptions) {
      setStore("currentToast", options)
      if (timeoutHandle) clearTimeout(timeoutHandle)
      timeoutHandle = setTimeout(() => {
        setStore("currentToast", null)
      }, options.duration ?? 3000)
    },
    error: (err: any) => {
      toast.show({
        variant: "error",
        message: err instanceof Error ? err.message : "An unknown error occurred",
      })
    },
    get currentToast() { return store.currentToast },
  }
}
```

---

## Part 5: Routes (Screen-Level Components)

### Home Route

**File:** `packages/opencode/src/cli/cmd/tui/routes/home.tsx`

```typescript
export function Home() {
  const sync = useSync()
  const { theme } = useTheme()
  const route = useRouteData("home")
  const command = useCommandDialog()

  // Register commands for this route
  command.register(() => [
    {
      title: "Hide tips",
      value: "tips.toggle",
      keybind: "tips_toggle",
      category: "System",
      onSelect: (dialog) => {
        kv.set("tips_hidden", true)
        dialog.clear()
      },
    },
  ])

  return (
    <>
      <box flexGrow={1} justifyContent="center" alignItems="center" gap={1}>
        <Logo />
        <box width="100%" maxWidth={75}>
          <Prompt ref={(r) => (prompt = r)} />
        </box>
        <Show when={showTips()}>
          <Tips />
        </Show>
        <Toast />
      </box>

      {/* Status bar */}
      <box flexDirection="row" paddingLeft={2} paddingRight={2} flexShrink={0}>
        <text fg={theme.textMuted}>{directory()}</text>
        <box flexGrow={1} />
        <text fg={theme.textMuted}>{Installation.VERSION}</text>
      </box>
    </>
  )
}
```

### The Prompt Component (Complex Input)

**File:** `packages/opencode/src/cli/cmd/tui/component/prompt/index.tsx`

This is a sophisticated input component with:
- Extmarks (for file attachments, @mentions, pasted content)
- Autocomplete support
- Keyboard shortcuts
- History navigation
- Image paste support
- External editor integration

```typescript
export function Prompt(props: PromptProps) {
  let input: TextareaRenderable
  let autocomplete: AutocompleteRef

  const keybind = useKeybind()
  const local = useLocal()
  const sdk = useSDK()

  // Extmark styles for different content types
  const fileStyleId = syntax().getStyleId("extmark.file")!
  const agentStyleId = syntax().getStyleId("extmark.agent")!
  const pasteStyleId = syntax().getStyleId("extmark.paste")!

  // Handle paste with image detection
  async function pasteImage(file: { filename?: string; content: string; mime: string }) {
    const virtualText = `[Image ${count + 1}]`
    const extmarkId = input.extmarks.create({
      start: currentOffset,
      end: currentOffset + virtualText.length,
      virtual: true,
      styleId: pasteStyleId,
    })
    // Store file part for submission
    setStore("prompt", "parts", [...parts, { type: "file", mime, url: dataUrl }])
  }

  return (
    <>
      <Autocomplete anchor={() => anchor} input={() => input} />
      <box border={["left"]} borderColor={highlight()}>
        <box paddingLeft={2} paddingRight={2} backgroundColor={theme.backgroundElement}>
          <textarea
            placeholder="Ask anything..."
            onContentChange={() => {
              setStore("prompt", "input", input.plainText)
              autocomplete.onInput(input.plainText)
            }}
            onKeyDown={async (e) => {
              // History navigation
              if (keybind.match("history_previous", e)) {
                const item = history.move(-1, input.plainText)
                if (item) input.setText(item.input)
              }
              // External editor
              if (keybind.match("editor_open", e)) {
                const content = await Editor.open({ value: input.plainText })
                if (content) input.setText(content)
              }
            }}
            onSubmit={submit}
            onPaste={async (event) => {
              // Image detection
              const file = Bun.file(filepath)
              if (file.type.startsWith("image/")) {
                event.preventDefault()
                await pasteImage({ filename: file.name, mime: file.type, content })
              }
            }}
          />
          <box flexDirection="row">
            <text fg={highlight()}>{local.agent.current().name}</text>
            <text fg={theme.text}>{local.model.parsed().model}</text>
          </box>
        </box>
      </box>
    </>
  )
}
```

---

## Part 6: Key Design Patterns

### Pattern 1: Command Registration

Components can register commands that appear in the command palette:

```typescript
const command = useCommandDialog()

command.register(() => [
  {
    title: "Switch theme",
    value: "theme.switch",
    keybind: "theme_list",
    category: "System",
    onSelect: () => {
      dialog.replace(() => <DialogThemeList />)
    },
  },
  {
    title: "Exit the app",
    value: "app.exit",
    onSelect: () => exit(),
    category: "System",
  },
])
```

### Pattern 2: Event-Driven Updates

The SDK context bridges SSE events to SolidJS reactivity:

```typescript
sdk.event.on(SessionApi.Event.Deleted.type, (evt) => {
  if (route.data.type === "session" && route.data.sessionID === evt.properties.info.id) {
    route.navigate({ type: "home" })
    toast.show({ variant: "info", message: "Session was deleted" })
  }
})

sdk.event.on(TuiEvent.ToastShow.type, (evt) => {
  toast.show({
    title: evt.properties.title,
    message: evt.properties.message,
    variant: evt.properties.variant,
  })
})
```

### Pattern 3: Keyboard-First Design

Everything is keyboard accessible with configurable keybinds:

```typescript
useKeyboard((evt) => {
  // Leader key mode (like vim's leader)
  if (!store.leader && result.match("leader", evt)) {
    leader(true)
    return
  }

  // Escape closes dialogs
  if (evt.name === "escape" && store.stack.length > 0) {
    const current = store.stack.at(-1)!
    current.onClose?.()
    setStore("stack", store.stack.slice(0, -1))
  }
})
```

### Pattern 4: Provider Tree

State is organized in a clear provider hierarchy:

```
ArgsProvider          → CLI arguments
└── ExitProvider      → Exit handling
    └── KVProvider    → Key-value storage
        └── ToastProvider   → Notifications
            └── RouteProvider    → Navigation
                └── SDKProvider      → Backend SDK
                    └── SyncProvider     → Data sync
                        └── ThemeProvider    → Theming
                            └── LocalProvider    → Local state
                                └── KeybindProvider  → Keybinds
                                    └── DialogProvider   → Modals
                                        └── CommandProvider  → Commands
                                            └── <App />
```

---

## Building Your Own CLI/TUI

### Step 1: CLI with Yargs

```typescript
// my-cli/src/index.ts
import yargs from "yargs"
import { hideBin } from "yargs/helpers"

const cli = yargs(hideBin(process.argv))
  .scriptName("my-tool")
  .command({
    command: "$0",  // Default command
    describe: "Start the TUI",
    handler: async () => {
      const { tui } = await import("./tui")
      await tui()
    },
  })
  .command({
    command: "run [task]",
    describe: "Run a task headlessly",
    handler: async (args) => {
      // Non-TUI mode
    },
  })

await cli.parse()
```

### Step 2: TUI with SolidJS + OpenTUI

```typescript
// my-cli/src/tui.tsx
import { render, useTerminalDimensions } from "@opentui/solid"
import { createSignal, Show } from "solid-js"

export function tui() {
  return new Promise<void>((resolve) => {
    render(
      () => <App onExit={resolve} />,
      { targetFps: 60, exitOnCtrlC: false }
    )
  })
}

function App(props: { onExit: () => void }) {
  const dims = useTerminalDimensions()
  const [route, setRoute] = createSignal<"home" | "detail">("home")

  return (
    <box width={dims().width} height={dims().height} backgroundColor="#1a1a1a">
      <Show when={route() === "home"} fallback={<DetailView />}>
        <HomeView onNavigate={() => setRoute("detail")} />
      </Show>
    </box>
  )
}
```

### Step 3: Add State Management

```typescript
// my-cli/src/context/data.tsx
import { createContext, useContext, ParentProps } from "solid-js"
import { createStore } from "solid-js/store"

const DataContext = createContext<ReturnType<typeof createDataContext>>()

function createDataContext() {
  const [store, setStore] = createStore({
    items: [] as Item[],
    loading: false,
  })

  return {
    get items() { return store.items },
    get loading() { return store.loading },
    async fetch() {
      setStore("loading", true)
      const items = await api.getItems()
      setStore({ items, loading: false })
    },
  }
}

export function DataProvider(props: ParentProps) {
  const value = createDataContext()
  return <DataContext.Provider value={value}>{props.children}</DataContext.Provider>
}

export function useData() {
  const ctx = useContext(DataContext)
  if (!ctx) throw new Error("useData must be within DataProvider")
  return ctx
}
```

---

## Key Dependencies

| Package | Purpose |
|---------|---------|
| `yargs` | CLI argument parsing |
| `@clack/prompts` | Beautiful CLI prompts (non-TUI mode) |
| `solid-js` | Reactive UI framework |
| `solid-js/store` | State management |
| `@opentui/solid` | Terminal renderer |
| `@opentui/core` | Terminal primitives |
| `@solid-primitives/event-bus` | Event emitter for SSE |

---

## Summary

| Component | Technology | Purpose |
|-----------|------------|---------|
| CLI Entry | Yargs | Command parsing |
| CLI Prompts | @clack/prompts | Interactive prompts |
| TUI Framework | SolidJS + OpenTUI | Terminal UI |
| State | SolidJS stores + contexts | State management |
| Events | SSE + solid-primitives | Real-time updates |
| Styling | Theme context + RGBA | Theming |

OpenCode's CLI/TUI architecture demonstrates that modern terminal applications can be:
- **Reactive** - Fine-grained updates with SolidJS
- **Beautiful** - Rich styling with OpenTUI
- **Keyboard-first** - Vim-like leader keys
- **Fast** - 60fps rendering target
- **Modular** - Clear separation of concerns

---

*Written by Claude (Opus 4.5) | 2026-01-22 16:30 PST*
