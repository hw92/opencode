# Instance Context Pattern in OpenCode

This document provides a comprehensive deep-dive into the Instance context pattern used in OpenCode - a sophisticated approach to managing multi-tenant, scoped state in a Node.js/Bun application using AsyncLocalStorage.

## Table of Contents

1. [Instance Abstraction](#1-instance-abstraction)
2. [State Management](#2-state-management)
3. [Lifecycle Hooks](#3-lifecycle-hooks)
4. [Multi-Tenancy](#4-multi-tenancy)
5. [Context Propagation](#5-context-propagation)
6. [AsyncLocalStorage Foundation](#6-asynclocalstorage-foundation)
7. [Comparison to Other Patterns](#7-comparison-to-other-patterns)
8. [Best Practices](#8-best-practices)

---

## 1. Instance Abstraction

### What is Instance?

The `Instance` module (`packages/opencode/src/project/instance.ts:17-91`) is the central abstraction for managing **project-scoped context** in OpenCode. It provides:

- **Context provisioning** - Establishes execution context for a directory
- **Lazy state management** - Creates scoped singletons per project
- **Lifecycle management** - Handles initialization and disposal
- **Path isolation** - Determines if paths are within project boundaries

### Core Interface

```typescript
// packages/opencode/src/project/instance.ts:9-14
interface Context {
  directory: string    // Working directory (cwd)
  worktree: string     // Git worktree root (sandbox boundary)
  project: Project.Info // Project metadata
}
```

### Primary API

```typescript
// packages/opencode/src/project/instance.ts:17-91
export const Instance = {
  // Establish context for a directory
  async provide<R>(input: {
    directory: string;
    init?: () => Promise<any>;
    fn: () => R
  }): Promise<R>

  // Access current context properties
  get directory(): string
  get worktree(): string
  get project(): Project.Info

  // Check path boundaries
  containsPath(filepath: string): boolean

  // Create scoped state
  state<S>(init: () => S, dispose?: (state: Awaited<S>) => Promise<void>): () => S

  // Cleanup
  async dispose(): Promise<void>
  async disposeAll(): Promise<void>
}
```

---

## 2. State Management

### How `Instance.state()` Creates Scoped State

The `Instance.state()` function is a **factory for creating lazy, per-project singletons**. It delegates to the `State` module for the actual implementation.

```typescript
// packages/opencode/src/project/instance.ts:62-64
state<S>(init: () => S, dispose?: (state: Awaited<S>) => Promise<void>): () => S {
  return State.create(() => Instance.directory, init, dispose)
}
```

### State Module Implementation

The `State` module (`packages/opencode/src/project/state.ts:1-66`) manages state using a two-level Map structure:

```typescript
// packages/opencode/src/project/state.ts:10-11
const recordsByKey = new Map<string, Map<any, Entry>>()

interface Entry {
  state: any
  dispose?: (state: any) => Promise<void>
}
```

### State Creation Pattern

```typescript
// packages/opencode/src/project/state.ts:12-29
export function create<S>(root: () => string, init: () => S, dispose?: (state: Awaited<S>) => Promise<void>) {
  return () => {
    const key = root()                           // Get project directory
    let entries = recordsByKey.get(key)          // Get state map for this project
    if (!entries) {
      entries = new Map<string, Entry>()
      recordsByKey.set(key, entries)
    }
    const exists = entries.get(init)             // Use init function as key (identity)
    if (exists) return exists.state as S         // Return cached state
    const state = init()                         // Create new state
    entries.set(init, { state, dispose })        // Cache it
    return state
  }
}
```

**Key Design Decisions:**
1. Uses the **init function reference** as the cache key (identity comparison)
2. **Lazy evaluation** - state is only created when first accessed
3. **Per-project isolation** - different projects get different state instances

### Real-World Usage Examples

#### Agent Registry (Async State)
```typescript
// packages/opencode/src/agent/agent.ts:46-242
const state = Instance.state(async () => {
  const cfg = await Config.get()
  const defaults = PermissionNext.fromConfig({ /* ... */ })
  const result: Record<string, Info> = {
    build: { /* ... */ },
    plan: { /* ... */ },
    // ... more agents
  }
  return result
})

export async function get(agent: string) {
  return state().then((x) => x[agent])  // state() returns Promise
}
```

#### MCP Client Management (With Dispose)
```typescript
// packages/opencode/src/mcp/index.ts:163-210
const state = Instance.state(
  async () => {
    const cfg = await Config.get()
    const clients: Record<string, MCPClient> = {}
    const status: Record<string, Status> = {}
    // ... initialization logic
    return { status, clients }
  },
  async (state) => {
    // Cleanup: close all MCP clients on dispose
    await Promise.all(
      Object.values(state.clients).map((client) =>
        client.close().catch((error) => {
          log.error("Failed to close MCP client", { error })
        }),
      ),
    )
    pendingOAuthTransports.clear()
  },
)
```

#### Bus Subscriptions (Synchronous State)
```typescript
// packages/opencode/src/bus/index.ts:18-39
const state = Instance.state(
  () => {
    const subscriptions = new Map<any, Subscription[]>()
    return { subscriptions }
  },
  async (entry) => {
    // Notify wildcard subscribers of disposal
    const wildcard = entry.subscriptions.get("*")
    if (!wildcard) return
    const event = {
      type: InstanceDisposed.type,
      properties: { directory: Instance.directory },
    }
    for (const sub of [...wildcard]) {
      sub(event)
    }
  },
)
```

---

## 3. Lifecycle Hooks

### Initialization Pattern

The `Instance.provide()` method supports an optional `init` callback that runs **once per directory** during context creation:

```typescript
// packages/opencode/src/project/instance.ts:18-40
async provide<R>(input: { directory: string; init?: () => Promise<any>; fn: () => R }): Promise<R> {
  let existing = cache.get(input.directory)
  if (!existing) {
    Log.Default.info("creating instance", { directory: input.directory })
    existing = iife(async () => {
      const { project, sandbox } = await Project.fromDirectory(input.directory)
      const ctx = { directory: input.directory, worktree: sandbox, project }
      await context.provide(ctx, async () => {
        await input.init?.()  // Run init inside context!
      })
      return ctx
    })
    cache.set(input.directory, existing)
  }
  const ctx = await existing
  return context.provide(ctx, async () => {
    return input.fn()
  })
}
```

### Bootstrap Process

The CLI bootstrap function demonstrates the full lifecycle:

```typescript
// packages/opencode/src/cli/bootstrap.ts:4-17
export async function bootstrap<T>(directory: string, cb: () => Promise<T>) {
  return Instance.provide({
    directory,
    init: InstanceBootstrap,  // Run on first creation
    fn: async () => {
      try {
        const result = await cb()
        return result
      } finally {
        await Instance.dispose()  // Always cleanup
      }
    },
  })
}
```

The `InstanceBootstrap` function initializes all project services:

```typescript
// packages/opencode/src/project/bootstrap.ts:15-31
export async function InstanceBootstrap() {
  Log.Default.info("bootstrapping", { directory: Instance.directory })
  await Plugin.init()
  Share.init()
  ShareNext.init()
  Format.init()
  await LSP.init()
  FileWatcher.init()
  File.init()
  Vcs.init()

  Bus.subscribe(Command.Event.Executed, async (payload) => {
    if (payload.properties.name === Command.Default.INIT) {
      await Project.setInitialized(Instance.project.id)
    }
  })
}
```

### Disposal Pattern

The `Instance.dispose()` method cleans up all state for the current context:

```typescript
// packages/opencode/src/project/instance.ts:65-78
async dispose() {
  Log.Default.info("disposing instance", { directory: Instance.directory })
  await State.dispose(Instance.directory)  // Clean up all state
  cache.delete(Instance.directory)          // Remove from cache
  GlobalBus.emit("event", {
    directory: Instance.directory,
    payload: {
      type: "server.instance.disposed",
      properties: { directory: Instance.directory },
    },
  })
}
```

The State disposal waits for all dispose callbacks:

```typescript
// packages/opencode/src/project/state.ts:31-65
export async function dispose(key: string) {
  const entries = recordsByKey.get(key)
  if (!entries) return

  log.info("waiting for state disposal to complete", { key })

  // Timeout warning for long disposals
  let disposalFinished = false
  setTimeout(() => {
    if (!disposalFinished) {
      log.warn("state disposal is taking an unusually long time...", { key })
    }
  }, 10000).unref()

  // Run all dispose callbacks in parallel
  const tasks: Promise<void>[] = []
  for (const entry of entries.values()) {
    if (!entry.dispose) continue
    const task = Promise.resolve(entry.state)
      .then((state) => entry.dispose!(state))
      .catch((error) => {
        log.error("Error while disposing state:", { error, key })
      })
    tasks.push(task)
  }

  entries.clear()
  recordsByKey.delete(key)
  await Promise.all(tasks)
  disposalFinished = true
}
```

---

## 4. Multi-Tenancy

### Project Isolation Architecture

OpenCode uses `Instance.directory` as the **tenant key**. Each unique directory gets:
- Its own cached context (`cache` Map)
- Its own state entries (`State.recordsByKey` Map)
- Its own event subscriptions
- Its own LSP clients, MCP servers, etc.

```
┌─────────────────────────────────────────────────────────────┐
│                     Global Level                            │
│  ┌─────────────────────────────────────────────────────┐   │
│  │  cache: Map<directory, Promise<Context>>             │   │
│  │  ┌──────────────┐  ┌──────────────┐                 │   │
│  │  │ /project/a   │  │ /project/b   │  ...            │   │
│  │  └──────────────┘  └──────────────┘                 │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                             │
│  ┌─────────────────────────────────────────────────────┐   │
│  │  State.recordsByKey: Map<key, Map<init, Entry>>     │   │
│  │  ┌────────────────────────────────────────────────┐ │   │
│  │  │ "/project/a" -> Map{                          │ │   │
│  │  │   agentInit -> { state: {...}, dispose: fn }, │ │   │
│  │  │   busInit   -> { state: {...}, dispose: fn }, │ │   │
│  │  │   mcpInit   -> { state: {...}, dispose: fn }, │ │   │
│  │  │ }                                              │ │   │
│  │  └────────────────────────────────────────────────┘ │   │
│  │  ┌────────────────────────────────────────────────┐ │   │
│  │  │ "/project/b" -> Map{ ... separate state ... }  │ │   │
│  │  └────────────────────────────────────────────────┘ │   │
│  └─────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
```

### HTTP Request Isolation

The server establishes Instance context for each request:

```typescript
// packages/opencode/src/server/server.ts:130-144
.use(async (c, next) => {
  let directory = c.req.query("directory") ||
                  c.req.header("x-opencode-directory") ||
                  process.cwd()
  try {
    directory = decodeURIComponent(directory)
  } catch {
    // fallback to original value
  }
  return Instance.provide({
    directory,
    init: InstanceBootstrap,
    async fn() {
      return next()
    },
  })
})
```

This ensures:
1. Each request can target a different project directory
2. Request handlers access the correct project's state via `Instance.directory`
3. Different browser tabs can work on different projects simultaneously

### Path Boundary Checking

The `containsPath` method enforces project boundaries:

```typescript
// packages/opencode/src/project/instance.ts:51-61
containsPath(filepath: string) {
  if (Filesystem.contains(Instance.directory, filepath)) return true
  // Non-git projects set worktree to "/" which would match ANY path
  // Skip worktree check to preserve external_directory permissions
  if (Instance.worktree === "/") return false
  return Filesystem.contains(Instance.worktree, filepath)
}
```

This is used by tools to determine if file access requires special permissions.

---

## 5. Context Propagation

### Context Flow Diagram

```
┌──────────────────────────────────────────────────────────────────┐
│                        HTTP Request                               │
│  GET /session?directory=/project/foo                             │
└─────────────────────────┬────────────────────────────────────────┘
                          │
                          ▼
┌──────────────────────────────────────────────────────────────────┐
│                   Server Middleware                               │
│  Instance.provide({ directory: "/project/foo", ... })            │
│  └─> AsyncLocalStorage.run(context, ...)                         │
└─────────────────────────┬────────────────────────────────────────┘
                          │
                          ▼
┌──────────────────────────────────────────────────────────────────┐
│                   Route Handler                                   │
│  Session.list() -> Instance.directory (reads "/project/foo")     │
│       │                                                          │
│       ▼                                                          │
│  Storage.list(["session", Instance.project.id, ...])            │
└─────────────────────────┬────────────────────────────────────────┘
                          │
                          ▼
┌──────────────────────────────────────────────────────────────────┐
│                   State Access                                    │
│  state() -> State.create(() => Instance.directory, init)        │
│  └─> Returns state scoped to "/project/foo"                      │
└──────────────────────────────────────────────────────────────────┘
```

### Automatic Context in Callbacks

Context automatically propagates through async callbacks because AsyncLocalStorage is bound to the execution context:

```typescript
// Example: Bus subscription receives events in correct context
Bus.subscribe(Session.Event.Created, async (event) => {
  // Instance.directory is automatically available here
  console.log("Session created in:", Instance.directory)
})
```

### Cross-Context Communication via GlobalBus

When events need to cross project boundaries, the GlobalBus is used:

```typescript
// packages/opencode/src/bus/global.ts:1-10
export const GlobalBus = new EventEmitter<{
  event: [
    {
      directory?: string     // Optional project context
      payload: any
    },
  ]
}>()
```

Local Bus publishes to GlobalBus with directory tag:

```typescript
// packages/opencode/src/bus/index.ts:59-63
GlobalBus.emit("event", {
  directory: Instance.directory,  // Tag with current project
  payload,
})
```

---

## 6. AsyncLocalStorage Foundation

### The Context Utility

OpenCode wraps Node.js AsyncLocalStorage in a simple utility:

```typescript
// packages/opencode/src/util/context.ts:1-25
import { AsyncLocalStorage } from "async_hooks"

export namespace Context {
  export class NotFound extends Error {
    constructor(public override readonly name: string) {
      super(`No context found for ${name}`)
    }
  }

  export function create<T>(name: string) {
    const storage = new AsyncLocalStorage<T>()
    return {
      use() {
        const result = storage.getStore()
        if (!result) {
          throw new NotFound(name)
        }
        return result
      },
      provide<R>(value: T, fn: () => R) {
        return storage.run(value, fn)
      },
    }
  }
}
```

### Usage in Instance

```typescript
// packages/opencode/src/project/instance.ts:14
const context = Context.create<Context>("instance")

// Providing context
return context.provide(ctx, async () => {
  return input.fn()
})

// Accessing context
get directory() {
  return context.use().directory
}
```

### How AsyncLocalStorage Works

AsyncLocalStorage maintains a **store** that is:
1. **Scoped to the execution context** - not global
2. **Inherited by child async operations** - Promise chains, setTimeout callbacks, etc.
3. **Isolated between parallel executions** - two concurrent requests have separate stores

```
Request 1                         Request 2
─────────                         ─────────
storage.run(ctx1, async () => {   storage.run(ctx2, async () => {
  await fetch(...)                   await fetch(...)
  storage.getStore() // ctx1         storage.getStore() // ctx2
})                                })
```

---

## 7. Comparison to Other Patterns

### vs. React Context

| Aspect | Instance Context | React Context |
|--------|------------------|---------------|
| **Runtime** | Node.js/Bun server | Browser/React |
| **Scope** | Execution context (async flow) | Component tree |
| **Storage** | AsyncLocalStorage | React fiber tree |
| **Access** | `Instance.directory` | `useContext(Ctx)` |
| **Provision** | `Instance.provide({...})` | `<Ctx.Provider value={...}>` |
| **Lazy State** | `Instance.state(init)` | `useMemo(init, [])` |

**Similarity:** Both provide scoped values that propagate through execution/render trees.
**Difference:** React Context is component-based; Instance Context is request/execution-based.

### vs. Dependency Injection Containers

| Aspect | Instance Context | DI Container (e.g., InversifyJS) |
|--------|------------------|----------------------------------|
| **Registration** | Implicit via `Instance.state()` | Explicit `container.bind()` |
| **Resolution** | `state()` (lazy singleton) | `container.get(Token)` |
| **Scope** | Per-directory (tenant) | Per-container scope |
| **Lifecycle** | Built-in dispose | Often manual or decorators |
| **Type Safety** | TypeScript inference | Often requires tokens |

**Similarity:** Both manage singleton lifecycles and scoping.
**Difference:** DI uses explicit registration; Instance uses implicit lazy creation.

### vs. Thread-Local Storage (Java/C#)

| Aspect | Instance Context | ThreadLocal |
|--------|------------------|-------------|
| **Binding** | Async execution context | OS thread |
| **Inheritance** | Automatic in async flows | Requires InheritableThreadLocal |
| **Use Case** | Multi-tenant web servers | Thread-specific data |

**Similarity:** Both provide execution-scoped storage.
**Difference:** AsyncLocalStorage handles JavaScript's async nature; ThreadLocal is for true threads.

### Pattern Summary

Instance Context is essentially a **Scoped Singleton** pattern with:
- **Async-aware scoping** via AsyncLocalStorage
- **Lazy initialization** via factory functions
- **Automatic cleanup** via dispose callbacks
- **Multi-tenancy** via directory-based keying

---

## 8. Best Practices

### 1. Always Define Dispose Callbacks for Resources

```typescript
// Good: Cleanup external resources
const state = Instance.state(
  async () => {
    const client = await createClient()
    return { client }
  },
  async (state) => {
    await state.client.close()  // Always cleanup!
  }
)

// Bad: Resource leak
const state = Instance.state(async () => {
  const client = await createClient()
  return { client }
  // No dispose - client stays open forever!
})
```

### 2. Use Async State for I/O Operations

```typescript
// Good: Async initialization
const state = Instance.state(async () => {
  const config = await Config.get()  // Async file read
  return processConfig(config)
})

// Access: await state() or state().then(...)
```

### 3. Keep State Initialization Pure

```typescript
// Good: Pure initialization
const state = Instance.state(() => ({
  subscriptions: new Map(),
  pending: new Set(),
}))

// Avoid: Side effects in init
const state = Instance.state(() => {
  console.log("Creating state!")  // Side effect - avoid
  return {}
})
```

### 4. Access Instance Properties Only Inside Context

```typescript
// Good: Inside provide() callback
Instance.provide({
  directory: "/project",
  fn: () => {
    console.log(Instance.directory)  // Works!
  }
})

// Bad: Outside context
console.log(Instance.directory)  // Throws Context.NotFound!
```

### 5. Use containsPath for Security Checks

```typescript
// Good: Check before file operations
async function readFile(filepath: string) {
  if (!Instance.containsPath(filepath)) {
    throw new Error("Access denied: path outside project")
  }
  return fs.readFile(filepath)
}
```

### 6. Prefer Lazy State Over Eager Initialization

```typescript
// Good: Lazy - created only when needed
const state = Instance.state(async () => {
  return await expensiveOperation()
})

// Less ideal: Eager - created during bootstrap
let cachedState: any
async function InstanceBootstrap() {
  cachedState = await expensiveOperation()  // Always runs
}
```

### 7. Handle State Access in Tests

```typescript
// packages/opencode/test/session/session.test.ts:12-39
test("should work within instance context", async () => {
  await Instance.provide({
    directory: projectRoot,
    fn: async () => {
      // Test code runs inside context
      const session = await Session.create({})
      expect(session.directory).toBe(projectRoot)
    },
  })
})
```

---

## Key File References

| File | Purpose |
|------|---------|
| `packages/opencode/src/project/instance.ts` | Main Instance abstraction |
| `packages/opencode/src/project/state.ts` | State management implementation |
| `packages/opencode/src/util/context.ts` | AsyncLocalStorage wrapper |
| `packages/opencode/src/project/project.ts` | Project detection and metadata |
| `packages/opencode/src/project/bootstrap.ts` | Instance initialization |
| `packages/opencode/src/cli/bootstrap.ts` | CLI lifecycle management |
| `packages/opencode/src/server/server.ts:130-144` | HTTP request context middleware |
| `packages/opencode/src/bus/index.ts` | Event bus with Instance.state |
| `packages/opencode/src/mcp/index.ts:163-210` | MCP client state with dispose |

---

*Written by Claude (Opus 4.5) | 2026-01-22 PST*
