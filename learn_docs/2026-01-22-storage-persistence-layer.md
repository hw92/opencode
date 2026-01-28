# OpenCode Storage/Persistence Layer

[TOC]

A study of OpenCode's file-based persistence system - a simple yet effective local-first storage approach.

---

## Overview

OpenCode uses a **file-based JSON storage** system rather than a traditional database. This design provides:

- **Simplicity** - Just JSON files on disk
- **Portability** - No database server required
- **Transparency** - Human-readable storage
- **Local-first** - Works offline, data stays on your machine

---

## Architecture

### Directory Structure

Storage uses XDG Base Directory specification for cross-platform compatibility:

```
~/.local/share/opencode/storage/     # Linux
~/Library/Application Support/opencode/storage/   # macOS
%APPDATA%/opencode/storage/          # Windows

storage/
├── migration                        # Migration version tracker
├── project/
│   ├── {projectID}.json            # Project metadata
│   └── global.json                 # Global project
├── session/
│   └── {projectID}/
│       └── {sessionID}.json        # Session metadata
├── message/
│   └── {sessionID}/
│       └── {messageID}.json        # Message content
├── part/
│   └── {messageID}/
│       └── {partID}.json           # Message parts (text, tool calls, etc.)
├── session_diff/
│   └── {sessionID}.json            # File diffs for session
├── session_share/
│   └── {sessionID}.json            # Share metadata
├── todo/
│   └── {sessionID}.json            # Todo list per session
└── permission/
    └── {projectID}.json            # Approved permissions
```

### Global Paths

```typescript
// src/global/index.ts
export namespace Global {
  export const Path = {
    data: "~/.local/share/opencode",      // XDG_DATA_HOME
    cache: "~/.cache/opencode",            // XDG_CACHE_HOME
    config: "~/.config/opencode",          // XDG_CONFIG_HOME
    state: "~/.local/state/opencode",      // XDG_STATE_HOME
    bin: "~/.local/share/opencode/bin",
    log: "~/.local/share/opencode/log",
  }
}
```

---

## Core API

**File:** `packages/opencode/src/storage/storage.ts` (227 lines)

### Key-Based Addressing

Storage uses **string arrays as keys** that map to file paths:

```typescript
// Key: ["session", "abc123", "sess_001"]
// Path: storage/session/abc123/sess_001.json

// Key: ["part", "msg_001", "part_001"]
// Path: storage/part/msg_001/part_001.json
```

### CRUD Operations

```typescript
export namespace Storage {
  // Read a JSON file
  export async function read<T>(key: string[]): Promise<T>

  // Write a JSON file (overwrites)
  export async function write<T>(key: string[], content: T): Promise<void>

  // Update with mutation function (read-modify-write)
  export async function update<T>(key: string[], fn: (draft: T) => void): Promise<T>

  // Delete a file
  export async function remove(key: string[]): Promise<void>

  // List all keys under a prefix
  export async function list(prefix: string[]): Promise<string[][]>
}
```

### Usage Examples

```typescript
// Create a session
await Storage.write(["session", projectID, sessionID], {
  id: sessionID,
  title: "My Session",
  time: { created: Date.now() }
})

// Read a session
const session = await Storage.read<Session.Info>(["session", projectID, sessionID])

// Update a session (atomic read-modify-write)
await Storage.update<Session.Info>(["session", projectID, sessionID], (draft) => {
  draft.title = "Updated Title"
  draft.time.updated = Date.now()
})

// List all sessions for a project
const sessionKeys = await Storage.list(["session", projectID])
// Returns: [["session", "abc123", "sess_001"], ["session", "abc123", "sess_002"], ...]

// Delete a session
await Storage.remove(["session", projectID, sessionID])
```

---

## Concurrency Control

### Read-Write Lock System

**File:** `packages/opencode/src/util/lock.ts`

OpenCode implements a proper **readers-writer lock** to handle concurrent access:

```typescript
export namespace Lock {
  // Multiple readers can access simultaneously
  export async function read(key: string): Promise<Disposable>

  // Only one writer, blocks all readers and other writers
  export async function write(key: string): Promise<Disposable>
}
```

### Lock Implementation Details

```typescript
const locks = new Map<string, {
  readers: number           // Count of active readers
  writer: boolean           // Is there an active writer?
  waitingReaders: (() => void)[]  // Queue of waiting readers
  waitingWriters: (() => void)[]  // Queue of waiting writers
}>()
```

**Key properties:**
1. **Multiple concurrent readers** - Many reads can happen simultaneously
2. **Exclusive writer** - Only one write at a time
3. **Writer priority** - Writers are processed before waiting readers (prevents starvation)
4. **Auto-cleanup** - Empty locks are removed from the map

### Using Disposable Pattern

The lock returns a `Disposable`, used with TypeScript's `using` keyword:

```typescript
export async function read<T>(key: string[]) {
  const target = path.join(dir, ...key) + ".json"

  using _ = await Lock.read(target)  // Acquires read lock
  const result = await Bun.file(target).json()
  return result as T
  // Lock automatically released when scope exits
}

export async function update<T>(key: string[], fn: (draft: T) => void) {
  const target = path.join(dir, ...key) + ".json"

  using _ = await Lock.write(target)  // Acquires write lock
  const content = await Bun.file(target).json()
  fn(content)  // Mutate in place
  await Bun.write(target, JSON.stringify(content, null, 2))
  return content as T
  // Lock automatically released
}
```

---

## Error Handling

### NotFoundError

Storage wraps file system errors into typed errors:

```typescript
export const NotFoundError = NamedError.create(
  "NotFoundError",
  z.object({ message: z.string() })
)

async function withErrorHandling<T>(body: () => Promise<T>) {
  return body().catch((e) => {
    if (!(e instanceof Error)) throw e
    const errnoException = e as NodeJS.ErrnoException
    if (errnoException.code === "ENOENT") {
      throw new NotFoundError({ message: `Resource not found: ${errnoException.path}` })
    }
    throw e
  })
}
```

### Graceful Fallbacks

Many callers handle NotFoundError gracefully:

```typescript
// Permission system - default to empty array if not found
const stored = await Storage.read<Ruleset>(["permission", projectID])
  .catch(() => [] as Ruleset)

// Project lookup - return undefined if not found
const existing = await Storage.read<Info>(["project", id])
  .catch(() => undefined)
```

---

## Migration System

Storage includes a built-in migration system for schema evolution:

```typescript
type Migration = (dir: string) => Promise<void>

const MIGRATIONS: Migration[] = [
  // Migration 0: Migrate from old project-based structure
  async (dir) => {
    // Move sessions from project/{id}/storage/session to session/{projectID}/
    // Reorganize messages and parts
  },

  // Migration 1: Extract diffs from session summary
  async (dir) => {
    // Move session.summary.diffs to separate session_diff/{id}.json
  },
]

// Run pending migrations on startup
const state = lazy(async () => {
  const dir = path.join(Global.Path.data, "storage")
  const migration = await Bun.file(path.join(dir, "migration"))
    .json()
    .catch(() => 0)

  for (let index = migration; index < MIGRATIONS.length; index++) {
    await MIGRATIONS[index](dir)
    await Bun.write(path.join(dir, "migration"), (index + 1).toString())
  }

  return { dir }
})
```

**Key features:**
- **Lazy execution** - Migrations run once on first Storage access
- **Version tracking** - `migration` file stores last completed migration
- **Sequential** - Migrations run in order
- **Error tolerant** - Failed migrations are logged but don't block startup

---

## Data Models

### Session Storage

```typescript
// Storage key: ["session", projectID, sessionID]
interface Session.Info {
  id: string
  slug: string
  version: string
  projectID: string
  parentID?: string         // For sub-sessions
  title: string
  share?: { url: string }
  time: {
    created: number
    updated: number
  }
}
```

### Message Storage

```typescript
// Storage key: ["message", sessionID, messageID]
interface MessageV2.Info {
  id: string
  sessionID: string
  role: "user" | "assistant"
  agent?: string
  model?: { providerID: string, modelID: string }
  time: { created: number, completed?: number }
  cost: number
  tokens: { input: number, output: number }
  finish?: string
  error?: Error
}
```

### Part Storage

```typescript
// Storage key: ["part", messageID, partID]
type MessageV2.Part =
  | TextPart
  | ReasoningPart
  | ToolPart
  | FilePart
  | StepStartPart
  | StepFinishPart
  | PatchPart
```

### Project Storage

```typescript
// Storage key: ["project", projectID]
interface Project.Info {
  id: string               // Git root commit hash or "global"
  vcs: "git" | "none"
  worktree: string         // Directory path
  time: {
    created: number
    initialized: number
    accessed?: number
  }
  archived?: boolean
}
```

---

## Usage Patterns Across Codebase

### Session Management

```typescript
// session/index.ts
export async function create(input) {
  const result = { id, title, time: { created: Date.now() } }
  await Storage.write(["session", Instance.project.id, result.id], result)
  Bus.publish(Event.Created, { info: result })
  return result
}

export async function update(id, editor) {
  const result = await Storage.update<Info>(["session", project.id, id], (draft) => {
    editor(draft)
    draft.time.updated = Date.now()
  })
  Bus.publish(Event.Updated, { info: result })
  return result
}

export async function* list() {
  for (const item of await Storage.list(["session", project.id])) {
    yield Storage.read<Info>(item)
  }
}

export async function remove(sessionID) {
  // Delete all parts
  for (const msg of await Storage.list(["message", sessionID])) {
    for (const part of await Storage.list(["part", msg.at(-1)!])) {
      await Storage.remove(part)
    }
    await Storage.remove(msg)
  }
  // Delete session
  await Storage.remove(["session", project.id, sessionID])
}
```

### Message Parts

```typescript
// session/message-v2.ts
export async function parts(messageID: string) {
  const result: Part[] = []
  for (const item of await Storage.list(["part", messageID])) {
    const read = await Storage.read<Part>(item)
    result.push(read)
  }
  result.sort((a, b) => a.id.localeCompare(b.id))
  return result
}
```

### Todo Persistence

```typescript
// session/todo.ts
export async function write(input) {
  await Storage.write(["todo", input.sessionID], input.todos)
}

export async function read(sessionID: string) {
  return Storage.read<Info[]>(["todo", sessionID])
}
```

### Permission Persistence

```typescript
// permission/next.ts
const state = Instance.state(async () => {
  const stored = await Storage.read<Ruleset>(["permission", projectID])
    .catch(() => [] as Ruleset)
  return { approved: stored }
})

// Note: Currently permissions are NOT persisted back to disk
// await Storage.write(["permission", Instance.project.id], s.approved)
```

---

## Design Decisions

### Why File-Based Storage?

1. **No dependencies** - No database server to install/maintain
2. **Portability** - Easy to backup, sync, or inspect
3. **Transparency** - JSON files are human-readable
4. **Local-first** - Works offline, data ownership
5. **Simplicity** - Fewer moving parts

### Trade-offs

| Aspect | File-Based | Database |
|--------|------------|----------|
| Setup | Zero config | Requires server |
| Queries | Limited (list/read) | Rich SQL |
| Transactions | File-level locks | ACID |
| Scalability | Limited | High |
| Inspection | Direct file access | Need tools |

### Key Architectural Choices

1. **Key as path segments** - Natural mapping to file system hierarchy
2. **JSON format** - Human-readable, easy debugging
3. **Pretty-printed** - `JSON.stringify(content, null, 2)` for readability
4. **Lazy initialization** - Migrations run on first access
5. **Disposable locks** - Automatic cleanup with `using` keyword

---

## Applying to Your Projects

### Simple Key-Value Store

```typescript
import { Storage } from "./storage"

// Store user preferences
await Storage.write(["preferences", userId], {
  theme: "dark",
  notifications: true,
})

// Read preferences
const prefs = await Storage.read<Preferences>(["preferences", userId])
  .catch(() => defaultPreferences)

// Update preferences
await Storage.update<Preferences>(["preferences", userId], (draft) => {
  draft.theme = "light"
})
```

### Hierarchical Data

```typescript
// Portfolio -> Holdings -> Transactions
await Storage.write(["portfolio", portfolioId], portfolioInfo)
await Storage.write(["holding", portfolioId, holdingId], holdingInfo)
await Storage.write(["transaction", holdingId, transactionId], txInfo)

// List all holdings for a portfolio
const holdingKeys = await Storage.list(["holding", portfolioId])
const holdings = await Promise.all(holdingKeys.map(k => Storage.read(k)))
```

### With Event Bus

```typescript
// Combine Storage with event-driven updates
async function updatePortfolio(id: string, editor: (draft: Portfolio) => void) {
  const result = await Storage.update<Portfolio>(["portfolio", id], editor)
  Bus.publish(PortfolioUpdated, { portfolio: result })
  return result
}
```

---

## Summary

OpenCode's storage layer demonstrates that **simple solutions often work best**:

| Component | Purpose |
|-----------|---------|
| `Storage.read/write/update/remove` | CRUD operations |
| `Storage.list` | Enumerate keys under prefix |
| `Lock.read/write` | Concurrency control |
| `NotFoundError` | Typed error handling |
| `MIGRATIONS` | Schema evolution |
| `Global.Path` | XDG-compliant paths |

The entire storage system is **~330 lines** of code (storage.ts + lock.ts + global) yet handles all persistence needs for sessions, messages, projects, and configuration.

---

*Written by Claude (Opus 4.5) | 2026-01-22 15:00 PST*
