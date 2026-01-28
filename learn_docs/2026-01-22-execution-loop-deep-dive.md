# OpenCode Execution Loop Architecture: Deep Dive

[TOC]

This document provides a comprehensive analysis of OpenCode's execution loop architecture, covering the stream processor, LLM abstraction, tool registry, and permission system.

---

## Table of Contents

1. [Stream Processor (`processor.ts`)](#1-stream-processor)
2. [LLM Abstraction Layer (`llm.ts`)](#2-llm-abstraction-layer)
3. [Tool Registry System (`registry.ts`)](#3-tool-registry-system)
4. [Permission System (`permission/next.ts`)](#4-permission-system)
5. [Supporting Components](#5-supporting-components)
6. [Design Patterns Summary](#6-design-patterns-summary)
7. [Application to Financial Agents](#7-application-to-financial-agents)

---

## 1. Stream Processor

**File:** `packages/opencode/src/session/processor.ts` (406 lines)

### Purpose

The processor consumes a stream of events from the LLM and transforms them into persisted state. It's the core engine that handles tool execution, text accumulation, and error recovery.

### Factory Pattern with Closure

```typescript
export function create(input: {
  assistantMessage: MessageV2.Assistant
  sessionID: string
  model: Provider.Model
  abort: AbortSignal
}) {
  // Private state via closure
  const toolcalls: Record<string, MessageV2.ToolPart> = {}
  let snapshot: string | undefined
  let blocked = false
  let attempt = 0
  let needsCompaction = false

  return {
    get message() { return input.assistantMessage },
    partFromToolCall(toolCallID: string) { return toolcalls[toolCallID] },
    async process(streamInput: LLM.StreamInput) { /* ... */ }
  }
}
```

**Key Insight:** The processor uses a factory function that returns an object with closured state. This isolates each processor instance's state completely.

### Event-Driven Processing Loop

The `process()` method contains a nested loop structure:

```
while(true)                    // Outer loop: Retry on transient errors
  └─ for await (event)         // Inner loop: Process stream events
       └─ switch(event.type)   // Event handler dispatch
```

### Event Types and Handlers

| Event Type | Action | Persisted State |
|------------|--------|-----------------|
| `start` | Set session status to "busy" | SessionStatus |
| `reasoning-start` | Create reasoning part | ReasoningPart (time.start) |
| `reasoning-delta` | Accumulate thinking text | ReasoningPart.text += delta |
| `reasoning-end` | Finalize reasoning | ReasoningPart (time.end) |
| `text-start` | Create text part | TextPart (time.start) |
| `text-delta` | Accumulate response text | TextPart.text += delta |
| `text-end` | Finalize text, trigger plugin | TextPart (time.end) |
| `tool-input-start` | Create tool part (pending) | ToolPart (status: "pending") |
| `tool-call` | Mark tool as running, check doom loop | ToolPart (status: "running") |
| `tool-result` | Store completed result | ToolPart (status: "completed") |
| `tool-error` | Store error, check for rejection | ToolPart (status: "error") |
| `start-step` | Take file system snapshot | Snapshot.track() |
| `finish-step` | Calculate usage, create patch | StepFinish, PatchPart |
| `error` | Throw for retry handling | - |

### Doom Loop Detection

Prevents infinite loops when the LLM keeps making identical tool calls:

```typescript
const DOOM_LOOP_THRESHOLD = 3

// Check if last 3 tool calls are identical
const parts = await MessageV2.parts(input.assistantMessage.id)
const lastThree = parts.slice(-DOOM_LOOP_THRESHOLD)

if (
  lastThree.length === DOOM_LOOP_THRESHOLD &&
  lastThree.every(
    (p) =>
      p.type === "tool" &&
      p.tool === value.toolName &&
      p.state.status !== "pending" &&
      JSON.stringify(p.state.input) === JSON.stringify(value.input),
  )
) {
  await PermissionNext.ask({
    permission: "doom_loop",
    patterns: [value.toolName],
    // ... interrupts the loop, asks user
  })
}
```

### Retry with Exponential Backoff

Error handling delegates to `SessionRetry`:

```typescript
catch (e) {
  const error = MessageV2.fromError(e, { providerID: input.model.providerID })
  const retry = SessionRetry.retryable(error)

  if (retry !== undefined) {
    attempt++
    const delay = SessionRetry.delay(attempt, error)  // 2s, 4s, 8s...
    SessionStatus.set(input.sessionID, {
      type: "retry",
      attempt,
      message: retry,
      next: Date.now() + delay,
    })
    await SessionRetry.sleep(delay, input.abort)
    continue  // Retry the entire stream
  }

  // Non-retryable error: store and stop
  input.assistantMessage.error = error
  Bus.publish(Session.Event.Error, { ... })
}
```

### Return Values

The `process()` method returns one of three values:

- `"continue"` - Keep looping in prompt.ts
- `"stop"` - Exit loop (error or permission rejection)
- `"compact"` - Context overflow, needs compaction

---

## 2. LLM Abstraction Layer

**File:** `packages/opencode/src/session/llm.ts` (279 lines)

### Purpose

Provides a unified interface for streaming text from 20+ LLM providers, handling provider-specific quirks, system prompts, and middleware.

### StreamInput Interface

```typescript
export type StreamInput = {
  user: MessageV2.User
  sessionID: string
  model: Provider.Model
  agent: Agent.Info
  system: string[]
  abort: AbortSignal
  messages: ModelMessage[]
  small?: boolean           // Use minimal reasoning
  tools: Record<string, Tool>
  retries?: number
}
```

### System Prompt Assembly

System prompts are assembled in a specific order with provider awareness:

```typescript
const system = SystemPrompt.header(input.model.providerID)  // e.g., Anthropic spoof
system.push(
  [
    // 1. Agent prompt OR provider default prompt
    ...(input.agent.prompt ? [input.agent.prompt] : SystemPrompt.provider(input.model)),
    // 2. Custom prompts passed to this call
    ...input.system,
    // 3. User message custom prompt
    ...(input.user.system ? [input.user.system] : []),
  ]
    .filter(Boolean)
    .join("\n"),
)
```

### Provider-Specific Options

Options are merged from multiple sources in priority order:

```typescript
const options = pipe(
  ProviderTransform.options({ model, sessionID, providerOptions }),  // Base
  mergeDeep(input.model.options),      // Model-specific overrides
  mergeDeep(input.agent.options),      // Agent-specific overrides
  mergeDeep(variant),                  // User variant (reasoning effort)
)
```

### Tool Resolution with Permission Filtering

```typescript
async function resolveTools(input) {
  const disabled = PermissionNext.disabled(Object.keys(input.tools), input.agent.permission)
  for (const tool of Object.keys(input.tools)) {
    if (input.user.tools?.[tool] === false || disabled.has(tool)) {
      delete input.tools[tool]
    }
  }
  return input.tools
}
```

### Middleware Chain

The LLM call uses Vercel AI SDK with middleware:

```typescript
model: wrapLanguageModel({
  model: language,
  middleware: [
    {
      // Provider-specific message transformation
      async transformParams(args) {
        args.params.prompt = ProviderTransform.message(args.params.prompt, input.model, options)
        return args.params
      },
    },
    // Extract <think> tags into reasoning events
    extractReasoningMiddleware({ tagName: "think", startWithReasoning: false }),
  ],
})
```

### LiteLLM Proxy Compatibility

Handles edge case where message history contains tool calls but no tools are active:

```typescript
const isLiteLLMProxy =
  provider.options?.["litellmProxy"] === true ||
  input.model.providerID.toLowerCase().includes("litellm")

if (isLiteLLMProxy && Object.keys(tools).length === 0 && hasToolCalls(input.messages)) {
  tools["_noop"] = tool({
    description: "Placeholder for LiteLLM compatibility",
    execute: async () => ({ output: "", title: "", metadata: {} }),
  })
}
```

---

## 3. Tool Registry System

**File:** `packages/opencode/src/tool/registry.ts` (143 lines)

### Purpose

Manages tool discovery, registration, and initialization. Combines built-in tools, custom tools from config directories, and plugin-provided tools.

### Lazy Initialization with Instance.state

```typescript
export const state = Instance.state(async () => {
  const custom = [] as Tool.Info[]
  const glob = new Bun.Glob("{tool,tools}/*.{js,ts}")

  // Scan config directories for custom tools
  for (const dir of await Config.directories()) {
    for await (const match of glob.scan({ cwd: dir, absolute: true })) {
      const namespace = path.basename(match, path.extname(match))
      const mod = await import(match)
      for (const [id, def] of Object.entries(mod)) {
        custom.push(fromPlugin(namespace, def))
      }
    }
  }

  // Load tools from plugins
  const plugins = await Plugin.list()
  for (const plugin of plugins) {
    for (const [id, def] of Object.entries(plugin.tool ?? {})) {
      custom.push(fromPlugin(id, def))
    }
  }

  return { custom }
})
```

### Built-in Tool List

The `all()` function returns all available tools:

```typescript
async function all(): Promise<Tool.Info[]> {
  const custom = await state().then((x) => x.custom)
  const config = await Config.get()

  return [
    InvalidTool,           // Handles invalid tool calls gracefully
    QuestionTool,          // Ask user questions
    BashTool,              // Execute shell commands
    ReadTool,              // Read file contents
    GlobTool,              // File pattern matching
    GrepTool,              // Content search
    EditTool,              // Edit files
    WriteTool,             // Write files
    TaskTool,              // Spawn sub-agents
    WebFetchTool,          // Fetch web content
    TodoWriteTool,         // Task management
    TodoReadTool,          // Read tasks
    WebSearchTool,         // Web search (conditional)
    CodeSearchTool,        // Code search (conditional)
    SkillTool,             // Invoke skills
    // Conditional tools based on flags
    ...(Flag.OPENCODE_EXPERIMENTAL_LSP_TOOL ? [LspTool] : []),
    ...(config.experimental?.batch_tool ? [BatchTool] : []),
    ...(Flag.OPENCODE_EXPERIMENTAL_PLAN_MODE ? [PlanExitTool, PlanEnterTool] : []),
    ...custom,             // User custom tools
  ]
}
```

### Tool Filtering by Provider

```typescript
export async function tools(providerID: string, agent?: Agent.Info) {
  const tools = await all()
  return Promise.all(
    tools
      .filter((t) => {
        // Enable websearch/codesearch for opencode users OR via flag
        if (t.id === "codesearch" || t.id === "websearch") {
          return providerID === "opencode" || Flag.OPENCODE_ENABLE_EXA
        }
        return true
      })
      .map(async (t) => ({
        id: t.id,
        ...(await t.init({ agent })),  // Lazy initialization
      })),
  )
}
```

### Tool.define() Pattern

**File:** `packages/opencode/src/tool/tool.ts`

The `Tool.define()` factory provides:
1. Zod validation of arguments
2. Automatic output truncation
3. Custom error formatting

```typescript
export function define<Parameters, Result>(
  id: string,
  init: Info["init"],
): Info {
  return {
    id,
    init: async (initCtx) => {
      const toolInfo = await init(initCtx)
      const execute = toolInfo.execute

      // Wrap execute with validation + truncation
      toolInfo.execute = async (args, ctx) => {
        try {
          toolInfo.parameters.parse(args)  // Zod validation
        } catch (error) {
          throw new Error(`Invalid arguments: ${error}`)
        }

        const result = await execute(args, ctx)

        // Auto-truncate unless tool handles it
        if (result.metadata.truncated !== undefined) return result

        const truncated = await Truncate.output(result.output, {}, initCtx?.agent)
        return {
          ...result,
          output: truncated.content,
          metadata: { ...result.metadata, truncated: truncated.truncated },
        }
      }
      return toolInfo
    },
  }
}
```

---

## 4. Permission System

**File:** `packages/opencode/src/permission/next.ts` (269 lines)

### Purpose

Runtime permission checks for tool execution. Evaluates actions against rulesets and manages interactive permission prompts.

### Core Types

```typescript
// Actions a rule can specify
export type Action = "allow" | "deny" | "ask"

// A single permission rule
export type Rule = {
  permission: string    // e.g., "bash", "edit", "read"
  pattern: string       // Wildcard pattern, e.g., "git *", "*.md"
  action: Action
}

// Collection of rules
export type Ruleset = Rule[]

// User's reply to a permission prompt
export type Reply = "once" | "always" | "reject"
```

### Rule Evaluation

Rules are matched using wildcard patterns, with the **last matching rule winning**:

```typescript
export function evaluate(permission: string, pattern: string, ...rulesets: Ruleset[]): Rule {
  const merged = merge(...rulesets)

  // Find the last rule that matches both permission and pattern
  const match = merged.findLast(
    (rule) =>
      Wildcard.match(permission, rule.permission) &&
      Wildcard.match(pattern, rule.pattern)
  )

  // Default to "ask" if no rule matches
  return match ?? { action: "ask", permission, pattern: "*" }
}
```

### The ask() Flow

```typescript
export const ask = fn(Request.partial({ id: true }).extend({ ruleset: Ruleset }),
  async (input) => {
    const s = await state()

    for (const pattern of request.patterns ?? []) {
      const rule = evaluate(request.permission, pattern, ruleset, s.approved)

      // Immediate deny
      if (rule.action === "deny") {
        throw new DeniedError(ruleset.filter(/* matching rules */))
      }

      // Need to ask user
      if (rule.action === "ask") {
        return new Promise<void>((resolve, reject) => {
          const info: Request = { id, ...request }
          s.pending[id] = { info, resolve, reject }
          Bus.publish(Event.Asked, info)  // UI subscribes to this
        })
      }

      // rule.action === "allow" - continue checking other patterns
    }
  }
)
```

### The reply() Flow

Handles user responses to permission prompts:

```typescript
export const reply = fn(z.object({ requestID, reply, message }), async (input) => {
  const s = await state()
  const existing = s.pending[input.requestID]

  switch (input.reply) {
    case "reject":
      // Reject this request and ALL other pending for same session
      existing.reject(input.message ? new CorrectedError(input.message) : new RejectedError())
      for (const [id, pending] of Object.entries(s.pending)) {
        if (pending.info.sessionID === sessionID) {
          delete s.pending[id]
          pending.reject(new RejectedError())
        }
      }
      break

    case "once":
      // Just resolve this one request
      existing.resolve()
      break

    case "always":
      // Add patterns to approved list
      for (const pattern of existing.info.always) {
        s.approved.push({
          permission: existing.info.permission,
          pattern,
          action: "allow",
        })
      }
      existing.resolve()

      // Also resolve any other pending that now match
      for (const [id, pending] of Object.entries(s.pending)) {
        const ok = pending.info.patterns.every(
          (p) => evaluate(pending.info.permission, p, s.approved).action === "allow"
        )
        if (ok) {
          delete s.pending[id]
          pending.resolve()
        }
      }
      break
  }
})
```

### Tool Disabling

Determine which tools are completely disabled by ruleset:

```typescript
const EDIT_TOOLS = ["edit", "write", "patch", "multiedit"]

export function disabled(tools: string[], ruleset: Ruleset): Set<string> {
  const result = new Set<string>()

  for (const tool of tools) {
    const permission = EDIT_TOOLS.includes(tool) ? "edit" : tool
    const rule = ruleset.findLast((r) => Wildcard.match(permission, r.permission))

    // Only disable if rule applies to ALL patterns ("*") and action is deny
    if (rule && rule.pattern === "*" && rule.action === "deny") {
      result.add(tool)
    }
  }

  return result
}
```

### Error Types

```typescript
// User clicked reject without message
export class RejectedError extends Error {
  constructor() {
    super(`The user rejected permission to use this specific tool call.`)
  }
}

// User rejected with feedback
export class CorrectedError extends Error {
  constructor(message: string) {
    super(`User rejected with feedback: ${message}`)
  }
}

// Auto-denied by config rule
export class DeniedError extends Error {
  constructor(public readonly ruleset: Ruleset) {
    super(`Rule prevents this tool call: ${JSON.stringify(ruleset)}`)
  }
}
```

---

## 5. Supporting Components

### Retry System (`retry.ts`)

```typescript
export namespace SessionRetry {
  export const RETRY_INITIAL_DELAY = 2000      // 2 seconds
  export const RETRY_BACKOFF_FACTOR = 2        // Exponential
  export const RETRY_MAX_DELAY_NO_HEADERS = 30_000  // 30 seconds max

  export function delay(attempt: number, error?: APIError) {
    // Check for retry-after headers first
    if (error?.data?.responseHeaders) {
      const retryAfterMs = headers["retry-after-ms"]
      if (retryAfterMs) return Number.parseFloat(retryAfterMs)

      const retryAfter = headers["retry-after"]
      if (retryAfter) return Math.ceil(Number.parseFloat(retryAfter) * 1000)
    }

    // Exponential backoff: 2s, 4s, 8s, 16s...
    return Math.min(
      RETRY_INITIAL_DELAY * Math.pow(RETRY_BACKOFF_FACTOR, attempt - 1),
      RETRY_MAX_DELAY_NO_HEADERS
    )
  }

  export function retryable(error): string | undefined {
    // Check for known retryable conditions
    if (error.data.isRetryable) return error.data.message
    if (error.includes("too_many_requests")) return "Too Many Requests"
    if (error.includes("rate_limit")) return "Rate Limited"
    if (error.includes("server_error")) return "Provider Server Error"
    return undefined  // Not retryable
  }
}
```

### System Prompt Assembly (`system.ts`)

```typescript
export namespace SystemPrompt {
  // Provider-specific headers (e.g., Anthropic spoof for compatibility)
  export function header(providerID: string) {
    if (providerID.includes("anthropic")) return [PROMPT_ANTHROPIC_SPOOF.trim()]
    return []
  }

  // Provider-specific base prompts
  export function provider(model: Provider.Model) {
    if (model.api.id.includes("gpt-5")) return [PROMPT_CODEX]
    if (model.api.id.includes("gpt-") || model.api.id.includes("o1")) return [PROMPT_BEAST]
    if (model.api.id.includes("gemini-")) return [PROMPT_GEMINI]
    if (model.api.id.includes("claude")) return [PROMPT_ANTHROPIC]
    return [PROMPT_ANTHROPIC_WITHOUT_TODO]
  }

  // Load custom instructions from AGENTS.md, CLAUDE.md, etc.
  export async function custom() {
    const paths = new Set<string>()

    // Local rule files (project-level)
    for (const file of ["AGENTS.md", "CLAUDE.md", "CONTEXT.md"]) {
      const matches = await Filesystem.findUp(file, Instance.directory)
      if (matches.length > 0) { paths.add(matches[0]); break }
    }

    // Global rule files (~/.claude/CLAUDE.md, ~/.config/opencode/AGENTS.md)
    for (const globalFile of GLOBAL_RULE_FILES) {
      if (await Bun.file(globalFile).exists()) {
        paths.add(globalFile); break
      }
    }

    return Promise.all(
      Array.from(paths).map(p => Bun.file(p).text())
    )
  }
}
```

### Provider Transform (`transform.ts`)

Handles provider-specific message transformations:

```typescript
export namespace ProviderTransform {
  // Normalize messages for Anthropic (empty content rejection)
  // Normalize tool IDs for Claude (alphanumeric + dash/underscore only)
  // Normalize tool IDs for Mistral (exactly 9 alphanumeric characters)
  // Apply prompt caching for Anthropic (ephemeral cache control)
  // Convert integer enums to strings for Gemini
  // Extract reasoning_content for interleaved thinking models

  export function message(msgs, model, options) {
    msgs = unsupportedParts(msgs, model)     // Replace unsupported modalities
    msgs = normalizeMessages(msgs, model)    // Provider-specific normalization
    if (isAnthropicLike(model)) {
      msgs = applyCaching(msgs, model.providerID)  // Add cache control
    }
    return msgs
  }
}
```

---

## 6. Design Patterns Summary

| Pattern | Component | Purpose |
|---------|-----------|---------|
| **Factory + Closure** | processor.ts | Isolate processor state per instance |
| **Event-Driven Processing** | processor.ts | Handle stream events via switch dispatch |
| **Accumulator** | processor.ts | Build text/reasoning parts incrementally |
| **State Machine** | processor.ts | Track tool lifecycle (pending → running → completed) |
| **Retry with Backoff** | retry.ts | Resilient error handling with exponential delays |
| **Adapter Pattern** | transform.ts | Provider-specific message/option handling |
| **Lazy Initialization** | registry.ts | Load tools on first access via Instance.state |
| **Promise-Based Permission** | next.ts | Async permission prompts with resolve/reject |
| **Wildcard Matching** | next.ts | Flexible permission rules with glob patterns |
| **Middleware Chain** | llm.ts | Composable message transformation |
| **Snapshot + Patch** | processor.ts | Track file system changes for rollback |

---

## 7. Application to Financial Agents

### Agent Configuration Pattern

Define specialized financial agents as `.md` files with YAML frontmatter:

```yaml
---
name: Macro Analyst
permission:
  read: "allow"
  bash: { "curl *": "allow", "*": "deny" }
  edit: "deny"
temperature: 0.3
---

You are a macro economic analyst. You have read-only access to...
```

### Custom Tool Pattern

Create financial tools using `Tool.define()`:

```typescript
// tools/market-data.ts
import { Tool } from "@/tool/tool"
import z from "zod"

export const MarketDataTool = Tool.define("market_data", async () => ({
  description: "Fetch market data for a ticker symbol",
  parameters: z.object({
    ticker: z.string().describe("Stock ticker symbol"),
    period: z.enum(["1d", "1w", "1m", "1y"]).default("1d"),
  }),
  async execute(args, ctx) {
    // Request permission for API call
    await ctx.ask({
      permission: "market_api",
      patterns: [`fetch:${args.ticker}`],
      metadata: { ticker: args.ticker },
      always: ["fetch:*"],
    })

    const data = await fetchMarketData(args.ticker, args.period)
    return {
      title: `${args.ticker} market data`,
      output: JSON.stringify(data, null, 2),
      metadata: { ticker: args.ticker },
    }
  },
}))
```

### Permission Rules for Financial Data

```typescript
const financialRuleset: Ruleset = [
  // Allow reading all files
  { permission: "read", pattern: "*", action: "allow" },

  // Allow market data for specific exchanges
  { permission: "market_api", pattern: "fetch:NYSE:*", action: "allow" },
  { permission: "market_api", pattern: "fetch:NASDAQ:*", action: "allow" },

  // Ask for other exchanges
  { permission: "market_api", pattern: "*", action: "ask" },

  // Never allow trading actions
  { permission: "trade", pattern: "*", action: "deny" },

  // Allow analysis bash commands only
  { permission: "bash", pattern: "python analyze.py *", action: "allow" },
  { permission: "bash", pattern: "*", action: "deny" },
]
```

### Loop Pattern for Multi-Step Analysis

```typescript
async function runAnalysis(agent: Agent.Info, query: string) {
  const session = await Session.create()
  const processor = SessionProcessor.create({
    assistantMessage: await Session.createAssistant(session.id, agent.name),
    sessionID: session.id,
    model: agent.model,
    abort: new AbortController().signal,
  })

  while (true) {
    const result = await processor.process({
      agent,
      tools: await resolveTools(agent),
      messages: await Session.getMessages(session.id),
    })

    if (result === "stop") break
    if (result === "compact") {
      await compactSession(session.id)
      continue
    }

    // Check if analysis is complete
    if (processor.message.finish === "end_turn") break
  }

  return processor.message
}
```

---

## Key Takeaways

1. **Event-driven architecture** - The processor handles a stream of events, making it easy to add new event types or modify handling.

2. **Closure-based isolation** - Each processor instance has completely isolated state through closures.

3. **Permission as a first-class concept** - The permission system is deeply integrated, with rules evaluated at runtime and async prompts.

4. **Provider abstraction** - The LLM layer hides 20+ provider differences behind a unified interface.

5. **Lazy tool loading** - Tools are discovered and initialized on-demand, supporting custom extensions.

6. **Resilient error handling** - Automatic retry with exponential backoff for transient errors.

7. **Wildcard-based rules** - Flexible permission patterns using glob-style matching.

---

*Written by Claude (Opus 4.5) | 2026-01-22 14:30 PST*
