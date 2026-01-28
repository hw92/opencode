# OpenCode MCP Integration: Deep Dive

[TOC]

A comprehensive study of OpenCode's Model Context Protocol (MCP) integration - how to extend LLM capabilities with external tools, resources, and prompts via a standardized protocol.

---

## What is MCP?

**Model Context Protocol (MCP)** is an open standard for connecting LLMs to external data sources and tools. It provides a standardized way for:

- **Tools** - Functions the LLM can call (database queries, API calls, file operations)
- **Resources** - Data the LLM can read (documents, configuration, live data)
- **Prompts** - Pre-defined prompt templates from servers

MCP uses a client-server architecture where OpenCode acts as the **client** connecting to **MCP servers**.

---

## Architecture Overview

```
┌─────────────────────────────────────────────────────────────────┐
│                       OpenCode                                   │
│                                                                  │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐          │
│  │  MCP Client  │  │  MCP Client  │  │  MCP Client  │          │
│  │  (server-a)  │  │  (server-b)  │  │  (server-c)  │          │
│  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘          │
│         │                 │                 │                   │
└─────────┼─────────────────┼─────────────────┼───────────────────┘
          │                 │                 │
          ▼                 ▼                 ▼
    ┌──────────┐      ┌──────────┐      ┌──────────┐
    │  Local   │      │  Remote  │      │  Remote  │
    │  Process │      │  HTTP    │      │  SSE     │
    │  (stdio) │      │ (OAuth)  │      │          │
    └──────────┘      └──────────┘      └──────────┘
```

### Transport Types

| Type | Config | Connection Method |
|------|--------|-------------------|
| **Local** | `type: "local"` | Spawns process, communicates via stdio |
| **Remote HTTP** | `type: "remote"` | StreamableHTTP or SSE transport |

---

## Core Components

### File Structure

```
packages/opencode/src/mcp/
├── index.ts           # Main MCP namespace - client management, tools, resources
├── auth.ts            # OAuth token/client storage
├── oauth-provider.ts  # OAuthClientProvider implementation
└── oauth-callback.ts  # Local HTTP server for OAuth callbacks
```

### MCP Namespace (`index.ts`)

**File:** `packages/opencode/src/mcp/index.ts` (927 lines)

The main namespace provides:

```typescript
export namespace MCP {
  // Connection management
  export async function add(name: string, mcp: Config.Mcp): Promise<{ status: Status }>
  export async function connect(name: string): Promise<void>
  export async function disconnect(name: string): Promise<void>
  export async function status(): Promise<Record<string, Status>>
  export async function clients(): Promise<Record<string, MCPClient>>

  // Tool/Resource/Prompt access
  export async function tools(): Promise<Record<string, Tool>>
  export async function prompts(): Promise<Record<string, PromptInfo>>
  export async function resources(): Promise<Record<string, ResourceInfo>>
  export async function getPrompt(client: string, name: string, args?: Record<string, string>)
  export async function readResource(client: string, uri: string)

  // OAuth authentication
  export async function startAuth(mcpName: string): Promise<{ authorizationUrl: string }>
  export async function authenticate(mcpName: string): Promise<Status>
  export async function finishAuth(mcpName: string, code: string): Promise<Status>
  export async function removeAuth(mcpName: string): Promise<void>
  export async function getAuthStatus(mcpName: string): Promise<AuthStatus>
}
```

### Connection States

```typescript
export const Status = z.discriminatedUnion("status", [
  z.object({ status: z.literal("connected") }),
  z.object({ status: z.literal("disabled") }),
  z.object({ status: z.literal("failed"), error: z.string() }),
  z.object({ status: z.literal("needs_auth") }),
  z.object({ status: z.literal("needs_client_registration"), error: z.string() }),
])
```

---

## Configuration

### Config Schema

```typescript
// Local MCP server (spawns a process)
export const McpLocal = z.object({
  type: z.literal("local"),
  command: z.string().array(),           // Command and args to run
  environment: z.record(z.string()).optional(),  // Env vars
  enabled: z.boolean().optional(),       // Enable/disable
  timeout: z.number().optional(),        // Request timeout (ms)
})

// Remote MCP server (HTTP/SSE)
export const McpRemote = z.object({
  type: z.literal("remote"),
  url: z.string(),                       // Server URL
  enabled: z.boolean().optional(),
  headers: z.record(z.string()).optional(), // Custom headers
  oauth: z.union([McpOAuth, z.literal(false)]).optional(),
  timeout: z.number().optional(),
})

// OAuth configuration
export const McpOAuth = z.object({
  clientId: z.string().optional(),       // Pre-registered client ID
  clientSecret: z.string().optional(),   // Client secret
  scope: z.string().optional(),          // OAuth scopes
})
```

### Example Configuration

```jsonc
// opencode.jsonc
{
  "mcp": {
    // Local server - filesystem access
    "filesystem": {
      "type": "local",
      "command": ["npx", "-y", "@modelcontextprotocol/server-filesystem", "/path/to/dir"],
      "environment": {
        "NODE_ENV": "development"
      }
    },

    // Remote server - no OAuth
    "api-server": {
      "type": "remote",
      "url": "https://api.example.com/mcp",
      "headers": {
        "X-API-Key": "your-key"
      }
    },

    // Remote server - with OAuth (dynamic registration)
    "oauth-server": {
      "type": "remote",
      "url": "https://secure.example.com/mcp"
      // OAuth enabled by default, will use dynamic client registration
    },

    // Remote server - with pre-registered OAuth client
    "enterprise-server": {
      "type": "remote",
      "url": "https://enterprise.example.com/mcp",
      "oauth": {
        "clientId": "your-client-id",
        "clientSecret": "your-client-secret",
        "scope": "read write"
      }
    },

    // Disabled server
    "disabled-server": {
      "type": "remote",
      "url": "https://example.com/mcp",
      "enabled": false
    }
  }
}
```

---

## Connection Flow

### Local Server Connection

```typescript
if (mcp.type === "local") {
  const [cmd, ...args] = mcp.command
  const transport = new StdioClientTransport({
    command: cmd,
    args,
    cwd: Instance.directory,
    env: { ...process.env, ...mcp.environment },
    stderr: "ignore",
  })

  const client = new Client({
    name: "opencode",
    version: Installation.VERSION,
  })

  await withTimeout(client.connect(transport), mcp.timeout ?? DEFAULT_TIMEOUT)
  registerNotificationHandlers(client, key)
}
```

### Remote Server Connection

```typescript
if (mcp.type === "remote") {
  // Try StreamableHTTP first, then SSE
  const transports = [
    { name: "StreamableHTTP", transport: new StreamableHTTPClientTransport(url, { authProvider }) },
    { name: "SSE", transport: new SSEClientTransport(url, { authProvider }) },
  ]

  for (const { name, transport } of transports) {
    try {
      const client = new Client({ name: "opencode", version })
      await withTimeout(client.connect(transport), timeout)
      return { mcpClient: client, status: { status: "connected" } }
    } catch (error) {
      if (error instanceof UnauthorizedError) {
        // OAuth needed
        pendingOAuthTransports.set(key, transport)
        return { status: { status: "needs_auth" } }
      }
    }
  }
}
```

### Notification Handlers

```typescript
function registerNotificationHandlers(client: MCPClient, serverName: string) {
  // Handle tool list changes from server
  client.setNotificationHandler(ToolListChangedNotificationSchema, async () => {
    Bus.publish(ToolsChanged, { server: serverName })
  })
}
```

---

## Tool Integration

### Tool Conversion

MCP tools are converted to AI SDK tools:

```typescript
async function convertMcpTool(mcpTool: MCPToolDef, client: MCPClient, timeout?: number): Promise<Tool> {
  const schema: JSONSchema7 = {
    ...mcpTool.inputSchema,
    type: "object",
    properties: mcpTool.inputSchema.properties ?? {},
    additionalProperties: false,
  }

  return dynamicTool({
    description: mcpTool.description ?? "",
    inputSchema: jsonSchema(schema),
    execute: async (args: unknown) => {
      return client.callTool(
        {
          name: mcpTool.name,
          arguments: args as Record<string, unknown>,
        },
        CallToolResultSchema,
        {
          resetTimeoutOnProgress: true,
          timeout,
        },
      )
    },
  })
}
```

### Tool Loading in Prompt Loop

```typescript
// packages/opencode/src/session/prompt.ts:728
for (const [key, item] of Object.entries(await MCP.tools())) {
  const execute = item.execute
  if (!execute) continue

  // Wrap execute to add plugin hooks
  item.execute = async (args, opts) => {
    const ctx = context(args, opts)

    await Plugin.trigger("tool.execute.before", { tool: key, ... }, { args })

    // Request permission (auto-approve MCP tools by default)
    await ctx.ask({
      permission: key,
      metadata: {},
      patterns: ["*"],
      always: ["*"],  // Auto-approve
    })

    const result = await execute(args, opts)

    await Plugin.trigger("tool.execute.after", { tool: key, ... }, result)

    return formatMcpResult(result)
  }

  tools[key] = item
}
```

### Tool Naming Convention

MCP tools are namespaced with the server name:

```
{sanitized_server_name}_{sanitized_tool_name}

Examples:
  filesystem_read_file
  database_query
  api_server_fetch
```

---

## Resources

### Resource Discovery

```typescript
export async function resources() {
  const result: Record<string, ResourceInfo & { client: string }> = {}
  const clientsSnapshot = await clients()

  for (const [clientName, client] of Object.entries(clientsSnapshot)) {
    if (s.status[clientName]?.status !== "connected") continue

    const resources = await client.listResources().catch(() => undefined)
    if (!resources) continue

    for (const resource of resources.resources) {
      const key = `${sanitize(clientName)}:${sanitize(resource.name)}`
      result[key] = { ...resource, client: clientName }
    }
  }

  return result
}
```

### Reading Resources

```typescript
// packages/opencode/src/session/prompt.ts:857
const resourceContent = await MCP.readResource(clientName, uri)

// Handle different content types
for (const content of resourceContent.contents) {
  if ("text" in content && content.text) {
    pieces.push({
      type: "text",
      synthetic: true,
      text: content.text,
    })
  } else if ("blob" in content && content.blob) {
    pieces.push({
      type: "text",
      synthetic: true,
      text: `[Binary content: ${content.mimeType}]`,
    })
  }
}
```

---

## Prompts

### Prompt Discovery

MCP prompts become available as commands in OpenCode:

```typescript
// packages/opencode/src/command/index.ts:94
for (const [name, prompt] of Object.entries(await MCP.prompts())) {
  result[name] = {
    name,
    mcp: true,
    description: prompt.description,
    get template() {
      return new Promise(async (resolve, reject) => {
        const template = await MCP.getPrompt(
          prompt.client,
          prompt.name,
          // Substitute arguments with $1, $2, etc.
          Object.fromEntries(prompt.arguments?.map((arg, i) => [arg.name, `$${i + 1}`]))
        )
        resolve(template?.messages.map(m => m.content.text).join("\n") || "")
      })
    },
    hints: prompt.arguments?.map((_, i) => `$${i + 1}`) ?? [],
  }
}
```

### Using MCP Prompts

```bash
# In OpenCode, MCP prompts appear as slash commands
/server_name:prompt_name arg1 arg2
```

---

## OAuth Authentication

### OAuth Provider Implementation

**File:** `packages/opencode/src/mcp/oauth-provider.ts`

```typescript
export class McpOAuthProvider implements OAuthClientProvider {
  constructor(
    private mcpName: string,
    private serverUrl: string,
    private config: McpOAuthConfig,
    private callbacks: McpOAuthCallbacks,
  ) {}

  get redirectUrl(): string {
    return `http://127.0.0.1:19876/mcp/oauth/callback`
  }

  get clientMetadata(): OAuthClientMetadata {
    return {
      redirect_uris: [this.redirectUrl],
      client_name: "OpenCode",
      client_uri: "https://opencode.ai",
      grant_types: ["authorization_code", "refresh_token"],
      response_types: ["code"],
      token_endpoint_auth_method: this.config.clientSecret ? "client_secret_post" : "none",
    }
  }

  // Client info from config or dynamic registration
  async clientInformation(): Promise<OAuthClientInformation | undefined>

  // Save dynamically registered client
  async saveClientInformation(info: OAuthClientInformationFull): Promise<void>

  // Token management
  async tokens(): Promise<OAuthTokens | undefined>
  async saveTokens(tokens: OAuthTokens): Promise<void>

  // PKCE support
  async saveCodeVerifier(codeVerifier: string): Promise<void>
  async codeVerifier(): Promise<string>

  // State parameter for CSRF protection
  async saveState(state: string): Promise<void>
  async state(): Promise<string>

  // Trigger browser redirect
  async redirectToAuthorization(authorizationUrl: URL): Promise<void>
}
```

### OAuth Callback Server

**File:** `packages/opencode/src/mcp/oauth-callback.ts`

```typescript
export namespace McpOAuthCallback {
  const CALLBACK_PORT = 19876
  const CALLBACK_PATH = "/mcp/oauth/callback"
  const CALLBACK_TIMEOUT_MS = 5 * 60 * 1000  // 5 minutes

  export async function ensureRunning(): Promise<void> {
    if (server) return

    server = Bun.serve({
      port: CALLBACK_PORT,
      fetch(req) {
        const url = new URL(req.url)
        if (url.pathname !== CALLBACK_PATH) return notFound()

        const code = url.searchParams.get("code")
        const state = url.searchParams.get("state")
        const error = url.searchParams.get("error")

        // Validate state parameter (CSRF protection)
        if (!state || !pendingAuths.has(state)) {
          return htmlError("Invalid or expired state parameter")
        }

        if (error) {
          pendingAuths.get(state)!.reject(new Error(error))
          return htmlError(error)
        }

        if (code) {
          pendingAuths.get(state)!.resolve(code)
          return htmlSuccess()
        }
      },
    })
  }

  export function waitForCallback(oauthState: string): Promise<string> {
    return new Promise((resolve, reject) => {
      const timeout = setTimeout(() => {
        reject(new Error("OAuth callback timeout"))
      }, CALLBACK_TIMEOUT_MS)

      pendingAuths.set(oauthState, { resolve, reject, timeout })
    })
  }
}
```

### Authentication Flow

```
┌─────────────┐     ┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│   OpenCode  │     │  Callback   │     │   Browser   │     │ Auth Server │
│             │     │   Server    │     │             │     │             │
└──────┬──────┘     └──────┬──────┘     └──────┬──────┘     └──────┬──────┘
       │                   │                   │                   │
       │ 1. startAuth()    │                   │                   │
       │──────────────────▶│                   │                   │
       │                   │                   │                   │
       │ 2. ensureRunning()│                   │                   │
       │   (port 19876)    │                   │                   │
       │                   │                   │                   │
       │ 3. Generate PKCE + state              │                   │
       │                   │                   │                   │
       │ 4. Open browser ─────────────────────▶│                   │
       │                   │                   │                   │
       │                   │                   │ 5. User authorizes│
       │                   │                   │──────────────────▶│
       │                   │                   │                   │
       │                   │                   │ 6. Redirect with  │
       │                   │ 7. Callback       │◀──code + state────│
       │                   │◀──────────────────│                   │
       │                   │                   │                   │
       │ 8. waitForCallback() returns code     │                   │
       │◀──────────────────│                   │                   │
       │                   │                   │                   │
       │ 9. Exchange code for tokens ─────────────────────────────▶│
       │                   │                   │                   │
       │ 10. Save tokens   │                   │                   │
       │                   │                   │                   │
```

### Token Storage

**File:** `packages/opencode/src/mcp/auth.ts`

```typescript
export namespace McpAuth {
  const filepath = path.join(Global.Path.data, "mcp-auth.json")

  export const Entry = z.object({
    tokens: Tokens.optional(),           // OAuth tokens
    clientInfo: ClientInfo.optional(),   // Dynamic registration info
    codeVerifier: z.string().optional(), // PKCE verifier
    oauthState: z.string().optional(),   // CSRF state
    serverUrl: z.string().optional(),    // URL tokens are valid for
  })

  // URL validation - invalidate tokens if URL changes
  export async function getForUrl(mcpName: string, serverUrl: string) {
    const entry = await get(mcpName)
    if (!entry?.serverUrl || entry.serverUrl !== serverUrl) {
      return undefined  // Invalid for this URL
    }
    return entry
  }

  export async function updateTokens(mcpName: string, tokens: Tokens, serverUrl?: string)
  export async function updateClientInfo(mcpName: string, clientInfo: ClientInfo, serverUrl?: string)
  export async function isTokenExpired(mcpName: string): Promise<boolean | null>
}
```

---

## CLI Commands

```bash
# List configured MCP servers
opencode mcp list

# Add a new MCP server (interactive)
opencode mcp add

# Authenticate with OAuth server
opencode mcp auth [server-name]

# List OAuth status
opencode mcp auth list

# Remove OAuth credentials
opencode mcp logout [server-name]

# Debug OAuth connection
opencode mcp debug <server-name>
```

### CLI Command Implementation

```typescript
// packages/opencode/src/cli/cmd/mcp.ts
export const McpCommand = cmd({
  command: "mcp",
  describe: "manage MCP servers",
  builder: (yargs) =>
    yargs
      .command(McpAddCommand)
      .command(McpListCommand)
      .command(McpAuthCommand)
      .command(McpLogoutCommand)
      .command(McpDebugCommand)
      .demandCommand(),
})
```

---

## Integration Points

### Where MCP is Used

| Location | Purpose |
|----------|---------|
| `prompt.ts:728` | Load MCP tools into execution loop |
| `command/index.ts:94` | Register MCP prompts as commands |
| `server/routes/experimental.ts:154` | Expose resources via API |
| `prompt.ts:857` | Read MCP resources into messages |

### Event System

```typescript
// Tool list changed notification
export const ToolsChanged = BusEvent.define(
  "mcp.tools.changed",
  z.object({ server: z.string() })
)

// Browser open failed (for headless environments)
export const BrowserOpenFailed = BusEvent.define(
  "mcp.browser.open.failed",
  z.object({ mcpName: z.string(), url: z.string() })
)
```

---

## Summary

| Component | File | Purpose |
|-----------|------|---------|
| MCP Client Manager | `mcp/index.ts` | Connection lifecycle, tool/resource access |
| Token Storage | `mcp/auth.ts` | Persist OAuth tokens and client info |
| OAuth Provider | `mcp/oauth-provider.ts` | Implement MCP SDK's OAuthClientProvider |
| Callback Server | `mcp/oauth-callback.ts` | Local HTTP server for OAuth redirects |
| CLI Commands | `cli/cmd/mcp.ts` | User-facing MCP management |
| Config Schema | `config/config.ts` | McpLocal, McpRemote, McpOAuth types |

### Key Design Decisions

1. **Transport Fallback** - Try StreamableHTTP first, fall back to SSE
2. **OAuth by Default** - Remote servers have OAuth enabled unless explicitly disabled
3. **Dynamic Registration** - Supports RFC 7591 when no clientId is provided
4. **URL Binding** - Tokens are invalidated if server URL changes
5. **CSRF Protection** - State parameter validated on all callbacks
6. **Auto-Approve Tools** - MCP tools default to `always: ["*"]` permission

---

*Written by Claude (Opus 4.5) | 2026-01-22 16:15 PST*
