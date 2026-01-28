# OpenCode Util Package Deep Dive

[TOC]

A comprehensive study of the shared utility functions used across the OpenCode monorepo.

---

## Overview

The Util package (`@opencode-ai/util`) provides **10 core utility modules** used throughout the codebase. It's schema-first with Zod at its core.

**Key Features:**
- Type-safe error handling
- Schema-validated functions
- Monotonic ID generation
- Retry with exponential backoff
- Zero external dependencies (except Zod)

---

## Utility Catalog

| Module | Purpose | Key Export |
|--------|---------|------------|
| `error.ts` | Type-safe errors | `NamedError` |
| `fn.ts` | Validated functions | `fn()` |
| `retry.ts` | Async retry | `retry()` |
| `lazy.ts` | Lazy evaluation | `lazy()` |
| `identifier.ts` | ID generation | `Identifier` |
| `slug.ts` | Random slugs | `Slug` |
| `encode.ts` | Encoding/hashing | `base64Encode`, `hash` |
| `path.ts` | Path parsing | `getFilename`, etc. |
| `binary.ts` | Binary search | `Binary.search()` |
| `iife.ts` | Immediate execution | `iife()` |

---

## Error Handling

### NamedError Pattern

```typescript
import { NamedError } from "@opencode-ai/util/error"

// Define typed error
export const InvalidError = NamedError.create(
  "InvalidError",
  z.object({
    path: z.string(),
    message: z.string().optional(),
  })
)

// Throw error
throw new InvalidError({ path: "/foo", message: "Not found" })

// Type-safe checking
if (InvalidError.isInstance(err)) {
  console.log(err.data.path)  // Typed!
}

// Serialize
error.toObject()  // { name: "InvalidError", data: { ... } }
```

### Features

- `.isInstance()` type guard
- `.toObject()` serialization
- `.Schema` for Zod validation
- Discriminated unions via `NamedError.Unknown`

### Usage Across Codebase

20+ custom error types:
- `Message.OutputLengthError`
- `Storage.NotFoundError`
- `Provider.ModelNotFoundError`
- `Worktree.NotGitError`

---

## Schema-Validated Functions

### Pattern

```typescript
import { fn } from "@opencode-ai/util/fn"

export const create = fn(
  z.object({
    sessionID: z.string(),
    title: z.string().optional(),
  }),
  async (input) => {
    // input is validated and typed
    return { id: input.sessionID }
  }
)

// Usage
await create({ sessionID: "abc" })

// Skip validation (trusted input)
await create.force({ sessionID: "abc" })

// Access schema
create.schema  // Zod schema
```

### Used In

- `Session.create`, `Session.fork`, `Session.share`
- `Share.create`, `Share.sync`
- `Permission.create`

---

## Identifier Generation

### Monotonic IDs

```typescript
import { Identifier } from "@opencode-ai/util/identifier"

// Ascending (chronological order)
const msgId = Identifier.ascending("message")
// → "msg_abc123def456789..."

// Descending (reverse chronological)
const eventId = Identifier.descending("session")
// → "ses_xyz987..."

// Extract timestamp
const ts = Identifier.timestamp(msgId)

// Schema validation
Identifier.schema("session")  // z.string().startsWith("ses_")
```

### Built-in Prefixes

```typescript
const prefixes = {
  session: "ses",
  message: "msg",
  permission: "per",
  question: "que",
  user: "usr",
  part: "prt",
  pty: "pty",
  tool: "tool",
}
```

### Features

- 26-character base62 IDs
- 6-byte timestamp (millisecond precision)
- Monotonic counter (same-ms collisions)
- Sortable ascending/descending

---

## Retry with Backoff

```typescript
import { retry } from "@opencode-ai/util/retry"

const result = await retry(
  async () => {
    return fetch(url)
  },
  {
    attempts: 3,
    delay: 500,      // Initial delay
    factor: 2,       // Exponential factor
    maxDelay: 10000, // Cap
    retryIf: (err) => isTransient(err),
  }
)

// Backoff: 500ms → 1000ms → 2000ms
```

### Transient Error Detection

```typescript
const TRANSIENT_MESSAGES = [
  "load failed",
  "network connection was lost",
  "failed to fetch",
  "econnreset",
  "etimedout",
  "socket hang up",
]
```

---

## Lazy Evaluation

```typescript
import { lazy } from "@opencode-ai/util/lazy"

export const Routes = lazy(() =>
  new Hono()
    .get("/", handler)
    .post("/", handler)
)

// First call: executes, memoizes
// Subsequent: returns cached value
```

---

## Encoding Utilities

```typescript
import { base64Encode, base64Decode, hash, checksum } from "@opencode-ai/util/encode"

// URL-safe Base64 (RFC 4648)
base64Encode("hello")  // No padding, +/ → -_
base64Decode(encoded)

// Cryptographic hash
await hash(content, "SHA-256")

// Fast checksum (FNV-1a)
checksum(content)  // For cache keys
```

---

## Slug Generation

```typescript
import { Slug } from "@opencode-ai/util/slug"

Slug.create()  // "brave-cabin", "clever-rocket"

// 30 adjectives × 31 nouns = ~930 combinations
```

---

## Path Utilities

```typescript
import { getFilename, getDirectory, getFileExtension } from "@opencode-ai/util/path"

getFilename("/dir/file.txt")      // "file.txt"
getDirectory("/dir/file.txt")     // "/dir/"
getFileExtension("file.txt")      // "txt"

// Handles both "/" and "\" separators
```

---

## Binary Search

```typescript
import { Binary } from "@opencode-ai/util/binary"

// Search sorted array
const result = Binary.search(
  array,
  targetId,
  (item) => item.id  // Compare function
)
// → { found: boolean, index: number }

// Insert maintaining sort order
const newArray = Binary.insert(array, newItem, (item) => item.id)
```

### Used In

Share data compaction for efficient deduplication.

---

## IIFE Helper

```typescript
import { iife } from "@opencode-ai/util/iife"

// Immediately-invoked async function
const promises = items.map(item =>
  iife(async () => {
    await Storage.write(key, item)
  })
)
await Promise.all(promises)
```

---

## Design Philosophy

### Schema-First

All utilities integrate with Zod:
- Errors have `.Schema`
- Functions have `.schema`
- IDs have `Identifier.schema()`

### Type Safety

```typescript
// Type inference from schema
export const Info = z.object({
  id: Identifier.schema("session"),
  slug: z.string(),
})
export type Info = z.infer<typeof Info>
```

### Minimal Dependencies

Only Zod required. No lodash, no utility libraries.

---

## Package Usage

```typescript
// Error handling
import { NamedError } from "@opencode-ai/util/error"

// Validated functions
import { fn } from "@opencode-ai/util/fn"

// IDs
import { Identifier } from "@opencode-ai/util/identifier"

// Retry
import { retry } from "@opencode-ai/util/retry"

// Encoding
import { base64Encode, hash } from "@opencode-ai/util/encode"
```

---

## Usage Statistics

| Utility | Usage Count | Primary Users |
|---------|-------------|---------------|
| `NamedError` | 20+ types | Core, Enterprise |
| `fn()` | 40+ functions | Session, Share, Permission |
| `Identifier` | 50+ places | All packages |
| `retry()` | Session processor | Core |
| `Binary.search` | 1 | Share compaction |

---

## Key Files

| File | Purpose |
|------|---------|
| `src/error.ts` | NamedError factory |
| `src/fn.ts` | Schema-validated functions |
| `src/identifier.ts` | ID generation |
| `src/retry.ts` | Retry with backoff |
| `src/encode.ts` | Encoding utilities |

---

*Written by Claude (Opus 4.5) | 2026-01-22 17:00 PST*
