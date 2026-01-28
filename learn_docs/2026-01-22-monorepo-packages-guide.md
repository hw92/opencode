# OpenCode Monorepo Packages Guide

An index of all packages beyond the core `opencode` package.

---

## Package Map

```
packages/
│
├── opencode/        ★ CORE ENGINE (fully documented)
│                      The CLI, TUI, server, execution loop, tools, etc.
│                      See: 15 deep-dive docs in learn_docs/
│
├── ─────────────────── CLIENT APPLICATIONS ───────────────────
│
├── app/             Tauri desktop app wrapper
├── desktop/         Desktop-specific platform code
├── web/             Marketing website (opencode.ai)
├── console/         Admin console for cloud management
│
├── ─────────────────── CLOUD INFRASTRUCTURE ───────────────────
│
├── enterprise/      Enterprise/team features
├── function/        Serverless API functions
├── slack/           Slack bot integration
├── plugin/          Cloud-hosted plugin system
├── script/          Deployment & build scripts
│
├── ─────────────────── SHARED LIBRARIES ───────────────────
│
├── sdk/             Client SDK for OpenCode API
├── ui/              Shared UI components
├── util/            Shared utility functions
│
├── ─────────────────── AUXILIARY ───────────────────
│
├── docs/            Documentation site
├── extensions/      IDE extensions
└── identity/        Brand assets
```

---

## Client Applications

### `app/` - Tauri Desktop App

**Purpose:** Wraps the core TUI in a native desktop application.

**Tech Stack:**
- Tauri (Rust-based desktop framework)
- React for UI components
- Embeds the `opencode` TUI

**Key Files:**
```
app/
├── src/
│   ├── App.tsx          # Main React component
│   └── main.tsx         # Entry point
├── src-tauri/
│   ├── src/main.rs      # Rust backend
│   └── tauri.conf.json  # Tauri config
└── index.html
```

**Relationship to Core:** Spawns `opencode` as a child process and displays TUI in a native window.

---

### `desktop/` - Desktop Platform Code

**Purpose:** Platform-specific code for the desktop app.

**Key Features:**
- Native file dialogs
- System tray integration
- Auto-update mechanism
- Platform shortcuts

---

### `web/` - Marketing Website

**Purpose:** The public website at opencode.ai.

**Tech Stack:**
- Astro (static site generator)
- MDX for content

**Key Files:**
```
web/
├── src/
│   ├── pages/           # Route pages
│   ├── components/      # UI components
│   └── content/         # MDX content
├── astro.config.mjs
└── public/              # Static assets
```

---

### `console/` - Admin Console

**Purpose:** Web-based admin dashboard for managing OpenCode cloud features.

**Structure:**
```
console/
├── app/                 # Frontend application
├── core/                # Shared logic
├── function/            # Backend functions
├── mail/                # Email templates
└── resource/            # Cloud resources
```

---

## Cloud Infrastructure

### `enterprise/` - Enterprise Features

**Purpose:** Team and organization management for enterprise customers.

**Features:**
- Team workspaces
- SSO integration
- Usage analytics
- Admin controls

**Tech:** SST (Serverless Stack) on AWS

---

### `function/` - Serverless Functions

**Purpose:** Backend API endpoints deployed as AWS Lambda functions.

**Key Endpoints:**
- Share sync API (`/share_create`, `/share_sync`)
- Plugin registry
- Usage tracking
- Webhook handlers

**Tech Stack:**
- SST (Serverless Stack)
- AWS Lambda
- API Gateway

**Key Files:**
```
function/
├── src/
│   ├── share.ts         # Share functionality
│   ├── plugin.ts        # Plugin registry
│   └── ...
└── sst-env.d.ts         # SST type definitions
```

---

### `slack/` - Slack Integration

**Purpose:** Slack bot for using OpenCode within Slack.

**Features:**
- Chat with AI in Slack channels
- Run commands via slash commands
- Thread-based conversations

**Tech:** SST + Slack Bolt SDK

---

### `plugin/` - Cloud Plugin System

**Purpose:** Hosts and serves plugins for the plugin registry.

**Features:**
- Plugin discovery API
- Version management
- Plugin validation

---

### `script/` - Deployment Scripts

**Purpose:** Build, deploy, and maintenance scripts.

**Common Scripts:**
- Release automation
- Database migrations
- Infrastructure updates

---

## Shared Libraries

### `sdk/` - Client SDK

**Purpose:** TypeScript/JavaScript SDK for interacting with OpenCode API.

**Structure:**
```
sdk/
├── js/                  # JavaScript SDK
│   ├── src/
│   │   ├── client.ts    # API client
│   │   ├── types.ts     # Type definitions
│   │   └── v2/          # V2 API
│   └── package.json
└── openapi.json         # OpenAPI specification
```

**Usage:**
```typescript
import { OpenCode } from "@opencode-ai/sdk"

const client = new OpenCode({ baseUrl: "..." })
const sessions = await client.session.list()
```

---

### `ui/` - Shared UI Components

**Purpose:** Reusable React components shared across apps.

**Components:**
- Message display
- Code blocks
- File diffs
- Theme system

---

### `util/` - Shared Utilities

**Purpose:** Common utility functions used across packages.

**Utilities:**
- String manipulation
- Date formatting
- Type helpers
- Validation helpers

---

## Auxiliary

### `docs/` - Documentation Site

**Purpose:** Official documentation hosted on Mintlify.

**Structure:**
```
docs/
├── ai-tools/            # AI tools documentation
├── essentials/          # Getting started guides
├── development.mdx      # Development guide
└── docs.json            # Mintlify config
```

---

### `extensions/` - IDE Extensions

**Purpose:** Extensions for various IDEs.

**Current:**
- `zed/` - Zed editor extension

**Potential Future:**
- VS Code extension
- JetBrains plugin
- Neovim plugin

---

### `identity/` - Brand Assets

**Purpose:** Official logos and brand materials.

**Assets:**
```
identity/
├── mark-512x512.png     # App icon
├── mark-512x512-light.png
├── mark-192x192.png
├── mark-96x96.png
└── mark-light.svg       # Vector logo
```

---

## Architecture Relationships

```
                    ┌─────────────┐
                    │   Users     │
                    └──────┬──────┘
                           │
         ┌─────────────────┼─────────────────┐
         ▼                 ▼                 ▼
   ┌───────────┐    ┌───────────┐    ┌───────────┐
   │  app/     │    │   web/    │    │  slack/   │
   │ (Desktop) │    │ (Website) │    │  (Bot)    │
   └─────┬─────┘    └───────────┘    └─────┬─────┘
         │                                 │
         ▼                                 ▼
   ┌───────────┐                    ┌───────────┐
   │ opencode/ │◄───────────────────│ function/ │
   │  (Core)   │                    │  (API)    │
   └─────┬─────┘                    └─────┬─────┘
         │                                 │
         ├────────────────┬────────────────┤
         ▼                ▼                ▼
   ┌───────────┐   ┌───────────┐   ┌───────────┐
   │   sdk/    │   │    ui/    │   │   util/   │
   │ (Client)  │   │(Components)│  │ (Helpers) │
   └───────────┘   └───────────┘   └───────────┘
```

---

## What to Study Next

| If you want to... | Study... |
|-------------------|----------|
| Build a desktop app | `app/`, `desktop/` |
| Deploy cloud features | `function/`, `enterprise/` |
| Create an IDE extension | `extensions/zed/` |
| Use OpenCode API | `sdk/js/` |
| Add shared components | `ui/` |
| Understand deployment | `script/`, SST configs |

---

## Tech Stack Summary

| Layer | Technology |
|-------|------------|
| Core Engine | Bun, TypeScript, Ink (React) |
| Desktop | Tauri (Rust), React |
| Website | Astro, MDX |
| Cloud | SST, AWS Lambda, API Gateway |
| SDK | TypeScript |
| Docs | Mintlify |

---

*Written by Claude (Opus 4.5) | 2026-01-22 16:30 PST*
