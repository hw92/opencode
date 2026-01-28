# OpenCode Permission System Deep Dive

[TOC]

A comprehensive study of OpenCode's tool permission rules, evaluation, and approval flow.

---

## Overview

OpenCode's permission system uses a **rule-based evaluation model** with three possible actions:
- **allow** - Grant permission without asking
- **deny** - Reject, halt execution
- **ask** - Prompt user for approval

The system uses **last-matching-wins** semantics with bidirectional wildcard matching.

---

## Architecture

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                      PERMISSION EVALUATION FLOW                              │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  Tool Execution                                                              │
│       │                                                                      │
│       ▼                                                                      │
│  ctx.ask({ permission, patterns, metadata })                                 │
│       │                                                                      │
│       ▼                                                                      │
│  ┌─────────────────────────────────────────────────────────────────────┐    │
│  │  PermissionNext.evaluate()                                          │    │
│  │                                                                     │    │
│  │  1. Merge rulesets: defaults + agent + config + approved            │    │
│  │  2. Find LAST rule matching:                                        │    │
│  │     - Permission (via wildcard)                                     │    │
│  │     - Pattern (via wildcard)                                        │    │
│  │  3. Return matched rule or default to "ask"                         │    │
│  └───────────────────────────────────┬─────────────────────────────────┘    │
│                                      │                                       │
│              ┌───────────────────────┼───────────────────────┐              │
│              ▼                       ▼                       ▼              │
│        ┌──────────┐           ┌──────────┐           ┌──────────┐          │
│        │  ALLOW   │           │   ASK    │           │   DENY   │          │
│        │          │           │          │           │          │          │
│        │ Continue │           │ Prompt   │           │ Throw    │          │
│        │ execution│           │ user     │           │ DeniedErr│          │
│        └──────────┘           └────┬─────┘           └──────────┘          │
│                                    │                                        │
│                    ┌───────────────┼───────────────┐                       │
│                    ▼               ▼               ▼                       │
│              ┌──────────┐   ┌──────────┐   ┌──────────┐                    │
│              │  ONCE    │   │  ALWAYS  │   │  REJECT  │                    │
│              │          │   │          │   │          │                    │
│              │ Allow    │   │ Add to   │   │ Throw    │                    │
│              │ this one │   │ approved │   │ Rejected │                    │
│              └──────────┘   └──────────┘   └──────────┘                    │
│                                                                              │
└──────────────────────────────────────────────────────────────────────────────┘
```

---

## Permission Rule Structure

### Rule Definition

```typescript
interface Rule {
  permission: string      // Tool/feature ID (bash, edit, read, etc.)
  pattern: string         // Pattern to match (* for wildcards)
  action: "allow" | "deny" | "ask"
}

type Ruleset = Rule[]
```

### Permission Request

```typescript
interface PermissionRequest {
  id: string              // Unique ascending ID
  sessionID: string       // Current session
  permission: string      // What permission
  patterns: string[]      // Specific items being accessed
  metadata: Record        // Additional context
  always: string[]        // Patterns to remember if "always"
  tool?: {
    messageID: string
    callID: string
  }
}
```

---

## Rule Evaluation Algorithm

The evaluation uses **last-matching-wins** semantics:

```typescript
function evaluate(permission: string, pattern: string, ruleset: Ruleset) {
  let matched: Rule | undefined

  for (const rule of ruleset) {
    // Check permission matches (with wildcards)
    if (!wildcardMatch(permission, rule.permission)) continue

    // Check pattern matches (with wildcards)
    if (!wildcardMatch(pattern, rule.pattern)) continue

    // Last match wins
    matched = rule
  }

  return matched ?? { action: "ask" }  // Default to ask
}
```

### Key Properties

1. **Order matters** - Later rules override earlier ones
2. **Bidirectional wildcards** - Both permission AND pattern use wildcards
3. **Last match wins** - `rm` followed by `*` = allow (not deny)
4. **Default fallback** - Unknown permissions default to "ask"

---

## Wildcard Matching

**File:** `src/util/wildcard.ts`

```typescript
// Pattern transformations:
// * → .* (matches any characters)
// ? → . (matches single character)
// Special regex chars escaped
// "ls *" → optional trailing part

function wildcardMatch(str: string, pattern: string): boolean {
  const regex = patternToRegex(pattern)
  return regex.test(str)
}
```

### Examples

| Permission | Pattern | Matches |
|------------|---------|---------|
| `bash` | `rm *` | "rm -rf /", "rm file.txt" |
| `edit` | `src/*` | "src/foo.ts", "src/components/Bar.tsx" |
| `*` | `*` | Everything (fallback rule) |
| `mcp_*` | `*` | "mcp_server", "mcp_client" |

---

## Permission Types

### File Operations

| Permission | Description |
|------------|-------------|
| `read` | File reading |
| `edit` | File editing (edit, write, patch, multiedit) |
| `external_directory` | Accessing files outside project |

### Tool Execution

| Permission | Description |
|------------|-------------|
| `bash` | Shell commands |
| `task` | Task operations |

### Code Analysis

| Permission | Description |
|------------|-------------|
| `glob` | File pattern matching |
| `grep` | Code searching |
| `codesearch` | Advanced code search |
| `lsp` | Language server operations |

### Information Access

| Permission | Description |
|------------|-------------|
| `webfetch` | HTTP requests |
| `websearch` | Web search |
| `question` | User questions |
| `list` | File listing |

### Agent Control

| Permission | Description |
|------------|-------------|
| `todowrite` | Writing todos |
| `todoread` | Reading todos |
| `doom_loop` | Loop detection |
| `plan_enter` | Enter plan mode |
| `plan_exit` | Exit plan mode |

---

## Agent-Specific Permissions

Each agent defines its own permission context:

```typescript
// Agent permissions merge in order:
// 1. Defaults (hardcoded baseline)
// 2. Agent-specific config
// 3. User config (opencode.json)

const agentRuleset = merge(
  defaults,
  agent.permission,
  config.permission
)
```

### Example: "explore" Agent

```typescript
// Explore agent is restricted to read-only operations
{
  permission: {
    "*": "deny",           // Default deny everything
    "grep": "allow",       // Allow searching
    "glob": "allow",       // Allow file finding
    "read": "allow",       // Allow reading
    "bash": {
      "*": "deny",
      "git log *": "allow" // Only git log allowed
    }
  }
}
```

---

## Default Permission Hierarchy

**File:** `src/agent/agent.ts`

```typescript
const defaults = {
  "*": "allow",                    // Everything allowed by default

  "doom_loop": "ask",              // Detect repetitive tool calls

  "external_directory": {
    "*": "ask",                    // Ask before outside project
    [Truncate.DIR]: "allow",       // Allow truncation directory
  },

  "question": "deny",              // Don't ask user questions
  "plan_enter": "deny",            // Don't enter plan mode
  "plan_exit": "deny",             // Don't exit plan mode

  "read": {
    "*": "allow",                  // Read everything
    "*.env": "ask",                // Except .env files
    "*.env.*": "ask",              // And .env.* files
    "*.env.example": "allow",      // But .env.example is ok
  },
}
```

---

## Approval Flow

### Request Flow

```
Tool calls ctx.ask()
      │
      ▼
PermissionNext.ask() evaluates rules
      │
      ├─ "allow" → Resolves immediately
      │
      ├─ "deny" → Throws DeniedError
      │
      └─ "ask" → Creates pending request
                 Bus publishes Event.Asked
                 Promise stored in pending[]
                 UI displays prompt
```

### Response Handling

```
User responds (once/always/reject)
      │
      ▼
PermissionNext.reply()
      │
      ├─ "once" → Resolve promise, continue
      │
      ├─ "always" → Add to approved[]
      │             Resolve ALL matching pending
      │
      └─ "reject" → Throw RejectedError
                    REJECT ALL pending in session
```

---

## Tool Integration

Every tool gets a `ctx.ask()` function:

```typescript
// In bash.ts
async function execute(args, ctx) {
  const { command } = args

  // Check for external directories
  const directories = findExternalDirs(command)
  if (directories.size > 0) {
    await ctx.ask({
      permission: "external_directory",
      patterns: Array.from(directories),
      always: Array.from(directories).map(d => path.dirname(d) + "*"),
      metadata: { command },
    })
  }

  // Check bash permission
  await ctx.ask({
    permission: "bash",
    patterns: [command],
    always: ["git *", "npm *"],  // Common safe patterns
    metadata: { command },
  })

  // Execute command...
}
```

---

## Error Types

```typescript
// User rejected without message
class RejectedError extends Error {
  // Halts execution
}

// User rejected with guidance
class CorrectedError extends Error {
  message: string  // User's feedback
  // Continues with feedback
}

// Config rule prevents action
class DeniedError extends Error {
  rule: Rule  // The denying rule
  // Halts execution
}
```

---

## Permission Persistence

```typescript
// Storage location
Storage.read(["permission", projectID])

// Loaded on Instance initialization
const approved = await Storage.read<Ruleset>(["permission", projectID])
  .catch(() => [] as Ruleset)

// Currently session-scoped only
// TODO: UI to manage persistent permissions
```

---

## Configuration Examples

```json
{
  "permission": {
    // All bash commands require approval
    "bash": "ask",

    // Nested permissions
    "edit": {
      "*": "deny",           // Default deny edits
      "src/**": "allow",     // Allow src/ edits
      "src/secret/*": "deny" // Except secret/
    },

    // External access
    "external_directory": "ask",

    // Sensitive files
    "read": {
      "*.env": "ask",
      "*.env.example": "allow"
    }
  }
}
```

---

## Disabled Tool Detection

```typescript
// Determines which tools are completely disabled
function disabled(tools: string[], ruleset: Ruleset): Set<string> {
  const result = new Set<string>()

  for (const tool of tools) {
    // Check if there's a deny rule with pattern "*"
    const rule = evaluate(tool, "*", ruleset)
    if (rule.action === "deny" && rule.pattern === "*") {
      result.add(tool)
    }
  }

  return result
}

// Used in LLM.resolveTools() to filter tool list
const disabled = PermissionNext.disabled(allTools, agent.permission)
const availableTools = allTools.filter(t => !disabled.has(t.id))
```

---

## Design Patterns

| Pattern | Purpose |
|---------|---------|
| **Rule Merging** | Flat arrays, order preserved |
| **Last Matching Wins** | Later rules override earlier |
| **Bidirectional Wildcards** | Permission AND pattern matching |
| **Session-Scoped Approval** | Rejecting one cancels all pending |
| **Pattern-Oriented** | About WHAT operations, not HOW |

---

## Key Files

| File | Purpose |
|------|---------|
| `src/permission/next.ts` | Core permission logic (269 lines) |
| `src/config/config.ts` | Permission schema |
| `src/agent/agent.ts` | Default permissions |
| `src/tool/tool.ts` | Tool context interface |
| `src/util/wildcard.ts` | Wildcard matching |
| `test/permission/next.test.ts` | 650+ lines of tests |

---

*Written by Claude (Opus 4.5) | 2026-01-22 16:00 PST*
