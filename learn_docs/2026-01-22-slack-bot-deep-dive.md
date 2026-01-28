# OpenCode Slack Bot Deep Dive

[TOC]

A comprehensive study of the Slack bot integration that brings OpenCode AI assistance into Slack conversations.

---

## Overview

The Slack bot allows teams to interact with OpenCode directly within Slack channels and threads. It uses **Socket Mode** for real-time communication and spawns a local OpenCode server to handle AI requests.

---

## Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                       Slack Workspace                            │
│  (User sends message in channel/thread)                         │
└──────────────────────────┬──────────────────────────────────────┘
                           │
                           │ Socket Mode (WebSocket)
                           ▼
        ┌──────────────────────────────────────┐
        │   OpenCode Slack Bot                  │
        │   (@slack/bolt v3.17.1)              │
        └──────────────────────────────────────┘
                           │
         ┌─────────────────┼─────────────────┐
         │                 │                 │
         ▼                 ▼                 ▼
    Session Map      Message Queue     Event Stream
    (per thread)     & Processing      (SSE from server)
         │                 │                 │
         └─────────────────┼─────────────────┘
                           │
                           ▼
        ┌──────────────────────────────────────┐
        │   OpenCode Core Server               │
        │   (opencode serve)                   │
        │   HTTP API + SSE Event Stream        │
        └──────────────────────────────────────┘
```

---

## Key Components

### Entry Point

**File:** `packages/slack/src/index.ts`

```typescript
import { App } from "@slack/bolt"
import { createOpencode } from "@opencode-ai/sdk"

const app = new App({
  token: process.env.SLACK_BOT_TOKEN,
  signingSecret: process.env.SLACK_SIGNING_SECRET,
  socketMode: true,
  appToken: process.env.SLACK_APP_TOKEN,
})

// Spawn local OpenCode server
const opencode = await createOpencode({ port: 0 })
```

### Session Management

One OpenCode session per Slack thread:

```typescript
const sessions = new Map<string, {
  client: any
  server: any
  sessionId: string
  channel: string
  thread: string
}>()

// Session key = channel + thread_ts
const sessionKey = `${channel}-${thread}`
```

---

## Message Flow

### 1. User Sends Message

```typescript
app.message(async ({ message, say }) => {
  // Skip non-text messages
  if (message.subtype || !("text" in message)) return

  const channel = message.channel
  const thread = message.thread_ts || message.ts
  const sessionKey = `${channel}-${thread}`
```

### 2. Create or Reuse Session

```typescript
let session = sessions.get(sessionKey)

if (!session) {
  // Create new OpenCode session
  const createResult = await client.session.create({
    body: { title: `Slack thread ${thread}` }
  })

  // Share session and post URL
  const shareResult = await client.session.share({
    path: { id: createResult.data.id }
  })

  await say({
    text: shareResult.data.share?.url,
    thread_ts: thread,
  })

  sessions.set(sessionKey, { ... })
}
```

### 3. Send to OpenCode

```typescript
const result = await session.client.session.prompt({
  path: { id: session.sessionId },
  body: {
    parts: [{ type: "text", text: message.text }]
  }
})
```

### 4. Post Response

```typescript
const responseText = result.data.parts
  ?.filter(p => p.type === "text")
  .map(p => p.text)
  .join("\n")

await say({ text: responseText, thread_ts: thread })
```

---

## Real-Time Tool Updates

The bot subscribes to OpenCode events to show tool execution status:

```typescript
const events = await opencode.client.event.subscribe()

for await (const event of events.stream) {
  if (event.type === "message.part.updated") {
    const part = event.properties.part

    if (part.type === "tool" && part.state.status === "completed") {
      // Find session and post update
      await app.client.chat.postMessage({
        channel: session.channel,
        thread_ts: session.thread,
        text: `*${part.tool}* - ${part.state.title}`,
      })
    }
  }
}
```

---

## Configuration

### Required Environment Variables

```bash
SLACK_BOT_TOKEN=xoxb-...      # Bot User OAuth Token
SLACK_SIGNING_SECRET=...       # Signing Secret
SLACK_APP_TOKEN=xapp-...       # App-Level Token (Socket Mode)
```

### Required OAuth Scopes

| Scope | Purpose |
|-------|---------|
| `chat:write` | Post messages |
| `app_mentions:read` | Listen to @mentions |
| `channels:history` | Read channel messages |
| `groups:history` | Read private channels |

---

## Slash Commands

```typescript
app.command("/test", async ({ command, ack, say }) => {
  await ack()
  await say("🤖 Bot is working!")
})
```

---

## SDK Integration

The bot uses the OpenCode SDK for all operations:

| Method | Purpose |
|--------|---------|
| `createOpencode()` | Boot local server + client |
| `client.session.create()` | Create new session |
| `client.session.share()` | Generate shareable URL |
| `client.session.prompt()` | Send message to AI |
| `client.event.subscribe()` | Subscribe to live events |

---

## Message Part Types

**Input:**
- `TextPartInput` - Plain text
- `FilePartInput` - File references
- `AgentPartInput` - Custom agents

**Response:**
- `TextPart` - AI response text
- `ToolPart` - Tool calls with status
- `ReasoningPart` - Internal reasoning
- `FilePart` - Modified files

---

## Error Handling

```typescript
if (createResult.error) {
  await say({
    text: "Sorry, I had trouble creating a session.",
    thread_ts: thread,
  })
  return
}

if (result.error) {
  await say({
    text: "Sorry, I had trouble processing your message.",
    thread_ts: thread,
  })
  return
}
```

---

## Design Patterns

| Pattern | Purpose |
|---------|---------|
| **Session Map** | One session per Slack thread |
| **Event-Driven** | Real-time tool updates via SSE |
| **Lazy Creation** | Sessions created on first message |
| **Socket Mode** | No public HTTP endpoint needed |

---

## Running the Bot

```bash
cd packages/slack
bun install
bun dev
```

---

## Key Files

| File | Purpose |
|------|---------|
| `packages/slack/src/index.ts` | Main bot implementation |
| `packages/slack/package.json` | Dependencies |
| `packages/slack/README.md` | Setup instructions |
| `packages/sdk/js/src/v2/` | SDK client code |

---

*Written by Claude (Opus 4.5) | 2026-01-22 16:45 PST*
