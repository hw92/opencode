# Agent-Native Principles Mapping to OpenCode

Analysis of how OpenCode's architecture maps to the agent-native principles from Dan Shipper & Claude's "Agent-native Architectures" guide (every.to/guides/agent-native).

## The 5 Core Principles

| # | Principle | Present in OpenCode? | Notes |
|---|-----------|---------------------|-------|
| 1 | Parity | Inverted | Agent can do *more* than UI. TUI is monitoring layer, not feature-rich UI |
| 2 | Granularity | Yes, strongly | Atomic tools: bash, read, write, edit, glob, grep, etc. |
| 3 | Composability | Yes | Skill system (SKILL.md) + instruction files = new features via prompts |
| 4 | Emergent Capability | Enabled, not measured | Architecture supports it; no feedback loop to discover patterns |
| 5 | Improvement Over Time | Yes | CLAUDE.md, AGENTS.md, auto-memory, session history |

## Detailed Mapping

### Granularity -- Strong match

Tools are atomic primitives in `packages/opencode/src/tool/`. Each does one thing. Features emerge from the agent composing them in a loop, not from monolithic tools.

```
Tools: read, write, edit, bash, glob, grep, webfetch, websearch, ...
Prompt: "Fix the bug in auth module"
-> Agent makes the decisions
-> To change behavior, edit the prompt
```

This is the article's "more granular" pattern almost exactly.

### Agent Loop -- Foundational

`packages/opencode/src/session/processor.ts` runs a `while(true)` loop. The agent calls tools, gets results, decides next action, repeats until the LLM stops issuing tool calls. Includes doom-loop detection for repeated identical tool calls.

Matches: "agent operating in a loop until the outcome is reached."

### Composability -- Via skills and prompts

- `SKILL.md` files define reusable behaviors as prompts
- `CLAUDE.md` / `AGENTS.md` customize agent behavior per project
- Plugin hooks can transform system prompts
- Users ship new "features" as prompt files, not code

### Improvement Over Time -- Multiple mechanisms

- **Accumulated context:** `CLAUDE.md`, `AGENTS.md` as persistent project context
- **Auto-memory:** `~/.claude/projects/.../memory/` persists across sessions
- **Session history:** SQLite-backed conversation persistence
- **User-level customization:** Users modify instruction files for their workflow

### Context Injection -- Multi-layered

System prompts assembled from:
1. Provider-specific instructions (Anthropic, OpenAI, etc.)
2. Environment info (working dir, platform, date)
3. Instruction files found by walking directory tree
4. Agent-specific prompts from config
5. Plugin transforms

Matches: "inject available resources and capabilities into system prompt."

### Model Tier Selection -- Yes

- 30+ provider support with model metadata (capabilities, cost, limits)
- Agent types use different model configs (build vs plan)
- Small model selection for lightweight tasks

### Agent-to-UI Communication -- Event bus

Typed event system (`Bus.publish/subscribe`) in `packages/opencode/src/bus/`:
- `session.created`, `session.updated`, `session.diff`
- `file.edited`, `session.error`
- Real-time streaming of tool calls and text to TUI

Matches: "no silent actions" and the AgentEvent pattern.

### Checkpoint/Resume -- Snapshots + sessions

- Git-based filesystem snapshots before changes
- Full session persistence in SQLite
- Session forking/branching
- Revert to any message point

### Dynamic Capability Discovery -- MCP

MCP (Model Context Protocol) dynamically discovers tools from external servers at runtime. Tool list change notifications update available capabilities. Matches the "discover + access" pattern.

### Approval / User Agency -- Permission system

Permission system gates tool execution (auto-allow, ask, deny). `PermissionNext.ask()` for human-in-the-loop decisions. Maps to the article's stakes/reversibility framework.

### Files as Interface -- Hybrid

- Config is file-based: `CLAUDE.md`, `opencode.json`, `SKILL.md`
- Core tool set is file-centric: read/write/edit/glob/grep
- But session/message storage uses SQLite, not flat files
- Hybrid approach: files for legibility, database for structure

## Principles Not Strongly Present

### Parity (as defined in the article)

The article's parity means "agent can do anything the UI can." OpenCode inverts this -- the agent is the primary actor with full system access; the TUI is a monitoring/interaction layer. This principle is more relevant to traditional apps adding agents, not agent-first tools.

### Latent Demand Discovery

No mechanism to observe what users ask and formalize emerging patterns. The architecture enables emergent capability but lacks the product feedback loop. More of a product strategy concern than architecture.

### Progressive Disclosure (as UX strategy)

The CLI is a power tool from the start. No deliberate progressive disclosure UX layers. However, the natural language interface inherently scales with the ask.

## Key Takeaway

OpenCode embodies most agent-native principles because it is the same class of tool as Claude Code, which the article explicitly cites as the paradigm's origin. The absent principles (parity, latent demand discovery) apply more to traditional apps being redesigned as agent-native, not to agent-first CLI tools.

---

*Written by Claude (claude-opus-4-6) | 2026-02-05 16:24 PST*
