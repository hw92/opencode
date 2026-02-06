# Core-First Architecture: A Design Thinking Guide

> How to architect AI applications by starting with the engine, not the UI.

[TOC]



## The Problem: UI-First Thinking

Most developers naturally think:

```
"I need to build a web app that does X"
```

This leads to:
- Business logic trapped in API routes
- Database queries mixed with UI concerns
- Impossible to reuse logic across CLI, mobile, bots
- Testing requires spinning up web servers

## The Solution: Core-First Thinking

Shift your mental model:

```
"I need to build an ENGINE that does X, which happens to have interfaces"
```

## The Architecture Pattern

```
┌─────────────────────────────────────────────────────────────┐
│                      INTERFACES (thin)                       │
│         Web App  │  CLI  │  Mobile  │  Slack Bot            │
└─────────────────────────┬───────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────────┐
│                      SDK / API Layer                         │
│              HTTP client, typed responses                    │
└─────────────────────────┬───────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────────┐
│                      CORE ENGINE (fat)                       │
│    Agents  │  Tools  │  Providers  │  Storage  │  Config    │
│                                                              │
│              ZERO UI FRAMEWORK DEPENDENCIES                  │
└─────────────────────────────────────────────────────────────┘
```

## Key Principles

### 1. The CLI Test

> "Can I run my core logic from a terminal without any web server?"

```bash
# If this works, your architecture is correct
$ myapp analyze AAPL
$ myapp ingest research.pdf
$ myapp backtest strategy.json
```

If your logic requires a browser or web server to run, it's coupled wrong.

### 2. The Library Test

> "Can someone import my core as a library?"

```python
# This should work without FastAPI, Flask, or any web framework
from myapp.core import Engine

engine = Engine()
result = engine.analyze("AAPL")
```

### 3. The Replace-UI Test

> "Could I delete my entire frontend and rebuild it in a week using just the SDK?"

If yes → good separation
If no → logic is leaking into UI

### 4. The Zero-Import Rule

Your `core/` folder should have **zero imports** from:
- FastAPI, Flask, Django
- React, Vue, Svelte
- Any HTTP/web framework

```python
# core/engine.py

# GOOD - pure Python
from core.agents import Analyst
from core.tools import MarketData

# BAD - web framework leaking in
from fastapi import Depends  # NO!
from flask import request    # NO!
```

## Two Pipelines Pattern

For AI applications, separate **ingestion** from **consumption**:

```
┌─────────────────────────────────────────────────────────────┐
│                  DATA INGESTION PIPELINE                     │
│                  "Teaching the system"                       │
├─────────────────────────────────────────────────────────────┤
│  Sources           Processing         Storage                │
│  ─────────         ──────────         ───────                │
│  PDFs/Docs    ──▶  Parser       ──▶   Vector DB             │
│  News APIs    ──▶  Fetcher      ──▶   Postgres              │
│  Market Data  ──▶  Normalizer   ──▶   Time-series DB        │
│                                                              │
│  WHO: Admin, Researcher, Cron Jobs                          │
│  WHEN: Periodically, manually                               │
│  INTERFACES: CLI, Admin Console                             │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│                  DATA CONSUMPTION PIPELINE                   │
│                  "Asking the system"                         │
├─────────────────────────────────────────────────────────────┤
│  User Query   ──▶  Context Assembly  ──▶  LLM Analysis      │
│                                                              │
│  WHO: End users, analysts                                   │
│  WHEN: On demand                                            │
│  INTERFACES: Web App, CLI, API                              │
└─────────────────────────────────────────────────────────────┘
```

## Context Assembly Pattern

For LLM-based applications, the **Context Builder** is crucial:

```python
class ContextBuilder:
    """Gathers all context the LLM needs to answer."""

    async def build(self, query: str) -> Context:
        return Context(
            # From vector DB (semantic search)
            relevant_docs=await self.kb.search(query),

            # From structured DB
            entity_data=await self.db.get_entity(query),

            # From real-time sources
            current_data=await self.market.get_current(query),

            # From historical data
            historical=await self.db.get_history(query),
        )
```

The LLM receives **assembled context**, not raw database access.

## Folder Structure Template

```
myapp/
├── core/                      # ZERO web framework imports
│   ├── ingestion/             # Data in
│   │   ├── parsers.py
│   │   ├── fetchers.py
│   │   └── embedders.py
│   │
│   ├── retrieval/             # Context assembly
│   │   ├── knowledge_base.py
│   │   └── context_builder.py
│   │
│   ├── agents/                # LLM reasoning
│   │   ├── analyst.py
│   │   └── prompts/
│   │
│   ├── tools/                 # Capabilities
│   │   ├── calculator.py
│   │   └── data_fetcher.py
│   │
│   ├── providers/             # External service adapters
│   │   ├── llm.py             # Claude, GPT, etc.
│   │   └── data.py            # Market data sources
│   │
│   └── engine.py              # Main orchestrator
│
├── api/                       # Thin FastAPI wrapper
│   ├── main.py
│   └── routes/
│
├── cli/                       # Thin CLI wrapper
│   └── main.py
│
├── sdk/                       # Client library
│   └── client.py
│
└── web/                       # React/Next.js (can be separate repo)
```

## Real-World Example: OpenCode

OpenCode demonstrates this pattern at scale:

| Layer | Package | Lines | Description |
|-------|---------|-------|-------------|
| Core | `packages/opencode` | 70K | Agent loop, tools, providers |
| SDK | `packages/sdk` | 17K | TypeScript client |
| TUI | `packages/app` | 35K | Ink/React terminal UI |
| Web | `packages/web` | 6K | Browser interface |
| Admin | `packages/console` | 20K | Admin dashboard |

The 70K-line core has **zero knowledge** of Ink or React. The 35K-line TUI is just one possible consumer.

## Mental Model Shifts

| Old Thinking | New Thinking |
|--------------|--------------|
| "Building a web app" | "Building an engine with a web interface" |
| "API routes contain logic" | "API routes just translate HTTP to core calls" |
| "Frontend calls database" | "Frontend calls SDK, SDK calls API, API calls core" |
| "Start with UI mockups" | "Start with CLI commands" |
| "One codebase = one interface" | "One core = many interfaces" |

## Validation Checklist

Before writing any UI code, verify:

- [ ] Can I run `core/` without any web server?
- [ ] Can I write tests for `core/` without mocking HTTP?
- [ ] Does `core/` have zero imports from web frameworks?
- [ ] Can I describe my CLI commands before my API routes?
- [ ] Is my API route handler < 10 lines (just translating HTTP)?

## Summary

1. **Design core first** — as if you're building a library
2. **Write CLI commands** — before any web routes
3. **Keep API thin** — just HTTP translation
4. **Separate pipelines** — ingestion vs consumption
5. **Build context explicitly** — don't let LLM access raw data

The UI is just a window into your engine. Build the engine first.

---

*Written by Claude (Opus 4.5) | 2026-01-27 PST*
