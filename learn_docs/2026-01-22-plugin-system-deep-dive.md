# OpenCode Plugin System: Deep Dive

[TOC]

A comprehensive study of OpenCode's plugin architecture - how to extend the system with custom tools, authentication methods, and event hooks.

---

## Overview

OpenCode's plugin system provides multiple extension points:

| Extension Point | Purpose | Example Use Case |
|-----------------|---------|------------------|
| **Tools** | Add custom LLM tools | Database queries, API calls |
| **Auth Hooks** | Custom authentication | OAuth providers, SSO |
| **Event Hooks** | React to system events | Logging, analytics |
| **Transform Hooks** | Modify system behavior | Custom prompts, message filtering |

---

## Architecture

### Two-Layer Design

```
┌─────────────────────────────────────────────────────────────────┐
│                    Plugin Consumer Layer                        │
│                  (packages/opencode/src/plugin/)                │
│                                                                 │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐             │
│  │   Plugin    │  │  BunProc    │  │   Config    │             │
│  │   Loader    │  │  Installer  │  │   Scanner   │             │
│  └──────┬──────┘  └──────┬──────┘  └──────┬──────┘             │
│         │                │                │                     │
│         └────────────────┼────────────────┘                     │
│                          ▼                                      │
│              ┌─────────────────────┐                            │
│              │    Plugin.trigger() │                            │
│              │    Plugin.list()    │                            │
│              │    Plugin.init()    │                            │
│              └─────────────────────┘                            │
└─────────────────────────────────────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────────────┐
│                     Plugin SDK Layer                            │
│                   (packages/plugin/src/)                        │
│                                                                 │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐             │
│  │    Types    │  │    tool()   │  │   BunShell  │             │
│  │   (index)   │  │   Helper    │  │    Types    │             │
│  └─────────────┘  └─────────────┘  └─────────────┘             │
│                                                                 │
│  Exports: Plugin, Hooks, PluginInput, ToolDefinition, etc.     │
└─────────────────────────────────────────────────────────────────┘
```

### Plugin Sources

Plugins can come from three sources:

```typescript
// 1. Internal plugins (compiled into OpenCode)
const INTERNAL_PLUGINS = [CodexAuthPlugin, CopilotAuthPlugin]

// 2. Built-in npm plugins (auto-installed)
const BUILTIN = ["opencode-anthropic-auth@0.0.9", "@gitlab/opencode-gitlab-auth@1.3.0"]

// 3. User-configured plugins (from config)
// In opencode.jsonc:
{
  "plugin": [
    "my-plugin@1.0.0",           // npm package
    "file:///path/to/plugin.ts"  // Local file
  ]
}

// 4. Local plugin files (auto-discovered)
// .opencode/plugin/*.ts or .opencode/plugins/*.ts
```

---

## Plugin SDK Types

**File:** `packages/plugin/src/index.ts`

### Plugin Entry Point

```typescript
export type Plugin = (input: PluginInput) => Promise<Hooks>

export type PluginInput = {
  client: ReturnType<typeof createOpencodeClient>  // SDK client
  project: Project                                  // Current project
  directory: string                                 // Working directory
  worktree: string                                  // Git worktree root
  serverUrl: URL                                    // OpenCode server URL
  $: BunShell                                       // Bun shell helper
}
```

### Hooks Interface

```typescript
export interface Hooks {
  // Core hooks
  event?: (input: { event: Event }) => Promise<void>
  config?: (input: Config) => Promise<void>
  tool?: { [key: string]: ToolDefinition }
  auth?: AuthHook

  // Lifecycle hooks
  "chat.message"?: (input, output) => Promise<void>
  "chat.params"?: (input, output) => Promise<void>
  "permission.ask"?: (input, output) => Promise<void>
  "tool.execute.before"?: (input, output) => Promise<void>
  "tool.execute.after"?: (input, output) => Promise<void>

  // Experimental transform hooks
  "experimental.chat.messages.transform"?: (input, output) => Promise<void>
  "experimental.chat.system.transform"?: (input, output) => Promise<void>
  "experimental.session.compacting"?: (input, output) => Promise<void>
  "experimental.text.complete"?: (input, output) => Promise<void>
}
```

---

## Creating Custom Tools

### Tool Definition Helper

**File:** `packages/plugin/src/tool.ts`

```typescript
import { z } from "zod"

export type ToolContext = {
  sessionID: string
  messageID: string
  agent: string
  abort: AbortSignal
  metadata(input: { title?: string; metadata?: any }): void
  ask(input: AskInput): Promise<void>  // Request permission
}

export function tool<Args extends z.ZodRawShape>(input: {
  description: string
  args: Args
  execute(args: z.infer<z.ZodObject<Args>>, context: ToolContext): Promise<string>
}) {
  return input
}

// Expose zod for schema definitions
tool.schema = z
```

### Simple Tool Example

```typescript
import { Plugin, tool } from "@opencode-ai/plugin"

export const MyPlugin: Plugin = async (ctx) => {
  return {
    tool: {
      greet: tool({
        description: "Greet a person by name",
        args: {
          name: tool.schema.string().describe("The person's name"),
        },
        async execute(args) {
          return `Hello, ${args.name}!`
        },
      }),
    },
  }
}
```

### Advanced Tool with Permissions

```typescript
export const DatabasePlugin: Plugin = async (ctx) => {
  return {
    tool: {
      query_db: tool({
        description: "Execute a SQL query on the database",
        args: {
          query: tool.schema.string().describe("SQL query to execute"),
          database: tool.schema.enum(["prod", "staging", "dev"]).describe("Target database"),
        },
        async execute(args, context) {
          // Request permission before executing
          await context.ask({
            permission: "database",
            patterns: [`${args.database}:query`],
            always: ["dev:query", "staging:query"],  // Auto-approve these
            metadata: { query: args.query, database: args.database },
          })

          // Update metadata for UI display
          context.metadata({
            title: `Query ${args.database}`,
            metadata: { query: args.query },
          })

          const result = await runQuery(args.database, args.query)
          return JSON.stringify(result, null, 2)
        },
      }),
    },
  }
}
```

### Tool Integration in Registry

Tools from plugins are automatically discovered and merged:

```typescript
// packages/opencode/src/tool/registry.ts
export const state = Instance.state(async () => {
  const custom = [] as Tool.Info[]

  // Load from plugins
  const plugins = await Plugin.list()
  for (const plugin of plugins) {
    for (const [id, def] of Object.entries(plugin.tool ?? {})) {
      custom.push(fromPlugin(id, def))
    }
  }

  return { custom }
})
```

---

## Authentication Hooks

### Auth Hook Structure

```typescript
export type AuthHook = {
  provider: string  // e.g., "openai", "github-copilot"

  // Called when auth is needed - can customize fetch behavior
  loader?: (
    auth: () => Promise<Auth>,
    provider: Provider
  ) => Promise<Record<string, any>>

  // Available auth methods
  methods: (OAuthMethod | ApiKeyMethod)[]
}

type OAuthMethod = {
  type: "oauth"
  label: string
  prompts?: PromptDefinition[]
  authorize(inputs?: Record<string, string>): Promise<AuthOauthResult>
}

type ApiKeyMethod = {
  type: "api"
  label: string
  prompts?: PromptDefinition[]
  authorize?(inputs?: Record<string, string>): Promise<AuthResult>
}
```

### OAuth Auth Example (Codex)

```typescript
export async function CodexAuthPlugin(input: PluginInput): Promise<Hooks> {
  return {
    auth: {
      provider: "openai",

      // Custom fetch handler for OAuth tokens
      async loader(getAuth, provider) {
        const auth = await getAuth()
        if (auth.type !== "oauth") return {}

        // Filter models to only Codex-compatible ones
        const allowedModels = new Set(["gpt-5.1-codex-max", "gpt-5.2"])
        for (const modelId of Object.keys(provider.models)) {
          if (!allowedModels.has(modelId)) {
            delete provider.models[modelId]
          }
        }

        return {
          apiKey: OAUTH_DUMMY_KEY,
          async fetch(requestInput, init) {
            const currentAuth = await getAuth()

            // Refresh token if expired
            if (currentAuth.expires < Date.now()) {
              const tokens = await refreshAccessToken(currentAuth.refresh)
              await input.client.auth.set({
                path: { id: "codex" },
                body: { type: "oauth", ...tokens },
              })
            }

            // Set auth header and rewrite URL
            const headers = new Headers(init?.headers)
            headers.set("authorization", `Bearer ${currentAuth.access}`)

            return fetch(CODEX_API_ENDPOINT, { ...init, headers })
          },
        }
      },

      methods: [
        {
          type: "oauth",
          label: "ChatGPT Pro/Plus",
          async authorize() {
            const pkce = await generatePKCE()
            const state = generateState()
            const authUrl = buildAuthorizeUrl(redirectUri, pkce, state)

            return {
              url: authUrl,
              instructions: "Complete authorization in your browser.",
              method: "auto",
              async callback() {
                const tokens = await waitForOAuthCallback(pkce, state)
                return {
                  type: "success",
                  refresh: tokens.refresh_token,
                  access: tokens.access_token,
                  expires: Date.now() + tokens.expires_in * 1000,
                }
              },
            }
          },
        },
        {
          type: "api",
          label: "Manually enter API Key",
        },
      ],
    },
  }
}
```

### Device Flow Auth Example (Copilot)

```typescript
export async function CopilotAuthPlugin(input: PluginInput): Promise<Hooks> {
  return {
    auth: {
      provider: "github-copilot",

      methods: [
        {
          type: "oauth",
          label: "Login with GitHub Copilot",
          prompts: [
            {
              type: "select",
              key: "deploymentType",
              message: "Select GitHub deployment type",
              options: [
                { label: "GitHub.com", value: "github.com" },
                { label: "GitHub Enterprise", value: "enterprise" },
              ],
            },
            {
              type: "text",
              key: "enterpriseUrl",
              message: "Enter your GitHub Enterprise URL",
              condition: (inputs) => inputs.deploymentType === "enterprise",
              validate: (value) => value ? undefined : "URL required",
            },
          ],

          async authorize(inputs = {}) {
            // Device code flow
            const deviceResponse = await fetch(DEVICE_CODE_URL, {
              method: "POST",
              body: JSON.stringify({ client_id: CLIENT_ID, scope: "read:user" }),
            })
            const deviceData = await deviceResponse.json()

            return {
              url: deviceData.verification_uri,
              instructions: `Enter code: ${deviceData.user_code}`,
              method: "auto",

              async callback() {
                // Poll for token
                while (true) {
                  const response = await fetch(ACCESS_TOKEN_URL, {
                    method: "POST",
                    body: JSON.stringify({
                      client_id: CLIENT_ID,
                      device_code: deviceData.device_code,
                      grant_type: "urn:ietf:params:oauth:grant-type:device_code",
                    }),
                  })
                  const data = await response.json()

                  if (data.access_token) {
                    return {
                      type: "success",
                      refresh: data.access_token,
                      access: data.access_token,
                      expires: 0,
                    }
                  }

                  if (data.error === "authorization_pending") {
                    await Bun.sleep(deviceData.interval * 1000)
                    continue
                  }

                  return { type: "failed" }
                }
              },
            }
          },
        },
      ],
    },
  }
}
```

---

## Event and Transform Hooks

### Hook Trigger Mechanism

```typescript
// packages/opencode/src/plugin/index.ts
export async function trigger<Name extends keyof Hooks>(
  name: Name,
  input: Input,
  output: Output
): Promise<Output> {
  for (const hook of await state().then((x) => x.hooks)) {
    const fn = hook[name]
    if (!fn) continue
    await fn(input, output)  // Mutate output in place
  }
  return output
}
```

### Where Hooks Are Called

| Hook | Location | Purpose |
|------|----------|---------|
| `chat.params` | `llm.ts:115` | Modify LLM parameters before call |
| `chat.message` | `prompt.ts:370` | Process incoming user message |
| `permission.ask` | `permission/index.ts:134` | Intercept permission requests |
| `tool.execute.before` | `prompt.ts:696` | Pre-process tool arguments |
| `tool.execute.after` | `prompt.ts:736` | Post-process tool results |
| `experimental.chat.system.transform` | `llm.ts:85` | Modify system prompts |
| `experimental.chat.messages.transform` | `prompt.ts:591` | Transform message history |
| `experimental.session.compacting` | `compaction.ts:136` | Customize compaction |
| `experimental.text.complete` | `processor.ts:308` | Process completed text |

### Transform Hook Example

```typescript
export const LoggingPlugin: Plugin = async (ctx) => {
  return {
    // Log all events
    async event({ event }) {
      console.log(`[${event.type}]`, event)
    },

    // Inject custom context into system prompt
    "experimental.chat.system.transform": async (input, output) => {
      output.system.push(`
        Additional context for this project:
        - Primary language: TypeScript
        - Framework: React
        - Testing: Vitest
      `)
    },

    // Log tool executions
    "tool.execute.before": async (input, output) => {
      console.log(`Tool ${input.tool} called with:`, output.args)
    },

    "tool.execute.after": async (input, output) => {
      console.log(`Tool ${input.tool} returned:`, output.output.substring(0, 100))
    },
  }
}
```

---

## Plugin Loading Lifecycle

### Installation Flow

```typescript
// packages/opencode/src/bun/index.ts
export async function install(pkg: string, version = "latest") {
  using _ = await Lock.write("bun-install")  // Prevent concurrent installs

  const mod = path.join(Global.Path.cache, "node_modules", pkg)
  const pkgjson = Bun.file(path.join(Global.Path.cache, "package.json"))

  // Check cache
  const parsed = await pkgjson.json().catch(() => ({ dependencies: {} }))
  if (parsed.dependencies[pkg] === version && await exists(mod)) {
    return mod  // Already installed
  }

  // Install via Bun
  await BunProc.run([
    "add", "--force", "--exact",
    "--cwd", Global.Path.cache,
    `${pkg}@${version}`,
  ])

  // Update cache manifest
  parsed.dependencies[pkg] = version
  await Bun.write(pkgjson.name!, JSON.stringify(parsed, null, 2))

  return mod
}
```

### Loading Flow

```typescript
// packages/opencode/src/plugin/index.ts
const state = Instance.state(async () => {
  const hooks: Hooks[] = []
  const input: PluginInput = {
    client: createOpencodeClient({ baseUrl: "http://localhost:4096" }),
    project: Instance.project,
    worktree: Instance.worktree,
    directory: Instance.directory,
    serverUrl: Server.url(),
    $: Bun.$,
  }

  // 1. Load internal plugins
  for (const plugin of INTERNAL_PLUGINS) {
    const init = await plugin(input)
    hooks.push(init)
  }

  // 2. Load configured + built-in plugins
  const plugins = [...(config.plugin ?? []), ...BUILTIN]

  for (let plugin of plugins) {
    // Install from npm if needed
    if (!plugin.startsWith("file://")) {
      const [pkg, version] = parsePackageSpec(plugin)
      plugin = await BunProc.install(pkg, version)
    }

    // Dynamic import
    const mod = await import(plugin)

    // Call all exported plugin functions
    for (const fn of Object.values(mod)) {
      const init = await fn(input)
      hooks.push(init)
    }
  }

  return { hooks, input }
})
```

### Initialization Flow

```typescript
export async function init() {
  const hooks = await state().then((x) => x.hooks)
  const config = await Config.get()

  // Call config hook on all plugins
  for (const hook of hooks) {
    await hook.config?.(config)
  }

  // Subscribe to all bus events and forward to plugins
  Bus.subscribeAll(async (input) => {
    for (const hook of hooks) {
      hook.event?.({ event: input })
    }
  })
}
```

---

## Plugin Discovery Locations

### Config-Based

```jsonc
// .opencode/opencode.jsonc or ~/.opencode/opencode.jsonc
{
  "plugin": [
    "my-company-plugin@1.0.0",           // npm package
    "@scope/opencode-tools@latest",       // Scoped package
    "file:///path/to/local/plugin.ts"    // Local file
  ]
}
```

### File-Based (Auto-Discovery)

```
.opencode/
├── plugin/
│   ├── custom-tool.ts      # Auto-loaded
│   └── analytics.ts        # Auto-loaded
└── plugins/                 # Alternative location
    └── my-plugin.ts        # Auto-loaded
```

### Discovery Priority (Highest to Lowest)

1. Local `plugin/` directory
2. Local `opencode.jsonc`
3. Global `~/.opencode/plugin/` directory
4. Global `~/.opencode/opencode.jsonc`
5. Built-in plugins

Later entries override earlier ones with the same name.

---

## Complete Plugin Template

```typescript
// .opencode/plugin/my-plugin.ts
import { Plugin, tool } from "@opencode-ai/plugin"

export const MyPlugin: Plugin = async (ctx) => {
  // Access context
  const { client, project, directory, $ } = ctx

  // Initialize plugin state
  const state = { requestCount: 0 }

  return {
    // Custom tools
    tool: {
      my_tool: tool({
        description: "Does something useful",
        args: {
          input: tool.schema.string().describe("Input value"),
          optional: tool.schema.boolean().optional().describe("Optional flag"),
        },
        async execute(args, context) {
          state.requestCount++

          // Request permission if needed
          await context.ask({
            permission: "my_permission",
            patterns: [args.input],
            always: [],
            metadata: { input: args.input },
          })

          // Update UI metadata
          context.metadata({
            title: `Processing: ${args.input}`,
            metadata: { count: state.requestCount },
          })

          // Do work
          const result = await doSomething(args.input)
          return result
        },
      }),
    },

    // Event logging
    async event({ event }) {
      if (event.type === "session.created") {
        console.log("New session:", event.sessionID)
      }
    },

    // Modify LLM parameters
    "chat.params": async (input, output) => {
      if (input.agent === "plan") {
        output.temperature = 0.3  // Lower temperature for planning
      }
    },

    // Inject system context
    "experimental.chat.system.transform": async (input, output) => {
      output.system.push(`Plugin request count: ${state.requestCount}`)
    },

    // Pre-process tool calls
    "tool.execute.before": async (input, output) => {
      console.log(`[${new Date().toISOString()}] ${input.tool}`, output.args)
    },

    // Post-process tool results
    "tool.execute.after": async (input, output) => {
      // Optionally modify output
      if (output.output.length > 10000) {
        output.output = output.output.substring(0, 10000) + "\n... (truncated)"
      }
    },
  }
}

export default MyPlugin
```

---

## Best Practices

### 1. Handle Errors Gracefully

```typescript
async execute(args, context) {
  try {
    return await riskyOperation()
  } catch (error) {
    // Return error as output rather than throwing
    return `Error: ${error.message}`
  }
}
```

### 2. Use Permission System

```typescript
// Always request permission for sensitive operations
await context.ask({
  permission: "external_api",
  patterns: [`${api_name}:${endpoint}`],
  always: [],  // Don't auto-approve sensitive operations
  metadata: { url, method },
})
```

### 3. Keep Tools Focused

```typescript
// Good: Single-purpose tools
tool: {
  fetch_user: tool({ ... }),
  update_user: tool({ ... }),
  delete_user: tool({ ... }),
}

// Avoid: Multi-purpose tools
tool: {
  user_crud: tool({ ... })  // Too broad
}
```

### 4. Provide Good Descriptions

```typescript
tool({
  description: `
    Fetch stock price data from Yahoo Finance.
    Returns JSON with: symbol, price, change, volume.
    Rate limited to 5 requests per minute.
  `.trim(),
  args: {
    symbol: tool.schema.string()
      .describe("Stock ticker symbol (e.g., AAPL, GOOGL)"),
  },
})
```

---

## Summary

| Component | File | Purpose |
|-----------|------|---------|
| Plugin SDK | `packages/plugin/src/index.ts` | Type definitions, Plugin interface |
| Tool Helper | `packages/plugin/src/tool.ts` | `tool()` function with Zod |
| Plugin Loader | `packages/opencode/src/plugin/index.ts` | Loading, triggering hooks |
| Package Installer | `packages/opencode/src/bun/index.ts` | npm package installation |
| Auth Integration | `packages/opencode/src/provider/auth.ts` | OAuth/API key handling |
| Tool Registry | `packages/opencode/src/tool/registry.ts` | Tool discovery |

The plugin system provides a powerful way to extend OpenCode while maintaining type safety through Zod schemas and proper isolation through the hooks architecture.

---

*Written by Claude (Opus 4.5) | 2026-01-22 15:45 PST*
