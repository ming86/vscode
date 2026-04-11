# 09 — Deployment & Remote Access

> Last updated: 2026-04-21

This document covers deploying the Copilot CLI Session webapp for remote access via `cloudflared` tunnel. For the core webapp architecture, see [06-webapp-extraction-guide.md](./06-webapp-extraction-guide.md). For constraints and performance budgets, see [08-constraints-and-requirements.md](./08-constraints-and-requirements.md).

> **Scope:** The webapp runs locally by default with nonce authentication (see doc 06, Section 5.1). This document addresses the additional configuration required when exposing the webapp over the network.

---

## Table of Contents

1. [Overview](#1-overview)
2. [Cloudflare Tunnel Setup](#2-cloudflare-tunnel-setup)
3. [Authentication for Remote Access](#3-authentication-for-remote-access)
4. [Security Controls](#4-security-controls)
5. [Implementation Roadmap — Phase 5](#5-implementation-roadmap--phase-5)
6. [Environment Variables & Configuration](#6-environment-variables--configuration)
7. [Constraints Reference](#7-constraints-reference)

---

## 1. Overview

In local mode, the webapp binds to `127.0.0.1:3000` and uses a per-startup nonce for authentication. This is sufficient when the browser and server are on the same machine.

For remote access — e.g., controlling a desktop Copilot session from a phone — the webapp must be exposed via a secure tunnel. The recommended approach is `cloudflared tunnel`, which provides:

- Encrypted transport without self-signed certificates
- Optional zero-trust authentication via Cloudflare Access
- No inbound port exposure on the host machine

The webapp itself does not start or manage the tunnel process. Tunnel configuration (DNS, access policies, team domains) is environment-specific and user-managed (see constraint OPS-07).

---

## 2. Cloudflare Tunnel Setup

### Prerequisites

| Requirement | Version |
|-------------|---------|
| **cloudflared** | v2026.1+ |
| **Cloudflare account** | With a registered domain |

### Configuration

The tunnel routes external HTTPS traffic to the local webapp:

```bash
# Create a named tunnel (one-time setup)
cloudflared tunnel create copilot-webapp

# Route DNS to the tunnel
cloudflared tunnel route dns copilot-webapp copilot.example.com

# Run the tunnel
cloudflared tunnel --url http://localhost:3000 run copilot-webapp
```

The webapp detects tunnel mode via configuration (environment variable or CLI flag) and adjusts authentication accordingly. It does **not** bind to `0.0.0.0` — `cloudflared` connects to `127.0.0.1:3000` locally.

---

## 3. Authentication for Remote Access

When exposed via `cloudflared tunnel`, nonce auth is insufficient — the nonce would need to travel over the network.

### Authentication Options

| Option | Complexity | Security | Recommendation |
|--------|-----------|----------|----------------|
| **Cloudflare Access** | Medium | High | Preferred for teams. Zero-trust; identity-aware; no code changes. |
| **Basic Auth (htpasswd)** | Low | Medium | Acceptable for personal use. Add Hono middleware. |
| **OAuth2 (GitHub)** | High | High | Natural fit since users already have GitHub accounts. |
| **Pre-shared token** | Low | Low | Better than nonce, but still a static secret. Last resort. |

### Recommended: Cloudflare Access + Bearer Token

```typescript
// ── Auth middleware for tunnel mode ────────────────────────────
import { verify } from 'jsonwebtoken';
import type { Context, Next } from 'hono';

interface AuthConfig {
  readonly mode: 'local' | 'tunnel';
  readonly nonce?: string;              // local mode
  readonly jwtSecret?: string;          // tunnel mode
  readonly cloudflareTeamDomain?: string; // e.g., 'myteam.cloudflareaccess.com'
}

function authMiddleware(config: AuthConfig) {
  return async (c: Context, next: Next) => {
    if (config.mode === 'local') {
      // Nonce-based
      if (c.req.header('authorization') !== `Nonce ${config.nonce}`) {
        return c.json({ error: 'Invalid nonce' }, 401);
      }
      return next();
    }

    // Tunnel mode: validate Cloudflare Access JWT
    const cfAuth = c.req.header('cf-access-jwt-assertion');
    if (cfAuth && config.cloudflareTeamDomain) {
      // Validate against Cloudflare's JWKS endpoint
      // https://{team}.cloudflareaccess.com/cdn-cgi/access/certs
      try {
        await validateCloudflareJwt(cfAuth, config.cloudflareTeamDomain);
        return next();
      } catch {
        return c.json({ error: 'Invalid CF token' }, 403);
      }
    }

    // Fallback: Bearer token
    const bearer = c.req.header('authorization')?.replace('Bearer ', '');
    if (bearer && config.jwtSecret) {
      try {
        verify(bearer, config.jwtSecret);
        return next();
      } catch {
        return c.json({ error: 'Invalid token' }, 401);
      }
    }

    return c.json({ error: 'Authentication required' }, 401);
  };
}
```

---

## 4. Security Controls

### Transport & Headers

| Control | Implementation |
|---------|---------------|
| **CORS** | Set `Access-Control-Allow-Origin` to the tunnel hostname |
| **CSP** | `Content-Security-Policy: default-src 'self'; connect-src 'self' wss://{tunnel-host}` |
| **Session tokens** | Issue short-lived JWTs (1h expiry) after initial auth; refresh on activity |
| **WS auth** | Validate token on WebSocket upgrade (`connection` event) before accepting |

### Tool Execution Sandboxing

MCP tools that execute commands (`run_in_terminal`) must run within the session's `workingDirectory`. No path traversal above the project root. This is especially critical in tunnel mode where the attack surface extends beyond the local machine. Validate all paths are within `workingDirectory` before execution. Reject `../` traversal. (See constraint SEC-11.)

### Secrets Management

No API keys, JWT secrets, or Cloudflare credentials in the SPA bundle. The nonce is the sole exception in local mode (injected into `index.html` at serve-time). All sensitive values remain server-side. The frontend authenticates via the nonce or session token — never directly with external services. (See constraint SEC-12.)

---

## 5. Implementation Roadmap — Phase 5

**Goal:** Secure access via `cloudflared tunnel`.

| Task | Deliverable |
|------|-------------|
| Auth middleware | Token-based auth for tunnel mode |
| `cloudflared` integration | Startup script or npm script |
| CORS configuration | Dynamic origin based on tunnel hostname |
| WS auth | Token validation on upgrade |

**Validation:** Access the webapp from a phone over a Cloudflare tunnel.

---

## 6. Environment Variables & Configuration

| Variable | Purpose | Default |
|----------|---------|---------|
| `COPILOT_WEBAPP_MODE` | `local` or `tunnel` | `local` |
| `COPILOT_WEBAPP_JWT_SECRET` | Secret for signing/verifying session JWTs (tunnel mode) | — (required in tunnel mode) |
| `COPILOT_WEBAPP_CF_TEAM_DOMAIN` | Cloudflare Access team domain for JWT validation | — (optional) |
| `COPILOT_WEBAPP_TUNNEL_HOSTNAME` | Tunnel hostname for CORS and CSP headers | — (required in tunnel mode) |

---

## 7. Constraints Reference

The following constraints from [08-constraints-and-requirements.md](./08-constraints-and-requirements.md) govern remote access behavior:

| ID | Summary |
|----|---------|
| SEC-06 | Tunnel mode requires additional authentication beyond nonce |
| SEC-08 | Content Security Policy must include tunnel host in `connect-src` |
| SEC-09 | Short-lived session tokens (1h expiry) in tunnel mode |
| SEC-11 | Tool execution sandboxing — critical in tunnel mode |
| SEC-12 | No secrets in client-side code |
| OPS-07 | `cloudflared` tunnel is optional and user-managed |
