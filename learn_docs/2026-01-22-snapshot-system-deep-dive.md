# OpenCode Snapshot System Deep Dive

This document provides a comprehensive analysis of OpenCode's snapshot system, which enables file state tracking, undo/rollback functionality, and change diffing across sessions.

## Architecture Overview

The snapshot system leverages Git's internal mechanisms to track file states without interfering with the user's actual Git repository. It maintains a separate, shadow Git repository dedicated to snapshot tracking.

```
                           OpenCode Snapshot Architecture
+-------------------------------------------------------------------------+
|                                                                         |
|  +------------------+      +-----------------------+                    |
|  |   Session/Edit   |----->|   Snapshot.track()    |                    |
|  |   Operations     |      |   (creates tree hash) |                    |
|  +------------------+      +-----------------------+                    |
|          |                           |                                  |
|          v                           v                                  |
|  +------------------+      +-----------------------+                    |
|  |   File Changes   |      |   Shadow Git Repo     |                    |
|  |   (edit/write)   |      |   ~/.local/share/     |                    |
|  +------------------+      |   opencode/snapshot/  |                    |
|          |                 |   {project-id}/       |                    |
|          v                 +-----------------------+                    |
|  +------------------+                |                                  |
|  |   Snapshot.patch |<---------------+                                  |
|  |   (track changes)|                                                   |
|  +------------------+                                                   |
|          |                                                              |
|          v                                                              |
|  +------------------+      +-----------------------+                    |
|  |   PatchPart      |----->|   Session Storage     |                    |
|  |   (persisted)    |      |   (message parts)     |                    |
|  +------------------+      +-----------------------+                    |
|          |                                                              |
|          v                                                              |
|  +------------------+      +-----------------------+                    |
|  |  Revert/Restore  |<-----|   SessionRevert       |                    |
|  |  Operations      |      |   (undo mechanism)    |                    |
|  +------------------+      +-----------------------+                    |
|                                                                         |
+-------------------------------------------------------------------------+
```

## Key Files and Their Purposes

| File | Purpose |
|------|---------|
| `/packages/opencode/src/snapshot/index.ts` | Core snapshot functionality - track, patch, restore, revert, diff |
| `/packages/opencode/src/session/revert.ts` | Session revert/unrevert logic using snapshots |
| `/packages/opencode/src/session/processor.ts` | Integrates snapshots into session processing lifecycle |
| `/packages/opencode/src/session/summary.ts` | Uses snapshots to compute session/message diffs |
| `/packages/opencode/src/session/message-v2.ts` | Defines PatchPart and StepStartPart/StepFinishPart schemas |
| `/packages/opencode/src/tool/edit.ts` | Uses Snapshot.FileDiff for tracking edit operations |
| `/packages/opencode/src/cli/cmd/debug/snapshot.ts` | CLI debugging commands for snapshot inspection |
| `/packages/opencode/src/config/config.ts` | Contains `snapshot` config option to enable/disable |

## Core Snapshot Namespace

The `Snapshot` namespace in `/packages/opencode/src/snapshot/index.ts` provides all core functionality:

### Key Functions

#### `track(): Promise<string | undefined>`

Creates a snapshot of the current working tree state.

```typescript
export async function track() {
  if (Instance.project.vcs !== "git") return
  const cfg = await Config.get()
  if (cfg.snapshot === false) return
  const git = gitdir()

  // Initialize shadow git repo if needed
  if (await fs.mkdir(git, { recursive: true })) {
    await $`git init`.env({
      GIT_DIR: git,
      GIT_WORK_TREE: Instance.worktree,
    }).quiet().nothrow()
    await $`git --git-dir ${git} config core.autocrlf false`.quiet().nothrow()
  }

  // Stage all files and create tree object
  await $`git --git-dir ${git} --work-tree ${Instance.worktree} add .`.quiet().nothrow()
  const hash = await $`git --git-dir ${git} --work-tree ${Instance.worktree} write-tree`
    .quiet().nothrow().text()

  return hash.trim()
}
```

**Key behaviors:**
- Only works for Git-based projects (`Instance.project.vcs === "git"`)
- Can be disabled via config (`cfg.snapshot === false`)
- Uses a shadow Git repository at `~/.local/share/opencode/snapshot/{project-id}/`
- Returns a tree hash (not a commit hash) representing file state

#### `patch(hash: string): Promise<Patch>`

Gets the list of files changed since a given snapshot.

```typescript
export async function patch(hash: string): Promise<Patch> {
  const git = gitdir()
  await $`git --git-dir ${git} --work-tree ${Instance.worktree} add .`.quiet().nothrow()
  const result = await $`git --git-dir ${git} diff --no-ext-diff --name-only ${hash} -- .`
    .quiet().nothrow()

  return {
    hash,
    files: result.text().trim().split("\n")
      .map(x => path.join(Instance.worktree, x))
  }
}
```

**Returns:**
```typescript
type Patch = {
  hash: string      // Original snapshot hash
  files: string[]   // Absolute paths of changed files
}
```

#### `restore(snapshot: string): Promise<void>`

Restores the working tree to match a snapshot exactly.

```typescript
export async function restore(snapshot: string) {
  const git = gitdir()
  await $`git --git-dir ${git} --work-tree ${Instance.worktree} read-tree ${snapshot} &&
          git --git-dir ${git} --work-tree ${Instance.worktree} checkout-index -a -f`
    .quiet().nothrow()
}
```

**Use case:** Full restoration during "unrevert" operations.

#### `revert(patches: Patch[]): Promise<void>`

Selectively reverts specific files to their snapshot states.

```typescript
export async function revert(patches: Patch[]) {
  const files = new Set<string>()
  const git = gitdir()

  for (const item of patches) {
    for (const file of item.files) {
      if (files.has(file)) continue

      const result = await $`git --git-dir ${git} checkout ${item.hash} -- ${file}`
        .quiet().nothrow()

      if (result.exitCode !== 0) {
        // Check if file existed in snapshot
        const checkTree = await $`git ls-tree ${item.hash} -- ${relativePath}`.quiet().nothrow()
        if (checkTree.text().trim()) {
          // File existed but checkout failed - keep current
        } else {
          // File didn't exist in snapshot - delete it
          await fs.unlink(file).catch(() => {})
        }
      }
      files.add(file)
    }
  }
}
```

**Key behaviors:**
- Processes files in order, skipping duplicates
- Handles files that were created (deletes them on revert)
- Handles files that were deleted (restores them)

#### `diff(hash: string): Promise<string>`

Gets a unified diff between snapshot and current state.

```typescript
export async function diff(hash: string) {
  const git = gitdir()
  await $`git --git-dir ${git} --work-tree ${Instance.worktree} add .`.quiet().nothrow()
  const result = await $`git --git-dir ${git} diff --no-ext-diff ${hash} -- .`.quiet().nothrow()
  return result.text().trim()
}
```

#### `diffFull(from: string, to: string): Promise<FileDiff[]>`

Computes detailed file-by-file diff information between two snapshots.

```typescript
export async function diffFull(from: string, to: string): Promise<FileDiff[]> {
  const git = gitdir()
  const result: FileDiff[] = []

  for await (const line of $`git diff --numstat ${from} ${to} -- .`.lines()) {
    const [additions, deletions, file] = line.split("\t")
    const before = await $`git show ${from}:${file}`.text()
    const after = await $`git show ${to}:${file}`.text()

    result.push({
      file,
      before,
      after,
      additions: parseInt(additions),
      deletions: parseInt(deletions),
    })
  }
  return result
}
```

**Returns:**
```typescript
type FileDiff = {
  file: string       // Relative file path
  before: string     // File content at 'from' snapshot
  after: string      // File content at 'to' snapshot
  additions: number  // Lines added
  deletions: number  // Lines deleted
}
```

## Storage Location

The shadow Git repository is stored at:

```
~/.local/share/opencode/snapshot/{project-id}/
```

This is determined by the `gitdir()` function:

```typescript
function gitdir() {
  const project = Instance.project
  return path.join(Global.Path.data, "snapshot", project.id)
}
```

Each project gets its own isolated snapshot repository, preventing cross-project conflicts.

## Snapshot Lifecycle

### 1. Creation: At Start of LLM Step

When the LLM begins processing, a snapshot is created:

```typescript
// In SessionProcessor.process()
case "start-step":
  snapshot = await Snapshot.track()
  await Session.updatePart({
    id: Identifier.ascending("part"),
    messageID: input.assistantMessage.id,
    sessionID: input.sessionID,
    snapshot,
    type: "step-start",
  })
```

### 2. Tracking: At End of LLM Step

After the step completes, changes are captured:

```typescript
case "finish-step":
  await Session.updatePart({
    id: Identifier.ascending("part"),
    snapshot: await Snapshot.track(),
    messageID: input.assistantMessage.id,
    sessionID: input.sessionID,
    type: "step-finish",
    // ... usage info
  })

  // Create patch if files changed
  if (snapshot) {
    const patch = await Snapshot.patch(snapshot)
    if (patch.files.length) {
      await Session.updatePart({
        id: Identifier.ascending("part"),
        messageID: input.assistantMessage.id,
        sessionID: input.sessionID,
        type: "patch",
        hash: patch.hash,
        files: patch.files,
      })
    }
  }
```

### 3. Revert: Undo Changes

The `SessionRevert` namespace handles undoing changes:

```typescript
export async function revert(input: RevertInput) {
  // Collect all patches after the revert point
  const patches: Snapshot.Patch[] = []
  for (const msg of all) {
    for (const part of msg.parts) {
      if (part.type === "patch") {
        patches.push(part)
      }
    }
  }

  // Store current state before reverting
  revert.snapshot = session.revert?.snapshot ?? (await Snapshot.track())

  // Perform the revert
  await Snapshot.revert(patches)

  // Store diff for UI display
  revert.diff = await Snapshot.diff(revert.snapshot)

  return Session.update(input.sessionID, (draft) => {
    draft.revert = revert
  })
}
```

### 4. Unrevert: Restore Reverted Changes

```typescript
export async function unrevert(input: { sessionID: string }) {
  const session = await Session.get(input.sessionID)
  if (!session.revert) return session

  // Restore to the snapshot taken before revert
  if (session.revert.snapshot) {
    await Snapshot.restore(session.revert.snapshot)
  }

  return Session.update(input.sessionID, (draft) => {
    draft.revert = undefined
  })
}
```

### 5. Cleanup: Finalize Revert

When a reverted session continues with new changes:

```typescript
export async function cleanup(session: Session.Info) {
  if (!session.revert) return

  // Remove messages after revert point from storage
  const [preserve, remove] = splitWhen(msgs, (x) => x.info.id === messageID)
  for (const msg of remove) {
    await Storage.remove(["message", sessionID, msg.info.id])
  }

  // Clear revert state
  await Session.update(sessionID, (draft) => {
    draft.revert = undefined
  })
}
```

## Integration with Edit Tool

The edit tool uses `Snapshot.FileDiff` to track changes:

```typescript
// In EditTool.execute()
const filediff: Snapshot.FileDiff = {
  file: filePath,
  before: contentOld,
  after: contentNew,
  additions: 0,
  deletions: 0,
}

for (const change of diffLines(contentOld, contentNew)) {
  if (change.added) filediff.additions += change.count || 0
  if (change.removed) filediff.deletions += change.count || 0
}

ctx.metadata({
  metadata: {
    diff,
    filediff,
    diagnostics: {},
  },
})
```

## Message Part Types

The snapshot system uses several message part types:

### StepStartPart

Marks the beginning of an LLM processing step with initial snapshot:

```typescript
export const StepStartPart = PartBase.extend({
  type: z.literal("step-start"),
  snapshot: z.string().optional(),  // Tree hash at step start
})
```

### StepFinishPart

Marks the end of a step with final snapshot and usage:

```typescript
export const StepFinishPart = PartBase.extend({
  type: z.literal("step-finish"),
  reason: z.string(),               // Finish reason
  snapshot: z.string().optional(),  // Tree hash at step end
  cost: z.number(),
  tokens: z.object({...}),
})
```

### PatchPart

Records files changed during a step:

```typescript
export const PatchPart = PartBase.extend({
  type: z.literal("patch"),
  hash: z.string(),        // Original snapshot hash
  files: z.string().array(),  // Changed file paths
})
```

## Session Summary Integration

The summary system uses snapshots to compute session-wide diffs:

```typescript
async function computeDiff(input: { messages: MessageV2.WithParts[] }) {
  let from: string | undefined
  let to: string | undefined

  // Find earliest 'from' snapshot
  for (const item of input.messages) {
    for (const part of item.parts) {
      if (part.type === "step-start" && part.snapshot) {
        from = part.snapshot
        break
      }
    }
    // Find latest 'to' snapshot
    for (const part of item.parts) {
      if (part.type === "step-finish" && part.snapshot) {
        to = part.snapshot
      }
    }
  }

  if (from && to) return Snapshot.diffFull(from, to)
  return []
}
```

This enables the session summary to show:
- Total files changed
- Total additions/deletions
- Per-file diffs

## Configuration

Snapshots can be disabled via configuration:

```typescript
// In opencode.json
{
  "snapshot": false
}
```

When disabled, `Snapshot.track()` returns early without creating snapshots.

## Debug Commands

The CLI provides debugging commands:

```bash
# Track current snapshot state
opencode debug snapshot track

# Show patch for a snapshot hash
opencode debug snapshot patch <hash>

# Show diff for a snapshot hash
opencode debug snapshot diff <hash>
```

## Design Patterns

### 1. Shadow Repository Pattern

Instead of modifying the user's Git repository, OpenCode maintains a separate shadow repository for snapshots. This ensures:
- User's Git history is unaffected
- No interference with user's staged changes
- Isolation between projects

### 2. Tree Object Pattern

Using Git's `write-tree` command creates tree objects rather than commits. This is more efficient:
- No commit metadata overhead
- Faster creation
- Suitable for frequent snapshots

### 3. Lazy Initialization

The shadow repository is initialized on first use:

```typescript
if (await fs.mkdir(git, { recursive: true })) {
  await $`git init`...
}
```

### 4. Selective Revert

Rather than restoring entire snapshots, the revert system:
- Tracks which files changed
- Reverts only those files
- Handles file creation/deletion correctly

## Use Cases

### 1. Undo Failed Changes

When the AI makes incorrect edits:
1. User triggers revert
2. System collects all patches since target message
3. Files are restored to their pre-edit state
4. User can continue with corrected approach

### 2. Explore Alternatives

User can:
1. Let AI make changes
2. Revert to try a different approach
3. Unrevert if the original was better

### 3. Change Tracking

For session summaries:
1. System tracks first and last snapshots
2. Computes full diff between them
3. Displays file count and line changes

### 4. Tool Metadata

Edit operations include:
- Before/after content
- Line additions/deletions
- For display in tool results

## Error Handling

The system handles errors gracefully:

```typescript
// Failed git operations return empty results
if (result.exitCode !== 0) {
  log.warn("failed to get diff", { hash, exitCode: result.exitCode })
  return { hash, files: [] }
}

// File operations use nothrow() and catch()
await fs.unlink(file).catch(() => {})
```

## Summary

The OpenCode snapshot system provides robust file state tracking by:

1. **Leveraging Git internals** - Uses tree objects for efficient state capture
2. **Maintaining isolation** - Shadow repository per project
3. **Supporting undo** - Full revert/unrevert workflow
4. **Enabling summaries** - Diff computation for session overview
5. **Being configurable** - Can be disabled when not needed
6. **Handling edge cases** - File creation/deletion, failures

This architecture enables a powerful undo system while remaining transparent to the user's actual Git workflow.

---

*Written by Claude (Opus 4.5) | 2026-01-22 01:49 PST*
