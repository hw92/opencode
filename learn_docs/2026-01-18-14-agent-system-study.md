# Agent System Study: `agent/agent.ts`



HW：这里面是设计系统里面的 agent。 Eacch agent maps to a markdown file. 我可以生成很多特定的 AI financial agent， Macro analyst agent, Equity research agent -> md files. 这里是 agent 的定义(Agent Schema), 怎么让这些 agent 动起来还要看下一篇。



Created learn_docs/2026-01-18-14-agent-system-study.md.

  The note covers:
  - Core concept — agents are configs, not code
  - Schema — the Zod definition
  - Built-in agents — 7 agents with their purposes
  - Permission layering — defaults → agent → user
  - System prompts — the .txt files and how they're used
  - State management — lazy singleton via Instance.state()
  - User config merging — how opencode.jsonc overrides work
  - Plugin system — .md files as shareable agent definitions
  - Dynamic generation — opencode agent create CLI
  - Architecture diagram — visual summary
  - Key takeaways — 5 main insights





[TOC]

正文：

A deep dive into how OpenCode defines, loads, and manages agents.

**File:** `packages/opencode/src/agent/agent.ts` (311 lines)

---

## Core Concept

**Agents are configuration objects, not code.**

The same execution engine (`session/prompt.ts`) runs all agents. What differs is:
- System prompt (what the agent "knows")
- Permissions (what tools it can use)
- Model settings (temperature, topP, model override)
- Mode (primary vs subagent)

---

## Agent Schema

```typescript
export const Info = z.object({
  name: z.string(),
  description: z.string().optional(),
  mode: z.enum(["subagent", "primary", "all"]),
  native: z.boolean().optional(),        // Built-in vs user-defined
  hidden: z.boolean().optional(),        // Exclude from user-facing lists
  permission: PermissionNext.Ruleset,    // Tool access rules
  model: z.object({                      // Optional model override
    modelID: z.string(),
    providerID: z.string(),
  }).optional(),
  prompt: z.string().optional(),         // System prompt
  temperature: z.number().optional(),
  topP: z.number().optional(),
  options: z.record(z.string(), z.any()), // Provider-specific options
  steps: z.number().int().positive().optional(),
})
```

---

## Built-in Agents

| Agent | Mode | Purpose | Key Permissions |
|-------|------|---------|-----------------|
| `build` | primary | Execute & implement code | `question: allow`, `plan_enter: allow` |
| `plan` | primary | Read-only planning | `edit: deny` except `.opencode/plans/*.md` |
| `explore` | subagent | Codebase search | Only `glob/grep/read/bash/webfetch` |
| `general` | subagent | Multi-task execution | `todoread/todowrite: deny` |
| `compaction` | hidden | Compress history | `*: deny` (text-only) |
| `title` | hidden | Generate titles | `*: deny`, `temperature: 0.5` |
| `summary` | hidden | Summarize work | `*: deny` |

---

## Permission Layering

Permissions cascade in order:

```
defaults → agent-specific → user-config
```

### Default Permissions (baseline for all agents)

```typescript
const defaults = PermissionNext.fromConfig({
  "*": "allow",                    // Allow everything by default
  doom_loop: "ask",                // Ask before infinite loops
  external_directory: { "*": "ask" },
  question: "deny",
  plan_enter: "deny",
  plan_exit: "deny",
  read: {
    "*.env": "ask",                // Sensitive files need approval
    "*.env.example": "allow",
  },
})
```

### Agent-Specific Override Example (`explore`)

```typescript
explore: {
  permission: PermissionNext.merge(
    defaults,
    PermissionNext.fromConfig({
      "*": "deny",           // Deny everything first
      grep: "allow",         // Then whitelist specific tools
      glob: "allow",
      read: "allow",
      bash: "allow",
      webfetch: "allow",
      codesearch: "allow",
    }),
    user,                    // User config can further modify
  ),
}
```

---

## System Prompts

Each agent can have a custom prompt loaded from `.txt` files:

| File | Agent | Purpose |
|------|-------|---------|
| `prompt/title.txt` | title | Generate ≤50 char session titles |
| `prompt/summary.txt` | summary | PR-style work summary |
| `prompt/compaction.txt` | compaction | Compress conversation history |
| `prompt/explore.txt` | explore | Codebase search specialist |
| `generate.txt` | *(meta)* | Teach LLM to create new agents |

```typescript
// Loaded at top of file
import PROMPT_EXPLORE from "./prompt/explore.txt"

// Assigned to agent
explore: {
  prompt: PROMPT_EXPLORE,
  // ...
}
```

---

## State Management Pattern

```typescript
const state = Instance.state(async () => {
  // Runs ONCE per project instance
  // Returns Record<string, Agent.Info>

  const result: Record<string, Info> = {
    build: { ... },
    plan: { ... },
    // ...
  }

  // Merge user config overrides
  for (const [key, value] of Object.entries(cfg.agent ?? {})) {
    // Apply overrides...
  }

  return result
})
```

`Instance.state()` creates a **lazy, cached singleton** per project.

---

## User Config Merging

Users can customize agents in `opencode.jsonc`:

```jsonc
{
  "agent": {
    "build": {
      "temperature": 0.3,
      "model": "anthropic/claude-sonnet-4"
    },
    "my-custom-agent": {
      "prompt": "You are a specialist...",
      "mode": "subagent",
      "permission": {
        "edit": "deny"
      }
    }
  }
}
```

The merging logic (lines 197-223):

```typescript
for (const [key, value] of Object.entries(cfg.agent ?? {})) {
  if (value.disable) {
    delete result[key]  // Can disable built-in agents
    continue
  }

  let item = result[key]
  if (!item) {
    // Create new custom agent
    item = result[key] = {
      name: key,
      mode: "all",
      permission: PermissionNext.merge(defaults, user),
      native: false,  // Marks as user-defined
    }
  }

  // Override properties
  item.model = Provider.parseModel(value.model)
  item.prompt = value.prompt ?? item.prompt
  item.temperature = value.temperature ?? item.temperature
  // ...
}
```

---

## Custom Agent Files (Plugin System)

Users can create agents as `.md` files:

```
~/.opencode/agent/security-reviewer.md    (global)
.opencode/agent/code-reviewer.md          (project)
```

### File Format

```markdown
---
description: "Use this agent when reviewing code for security..."
mode: subagent
tools:
  write: false
  edit: false
---

You are an expert security auditor specializing in...
```

### Dynamic Generation

Users can generate agents via CLI:

```bash
opencode agent create
# Prompts for description, tools, mode
# Uses LLM + generate.txt to create the .md file
```

The `Agent.generate()` function:

```typescript
export async function generate(input: { description: string }) {
  const result = await generateObject({
    schema: z.object({
      identifier: z.string(),      // Agent name
      whenToUse: z.string(),       // Description
      systemPrompt: z.string(),    // Full prompt
    }),
    // ...
  })
  return result.object
}
```

---

## Public API

```typescript
// Get single agent by name
Agent.get("explore")  // → Agent.Info | undefined

// List all agents (sorted, default first)
Agent.list()  // → Agent.Info[]

// Get default agent name
Agent.defaultAgent()  // → "build" (or configured default)

// Generate new agent config via LLM
Agent.generate({ description: "..." })  // → { identifier, whenToUse, systemPrompt }
```

---

## Architecture Diagram

```
┌─────────────────────────────────────────────────────────────────┐
│                        agent/agent.ts                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌──────────────┐                                               │
│  │  Info Schema │  ← Zod schema defining agent structure        │
│  └──────────────┘                                               │
│          │                                                      │
│          ▼                                                      │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │                 Instance.state()                        │    │
│  │                                                         │    │
│  │   ┌───────────┐                                         │    │
│  │   │ defaults  │  ← Base permissions (*.env=ask, etc.)   │    │
│  │   └─────┬─────┘                                         │    │
│  │         │ merge                                         │    │
│  │         ▼                                               │    │
│  │   ┌─────────────────────────────────────────────────┐   │    │
│  │   │ Built-in Agents                                 │   │    │
│  │   │ • build   (primary)                             │   │    │
│  │   │ • plan    (primary, restricted edit)            │   │    │
│  │   │ • explore (subagent, read-only)                 │   │    │
│  │   │ • general (subagent)                            │   │    │
│  │   │ • compaction/title/summary (hidden utilities)   │   │    │
│  │   └─────┬───────────────────────────────────────────┘   │    │
│  │         │ merge                                         │    │
│  │         ▼                                               │    │
│  │   ┌───────────────┐                                     │    │
│  │   │  User Config  │  ← opencode.jsonc + .md files       │    │
│  │   └───────────────┘                                     │    │
│  │                                                         │    │
│  └─────────────────────────────────────────────────────────┘    │
│          │                                                      │
│          ▼                                                      │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │ Public API                                              │    │
│  │ • get(name)       → single agent                        │    │
│  │ • list()          → all agents                          │    │
│  │ • defaultAgent()  → default agent name                  │    │
│  │ • generate()      → LLM-generated agent config          │    │
│  └─────────────────────────────────────────────────────────┘    │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## Key Takeaways

1. **Agents = Configuration, Not Code**
   - Same execution loop runs all agents
   - Differences are in prompt, permissions, and model settings

2. **Permission Layering**
   - `defaults → agent-specific → user-config`
   - Fine-grained control via patterns (e.g., `"*.env": "ask"`)

3. **Plugin System via .md Files**
   - Save agents in `~/.opencode/agent/` (global) or `.opencode/agent/` (project)
   - Share by copying files
   - Generate via CLI with LLM assistance

4. **Smart Defaults**
   - `explore` locked to read-only tools
   - `plan` can only write to `.opencode/plans/`
   - Sensitive files (`.env`) require approval

5. **Lazy Singleton Pattern**
   - `Instance.state()` computes agent registry once per project
   - Efficient for repeated `Agent.get()` calls

---

*Written by Claude (Opus 4.5) | 2026-01-18 14:00 PST*
