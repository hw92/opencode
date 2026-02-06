# Session Summary: OpenCode Architecture Deep Dive

**Date:** 2026-02-05
**Project:** OpenCode
**Focus:** Architecture Explanation & Documentation

## Executive Summary
This session focused on a comprehensive "Deep Dive" into the OpenCode architecture, explaining the system's core design philosophy and technical implementation. We tailored the explanation to the user's specific questions, culminating in a detailed guide: `learn_docs/design_thinking/2026-02-05-modern-architecture-explained.md`.

The session established the "AI Anatomy" metaphor to deconstruct the system into seven key components: Brain, Hands, Memory, Face, Brake, Mind, and Eyes.

## Key Workflows & Actions
1.  **Architecture Documentation**: Created and iteratively updated `modern-architecture-explained.md`.
2.  **Code Investigation**: deeply analyzed `src/session/processor.ts`, `src/session/prompt.ts`, `src/tool/bash.ts`, `src/tool/skill.ts`, and `src/session/compaction.ts` to reverse-engineer the behaviors.
3.  **Concept Clarification**:
    *   **Cancellation**: Explained `AbortController` + `treeKill` mechanism.
    *   **Infinite Session**: Explained "Idle vs Busy" states and lack of "Completed" state.
    *   **Context Management**: Explained "Pruning" (Tool Output) vs "Summarization" (The Zip File).
    *   **Tool Loading**: Explained "Toolbelt" (Bash/Read) vs "Library" (Skills).
    *   **Stop Condition**: Explained implicit "Act vs Talk" rule in the recursive loop.
    *   **Cognitive Patterns**: Explained how to teach "Notebook" behavior via Skills.
    *   **Retrieval**: Explained the "Librarian" pattern using search tools.

## Component Details (The AI Anatomy)
*   🧠 **Brain**: The Recursive Agent Loop (`src/session/processor.ts`) that cycles through Think -> Act -> Observe.
*   ✋ **Hands**: Local Tool Execution (`src/tool/bash.ts`) allowing filesystem and process control.
*   💾 **Memory**: Flat-File Persistence (`src/session/status.ts`) and Context Compaction (`src/session/compaction.ts`).
*   🗣️ **Face**: The JSON Event Stream that communicates state to the UI.
*   🛑 **Brake**: The Cancellation mechanism using `AbortController` and process tree killing.
*   📚 **Mind**: Skills (`src/tool/skill.ts`) as explicit protocols to guide agent behavior.
*   👀 **Eyes**: Search Tools (`grep`, `codesearch`) for retrieving knowledge without overloading context.

## Relevant Code Snippets
### The Stop Condition (`src/session/prompt.ts`)
```typescript
// If the Agent just finished thinking...
if (lastAssistant.finish !== "tool-calls") {
  // ...and it didn't call a tool, it must be talking to the user.
  BREAK LOOP;
}
```

### The Skill Pattern
```markdown
# Research Skill
1. Create `notes.md`.
2. Read file.
3. Append facts to `notes.md`.
4. Clear context.
```
