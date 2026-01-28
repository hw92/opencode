# OpenCode Tauri Desktop App Deep Dive

[TOC]

A comprehensive study of how OpenCode is packaged as a native desktop application using Tauri.

---

## Overview

The desktop app wraps OpenCode in a native window using **Tauri v2**, providing:
- Native file dialogs
- System notifications
- Auto-updates
- Window state persistence
- Sidecar process management

---

## Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                    TAURI DESKTOP APP                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ┌────────────────────────────────────────────────────────────┐ │
│  │  Frontend (packages/app)                                    │ │
│  │  • Solid.js + React                                         │ │
│  │  • Platform abstraction layer                               │ │
│  │  • Shared UI components                                     │ │
│  └─────────────────────────────┬──────────────────────────────┘ │
│                                │                                 │
│                         IPC (invoke)                             │
│                                │                                 │
│  ┌─────────────────────────────▼──────────────────────────────┐ │
│  │  Rust Backend (src-tauri)                                   │ │
│  │  • Sidecar process management                               │ │
│  │  • Native dialogs, notifications                            │ │
│  │  • Auto-updater                                             │ │
│  └─────────────────────────────┬──────────────────────────────┘ │
│                                │                                 │
│                          spawn/HTTP                              │
│                                │                                 │
│  ┌─────────────────────────────▼──────────────────────────────┐ │
│  │  OpenCode Sidecar (opencode serve)                          │ │
│  │  • HTTP API server                                          │ │
│  │  • SSE event stream                                         │ │
│  │  • Full CLI functionality                                   │ │
│  └────────────────────────────────────────────────────────────┘ │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

---

## Package Structure

```
packages/
├── app/                          # Shared web UI
│   ├── src/
│   │   ├── app.tsx              # Root component
│   │   ├── context/platform.tsx # Platform abstraction
│   │   └── pages/               # UI pages
│   └── package.json
│
└── desktop/                      # Tauri wrapper
    ├── src/
    │   ├── index.tsx            # Desktop platform impl
    │   ├── cli.ts               # CLI installation
    │   └── updater.ts           # Auto-update logic
    ├── src-tauri/
    │   ├── src/
    │   │   ├── main.rs          # Entry point
    │   │   ├── lib.rs           # Core logic
    │   │   └── cli.rs           # Sidecar management
    │   ├── Cargo.toml
    │   └── tauri.conf.json
    └── vite.config.ts
```

---

## Sidecar Process Management

### Spawning OpenCode Server

```rust
// lib.rs
pub fn spawn_sidecar(app: &AppHandle, port: u32, password: &str) -> CommandChild {
    let (mut rx, child) = cli::create_command(app, format!("serve --port {port}"))
        .env("OPENCODE_SERVER_PASSWORD", password)
        .spawn()
        .expect("Failed to spawn opencode");

    // Capture stdout/stderr
    tauri::async_runtime::spawn(async move {
        while let Some(event) = rx.recv().await {
            match event {
                CommandEvent::Stdout(line) => { /* log */ },
                CommandEvent::Stderr(line) => { /* log */ },
                _ => {}
            }
        }
    });

    child
}
```

### Server Connection Flow

```
1. Check for custom server URL in settings
         │
         ▼
2. If not found, check config file
         │
         ▼
3. If no external server, spawn local sidecar
         │
         ▼
4. Find available port (TcpListener bind :0)
         │
         ▼
5. Generate UUID password for auth
         │
         ▼
6. Poll /global/health until ready (30s timeout)
         │
         ▼
7. Return { url, password } to frontend
```

---

## Platform Abstraction Layer

**File:** `packages/app/src/context/platform.tsx`

```typescript
export type Platform = {
  platform: "web" | "desktop"
  os?: "macos" | "windows" | "linux"

  // Native dialogs
  openDirectoryPickerDialog?(): Promise<string>
  openFilePickerDialog?(): Promise<string>
  saveFilePickerDialog?(): Promise<string>

  // System integration
  openLink(url: string): void
  notify(title: string, body?: string): Promise<void>
  restart(): Promise<void>

  // Auto-update
  checkUpdate?(): Promise<{ updateAvailable: boolean; version?: string }>
  update?(): Promise<void>

  // Storage
  storage?: (name?: string) => AsyncStorage

  // Auth-aware fetch
  fetch?: typeof fetch

  // Server config
  getDefaultServerUrl?(): Promise<string | null>
  setDefaultServerUrl?(url: string | null): Promise<void>
}
```

### Desktop Implementation

```typescript
// packages/desktop/src/index.tsx
const createPlatform = (password: Accessor<string | null>): Platform => ({
  platform: "desktop",
  os: ostype() as "macos" | "windows" | "linux",

  async openDirectoryPickerDialog(opts) {
    return open({ directory: true, ...opts })
  },

  openLink(url: string) {
    void shellOpen(url)
  },

  // Add Basic auth header
  fetch: (input, init) => {
    const pw = password()
    const headers = new Headers(init?.headers)
    if (pw) headers.append("Authorization", `Basic ${btoa(`opencode:${pw}`)}`)
    return tauriFetch(input, { ...init, headers })
  },

  checkUpdate: async () => {
    const next = await check()
    if (next) {
      await next.download()
      return { updateAvailable: true, version: next.version }
    }
    return { updateAvailable: false }
  },
})
```

---

## Tauri Commands (IPC)

```rust
#[tauri::command]
async fn ensure_server_ready(state: State<'_, ServerState>) -> Result<ServerReadyData, String> {
    // Wait for server initialization, return URL + password
}

#[tauri::command]
fn kill_sidecar(app: AppHandle) {
    // Kill CLI process on app exit
}

#[tauri::command]
fn install_cli(app: tauri::AppHandle) -> Result<String, String> {
    // Install CLI to ~/.opencode/bin/opencode
}

#[tauri::command]
async fn get_default_server_url(app: AppHandle) -> Result<Option<String>, String> {
    // Read from persistent store
}

#[tauri::command]
async fn set_default_server_url(app: AppHandle, url: Option<String>) -> Result<(), String> {
    // Write to persistent store
}
```

---

## Tauri Plugins

| Plugin | Purpose |
|--------|---------|
| `tauri-plugin-shell` | Spawn CLI processes |
| `tauri-plugin-dialog` | File dialogs |
| `tauri-plugin-updater` | Auto-update |
| `tauri-plugin-store` | Persistent storage |
| `tauri-plugin-window-state` | Save window position |
| `tauri-plugin-notification` | Desktop notifications |
| `tauri-plugin-single-instance` | Prevent duplicate windows |
| `tauri-plugin-clipboard-manager` | Clipboard operations |

---

## Server Gate Component

The app waits for backend before rendering:

```typescript
function ServerGate(props) {
  const [serverData] = createResource(() => invoke("ensure_server_ready"))

  return (
    <Show
      when={serverData()}
      fallback={
        <div class="animate-pulse">
          <Logo />
          <div>Initializing...</div>
        </div>
      }
    >
      {(data) => props.children(data)}
    </Show>
  )
}
```

---

## Platform-Specific Code

### Windows Job Object

Ensures child processes are killed on crash:

```rust
// job_object.rs
pub fn assign_pid(&self, pid: u32) -> Result<()> {
    unsafe {
        let process = OpenProcess(PROCESS_SET_QUOTA | PROCESS_TERMINATE, false, pid)?;
        AssignProcessToJobObject(self.0, process)?;
    }
}
```

### macOS Title Bar

```rust
#[cfg(target_os = "macos")]
let window_builder = window_builder
    .title_bar_style(tauri::TitleBarStyle::Overlay)
    .hidden_title(true);
```

### Linux Pinch Zoom

Custom plugin disables pinch-to-zoom:

```rust
#[cfg(target_os = "linux")]
unsafe {
    // Disconnect GtkGestureZoom signal handlers
}
```

---

## Auto-Update System

```typescript
// updater.ts
checkUpdate: async () => {
  const next = await check()  // GitHub releases
  if (next) {
    await next.download()     // Download in background
    update = next
    return { updateAvailable: true, version: next.version }
  }
  return { updateAvailable: false }
},

update: async () => {
  if (update) {
    // Kill sidecar on Windows before replacing
    if (ostype() === "windows") await invoke("kill_sidecar")
    await update.install()    // Replace and restart
  }
}
```

---

## Build Process

### Development

```bash
cd packages/desktop
bun install
bun run tauri dev
```

### Production Build

```bash
bun run tauri build
```

**Pre-build steps:**
1. Build OpenCode CLI binary
2. Copy to sidecar folder
3. Vite builds frontend
4. Rust compiles to native binary

**Outputs:**
- macOS: `.dmg`, `.app` bundle
- Windows: NSIS installer (`.exe`)
- Linux: `.deb`, `.rpm`

---

## Tauri Configuration

**File:** `src-tauri/tauri.conf.json`

```json
{
  "productName": "OpenCode",
  "identifier": "ai.opencode.desktop",
  "build": {
    "beforeDevCommand": "bun run dev",
    "devUrl": "http://localhost:1420",
    "frontendDist": "../dist"
  },
  "app": {
    "windows": [{
      "label": "main",
      "titleBarStyle": "Overlay",
      "hiddenTitle": true
    }],
    "security": { "csp": null },
    "macOSPrivateApi": true
  },
  "bundle": {
    "externalBin": ["sidecars/opencode-cli"]
  }
}
```

---

## Security Model

### Capabilities

```json
{
  "windows": ["main"],
  "permissions": [
    "core:default",
    "shell:default",
    "dialog:default",
    "http:default",
    "notification:default",
    "store:default",
    "updater:default"
  ]
}
```

### Local Server Auth

```typescript
// UUID-based password for sidecar
const password = crypto.randomUUID()

// All requests include auth header
headers.append("Authorization", `Basic ${btoa(`opencode:${password}`)}`)
```

---

## Key Design Patterns

| Pattern | Purpose |
|---------|---------|
| **Sidecar Process** | CLI runs independently |
| **Platform Abstraction** | Same code for web/desktop |
| **IPC Commands** | Type-safe Rust ↔ JS bridge |
| **Server Gate** | Wait for backend before UI |
| **Job Object** | Automatic cleanup on Windows |

---

## Key Files

| File | Purpose |
|------|---------|
| `desktop/src-tauri/src/lib.rs` | Core Rust logic |
| `desktop/src-tauri/src/cli.rs` | Sidecar management |
| `desktop/src/index.tsx` | Platform implementation |
| `app/src/context/platform.tsx` | Platform abstraction |
| `desktop/src-tauri/tauri.conf.json` | Tauri config |

---

*Written by Claude (Opus 4.5) | 2026-01-22 16:45 PST*
