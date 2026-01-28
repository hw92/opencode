# Streaming Patterns in OpenCode

This document provides a comprehensive analysis of streaming patterns used throughout the OpenCode codebase, covering LLM streaming, event streaming, SSE endpoints, async iterators, and backpressure handling.

## Table of Contents

1. [Architecture Overview](#architecture-overview)
2. [LLM Streaming](#llm-streaming)
3. [Event Bus System](#event-bus-system)
4. [SSE Server Implementation](#sse-server-implementation)
5. [Async Iterator Patterns](#async-iterator-patterns)
6. [Stream Processing and Transformation](#stream-processing-and-transformation)
7. [Backpressure and Flow Control](#backpressure-and-flow-control)
8. [Client-Side Event Consumption](#client-side-event-consumption)
9. [Best Practices](#best-practices)

---

## Architecture Overview

OpenCode implements a multi-layered streaming architecture:

```
+------------------+     +------------------+     +------------------+
|   LLM Provider   | --> |  Session Layer   | --> |   Event Bus      |
| (Vercel AI SDK)  |     |  (Processor)     |     |   (Pub/Sub)      |
+------------------+     +------------------+     +------------------+
                                                          |
                                                          v
+------------------+     +------------------+     +------------------+
|   SSE Client     | <-- |   SSE Server     | <-- |   GlobalBus      |
| (SDK/TUI/CLI)    |     |   (Hono)         |     |   (EventEmitter) |
+------------------+     +------------------+     +------------------+
```

**Key Files:**
- `packages/opencode/src/session/llm.ts` - LLM streaming integration
- `packages/opencode/src/session/processor.ts` - Stream processing
- `packages/opencode/src/server/server.ts` - SSE endpoints
- `packages/opencode/src/bus/index.ts` - Event bus
- `packages/sdk/js/src/v2/gen/core/serverSentEvents.gen.ts` - Client SSE

---

## LLM Streaming

### Vercel AI SDK Integration

OpenCode uses the Vercel AI SDK's `streamText` function for streaming LLM responses.

**File:** `packages/opencode/src/session/llm.ts:165-256`

```typescript
import { streamText, wrapLanguageModel, type StreamTextResult } from "ai"

export async function stream(input: StreamInput): Promise<StreamOutput> {
  return streamText({
    onError(error) {
      l.error("stream error", { error })
    },

    // Middleware for model transformation
    model: wrapLanguageModel({
      model: language,
      middleware: [
        {
          async transformParams(args) {
            if (args.type === "stream") {
              args.params.prompt = ProviderTransform.message(
                args.params.prompt, input.model, options
              )
            }
            return args.params
          },
        },
        extractReasoningMiddleware({ tagName: "think", startWithReasoning: false }),
      ],
    }),

    // Streaming configuration
    temperature: params.temperature,
    topP: params.topP,
    maxOutputTokens,
    abortSignal: input.abort,
    tools,
    messages: [/* system + user messages */],
  })
}
```

### Stream Event Types

The AI SDK produces a `fullStream` async iterator with various event types:

**File:** `packages/opencode/src/session/processor.ts:55-336`

```typescript
for await (const value of stream.fullStream) {
  input.abort.throwIfAborted()

  switch (value.type) {
    case "start":
      // Stream started
      SessionStatus.set(input.sessionID, { type: "busy" })
      break

    case "reasoning-start":
    case "reasoning-delta":
    case "reasoning-end":
      // Extended thinking/reasoning tokens
      break

    case "text-start":
    case "text-delta":
    case "text-end":
      // Regular text output
      break

    case "tool-input-start":
    case "tool-input-delta":
    case "tool-input-end":
    case "tool-call":
    case "tool-result":
    case "tool-error":
      // Tool execution lifecycle
      break

    case "start-step":
    case "finish-step":
      // Multi-step execution boundaries
      break

    case "error":
      throw value.error

    case "finish":
      // Stream complete
      break
  }
}
```

### Delta Accumulation Pattern

Text and reasoning are accumulated incrementally:

```typescript
case "text-delta":
  if (currentText) {
    currentText.text += value.text
    if (value.providerMetadata) currentText.metadata = value.providerMetadata
    if (currentText.text)
      await Session.updatePart({
        part: currentText,
        delta: value.text,  // Delta for efficient updates
      })
  }
  break
```

---

## Event Bus System

### Local Event Bus

**File:** `packages/opencode/src/bus/index.ts`

The event bus uses a subscription-based pattern with instance-scoped state:

```typescript
export namespace Bus {
  export async function publish<Definition extends BusEvent.Definition>(
    def: Definition,
    properties: z.output<Definition["properties"]>,
  ) {
    const payload = { type: def.type, properties }

    // Notify local subscribers
    const pending = []
    for (const key of [def.type, "*"]) {
      const match = state().subscriptions.get(key)
      for (const sub of match ?? []) {
        pending.push(sub(payload))
      }
    }

    // Propagate to global bus
    GlobalBus.emit("event", {
      directory: Instance.directory,
      payload,
    })

    return Promise.all(pending)
  }

  export function subscribe<Definition extends BusEvent.Definition>(
    def: Definition,
    callback: (event: {...}) => void,
  ) {
    return raw(def.type, callback)
  }

  export function subscribeAll(callback: (event: any) => void) {
    return raw("*", callback)  // Wildcard subscription
  }
}
```

### Global Event Bus

**File:** `packages/opencode/src/bus/global.ts`

```typescript
import { EventEmitter } from "events"

export const GlobalBus = new EventEmitter<{
  event: [{
    directory?: string
    payload: any
  }]
}>()
```

### Event Definition Pattern

**File:** `packages/opencode/src/bus/bus-event.ts`

```typescript
export namespace BusEvent {
  const registry = new Map<string, Definition>()

  export function define<Type extends string, Properties extends ZodType>(
    type: Type,
    properties: Properties
  ) {
    const result = { type, properties }
    registry.set(type, result)
    return result
  }
}

// Usage examples:
export const Event = {
  Updated: BusEvent.define("message.updated", z.object({ info: Info })),
  PartUpdated: BusEvent.define("message.part.updated", z.object({
    part: Part,
    delta: z.string().optional(),  // Enables incremental updates
  })),
}
```

---

## SSE Server Implementation

### Main Event Endpoint

**File:** `packages/opencode/src/server/server.ts:449-504`

```typescript
import { streamSSE } from "hono/streaming"

.get("/event", async (c) => {
  return streamSSE(c, async (stream) => {
    // Initial connection event
    stream.writeSSE({
      data: JSON.stringify({
        type: "server.connected",
        properties: {},
      }),
    })

    // Subscribe to all events
    const unsub = Bus.subscribeAll(async (event) => {
      await stream.writeSSE({
        data: JSON.stringify(event),
      })

      // Close stream on disposal
      if (event.type === Bus.InstanceDisposed.type) {
        stream.close()
      }
    })

    // Heartbeat to prevent WebView timeout (WKWebView: 60s)
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
      })
    })
  })
})
```

### Global Event Endpoint

**File:** `packages/opencode/src/server/routes/global.ts:65-104`

```typescript
.get("/event", async (c) => {
  return streamSSE(c, async (stream) => {
    stream.writeSSE({
      data: JSON.stringify({
        payload: { type: "server.connected", properties: {} },
      }),
    })

    async function handler(event: any) {
      await stream.writeSSE({
        data: JSON.stringify(event),
      })
    }

    GlobalBus.on("event", handler)

    const heartbeat = setInterval(() => {
      stream.writeSSE({
        data: JSON.stringify({
          payload: { type: "server.heartbeat", properties: {} },
        }),
      })
    }, 30000)

    await new Promise<void>((resolve) => {
      stream.onAbort(() => {
        clearInterval(heartbeat)
        GlobalBus.off("event", handler)
        resolve()
      })
    })
  })
})
```

---

## Async Iterator Patterns

### Generator Function for Message Streaming

**File:** `packages/opencode/src/session/message-v2.ts:571-579`

```typescript
export const stream = fn(Identifier.schema("session"), async function* (sessionID) {
  const list = await Array.fromAsync(await Storage.list(["message", sessionID]))

  // Yield messages in reverse chronological order
  for (let i = list.length - 1; i >= 0; i--) {
    yield await get({
      sessionID,
      messageID: list[i][2],
    })
  }
})
```

### File System Generator with Buffered Reading

**File:** `packages/opencode/src/file/ripgrep.ts:208-266`

```typescript
export async function* files(input: {
  cwd: string
  glob?: string[]
  hidden?: boolean
  follow?: boolean
  maxDepth?: number
}) {
  const proc = Bun.spawn(args, {
    cwd: input.cwd,
    stdout: "pipe",
    stderr: "ignore",
    maxBuffer: 1024 * 1024 * 20,  // 20MB buffer
  })

  const reader = proc.stdout.getReader()
  const decoder = new TextDecoder()
  let buffer = ""

  try {
    while (true) {
      const { done, value } = await reader.read()
      if (done) break

      buffer += decoder.decode(value, { stream: true })

      // Handle both Unix (\n) and Windows (\r\n) line endings
      const lines = buffer.split(/\r?\n/)
      buffer = lines.pop() || ""

      for (const line of lines) {
        if (line) yield line
      }
    }

    // Yield any remaining content
    if (buffer) yield buffer
  } finally {
    reader.releaseLock()
    await proc.exited
  }
}
```

### AsyncQueue Implementation

**File:** `packages/opencode/src/util/queue.ts`

```typescript
export class AsyncQueue<T> implements AsyncIterable<T> {
  private queue: T[] = []
  private resolvers: ((value: T) => void)[] = []

  push(item: T) {
    const resolve = this.resolvers.shift()
    if (resolve) resolve(item)
    else this.queue.push(item)
  }

  async next(): Promise<T> {
    if (this.queue.length > 0) return this.queue.shift()!
    return new Promise((resolve) => this.resolvers.push(resolve))
  }

  async *[Symbol.asyncIterator]() {
    while (true) yield await this.next()
  }
}
```

---

## Stream Processing and Transformation

### Session Processor Pattern

**File:** `packages/opencode/src/session/processor.ts:26-405`

The processor wraps stream consumption with state management:

```typescript
export function create(input: {
  assistantMessage: MessageV2.Assistant
  sessionID: string
  model: Provider.Model
  abort: AbortSignal
}) {
  const toolcalls: Record<string, MessageV2.ToolPart> = {}
  let snapshot: string | undefined
  let blocked = false
  let attempt = 0
  let needsCompaction = false

  return {
    get message() { return input.assistantMessage },

    partFromToolCall(toolCallID: string) {
      return toolcalls[toolCallID]
    },

    async process(streamInput: LLM.StreamInput) {
      const shouldBreak = (await Config.get()).experimental?.continue_loop_on_deny !== true

      while (true) {
        try {
          const stream = await LLM.stream(streamInput)

          for await (const value of stream.fullStream) {
            input.abort.throwIfAborted()
            // ... event handling
          }
        } catch (e: any) {
          // Retry logic with exponential backoff
          const retry = SessionRetry.retryable(error)
          if (retry !== undefined) {
            attempt++
            const delay = SessionRetry.delay(attempt, error)
            await SessionRetry.sleep(delay, input.abort)
            continue
          }
          // ... error handling
        }

        // Exit conditions
        if (needsCompaction) return "compact"
        if (blocked) return "stop"
        if (input.assistantMessage.error) return "stop"
        return "continue"
      }
    }
  }
}
```

### Message Filtering with Async Iteration

**File:** `packages/opencode/src/session/message-v2.ts:604-619`

```typescript
export async function filterCompacted(stream: AsyncIterable<MessageV2.WithParts>) {
  const result = [] as MessageV2.WithParts[]
  const completed = new Set<string>()

  for await (const msg of stream) {
    result.push(msg)

    // Check for compaction marker
    if (
      msg.info.role === "user" &&
      completed.has(msg.info.id) &&
      msg.parts.some((part) => part.type === "compaction")
    ) break

    // Track completed summaries
    if (msg.info.role === "assistant" && msg.info.summary && msg.info.finish)
      completed.add(msg.info.parentID)
  }

  result.reverse()
  return result
}
```

---

## Backpressure and Flow Control

### Event Batching in TUI

**File:** `packages/opencode/src/cli/cmd/tui/context/sdk.tsx:25-55`

Implements a time-based batching strategy to prevent render flooding:

```typescript
let queue: Event[] = []
let timer: Timer | undefined
let last = 0

const flush = () => {
  if (queue.length === 0) return
  const events = queue
  queue = []
  timer = undefined
  last = Date.now()

  // Batch all event emissions for single render
  batch(() => {
    for (const event of events) {
      emitter.emit(event.type, event)
    }
  })
}

const handleEvent = (event: Event) => {
  queue.push(event)
  const elapsed = Date.now() - last

  if (timer) return

  // If flushed recently (within 16ms/60fps), batch with future events
  if (elapsed < 16) {
    timer = setTimeout(flush, 16)
    return
  }

  // Otherwise process immediately
  flush()
}
```

### Worker Event Stream with Reconnection

**File:** `packages/opencode/src/cli/cmd/tui/worker.ts:46-95`

```typescript
const startEventStream = (directory: string) => {
  if (eventStream.abort) eventStream.abort.abort()
  const abort = new AbortController()
  eventStream.abort = abort
  const signal = abort.signal

  ;(async () => {
    while (!signal.aborted) {
      const events = await Promise.resolve(
        sdk.event.subscribe({}, { signal })
      ).catch(() => undefined)

      if (!events) {
        await Bun.sleep(250)  // Backoff on connection failure
        continue
      }

      for await (const event of events.stream) {
        Rpc.emit("event", event as Event)
      }

      if (!signal.aborted) {
        await Bun.sleep(250)  // Brief pause before reconnecting
      }
    }
  })().catch((error) => {
    Log.Default.error("event stream error", { error })
  })
}
```

### SSE Heartbeat for Keep-Alive

Both SSE endpoints implement a 30-second heartbeat:

```typescript
const heartbeat = setInterval(() => {
  stream.writeSSE({
    data: JSON.stringify({
      type: "server.heartbeat",
      properties: {},
    }),
  })
}, 30000)  // WKWebView timeout is 60s
```

---

## Client-Side Event Consumption

### SDK SSE Client

**File:** `packages/sdk/js/src/v2/gen/core/serverSentEvents.gen.ts:78-239`

Full-featured SSE client with retry logic:

```typescript
export const createSseClient = <TData = unknown>(options): ServerSentEventsResult<TData> => {
  let lastEventId: string | undefined

  const createStream = async function* () {
    let retryDelay: number = sseDefaultRetryDelay ?? 3000
    let attempt = 0
    const signal = options.signal ?? new AbortController().signal

    while (true) {
      if (signal.aborted) break
      attempt++

      try {
        const response = await _fetch(request)

        if (!response.ok) throw new Error(`SSE failed: ${response.status}`)
        if (!response.body) throw new Error("No body in SSE response")

        const reader = response.body
          .pipeThrough(new TextDecoderStream())
          .getReader()

        let buffer = ""

        while (true) {
          const { done, value } = await reader.read()
          if (done) break

          buffer += value
          buffer = buffer.replace(/\r\n/g, "\n").replace(/\r/g, "\n")

          const chunks = buffer.split("\n\n")
          buffer = chunks.pop() ?? ""

          for (const chunk of chunks) {
            // Parse SSE format
            const lines = chunk.split("\n")
            const dataLines: Array<string> = []

            for (const line of lines) {
              if (line.startsWith("data:")) {
                dataLines.push(line.replace(/^data:\s*/, ""))
              } else if (line.startsWith("id:")) {
                lastEventId = line.replace(/^id:\s*/, "")
              } else if (line.startsWith("retry:")) {
                const parsed = parseInt(line.replace(/^retry:\s*/, ""), 10)
                if (!isNaN(parsed)) retryDelay = parsed
              }
            }

            if (dataLines.length) {
              const rawData = dataLines.join("\n")
              const data = JSON.parse(rawData)
              yield data
            }
          }
        }

        break  // Exit on normal completion
      } catch (error) {
        onSseError?.(error)

        if (sseMaxRetryAttempts && attempt >= sseMaxRetryAttempts) break

        // Exponential backoff: double each attempt, cap at 30s
        const backoff = Math.min(retryDelay * 2 ** (attempt - 1), sseMaxRetryDelay ?? 30000)
        await sleep(backoff)
      }
    }
  }

  return { stream: createStream() }
}
```

### CLI Event Consumer

**File:** `packages/opencode/src/cli/cmd/run.ts:154-229`

```typescript
const events = await sdk.event.subscribe()
let errorMsg: string | undefined

const eventProcessor = (async () => {
  for await (const event of events.stream) {
    if (event.type === "message.part.updated") {
      const part = event.properties.part
      if (part.sessionID !== sessionID) continue

      if (part.type === "tool" && part.state.status === "completed") {
        printEvent(color, tool, title)
      }

      if (part.type === "text" && part.time?.end) {
        process.stdout.write(UI.markdown(part.text) + EOL)
      }
    }

    if (event.type === "session.error") {
      errorMsg = String(props.error.data.message)
      UI.error(errorMsg)
    }

    if (event.type === "session.idle" && event.properties.sessionID === sessionID) {
      break  // Session completed
    }

    if (event.type === "permission.asked") {
      // Interactive permission handling
      const result = await select({ /* ... */ })
      await sdk.permission.respond({ /* ... */ })
    }
  }
})()

// Wait for event processing to complete
await eventProcessor
```

---

## Best Practices

### 1. Use Abort Signals Consistently

```typescript
// Always check abort before expensive operations
input.abort.throwIfAborted()

// Pass abort signals through the entire chain
const stream = await LLM.stream({ ...input, abort: input.abort })
```

### 2. Implement Proper Cleanup

```typescript
try {
  // Stream processing
} finally {
  reader.releaseLock()
  await proc.exited
  unsub()
  clearInterval(heartbeat)
}
```

### 3. Handle Reconnection with Backoff

```typescript
while (!signal.aborted) {
  const events = await sdk.event.subscribe().catch(() => undefined)
  if (!events) {
    await sleep(250)  // Brief backoff
    continue
  }
  // Process events...
}
```

### 4. Batch High-Frequency Updates

```typescript
// Buffer events and flush at 60fps intervals
if (elapsed < 16) {
  timer = setTimeout(flush, 16)
  return
}
flush()
```

### 5. Use Delta Updates for Efficiency

```typescript
// Send only the change, not the entire content
await Session.updatePart({
  part: currentText,
  delta: value.text,  // Just the new content
})
```

### 6. Implement Heartbeats for Long-Running Connections

```typescript
// Prevent timeout in WebViews and proxies
const heartbeat = setInterval(() => {
  stream.writeSSE({ type: "heartbeat", properties: {} })
}, 30000)
```

### 7. Use Typed Event Definitions

```typescript
export const Event = {
  Updated: BusEvent.define("message.updated", z.object({ info: Info })),
  PartUpdated: BusEvent.define("message.part.updated", z.object({
    part: Part,
    delta: z.string().optional(),
  })),
}
```

---

## Summary

OpenCode's streaming architecture demonstrates several key patterns:

| Pattern | Purpose | Location |
|---------|---------|----------|
| Vercel AI SDK `streamText` | LLM response streaming | `session/llm.ts` |
| Event Bus (Pub/Sub) | Decoupled component communication | `bus/index.ts` |
| SSE with Hono | Real-time client updates | `server/server.ts` |
| Async Generators | Lazy iteration over large datasets | `file/ripgrep.ts` |
| Event Batching | UI performance optimization | `cli/cmd/tui/context/sdk.tsx` |
| Exponential Backoff | Resilient reconnection | SDK SSE client |
| Heartbeats | Connection keep-alive | SSE endpoints |

---

*Written by Claude (Opus 4.5) | 2026-01-22 13:45 PST*
