# OpenCode Enterprise Package Deep Dive

[TOC]

A comprehensive study of the enterprise/team features for multi-tenant deployments.

---

## Overview

The Enterprise package provides the **session sharing and replay system** with multi-tenant team features. Built on SolidStart with Hono API.

**Tech Stack:**
- Frontend: SolidStart + Solid.js
- API: Hono with OpenAPI support
- Storage: S3/R2 abstraction
- Deployment: Node.js or Cloudflare Workers

---

## Architecture

```
packages/enterprise/
├── src/
│   ├── app.tsx                 # SolidStart root
│   ├── core/
│   │   ├── share.ts           # Share system
│   │   └── storage.ts         # S3/R2 abstraction
│   ├── routes/
│   │   ├── share/[shareID].tsx # Share viewer UI
│   │   └── api/[...path].ts   # Hono API
│   └── global.d.ts
├── vite.config.ts
└── package.json
```

---

## Share System

### Data Models

```typescript
// Share metadata
Share.Info = {
  id: string        // Last 8 chars of sessionID
  secret: string    // UUID for authentication
  sessionID: string // Full session ID
}

// Share data (discriminated union)
Share.Data =
  | { type: "session"; data: Session }
  | { type: "message"; data: Message }
  | { type: "part"; data: Part }
  | { type: "session_diff"; data: FileDiff[] }
  | { type: "model"; data: Model[] }
```

### Core Operations

```typescript
// Create share
Share.create(sessionID)
// → { id, secret, url }

// Sync data (batched)
Share.sync(share, data[])
// Validates secret, writes events

// Get share data
Share.data(shareID)
// → Compacted data with deduplication

// Delete share
Share.remove(id, secret)
// Cascades delete all data
```

### Storage Structure

```
share/[shareID].json              # Metadata
share_event/[shareID]/[ID].json   # Event batches
share_compaction/[shareID].json   # Compacted state
```

---

## Storage Abstraction

### Adapter Interface

```typescript
interface Adapter {
  read(path: string): Promise<string | undefined>
  write(path: string, value: string): Promise<void>
  remove(path: string): Promise<void>
  list(options?: { prefix, limit, after, before }): Promise<string[]>
}
```

### S3/R2 Support

```typescript
// S3
https://s3.${region}.amazonaws.com/${bucket}/

// R2
https://${accountId}.r2.cloudflarestorage.com/${bucket}/
```

### Environment Variables

```bash
OPENCODE_STORAGE_ADAPTER=s3|r2
OPENCODE_STORAGE_ACCESS_KEY_ID=...
OPENCODE_STORAGE_SECRET_ACCESS_KEY=...
OPENCODE_STORAGE_BUCKET=...
OPENCODE_STORAGE_REGION=us-east-1        # S3 only
OPENCODE_STORAGE_ACCOUNT_ID=...          # R2 only
```

---

## API Endpoints

### POST /api/share

Create a new share:

```typescript
// Input
{ sessionID: string }

// Output
{ id: string, secret: string, url: string }
```

### POST /api/share/:shareID/sync

Sync session data:

```typescript
// Input
{ secret: string, data: Share.Data[] }

// Output
{}
```

### GET /api/share/:shareID/data

Get share data (public, no auth):

```typescript
// Output
Share.Data[]
```

### DELETE /api/share/:shareID

Delete share:

```typescript
// Input
{ secret: string }
```

---

## Share Viewer UI

**Route:** `/share/[shareID]`

### Features

- Server-side data preloading
- Responsive layout (mobile tabs vs desktop split)
- Code diff viewer (unified/split)
- Session turn-by-turn replay
- Social card generation for sharing

### Layout Options

**Wide (no diffs):**
- Single column with session turns
- Full width content

**Split (with diffs):**
- Left: Session messages
- Right: Code diff viewer
- Mobile: Tabbed interface

### Social Cards

```
URL: https://social-cards.sst.dev/opencode-share/
Parameters: title (base64), model, version, id
```

---

## Team Features (via Console)

The enterprise package integrates with console/core for team features:

### Database Models

```typescript
// Workspace multi-tenancy
Workspace {
  id: string
  slug: string
  name: string
}

// User with roles
User {
  workspaceID: string
  accountID: string
  role: "admin" | "member"
  monthlyLimit: number
  monthlyUsage: number
}

// Authentication
Auth {
  provider: "email" | "github" | "google"
  subject: string
  accountID: string
}
```

### User Management

- Workspace creation with admin user
- User invitation system
- Role-based access control
- Monthly usage quotas
- API key generation

---

## Data Compaction

Optimizes storage for large shares:

```typescript
// Binary search for deduplication
const result = Binary.search(compaction.data, id, key)

if (result.found) {
  compaction.data[result.index] = item  // Update
} else {
  compaction.data.splice(result.index, 0, item)  // Insert
}
```

**Trigger:** When events exceed threshold, compact into single state.

---

## Authentication

### Share Secret Pattern

```typescript
// Create returns secret
const { secret, url } = await Share.create(sessionID)

// All writes require secret
await Share.sync({ sessionID, secret, data })

// Reads are public (no auth)
const data = await Share.data(shareID)
```

### Team Auth (SSO)

- GitHub OAuth
- Google OIDC
- Email verification
- Provider linking (same email = linked)

---

## Deployment

### Targets

```typescript
// Node.js (default)
bun run build

// Cloudflare Workers
OPENCODE_DEPLOYMENT_TARGET=cloudflare bun run build
```

### Infrastructure (SST)

```typescript
// Secrets
ADMIN_SECRET
STRIPE_SECRET_KEY
GITHUB_APP_*
GOOGLE_CLIENT_ID

// Resources
EnterpriseStorage: R2Bucket
Database: PostgreSQL
```

---

## Key Design Patterns

| Pattern | Purpose |
|---------|---------|
| **Event Sourcing** | Track all changes as events |
| **Compaction** | Merge events for performance |
| **Secret Auth** | Simple ownership verification |
| **S3/R2 Abstraction** | Cloud-agnostic storage |
| **SSR + Client** | Fast initial load + interactivity |

---

## Key Files

| File | Purpose |
|------|---------|
| `src/core/share.ts` | Share system logic |
| `src/core/storage.ts` | S3/R2 abstraction |
| `src/routes/api/[...path].ts` | Hono API |
| `src/routes/share/[shareID].tsx` | Share viewer |

---

*Written by Claude (Opus 4.5) | 2026-01-22 17:00 PST*
