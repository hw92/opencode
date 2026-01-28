# OpenCode Config System Deep Dive

[TOC]

A comprehensive study of OpenCode's configuration loading, validation, and merging system.

---

## Overview

OpenCode uses a **multi-layer configuration system** that merges settings from multiple sources. The system supports JSON/JSONC files, YAML frontmatter in markdown, and environment variables.

---

## Configuration Hierarchy

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    CONFIGURATION PRECEDENCE                                  │
│                    (Lowest to Highest Priority)                              │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  1. Remote Config     ─► .well-known/opencode (organizational defaults)     │
│         ↓                                                                    │
│  2. Global Config     ─► ~/.config/opencode/opencode.json (user prefs)      │
│         ↓                                                                    │
│  3. Custom Config     ─► OPENCODE_CONFIG env var (custom path)              │
│         ↓                                                                    │
│  4. Project Config    ─► opencode.json in project root                      │
│         ↓                                                                    │
│  5. .opencode Dirs    ─► agents, commands, plugins folders                  │
│         ↓                                                                    │
│  6. Inline Config     ─► OPENCODE_CONFIG_CONTENT env var (runtime)          │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## File Locations (XDG Paths)

OpenCode follows the XDG Base Directory Specification:

```typescript
// packages/opencode/src/global/index.ts
export namespace Global {
  export const Path = {
    data:   "$XDG_DATA_HOME/opencode",    // ~/.local/share/opencode
    config: "$XDG_CONFIG_HOME/opencode",  // ~/.config/opencode
    cache:  "$XDG_CACHE_HOME/opencode",   // ~/.cache/opencode
    state:  "$XDG_STATE_HOME/opencode",   // ~/.local/state/opencode
    bin:    "$XDG_DATA_HOME/opencode/bin",
    log:    "$XDG_DATA_HOME/opencode/log",
  }
}
```

---

## Supported File Formats

| Format | File Names | Notes |
|--------|------------|-------|
| JSON | `opencode.json` | Strict JSON |
| JSONC | `opencode.jsonc` | JSON with Comments |
| Legacy | `config.json` | Backwards compatibility |
| TOML | `opencode.toml` | Deprecated, auto-migrated |

---

## Configuration Schema

**File:** `packages/opencode/src/config/config.ts`

### Core Sections

```typescript
interface Config.Info {
  // Schema reference
  $schema?: string

  // Display
  theme?: string
  username?: string

  // Models
  model?: string              // "provider/model" format
  small_model?: string        // For lightweight tasks

  // UI
  keybinds?: Record<string, string>
  tui?: {
    scrollSpeed?: number
    acceleration?: number
    diffStyle?: "unified" | "split"
  }

  // Server
  server?: {
    port?: number
    hostname?: string
    mdns?: boolean
    cors?: string[]
  }

  // Logging
  logLevel?: "debug" | "info" | "warn" | "error"
}
```

### Agent Configuration

```typescript
interface Config.Info {
  agent?: Record<string, AgentConfig>
  default_agent?: string

  // Deprecated (auto-migrated to agent)
  mode?: Record<string, ModeConfig>
}

interface AgentConfig {
  model?: string
  temperature?: number
  prompt?: string
  permission?: PermissionRuleset
  tools?: string[]
}
```

### Command Configuration

```typescript
interface Config.Info {
  command?: Record<string, CommandConfig>
}

interface CommandConfig {
  template?: string
  description?: string
}
```

### Plugin System

```typescript
interface Config.Info {
  plugin?: string[]  // npm packages or file:// URLs
}

// Examples:
// - "@opencode/plugin-custom"
// - "file:///path/to/plugin.js"
```

### Provider Configuration

```typescript
interface Config.Info {
  provider?: Record<string, ProviderConfig>
  enabled_providers?: string[]   // Allowlist
  disabled_providers?: string[]  // Blocklist
}

interface ProviderConfig {
  api_key?: string
  base_url?: string
  models?: Record<string, ModelOverride>
}
```

### MCP Servers

```typescript
interface Config.Info {
  mcp?: Record<string, McpConfig>
}

type McpConfig = McpLocal | McpRemote

interface McpLocal {
  command: string[]
  env?: Record<string, string>
}

interface McpRemote {
  url: string
  oauth?: OAuthConfig
}
```

### Permissions

```typescript
interface Config.Info {
  permission?: PermissionRuleset

  // Legacy (auto-converted)
  tools?: Record<string, boolean>
}

type PermissionRuleset = Record<string, PermissionRule>
type PermissionRule = "allow" | "deny" | "ask" | Record<string, PermissionRule>
```

### Advanced Options

```typescript
interface Config.Info {
  // Sharing
  share?: "manual" | "auto" | "disabled"

  // Updates
  autoupdate?: "stable" | "preview" | "disabled"

  // Context management
  compaction?: {
    threshold?: number
    target?: number
  }

  // File watching
  watcher?: {
    ignore?: string[]
  }

  // Instructions
  instructions?: string[]  // File paths/globs

  // Experimental
  experimental?: {
    hooks?: boolean
    batching?: boolean
    mcpTimeout?: number
  }
}
```

---

## Config Merging Strategy

### Deep Merge with Array Concatenation

```typescript
// packages/opencode/src/config/config.ts
function mergeConfigConcatArrays(target: Info, source: Info): Info {
  // Deep merge objects
  const merged = mergeDeep(target, source)

  // Special handling: concatenate and deduplicate arrays
  if (target.plugin && source.plugin) {
    merged.plugin = Array.from(new Set([
      ...target.plugin,
      ...source.plugin
    ]))
  }

  if (target.instructions && source.instructions) {
    merged.instructions = Array.from(new Set([
      ...target.instructions,
      ...source.instructions
    ]))
  }

  return merged
}
```

### Key Behaviors

| Type | Merge Behavior |
|------|----------------|
| Objects | Deep merge (later overrides earlier) |
| Arrays (plugin, instructions) | Concatenate + deduplicate |
| Scalars | Later wins |
| Undefined | Preserves earlier value |

---

## Environment Variable Handling

### Config-Specific Variables

```bash
# Custom config file path
OPENCODE_CONFIG=/path/to/config.json

# Custom config directory
OPENCODE_CONFIG_DIR=/path/to/dir

# Inline JSON config (highest priority)
OPENCODE_CONFIG_CONTENT='{"model":"anthropic/claude-3-opus"}'

# JSON permission rules
OPENCODE_PERMISSION='{"bash":"allow"}'
```

### Template Substitution

Config values support interpolation:

```json
{
  "provider": {
    "openai": {
      "api_key": "{env:OPENAI_API_KEY}"
    }
  },
  "instructions": [
    "{file:/path/to/instructions.md}"
  ]
}
```

---

## Markdown-Based Config

### Agent/Command Loading

```
.opencode/
├── agents/
│   └── analyst.md      ─► Loaded as agent config
├── commands/
│   └── deploy.md       ─► Loaded as command
└── modes/
    └── review.md       ─► Deprecated (migrated to agents)
```

### Frontmatter Parsing

```markdown
---
model: anthropic/claude-3-opus
temperature: 0.7
permission:
  bash: allow
  edit:
    "*": deny
    "src/**": allow
---

You are a code analyst. Focus on...
```

```typescript
// ConfigMarkdown.parse() handles YAML frontmatter
// Preprocessor converts colons in values to block scalars
// Glob patterns: {agent,agents}/**/*.md
```

---

## Plugin Deduplication

```typescript
function deduplicatePlugins(plugins: string[]): string[] {
  const seen = new Map<string, string>()

  for (const specifier of plugins) {
    // Extract canonical name
    // file:///path/plugin.js → "plugin"
    // @scope/pkg@1.0.0 → "@scope/pkg"
    const canonical = extractCanonical(specifier)

    // Later entries win
    seen.set(canonical, specifier)
  }

  return Array.from(seen.values())
}
```

---

## Config Access Patterns

### Runtime Access

```typescript
// Get merged config
const config = await Config.get()

// Get config with metadata
const { config, directories } = await Config.state()

// Get directories used for config
const dirs = await Config.directories()

// Get global config only
const global = await Config.global()
```

### Lazy Initialization

```typescript
// Config uses Instance.state() for memoized lazy loading
const state = Instance.state(async () => {
  const config = await loadAndMergeConfigs()
  return { config, directories }
})

export async function get() {
  return (await state()).config
}
```

---

## Error Handling

### Custom Error Types

```typescript
// JSON parsing errors
ConfigJsonError

// Zod validation failures
ConfigInvalidError

// YAML frontmatter issues
ConfigFrontmatterError

// Directory path suggestions
ConfigDirectoryTypoError
```

### Graceful Fallbacks

```typescript
// Missing config file → empty object
const config = await loadConfig(path).catch(() => ({}))

// Invalid field → use default
const model = config.model ?? "anthropic/claude-3-sonnet"
```

---

## Validation with Zod

```typescript
// packages/opencode/src/config/config.ts
export const Info = z.object({
  $schema: z.string().optional(),
  theme: z.string().optional(),
  model: z.string().optional(),
  // ... all fields with Zod schemas
})
.passthrough()  // Allow unknown fields
.refine(customValidation)

// Custom validations
.refine((data) => {
  // Custom LSP servers must have extensions
  if (data.lsp && typeof data.lsp !== "boolean") {
    for (const [id, config] of Object.entries(data.lsp)) {
      if (!isBuiltIn(id) && !config.disabled && !config.extensions) {
        return false
      }
    }
  }
  return true
})
```

---

## Config Example

```jsonc
// opencode.json
{
  "$schema": "https://opencode.ai/schema/config.json",

  // Default model
  "model": "anthropic/claude-3-opus",
  "small_model": "anthropic/claude-3-haiku",

  // Agents
  "agent": {
    "analyst": {
      "model": "anthropic/claude-3-opus",
      "temperature": 0.3,
      "permission": {
        "bash": "deny",
        "read": "allow"
      }
    }
  },

  // Providers
  "provider": {
    "openai": {
      "api_key": "{env:OPENAI_API_KEY}"
    }
  },

  // MCP servers
  "mcp": {
    "filesystem": {
      "command": ["npx", "-y", "@modelcontextprotocol/server-filesystem"]
    }
  },

  // Permissions
  "permission": {
    "bash": {
      "*": "ask",
      "git *": "allow",
      "npm *": "allow"
    }
  },

  // Instructions
  "instructions": [
    "CLAUDE.md",
    ".github/CONTRIBUTING.md"
  ]
}
```

---

## Key Files

| File | Purpose |
|------|---------|
| `src/config/config.ts` | Core config logic and schema |
| `src/config/markdown.ts` | Frontmatter parsing |
| `src/global/index.ts` | XDG path definitions |

---

*Written by Claude (Opus 4.5) | 2026-01-22 16:00 PST*
