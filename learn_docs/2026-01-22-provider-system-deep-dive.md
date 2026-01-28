# OpenCode Provider System Deep Dive

[TOC]

A comprehensive study of how OpenCode abstracts 20+ LLM providers into a unified interface.

---

## Overview

OpenCode's provider system uses the **Vercel AI SDK** as its foundation, supporting 20+ LLM providers through a unified abstraction layer. The system handles provider registration, model capabilities, message transformations, and API-specific quirks.

---

## Architecture

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         PROVIDER SYSTEM                                      │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  ┌──────────────────────────────────────────────────────────────────────┐   │
│  │                      Provider.get() / Provider.list()                 │   │
│  │  • Load from models.dev database                                      │   │
│  │  • Apply environment variables                                        │   │
│  │  • Merge user config overrides                                        │   │
│  └────────────────────────────────────────────────────────────────────┬─┘   │
│                                                                        │     │
│  ┌────────────────────────────────────────────────────────────────────▼─┐   │
│  │                        Custom Loaders                                 │   │
│  │  • Provider-specific initialization                                   │   │
│  │  • OAuth/API key handling                                             │   │
│  │  • Region/endpoint configuration                                      │   │
│  │  • Model filtering and auto-loading                                   │   │
│  └────────────────────────────────────────────────────────────────────┬─┘   │
│                                                                        │     │
│  ┌────────────────────────────────────────────────────────────────────▼─┐   │
│  │                     ProviderTransform                                 │   │
│  │  • normalizeMessages() - Provider-specific message fixes              │   │
│  │  • applyCaching() - Automatic prompt caching                          │   │
│  │  • unsupportedParts() - Filter unsupported modalities                 │   │
│  │  • temperature() - Model-specific defaults                            │   │
│  │  • variants() - Reasoning effort configurations                       │   │
│  └────────────────────────────────────────────────────────────────────┬─┘   │
│                                                                        │     │
│  ┌────────────────────────────────────────────────────────────────────▼─┐   │
│  │                        AI SDK                                         │   │
│  │  • streamText() / generateText()                                      │   │
│  │  • Provider-specific SDK instances                                    │   │
│  └──────────────────────────────────────────────────────────────────────┘   │
│                                                                              │
└──────────────────────────────────────────────────────────────────────────────┘
```

---

## Bundled Providers

OpenCode includes 20+ bundled providers:

| Provider | ID | Special Features |
|----------|-----|------------------|
| Anthropic | `anthropic` | Cache control, extended API |
| OpenAI | `openai` | Codex mode, responses API |
| Azure OpenAI | `azure` | Store ID handling |
| Google Generative AI | `google` | Thinking configs |
| Google Vertex | `vertex` | Project/location config |
| Amazon Bedrock | `bedrock` | Region prefixing |
| OpenRouter | `openrouter` | Multi-provider routing |
| Mistral | `mistral` | Tool ID normalization |
| Groq | `groq` | Fast inference |
| DeepInfra | `deepinfra` | - |
| Cerebras | `cerebras` | - |
| Cohere | `cohere` | - |
| TogetherAI | `togetherai` | - |
| Perplexity | `perplexity` | - |
| XAI | `xai` | - |
| GitLab | `gitlab` | - |
| GitHub Copilot | `copilot` | Device flow auth |
| Vercel AI Gateway | `vercel` | - |

---

## Provider Registration Pattern

### Three-Level Loading System

```typescript
// 1. Database Level - Models from models.dev
const modelsDb = await ModelsDev.get()

// 2. Environment Level - API keys from env
const apiKey = process.env[`${PROVIDER}_API_KEY`]

// 3. Config Level - User overrides in opencode.json
const userConfig = config.provider?.[providerID]
```

### Custom Loaders

Each provider can have a custom loader for initialization:

```typescript
const CUSTOM_LOADERS: Record<string, CustomLoader> = {
  anthropic: async (provider, config) => {
    // Add beta headers for claude-code features
    // Handle caching optimization
    return { ...provider, options: { ... } }
  },
  bedrock: async (provider, config) => {
    // Region-aware model ID prefixing
    // AWS credential chain handling
    return { ...provider, models: transformedModels }
  },
  // ... more loaders
}
```

---

## Model Capabilities System

### Capability Schema

```typescript
interface ModelCapabilities {
  temperature: boolean      // Supports temperature control
  reasoning: boolean        // Has reasoning/thinking mode
  attachment: boolean       // Supports file attachments
  toolcall: boolean         // Can use tools
  input: {
    text: boolean
    audio: boolean
    image: boolean
    video: boolean
    pdf: boolean
  }
  output: {
    text: boolean
    audio: boolean
    image: boolean
    video: boolean
    pdf: boolean
  }
  interleaved: boolean | {
    field: "reasoning_content" | "reasoning_details"
  }
}
```

### Cost Tracking

```typescript
interface ModelCost {
  input: number           // Cost per input token
  output: number          // Cost per output token
  cacheRead?: number      // Cache read cost
  cacheWrite?: number     // Cache write cost
  // Special pricing for 200K+ context
}
```

---

## Message Transformations

**File:** `packages/opencode/src/provider/transform.ts`

### Key Transform Functions

```typescript
export namespace ProviderTransform {
  // Fix provider-specific message issues
  export function normalizeMessages(messages, model) {
    // Anthropic: Filter empty content
    // Claude: Normalize tool call IDs
    // Mistral: Add assistant response after tool messages
  }

  // Setup automatic prompt caching
  export function applyCaching(messages, model) {
    // Anthropic, OpenRouter, Bedrock support
  }

  // Filter unsupported modalities
  export function unsupportedParts(messages, model) {
    // Returns helpful error messages
  }

  // Model-specific temperature defaults
  export function temperature(model) {
    // Reasoning models: undefined (no temp)
    // Standard models: configured default
  }

  // Reasoning effort configurations
  export function variants(model) {
    // low/medium/high reasoning effort
  }

  // Token budget for reasoning models
  export function maxOutputTokens(model) {
    // Proper budgeting for thinking tokens
  }

  // JSON schema normalization for Gemini
  export function schema(schema, model) {
    // Enum values → strings
  }
}
```

---

## Provider-Specific Handling

### Amazon Bedrock

Complex region prefixing logic for cross-region inference:

```typescript
// Region-aware model ID prefixing
// us., eu., ap., au., jp., apac.
const regionPrefix = getRegionPrefix(region)
const modelId = `${regionPrefix}${baseModelId}`
```

### Anthropic

```typescript
// Beta headers for claude-code features
headers: {
  "anthropic-beta": "claude-code-2024-12-01"
}

// Empty message filtering
messages = messages.filter(m => m.content.length > 0)
```

### GitHub Copilot

```typescript
// Custom responses() API for Codex models
// Dual authentication (copilot + enterprise)
// Feature gate handling
```

### Mistral

```typescript
// Tool IDs must be 9 alphanumeric chars
// Tool messages need preceding assistant response
```

---

## SDK Loading & Caching

### Dynamic SDK Management

```typescript
async function getSDK(provider: Provider.Info) {
  const key = `${provider.id}:${JSON.stringify(provider.options)}`

  if (sdkCache.has(key)) {
    return sdkCache.get(key)
  }

  // Bundled providers loaded directly
  if (BUNDLED_PROVIDERS.includes(provider.id)) {
    const sdk = loadBundledSDK(provider)
    sdkCache.set(key, sdk)
    return sdk
  }

  // External providers installed via BunProc.install()
  await BunProc.install(provider.package)
  const sdk = await import(provider.package)
  sdkCache.set(key, sdk)
  return sdk
}
```

---

## Configuration Merging

### Priority Order (Lowest to Highest)

1. **Models.dev database** - Base model definitions
2. **Config file definitions** - `opencode.json` provider config
3. **Plugin-provided auth** - OAuth/API key loaders
4. **Auth stored credentials** - Saved OAuth tokens
5. **Environment variables** - `PROVIDER_API_KEY`

### Model Filtering

```typescript
// Blacklist/whitelist support
// Alpha model gating via flag
// Deprecated model removal
// Variant-level disabling

const filtered = models.filter(m => {
  if (config.blacklist?.includes(m.id)) return false
  if (config.whitelist && !config.whitelist.includes(m.id)) return false
  if (m.alpha && !Flag.ENABLE_ALPHA) return false
  return !m.deprecated
})
```

---

## Adding a New Provider

1. **Add to BUNDLED_PROVIDERS** or external SDK package
2. **Create custom loader** in `CUSTOM_LOADERS` for setup logic
3. **Define model list** in `models.dev` database
4. **Add transformations** in `ProviderTransform` if needed
5. **Leverage existing adapters** or build custom ones

```typescript
// Example custom loader
CUSTOM_LOADERS["my-provider"] = async (provider, config) => {
  const apiKey = process.env.MY_PROVIDER_API_KEY
  if (!apiKey) return undefined

  return {
    ...provider,
    options: { apiKey },
    models: await fetchModels(apiKey),
  }
}
```

---

## Key Design Patterns

| Pattern | Purpose |
|---------|---------|
| **Namespace Pattern** | `Provider` namespace groups all logic |
| **State Management** | `Instance.state()` for singleton caching |
| **Custom Loaders** | Provider-specific setup as first-class citizens |
| **Transformation Pipeline** | Composable message/schema transforms |
| **Zod Schemas** | Type-safe provider/model definitions |
| **Lazy Loading** | SDK only instantiated when used |

---

## Key Files

| File | Purpose |
|------|---------|
| `src/provider/provider.ts` | Core provider logic (~1000 lines) |
| `src/provider/transform.ts` | Message transformations |
| `src/provider/auth.ts` | Authentication management |
| `src/session/llm.ts` | LLM streaming integration |

---

*Written by Claude (Opus 4.5) | 2026-01-22 16:00 PST*
