# OpenCode Architecture Study Notes



Focus

```
Key Files
  ┌────────────────────┬───────────────────────────────────────┐
  │        File        │                Purpose                │
  ├────────────────────┼───────────────────────────────────────┤
  │ agent/agent.ts     │ Agent configuration, loading, merging │
  ├────────────────────┼───────────────────────────────────────┤
  │ session/prompt.ts  │ Main execution loop, message handling │
  ├────────────────────┼───────────────────────────────────────┤
  │ session/llm.ts     │ LLM streaming, prompt assembly        │
  ├────────────────────┼───────────────────────────────────────┤
  │ tool/registry.ts   │ Tool discovery and filtering          │
  ├────────────────────┼───────────────────────────────────────┤
  │ permission/next.ts │ Rule-based permission evaluation      │
  ├────────────────────┼───────────────────────────────────────┤
  │ prompt/            │ System prompts per provider           │
  └────────────────────┴───────────────────────────────────────┘
  
```















[TOC]



A comprehensive study of the OpenCode codebase—an open-source AI coding agent similar to Claude Code.

## Project Overview

**Repository:** `github.com/anomalyco/opencode`
**License:** MIT
**Version:** 1.1.25
**Core Package Size:** ~38,861 lines of TypeScript

---

## Tech Stack

| Layer | Technology |
|-------|------------|
| Runtime | Bun 1.3.5+ |
| Language | TypeScript 5.8.2 |
| CLI Framework | Yargs |
| Frontend | SolidJS + SolidStart |
| Desktop | Tauri 2.x (Rust) |
| Web Server | Hono |
| Database | PlanetScale (MySQL) + Drizzle ORM |
| AI SDK | Vercel AI SDK 5.0.119 |
| Build | Turbo (monorepo), Vite |
| Infrastructure | SST + Cloudflare |
| UI Components | Kobalte (headless) + Tailwind CSS 4.x |
| Marketing Site | Astro 5.7 |

---

## Project Structure

```
opencode/
├── packages/                 # Monorepo workspace (18 packages)
│   ├── opencode/            # Core CLI + server (~38k lines)
│   │   ├── src/
│   │   │   ├── agent/       # Multi-agent framework
│   │   │   ├── provider/    # LLM provider abstraction
│   │   │   ├── session/     # Chat sessions & message history
│   │   │   ├── tool/        # 45+ built-in tools
│   │   │   ├── command/     # Custom command framework
│   │   │   ├── server/      # HTTP server + routes
│   │   │   ├── mcp/         # Model Context Protocol
│   │   │   ├── lsp/         # Language Server Protocol
│   │   │   ├── permission/  # Access control layer
│   │   │   ├── config/      # Configuration management
│   │   │   ├── storage/     # File-based persistence
│   │   │   ├── prompt/      # System prompts per provider
│   │   │   └── ...
│   │   └── bin/opencode     # CLI entry point
│   ├── app/                 # Web UI (SolidJS)
│   ├── desktop/             # Desktop app (Tauri)
│   ├── console/             # Console web interface
│   │   ├── core/            # Backend (Drizzle, DB)
│   │   ├── app/             # Frontend (SolidStart)
│   │   └── mail/            # Email services
│   ├── ui/                  # Shared UI component library
│   ├── sdk/js/              # JavaScript SDK
│   ├── web/                 # Marketing/docs site (Astro)
│   ├── plugin/              # Plugin system
│   └── util/                # Shared utilities
├── infra/                   # Infrastructure as Code (SST)
├── sdks/                    # VS Code SDK
├── specs/                   # API specifications
└── .opencode/               # Project configuration
```

---

## Key Design Patterns

### 1. Namespace-Based Module Organization

The dominant pattern—every module exports a TypeScript namespace:

```typescript
// Example: provider/provider.ts
export namespace Provider {
  export const Info = z.object({
    id: z.string(),
    name: z.string(),
    // ...
  })

  export async function list() { /* ... */ }
  export async function getModel(providerID, modelID) { /* ... */ }
}
```

**Benefits:**
- Clear module boundaries without class hierarchies
- Related functions, types, and constants grouped together
- Easy to import: `import { Provider } from "./provider"`

### 2. Instance-Based State Management

Per-project state with lifecycle hooks:

```typescript
const state = Instance.state(
  () => ({ /* initial state */ }),
  async (entry) => { /* cleanup callback */ }
)

// Usage
const data = await state()
```

### 3. Event-Driven Architecture (Bus Pattern)

Typed pub/sub communication with Zod schemas:

```typescript
// Define event
export const SessionCreated = BusEvent.define("session.created",
  z.object({ sessionID: z.string() })
)

// Publish
Bus.publish(SessionCreated, { sessionID: "..." })

// Subscribe
Bus.subscribe(SessionCreated, (event) => { /* handle */ })
```

### 4. Zod Schemas Everywhere

Runtime validation for all data structures:

```typescript
export const Info = z.object({
  name: z.string(),
  mode: z.enum(["subagent", "primary", "all"]),
  permission: PermissionNext.Ruleset,
  // ...
})

export type Info = z.infer<typeof Info>
```

### 5. Permission-First Security

Rule-based access control evaluated at tool execution:

```typescript
export type Rule = {
  permission: string    // Tool name or special permission
  pattern: string       // Wildcard pattern for arguments
  action: "allow" | "deny" | "ask"
}
```

### 6. Provider Abstraction

Unified interface for 20+ LLM providers:

```typescript
// Providers supported via @ai-sdk/* packages:
// Anthropic, OpenAI, Google, Azure, Bedrock, Mistral,
// Groq, Cerebras, Cohere, Perplexity, XAI, Together, etc.

const stream = await LLM.stream({
  agent: agentInfo,
  model: Provider.Model,
  system: [...],
  tools: {...},
})
```

---

## Agent System Architecture

### Agent Types

| Agent | Mode | Purpose |
|-------|------|---------|
| `build` | primary | Execution-focused, can modify files |
| `plan` | primary | Read-only planning, writes to `.opencode/plans/` only |
| `general` | subagent | Multi-tasking, called via Task tool |
| `explore` | subagent | Codebase exploration, read-only |
| `compaction` | hidden | Compresses session history |
| `title` | hidden | Generates session titles |

### Agent Configuration Schema

```typescript
// From agent/agent.ts
export const Info = z.object({
  name: z.string(),
  description: z.string().optional(),
  mode: z.enum(["subagent", "primary", "all"]),
  permission: PermissionNext.Ruleset,
  model: z.object({ modelID, providerID }).optional(),
  prompt: z.string().optional(),
  temperature: z.number().optional(),
  topP: z.number().optional(),
  steps: z.number().int().positive().optional(),
  options: z.record(z.string(), z.any()),
})
```

### Execution Flow

```
User Message
     │
     ▼
SessionPrompt.prompt()
├── Load agent config
├── Assemble system prompts
└── Filter tools by permissions
     │
     ▼
SessionPrompt.loop() ─────────────────┐
│                                     │
├── 1. Get messages from session      │
├── 2. Stream LLM response            │
├── 3. Process events                 │
│   ├── tool-call → execute tool      │
│   ├── text-delta → accumulate       │
│   └── finish-step → check exit      │
├── 4. Permission checks              │
└── 5. Continue or exit ──────────────┘
```

### Multi-Agent Coordination

Primary agents spawn subagents via the **Task tool**:

```typescript
// Task tool creates child session
const session = await Session.create({
  parentID: ctx.sessionID,  // Links to parent
  title: params.description,
})

await SessionPrompt.prompt({
  sessionID: session.id,
  agent: params.subagent_type,  // "explore", "general"
  tools: {
    task: hasTaskPermission,    // Prevents infinite recursion
    todowrite: false,           // Subagents can't modify todos
  }
})
```

**Coordination patterns:**
- Parent-child sessions maintain hierarchy
- Permissions filtered to prevent escalation
- Recursion prevented via task permission control
- Results aggregated back to parent

### Permission Hierarchy

Default permissions (from `agent.ts`):

```typescript
const defaults = PermissionNext.fromConfig({
  "*": "allow",
  doom_loop: "ask",              // Prevent infinite tool loops
  external_directory: { "*": "ask" },
  question: "deny",
  plan_enter: "deny",
  plan_exit: "deny",
  read: { "*.env": "ask" },      // Sensitive files
})
```

Agent-specific overrides:

| Agent | Key Overrides |
|-------|---------------|
| `plan` | `edit: deny`, only writes to `.opencode/plans/*` |
| `explore` | `*: deny`, only `glob/grep/read/bash/webfetch` |
| `build` | `question: allow`, `plan_enter: allow` |

### System Prompt Assembly

Layered by priority:

```
1. Provider header (caching optimization)
     ↓
2. Agent custom prompt  OR  Provider default prompt
     ↓
3. Session system instructions (CLAUDE.md/AGENTS.md)
     ↓
4. User message context
```

---

## Configuration System

### Hierarchy

```
~/.opencode/opencode.jsonc     (global)
     ↓
./.opencode/opencode.jsonc     (project)
     ↓
Runtime flags
```

### Custom Agent Example

```jsonc
// opencode.jsonc
{
  "agent": {
    "security-auditor": {
      "model": "anthropic/claude-sonnet-4-20250514",
      "temperature": 0.3,
      "prompt": "You are a security-focused code auditor...",
      "permission": {
        "edit": "deny",
        "read": "allow"
      }
    }
  }
}
```

---

## Key Entry Points

| Entry Point | Location |
|-------------|----------|
| CLI | `packages/opencode/src/index.ts` |
| Bin | `packages/opencode/bin/opencode` |
| Web UI | `packages/app/src/` |
| Desktop | `packages/desktop/src/` |
| Server Routes | `packages/opencode/src/server/routes/` |

---

## Core Modules Reference

| Module | Location | Purpose |
|--------|----------|---------|
| Agent | `src/agent/agent.ts` | Agent config, loading, merging |
| Session | `src/session/` | Message history, execution loop |
| LLM | `src/session/llm.ts` | LLM streaming, prompt assembly |
| Provider | `src/provider/` | 20+ LLM provider abstraction |
| Tool | `src/tool/` | 45+ built-in tools |
| Permission | `src/permission/next.ts` | Rule-based access control |
| Config | `src/config/` | Configuration management |
| Storage | `src/storage/` | File-based persistence |
| MCP | `src/mcp/` | Model Context Protocol |
| Bus | `src/bus/` | Event pub/sub system |

---

## Notable Architectural Decisions

1. **Namespace over classes** - Clear module boundaries, simpler imports
2. **Zod everywhere** - Runtime validation catches bugs early
3. **Provider-agnostic** - Easy to swap/add LLM providers
4. **Multi-frontend** - Single backend serves CLI, Web, Desktop
5. **Local-first storage** - File-based, optional cloud sync
6. **Permission-first** - Deny by default, explicit allow rules
7. **MCP integration** - Standard protocol for tool extension
8. **Bun runtime** - Fast startup, native TypeScript execution

---

## Development Commands

```bash
# Install dependencies
bun install

# Run in development
bun dev [directory]

# Build standalone binary
./packages/opencode/script/build.ts --single

# Web development
bun run --cwd packages/app dev

# Desktop development
bun run --cwd packages/desktop tauri dev
```

---

*Written by Claude (Opus 4.5) | 2026-01-18 PST*
