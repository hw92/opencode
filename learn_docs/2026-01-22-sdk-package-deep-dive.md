# OpenCode SDK Package Deep Dive

[TOC]

A comprehensive study of the TypeScript/JavaScript client library for the OpenCode API.

---

## Overview

The SDK (`@opencode-ai/sdk`) provides a fully-typed client for interacting with OpenCode's API. It's auto-generated from an OpenAPI specification using `@hey-api/openapi-ts`.

**Key Features:**
- Zero runtime dependencies
- Full TypeScript type safety
- Auto-generated from OpenAPI spec
- SSE streaming for real-time events
- Server bootstrapping utilities

---

## Architecture

```
packages/sdk/
├── js/
│   ├── src/
│   │   ├── index.ts              # Main entry (v1)
│   │   ├── client.ts             # Client factory
│   │   ├── server.ts             # Server bootstrapping
│   │   ├── gen/                  # Auto-generated v1
│   │   │   ├── core/             # HTTP client, SSE, auth
│   │   │   ├── sdk.gen.ts        # SDK methods (~1,200 lines)
│   │   │   └── types.gen.ts      # Types (~3,900 lines)
│   │   └── v2/                   # V2 API
│   │       ├── index.ts
│   │       ├── client.ts
│   │       ├── server.ts
│   │       └── gen/              # V2 generated code
│   └── openapi.json              # API specification
└── openapi.json
```

---

## Client Creation

### Basic Client

```typescript
import { createOpencodeClient } from "@opencode-ai/sdk"

const client = createOpencodeClient({
  baseUrl: "http://localhost:4096",
  directory: "/path/to/project",
})
```

### With Server Bootstrap

```typescript
import { createOpencode } from "@opencode-ai/sdk"

// Starts local server and returns client
const { client, server } = await createOpencode({
  port: 0,  // Auto-assign port
})

// Use client...

// Cleanup
server.close()
```

---

## Client Namespaces

The SDK organizes methods into 16 namespaces:

### Session (`client.session.*`)

```typescript
// CRUD
client.session.list()
client.session.create({ body: { title: "..." } })
client.session.get({ path: { id } })
client.session.update({ path: { id }, body: { ... } })
client.session.delete({ path: { id } })

// Messaging
client.session.prompt({ path: { id }, body: { parts: [...] } })
client.session.promptAsync({ path: { id }, body: { parts: [...] } })
client.session.messages({ path: { id } })

// Lifecycle
client.session.fork({ path: { id }, body: { messageID } })
client.session.abort({ path: { id } })
client.session.revert({ path: { id }, body: { messageID } })
client.session.share({ path: { id } })
```

### Project (`client.project.*`)

```typescript
client.project.list()
client.project.current()
```

### File (`client.file.*`)

```typescript
client.file.list({ query: { path } })
client.file.read({ query: { path } })
client.file.status()
```

### Search (`client.find.*`)

```typescript
client.find.text({ query: { pattern, path } })
client.find.files({ query: { pattern } })
client.find.symbols({ query: { query } })
```

### Provider (`client.provider.*`)

```typescript
client.provider.list()
client.provider.auth()
client.provider.oauth.authorize({ path: { providerID } })
```

### Events (`client.event.*`)

```typescript
const { stream } = await client.event.subscribe()

for await (const event of stream) {
  console.log(event.type, event.properties)
}
```

### Other Namespaces

- `client.config.*` - Configuration
- `client.tool.*` - Tool management
- `client.app.*` - Agents, logging
- `client.pty.*` - Terminal sessions
- `client.mcp.*` - MCP servers
- `client.vcs.*` - Version control
- `client.lsp.*` - Language servers
- `client.tui.*` - TUI commands

---

## Type System

### Core Types

```typescript
// Session
type Session = {
  id: string
  projectID: string
  title: string
  time: { created: number; updated: number }
  share?: { url: string }
  summary?: { additions: number; deletions: number; files: number }
}

// Message
type Message = UserMessage | AssistantMessage

type UserMessage = {
  id: string
  role: "user"
  model: { providerID: string; modelID: string }
  agent: string
}

type AssistantMessage = {
  id: string
  role: "assistant"
  tokens: { input: number; output: number }
  cost: number
}

// Parts
type Part = TextPart | ReasoningPart | ToolPart | FilePart | ...
```

### Event Types (Discriminated Union)

```typescript
type Event =
  | { type: "session.created"; properties: Session }
  | { type: "session.started"; properties: { ... } }
  | { type: "message.updated"; properties: { ... } }
  | { type: "file.edited"; properties: { file: string } }
  // ... 20+ event types
```

### Error Types

```typescript
type ProviderAuthError = {
  name: "ProviderAuthError"
  data: { providerID: string; message: string }
}

type ApiError = {
  name: "APIError"
  data: {
    message: string
    statusCode?: number
    isRetryable: boolean
  }
}
```

---

## Response Handling

### Default Mode

```typescript
const result = await client.session.create()

if (result.error) {
  console.error(result.error)
} else {
  console.log(result.data)
}
```

### Throw Mode

```typescript
const result = await client.session.create({
  throwOnError: true
})
// Throws on 4xx/5xx, returns data directly
```

---

## Event Streaming (SSE)

```typescript
// Subscribe to events
const { stream } = await client.event.subscribe({
  query: { directory: "/project" }
})

// Consume with async iterator
for await (const event of stream) {
  switch (event.type) {
    case "session.created":
      console.log("New session:", event.properties.id)
      break
    case "message.updated":
      console.log("Message updated")
      break
  }
}
```

**SSE Features:**
- Auto-reconnect with exponential backoff (3s-30s)
- Last-Event-ID tracking for resumption
- Configurable retry attempts

---

## Server Bootstrapping

### `createOpencodeServer()`

```typescript
const server = await createOpencodeServer({
  hostname: "127.0.0.1",
  port: 4096,
  timeout: 5000,
  config: { /* opencode config */ }
})

// server.url = "http://127.0.0.1:4096"
// server.close() to stop
```

### `createOpencodeTui()`

```typescript
const tui = createOpencodeTui({
  project: "/path/to/project",
  model: "anthropic/claude-3-opus",
  session: "existing-session-id",
  agent: "build"
})

// tui.close() to stop
```

---

## Package Exports

```typescript
// Main
import { createOpencode, createOpencodeClient } from "@opencode-ai/sdk"

// Server only
import { createOpencodeServer } from "@opencode-ai/sdk/server"

// Client only
import { createOpencodeClient } from "@opencode-ai/sdk/client"

// V2 API
import { createOpencode } from "@opencode-ai/sdk/v2"
```

---

## Usage in Other Packages

### Slack Bot

```typescript
const { client, server } = await createOpencode({ port: 0 })

const session = await client.session.create()
const response = await client.session.prompt({
  path: { id: session.data.id },
  body: { parts: [{ type: "text", text: userMessage }] }
})
```

### Desktop App

```typescript
const sdk = createOpencodeClient({
  baseUrl: serverUrl,
  fetch: platform.fetch,  // Tauri HTTP
  directory: projectDir,
})

// Subscribe to events for UI updates
const { stream } = await sdk.event.subscribe()
```

---

## Code Generation

The SDK is generated from OpenAPI spec:

```bash
# In packages/sdk/js
bun run build
```

This runs:
1. Generate OpenAPI spec from core package
2. Run `@hey-api/openapi-ts` to generate TypeScript
3. Format with Prettier
4. Compile with TypeScript

---

## Key Design Patterns

| Pattern | Purpose |
|---------|---------|
| **Namespace Organization** | Logical grouping of methods |
| **Discriminated Unions** | Type-safe event/error handling |
| **AsyncGenerator** | SSE stream consumption |
| **Dual Response Mode** | `{ data, error }` vs throw |
| **Custom Fetch** | Platform-specific HTTP (Tauri, Workers) |

---

## Key Files

| File | Purpose |
|------|---------|
| `js/src/index.ts` | Main entry point |
| `js/src/client.ts` | Client factory |
| `js/src/server.ts` | Server bootstrap |
| `js/src/gen/sdk.gen.ts` | Generated SDK methods |
| `js/src/gen/types.gen.ts` | Generated types |
| `js/openapi.json` | API specification |

---

*Written by Claude (Opus 4.5) | 2026-01-22 17:00 PST*
