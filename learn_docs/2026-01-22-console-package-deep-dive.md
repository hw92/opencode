# OpenCode Console Package Deep Dive

[TOC]

A comprehensive study of the admin dashboard for cloud management.

---

## Overview

The Console package is a **multi-tenant admin dashboard** for managing OpenCode cloud features. Built on SolidStart with Drizzle ORM.

**Tech Stack:**
- Frontend: SolidStart + Solid.js
- Backend: Hono + Cloudflare Workers
- Database: MySQL via Drizzle ORM
- Auth: OpenAuth (GitHub, Google)
- Payments: Stripe

---

## Architecture

```
packages/console/
├── app/                    # SolidJS frontend
│   ├── src/routes/        # Page routing
│   ├── src/component/     # UI components
│   └── src/context/       # Auth, sessions
├── core/                   # Business logic
│   ├── src/schema/        # Drizzle tables
│   └── src/               # Services
├── function/              # Backend handlers
│   └── src/auth.ts       # OAuth issuer
├── resource/              # Cloud config
└── mail/                  # Email templates
```

---

## Feature Inventory

### Workspace Management

```typescript
Workspace.create({ name })   // Creates admin user + billing
Workspace.update({ name })   // Update name
Workspace.remove()           // Soft delete
```

### User & Team Management

```typescript
User.list()                              // List workspace users
User.invite({ email, role, monthlyLimit }) // Invite + send email
User.update({ id, role, monthlyLimit })    // Update permissions
User.remove(id)                          // Soft delete
User.joinInvitedWorkspaces()            // Accept invitations
```

**Roles:**
- **Admin**: Manage models, members, billing
- **Member**: Only generate own API keys

### API Key Management

```typescript
Key.list()                    // List keys (filtered by role)
Key.create({ userID, name }) // Generate: sk-<64 chars>
Key.remove({ id })            // Delete key
```

### Billing & Payment

```typescript
Billing.get()                    // Get workspace billing
Billing.reload()                 // Auto-reload balance
Billing.grantCredit(workspaceID, amount)
Billing.generateCheckoutUrl({ amount })
Billing.generateSessionUrl({ returnUrl })
Billing.setMonthlyLimit(amount)
```

**Features:**
- Stripe integration
- Auto-reload on low balance
- Monthly spending limits
- Payment history
- Invoice tracking

---

## Database Schema

### Core Tables

| Table | Purpose | Key Fields |
|-------|---------|------------|
| `auth` | OAuth identity | provider, subject, accountID |
| `user` | Workspace users | email, role, monthlyLimit |
| `workspace` | Container | name, slug |
| `key` | API keys | name, key, userID |
| `billing` | Workspace billing | balance, customerID |
| `payment` | Payment records | amount, invoiceID |
| `usage` | Token tracking | model, tokens, cost |

### Multi-Tenancy

```typescript
// All tables have composite primary key
{ workspaceID, id }

// Soft deletes
timeDeleted: timestamp

// Workspace isolation in queries
WHERE workspaceID = ?
```

---

## Authentication Flow

```
User Browser
    ↓
/auth/authorize (OAuth2)
    ↓
OpenAuth Provider (GitHub/Google)
    ↓
/auth/callback
    ↓
Validate email, create account
    ↓
Join invited workspaces
    ↓
Create default workspace if none
    ↓
Session cookie (365 days)
    ↓
Redirect to dashboard
```

### Auth Context

```typescript
// Frontend
const { session } = useAuthSession()

// Actor model
Actor.assertAdmin()        // Throws if not admin
Actor.workspace()          // Current workspace
Actor.userID()             // Current user
Actor.userRole()           // "admin" | "member"
```

### Supported Providers

- GitHub OAuth
- Google OIDC
- Email verification required

---

## Admin UI Routes

| Route | Purpose | Access |
|-------|---------|--------|
| `/workspace/[id]` | Dashboard | Admin |
| `/workspace/[id]/settings` | Settings | Admin |
| `/workspace/[id]/members` | Team management | Admin |
| `/workspace/[id]/keys` | API keys | Admin/Member |
| `/workspace/[id]/billing` | Billing | Admin |

### Member Management UI

```tsx
// Table with: email, role, limit, status
// Inline edit for role changes
// Invite workflow with validation
// Delete confirmation
```

### API Key UI

```tsx
// Create with custom name
// Masked display: sk-abc...xyz
// Copy-to-clipboard
// Last used tracking
```

### Billing UI

```tsx
// Balance display
// Enable billing (Stripe checkout)
// Reload configuration
// Monthly limits
// Payment history
```

---

## API Routes

### Backend Functions

```typescript
// Auth
GET /auth/authorize
GET /auth/[...callback]
GET /auth/status
GET /auth/logout

// Workspace
POST /workspace/create
PATCH /workspace/update
DELETE /workspace/remove

// Users
GET /users
POST /users/invite
PATCH /users/update
DELETE /users/remove

// Keys
GET /keys
POST /keys/create
DELETE /keys/remove

// Billing
GET /billing
POST /billing/checkout
POST /billing/reload
```

---

## Security & Authorization

### Role-Based Access

```typescript
// Admin-only operations
Actor.assertAdmin()

// Workspace isolation
const users = await User.list()  // Auto-filtered by workspaceID

// User-scoped operations
if (Actor.userRole() !== "admin") {
  // Can only see own keys
}
```

### Data Protection

- API keys stored securely
- Session tokens in HttpOnly cookies
- Email verification required
- Monthly spending limits
- Soft deletes for audit

---

## Frontend Patterns

### Query/Action Pattern

```typescript
// Server query
const listMembers = query(async (workspaceID) => {
  return withActor(async () => ({
    members: await User.list(),
    actorRole: Actor.userRole(),
  }), workspaceID)
})

// Server action
const inviteMember = action(async (form: FormData) => {
  return json(
    await withActor(
      () => User.invite({ email, role }),
      workspaceID
    ),
    { revalidate: listMembers.key }
  )
})
```

### Component Structure

```tsx
function MemberSection() {
  const data = createAsync(() => listMembers(workspaceID))

  return (
    <Show when={data()}>
      {(members) => (
        <For each={members().members}>
          {(member) => <MemberRow member={member} />}
        </For>
      )}
    </Show>
  )
}
```

---

## Integrations

### Stripe

- Checkout for adding credits
- Billing portal for payment methods
- Invoice with line items
- Auto-reload on low balance
- Tax ID collection

### Email (AWS SES)

- JSX-based templates
- Invite emails
- Transactional notifications

### OpenAuth

- GitHub OAuth
- Google OIDC
- Provider linking

---

## Deployment

### Environment

```bash
# Auth
GITHUB_CLIENT_ID
GITHUB_CLIENT_SECRET
GOOGLE_CLIENT_ID

# Stripe
STRIPE_SECRET_KEY
STRIPE_WEBHOOK_SECRET

# Database
DATABASE_URL
```

### Infrastructure (SST)

- Cloudflare Workers for API
- MySQL for database
- AWS SES for email
- Stripe for payments

---

## Key Design Patterns

| Pattern | Purpose |
|---------|---------|
| **Multi-Tenancy** | workspaceID partitioning |
| **Actor Model** | Context-aware authorization |
| **Soft Deletes** | Audit trails |
| **Query/Action** | Server-side data fetching |
| **RBAC** | Role-based access control |

---

## Key Files

| File | Purpose |
|------|---------|
| `app/src/routes/` | Page routing |
| `core/src/schema/` | Database tables |
| `core/src/user.ts` | User management |
| `core/src/billing.ts` | Payment logic |
| `function/src/auth.ts` | OAuth handler |

---

*Written by Claude (Opus 4.5) | 2026-01-22 17:00 PST*
