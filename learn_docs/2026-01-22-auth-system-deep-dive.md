# OpenCode Auth System Deep Dive

[TOC]

A comprehensive study of OpenCode's authentication for API access, providers, and enterprise features.

---

## Overview

OpenCode supports three primary authentication methods:

1. **OAuth 2.0** - For providers supporting OAuth (GitHub, OpenAI, Anthropic)
2. **API Keys** - Direct API key authentication
3. **Well-Known** - Enterprise authentication via well-known endpoints

---

## Architecture

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        AUTHENTICATION SYSTEM                                 │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  ┌──────────────────────────────────────────────────────────────────────┐   │
│  │                      Auth Storage                                     │   │
│  │  ~/.opencode/data/auth.json (0o600 permissions)                       │   │
│  │  ~/.opencode/data/mcp-auth.json (MCP servers)                         │   │
│  └────────────────────────────────────────────────────────────────────┬─┘   │
│                                                                        │     │
│  ┌─────────────────────┬─────────────────────┬────────────────────────▼┐    │
│  │                     │                     │                         │    │
│  │  ┌───────────────┐  │  ┌───────────────┐  │  ┌───────────────┐     │    │
│  │  │ OAuth Flow    │  │  │ API Key       │  │  │ Well-Known    │     │    │
│  │  │               │  │  │               │  │  │               │     │    │
│  │  │ • PKCE        │  │  │ • Env vars    │  │  │ • Enterprise  │     │    │
│  │  │ • State param │  │  │ • Config file │  │  │ • Org config  │     │    │
│  │  │ • Device flow │  │  │ • Auth store  │  │  │ • Remote URL  │     │    │
│  │  └───────────────┘  │  └───────────────┘  │  └───────────────┘     │    │
│  │                     │                     │                         │    │
│  └─────────────────────┴─────────────────────┴─────────────────────────┘    │
│                                                                              │
│  ┌──────────────────────────────────────────────────────────────────────┐   │
│  │                      Provider Auth                                    │   │
│  │  • Anthropic (API key, OAuth)                                         │   │
│  │  • OpenAI (API key, OAuth with PKCE)                                  │   │
│  │  • GitHub Copilot (Device flow, Enterprise)                           │   │
│  │  • Azure (Managed identity, API key)                                  │   │
│  │  • Google (Service account, OAuth)                                    │   │
│  └──────────────────────────────────────────────────────────────────────┘   │
│                                                                              │
└──────────────────────────────────────────────────────────────────────────────┘
```

---

## Credential Storage

### Storage Locations

```typescript
// Main auth storage
~/.opencode/data/auth.json

// MCP-specific auth
~/.opencode/data/mcp-auth.json

// File permissions: 0o600 (owner-only)
```

### Storage Structure

```typescript
// auth.json structure
{
  "anthropic": {
    "type": "api",
    "apiKey": "sk-..."
  },
  "openai": {
    "type": "oauth",
    "accessToken": "...",
    "refreshToken": "...",
    "expiresAt": 1234567890,
    "accountID": "..."
  },
  "copilot": {
    "type": "oauth",
    "accessToken": "...",
    "enterpriseUrl": "https://github.mycompany.com"
  }
}
```

---

## OAuth Flow

### Standard OAuth (PKCE)

```
1. CLI: opencode auth login <provider>
         │
         ▼
2. Generate code_verifier and state
         │
         ▼
3. Open browser to authorization URL
   https://provider.com/oauth/authorize?
     client_id=...
     redirect_uri=...
     code_challenge=...
     state=...
         │
         ▼
4. User authorizes in browser
         │
         ▼
5. Callback to local server (port 19876)
   or manual code copy
         │
         ▼
6. Exchange code for tokens
         │
         ▼
7. Store tokens in auth.json
```

### Device Flow (GitHub Copilot)

```
1. Request device code
   POST /login/device/code
         │
         ▼
2. Display user_code to user
   "Enter code: ABCD-1234 at https://github.com/login/device"
         │
         ▼
3. Poll for access token
   POST /login/oauth/access_token
         │
         ▼
4. Store token when authorized
```

---

## Provider-Specific Auth

### Anthropic

```typescript
// API Key
{
  type: "api",
  apiKey: process.env.ANTHROPIC_API_KEY
}

// OAuth (for Claude.ai accounts)
{
  type: "oauth",
  accessToken: "...",
  refreshToken: "..."
}
```

### OpenAI / ChatGPT

```typescript
// PKCE-based OAuth
{
  type: "oauth",
  accessToken: "...",
  refreshToken: "...",
  accountID: "..."  // Extracted from JWT
}

// Automatic token refresh
async function refreshAccessToken(refreshToken: string) {
  const response = await fetch("/oauth/token", {
    method: "POST",
    body: { grant_type: "refresh_token", refresh_token: refreshToken }
  })
  return response.json()
}
```

### GitHub Copilot

```typescript
// Cloud deployment
{
  type: "oauth",
  accessToken: "ghu_..."
}

// Enterprise deployment
{
  type: "oauth",
  accessToken: "ghu_...",
  enterpriseUrl: "https://github.mycompany.com"
}

// Dynamic endpoint resolution
function getEndpoint(auth) {
  if (auth.enterpriseUrl) {
    return `${auth.enterpriseUrl}/api/v3`
  }
  return "https://api.github.com"
}
```

### Azure OpenAI

```typescript
// API Key
{
  type: "api",
  apiKey: process.env.AZURE_OPENAI_API_KEY,
  resourceName: "my-resource",
  deploymentId: "gpt-4"
}

// Managed Identity
{
  type: "managed_identity"
  // Uses DefaultAzureCredential
}
```

---

## MCP OAuth

### Storage

```typescript
// ~/.opencode/data/mcp-auth.json
{
  "server-name": {
    "accessToken": "...",
    "refreshToken": "...",
    "expiresAt": 1234567890,
    "serverUrl": "https://mcp-server.com"  // For validation
  }
}
```

### PKCE Flow

```typescript
// Generate PKCE challenge
const codeVerifier = generateCodeVerifier()
const codeChallenge = await sha256(codeVerifier)

// Store verifier for callback
await storeCodeVerifier(serverName, codeVerifier)

// Authorization URL
const authUrl = new URL(authEndpoint)
authUrl.searchParams.set("code_challenge", codeChallenge)
authUrl.searchParams.set("code_challenge_method", "S256")
authUrl.searchParams.set("state", state)
```

### Server URL Validation

```typescript
// Invalidate if server URL changed (security)
if (stored.serverUrl !== currentServerUrl) {
  await removeAuth(serverName)
  return undefined
}
```

---

## Server Authentication

### Basic Auth Middleware

```typescript
// Optional basic auth for server API
function basicAuth(username: string, password: string) {
  return async (c, next) => {
    const auth = c.req.header("Authorization")
    if (!auth?.startsWith("Basic ")) {
      return c.text("Unauthorized", 401)
    }

    const [user, pass] = atob(auth.slice(6)).split(":")
    if (user !== username || pass !== password) {
      return c.text("Unauthorized", 401)
    }

    await next()
  }
}
```

### OAuth Endpoints

```typescript
// Provider OAuth authorization
app.get("/provider/:providerID/oauth/authorize", async (c) => {
  const authUrl = buildAuthUrl(provider)
  return c.redirect(authUrl)
})

// OAuth callback
app.get("/provider/:providerID/oauth/callback", async (c) => {
  const { code, state } = c.req.query()
  const tokens = await exchangeCode(code)
  await storeAuth(providerID, tokens)
  return c.html("Authentication successful!")
})
```

---

## Enterprise Features

### Well-Known Configuration

Organizations can provide default config via well-known endpoint:

```
https://company.com/.well-known/opencode
```

```json
{
  "provider": {
    "anthropic": {
      "base_url": "https://api.anthropic.company.com"
    }
  },
  "disabled_providers": ["openai"],
  "permission": {
    "bash": "ask"
  }
}
```

### Multi-Account Support

```typescript
// Account IDs stored per provider
{
  "openai": {
    "accountID": "org-123",
    "accessToken": "..."
  }
}

// Select account at runtime
const accounts = await listAccounts(providerID)
const selected = await promptSelectAccount(accounts)
```

---

## CLI Commands

```bash
# Interactive login
opencode auth login

# Login to specific provider
opencode auth login anthropic

# List authenticated providers
opencode auth list

# Logout from provider
opencode auth logout openai

# Show current auth status
opencode auth status
```

---

## Security Implementations

### File Permissions

```typescript
// Credential files are owner-only
await Bun.write(authPath, JSON.stringify(auth), {
  mode: 0o600
})
```

### State Parameter

```typescript
// CSRF protection with timeout
const state = crypto.randomUUID()
stateStore.set(state, {
  createdAt: Date.now(),
  providerID
})

// Validate within 5 minutes
if (Date.now() - stored.createdAt > 5 * 60 * 1000) {
  throw new Error("State expired")
}
```

### Token Expiry

```typescript
// Check expiry before use
function isExpired(auth: OAuthAuth): boolean {
  if (!auth.expiresAt) return false
  return Date.now() > auth.expiresAt * 1000
}

// Auto-refresh if expired
async function getAccessToken(providerID: string) {
  const auth = await getAuth(providerID)
  if (isExpired(auth) && auth.refreshToken) {
    const refreshed = await refreshToken(auth.refreshToken)
    await storeAuth(providerID, refreshed)
    return refreshed.accessToken
  }
  return auth.accessToken
}
```

---

## Key Files

| File | Purpose |
|------|---------|
| `src/auth/index.ts` | Core Auth namespace |
| `src/provider/auth.ts` | Provider authentication |
| `src/mcp/auth.ts` | MCP-specific auth storage |
| `src/mcp/oauth-provider.ts` | MCP OAuth provider |
| `src/mcp/oauth-callback.ts` | OAuth callback server |
| `src/server/routes/provider.ts` | Provider OAuth endpoints |
| `src/cli/cmd/auth.ts` | CLI auth commands |
| `src/plugin/copilot.ts` | GitHub Copilot auth |
| `src/plugin/codex.ts` | OpenAI auth |

---

## Configuration

```json
{
  "provider": {
    "anthropic": {
      "api_key": "{env:ANTHROPIC_API_KEY}"
    },
    "openai": {
      "api_key": "{env:OPENAI_API_KEY}"
    },
    "azure": {
      "api_key": "{env:AZURE_API_KEY}",
      "resource_name": "my-resource"
    }
  }
}
```

---

*Written by Claude (Opus 4.5) | 2026-01-22 16:00 PST*
