# OpenCode File Operations System - Deep Dive

This document provides a comprehensive analysis of OpenCode's file operations system, covering the core file module and the tool implementations for reading, writing, editing, globbing, and grepping files.

## Architecture Overview

```
+------------------------------------------------------------------+
|                        Tool Registry                              |
|  (packages/opencode/src/tool/registry.ts)                        |
|  - Registers all tools: Read, Write, Edit, Glob, Grep            |
|  - Manages custom/plugin tools                                    |
+------------------------------------------------------------------+
                              |
                              v
+------------------------------------------------------------------+
|                     Tool.define() Pattern                         |
|  (packages/opencode/src/tool/tool.ts)                            |
|  - Validates parameters with Zod                                  |
|  - Auto-truncates output via Truncate.output()                   |
|  - Wraps execute() with error handling                           |
+------------------------------------------------------------------+
                              |
          +-------------------+-------------------+
          |                   |                   |
          v                   v                   v
+------------------+  +------------------+  +------------------+
|    Read Tool     |  |   Write Tool     |  |    Edit Tool     |
| (read.ts)        |  | (write.ts)       |  | (edit.ts)        |
| - Line limits    |  | - Diff preview   |  | - String replace |
| - Binary detect  |  | - LSP errors     |  | - Fuzzy matching |
| - Image/PDF      |  | - File events    |  | - Multiple       |
+------------------+  +------------------+  |   replacers      |
                                           +------------------+
          +-------------------+-------------------+
          |                                       |
          v                                       v
+------------------+                   +------------------+
|   Glob Tool      |                   |   Grep Tool      |
| (glob.ts)        |                   | (grep.ts)        |
| - Pattern match  |                   | - Regex search   |
| - Uses Ripgrep   |                   | - Uses Ripgrep   |
| - Sorted by mtime|                   | - Line truncation|
+------------------+                   +------------------+
          |                                       |
          +-------------------+-------------------+
                              |
                              v
+------------------------------------------------------------------+
|                      File Module                                  |
|  (packages/opencode/src/file/)                                   |
|  +-------------+  +-------------+  +-------------+  +----------+ |
|  | index.ts    |  | ripgrep.ts  |  | time.ts     |  | ignore.ts| |
|  | - File.read |  | - rg wrapper|  | - Read times|  | - Ignore | |
|  | - File.list |  | - Auto-dl   |  | - Write lock|  |   rules  | |
|  | - File.search| | - JSON parse|  | - Staleness |  +----------+ |
|  +-------------+  +-------------+  +-------------+  +----------+ |
|                                                     | watcher.ts||
|                                                     | - Parcel  | |
|                                                     |   watcher | |
|                                                     +----------+ |
+------------------------------------------------------------------+
                              |
                              v
+------------------------------------------------------------------+
|                     Output Truncation                             |
|  (packages/opencode/src/tool/truncation.ts)                      |
|  - MAX_LINES: 2000, MAX_BYTES: 50KB                              |
|  - Saves full output to disk when truncated                       |
|  - Provides hints for using Task/Grep/Read with offset           |
+------------------------------------------------------------------+
```

## Key Files and Their Purposes

### File Module (`packages/opencode/src/file/`)

| File | Purpose |
|------|---------|
| `index.ts` | Core File namespace with read, list, search, and status functions |
| `ripgrep.ts` | Ripgrep wrapper - auto-downloads binary, provides files() and search() generators |
| `time.ts` | FileTime namespace - tracks read times per session, provides write locks, staleness checks |
| `ignore.ts` | FileIgnore namespace - default ignore patterns for folders and files |
| `watcher.ts` | FileWatcher namespace - Parcel watcher integration for file change events |

### Tool Module (`packages/opencode/src/tool/`)

| File | Purpose |
|------|---------|
| `tool.ts` | Tool.define() factory - creates tools with validation and truncation |
| `read.ts` | ReadTool - reads files with line limits, handles images/PDFs/binary detection |
| `write.ts` | WriteTool - writes files with diff preview, LSP diagnostics integration |
| `edit.ts` | EditTool - string replacement with multiple fuzzy matching strategies |
| `glob.ts` | GlobTool - file pattern matching using ripgrep's --files mode |
| `grep.ts` | GrepTool - content search using ripgrep with regex support |
| `truncation.ts` | Truncate namespace - handles output size limits and file persistence |

---

## Tool Definition Pattern with Tool.define()

OpenCode uses a consistent pattern for defining tools via `Tool.define()`. This factory function provides:

1. **ID Assignment** - Each tool gets a unique identifier
2. **Parameter Validation** - Zod schema validation with custom error formatting
3. **Automatic Truncation** - Output is truncated if it exceeds limits
4. **Metadata Handling** - Title, metadata, and output structure

### Core Tool.define() Implementation

```typescript
// From packages/opencode/src/tool/tool.ts
export function define<Parameters extends z.ZodType, Result extends Metadata>(
  id: string,
  init: Info<Parameters, Result>["init"] | Awaited<ReturnType<Info<Parameters, Result>["init"]>>,
): Info<Parameters, Result> {
  return {
    id,
    init: async (initCtx) => {
      const toolInfo = init instanceof Function ? await init(initCtx) : init
      const execute = toolInfo.execute

      // Wrap execute with validation and truncation
      toolInfo.execute = async (args, ctx) => {
        try {
          toolInfo.parameters.parse(args)  // Zod validation
        } catch (error) {
          if (error instanceof z.ZodError && toolInfo.formatValidationError) {
            throw new Error(toolInfo.formatValidationError(error), { cause: error })
          }
          throw new Error(
            `The ${id} tool was called with invalid arguments: ${error}.\nPlease rewrite the input so it satisfies the expected schema.`,
            { cause: error },
          )
        }

        const result = await execute(args, ctx)

        // Skip truncation for tools that handle it themselves
        if (result.metadata.truncated !== undefined) {
          return result
        }

        // Auto-truncate output
        const truncated = await Truncate.output(result.output, {}, initCtx?.agent)
        return {
          ...result,
          output: truncated.content,
          metadata: {
            ...result.metadata,
            truncated: truncated.truncated,
            ...(truncated.truncated && { outputPath: truncated.outputPath }),
          },
        }
      }
      return toolInfo
    },
  }
}
```

### Tool Context Interface

Each tool's execute function receives a context object:

```typescript
export type Context<M extends Metadata = Metadata> = {
  sessionID: string
  messageID: string
  agent: string
  abort: AbortSignal
  callID?: string
  extra?: { [key: string]: any }
  metadata(input: { title?: string; metadata?: M }): void
  ask(input: Omit<PermissionNext.Request, "id" | "sessionID" | "tool">): Promise<void>
}
```

---

## Read Tool

The Read tool (`read.ts`) handles file reading with intelligent constraints.

### Configuration Constants

```typescript
const DEFAULT_READ_LIMIT = 2000    // Default lines to read
const MAX_LINE_LENGTH = 2000      // Truncate lines longer than this
const MAX_BYTES = 50 * 1024       // 50KB max output
```

### Zod Schema

```typescript
parameters: z.object({
  filePath: z.string().describe("The path to the file to read"),
  offset: z.coerce.number().describe("The line number to start reading from (0-based)").optional(),
  limit: z.coerce.number().describe("The number of lines to read (defaults to 2000)").optional(),
})
```

### Key Features

1. **Line Number Formatting**: Output uses `cat -n` style (`00001| content`)
2. **Binary Detection**: Checks file extension and content for binary files
3. **Image/PDF Handling**: Returns base64-encoded attachments for visual files
4. **Byte Limit**: Stops reading when 50KB is reached
5. **Line Truncation**: Lines over 2000 chars get truncated with `...`

### Binary Detection Logic

```typescript
async function isBinaryFile(filepath: string, file: Bun.BunFile): Promise<boolean> {
  // Check known binary extensions
  const ext = path.extname(filepath).toLowerCase()
  const binaryExtensions = [
    ".zip", ".tar", ".gz", ".exe", ".dll", ".so", ".class",
    ".jar", ".war", ".7z", ".doc", ".docx", ".xls", ".xlsx",
    ".ppt", ".pptx", ".odt", ".ods", ".odp", ".bin", ".dat",
    ".obj", ".o", ".a", ".lib", ".wasm", ".pyc", ".pyo"
  ]
  if (binaryExtensions.includes(ext)) return true

  // Read first 4KB and check for null bytes or >30% non-printable
  const bytes = new Uint8Array(buffer.slice(0, 4096))
  let nonPrintableCount = 0
  for (let i = 0; i < bytes.length; i++) {
    if (bytes[i] === 0) return true  // Null byte = binary
    if (bytes[i] < 9 || (bytes[i] > 13 && bytes[i] < 32)) {
      nonPrintableCount++
    }
  }
  return nonPrintableCount / bytes.length > 0.3
}
```

### Output Format Example

```
<file>
00001| import z from "zod"
00002| import * as fs from "fs"
00003| import * as path from "path"

(File has more lines. Use 'offset' parameter to read beyond line 2000)
</file>
```

---

## Write Tool

The Write tool (`write.ts`) creates or overwrites files with diff preview.

### Zod Schema

```typescript
parameters: z.object({
  content: z.string().describe("The content to write to the file"),
  filePath: z.string().describe("The absolute path to the file to write"),
})
```

### Key Features

1. **Staleness Check**: Uses `FileTime.assert()` to ensure file wasn't modified since last read
2. **Diff Preview**: Shows unified diff before writing for permission approval
3. **LSP Integration**: Reports diagnostics (errors) after writing
4. **Event Publishing**: Emits `File.Event.Edited` for other systems to react

### Workflow

```typescript
async execute(params, ctx) {
  // 1. Resolve path and check external directory permissions
  const filepath = path.isAbsolute(params.filePath)
    ? params.filePath
    : path.join(Instance.directory, params.filePath)
  await assertExternalDirectory(ctx, filepath)

  // 2. Check if file was read before (staleness protection)
  const file = Bun.file(filepath)
  const exists = await file.exists()
  if (exists) await FileTime.assert(ctx.sessionID, filepath)

  // 3. Generate diff for permission request
  const contentOld = exists ? await file.text() : ""
  const diff = trimDiff(createTwoFilesPatch(filepath, filepath, contentOld, params.content))

  // 4. Ask for permission with diff preview
  await ctx.ask({
    permission: "edit",
    patterns: [path.relative(Instance.worktree, filepath)],
    metadata: { filepath, diff },
  })

  // 5. Write file and publish event
  await Bun.write(filepath, params.content)
  await Bus.publish(File.Event.Edited, { file: filepath })
  FileTime.read(ctx.sessionID, filepath)

  // 6. Return with LSP diagnostics
  await LSP.touchFile(filepath, true)
  const diagnostics = await LSP.diagnostics()
  // ... format diagnostics output
}
```

---

## Edit Tool

The Edit tool (`edit.ts`) performs string replacements with multiple fuzzy matching strategies.

### Zod Schema

```typescript
parameters: z.object({
  filePath: z.string().describe("The absolute path to the file to modify"),
  oldString: z.string().describe("The text to replace"),
  newString: z.string().describe("The text to replace it with"),
  replaceAll: z.boolean().optional().describe("Replace all occurrences (default false)"),
})
```

### Replacer Chain

The Edit tool uses a chain of "replacers" - each attempting to find a match with increasing flexibility:

```typescript
const replacers = [
  SimpleReplacer,              // Exact match
  LineTrimmedReplacer,         // Ignores leading/trailing whitespace per line
  BlockAnchorReplacer,         // Matches blocks by first/last line anchors
  WhitespaceNormalizedReplacer, // Collapses whitespace
  IndentationFlexibleReplacer, // Ignores indentation differences
  EscapeNormalizedReplacer,    // Handles escape sequences
  TrimmedBoundaryReplacer,     // Trims boundaries
  ContextAwareReplacer,        // Uses context lines as anchors
  MultiOccurrenceReplacer,     // Finds all exact matches
]
```

### Replacer Examples

**1. SimpleReplacer** - Direct string match:
```typescript
export const SimpleReplacer: Replacer = function* (_content, find) {
  yield find  // Just returns the search string as-is
}
```

**2. LineTrimmedReplacer** - Matches lines ignoring whitespace:
```typescript
export const LineTrimmedReplacer: Replacer = function* (content, find) {
  const originalLines = content.split("\n")
  const searchLines = find.split("\n")

  for (let i = 0; i <= originalLines.length - searchLines.length; i++) {
    let matches = true
    for (let j = 0; j < searchLines.length; j++) {
      if (originalLines[i + j].trim() !== searchLines[j].trim()) {
        matches = false
        break
      }
    }
    if (matches) {
      // Yield the actual content (preserving original whitespace)
      yield content.substring(matchStartIndex, matchEndIndex)
    }
  }
}
```

**3. BlockAnchorReplacer** - Uses Levenshtein distance for fuzzy block matching:
```typescript
// Uses first and last lines as anchors
// Calculates similarity of middle lines using Levenshtein distance
// Thresholds:
const SINGLE_CANDIDATE_SIMILARITY_THRESHOLD = 0.0   // Accept any single match
const MULTIPLE_CANDIDATES_SIMILARITY_THRESHOLD = 0.3 // Need 30% similarity for multiple
```

### Replace Function

```typescript
export function replace(content: string, oldString: string, newString: string, replaceAll = false): string {
  for (const replacer of replacers) {
    for (const search of replacer(content, oldString)) {
      const index = content.indexOf(search)
      if (index === -1) continue

      if (replaceAll) {
        return content.replaceAll(search, newString)
      }

      // Check for unique match
      const lastIndex = content.lastIndexOf(search)
      if (index !== lastIndex) continue  // Multiple matches, skip this replacer

      return content.substring(0, index) + newString + content.substring(index + search.length)
    }
  }

  throw new Error("oldString not found in content")
}
```

---

## Glob Tool

The Glob tool (`glob.ts`) finds files by pattern using ripgrep.

### Zod Schema

```typescript
parameters: z.object({
  pattern: z.string().describe("The glob pattern to match files against"),
  path: z.string().optional().describe("The directory to search in"),
})
```

### Implementation

```typescript
export const GlobTool = Tool.define("glob", {
  async execute(params, ctx) {
    const limit = 100
    const files = []

    // Use ripgrep's --files mode with glob filter
    for await (const file of Ripgrep.files({
      cwd: search,
      glob: [params.pattern],
    })) {
      if (files.length >= limit) {
        truncated = true
        break
      }

      // Get modification time for sorting
      const stats = await Bun.file(full).stat()
      files.push({
        path: full,
        mtime: stats.mtime.getTime(),
      })
    }

    // Sort by modification time (newest first)
    files.sort((a, b) => b.mtime - a.mtime)

    return {
      title: path.relative(Instance.worktree, search),
      metadata: { count: files.length, truncated },
      output: files.map((f) => f.path).join("\n"),
    }
  },
})
```

---

## Grep Tool

The Grep tool (`grep.ts`) searches file contents using ripgrep.

### Zod Schema

```typescript
parameters: z.object({
  pattern: z.string().describe("The regex pattern to search for"),
  path: z.string().optional().describe("The directory to search in"),
  include: z.string().optional().describe('File pattern filter (e.g. "*.js")'),
})
```

### Ripgrep Arguments

```typescript
const args = [
  "-nH",                           // Line numbers, filenames
  "--hidden",                      // Search hidden files
  "--follow",                      // Follow symlinks
  "--no-messages",                 // Suppress error messages
  "--field-match-separator=|",     // Use | as separator
  "--regexp", params.pattern,      // The search pattern
]
if (params.include) {
  args.push("--glob", params.include)
}
```

### Output Format

```
Found 15 matches

/path/to/file1.ts:
  Line 42: const result = searchPattern.match(...)
  Line 89: function searchPattern() {

/path/to/file2.ts:
  Line 12: import { searchPattern } from "./utils"

(Results are truncated. Consider using a more specific path or pattern.)
```

---

## Output Truncation Logic

The Truncate namespace (`truncation.ts`) handles output size limits.

### Constants

```typescript
export const MAX_LINES = 2000
export const MAX_BYTES = 50 * 1024  // 50KB
export const DIR = path.join(Global.Path.data, "tool-output")
const RETENTION_MS = 7 * 24 * 60 * 60 * 1000  // 7 days
```

### Truncation Algorithm

```typescript
export async function output(text: string, options: Options = {}, agent?: Agent.Info): Promise<Result> {
  const maxLines = options.maxLines ?? MAX_LINES
  const maxBytes = options.maxBytes ?? MAX_BYTES
  const direction = options.direction ?? "head"

  const lines = text.split("\n")
  const totalBytes = Buffer.byteLength(text, "utf-8")

  // No truncation needed
  if (lines.length <= maxLines && totalBytes <= maxBytes) {
    return { content: text, truncated: false }
  }

  // Truncate by lines and bytes
  const out: string[] = []
  let bytes = 0

  if (direction === "head") {
    for (let i = 0; i < lines.length && i < maxLines; i++) {
      const size = Buffer.byteLength(lines[i], "utf-8") + (i > 0 ? 1 : 0)
      if (bytes + size > maxBytes) break
      out.push(lines[i])
      bytes += size
    }
  }

  // Save full output to file
  const filepath = path.join(DIR, Identifier.ascending("tool"))
  await Bun.write(Bun.file(filepath), text)

  // Return truncated content with hint
  const hint = hasTaskTool(agent)
    ? `Full output saved to: ${filepath}\nUse the Task tool to have explore agent process this file.`
    : `Full output saved to: ${filepath}\nUse Grep to search or Read with offset/limit.`

  return { content: `${preview}\n\n...${removed} ${unit} truncated...\n\n${hint}`, truncated: true, outputPath: filepath }
}
```

---

## Ripgrep Integration

The Ripgrep module (`file/ripgrep.ts`) wraps the ripgrep binary.

### Auto-Download

If ripgrep isn't installed, it downloads the correct binary:

```typescript
const PLATFORM = {
  "arm64-darwin": { platform: "aarch64-apple-darwin", extension: "tar.gz" },
  "arm64-linux": { platform: "aarch64-unknown-linux-gnu", extension: "tar.gz" },
  "x64-darwin": { platform: "x86_64-apple-darwin", extension: "tar.gz" },
  "x64-linux": { platform: "x86_64-unknown-linux-musl", extension: "tar.gz" },
  "x64-win32": { platform: "x86_64-pc-windows-msvc", extension: "zip" },
}

// Download and extract
const version = "14.1.1"
const url = `https://github.com/BurntSushi/ripgrep/releases/download/${version}/${filename}`
```

### Files Generator

```typescript
export async function* files(input: {
  cwd: string
  glob?: string[]
  hidden?: boolean
  follow?: boolean
  maxDepth?: number
}) {
  const args = [await filepath(), "--files", "--glob=!.git/*"]
  if (input.follow !== false) args.push("--follow")
  if (input.hidden !== false) args.push("--hidden")
  if (input.glob) {
    for (const g of input.glob) {
      args.push(`--glob=${g}`)
    }
  }

  const proc = Bun.spawn(args, { cwd: input.cwd, stdout: "pipe" })
  const reader = proc.stdout.getReader()

  // Stream lines
  while (true) {
    const { done, value } = await reader.read()
    if (done) break
    // Parse and yield file paths
  }
}
```

### Search Function

```typescript
export async function search(input: {
  cwd: string
  pattern: string
  glob?: string[]
  limit?: number
}) {
  const args = [`${await filepath()}`, "--json", "--hidden"]
  args.push("--", input.pattern)

  const result = await $`${{ raw: command }}`.cwd(input.cwd).quiet().nothrow()

  // Parse JSON output from ripgrep
  return lines
    .map((line) => JSON.parse(line))
    .map((parsed) => Result.parse(parsed))
    .filter((r) => r.type === "match")
    .map((r) => r.data)
}
```

---

## FileTime - Staleness Protection

The FileTime module (`file/time.ts`) prevents overwriting files modified externally.

### State Structure

```typescript
const state = {
  read: {
    [sessionID: string]: {
      [path: string]: Date | undefined
    }
  },
  locks: Map<string, Promise<void>>  // Write locks per file
}
```

### Key Functions

```typescript
// Record when a file was read
export function read(sessionID: string, file: string) {
  state.read[sessionID] = state.read[sessionID] || {}
  state.read[sessionID][file] = new Date()
}

// Serialize writes to the same file
export async function withLock<T>(filepath: string, fn: () => Promise<T>): Promise<T> {
  const currentLock = state.locks.get(filepath) ?? Promise.resolve()
  // Chain promises to serialize access
  // ...
}

// Check if file was modified since last read
export async function assert(sessionID: string, filepath: string) {
  const time = get(sessionID, filepath)
  if (!time) throw new Error(`You must read the file ${filepath} before overwriting it.`)

  const stats = await Bun.file(filepath).stat()
  if (stats.mtime.getTime() > time.getTime()) {
    throw new Error(`File ${filepath} has been modified since it was last read.`)
  }
}
```

---

## Design Patterns Used

### 1. Namespace Pattern
All modules use TypeScript namespaces for organization:
```typescript
export namespace File { ... }
export namespace Ripgrep { ... }
export namespace FileTime { ... }
export namespace Truncate { ... }
```

### 2. Lazy Initialization
Heavy resources are loaded lazily:
```typescript
import { lazy } from "../util/lazy"
const state = lazy(async () => { /* expensive init */ })
```

### 3. Generator Pattern
File listing uses async generators for memory efficiency:
```typescript
export async function* files(input) {
  for await (const line of readLines(proc.stdout)) {
    yield line
  }
}
```

### 4. Chain of Responsibility
Edit tool uses multiple replacers in sequence:
```typescript
for (const replacer of [SimpleReplacer, LineTrimmedReplacer, ...]) {
  for (const search of replacer(content, oldString)) {
    // Try to use this match
  }
}
```

### 5. Event-Driven Updates
File changes emit events for reactive updates:
```typescript
await Bus.publish(File.Event.Edited, { file: filepath })
```

### 6. Permission-Based Execution
All tools request permission before executing:
```typescript
await ctx.ask({
  permission: "edit",
  patterns: [filepath],
  metadata: { diff },
})
```

---

## Summary

OpenCode's file operations system is built on:

1. **Ripgrep** - High-performance file searching and listing
2. **Zod** - Runtime type validation for all tool parameters
3. **Tool.define()** - Consistent tool creation with auto-truncation
4. **FileTime** - Staleness protection and write serialization
5. **Smart Truncation** - Output limits with full-content file backup
6. **Fuzzy Matching** - Multiple replacer strategies for flexible editing
7. **LSP Integration** - Real-time diagnostics after file modifications

The architecture prioritizes:
- **Safety** - Permission checks, staleness detection, write locks
- **Performance** - Streaming, generators, lazy loading
- **Developer Experience** - Clear error messages, helpful hints, fuzzy matching

---

*Written by Claude (Opus 4.5) | 2026-01-22 11:45 PST*
