# OpenCode Serverless Functions Deep Dive

[TOC]

A comprehensive study of the cloud backend powering OpenCode's collaborative features.

---

## Overview

The serverless functions package provides the cloud backend for:
- **Session sharing** - Create shareable links
- **Real-time sync** - WebSocket-based live updates
- **GitHub integration** - OAuth token exchange for CI/CD

**Stack:** Cloudflare Workers + Durable Objects + R2 Storage + SST

---

## Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                    SERVERLESS STACK                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  Deployment (SST)                                               │
│  ├── Cloudflare Worker (api.ts)                                 │
│  ├── Durable Objects (SyncServer)                               │
│  ├── R2 Storage (Bucket)                                        │
│  └── Custom Domain (api.opencode.ai)                            │
│                                                                  │
│  Application (Hono)                                             │
│  ├── Share Management Routes                                    │
│  ├── GitHub OAuth Routes                                        │
│  └── WebSocket Real-time Sync                                   │
│                                                                  │
│  Data Layer                                                     │
│  ├── Durable Object Storage (in-memory)                         │
│  ├── R2 Bucket (persistent JSON)                                │
│  └── KV Namespace (auth tokens)                                 │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## SST Configuration

**File:** `sst.config.ts`

```typescript
export default $config({
  app(input) {
    return {
      name: "opencode",
      home: "cloudflare",
      providers: {
        stripe: { apiKey: process.env.STRIPE_SECRET_KEY },
        planetscale: "0.4.1",
      },
    }
  },
})
```

**API Worker:**

```typescript
export const api = new sst.cloudflare.Worker("Api", {
  domain: `api.${domain}`,
  handler: "packages/function/src/api.ts",
  link: [bucket, GITHUB_APP_ID, GITHUB_APP_PRIVATE_KEY],
  transform: {
    worker: (args) => {
      args.bindings = [
        ...bindings,
        {
          name: "SYNC_SERVER",
          type: "durable_object_namespace",
          className: "SyncServer",
        },
      ]
    },
  },
})
```

---

## Domain Routing

```typescript
export const domain = (() => {
  if ($app.stage === "production") return "opencode.ai"
  if ($app.stage === "dev") return "dev.opencode.ai"
  return `${$app.stage}.dev.opencode.ai`
})()

export const shortDomain = (() => {
  if ($app.stage === "production") return "opncd.ai"
  if ($app.stage === "dev") return "dev.opncd.ai"
  return `${$app.stage}.dev.opncd.ai`
})()
```

---

## Durable Object: SyncServer

One instance per share, maintains state and WebSocket connections:

```typescript
export class SyncServer extends DurableObject<Env> {
  // Create share with secret
  async share(sessionID: string): Promise<string> {
    const secret = crypto.randomUUID()
    await this.ctx.storage.put("secret", secret)
    await this.ctx.storage.put("sessionID", sessionID)
    return secret
  }

  // Publish update to all subscribers
  async publish(key: string, content: any) {
    // Store in Durable Object
    await this.ctx.storage.put(key, content)

    // Store in R2 for persistence
    await this.env.Bucket.put(`share/${key}.json`, JSON.stringify(content))

    // Broadcast to WebSocket clients
    this.ctx.getWebSockets().forEach(ws => {
      ws.send(JSON.stringify({ key, content }))
    })
  }

  // Get all session data
  async getData() {
    const entries = await this.ctx.storage.list()
    return Array.from(entries)
      .filter(([key]) => key.startsWith("session/"))
      .map(([key, content]) => ({ key, content }))
  }

  // Delete share
  async clear() {
    await this.ctx.storage.deleteAll()
    // Delete R2 objects...
  }
}
```

---

## API Endpoints

### Share Management

#### POST /share_create

Create a new share:

```typescript
.post("/share_create", async (c) => {
  const { sessionID } = await c.req.json()
  const short = SyncServer.shortName(sessionID)  // Last 8 chars
  const id = c.env.SYNC_SERVER.idFromName(short)
  const stub = c.env.SYNC_SERVER.get(id)
  const secret = await stub.share(sessionID)

  return c.json({
    secret,
    url: `https://${c.env.WEB_DOMAIN}/s/${short}`
  })
})
```

#### POST /share_sync

Push incremental updates:

```typescript
.post("/share_sync", async (c) => {
  const { sessionID, secret, key, content } = await c.req.json()
  const stub = c.env.SYNC_SERVER.get(...)

  await stub.assertSecret(secret)  // Verify ownership
  await stub.publish(key, content) // Broadcast

  return c.json({})
})
```

**Key Formats:**
- `session/info/{sessionID}` - Session metadata
- `session/message/{sessionID}/{messageID}` - Message
- `session/part/{sessionID}/{messageID}/{partID}` - Part

#### GET /share_poll

WebSocket subscription:

```typescript
.get("/share_poll", async (c) => {
  if (c.req.header("Upgrade") !== "websocket") {
    return c.text("WebSocket required", { status: 426 })
  }

  const id = c.req.query("id")
  const stub = c.env.SYNC_SERVER.get(...)

  return stub.fetch(c.req.raw)  // Delegate to DO
})
```

#### GET /share_data

HTTP polling fallback:

```typescript
.get("/share_data", async (c) => {
  const stub = c.env.SYNC_SERVER.get(...)
  const data = await stub.getData()

  // Reorganize by type
  return c.json({ info, messages })
})
```

#### POST /share_delete

Delete share:

```typescript
.post("/share_delete", async (c) => {
  const { sessionID, secret } = await c.req.json()
  const stub = c.env.SYNC_SERVER.get(...)

  await stub.assertSecret(secret)
  await stub.clear()

  return c.json({})
})
```

---

### GitHub Integration

#### POST /exchange_github_app_token

Exchange GitHub Actions OIDC token for installation token:

```typescript
.post("/exchange_github_app_token", async (c) => {
  // 1. Get OIDC token from header
  const token = c.req.header("Authorization")?.replace(/^Bearer /, "")

  // 2. Verify with GitHub JWKS
  const JWKS = createRemoteJWKSet(new URL(JWKS_URL))
  const { payload } = await jwtVerify(token, JWKS, {
    issuer: "https://token.actions.githubusercontent.com",
    audience: "opencode-github-action",
  })

  // 3. Parse repo from subject
  const [owner, repo] = payload.sub.split(":")[1].split("/")

  // 4. Create GitHub App JWT
  const auth = createAppAuth({
    appId: Resource.GITHUB_APP_ID.value,
    privateKey: Resource.GITHUB_APP_PRIVATE_KEY.value,
  })

  // 5. Get installation token
  const { data: installation } = await octokit.apps.getRepoInstallation({
    owner, repo,
  })

  const installationAuth = await auth({
    type: "installation",
    installationId: installation.id,
  })

  return c.json({ token: installationAuth.token })
})
```

---

## Data Flow

### Share Creation & Sync

```
Client                      API Worker              Durable Object
  │                              │                        │
  │ POST /share_create           │                        │
  ├─────────────────────────────>│                        │
  │                              │ share()                │
  │                              ├───────────────────────>│
  │<─────────────────────────────┤<───────────────────────┤
  │ { secret, url }              │                        │
  │                              │                        │
  │ POST /share_sync             │                        │
  ├─────────────────────────────>│                        │
  │ { secret, key, content }     │ publish()              │
  │                              ├───────────────────────>│
  │                              │ → store + broadcast    │
  │<─────────────────────────────┤<───────────────────────┤
```

### WebSocket Real-Time

```
Web Viewer                  API Worker              Durable Object
  │                              │                        │
  │ GET /share_poll (WS)         │                        │
  ├─────────────────────────────>│ fetch()                │
  │                              ├───────────────────────>│
  │<─────────────────────────────┤<── send(all data) ─────┤
  │                              │                        │
  │ [listening...]               │                        │
  │                              │ publish() from client  │
  │                              │───────────────────────>│
  │<─────────────────────────────┤<── broadcast ──────────┤
  │ { key, content }             │                        │
```

---

## Storage Strategy

### Double Storage (DO + R2)

```typescript
// Fast (memory)
await this.ctx.storage.put(key, content)

// Persistent
await this.env.Bucket.put(`share/${key}.json`, JSON.stringify(content))
```

**Why both?**
- DO storage is fast for broadcasting
- R2 is durable across restarts
- R2 allows replay/audit

### Storage Structure

```
Durable Object:
├── "secret" → UUID
├── "sessionID" → Original ID
├── "session/info/{id}" → Metadata
├── "session/message/{id}/{msgId}" → Message
└── "session/part/{id}/{msgId}/{partId}" → Part

R2 Bucket:
└── share/{key}.json → Persistent backup
```

---

## Authentication

### Share Secret

```typescript
// Client holds secret from creation
const { secret, url } = await createShare(sessionID)

// Every write includes secret
await sync({ sessionID, secret, key, content })

// Server validates
await stub.assertSecret(secret)  // throws if mismatch
```

### GitHub App Flow

```
GitHub Actions  ─OIDC Token─>  API Worker  ─App JWT─>  GitHub API
(Untrusted)                    (Verifies)              (Trusted)
      │                             │                       │
      └── Verify JWKS Signature ────┘                       │
                                                            │
      ┌──── Create Installation Token ──────────────────────┘
      │
      └──> Scoped to: repo, read/write
```

---

## Design Patterns

| Pattern | Purpose |
|---------|---------|
| **DO per Share** | Isolated state per share |
| **Double Storage** | Speed + durability |
| **Batched Sync** | Client batches updates (1s window) |
| **Named DOs** | Deterministic IDs from session |
| **Secret Auth** | Simple ownership verification |

---

## Environment & Secrets

```typescript
// Secrets (SST)
ADMIN_SECRET              // Admin operations
GITHUB_APP_ID             // GitHub App ID
GITHUB_APP_PRIVATE_KEY    // GitHub App key
STRIPE_SECRET_KEY         // Payments

// Cloudflare Bindings
SYNC_SERVER               // Durable Object namespace
Bucket                    // R2 storage
AuthStorage               // KV namespace
```

---

## Staging Strategy

| Stage | Domain |
|-------|--------|
| production | api.opencode.ai |
| dev | api.dev.opencode.ai |
| staging | staging.dev.opencode.ai |
| ephemeral | {stage}.dev.opencode.ai |

---

## Key Files

| File | Purpose |
|------|---------|
| `sst.config.ts` | SST root config |
| `infra/app.ts` | API Worker deployment |
| `packages/function/src/api.ts` | Main handler |
| `packages/opencode/src/share/share-next.ts` | Client sync |

---

## Client Integration

**File:** `packages/opencode/src/share/share-next.ts`

```typescript
// 1-second batching window
async function sync(sessionID: string, data: Data[]) {
  // Collect updates
  // Timer fires after 1s
  // Single POST with all data
  await fetch(`${url}/api/share/${id}/sync`, {
    method: "POST",
    body: JSON.stringify({ secret, data }),
  })
}
```

---

*Written by Claude (Opus 4.5) | 2026-01-22 16:45 PST*
