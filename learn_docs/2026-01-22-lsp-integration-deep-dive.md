# OpenCode LSP Integration Deep Dive

[TOC]

A comprehensive study of OpenCode's Language Server Protocol integration for code intelligence.

---

## Overview

OpenCode integrates with **45+ language servers** via the Language Server Protocol (LSP) to provide real-time diagnostics, code navigation, and semantic analysis. The system manages server lifecycles, pools connections, and integrates diagnostics into file editing tools.

---

## Architecture

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         LSP INTEGRATION                                      │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  ┌──────────────────────────────────────────────────────────────────────┐   │
│  │                     Tools (Read/Edit/Write)                           │   │
│  │                            │                                          │   │
│  │                   LSP.touchFile(path)                                 │   │
│  │                            │                                          │   │
│  └────────────────────────────┼─────────────────────────────────────────┘   │
│                               ▼                                              │
│  ┌──────────────────────────────────────────────────────────────────────┐   │
│  │                      LSP Namespace                                    │   │
│  │  • getClients(file) - Get relevant LSP clients                        │   │
│  │  • touchFile(file) - Notify LSP of file changes                       │   │
│  │  • diagnostics() - Aggregate all diagnostics                          │   │
│  │  • hover/definition/references - Code navigation                      │   │
│  └────────────────────────────┼─────────────────────────────────────────┘   │
│                               │                                              │
│         ┌─────────────────────┼─────────────────────┐                       │
│         ▼                     ▼                     ▼                       │
│  ┌─────────────┐      ┌─────────────┐      ┌─────────────┐                 │
│  │ LSPClient   │      │ LSPClient   │      │ LSPClient   │                 │
│  │ TypeScript  │      │ Python      │      │ Go          │                 │
│  └──────┬──────┘      └──────┬──────┘      └──────┬──────┘                 │
│         │                    │                    │                         │
│         │ JSON-RPC (stdio)   │                    │                         │
│         ▼                    ▼                    ▼                         │
│  ┌─────────────┐      ┌─────────────┐      ┌─────────────┐                 │
│  │ ts-server   │      │ pyright     │      │ gopls       │                 │
│  └─────────────┘      └─────────────┘      └─────────────┘                 │
│                                                                              │
└──────────────────────────────────────────────────────────────────────────────┘
```

---

## Supported Languages (45+)

| Language | Server | Auto-Download | Detection |
|----------|--------|---------------|-----------|
| TypeScript/JS | typescript-lsp | npm | package.json |
| Python | pyright | pip | pyproject.toml |
| Go | gopls | go install | go.mod |
| Rust | rust-analyzer | GitHub | Cargo.toml |
| Java | JDTLS | Download | pom.xml |
| C/C++ | clangd | GitHub | CMakeLists.txt |
| C# | csharp-ls | dotnet | .csproj |
| Vue | @vue/language-server | npm | package.json |
| Svelte | svelte-language-server | npm | package.json |
| Ruby | ruby-lsp | gem | Gemfile |
| Kotlin | kotlin-lsp | GitHub | build.gradle.kts |
| Elixir | elixir-ls | GitHub | mix.exs |
| Lua | lua-language-server | GitHub | .luarc.json |
| PHP | intelephense | npm | composer.json |
| Terraform | terraform-ls | GitHub | .tf |
| Zig | zls | GitHub | build.zig |
| And 30+ more... | | | |

---

## Core Components

### LSP Index (`src/lsp/index.ts`)

Central coordinator for all LSP operations:

```typescript
export namespace LSP {
  // Initialize LSP system
  export async function init(): Promise<State>

  // Get clients that handle a file
  export async function getClients(file: string): Promise<LSPClient.Info[]>

  // Check if LSP is available for file
  export async function hasClients(file: string): boolean

  // Notify LSP of file changes
  export async function touchFile(file: string, wait?: boolean)

  // Get all diagnostics
  export async function diagnostics(): Promise<Record<string, Diagnostic[]>>

  // Code navigation
  export async function hover(input): Promise<any>
  export async function definition(input): Promise<any>
  export async function references(input): Promise<any>
  export async function workspaceSymbol(query: string): Promise<Symbol[]>
}
```

### LSP Client (`src/lsp/client.ts`)

Manages individual server connections:

```typescript
export namespace LSPClient {
  interface Info {
    id: string
    serverID: string
    root: string
    diagnostics: Map<string, Diagnostic[]>
    notify: {
      open(path: string): void
    }
    request: {
      hover(params): Promise<any>
      definition(params): Promise<any>
      // ... other LSP requests
    }
    waitForDiagnostics(path: string): Promise<void>
    shutdown(): Promise<void>
  }
}
```

### LSP Server Registry (`src/lsp/server.ts`)

Defines 45+ built-in servers:

```typescript
export namespace LSPServer {
  interface Info {
    id: string
    extensions: string[]
    global?: boolean
    root(file: string): Promise<string | undefined>
    spawn(root: string): Promise<Handle | undefined>
  }

  // Built-in servers
  export const TypeScript: Info
  export const Python: Info
  export const Go: Info
  // ... 40+ more
}
```

---

## Client Lifecycle

### Initialization Sequence

```
1. Tool calls LSP.touchFile(filePath)
         │
         ▼
2. LSP.getClients(filePath) invoked
         │
         ▼
3. For each configured server:
   ├─ Check file extension match
   ├─ Check broken server cache
   ├─ Look up root directory
   ├─ Check if client exists
   └─ Check if spawn in-flight
         │
         ▼
4. If needed: spawn new client
   ├─ Spawn server process
   ├─ Setup stdio connection
   ├─ Send initialize request
   ├─ Wait for response
   ├─ Send initialized notification
   └─ Register diagnostic listener
         │
         ▼
5. Client added to pool
```

### Deduplication Strategy

```typescript
// Clients keyed by root + serverId
const key = root + server.id

// Check existing
const existing = clients.find(c =>
  c.root === root && c.serverID === server.id
)
if (existing) return existing

// Check in-flight
const inflight = spawning.get(key)
if (inflight) return await inflight

// Spawn new
const task = schedule(server, root, key)
spawning.set(key, task)
```

---

## Diagnostic Integration

### Flow Diagram

```
LSP Server
    │
    ▼ textDocument/publishDiagnostics
LSPClient
    │
    ▼ Update diagnostics Map
    │
    ▼ Bus.publish(Event.Diagnostics)
    │
    ▼ (Debounce 150ms)
    │
Edit/Write Tools
    │
    ▼ LSP.diagnostics()
    │
    ▼ Filter severity === 1 (errors)
    │
    ▼ Format with LSP.Diagnostic.pretty()
    │
    ▼ Append to tool output
```

### Diagnostic Structure

```typescript
interface Diagnostic {
  range: {
    start: { line: number; character: number }
    end: { line: number; character: number }
  }
  severity?: 1 | 2 | 3 | 4  // Error | Warning | Info | Hint
  message: string
  source?: string
  code?: string | number
}
```

### Tool Integration Example

```typescript
// In edit.ts
await file.write(contentNew)
await LSP.touchFile(filePath, true)  // Wait for diagnostics

const diagnostics = await LSP.diagnostics()
const errors = diagnostics[filePath]?.filter(d => d.severity === 1) ?? []

if (errors.length > 0) {
  output += `\n\nLSP errors detected:\n<diagnostics file="${filePath}">\n`
  output += errors.slice(0, 20).map(LSP.Diagnostic.pretty).join("\n")
  output += `\n</diagnostics>`
}
```

---

## Configuration

### Config Schema

```json
{
  "lsp": {
    // Disable a built-in server
    "typescript": {
      "disabled": true
    },

    // Override built-in server
    "python": {
      "command": ["custom-python-lsp"],
      "env": { "PYTHONPATH": "/custom" }
    },

    // Add custom server
    "my-lsp": {
      "command": ["my-lsp-server", "--stdio"],
      "extensions": [".custom"],
      "initialization": {}
    }
  }
}
```

### Disable All LSP

```json
{
  "lsp": false
}
```

### Feature Flags

```bash
# Experimental Python LSP (ty) instead of pyright
OPENCODE_EXPERIMENTAL_LSP_TY=1

# Disable automatic binary downloads
OPENCODE_DISABLE_LSP_DOWNLOAD=1
```

---

## LSP Tool

The LSP tool exposes code intelligence to the AI:

```typescript
operations:
  - goToDefinition      // Navigate to symbol definition
  - findReferences      // Find all references
  - hover               // Get hover information
  - documentSymbol      // List symbols in file
  - workspaceSymbol     // Search symbols across project
  - goToImplementation  // Find implementations
  - incomingCalls       // Call hierarchy (incoming)
  - outgoingCalls       // Call hierarchy (outgoing)

parameters:
  - operation: string   // One of the above
  - filePath: string    // Absolute or relative path
  - line: number        // 1-based line number
  - character: number   // 1-based character offset
```

---

## Advanced Features

### Call Hierarchy

```typescript
// Prepare call hierarchy
const items = await LSP.prepareCallHierarchy({ file, line, character })

// Get incoming calls
const incoming = await LSP.incomingCalls({ item: items[0] })

// Get outgoing calls
const outgoing = await LSP.outgoingCalls({ item: items[0] })
```

### Workspace Symbols

```typescript
// Search for symbols across project
const symbols = await LSP.workspaceSymbol("myFunction")
// Returns: [{ name, kind, location, containerName }]
```

### Document Symbols

```typescript
// Get symbol outline for a file
const symbols = await LSP.documentSymbol(fileUri)
// Returns hierarchical DocumentSymbol[]
```

---

## Design Patterns

| Pattern | Purpose |
|---------|---------|
| **Client Pooling** | One client per server per root |
| **Lazy Initialization** | Spawn on first file access |
| **Event-Driven Diagnostics** | Push from servers via bus |
| **Debounced Waiting** | 150ms debounce for batch diagnostics |
| **Capability Negotiation** | Declare supported features |
| **Automatic Downloads** | Fetch binaries from GitHub releases |

---

## Troubleshooting

### No LSP for File Type

```bash
# Check LSP status
opencode lsp status

# Verify server is installed
which gopls  # for Go
```

### Diagnostics Not Appearing

1. Check if LSP is enabled: `opencode lsp status`
2. Manually trigger: `opencode lsp diagnostics <file>`
3. Check logs: `OPENCODE_LOG_LEVEL=debug`

### Slow Diagnostics

The system has a 3-second timeout. For slow servers:
- Consider disabling waiting: `LSP.touchFile(file, false)`
- Check server performance

---

## Key Files

| File | Purpose |
|------|---------|
| `src/lsp/index.ts` | Central LSP coordinator |
| `src/lsp/client.ts` | Server connection management |
| `src/lsp/server.ts` | Built-in server definitions |
| `src/lsp/language.ts` | Extension → language mapping |
| `src/tool/lsp.ts` | LSP tool for AI |

---

*Written by Claude (Opus 4.5) | 2026-01-22 16:00 PST*
