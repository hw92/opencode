# Session Execution Engine: `session/prompt.ts`

This is the **runtime that makes agents work**. While `agent/agent.ts` defines what agents are, `session/prompt.ts` actually runs them.

**File:** `packages/opencode/src/session/prompt.ts` (1793 lines)

---

## Core Concept

**`session/prompt.ts` is the execution loop.**

It takes user input, runs the agent loop, executes tools, and manages conversation flow until the agent completes its task.

---

## Main Functions

| Function | Purpose |
|----------|---------|
| `prompt()` | Entry point - receives user message |
| `loop()` | Main agentic loop - keeps running until done |
| `resolveTools()` | Filters tools by agent permissions |
| `createUserMessage()` | Parses input (text, files, @agent) |
| `insertReminders()` | Injects plan mode prompts |
| `command()` | Executes slash commands (/commit, etc.) |
| `shell()` | Direct bash execution by user |
| `ensureTitle()` | Auto-generates session titles |

---

## The Main Loop: `loop()`

This is the heart of the system (lines 257-634):

```typescript
while (true) {
  // 1. Get messages from session
  let msgs = await MessageV2.stream(sessionID)

  // 2. Find last user & assistant messages
  let lastUser, lastAssistant

  // 3. Exit condition: agent finished and no pending user message
  if (lastAssistant?.finish && lastUser.id < lastAssistant.id) {
    break
  }

  // 4. Handle pending subtask (Task tool spawned a subagent)
  if (task?.type === "subtask") {
    await taskTool.execute(...)
    continue
  }

  // 5. Handle compaction (context overflow)
  if (task?.type === "compaction") {
    await SessionCompaction.process(...)
    continue
  }

  // 6. Normal processing
  const agent = await Agent.get(lastUser.agent)
  const tools = await resolveTools({ agent, session, model })

  const result = await processor.process({
    agent, messages, tools, model
  })

  if (result === "stop") break
}
```

**Key insight:** The loop continues until the LLM returns a finish state that isn't `tool-calls`.

---

## Tool Resolution: `resolveTools()`

Where **agent permissions meet tool availability** (lines 643-819):

```typescript
async function resolveTools(input) {
  const tools = {}

  // 1. Get tools filtered by agent permissions
  for (const item of await ToolRegistry.tools(providerID, agent)) {
    tools[item.id] = tool({
      async execute(args, options) {
        // Plugin hooks wrap execution
        await Plugin.trigger("tool.execute.before", ...)
        const result = await item.execute(args, ctx)
        await Plugin.trigger("tool.execute.after", ...)
        return result
      },
    })
  }

  // 2. Add MCP tools with permission checks
  for (const [key, item] of await MCP.tools()) {
    item.execute = async (args) => {
      await ctx.ask({ permission: key })  // Permission check
      return execute(args)
    }
    tools[key] = item
  }

  return tools
}
```

**Result:** Only tools allowed by agent permissions are available to the LLM.

---

## Message Parsing: `createUserMessage()`

Handles different input types (lines 821-1188):

| Input Type | Handling |
|------------|----------|
| `text` | Direct text content |
| `file` (text) | Reads via `ReadTool`, adds content |
| `file` (directory) | Lists via `ListTool` |
| `file` (image/binary) | Encodes as base64 data URL |
| `agent` (@explore) | Adds instruction to call Task tool |
| MCP resource | Fetches from MCP server |

**Example:** When you type `@explore find auth`:
```typescript
if (part.type === "agent") {
  return [
    { ...part },
    {
      type: "text",
      synthetic: true,
      text: "Use the above message to call task tool with subagent: explore",
    },
  ]
}
```

---

## Plan Mode: `insertReminders()`

Injects special prompts when entering/exiting plan mode (lines 1190-1328):

**Entering plan mode:**
```typescript
if (agent.name === "plan") {
  userMessage.parts.push({
    text: `<system-reminder>
      Plan mode is active. You MUST NOT make any edits...

      ## Plan Workflow
      ### Phase 1: Initial Understanding
      ### Phase 2: Design
      ### Phase 3: Review
      ### Phase 4: Final Plan
      ### Phase 5: Call plan_exit tool
    </system-reminder>`,
    synthetic: true,
  })
}
```

**Switching from plan to build:**
```typescript
if (agent.name !== "plan" && previousAgent === "plan") {
  userMessage.parts.push({
    text: BUILD_SWITCH + `A plan file exists at ${plan}...`,
  })
}
```

---

## Slash Commands: `command()`

Executes custom commands (lines 1594-1719):

```typescript
export async function command(input) {
  const command = await Command.get(input.command)  // e.g., "/commit"

  // Replace placeholders
  let template = command.template
    .replaceAll("$1", args[0])
    .replaceAll("$ARGUMENTS", input.arguments)

  // Execute shell snippets: !`git status`
  const shellResults = await executeShellSnippets(template)

  // Run as subtask if agent is subagent mode
  if (agent.mode === "subagent") {
    parts = [{ type: "subtask", agent: agent.name, prompt: template }]
  }

  return prompt({ sessionID, parts, agent })
}
```

---

## Execution Flow

```
User: "fix the bug in auth.ts"
           │
           ▼
┌──────────────────────────────────────────────────────────────┐
│  prompt()                                                    │
│  └─► createUserMessage() → Save to session                   │
│  └─► loop()                                                  │
└──────────────────────┬───────────────────────────────────────┘
                       │
        ┌──────────────┴──────────────┐
        ▼                             │
┌───────────────────┐                 │
│  Step 1           │                 │
│  • Load agent     │                 │
│  • Resolve tools  │                 │
│  • Call LLM       │                 │
│  ↓                │                 │
│  LLM: read(auth)  │                 │
│  → Execute tool   │                 │
└───────┬───────────┘                 │
        │                             │
        ▼                             │
┌───────────────────┐                 │
│  Step 2           │                 │
│  • LLM sees file  │                 │
│  ↓                │                 │
│  LLM: edit(auth)  │                 │
│  → Execute tool   │                 │
└───────┬───────────┘                 │
        │                             │
        ▼                             │
┌───────────────────┐                 │
│  Step 3           │                 │
│  • LLM: done      │                 │
│  • finish="end"   │                 │
│  → Exit loop ─────┼─────────────────┘
└───────────────────┘
           │
           ▼
    Return response
```

---

## Relationship with `agent/agent.ts`

| `agent/agent.ts` | `session/prompt.ts` |
|------------------|---------------------|
| **Defines** agents | **Runs** agents |
| Schema & permissions | Execution loop |
| Static configuration | Dynamic runtime |
| What tools are allowed | Actually calls tools |
| System prompts | Assembles full prompt |

```
agent/agent.ts                    session/prompt.ts
──────────────                    ─────────────────
Agent.Info {                      loop() {
  name: "explore"                   agent = Agent.get("explore")
  permission: [...]     ────►       tools = resolveTools(agent)
  prompt: "You are..."              result = LLM.stream({ agent, tools })
}                                 }
```

---

## Key Takeaways

1. **`loop()` is the engine** - Runs until LLM signals completion
2. **Tools are permission-wrapped** - `resolveTools()` enforces agent permissions
3. **Subtasks create child sessions** - Task tool spawns isolated agent runs
4. **Context overflow triggers compaction** - `compaction` agent summarizes history
5. **Synthetic parts guide behavior** - Hidden text steers agent (plan mode, @agent)
6. **Everything persists** - Messages and parts saved to session storage
7. **Plugin hooks everywhere** - `tool.execute.before/after` for extensibility

---

## Files in This Study Series

| File | Topic |
|------|-------|
| `2026-01-18-opencode-architecture-study.md` | Overall codebase structure |
| `2026-01-18-14-agent-system-study.md` | Agent definition & configuration |
| `2026-01-18-22-session-prompt-study.md` | Execution engine (this file) |

**Next:** `session/llm.ts` (LLM streaming), `tool/registry.ts` (tool discovery), or `session/processor.ts` (stream processing)

---

*Written by Claude (Opus 4.5) | 2026-01-18 22:00 PST*
