# OpenCode Share System Deep Dive

[TOC]

A study of how OpenCode enables session sharing - converting local conversations into shareable web links with real-time synchronization.

---

## Overview

The Share System allows users to:
1. **Create shareable links** for their coding sessions
2. **Real-time sync** updates as the session progresses
3. **Import shared sessions** from URLs back into their local OpenCode
4. **Export sessions** as JSON for backup/transfer

**Key URLs:**
- Share viewer: `https://opncd.ai/share/{slug}`
- API endpoint: `https://api.opencode.ai` (prod) or `https://api.dev.opencode.ai` (dev)

---

## Architecture

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        SHARE SYSTEM ARCHITECTURE                            │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  Local OpenCode                                                             │
│       │                                                                     │
│       ▼                                                                     │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │  Session Events (Bus)                                               │   │
│  │  • Session.Event.Updated                                            │   │
│  │  • MessageV2.Event.Updated                                          │   │
│  │  • MessageV2.Event.PartUpdated                                      │   │
│  │  • Session.Event.Diff                                               │   │
│  └─────────────────────┬───────────────────────────────────────────────┘   │
│                        │                                                   │
│                        ▼                                                   │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │  ShareNext.sync()                                                   │   │
│  │  • Batches updates (1 second window)                                │   │
│  │  • Queues by sessionID                                              │   │
│  │  • Sends to API                                                     │   │
│  └─────────────────────┬───────────────────────────────────────────────┘   │
│                        │                                                   │
│                        ▼                                                   │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │  Remote API (opncd.ai)                                              │   │
│  │  • POST /api/share (create)                                         │   │
│  │  • POST /api/share/{id}/sync (update)                               │   │
│  │  • DELETE /api/share/{id} (remove)                                  │   │
│  │  • GET /api/share/{slug} (fetch for import)                         │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  Web Viewer                                                                 │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │  https://opncd.ai/share/{slug}                                      │   │
│  │  • Renders session as web page                                      │   │
│  │  • Shows messages, tool calls, diffs                                │   │
│  │  • Read-only view for sharing                                       │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## Part 1: Creating a Share

### Session.share()

**File:** `packages/opencode/src/session/index.ts`

```typescript
export const share = fn(Identifier.schema("session"), async (id) => {
  const cfg = await Config.get()

  // Check if sharing is disabled
  if (cfg.share === "disabled") {
    throw new Error("Sharing is disabled in configuration")
  }

  // Use ShareNext to create the share
  const { ShareNext } = await import("@/share/share-next")
  const share = await ShareNext.create(id)

  // Update session with share URL
  await update(id, (draft) => {
    draft.share = {
      url: share.url,
    }
  })

  return share
})
```

### ShareNext.create()

**File:** `packages/opencode/src/share/share-next.ts`

```typescript
export async function create(sessionID: string) {
  log.info("creating share", { sessionID })

  // 1. Call API to create share record
  const result = await fetch(`${await url()}/api/share`, {
    method: "POST",
    headers: { "Content-Type": "application/json" },
    body: JSON.stringify({ sessionID: sessionID }),
  })
    .then((x) => x.json())
    .then((x) => x as { id: string; url: string; secret: string })

  // 2. Store share metadata locally
  await Storage.write(["session_share", sessionID], result)

  // 3. Perform full sync to upload all existing data
  fullSync(sessionID)

  return result
}
```

### Full Sync (Initial Upload)

When a share is created, all existing session data is uploaded:

```typescript
async function fullSync(sessionID: string) {
  log.info("full sync", { sessionID })

  // Gather all session data
  const session = await Session.get(sessionID)
  const diffs = await Session.diff(sessionID)
  const messages = await Array.fromAsync(MessageV2.stream(sessionID))

  // Get model info for display
  const models = await Promise.all(
    messages
      .filter((m) => m.info.role === "user")
      .map((m) => Provider.getModel(m.info.model.providerID, m.info.model.modelID))
  )

  // Send everything in one batch
  await sync(sessionID, [
    { type: "session", data: session },
    ...messages.map((x) => ({ type: "message" as const, data: x.info })),
    ...messages.flatMap((x) => x.parts.map((y) => ({ type: "part" as const, data: y }))),
    { type: "session_diff", data: diffs },
    { type: "model", data: models },
  ])
}
```

---

## Part 2: Real-Time Synchronization

### Event-Driven Updates

ShareNext subscribes to Bus events and syncs changes:

```typescript
export async function init() {
  // Session metadata changes
  Bus.subscribe(Session.Event.Updated, async (evt) => {
    await sync(evt.properties.info.id, [
      { type: "session", data: evt.properties.info },
    ])
  })

  // New/updated messages
  Bus.subscribe(MessageV2.Event.Updated, async (evt) => {
    await sync(evt.properties.info.sessionID, [
      { type: "message", data: evt.properties.info },
    ])

    // Also sync model info for user messages
    if (evt.properties.info.role === "user") {
      const model = await Provider.getModel(
        evt.properties.info.model.providerID,
        evt.properties.info.model.modelID
      )
      await sync(evt.properties.info.sessionID, [
        { type: "model", data: [model] },
      ])
    }
  })

  // Message parts (tool calls, text, reasoning)
  Bus.subscribe(MessageV2.Event.PartUpdated, async (evt) => {
    await sync(evt.properties.part.sessionID, [
      { type: "part", data: evt.properties.part },
    ])
  })

  // File diffs
  Bus.subscribe(Session.Event.Diff, async (evt) => {
    await sync(evt.properties.sessionID, [
      { type: "session_diff", data: evt.properties.diff },
    ])
  })
}
```

### Batched Sync Queue

Updates are batched to avoid flooding the API:

```typescript
const queue = new Map<string, {
  timeout: NodeJS.Timeout
  data: Map<string, Data>
}>()

async function sync(sessionID: string, data: Data[]) {
  const existing = queue.get(sessionID)

  if (existing) {
    // Add to existing batch
    for (const item of data) {
      existing.data.set("id" in item ? (item.id as string) : ulid(), item)
    }
    return
  }

  // Create new batch
  const dataMap = new Map<string, Data>()
  for (const item of data) {
    dataMap.set("id" in item ? (item.id as string) : ulid(), item)
  }

  // Flush after 1 second
  const timeout = setTimeout(async () => {
    const queued = queue.get(sessionID)
    if (!queued) return
    queue.delete(sessionID)

    // Check if share still exists
    const share = await get(sessionID).catch(() => undefined)
    if (!share) return

    // Send batched data
    await fetch(`${await url()}/api/share/${share.id}/sync`, {
      method: "POST",
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify({
        secret: share.secret,
        data: Array.from(queued.data.values()),
      }),
    })
  }, 1000)  // 1 second batching window

  queue.set(sessionID, { timeout, data: dataMap })
}
```

### Data Types Synced

```typescript
type Data =
  | { type: "session"; data: SDK.Session }        // Session metadata
  | { type: "message"; data: SDK.Message }        // Message info
  | { type: "part"; data: SDK.Part }              // Text, tool calls, etc.
  | { type: "session_diff"; data: SDK.FileDiff[] }// File changes
  | { type: "model"; data: SDK.Model[] }          // Model info for display
```

---

## Part 3: Share Configuration

**File:** `packages/opencode/src/config/config.ts`

```yaml
# opencode.json or opencode.yaml
share: "manual"   # "manual" | "auto" | "disabled"
```

| Mode | Behavior |
|------|----------|
| `manual` | Share via command only |
| `auto` | Auto-share new sessions |
| `disabled` | Sharing completely disabled |

### Auto-Share on Session Create

```typescript
// In Session.create()
if (!result.parentID && (Flag.OPENCODE_AUTO_SHARE || cfg.share === "auto")) {
  share(result.id)
    .then((share) => {
      update(result.id, (draft) => {
        draft.share = share
      })
    })
    .catch(() => {})  // Fail silently
}
```

---

## Part 4: Removing a Share

### Session.unshare()

```typescript
export const unshare = fn(Identifier.schema("session"), async (id) => {
  const { ShareNext } = await import("@/share/share-next")
  await ShareNext.remove(id)

  await update(id, (draft) => {
    draft.share = undefined
  })
})
```

### ShareNext.remove()

```typescript
export async function remove(sessionID: string) {
  log.info("removing share", { sessionID })

  const share = await get(sessionID)
  if (!share) return

  // Delete from remote
  await fetch(`${await url()}/api/share/${share.id}`, {
    method: "DELETE",
    headers: { "Content-Type": "application/json" },
    body: JSON.stringify({ secret: share.secret }),
  })

  // Remove local metadata
  await Storage.remove(["session_share", sessionID])
}
```

---

## Part 5: Import/Export

### Export Command

**File:** `packages/opencode/src/cli/cmd/export.ts`

```bash
# Export specific session
opencode export sess_abc123 > session.json

# Interactive session selection
opencode export > session.json
```

```typescript
const exportData = {
  info: sessionInfo,
  messages: messages.map((msg) => ({
    info: msg.info,
    parts: msg.parts,
  })),
}

process.stdout.write(JSON.stringify(exportData, null, 2))
```

### Import Command

**File:** `packages/opencode/src/cli/cmd/import.ts`

```bash
# Import from local file
opencode import session.json

# Import from share URL
opencode import https://opncd.ai/share/abc123xyz
```

```typescript
export const ImportCommand = cmd({
  command: "import <file>",
  describe: "import session data from JSON file or URL",
  handler: async (args) => {
    let exportData: { info: Session.Info; messages: Array<...> }

    const isUrl = args.file.startsWith("http://") || args.file.startsWith("https://")

    if (isUrl) {
      // Parse share URL
      const urlMatch = args.file.match(/https?:\/\/opncd\.ai\/share\/([a-zA-Z0-9_-]+)/)
      if (!urlMatch) {
        process.stdout.write(`Invalid URL format. Expected: https://opncd.ai/share/<slug>`)
        return
      }

      const slug = urlMatch[1]
      const response = await fetch(`https://opncd.ai/api/share/${slug}`)
      const data = await response.json()

      exportData = {
        info: data.info,
        messages: Object.values(data.messages).map((msg: any) => {
          const { parts, ...info } = msg
          return { info, parts }
        }),
      }
    } else {
      // Read from file
      exportData = await Bun.file(args.file).json()
    }

    // Write to local storage
    await Storage.write(["session", Instance.project.id, exportData.info.id], exportData.info)

    for (const msg of exportData.messages) {
      await Storage.write(["message", exportData.info.id, msg.info.id], msg.info)
      for (const part of msg.parts) {
        await Storage.write(["part", msg.info.id, part.id], part)
      }
    }

    process.stdout.write(`Imported session: ${exportData.info.id}`)
  },
})
```

---

## Part 6: API Endpoints

### Server Routes

**File:** `packages/opencode/src/server/routes/session.ts`

```typescript
// Create share
.post(
  "/:sessionID/share",
  describeRoute({
    summary: "Share session",
    description: "Create a shareable link for a session",
    operationId: "session.share",
  }),
  async (c) => {
    const sessionID = c.req.valid("param").sessionID
    await Session.share(sessionID)
    const session = await Session.get(sessionID)
    return c.json(session)
  },
)

// Remove share
.delete(
  "/:sessionID/share",
  describeRoute({
    summary: "Unshare session",
    description: "Remove the shareable link for a session",
    operationId: "session.unshare",
  }),
  async (c) => {
    const sessionID = c.req.valid("param").sessionID
    await Session.unshare(sessionID)
    const session = await Session.get(sessionID)
    return c.json(session)
  },
)
```

---

## Part 7: Storage Schema

### Share Metadata

```typescript
// Storage key: ["session_share", sessionID]
interface ShareMetadata {
  id: string      // Share ID on remote
  secret: string  // Secret for authenticated sync
  url: string     // Public share URL
}
```

### Session with Share

```typescript
interface Session.Info {
  id: string
  title: string
  share?: {
    url: string
  }
  // ...
}
```

---

## Key Design Patterns

### Pattern 1: Event-Driven Sync

Changes are automatically synced via Bus subscriptions:

```
User types prompt
      │
      ▼
Session.prompt() creates message
      │
      ▼
MessageV2.Event.Updated published
      │
      ▼
ShareNext.init() subscription triggers
      │
      ▼
sync() adds to queue
      │
      ▼
After 1 second, batch sent to API
```

### Pattern 2: Secret-Based Authentication

Each share has a secret that proves ownership:

```typescript
// Create returns secret
const { id, url, secret } = await ShareNext.create(sessionID)

// All subsequent calls require secret
await fetch(`/api/share/${id}/sync`, {
  body: JSON.stringify({ secret, data: [...] }),
})
```

### Pattern 3: Batched Updates

Updates are batched by sessionID with a 1-second window:

```
T+0ms:   Part update arrives → Start 1s timer
T+50ms:  Text delta arrives → Add to batch
T+100ms: Tool result arrives → Add to batch
T+1000ms: Timer fires → Send all 3 updates in one request
```

### Pattern 4: Graceful Degradation

Sharing failures don't break the main flow:

```typescript
if (cfg.share === "auto") {
  share(result.id)
    .catch(() => {})  // Silent failure
}
```

---

## Comparison: Share vs Memories

| Aspect | Share System | Memories System |
|--------|--------------|-----------------|
| Purpose | Share sessions externally | Personal recall |
| Storage | Remote API (opncd.ai) | Local CLAUDE.md |
| Access | Public URL | Private/local |
| Data | Full session + messages | Distilled insights |
| Sync | Real-time | Manual/on-demand |

The Share System is about **collaboration** and **transparency** - letting others see your AI coding session.

Memories (CLAUDE.md) is about **personal context** - storing learnings across sessions.

---

## Applying to Your Projects

### Simple Share System

```typescript
// 1. Define share metadata storage
interface ShareInfo {
  id: string
  secret: string
  url: string
}

// 2. Create share
async function createShare(entityId: string): Promise<ShareInfo> {
  const response = await fetch(`${API_URL}/share`, {
    method: "POST",
    body: JSON.stringify({ entityId }),
  })
  const share = await response.json()

  // Store locally
  await storage.write(["share", entityId], share)

  // Sync existing data
  await fullSync(entityId, share)

  return share
}

// 3. Real-time sync on changes
eventBus.subscribe("entity.updated", async (evt) => {
  const share = await storage.read(["share", evt.entityId]).catch(() => null)
  if (!share) return

  await fetch(`${API_URL}/share/${share.id}/sync`, {
    method: "POST",
    body: JSON.stringify({
      secret: share.secret,
      data: evt.data,
    }),
  })
})
```

---

## Summary

| Component | Purpose |
|-----------|---------|
| `ShareNext.create()` | Create share + full sync |
| `ShareNext.sync()` | Batched incremental sync |
| `ShareNext.remove()` | Delete share |
| `ShareNext.init()` | Subscribe to Bus events |
| `Session.share()` | High-level share API |
| `ImportCommand` | Import from URL/file |
| `ExportCommand` | Export to JSON |

The Share System demonstrates:
- **Event-driven architecture** - Sync triggered by Bus events
- **Batching** - Reduce API calls with time-based batching
- **Secret-based auth** - Simple ownership verification
- **Graceful failures** - Sharing failures don't break main flow
- **Full + incremental sync** - Initial full upload, then deltas

---

*Written by Claude (Opus 4.5) | 2026-01-22 17:00 PST*
