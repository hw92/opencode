# OpenCode Server/API Deep Dive

[TOC]

This document provides an in-depth analysis of OpenCode's HTTP server architecture, built on the Hono framework with Bun runtime. The server powers both the TUI client and potential external integrations through a well-structured REST API with real-time event streaming.

## Architecture Overview

```
+------------------------------------------------------------------+
|                       OpenCode Server                             |
+------------------------------------------------------------------+
|                                                                   |
|  +------------------------+    +-----------------------------+   |
|  |    Hono Framework     |    |      Bun Runtime            |   |
|  |  - Routing            |    |  - HTTP Server              |   |
|  |  - Middleware Chain   |    |  - WebSocket Support        |   |
|  |  - Request Handling   |    |  - Native Performance       |   |
|  +------------------------+    +-----------------------------+   |
|                                                                   |
|  +-------------------------------------------------------------+ |
|  |                    Middleware Stack                          | |
|  | +----------+ +----------+ +----------+ +------------------+  | |
|  | | Basic    | | Request  | | CORS     | | Instance         |  | |
|  | | Auth     | | Logging  | | Handler  | | Context Provider |  | |
|  | +----------+ +----------+ +----------+ +------------------+  | |
|  +-------------------------------------------------------------+ |
|                                                                   |
|  +-------------------------------------------------------------+ |
|  |                     Route Modules                            | |
|  | +----------+ +----------+ +----------+ +----------+          | |
|  | | /global  | | /session | | /project | | /config  |          | |
|  | +----------+ +----------+ +----------+ +----------+          | |
|  | +----------+ +----------+ +----------+ +----------+          | |
|  | | /pty     | | /mcp     | | /tui     | | /provider|          | |
|  | +----------+ +----------+ +----------+ +----------+          | |
|  | +----------+ +----------+ +----------+ +----------+          | |
|  | | /file    | | /question| | /permission | /experimental      | |
|  | +----------+ +----------+ +----------+ +----------+          | |
|  +-------------------------------------------------------------+ |
|                                                                   |
|  +-------------------------------------------------------------+ |
|  |                  Real-Time Features                          | |
|  | +-------------------------+ +----------------------------+   | |
|  | | SSE Event Streaming    | | WebSocket PTY Connection   |   | |
|  | | - /event               | | - /pty/:id/connect         |   | |
|  | | - /global/event        | |                            |   | |
|  | +-------------------------+ +----------------------------+   | |
|  +-------------------------------------------------------------+ |
|                                                                   |
|  +-------------------------------------------------------------+ |
|  |                  OpenAPI Generation                          | |
|  | +------------------+ +------------------------------------+  | |
|  | | hono-openapi     | | Zod Schema Validation              |  | |
|  | | - describeRoute  | | - Request/Response Types           |  | |
|  | | - resolver       | | - Auto-generated Documentation     |  | |
|  | +------------------+ +------------------------------------+  | |
|  +-------------------------------------------------------------+ |
+------------------------------------------------------------------+
```

## Key Files and Their Purposes

### Core Server Files

| File | Purpose |
|------|---------|
| `/packages/opencode/src/server/server.ts` | Main server setup, middleware chain, root routes, SSE event streaming, server lifecycle management |
| `/packages/opencode/src/server/error.ts` | Centralized error response definitions (400, 404 schemas) |
| `/packages/opencode/src/server/mdns.ts` | mDNS service discovery using Bonjour for local network discovery |

### Route Modules

| File | Prefix | Purpose |
|------|--------|---------|
| `routes/global.ts` | `/global` | Health checks, global events SSE, dispose all instances |
| `routes/session.ts` | `/session` | Full session CRUD, messages, prompts, fork/share, revert |
| `routes/project.ts` | `/project` | Project listing and current project info |
| `routes/config.ts` | `/config` | Configuration get/update, provider listing |
| `routes/pty.ts` | `/pty` | PTY session management, WebSocket terminal connections |
| `routes/mcp.ts` | `/mcp` | MCP server status, OAuth, connect/disconnect |
| `routes/tui.ts` | `/tui` | TUI-specific commands, event publishing, session control |
| `routes/file.ts` | `/` (root) | File search (ripgrep), file listing, content reading |
| `routes/provider.ts` | `/provider` | Provider listing, OAuth authorization |
| `routes/permission.ts` | `/permission` | Permission request listing and replies |
| `routes/question.ts` | `/question` | Question request handling and responses |
| `routes/experimental.ts` | `/experimental` | Tools, worktrees, MCP resources |

## Server Initialization

The server is created using Bun's native HTTP server with Hono as the routing framework:

```typescript
// server.ts - Server listen function
export function listen(opts: { port: number; hostname: string; mdns?: boolean; cors?: string[] }) {
  _corsWhitelist = opts.cors ?? []

  const args = {
    hostname: opts.hostname,
    idleTimeout: 0,
    fetch: App().fetch,
    websocket: websocket,
  } as const

  const tryServe = (port: number) => {
    try {
      return Bun.serve({ ...args, port })
    } catch {
      return undefined
    }
  }

  // Smart port selection: try 4096 first, then fallback to random
  const server = opts.port === 0 ? (tryServe(4096) ?? tryServe(0)) : tryServe(opts.port)
  if (!server) throw new Error(`Failed to start server on port ${opts.port}`)

  _url = server.url

  // Optional mDNS publishing for network discovery
  const shouldPublishMDNS = opts.mdns &&
    server.port &&
    opts.hostname !== "127.0.0.1" &&
    opts.hostname !== "localhost" &&
    opts.hostname !== "::1"

  if (shouldPublishMDNS) {
    MDNS.publish(server.port!, `opencode-${server.port!}`)
  }

  return server
}
```

## Middleware Chain

The server uses a carefully ordered middleware stack:

### 1. Global Error Handler

```typescript
app.onError((err, c) => {
  log.error("failed", { error: err })

  // Named errors get appropriate HTTP status codes
  if (err instanceof NamedError) {
    let status: ContentfulStatusCode
    if (err instanceof Storage.NotFoundError) status = 404
    else if (err instanceof Provider.ModelNotFoundError) status = 400
    else if (err.name.startsWith("Worktree")) status = 400
    else status = 500
    return c.json(err.toObject(), { status })
  }

  // HTTP exceptions pass through
  if (err instanceof HTTPException) return err.getResponse()

  // Unknown errors become 500
  const message = err instanceof Error && err.stack ? err.stack : err.toString()
  return c.json(new NamedError.Unknown({ message }).toObject(), { status: 500 })
})
```

### 2. Basic Authentication (Optional)

```typescript
.use((c, next) => {
  const password = Flag.OPENCODE_SERVER_PASSWORD
  if (!password) return next()  // Skip if no password configured
  const username = Flag.OPENCODE_SERVER_USERNAME ?? "opencode"
  return basicAuth({ username, password })(c, next)
})
```

### 3. Request Logging

```typescript
.use(async (c, next) => {
  const skipLogging = c.req.path === "/log"  // Avoid recursive logging
  if (!skipLogging) {
    log.info("request", { method: c.req.method, path: c.req.path })
  }
  const timer = log.time("request", { method: c.req.method, path: c.req.path })
  await next()
  if (!skipLogging) {
    timer.stop()
  }
})
```

### 4. CORS Configuration

```typescript
.use(cors({
  origin(input) {
    if (!input) return

    // Allow localhost development
    if (input.startsWith("http://localhost:")) return input
    if (input.startsWith("http://127.0.0.1:")) return input

    // Allow Tauri desktop app
    if (input === "tauri://localhost" || input === "http://tauri.localhost") return input

    // Allow *.opencode.ai (HTTPS only)
    if (/^https:\/\/([a-z0-9-]+\.)*opencode\.ai$/.test(input)) {
      return input
    }

    // Allow custom whitelist
    if (_corsWhitelist.includes(input)) {
      return input
    }

    return  // Deny unknown origins
  },
}))
```

### 5. Instance Context Provider

After the `/global` routes, all other routes require an instance context:

```typescript
.use(async (c, next) => {
  // Get directory from query param, header, or default to cwd
  let directory = c.req.query("directory") ||
                  c.req.header("x-opencode-directory") ||
                  process.cwd()
  try {
    directory = decodeURIComponent(directory)
  } catch {
    // fallback to original value
  }

  // Provide instance context for the request
  return Instance.provide({
    directory,
    init: InstanceBootstrap,
    async fn() {
      return next()
    },
  })
})
```

## Route Patterns and OpenAPI Integration

### Lazy Route Initialization

All route modules use lazy initialization to defer loading until needed:

```typescript
import { lazy } from "../../util/lazy"

export const SessionRoutes = lazy(() =>
  new Hono()
    .get("/", /* ... */)
    .post("/", /* ... */)
    // ...
)
```

### Route Definition Pattern

Each route uses `hono-openapi` decorators for automatic OpenAPI spec generation:

```typescript
import { describeRoute, validator, resolver } from "hono-openapi"

.get(
  "/:sessionID",
  describeRoute({
    summary: "Get session",
    description: "Retrieve detailed information about a specific OpenCode session.",
    tags: ["Session"],
    operationId: "session.get",
    responses: {
      200: {
        description: "Get session",
        content: {
          "application/json": {
            schema: resolver(Session.Info),
          },
        },
      },
      ...errors(400, 404),  // Include standard error responses
    },
  }),
  validator(
    "param",
    z.object({
      sessionID: Session.get.schema,
    }),
  ),
  async (c) => {
    const sessionID = c.req.valid("param").sessionID
    const session = await Session.get(sessionID)
    return c.json(session)
  },
)
```

### Error Response Helper

```typescript
// error.ts
export const ERRORS = {
  400: {
    description: "Bad request",
    content: {
      "application/json": {
        schema: resolver(
          z.object({
            data: z.any(),
            errors: z.array(z.record(z.string(), z.any())),
            success: z.literal(false),
          }).meta({ ref: "BadRequestError" }),
        ),
      },
    },
  },
  404: {
    description: "Not found",
    content: {
      "application/json": {
        schema: resolver(Storage.NotFoundError.Schema),
      },
    },
  },
} as const

export function errors(...codes: number[]) {
  return Object.fromEntries(codes.map((code) => [code, ERRORS[code as keyof typeof ERRORS]]))
}
```

## SSE (Server-Sent Events) Implementation

### Instance-Scoped Events (`/event`)

The main event stream for a specific project instance:

```typescript
.get("/event",
  describeRoute({
    summary: "Subscribe to events",
    description: "Get events",
    operationId: "event.subscribe",
    responses: {
      200: {
        description: "Event stream",
        content: {
          "text/event-stream": {
            schema: resolver(BusEvent.payloads()),
          },
        },
      },
    },
  }),
  async (c) => {
    log.info("event connected")
    return streamSSE(c, async (stream) => {
      // Send initial connection event
      stream.writeSSE({
        data: JSON.stringify({
          type: "server.connected",
          properties: {},
        }),
      })

      // Subscribe to all bus events
      const unsub = Bus.subscribeAll(async (event) => {
        await stream.writeSSE({
          data: JSON.stringify(event),
        })
        // Close stream when instance is disposed
        if (event.type === Bus.InstanceDisposed.type) {
          stream.close()
        }
      })

      // Heartbeat every 30s (prevents WKWebView 60s timeout)
      const heartbeat = setInterval(() => {
        stream.writeSSE({
          data: JSON.stringify({
            type: "server.heartbeat",
            properties: {},
          }),
        })
      }, 30000)

      // Cleanup on disconnect
      await new Promise<void>((resolve) => {
        stream.onAbort(() => {
          clearInterval(heartbeat)
          unsub()
          resolve()
          log.info("event disconnected")
        })
      })
    })
  },
)
```

### Global Events (`/global/event`)

Events that span all instances:

```typescript
.get("/event",
  describeRoute({
    summary: "Get global events",
    description: "Subscribe to global events from the OpenCode system using server-sent events.",
    operationId: "global.event",
    responses: {
      200: {
        description: "Event stream",
        content: {
          "text/event-stream": {
            schema: resolver(
              z.object({
                directory: z.string(),
                payload: BusEvent.payloads(),
              }).meta({ ref: "GlobalEvent" }),
            ),
          },
        },
      },
    },
  }),
  async (c) => {
    log.info("global event connected")
    return streamSSE(c, async (stream) => {
      // Initial connection
      stream.writeSSE({
        data: JSON.stringify({
          payload: {
            type: "server.connected",
            properties: {},
          },
        }),
      })

      async function handler(event: any) {
        await stream.writeSSE({
          data: JSON.stringify(event),
        })
      }
      GlobalBus.on("event", handler)

      // Heartbeat every 30s
      const heartbeat = setInterval(() => {
        stream.writeSSE({
          data: JSON.stringify({
            payload: {
              type: "server.heartbeat",
              properties: {},
            },
          }),
        })
      }, 30000)

      await new Promise<void>((resolve) => {
        stream.onAbort(() => {
          clearInterval(heartbeat)
          GlobalBus.off("event", handler)
          resolve()
          log.info("global event disconnected")
        })
      })
    })
  },
)
```

## Event Bus System

The server uses a sophisticated event bus for publishing state changes:

### BusEvent Definition

```typescript
// bus/bus-event.ts
export namespace BusEvent {
  const registry = new Map<string, Definition>()

  export function define<Type extends string, Properties extends ZodType>(type: Type, properties: Properties) {
    const result = { type, properties }
    registry.set(type, result)
    return result
  }

  // Generates OpenAPI schema for all registered events
  export function payloads() {
    return z.discriminatedUnion(
      "type",
      registry.entries()
        .map(([type, def]) => {
          return z.object({
            type: z.literal(type),
            properties: def.properties,
          }).meta({ ref: "Event" + "." + def.type })
        })
        .toArray() as any,
    ).meta({ ref: "Event" })
  }
}
```

### Bus Publish/Subscribe

```typescript
// bus/index.ts
export namespace Bus {
  export async function publish<Definition extends BusEvent.Definition>(
    def: Definition,
    properties: z.output<Definition["properties"]>,
  ) {
    const payload = { type: def.type, properties }
    log.info("publishing", { type: def.type })

    const pending = []
    for (const key of [def.type, "*"]) {
      const match = state().subscriptions.get(key)
      for (const sub of match ?? []) {
        pending.push(sub(payload))
      }
    }

    // Also publish to global bus for cross-instance events
    GlobalBus.emit("event", {
      directory: Instance.directory,
      payload,
    })

    return Promise.all(pending)
  }

  export function subscribeAll(callback: (event: any) => void) {
    return raw("*", callback)
  }
}
```

### Common Event Types

```typescript
// Session events
Session.Event = {
  Created: BusEvent.define("session.created", Session.Info),
  Updated: BusEvent.define("session.updated", Session.Info),
  Deleted: BusEvent.define("session.deleted", z.object({ sessionID: z.string() })),
}

// Message events
MessageV2.Event = {
  Updated: BusEvent.define("message.updated", /* ... */),
  PartUpdated: BusEvent.define("message.part.updated", /* ... */),
}

// PTY events
Pty.Event = {
  Created: BusEvent.define("pty.created", z.object({ info: Info })),
  Exited: BusEvent.define("pty.exited", z.object({ id: ..., exitCode: z.number() })),
}
```

## WebSocket PTY Connection

Terminal sessions use WebSocket for bidirectional communication:

```typescript
// routes/pty.ts
.get(
  "/:ptyID/connect",
  describeRoute({
    summary: "Connect to PTY session",
    description: "Establish a WebSocket connection to interact with a pseudo-terminal (PTY) session in real-time.",
    operationId: "pty.connect",
    // ...
  }),
  validator("param", z.object({ ptyID: z.string() })),
  upgradeWebSocket((c) => {
    const id = c.req.param("ptyID")
    let handler: ReturnType<typeof Pty.connect>
    if (!Pty.get(id)) throw new Error("Session not found")
    return {
      onOpen(_event, ws) {
        handler = Pty.connect(id, ws)
      },
      onMessage(event) {
        handler?.onMessage(String(event.data))
      },
      onClose() {
        handler?.onClose()
      },
    }
  }),
)
```

## TUI Communication Pattern

The TUI uses an async queue pattern for request/response coordination:

```typescript
// routes/tui.ts
const request = new AsyncQueue<TuiRequest>()
const response = new AsyncQueue<any>()

export async function callTui(ctx: Context) {
  const body = await ctx.req.json()
  request.push({
    path: ctx.req.path,
    body,
  })
  return response.next()  // Wait for TUI to process and respond
}

// TUI polls for next request
.get("/control/next", async (c) => {
  const req = await request.next()  // Blocks until request available
  return c.json(req)
})

// TUI sends response
.post("/control/response", validator("json", z.any()), async (c) => {
  const body = c.req.valid("json")
  response.push(body)
  return c.json(true)
})
```

### TUI Event Publishing

The TUI routes allow publishing events to control the terminal interface:

```typescript
.post("/execute-command",
  validator("json", z.object({ command: z.string() })),
  async (c) => {
    const command = c.req.valid("json").command
    await Bus.publish(TuiEvent.CommandExecute, {
      command: {
        session_new: "session.new",
        session_share: "session.share",
        session_interrupt: "session.interrupt",
        agent_cycle: "agent.cycle",
        // ... more command mappings
      }[command],
    })
    return c.json(true)
  },
)
```

## OpenAPI Schema Generation

The server can generate a complete OpenAPI specification:

```typescript
// server.ts
export async function openapi() {
  const result = await generateSpecs(App() as Hono, {
    documentation: {
      info: {
        title: "opencode",
        version: "1.0.0",
        description: "opencode api",
      },
      openapi: "3.1.1",
    },
  })
  return result
}

// Available at /doc endpoint
.get("/doc",
  openAPIRouteHandler(app, {
    documentation: {
      info: {
        title: "opencode",
        version: "0.0.3",
        description: "opencode api",
      },
      openapi: "3.1.1",
    },
  }),
)
```

## Proxy Fallback

Unknown routes are proxied to the OpenCode web app:

```typescript
.all("/*", async (c) => {
  const path = c.req.path
  const response = await proxy(`https://app.opencode.ai${path}`, {
    ...c.req,
    headers: {
      ...c.req.raw.headers,
      host: "app.opencode.ai",
    },
  })
  response.headers.set(
    "Content-Security-Policy",
    "default-src 'self'; script-src 'self'; style-src 'self' 'unsafe-inline'; img-src 'self' data: https:; font-src 'self' data:; connect-src 'self'",
  )
  return response
})
```

## mDNS Service Discovery

For local network discovery, the server can advertise itself via mDNS:

```typescript
// mdns.ts
export namespace MDNS {
  let bonjour: Bonjour | undefined

  export function publish(port: number, name = "opencode") {
    if (currentPort === port) return
    if (bonjour) unpublish()

    try {
      bonjour = new Bonjour()
      const service = bonjour.publish({
        name,
        type: "http",
        port,
        txt: { path: "/" },
      })

      service.on("up", () => {
        log.info("mDNS service published", { name, port })
      })

      service.on("error", (err) => {
        log.error("mDNS service error", { error: err })
      })

      currentPort = port
    } catch (err) {
      log.error("mDNS publish failed", { error: err })
      // cleanup on error
    }
  }

  export function unpublish() {
    if (bonjour) {
      bonjour.unpublishAll()
      bonjour.destroy()
      bonjour = undefined
      currentPort = undefined
      log.info("mDNS service unpublished")
    }
  }
}
```

## Design Patterns Summary

### 1. Namespace Pattern
All modules use TypeScript namespaces for organization:
```typescript
export namespace Server { /* ... */ }
export namespace Bus { /* ... */ }
export namespace MDNS { /* ... */ }
```

### 2. Lazy Initialization
Route modules defer initialization until first access:
```typescript
export const SessionRoutes = lazy(() => new Hono() /* ... */)
```

### 3. Zod-First Schema Design
All data types start with Zod schemas, which generate:
- TypeScript types via `z.infer<>`
- OpenAPI schemas via `hono-openapi`
- Runtime validation

### 4. Event-Driven Architecture
State changes are published via the Bus system:
- Instance-scoped events via `Bus.publish()`
- Cross-instance events via `GlobalBus`
- SSE streaming delivers events to clients

### 5. Context Providers
Async local storage provides request-scoped state:
```typescript
Instance.provide({
  directory,
  init: InstanceBootstrap,
  async fn() { return next() }
})
```

### 6. Declarative Route Definitions
Routes are self-documenting with OpenAPI decorators:
```typescript
describeRoute({
  summary: "...",
  description: "...",
  operationId: "...",
  responses: { /* typed responses */ }
})
```

## API Endpoint Summary

| Method | Path | Description |
|--------|------|-------------|
| GET | `/global/health` | Server health check |
| GET | `/global/event` | Global SSE event stream |
| POST | `/global/dispose` | Dispose all instances |
| GET | `/event` | Instance SSE event stream |
| GET | `/doc` | OpenAPI specification |
| GET/POST/PATCH/DELETE | `/session/*` | Session management |
| GET/POST/PUT/DELETE | `/pty/*` | PTY session management |
| GET | `/pty/:id/connect` | WebSocket PTY connection |
| GET/PATCH | `/config/*` | Configuration management |
| GET/POST | `/mcp/*` | MCP server management |
| GET/POST | `/provider/*` | Provider and OAuth management |
| GET | `/find/*` | Text/file/symbol search |
| GET | `/file/*` | File operations |
| POST | `/tui/*` | TUI command execution |
| GET/POST | `/permission/*` | Permission handling |
| GET/POST | `/question/*` | Question handling |
| GET/POST | `/experimental/*` | Experimental features |
| GET | `/path` | Path information |
| GET | `/vcs` | Version control info |
| GET | `/command` | List commands |
| GET | `/agent` | List agents |
| GET | `/skill` | List skills |
| GET | `/lsp` | LSP status |
| GET | `/formatter` | Formatter status |
| PUT | `/auth/:providerID` | Set auth credentials |
| POST | `/instance/dispose` | Dispose current instance |
| POST | `/log` | Write log entry |

---

*Written by Claude (Opus 4.5) | 2026-01-22 01:52 PST*
