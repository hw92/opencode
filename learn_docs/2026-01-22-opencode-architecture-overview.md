# OpenCode Architecture Overview

A bird's eye view of the entire OpenCode system.

---

## The Big Picture

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                              OPENCODE ARCHITECTURE                                   │
├─────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                      │
│  ┌─────────────────────────────────────────────────────────────────────────────┐    │
│  │                            USER INTERFACES                                   │    │
│  │                                                                              │    │
│  │    ┌──────────┐     ┌──────────┐     ┌──────────┐     ┌──────────┐         │    │
│  │    │   CLI    │     │   TUI    │     │  Server  │     │  Tauri   │         │    │
│  │    │ (cmds)   │     │ (Ink/React)    │  (Hono)  │     │  (App)   │         │    │
│  │    └────┬─────┘     └────┬─────┘     └────┬─────┘     └────┬─────┘         │    │
│  │         └────────────────┴────────────────┴────────────────┘                │    │
│  └─────────────────────────────────────┬───────────────────────────────────────┘    │
│                                        │                                            │
│  ┌─────────────────────────────────────▼───────────────────────────────────────┐    │
│  │                           CORE ENGINE                                        │    │
│  │                                                                              │    │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐        │    │
│  │  │   Session   │  │    Agent    │  │    Tool     │  │ Permission  │        │    │
│  │  │   prompt()  │  │   configs   │  │  registry   │  │   rules     │        │    │
│  │  │   loop()    │  │   prompts   │  │   execute   │  │   evaluate  │        │    │
│  │  └──────┬──────┘  └──────┬──────┘  └──────┬──────┘  └──────┬──────┘        │    │
│  │         │                │                │                │                │    │
│  │         └────────────────┴────────────────┴────────────────┘                │    │
│  │                                   │                                          │    │
│  │  ┌────────────────────────────────▼────────────────────────────────────┐    │    │
│  │  │                         PROCESSOR                                    │    │    │
│  │  │  • Stream LLM responses    • Handle tool calls                      │    │    │
│  │  │  • Persist message parts   • Detect doom loops                      │    │    │
│  │  │  • Retry on errors         • Track snapshots                        │    │    │
│  │  └────────────────────────────────┬────────────────────────────────────┘    │    │
│  │                                   │                                          │    │
│  └───────────────────────────────────┼──────────────────────────────────────────┘    │
│                                      │                                              │
│  ┌───────────────────────────────────▼──────────────────────────────────────────┐   │
│  │                          INTEGRATION LAYER                                    │   │
│  │                                                                               │   │
│  │  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐           │   │
│  │  │ Provider │ │   MCP    │ │   LSP    │ │  Plugin  │ │ Snapshot │           │   │
│  │  │ 20+ LLMs │ │ servers  │ │ 45+ lang │ │ system   │ │ git-based│           │   │
│  │  └──────────┘ └──────────┘ └──────────┘ └──────────┘ └──────────┘           │   │
│  │                                                                               │   │
│  └───────────────────────────────────┬───────────────────────────────────────────┘   │
│                                      │                                              │
│  ┌───────────────────────────────────▼──────────────────────────────────────────┐   │
│  │                          FOUNDATION LAYER                                     │   │
│  │                                                                               │   │
│  │  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐           │   │
│  │  │ Storage  │ │   Bus    │ │  Config  │ │   Auth   │ │  Global  │           │   │
│  │  │ JSON files│ │ pub/sub │ │ 6-layer  │ │ OAuth/API│ │ XDG paths│           │   │
│  │  └──────────┘ └──────────┘ └──────────┘ └──────────┘ └──────────┘           │   │
│  │                                                                               │   │
│  └───────────────────────────────────────────────────────────────────────────────┘   │
│                                                                                      │
└──────────────────────────────────────────────────────────────────────────────────────┘
```

---

## Layer Summary

| Layer | Components | Purpose |
|-------|------------|---------|
| **UI** | CLI, TUI, Server, Tauri | User interaction |
| **Core Engine** | Session, Agent, Tool, Permission | Execution orchestration |
| **Integration** | Provider, MCP, LSP, Plugin, Snapshot | External systems |
| **Foundation** | Storage, Bus, Config, Auth, Global | Infrastructure |

---

## Data Flow: User Message → AI Response

```
User types message
       │
       ▼
┌──────────────────┐
│  1. CLI/TUI      │  Parse input, handle @mentions, file attachments
└────────┬─────────┘
         │
         ▼
┌──────────────────┐
│  2. Session      │  Create user message, enter execution loop
│     prompt()     │
└────────┬─────────┘
         │
         ▼
┌──────────────────┐
│  3. Agent        │  Load agent config, permissions, system prompt
└────────┬─────────┘
         │
         ▼
┌──────────────────┐
│  4. Tool         │  Resolve available tools based on permissions
│     Registry     │
└────────┬─────────┘
         │
         ▼
┌──────────────────┐
│  5. Processor    │  Stream LLM, handle events, persist parts
└────────┬─────────┘
         │
         ▼
┌──────────────────┐
│  6. LLM          │  Transform messages, call AI SDK, stream response
└────────┬─────────┘
         │
         ▼
┌──────────────────┐
│  7. Provider     │  Provider-specific handling (Anthropic, OpenAI, etc.)
└────────┬─────────┘
         │
         ▼
   AI Response (text, tool calls, reasoning)
         │
         ▼
┌──────────────────┐
│  8. Tool         │  Execute tool calls (read, edit, bash, etc.)
│     Execute      │
└────────┬─────────┘
         │
         ▼
┌──────────────────┐
│  9. Permission   │  Check rules, prompt user if needed
└────────┬─────────┘
         │
         ▼
┌──────────────────┐
│ 10. Loop         │  Continue until finish_reason = "end_turn"
└────────┬─────────┘
         │
         ▼
   Display response to user
```

---

## Package Structure

```
packages/opencode/src/
├── cli/              # Command-line interface
│   ├── cmd/          # CLI commands (auth, session, config, etc.)
│   └── tui/          # Terminal UI (Ink/React components)
│
├── server/           # HTTP server (Hono)
│   └── routes/       # API endpoints
│
├── session/          # Core execution engine
│   ├── prompt.ts     # Main loop orchestrator
│   ├── processor.ts  # Stream processing
│   ├── llm.ts        # LLM abstraction
│   └── message-v2.ts # Message/part handling
│
├── agent/            # Agent definitions
├── tool/             # Built-in tools (30+)
├── permission/       # Permission system
│
├── provider/         # LLM providers (20+)
├── mcp/              # Model Context Protocol
├── lsp/              # Language Server Protocol
├── plugin/           # Plugin system
│
├── storage/          # File-based persistence
├── bus/              # Event pub/sub
├── config/           # Configuration loading
├── auth/             # Authentication
├── global/           # Global paths (XDG)
│
├── file/             # File operations
│   ├── snapshot.ts   # Git-based snapshots
│   └── ripgrep.ts    # Search integration
│
└── share/            # Session sharing
```

---

## Key Design Patterns

| Pattern | Where Used | Purpose |
|---------|------------|---------|
| **Namespace** | Every module | Encapsulation without classes |
| **Event Bus** | Bus.publish/subscribe | Decoupled communication |
| **Lazy State** | Instance.state() | Singleton with cleanup |
| **While Loop** | prompt.ts | Agentic execution |
| **Stream Processing** | processor.ts | Handle LLM events |
| **Last-Match-Wins** | permission/next.ts | Rule evaluation |
| **Shadow Repository** | snapshot.ts | File state tracking |
| **Adapter** | provider/transform.ts | Provider normalization |

---

## The Execution Loop (Heart of the System)

```typescript
// Simplified from session/prompt.ts
async function loop(sessionID: string) {
  while (true) {
    // Exit if assistant finished
    if (lastAssistant?.finish && lastUser.id < lastAssistant.id) break

    // Handle subtasks (spawn child session)
    if (task?.type === "subtask") { await handleSubtask(); continue }

    // Handle compaction (context overflow)
    if (task?.type === "compaction") { await compact(); continue }

    // Normal processing
    const result = await processor.process({
      agent,
      tools,
      messages,
      onEvent: (event) => { /* persist parts, update UI */ }
    })

    if (result === "stop") break
    if (result === "compact") { /* trigger compaction */ }
  }
}
```

---

## Tool Execution Flow

```
Tool Call from LLM
       │
       ▼
┌──────────────────┐
│ 1. Validate args │  Zod schema validation
└────────┬─────────┘
         │
         ▼
┌──────────────────┐
│ 2. Check perms   │  ctx.ask() → Permission.evaluate()
└────────┬─────────┘
         │
    ┌────┴────┐
    ▼         ▼
 ALLOW      ASK/DENY
    │         │
    │    ┌────┴────┐
    │    ▼         ▼
    │  User     Throw
    │  Prompt   Error
    │    │
    │    ▼
    │  once/always/reject
    │    │
    └────┴────┐
              ▼
┌──────────────────┐
│ 3. Snapshot      │  Capture file state before changes
└────────┬─────────┘
         │
         ▼
┌──────────────────┐
│ 4. Execute       │  Run tool logic (read, edit, bash, etc.)
└────────┬─────────┘
         │
         ▼
┌──────────────────┐
│ 5. Truncate      │  Limit output (2000 lines / 50KB)
└────────┬─────────┘
         │
         ▼
┌──────────────────┐
│ 6. LSP Check     │  Get diagnostics if file was edited
└────────┬─────────┘
         │
         ▼
   Return result to LLM
```

---

## Storage Model

```
~/.local/share/opencode/storage/
│
├── project/
│   └── {projectID}.json        # Project metadata
│
├── session/
│   └── {projectID}/
│       └── {sessionID}.json    # Session info
│
├── message/
│   └── {sessionID}/
│       └── {messageID}.json    # Message metadata
│
├── part/
│   └── {messageID}/
│       └── {partID}.json       # Text, tool calls, reasoning
│
├── session_diff/
│   └── {sessionID}.json        # File changes summary
│
├── permission/
│   └── {projectID}.json        # Approved permissions
│
└── todo/
    └── {sessionID}.json        # Todo list
```

---

## Event System

```
┌─────────────────────────────────────────────────────────────────┐
│                        EVENT BUS                                 │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  Publishers                          Subscribers                 │
│  ──────────                          ───────────                 │
│  Session.Event.Created    ────────►  TUI (update list)          │
│  Session.Event.Updated    ────────►  Share (sync to cloud)      │
│  MessageV2.Event.Updated  ────────►  Server SSE (push to UI)    │
│  MessageV2.Event.PartUpdated ─────►  Storage (persist)          │
│  LSPClient.Event.Diagnostics ─────►  Edit tool (show errors)    │
│  Permission.Event.Asked   ────────►  TUI (show prompt)          │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## Configuration Hierarchy

```
Priority (Low → High)
──────────────────────

1. Remote          .well-known/opencode (org defaults)
       ↓
2. Global          ~/.config/opencode/opencode.json
       ↓
3. Custom          OPENCODE_CONFIG env var
       ↓
4. Project         ./opencode.json
       ↓
5. .opencode/      agents/, commands/, plugins/
       ↓
6. Inline          OPENCODE_CONFIG_CONTENT env var
```

---

## Integration Points

### Provider System
- **Input**: Model selection, messages, tools
- **Output**: Streaming text, tool calls, usage stats
- **Adapts**: 20+ providers via transformation layer

### MCP System
- **Input**: Server configs, tool requests
- **Output**: Additional tools, resources, prompts
- **Protocol**: JSON-RPC over stdio/HTTP

### LSP System
- **Input**: File paths, positions
- **Output**: Diagnostics, definitions, references
- **Protocol**: JSON-RPC over stdio

### Plugin System
- **Input**: npm packages, file:// URLs
- **Output**: Tools, providers, auth loaders
- **Pattern**: Dynamic import with capability registration

### Snapshot System
- **Input**: File operations from tools
- **Output**: Revert/restore capability
- **Mechanism**: Shadow git repository with tree objects

---

## Key Numbers

| Metric | Count |
|--------|-------|
| LLM Providers | 20+ |
| Built-in Tools | 30+ |
| LSP Servers | 45+ |
| Config Layers | 6 |
| Permission Types | 20+ |
| Source Files | ~150 |
| Total Lines | ~25,000 |

---

## Mental Model

Think of OpenCode as:

1. **A Loop** - Keep running until the AI says "done"
2. **With Tools** - AI can read, write, search, execute
3. **Gated by Permissions** - User controls what's allowed
4. **Backed by Storage** - Everything persisted as JSON
5. **Connected via Events** - Components communicate through pub/sub
6. **Extensible** - Plugins, MCP servers, custom agents

---

## For Your Financial Agents

Apply these patterns:

| OpenCode | Your System |
|----------|-------------|
| Agent configs | Analyst, Researcher, Trader roles |
| Tool registry | Market data, SEC filings, calculations |
| Permission system | Prevent unauthorized trades |
| Execution loop | Keep analyzing until complete |
| Event bus | Real-time updates to dashboard |
| Storage | Persist analysis results |
| Snapshot | Audit trail for decisions |

---

*Written by Claude (Opus 4.5) | 2026-01-22 16:15 PST*
