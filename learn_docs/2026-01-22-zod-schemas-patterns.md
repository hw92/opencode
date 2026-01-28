# Zod Schema Patterns in OpenCode

This document provides a deep-dive into how the OpenCode codebase uses Zod schemas to define data models, enable type safety, and validate data throughout the application.

## Table of Contents

1. [Core Zod Patterns](#core-zod-patterns)
2. [Type Inference Patterns](#type-inference-patterns)
3. [Validation Patterns](#validation-patterns)
4. [Schema Composition Patterns](#schema-composition-patterns)
5. [Custom Validators and Extensions](#custom-validators-and-extensions)
6. [Real-World Examples](#real-world-examples)
7. [Best Practices](#best-practices)

---

## Core Zod Patterns

### Pattern 1: Namespace + Schema + Type Export

OpenCode follows a consistent pattern of organizing related schemas within TypeScript namespaces, exporting both the schema and its inferred type with the same name.

**File:** `packages/opencode/src/session/index.ts:42-83`

```typescript
export namespace Session {
  export const Info = z
    .object({
      id: Identifier.schema("session"),
      slug: z.string(),
      projectID: z.string(),
      directory: z.string(),
      parentID: Identifier.schema("session").optional(),
      summary: z
        .object({
          additions: z.number(),
          deletions: z.number(),
          files: z.number(),
          diffs: Snapshot.FileDiff.array().optional(),
        })
        .optional(),
      title: z.string(),
      version: z.string(),
      time: z.object({
        created: z.number(),
        updated: z.number(),
        compacting: z.number().optional(),
        archived: z.number().optional(),
      }),
    })
    .meta({
      ref: "Session",
    })
  export type Info = z.output<typeof Info>
}
```

**Benefits:**
- Schema and type share the same name, making imports intuitive
- Namespaces group related schemas together
- The `.meta({ ref: "..." })` pattern enables OpenAPI schema generation

---

### Pattern 2: Schema Metadata with `.meta()`

OpenCode extensively uses `.meta({ ref: "..." })` to annotate schemas for API documentation and OpenAPI generation.

**File:** `packages/opencode/src/session/message.ts:14-25`

```typescript
export const ToolCall = z
  .object({
    state: z.literal("call"),
    step: z.number().optional(),
    toolCallId: z.string(),
    toolName: z.string(),
    args: z.custom<Required<unknown>>(),
  })
  .meta({
    ref: "ToolCall",
  })
export type ToolCall = z.infer<typeof ToolCall>
```

The `ref` metadata provides a reference name used when generating OpenAPI schemas for the REST API.

---

## Type Inference Patterns

### Pattern 1: Direct Type Inference with `z.infer`

The most common pattern extracts a TypeScript type from a Zod schema.

**File:** `packages/opencode/src/provider/provider.ts:503-572`

```typescript
export const Model = z
  .object({
    id: z.string(),
    providerID: z.string(),
    api: z.object({
      id: z.string(),
      url: z.string(),
      npm: z.string(),
    }),
    name: z.string(),
    capabilities: z.object({
      temperature: z.boolean(),
      reasoning: z.boolean(),
      // ...more fields
    }),
    // ...more fields
  })
  .meta({
    ref: "Model",
  })
export type Model = z.infer<typeof Model>
```

### Pattern 2: `z.output` vs `z.infer`

When schemas have transforms, `z.output` ensures you get the transformed type.

**File:** `packages/opencode/src/config/config.ts:1076-1077`

```typescript
export const Info = z.object({ /* ... */ }).strict().meta({ ref: "Config" })
export type Info = z.output<typeof Info>
```

Using `z.output` is important when the schema includes `.transform()` methods, as it gives you the type after transformation rather than the input type.

### Pattern 3: Schema Property Access with `.shape`

Access individual fields from a schema using `.shape` for reuse.

**File:** `packages/opencode/src/session/index.ts:134-136`

```typescript
export const create = fn(
  z.object({
    parentID: Identifier.schema("session").optional(),
    title: z.string().optional(),
    permission: Info.shape.permission,  // Reuse field from Info schema
  }).optional(),
  async (input) => { /* ... */ }
)
```

### Pattern 4: Generic Type Inference from Tools

**File:** `packages/opencode/src/tool/tool.ts:44-45`

```typescript
export type InferParameters<T extends Info> = T extends Info<infer P> ? z.infer<P> : never
export type InferMetadata<T extends Info> = T extends Info<any, infer M> ? M : never
```

This advanced pattern extracts type parameters from generic tool definitions.

---

## Validation Patterns

### Pattern 1: The `fn()` Helper for Runtime Validation

OpenCode uses a custom `fn()` helper that wraps functions with Zod validation.

**File:** `packages/opencode/src/util/fn.ts:1-11`

```typescript
import { z } from "zod"

export function fn<T extends z.ZodType, Result>(schema: T, cb: (input: z.infer<T>) => Result) {
  const result = (input: z.infer<T>) => {
    const parsed = schema.parse(input)  // Validates at runtime
    return cb(parsed)
  }
  result.force = (input: z.infer<T>) => cb(input)  // Skip validation
  result.schema = schema  // Expose schema for introspection
  return result
}
```

**Usage throughout the codebase:**

**File:** `packages/opencode/src/session/index.ts:186-190`

```typescript
export const touch = fn(Identifier.schema("session"), async (sessionID) => {
  await update(sessionID, (draft) => {
    draft.time.updated = Date.now()
  })
})
```

**File:** `packages/opencode/src/session/index.ts:292-306`

```typescript
export const messages = fn(
  z.object({
    sessionID: Identifier.schema("session"),
    limit: z.number().optional(),
  }),
  async (input) => {
    const result = [] as MessageV2.WithParts[]
    for await (const msg of MessageV2.stream(input.sessionID)) {
      if (input.limit && result.length >= input.limit) break
      result.push(msg)
    }
    result.reverse()
    return result
  },
)
```

**Benefits of `fn()`:**
- Automatic runtime validation
- Exposes `.schema` property for API documentation
- Provides `.force()` for bypassing validation when needed
- Enables type-safe function composition

### Pattern 2: Safe Parsing with `.safeParse()`

For cases where validation might fail gracefully:

**File:** `packages/opencode/src/config/config.ts:1183-1204`

```typescript
const parsed = Info.safeParse(data)
if (parsed.success) {
  if (!parsed.data.$schema) {
    parsed.data.$schema = "https://opencode.ai/config.json"
    await Bun.write(configFilepath, JSON.stringify(parsed.data, null, 2)).catch(() => {})
  }
  return parsed.data
}

throw new InvalidError({
  path: configFilepath,
  issues: parsed.error.issues,
})
```

### Pattern 3: API Route Validation with Hono

**File:** `packages/opencode/src/server/routes/session.ts:41-52`

```typescript
validator(
  "query",
  z.object({
    directory: z.string().optional().meta({ description: "Filter sessions by project directory" }),
    roots: z.coerce.boolean().optional().meta({ description: "Only return root sessions" }),
    start: z.coerce.number().optional().meta({ description: "Filter sessions updated after timestamp" }),
    search: z.string().optional().meta({ description: "Filter sessions by title" }),
    limit: z.coerce.number().optional().meta({ description: "Maximum number of sessions" }),
  }),
),
```

The `z.coerce` modifier automatically converts string query parameters to the appropriate type.

### Pattern 4: Tool Parameter Validation

**File:** `packages/opencode/src/tool/bash.ts:62-76`

```typescript
parameters: z.object({
  command: z.string().describe("The command to execute"),
  timeout: z.number().describe("Optional timeout in milliseconds").optional(),
  workdir: z.string()
    .describe(`The working directory to run the command in. Defaults to ${Instance.directory}.`)
    .optional(),
  description: z.string()
    .describe("Clear, concise description of what this command does in 5-10 words."),
}),
```

The `.describe()` method provides documentation that's included in tool schemas sent to LLMs.

---

## Schema Composition Patterns

### Pattern 1: Base Schema Extension with `.extend()`

Build complex schemas by extending base schemas.

**File:** `packages/opencode/src/session/message-v2.ts:38-75`

```typescript
const PartBase = z.object({
  id: z.string(),
  sessionID: z.string(),
  messageID: z.string(),
})

export const SnapshotPart = PartBase.extend({
  type: z.literal("snapshot"),
  snapshot: z.string(),
}).meta({ ref: "SnapshotPart" })
export type SnapshotPart = z.infer<typeof SnapshotPart>

export const TextPart = PartBase.extend({
  type: z.literal("text"),
  text: z.string(),
  synthetic: z.boolean().optional(),
  ignored: z.boolean().optional(),
  time: z.object({
    start: z.number(),
    end: z.number().optional(),
  }).optional(),
  metadata: z.record(z.string(), z.any()).optional(),
}).meta({ ref: "TextPart" })
export type TextPart = z.infer<typeof TextPart>
```

### Pattern 2: Discriminated Unions

Create type-safe unions with a discriminator field.

**File:** `packages/opencode/src/session/message-v2.ts:329-347`

```typescript
export const Part = z
  .discriminatedUnion("type", [
    TextPart,
    SubtaskPart,
    ReasoningPart,
    FilePart,
    ToolPart,
    StepStartPart,
    StepFinishPart,
    SnapshotPart,
    PatchPart,
    AgentPart,
    RetryPart,
    CompactionPart,
  ])
  .meta({ ref: "Part" })
export type Part = z.infer<typeof Part>
```

**File:** `packages/opencode/src/session/message-v2.ts:393-396`

```typescript
export const Info = z.discriminatedUnion("role", [User, Assistant]).meta({
  ref: "Message",
})
export type Info = z.infer<typeof Info>
```

### Pattern 3: Partial Schemas with `.partial()` and `.omit()`

**File:** `packages/opencode/src/session/prompt.ts:104-144`

```typescript
parts: z.array(
  z.discriminatedUnion("type", [
    MessageV2.TextPart.omit({
      messageID: true,
      sessionID: true,
    })
      .partial({
        id: true,
      })
      .meta({
        ref: "TextPartInput",
      }),
    // ...more variants
  ]),
),
```

### Pattern 4: Schema Merging with `catchall()`

Allow additional properties beyond defined fields.

**File:** `packages/opencode/src/config/config.ts:548-577`

```typescript
export const Agent = z
  .object({
    model: z.string().optional(),
    temperature: z.number().optional(),
    prompt: z.string().optional(),
    // ...defined fields
  })
  .catchall(z.any())  // Accept any additional properties
  .transform((agent, ctx) => {
    // Transform unknown properties into options
    const knownKeys = new Set(["model", "prompt", /* ... */])
    const options: Record<string, unknown> = { ...agent.options }
    for (const [key, value] of Object.entries(agent)) {
      if (!knownKeys.has(key)) options[key] = value
    }
    return { ...agent, options }
  })
```

### Pattern 5: Schema Arrays

**File:** `packages/opencode/src/permission/next.ts:31-34`

```typescript
export const Ruleset = Rule.array().meta({
  ref: "PermissionRuleset",
})
export type Ruleset = z.infer<typeof Ruleset>
```

---

## Custom Validators and Extensions

### Pattern 1: Custom Identifier Schema Factory

**File:** `packages/opencode/src/id/id.ts:16-18`

```typescript
export function schema(prefix: keyof typeof prefixes) {
  return z.string().startsWith(prefixes[prefix])
}
```

This creates reusable ID validators with prefix checking.

**Usage:**

```typescript
sessionID: Identifier.schema("session"),  // Must start with "ses_"
messageID: Identifier.schema("message"),  // Must start with "msg_"
```

### Pattern 2: NamedError Factory with Zod

A powerful pattern for creating typed error classes with Zod schemas.

**File:** `packages/util/src/error.ts:1-54`

```typescript
export abstract class NamedError extends Error {
  abstract schema(): z.core.$ZodType
  abstract toObject(): { name: string; data: any }

  static create<Name extends string, Data extends z.core.$ZodType>(name: Name, data: Data) {
    const schema = z
      .object({
        name: z.literal(name),
        data,
      })
      .meta({ ref: name })

    const result = class extends NamedError {
      public static readonly Schema = schema
      public override readonly name = name as Name

      constructor(
        public readonly data: z.input<Data>,
        options?: ErrorOptions,
      ) {
        super(name, options)
        this.name = name
      }

      static isInstance(input: any): input is InstanceType<typeof result> {
        return typeof input === "object" && "name" in input && input.name === name
      }

      toObject() {
        return { name: name, data: this.data }
      }
    }
    return result
  }
}
```

**Usage:**

**File:** `packages/opencode/src/session/message-v2.ts:16-35`

```typescript
export const OutputLengthError = NamedError.create("MessageOutputLengthError", z.object({}))
export const AbortedError = NamedError.create("MessageAbortedError", z.object({ message: z.string() }))
export const AuthError = NamedError.create(
  "ProviderAuthError",
  z.object({
    providerID: z.string(),
    message: z.string(),
  }),
)
```

This pattern provides:
- Type-safe error data
- Static `.isInstance()` type guard
- Serializable `.toObject()` for API responses
- Auto-generated OpenAPI schemas via `.Schema`

### Pattern 3: Preprocess and Transform

**File:** `packages/opencode/src/config/config.ts:488-536`

```typescript
const permissionPreprocess = (val: unknown) => {
  if (typeof val === "object" && val !== null && !Array.isArray(val)) {
    return { __originalKeys: Object.keys(val), ...val }
  }
  return val
}

const permissionTransform = (x: unknown): Record<string, PermissionRule> => {
  if (typeof x === "string") return { "*": x as PermissionAction }
  const obj = x as { __originalKeys?: string[] } & Record<string, unknown>
  const { __originalKeys, ...rest } = obj
  if (!__originalKeys) return rest as Record<string, PermissionRule>
  const result: Record<string, PermissionRule> = {}
  for (const key of __originalKeys) {
    if (key in rest) result[key] = rest[key] as PermissionRule
  }
  return result
}

export const Permission = z
  .preprocess(
    permissionPreprocess,
    z.object({
      __originalKeys: z.string().array().optional(),
      read: PermissionRule.optional(),
      edit: PermissionRule.optional(),
      // ...more fields
    })
    .catchall(PermissionRule)
    .or(PermissionAction),
  )
  .transform(permissionTransform)
```

This preserves key ordering through the parse/transform cycle.

### Pattern 4: Refinements with `.refine()`

**File:** `packages/opencode/src/config/config.ts:996-1011`

```typescript
lsp: z
  .union([
    z.literal(false),
    z.record(z.string(), z.union([/* ... */])),
  ])
  .optional()
  .refine(
    (data) => {
      if (!data) return true
      if (typeof data === "boolean") return true
      const serverIds = new Set(Object.values(LSPServer).map((s) => s.id))

      return Object.entries(data).every(([id, config]) => {
        if (config.disabled) return true
        if (serverIds.has(id)) return true
        return Boolean(config.extensions)
      })
    },
    {
      error: "For custom LSP servers, 'extensions' array is required.",
    },
  ),
```

### Pattern 5: Event Definition with BusEvent

**File:** `packages/opencode/src/bus/bus-event.ts:12-19`

```typescript
export function define<Type extends string, Properties extends ZodType>(type: Type, properties: Properties) {
  const result = {
    type,
    properties,
  }
  registry.set(type, result)
  return result
}
```

**Usage:**

**File:** `packages/opencode/src/session/index.ts:95-128`

```typescript
export const Event = {
  Created: BusEvent.define(
    "session.created",
    z.object({ info: Info }),
  ),
  Updated: BusEvent.define(
    "session.updated",
    z.object({ info: Info }),
  ),
  Deleted: BusEvent.define(
    "session.deleted",
    z.object({ info: Info }),
  ),
  Diff: BusEvent.define(
    "session.diff",
    z.object({
      sessionID: z.string(),
      diff: Snapshot.FileDiff.array(),
    }),
  ),
}
```

---

## Real-World Examples

### Example 1: Complete Message Schema Hierarchy

**File:** `packages/opencode/src/session/message-v2.ts`

The message system demonstrates a complete schema hierarchy:

1. **Base schemas** define common fields
2. **Part types** extend the base with specific fields and literal type discriminators
3. **Discriminated unions** combine all parts into a single type
4. **Message types** (User, Assistant) include parts and metadata
5. **Final union** combines message types by role

```
PartBase (base)
  ├── TextPart (type: "text")
  ├── FilePart (type: "file")
  ├── ToolPart (type: "tool")
  └── ...more part types

Base (base for messages)
  ├── User (role: "user")
  └── Assistant (role: "assistant")

Info = discriminatedUnion([User, Assistant])
WithParts = { info: Info, parts: Part[] }
```

### Example 2: Tool State Machine

**File:** `packages/opencode/src/session/message-v2.ts:220-286`

```typescript
export const ToolStatePending = z.object({
  status: z.literal("pending"),
  input: z.record(z.string(), z.any()),
  raw: z.string(),
})

export const ToolStateRunning = z.object({
  status: z.literal("running"),
  input: z.record(z.string(), z.any()),
  title: z.string().optional(),
  time: z.object({ start: z.number() }),
})

export const ToolStateCompleted = z.object({
  status: z.literal("completed"),
  input: z.record(z.string(), z.any()),
  output: z.string(),
  title: z.string(),
  time: z.object({
    start: z.number(),
    end: z.number(),
    compacted: z.number().optional(),
  }),
})

export const ToolStateError = z.object({
  status: z.literal("error"),
  input: z.record(z.string(), z.any()),
  error: z.string(),
  time: z.object({ start: z.number(), end: z.number() }),
})

export const ToolState = z.discriminatedUnion("status", [
  ToolStatePending,
  ToolStateRunning,
  ToolStateCompleted,
  ToolStateError,
])
```

### Example 3: Config Schema with Strict Mode

**File:** `packages/opencode/src/config/config.ts:865-1075`

```typescript
export const Info = z
  .object({
    $schema: z.string().optional(),
    theme: z.string().optional(),
    keybinds: Keybinds.optional(),
    // ...many more fields
  })
  .strict()  // Reject unknown properties
  .meta({ ref: "Config" })
```

The `.strict()` modifier ensures configuration files don't contain typos or unknown fields.

---

## Best Practices

### 1. Always Export Both Schema and Type

```typescript
export const MySchema = z.object({ /* ... */ })
export type MySchema = z.infer<typeof MySchema>
```

### 2. Use `.meta({ ref: "..." })` for API Schemas

This enables automatic OpenAPI generation.

### 3. Prefer Discriminated Unions Over Regular Unions

Discriminated unions provide better type narrowing and error messages.

### 4. Use the `fn()` Helper for Validated Functions

```typescript
export const myFunction = fn(
  z.object({ input: z.string() }),
  async (params) => { /* ... */ }
)
```

### 5. Leverage `.shape` for Field Reuse

```typescript
const newSchema = z.object({
  existingField: ExistingSchema.shape.field,
  newField: z.string(),
})
```

### 6. Use `.safeParse()` When Validation Might Fail

```typescript
const result = schema.safeParse(data)
if (result.success) {
  // Use result.data
} else {
  // Handle result.error
}
```

### 7. Document Tool Parameters with `.describe()`

```typescript
z.object({
  command: z.string().describe("The shell command to execute"),
  timeout: z.number().optional().describe("Timeout in milliseconds"),
})
```

### 8. Use `z.coerce` for Query Parameters

```typescript
validator("query", z.object({
  limit: z.coerce.number().optional(),
  active: z.coerce.boolean().optional(),
}))
```

### 9. Create Reusable Schema Factories

```typescript
// Instead of repeating patterns
function createIdSchema(prefix: string) {
  return z.string().startsWith(prefix)
}
```

### 10. Use NamedError for Type-Safe Errors

```typescript
export const MyError = NamedError.create(
  "MyErrorName",
  z.object({
    field1: z.string(),
    field2: z.number(),
  })
)
```

---

## Summary

OpenCode's use of Zod provides:

1. **Runtime type safety** - Validation at boundaries (API, functions, storage)
2. **Static type inference** - TypeScript types derived from schemas
3. **Documentation** - Self-documenting schemas with `.describe()` and `.meta()`
4. **API generation** - OpenAPI schemas from Zod definitions
5. **Composability** - Building complex schemas from simple primitives
6. **Error handling** - Type-safe errors with NamedError pattern

The consistent patterns throughout the codebase make it easy to understand data shapes and maintain type safety across the entire application.

---

*Written by Claude (Opus 4.5) | 2026-01-22 PST*
