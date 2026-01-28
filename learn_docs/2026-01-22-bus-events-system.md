# OpenCode Bus/Events System: Deep Dive

[TOC]

A comprehensive study of OpenCode's event-driven architecture - the pub/sub system that enables loose coupling between components.

---

## Overview

OpenCode uses a **publish-subscribe (pub/sub)** event bus for component communication. This pattern provides:

- **Loose coupling** - Publishers don't know about subscribers
- **Extensibility** - Easy to add new listeners without modifying publishers
- **Type safety** - Events are defined with Zod schemas
- **Instance isolation** - Each project instance has its own event bus

---

## Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                        Event Flow                                │
│                                                                  │
│   Publisher                    Bus                  Subscribers  │
│   ─────────                    ───                  ───────────  │
│                                                                  │
│   Session.create()  ─────▶  Bus.publish()  ─────▶  ShareNext    │
│                                   │                              │
│   MessageV2.update() ────▶       │         ─────▶  Plugin.event │
│                                   │                              │
│   File.edit()  ──────────▶       │         ─────▶  Format       │
│                                   │                              │
│                                   ▼                              │
│                            GlobalBus.emit()  ───▶  SSE Stream   │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Core Components

```
packages/opencode/src/bus/
├── index.ts        # Main Bus namespace - publish, subscribe, state
├── bus-event.ts    # BusEvent.define() factory and registry
└── global.ts       # GlobalBus EventEmitter for cross-instance events
```

---

## Core Implementation

### BusEvent Definition Factory

**File:** `packages/opencode/src/bus/bus-event.ts` (43 lines)

```typescript
export namespace BusEvent {
  const registry = new Map<string, Definition>()

  export function define<Type extends string, Properties extends ZodType>(
    type: Type,
    properties: Properties
  ) {
    const result = {
      type,
      properties,
    }
    registry.set(type, result)
    return result
  }

  // Generate discriminated union of all events (for API schema)
  export function payloads() {
    return z.discriminatedUnion(
      "type",
      registry.entries().map(([type, def]) =>
        z.object({
          type: z.literal(type),
          properties: def.properties,
        })
      ).toArray()
    )
  }
}
```

**Key features:**
1. **Type-safe event definitions** - Events have a type string and Zod schema
2. **Global registry** - All events are registered for schema generation
3. **Discriminated union** - `payloads()` generates a union type for API validation

### Bus Namespace

**File:** `packages/opencode/src/bus/index.ts` (105 lines)

```typescript
export namespace Bus {
  // Instance-scoped state with cleanup
  const state = Instance.state(
    () => ({
      subscriptions: new Map<string, Subscription[]>(),
    }),
    async (entry) => {
      // On cleanup, notify wildcard subscribers of disposal
      const wildcard = entry.subscriptions.get("*")
      if (wildcard) {
        for (const sub of wildcard) {
          sub({ type: InstanceDisposed.type, properties: { directory: Instance.directory } })
        }
      }
    }
  )

  // Publish an event
  export async function publish<Definition extends BusEvent.Definition>(
    def: Definition,
    properties: z.output<Definition["properties"]>
  ) {
    const payload = { type: def.type, properties }
    log.info("publishing", { type: def.type })

    const pending = []
    // Notify specific subscribers and wildcard subscribers
    for (const key of [def.type, "*"]) {
      const match = state().subscriptions.get(key)
      for (const sub of match ?? []) {
        pending.push(sub(payload))
      }
    }

    // Also emit to global bus (for cross-instance communication)
    GlobalBus.emit("event", {
      directory: Instance.directory,
      payload,
    })

    return Promise.all(pending)
  }

  // Subscribe to a specific event type
  export function subscribe<Definition extends BusEvent.Definition>(
    def: Definition,
    callback: (event: { type: Definition["type"]; properties: z.infer<Definition["properties"]> }) => void
  ) {
    return raw(def.type, callback)
  }

  // Subscribe once, auto-unsubscribe when callback returns "done"
  export function once<Definition extends BusEvent.Definition>(
    def: Definition,
    callback: (event) => "done" | undefined
  ) {
    const unsub = subscribe(def, (event) => {
      if (callback(event)) unsub()
    })
  }

  // Subscribe to ALL events (wildcard)
  export function subscribeAll(callback: (event: any) => void) {
    return raw("*", callback)
  }

  // Internal: raw subscription management
  function raw(type: string, callback: (event: any) => void) {
    const subscriptions = state().subscriptions
    let match = subscriptions.get(type) ?? []
    match.push(callback)
    subscriptions.set(type, match)

    // Return unsubscribe function
    return () => {
      const match = subscriptions.get(type)
      if (!match) return
      const index = match.indexOf(callback)
      if (index === -1) return
      match.splice(index, 1)
    }
  }
}
```

### GlobalBus

**File:** `packages/opencode/src/bus/global.ts` (10 lines)

```typescript
import { EventEmitter } from "events"

export const GlobalBus = new EventEmitter<{
  event: [
    {
      directory?: string
      payload: any
    },
  ]
}>()
```

The GlobalBus is used for cross-instance communication, particularly for SSE streaming to clients.

---

## Event Definitions by Domain

### Session Events

```typescript
// packages/opencode/src/session/index.ts
export const Event = {
  Created: BusEvent.define("session.created", z.object({ info: Info })),
  Updated: BusEvent.define("session.updated", z.object({ info: Info })),
  Deleted: BusEvent.define("session.deleted", z.object({ info: Info })),
  Diff: BusEvent.define("session.diff", z.object({
    sessionID: z.string(),
    diff: Snapshot.FileDiff.array(),
  })),
  Error: BusEvent.define("session.error", z.object({
    sessionID: z.string().optional(),
    error: MessageV2.Assistant.shape.error,
  })),
}

// packages/opencode/src/session/status.ts
export const Event = {
  Status: BusEvent.define("session.status", z.object({
    sessionID: z.string(),
    status: Info,
  })),
  Idle: BusEvent.define("session.idle", z.object({
    sessionID: z.string(),
  })),
}

// packages/opencode/src/session/compaction.ts
export const Event = {
  Compacted: BusEvent.define("session.compacted", z.object({
    sessionID: z.string(),
  })),
}
```

### Message Events

```typescript
// packages/opencode/src/session/message-v2.ts
export const Event = {
  Updated: BusEvent.define("message.updated", z.object({ info: Info })),
  Removed: BusEvent.define("message.removed", z.object({
    sessionID: z.string(),
    messageID: z.string(),
  })),
  PartUpdated: BusEvent.define("message.part.updated", z.object({
    part: Part,
    delta: z.string().optional(),  // For streaming text
  })),
  PartRemoved: BusEvent.define("message.part.removed", z.object({
    sessionID: z.string(),
    messageID: z.string(),
    partID: z.string(),
  })),
}
```

### Permission Events

```typescript
// packages/opencode/src/permission/next.ts
export const Event = {
  Asked: BusEvent.define("permission.asked", Request),
  Replied: BusEvent.define("permission.replied", z.object({
    sessionID: z.string(),
    requestID: z.string(),
    reply: Reply,
  })),
}
```

### Question Events

```typescript
// packages/opencode/src/question/index.ts
export const Event = {
  Asked: BusEvent.define("question.asked", Request),
  Replied: BusEvent.define("question.replied", z.object({
    sessionID: z.string(),
    requestID: z.string(),
    answers: z.array(Answer),
  })),
  Rejected: BusEvent.define("question.rejected", z.object({
    sessionID: z.string(),
    requestID: z.string(),
  })),
}
```

### File Events

```typescript
// packages/opencode/src/file/index.ts
export const Event = {
  Edited: BusEvent.define("file.edited", z.object({ file: z.string() })),
}

// packages/opencode/src/file/watcher.ts
export const Event = {
  Updated: BusEvent.define("file.watcher.updated", z.object({
    file: z.string(),
    event: z.union([z.literal("add"), z.literal("change"), z.literal("unlink")]),
  })),
}
```

### MCP Events

```typescript
// packages/opencode/src/mcp/index.ts
export const ToolsChanged = BusEvent.define("mcp.tools.changed", z.object({
  server: z.string(),
}))

export const BrowserOpenFailed = BusEvent.define("mcp.browser.open.failed", z.object({
  mcpName: z.string(),
  url: z.string(),
}))
```

### TUI Events

```typescript
// packages/opencode/src/cli/cmd/tui/event.ts
export const TuiEvent = {
  PromptAppend: BusEvent.define("tui.prompt.append", z.object({
    text: z.string(),
  })),
  CommandExecute: BusEvent.define("tui.command.execute", z.object({
    command: z.union([
      z.enum(["session.list", "session.new", "session.share", ...]),
      z.string(),
    ]),
  })),
  ToastShow: BusEvent.define("tui.toast.show", z.object({
    title: z.string().optional(),
    message: z.string(),
    variant: z.enum(["info", "success", "warning", "error"]),
    duration: z.number().default(5000).optional(),
  })),
  SessionSelect: BusEvent.define("tui.session.select", z.object({
    sessionID: z.string().regex(/^ses/),
  })),
}
```

### Other Events

```typescript
// Project
Project.Event.Updated: BusEvent.define("project.updated", Info)
VCS.Event.BranchUpdated: BusEvent.define("vcs.branch.updated", z.object({ branch: z.string().optional() }))

// Installation
Installation.Event.Updated: BusEvent.define("installation.updated", z.object({ version: z.string() }))
Installation.Event.UpdateAvailable: BusEvent.define("installation.update-available", z.object({ version: z.string() }))

// Todo
Todo.Event.Updated: BusEvent.define("todo.updated", z.object({ sessionID, todos }))

// Command
Command.Event.Executed: BusEvent.define("command.executed", z.object({ name, sessionID, arguments }))

// PTY (Terminal)
PTY.Event.Created: BusEvent.define("pty.created", z.object({ info: Info }))
PTY.Event.Updated: BusEvent.define("pty.updated", z.object({ info: Info }))
PTY.Event.Exited: BusEvent.define("pty.exited", z.object({ id, exitCode }))
PTY.Event.Deleted: BusEvent.define("pty.deleted", z.object({ id }))

// LSP
LSP.Event.Updated: BusEvent.define("lsp.updated", z.object({}))
LSPClient.Event.Diagnostics: BusEvent.define("lsp.client.diagnostics", z.object({ serverID, path }))

// IDE
IDE.Event.Installed: BusEvent.define("ide.installed", z.object({ ide: z.string() }))

// Server
Server.Event.Connected: BusEvent.define("server.connected", z.object({}))
Server.Event.Disposed: BusEvent.define("global.disposed", z.object({}))
Bus.InstanceDisposed: BusEvent.define("server.instance.disposed", z.object({ directory }))
```

---

## Usage Patterns

### Pattern 1: Simple Publish

```typescript
// Publishing an event
await Bus.publish(Session.Event.Created, {
  info: sessionInfo,
})

// Publishing with computed properties
await Bus.publish(MessageV2.Event.PartUpdated, {
  part: currentPart,
  delta: textDelta,  // Optional streaming delta
})
```

### Pattern 2: Simple Subscribe

```typescript
// Subscribe to specific event type
const unsubscribe = Bus.subscribe(Session.Event.Updated, async (evt) => {
  console.log("Session updated:", evt.properties.info.id)
  await syncToRemote(evt.properties.info)
})

// Later: cleanup
unsubscribe()
```

### Pattern 3: Subscribe All (Wildcard)

```typescript
// Plugin system forwards all events to plugin hooks
Bus.subscribeAll(async (input) => {
  const hooks = await state().then((x) => x.hooks)
  for (const hook of hooks) {
    hook.event?.({ event: input })
  }
})

// SSE streaming to clients
Bus.subscribeAll(async (event) => {
  writer.write(`data: ${JSON.stringify(event)}\n\n`)
})
```

### Pattern 4: One-Time Subscribe

```typescript
// Wait for a specific condition, then unsubscribe
Bus.once(Session.Event.Updated, (event) => {
  if (event.properties.info.id === targetSessionId) {
    handleUpdate(event.properties.info)
    return "done"  // Unsubscribes
  }
  return undefined  // Keep listening
})
```

### Pattern 5: Subscribe with Unsubscribe on Cleanup

```typescript
// In a component or module lifecycle
export async function init() {
  const unsubscribe = Bus.subscribe(FileWatcher.Event.Updated, async (evt) => {
    if (evt.properties.event === "change") {
      await handleFileChange(evt.properties.file)
    }
  })

  // Return cleanup function
  return () => {
    unsubscribe()
  }
}
```

### Pattern 6: Cross-Component Communication

```typescript
// MCP auth shows toast via TUI event
Bus.publish(TuiEvent.ToastShow, {
  title: "MCP Authentication Required",
  message: `Server "${key}" requires authentication.`,
  variant: "warning",
  duration: 8000,
})

// TUI subscribes and shows the toast
Bus.subscribe(TuiEvent.ToastShow, (evt) => {
  showToast(evt.properties)
})
```

### Pattern 7: Reactive Data Flow

```typescript
// ShareNext reacts to session/message updates
Bus.subscribe(Session.Event.Updated, async (evt) => {
  const info = evt.properties.info
  if (!info.share) return
  await uploadSessionUpdate(info)
})

Bus.subscribe(MessageV2.Event.Updated, async (evt) => {
  const session = await Session.get(evt.properties.info.sessionID)
  if (!session.share) return
  await uploadMessageUpdate(evt.properties.info)
})

Bus.subscribe(MessageV2.Event.PartUpdated, async (evt) => {
  const part = evt.properties.part
  if (part.type !== "text" || part.synthetic) return
  await uploadPartUpdate(part, evt.properties.delta)
})
```

---

## SSE Streaming

The Bus integrates with Server-Sent Events for real-time client updates:

```typescript
// packages/opencode/src/server/server.ts
.get("/stream", async (c) => {
  return streamSSE(c, async (stream) => {
    const unsub = Bus.subscribeAll(async (event) => {
      await stream.writeSSE({
        data: JSON.stringify(event),
      })
    })

    // Keep alive
    while (true) {
      await stream.writeSSE({ data: "", event: "keepalive" })
      await stream.sleep(30_000)
    }
  })
})
```

---

## Event Flow Examples

### Session Creation Flow

```
1. Session.create()
   │
   ├─▶ Storage.write(session)
   │
   ├─▶ Bus.publish(Session.Event.Created)
   │   │
   │   ├─▶ ShareNext listener → upload to share server
   │   │
   │   ├─▶ Plugin.event hook → notify plugins
   │   │
   │   └─▶ GlobalBus.emit → SSE to clients
   │
   └─▶ Return session info
```

### Tool Execution Flow

```
1. Tool executes
   │
   ├─▶ Bus.publish(MessageV2.Event.PartUpdated, { part, delta })
   │   │
   │   ├─▶ TUI listener → update display
   │   │
   │   ├─▶ ShareNext listener → upload part
   │   │
   │   └─▶ Task listener (subtask) → forward to parent
   │
   └─▶ Continue execution
```

### Permission Flow

```
1. Tool requests permission
   │
   ├─▶ Bus.publish(Permission.Event.Asked)
   │   │
   │   └─▶ TUI listener → show permission prompt
   │
2. User responds
   │
   ├─▶ Bus.publish(Permission.Event.Replied)
   │   │
   │   └─▶ Permission system resolves promise
   │
   └─▶ Tool continues or aborts
```

---

## Design Decisions

### Why Pub/Sub?

| Aspect | Pub/Sub | Direct Calls |
|--------|---------|--------------|
| Coupling | Loose | Tight |
| Extensibility | Easy | Requires modification |
| Testing | Easy to mock | Harder to isolate |
| Debugging | Event stream | Call stack |

### Instance Isolation

Each project instance has its own subscription map:

```typescript
const state = Instance.state(
  () => ({
    subscriptions: new Map<string, Subscription[]>(),
  }),
  // Cleanup notifies wildcard subscribers
)
```

This prevents cross-contamination between projects in multi-project scenarios.

### Async Publish

`Bus.publish()` is async and waits for all subscribers:

```typescript
export async function publish(def, properties) {
  const pending = []
  for (const sub of subscribers) {
    pending.push(sub(payload))  // Collect promises
  }
  return Promise.all(pending)  // Wait for all
}
```

This ensures side effects complete before the publisher continues.

---

## Summary

| Component | Purpose |
|-----------|---------|
| `BusEvent.define()` | Create type-safe event definitions |
| `Bus.publish()` | Emit events to subscribers |
| `Bus.subscribe()` | Listen for specific event types |
| `Bus.subscribeAll()` | Listen for all events (wildcard) |
| `Bus.once()` | One-time subscription with auto-cleanup |
| `GlobalBus` | Cross-instance event emitter for SSE |

### Event Categories

| Category | Events | Purpose |
|----------|--------|---------|
| Session | created, updated, deleted, error, diff | Session lifecycle |
| Message | updated, removed, partUpdated, partRemoved | Message/part streaming |
| Permission | asked, replied | Permission flow |
| Question | asked, replied, rejected | User interaction |
| File | edited, watcher.updated | File system changes |
| TUI | promptAppend, commandExecute, toastShow | UI communication |
| MCP | toolsChanged, browserOpenFailed | MCP server events |

The Bus system is the backbone of OpenCode's reactive architecture, enabling loose coupling while maintaining type safety through Zod schemas.

---

*Written by Claude (Opus 4.5) | 2026-01-22 16:45 PST*
