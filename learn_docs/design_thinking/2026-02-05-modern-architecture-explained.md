# Design Thinking: Modern AI-Native Architecture Explained

## 1. The High-Level Philosophy
You asked: *"Why is this different from my text-book Frontend/Backend/Supabase stack?"*

Traditional apps are **CRUD-first**: Create, Read, Update, Delete data from a database.
**OpenCode** is **Agent-first** and **Local-first**.

*   **Logic lives on the Edge (and Local)**: Instead of a central server, logic runs on Cloudflare Workers (Edge) for speed and usually runs LOCALLY on your machine (`opencode` CLI) for privacy and zero-latency code editing.
*   **State is Live**: Instead of just saving to a DB, we use **Durable Objects** to create "Live Sessions". This allows the AI and the User to see the same thing in real-time, like a multiplayer game.
*   **Serverless SQL**: We use **PlanetScale** (MySQL) because it scales infinitely without managing servers, matching the "hands-off" philosophy of Serverless.

---

## 2. Architecture Map: Old School vs. New School

| Concept | Traditional Stack (Supabase/Rails) | OpenCode (Modern AI-Native) | Location in Code |
| :--- | :--- | :--- | :--- |
| **Frontend** | React/Next.js | **SolidJS** + **Vite** (Faster, lighter) | `packages/app` |
| **Backend** | Node/Python Server (API) | **Cloudflare Workers** (Hono) | `packages/function` |
| **Database** | Postgres (Rows & Columns) | **PlanetScale** (Relational) + **Durable Objects** (Real-time Sync) | `packages/console`, `packages/function` |
| **Auth** | Supabase Auth / JWT | **Cloudflare Workers** (Custom OAuth) | `packages/console/function/src/auth.ts` |
| **AI Engine** | Python Backend Script | **Local CLI Sidecar** (TypeScript) | `packages/opencode` |
| **Infrastructure**| Docker / AWS EC2 | **SST** (Infrastructure as Code) | `infra/` |

---

## 3. The Orchestration: How it works together

### The "Brain" is Local (`packages/opencode`)
Unlike a SaaS where the AI lives in the cloud, specific logic here runs on **your machine**.
*   **Why?** To read your files, run terminal commands, and use tools without uploading your whole repo to a server.
*   **How?** The `opencode` package is a CLI acting as a local server (Sidecar). The UI talks to it to run "Agents".

### The "Cloud" is for Sync & Admin (`packages/function`, `packages/console`)
*   **SyncServer**: When you "share" a session, it doesn't just save to a DB. It spins up a **Durable Object**. This is a tiny, stateful mini-server dedicated to *just that session*. It handles WebSocket connections so multiple people (or you + web) can watch the same stream.
*   **Console**: Handles the boring stuff—Billing (Stripe), User Management, and Organization settings.

### The "Glue" (`packages/sdk`)
*   This package defines the **Contract**. It exports typed clients so the Frontend knows exactly what the Backend expects. No guessing API endpoints.

---

## 4. Directory Guide related to "User Experience"

If you want to touch...

*   **The UI (Buttons, Editors, Chat)**: Go to `packages/app`.
    *   `src/components`: The visual bricks.
    *   `src/pages`: The screens.
*   **The AI Logic (Prompts, Tools, Agents)**: Go to `packages/opencode`.
    *   `src/agent`: Where the AI behavior lives.
    *   `src/cli`: The commands that trigger the AI.
*   **The Backend API (Syncing, saving)**: Go to `packages/function`.
    *   `src/api.ts`: The main entry point.
*   **The Database Schema**: Go to `packages/console/core`.
    *   Look for `src/schema.ts` (usually where Drizzle/ORM definitions live).
*   **The Infrastructure**: Go to `infra/`.
    *   `app.ts`: Defines the API and Web resources.

## 5. You Nailed It: The "Dual Backend" Model
You are exactly right. This app has **Two Backends** working in parallel:

| Backend Type | Where it runs | What it does |
| :--- | :--- | :--- |
| **1. The Local Backend** | **Your Mac** (`packages/opencode`) | **The Muscle**. It has direct access to your files, terminal, and git. It's the "Body" of the AI. |
| **2. The Cloud Backend** | **Cloudflare** (`packages/function`) | **The Nervous System**. It handles things that *must* be shared: Auth, Billing, and syncing data between devices. |

This "Split Brain" is why the app feels local (fast, private) but still has cloud features (shareable links, team sync).

## 6. Technology Spotlight: Cloudflare Workers & Hono
You asked: *"What is Cloudflare Workers (Hono)?"*

### Cloudflare Workers vs. Node.js
*   **The Old Way (Node.js)**: You rent a server (or container) in `us-east-1`. Users in Tokyo have to wait for signals to travel halfway around the world. It's like having one single pizza shop for the whole world.
*   **The Modern Way (Cloudflare Workers)**: Your code runs on **Cloudflare's Edge Network** (hundreds of locations worldwide). When a user in Tokyo visits your site, the code runs **in Tokyo**. It's like having a pizza shop in every neighborhood.
    *   **Technical Difference**: It uses "V8 Isolates" (Google Chrome's engine) instead of full OS containers. This means 0ms "cold starts" and cheaper billing.

### Hono vs. Express
*   **The Old Way (Express)**: The standard web framework for Node.js. It's great but heavy and designed for long-running servers.
*   **The Modern Way (Hono)**: Think of it as **"Express for the Edge"**.
    *   **Same Feel**: It looks just like Express (`app.get('/', ...)`).
    *   **Different Core**: It's ultra-lightweight (14kB) and uses standard Web APIs (Request/Response) instead of Node-specific ones. This makes it perfect for the serverless/edge environment where every millisecond counts.

In this codebase, **Hono** is the framework running inside the **Cloudflare Workers**.

## 7. The "Shopping List": What accounts do you need?
To run this full stack in production, you would need accounts with:

1.  **Cloudflare** (The Main Host):
    *   **Why?** Runs the Workers (Backend), hosts the Static Assets (Frontend), and handles DNS.
    *   **Cost:** Generous free tier, then pay-as-you-go.
2.  **PlanetScale** (The Database):
    *   **Why?** Provides the serverless MySQL database.
3.  **AWS** (Specific Services):
    *   **Why?** Used for transactional emails (SES). *Note: This is often optional or swappable.*
4.  **Stripe** (Payments):
    *   **Why?** If you plan to charge users (billing logic is in `packages/console`).

## 8. You are Right: The "Local-First" AI Advantage
You correctly observed that the **AI Agent needs to run locally**.

*   **Direct System Access**: The AI needs to run commands (`bash`, `git`) and edit files (`write`, `replace`).
*   **Safety & Power**: By running on your machine (via the `opencode` CLI), the AI has the same power you do. It's not sandboxed in a remote cloud container; it's right there in your terminal.
*   **The Code Proof**: In `packages/opencode/src/tool/bash.ts`, you can see the agent uses Node.js `spawn` to run shell commands directly on your laptop.

## 9. Definitive Answer: Where does the Agent Run?
***The Agent runs on YOUR machine.***

I verified this by checking the code:
1.  **Cloudflare (`packages/function`)**: I searched for "agent" and "AI" logic here. **Result: None.** It only handles auth, syncing, and simplistic API tasks.
2.  **Local CLI (`packages/opencode`)**: I found `src/agent/agent.ts`. This file contains the "Brain" — it imports the AI SDK, manages prompts, and decides what tools to call.

**Why?**
*   **Privacy**: Your code doesn't leave your machine unless you explicitly share it.
*   **Cost**: You pay for the API calls (OpenAI/Anthropic) directly or via your local keys, not running heavy compute on Cloudflare.
*   **Context**: The agent needs to read *all* your files. Uploading your entire hard drive to Cloudflare for every request would be slow and expensive. It reads them locally instanty.

## 10. Under the Hood: Inside the "Brain" (`packages/opencode`)
You asked: *"How is the code organized inside the local engine?"*

It follows a clean **input -> process -> action** flow:

| Directory | Purpose | Analogy |
| :--- | :--- | :--- |
| **`src/cli`** | **Entry Point**. Parses commands like `opencode run`. | **The Ears**. It hears what you want to do. |
| **`src/session`** | **State Machine**. Manages the chat history, context window, and the "loop" of talking to the AI. | **The Memory**. Remembers what we just said. |
| **`src/agent`** | **Configuration**. Defines *who* the agent is (e.g., "Build Agent" vs "Plan Agent"), its system prompt, and its permissions. | **The Personality**. Defines behavior and rules. |
| **`src/provider`** | **LLM Adapters**. Connects to OpenAI, Anthropic, or Local LLMs. | **The Part of the Brain that talks**. Abstraction layer for Intelligence. |
| **`src/tool`** | **Capabilities**. Actual code for `bash`, `read_file`, `grep`. | **The Hands**. The logic that actually touches your computer. |

**The Flow**:
1.  **CLI** starts a **Session**.
2.  **Session** loads an **Agent** ("Planner").
3.  **Agent** uses a **Provider** (Claude 3.5) to think.
4.  **Provider** decides to call a **Tool** (`bash`).
5.  **Tool** executes and returns output to **Session**.

## 11. The Heartbeat: The Agent Loop
You asked: *"Show me the agent loop."*

The core logic lives in `packages/opencode/src/session/processor.ts`. It's a `while (true)` loop that keeps the conversation alive until the AI is "done".

### The Cycle
1.  **Stream**: It calls `LLM.stream()`. This opens a connection to the brain (OpenAI/Anthropic).
2.  **Event Listeners**: It listens for specific events from the brain:
    *   `text-delta`: "I am generating text..." (updates the UI in real-time).
    *   `tool-call`: "I want to run `list_dir`."
    *   `reasoning-delta`: "I am thinking..." (Chain of thought).
3.  **Tool Execution**: If the AI asks for a tool:
    *   The app pauses the AI.
    *   It executes the tool locally (e.g., runs `ls -la`).
    *   It feeds the Result (`tool-result`) back into the chat history.
    *   **The Loop Restarts**: It goes back to Step 1 with the new information.
4.  **Safety Net**: It has a `DOOM_LOOP_THRESHOLD` (set to 3). If the AI tries to run the *exact same tool* with the *exact same input* 3 times in a row, it hits the "Emergency Brake" and stops to prevent infinite loops.

## 12. Anatomy of a Tool: How `bash` works
You asked: *"How are tools implemented?"*

All tools live in `packages/opencode/src/tool/`. They share a simple structure defined in `tool.ts`.

Taking **Bash** (`src/tool/bash.ts`) as an example:

```typescript
export const BashTool = Tool.define("bash", async () => {
  return {
    description: "Executes a shell command on the local machine",
    // 1. The Schema (What the AI needs to send)
    parameters: z.object({
      command: z.string().describe("The command to execute (e.g., 'ls -la')"),
      workdir: z.string().optional(),
    }),
    // 2. The Logic (What happens on your Mac)
    async execute({ command, workdir }, ctx) {
       // ...security checks...
       const proc = spawn(command, { cwd: workdir || process.cwd() });
       // ...wait for output...
       return {
         title: "Run command",
         output: proc.stdout, // Sends this back to the AI
         metadata: { exitCode: 0 }
       }
    }
  }
})
```

**Key Parts**:
1.  **Zod Schema**: Tells the LLM exactly what arguments are expected (`command` is a string).
2.  **Execute Function**: The actual Node.js code that runs. It has access to your full system (fs, child_process).
3.  **Return Value**: The output string that gets fed back into the AI's context window.

## 13. The Memory: How Session State is Saved
You asked: *"How is the memory saved?"*

I checked `packages/opencode/src/storage/storage.ts` and found something surprising:
**There is no database.**

It uses a **Flat File System (JSON)**.
*   **Location**: `~/.opencode/storage/` (or similar data dir).
*   **Format**: Every session, message, and part is saved as a tiny `.json` file.
*   **Why?**
    *   **Zero Dependencies**: No need to install SQLite or Postgres.
    *   **Git Friendly**: You can technically commit your chat history.
    *   **Debuggable**: You can open any JSON file and read the exact state.

It uses **Bun's File API** (`Bun.file`, `Bun.write`) for high-performance IO.

## 14. The Neural Link: Frontend <-> CLI
You asked: *"How does the UI talk to the Agent?"*

Since the Agent runs on your Mac, but the UI is a Web App (usually sandboxed), how do they connect?

**The Secret: Localhost HTTP Server**
1.  **The CLI starts a Server**: When you run `opencode run`, it spins up a hidden web server (e.g., `http://localhost:3000`) inside `packages/opencode/src/server`.
2.  **The UI connects via SDK**: The frontend (`packages/app`) uses a generated client (`@opencode-ai/sdk`) to talk to this localhost address.
3.  **Real-time Stream**: They use **Server-Sent Events (SSE)** or HTTP streams to push text updates (the `text-delta` events we saw earlier) to the UI instantly.

### Wait, how does it touch files?
You might ask: *"If it's a web server, isn't it sandboxed like a browser?"*

**No.**
*   **The Browser (Frontend)** is sandboxed. It *cannot* touch your files directly.
*   **The Server (at localhost)** is a **Node.js/Bun process**. It runs directly on your OS with your user permissions (just like when you type `ls` in terminal).
*   **The Flow**:
    1.  Frontend says: "Agent, please read `README.md`".
    2.  Server (which has disk access) reads `README.md`.
    3.  Server sends the *text content* back to the Frontend.

The Server acts as a **Bridge** between the sandboxed Web UI and your actual Hard Drive.

### Mental Model: "Just like Jupyter"
You mentioned **Jupyter Notebooks**, and that is the **perfect analogy**:

| Concept | Jupyter Notebooks | OpenCode (This App) |
| :--- | :--- | :--- |
| **The "Kernel"** | Python process (Local) | `opencode` CLI (Local) |
| **The "Dashboard"** | Web Interface (Client) | `packages/app` (Client) |
| **The Connection** | WebSockets / ZeroMQ | HTTP / SSE |
| **Why?** | To run Python code on your laptop via a Browser. | To run Agents/Bash on your laptop via a Browser. |

It's the exact same architecture, just specialized for **Coding Agents** instead of Data Science.

### The "Magic Trick": Why the GUI is Nice
You asked: *"Why can we use a nice GUI to control a terminal?"*

If we just streamed raw text (like SSH), the UI would look like a boring black box.
The secret is that **OpenCode doesn't send text; it sends Objects.**

1.  **Terminal**: Sends raw text: `Running git status...`
2.  **OpenCode Server**: Sends **JSON Events**:
    ```json
    {
      "type": "tool-call",
      "tool": "git",
      "args": { "command": "status" },
      "status": "loading"
    }
    ```
3.  **The GUI**: Receives this JSON and thinks: *"Ah! `status: loading`. I will show a **Spinner Component** and a **Cancel Button**."*

Because the "Brain" and "Hands" speak **JSON**, the "Face" (UI) can render beautiful interactive components instead of just lines of text.

### 15. The Emergency Stop: How "Cancel" Works
You asked: *"How does the Cancel button actually kill the process?"*

It is not magic. It is a precise chain of events:

1.  **The Click**: You click "Stop" in the UI.
2.  **The Signal**: The Frontend sends a `POST /session/:id/abort` request to the Localhost Server.
3.  **The Trigger**: The Server (`SessionRoutes`) calls `SessionPrompt.cancel()`.
4.  **The Abort**: This triggers an `AbortController` inside the running Agent Loop.
5.  **The Kill**: The active Tool (e.g., `bash.ts`) is listening for this `abort` event.
    *   It immediately calls `treeKill()` on the spawned process.
    *   It sends a `SIGTERM` signal to the actual shell process on your Mac.

**Result**: The process dies immediately, and the Agent stops thinking.

### 15.1 Pause, not Stop
You asked: *"Does the session end? Is history lost?"*

**No.**
*   **The Session Lives On**: The JSON history files are safe on disk.
*   **The Context is Saved**: The agent remembers that it *tried* to run a command and was stopped.
*   **Resumption**: If you type "Continue", the Agent reads the entire history (including the stopped action) and thinks: *"Oh, I was stopped. I should try again or ask what to do."*

It is just a **Turn Stop**, not a Session End.

### 15.2 The Infinite Session
You asked: *"What defines a session end?"*

Technically? **Nothing.**
There is no "Closed" or "Finished" state for a Session in the database.

*   **States**: A session is only ever `IDLE` (waiting for you) or `BUSY` (thinking).
*   **Lifecycle**: It lasts forever, stored on your disk, until you explicitly **Delete** it.
*   **Mental Model**: Think of it like a Slack channel or a Discord thread. It doesn't "end" just because you stopped typing. It is always there, ready for more context.

### 16. The Infinite Memory Problem (Compaction)
You asked: *"How do we manage context length if the session lasts forever?"*

If we just sent the whole history every time, we would crash the LLM or go bankrupt.
OpenCode uses a **Two-Stage Compaction Strategy** (`src/session/compaction.ts`):

#### Stage 1: "Pruning" (The Surgical Strike)
The system looks at old tool calls.
Did you print a 5,000-line file 20 turns ago? You probably don't need that raw text anymore.
*   **Action**: The system deletes the *output* of old tool calls but keeps the *record* that they happened.
*   **Result**: Massive savings (sometimes 90% of tokens) with zero loss of logical flow.

#### Stage 2: "Summarization" (The Zip File)
When a conversation gets too long, the Agent pauses and talks to *itself*.
1.  It sends the full history to a fast model.
2.  It asks: *"Summarize what we have done so far, including file paths and key decisions."*
3.  It replaces the old messages with this new **Summary Block**.

This allows a session to go on for thousands of turns while only keeping the "Fresh" context in the LLM's active memory.


### 17. The Library Model (Lazy Loading)
You asked: *"Do we carry all the books (skills) at once? Isn't that too heavy?"*

You are absolutely right. Loading every possible instruction set into the context would be wasteful.
We solve this with a **Toolbelt vs. Library** separation:

1.  **The Toolbelt (Always Loaded)**:
    *   The Agent *always* carries the "Hands": `bash`, `read_file`, `write_file`.
    *   These are small definitions, so the "weight" is low.

2.  **The Library (Lazy Loaded)**:
    *   The Agent does *not* carry all Skills (e.g., "How to Deploy", "How to Test").
    *   Instead, it carries a **Library Card** (`src/tool/skill.ts`).
    *   The context only lists the **Titles** of available skills.
    *   **Action**: When the Agent needs to do a complex task, it calls `skill("deploy_app")`.
    *   **Result**: The system *then* reads the markdown file for that skill and injects it into the context for that turn.

It is **Just-in-Time Knowledge**.
The Agent travels light, but has access to the entire library.

### 18. The Stop Condition
You asked: *"How does the agent know when to stop?"*

In the recursive loop (`src/session/prompt.ts`), there is a specific check at the start of every turn:

```typescript
if (lastAssistant.finish !== "tool-calls") {
  break;
}
```

*   **Scenario A (Tool Call)**: The LLM generates a tool call. The finish reason is `"tool-calls"`. The loop **CONTINUES**.
*   **Scenario B (Text Response)**: The LLM generates a plain answer (e.g., "Done!"). The finish reason is `"stop"`. The loop **BREAKS**.

**Simple Rule**:
*   If the Agent *Acts* (uses a tool), it gets another turn.
*   If the Agent *Talks* (to you), it passes the microphone back to you.

Are you ready to graduate from "Architecture Student" to "Agent Engineer"? 😉

### 19. The Cognitive Pattern (The Notebook)
You asked: *"Does the agent know how to use a notebook automatically? Or do I need to teach it?"*

The answer is **No**. The Agent does not know "how to study" by default.
If you don't tell it anything, it will try to memorize everything (put it in context) and fail.

**You must teach it** using a **Skill**.

#### How to Teach the "Notebook" Pattern
You write a skill file (e.g., `skills/research.md`) that explicitly defines the process:

```markdown
# Research Skill
Description: precise instructions for analyzing large codebases

## The Protocol
1.  **Create a Notebook**: Run `write_file("notes.md", "")` to start a scratchpad.
2.  **Read & Summarize**:
    *   Read one file at a time.
    *   Extract key facts.
    *   Run `write_file` to APPEND these facts to `notes.md`.
3.  **Clear Memory**: Do not output the full file content to me. Only confirm you saved the notes.
4.  **Final Answer**: Read your own `notes.md` to answer the user's question.
```

*   **Without this Skill**: The Agent reads 50 files, overflows the context, and crashes.
*   **With this Skill**: The Agent acts like a disciplined student, keeping the context clean and the knowledge safe in `notes.md`.

**Verdict**: The Architecture provides the *capability* (tools), but YOU provide the *methodology* (skills).

### 19.1 The Spectrum of Specificity
You asked: *"Do I need to spell out every step? Can't the LLM figure it out?"*

It depends on **Reliability**.
Tools (`write_file`) are primitives (Hammer). Skills are procedures (Blueprint).

*   **Implicit (Lazy)**: *"Research these files."*
    *   **Result**: The Agent might read 5 files, hallucinate a summary, or crash the context. It relies on "Generic Intelligence."
*   **Explicit (Robust)**: *"Create `notes.md`. Read file A. Append exact quotes. Read file B..."*
    *   **Result**: The Agent follows your Standard Operating Procedure (SOP) exactly.

**Rule of Thumb**:
For **Domain-Specific Tasks** (e.g., "Audit Crypto Transaction"), you **must be explicit**.
The LLM knows how to write English, but it doesn't know *your* company's audit compliance rules unless you put them in the Skill file.

### 20. The Librarian (Retrieval)
You asked: *"Can it search a database of legal documents without reading all of them?"*

**Yes.** But again, it's not magic. It's a **Tool**.
Just like `bash` lets it touch files, we give it a **Search Tool** to touch Knowledge.

*   **Codebase**: It uses `grep` (Regex) or `codesearch` (Semantic/Vector).
    *   *Agent*: "Find code about 'authentication'." -> *Tool*: "See `auth.ts` lines 50-100."
*   **Legal/Docs**: It would use a hypothetical `rag_search` tool.
    *   *Agent*: "Find precedents for 'Contract Breach'." -> *Vector DB*: "See Case 402, Page 12."
*   **Databases**: It would use a `sql_query` tool.
    *   *Agent*: "Select * from users where..." -> *DB*: "10 rows returned."

**The Architectural Pattern**:
1.  **Query**: Agent sends a keyword/vector query.
2.  **Index**: The external system (Vector DB, SQL, Grep) filters 1TB -> 1KB.
3.  **Read**: The Agent puts *only* that 1KB into its Context.

**The "Librarian" Metaphor**:
The Agent doesn't memorize the library. It asks the Librarian (The Tool) to fetch the right book.

## Conclusion: Mission Accomplished?

We have now covered every organ of the AI Anatomy:

*   🧠 **Brain**: Recursive Loop (`src/session/processor.ts`)
*   ✋ **Hands**: Bash/File Tools (`src/tool/bash.ts`)
*   💾 **Memory**: Status/Compaction (`src/session/status.ts`)
*   🗣️ **Face**: JSON Stream
*   🛑 **Brake**: Cancel (`treeKill`)
*   📚 **Mind**: Skills (Protocols)
*   👀 **Eyes**: Search Tools (Retrieval)

You now have a complete mental model of the entire system.
Are you ready to graduate from "Architecture Student" to "Agent Engineer"? 😉
