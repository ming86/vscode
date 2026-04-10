# Quota Tracking and Exhaustion Handling

## Overview

The extension tracks premium request consumption through quota buckets, updated from three independent data sources (HTTP headers, SSE events, and the user-info endpoint). When a bucket is exhausted, the system switches the active model to `copilot-base` and surfaces plan-specific messaging.

## Quota Buckets

Three static quota buckets are defined in `CopilotUserQuotaInfo`:

| Bucket | Scope |
|--------|-------|
| `chat` | Standard model usage (Free plan users) |
| `completions` | Code completion quota |
| `premium_interactions` | Premium model usage (paid users) |

> **Note:** `premium_models` is not a static bucket in `CopilotUserQuotaInfo`. It appears as a dynamic key in HTTP response headers (`x-quota-snapshot-premium_models`) and SSE/WebSocket quota snapshot payloads. The `processQuotaHeaders()` and `processQuotaSnapshots()` methods fall back from `premium_models` to `premium_interactions` for paid users.

Each bucket carries the following shape:

> **Documentation alias:** `QuotaBucket` is not a named type in the source. It represents the anonymous inline type used for each quota bucket in `CopilotUserQuotaInfo`. The runtime snapshot type is `QuotaSnapshots = Record<string, QuotaSnapshot>` (a dynamic record keyed by bucket name).

```typescript
// From chatQuotaService.ts
interface CopilotUserQuotaInfo {
  quota_reset_date?: string;   // Optional — absent when quota info unavailable
  quota_snapshots?: {          // Optional — absent when quota info unavailable
    chat: {
      quota_id: string;
      entitlement: number;        // Total allowed (e.g., 300 for Pro)
      remaining: number;          // Remaining count
      unlimited: boolean;         // True for some enterprise tiers
      overage_count: number;      // Units consumed beyond entitlement
      overage_permitted: boolean; // Whether overage is allowed
      percent_remaining: number;  // Percentage remaining (0-100)
    };
    completions: { /* same shape */ };
    premium_interactions: { /* same shape */ };
  };
}
```

## IChatQuotaService Interface

**Source:** `src/platform/chat/common/chatQuotaService.ts`

```typescript
interface IChatQuota {
  quota: number;         // Total entitlement
  used: number;          // Calculated: entitlement * (1 - percentRemaining/100)
  unlimited: boolean;    // -1 indicates unlimited
  overageUsed: number;   // Decimal overage consumed
  overageEnabled: boolean;
  resetDate: Date;
}

interface IChatQuotaService {
  readonly _serviceBrand: undefined;
  quotaExhausted: boolean;     // Is the quota fully used?
  overagesEnabled: boolean;    // Can user use beyond entitlement?
  processQuotaHeaders(headers: IHeaders): void;
  processQuotaSnapshots(snapshots: QuotaSnapshots): void;
  clearQuota(): void;
}
```

## Three Sources of Quota Data

### Source 1: HTTP Response Headers

After every API call, response headers contain quota snapshots.

**Source:** `ChatQuotaService.processQuotaHeaders()`

```typescript
// Free users get chat quota header, paid users get premium models header
const quotaHeader = this._authService.copilotToken?.isFreeUser
  ? headers.get('x-quota-snapshot-chat')
  : headers.get('x-quota-snapshot-premium_models')
    || headers.get('x-quota-snapshot-premium_interactions');

// Header is URL-encoded: ent=300&ov=0.0&ovPerm=true&rem=85.5&rst=2025-11-01
const params = new URLSearchParams(quotaHeader);
const entitlement = parseInt(params.get('ent') || '0', 10);
const overageUsed = parseFloat(params.get('ov') || '0.0');
const overageEnabled = params.get('ovPerm') === 'true';
const percentRemaining = parseFloat(params.get('rem') || '0.0');
const resetDateString = params.get('rst');
const used = Math.max(0, entitlement * (1 - percentRemaining / 100));
```

> **Note:** `isFreeUser` is a property of the `CopilotToken` class (derived from the authentication token's `sku` field), not part of quota info. See `src/platform/authentication/common/copilotToken.ts`.

Header parameters:

| Parameter | Meaning |
|-----------|---------|
| `ent` | Entitlement count (total allowed) |
| `ov` | Overage count (usage beyond entitlement) |
| `ovPerm` | Overage permitted flag |
| `rem` | Percent remaining (0-100) |
| `rst` | Reset date (ISO 8601) |

### Source 2: Copilot Quota Snapshots in SSE Events and WebSocket Frames

**Source:** `chatMLFetcher.ts` (SSE) and `chatWebSocketManager.ts` (WebSocket).

The `response.completed` SSE event may contain `copilot_quota_snapshots`. Access is untyped — the property is read via an `(event as any)` cast, as the SSE event type does not declare `copilot_quota_snapshots`.

```typescript
// Untyped access — copilot_quota_snapshots is not on the declared event type
if (event.type === 'response.completed') {
  const snapshots = (event as any).copilot_quota_snapshots;
  if (snapshots && typeof snapshots === 'object') {
    this._chatQuotaService.processQuotaSnapshots(snapshots);
  }
}
```

The WebSocket path carries `copilot_quota_snapshots` on **error events** (`CAPIWebSocketErrorEvent`), not `response.completed`. These fire for non-recoverable errors (rate limits, quota exhaustion, upstream failures). Unlike the SSE path, the WebSocket error event interface declares `copilot_quota_snapshots?: QuotaSnapshots` as a typed optional property.

`processQuotaSnapshots()` mirrors `processQuotaHeaders()` but reads from snapshot objects:

```typescript
const snapshot = isFreeUser
  ? snapshots['chat']
  : snapshots['premium_models'] ?? snapshots['premium_interactions'];
```

### Source 3: Copilot Token (User Info Endpoint)

**Source:** `processUserInfoQuotaSnapshot()` — the authentication token includes quota info from the `copilot_internal/user` endpoint.

```typescript
private processUserInfoQuotaSnapshot(quotaInfo: CopilotUserQuotaInfo | undefined) {
  if (!quotaInfo || !quotaInfo.quota_snapshots || !quotaInfo.quota_reset_date) {
    return;
  }
  this._quotaInfo = {
    unlimited: quotaInfo.quota_snapshots.premium_interactions.unlimited,
    overageEnabled: quotaInfo.quota_snapshots.premium_interactions.overage_permitted,
    overageUsed: quotaInfo.quota_snapshots.premium_interactions.overage_count,
    quota: quotaInfo.quota_snapshots.premium_interactions.entitlement,
    resetDate: new Date(quotaInfo.quota_reset_date),
    used: Math.max(0,
      quotaInfo.quota_snapshots.premium_interactions.entitlement
        * (1 - quotaInfo.quota_snapshots.premium_interactions.percent_remaining / 100)),
  };
}
```

## Quota Exhaustion Check

```typescript
get quotaExhausted(): boolean {
  if (!this._quotaInfo) return false;
  return this._quotaInfo.used >= this._quotaInfo.quota
    && !this._quotaInfo.overageEnabled
    && !this._quotaInfo.unlimited;
}
```

Exhausted when all three conditions hold:
1. `used >= quota`
2. Overage is **not** enabled
3. Tier is **not** unlimited

## Quota Exhaustion Response — Model Switching

**Source:** `chatParticipants.ts`

When quota is exhausted, the system switches to `copilot-base` (0× multiplier):

```typescript
private async switchToBaseModel(request, stream): Promise<ChatRequest> {
  const endpoint = await this.endpointProvider.getChatEndpoint(request);
  // 0x multiplier = free model, no switching needed. BYOK models (vendor !== 'copilot') are also free.
  if (endpoint.multiplier === 0 || request.model.vendor !== 'copilot' || endpoint.multiplier === undefined) {
    return request;
  }
  if (this._chatQuotaService.overagesEnabled || !this._chatQuotaService.quotaExhausted) {
    return request;
  }
  // Switch to copilot-base (free model) and warn user
}
```

## Quota Exceeded Error Codes

**Source:** `commonTypes.ts` (except where noted)

| Error Code | Meaning | Source |
|-----------|---------|--------|
| `quota_exceeded` | Generic premium quota exhausted | `commonTypes.ts` |
| `free_quota_exceeded` | Free plan specific (remapped to `quota_exceeded`) | `commonTypes.ts` |
| `overage_limit_reached` | Cannot accrue more paid premium requests | `commonTypes.ts` |
| `billing_not_configured` | Billing not configured for user's account | `chatMLFetcher.ts` |

## Plan-Specific Error Messages

**Source:** `getQuotaHitMessage()`

| Plan | `copilotPlan` | Message |
|------|---------------|---------|
| **Free plan** | `'free'` | "You've reached your monthly chat messages quota. Upgrade to Copilot Pro (30-day free trial) or wait for your allowance to renew." |
| **Individual (Pro)** | `'individual'` | "You've exhausted your premium model quota. Please enable additional paid premium requests, upgrade to Copilot Pro+, or wait for your allowance to renew." |
| **Individual Pro (Pro+)** | `'individual_pro'` | "You've exhausted your premium model quota. Please enable additional paid premium requests or wait for your allowance to renew." |
| **Business / Enterprise** | `'business'` / `'enterprise'` (via default case) | "You've exhausted your premium model quota. To continue working, switch to Auto. For additional paid premium requests, please reach out to your organization's Copilot admin or wait for your allowance to renew." |

## Overage Management Command

**Source:** `src/extension/chat/vscode-node/chatQuota.contribution.ts`

```typescript
commands.registerCommand('chat.enablePremiumOverages', () => {
  chatQuotaService.clearQuota();
  env.openExternal(Uri.parse('https://aka.ms/github-copilot-manage-overage'));
});
```

## HTTP Status Codes

| Status | Meaning |
|--------|---------|
| `402` | Free plan quota exceeded (triggers token refresh) |
| `429` | Rate limited (separate from quota — per-model or per-user) |

## Rate Limit Error Types

Distinct from quota exhaustion. These are transient capacity constraints.

**Source:** `commonTypes.ts`

| Error Type | Scope |
|-----------|-------|
| `agent_mode_limit_exceeded` | Agent mode specific rate limit |
| `model_overloaded` / `upstream_provider_rate_limit` | Model capacity |
| `user_global_rate_limited` | User's global rate limit |
| `user_model_rate_limited` | Model-specific rate limit |
| `integration_rate_limited` | Service-level rate limit |

## Related Documentation

*For on-demand quota queries via RPC, see [Quota API](06-quota-api.md).*
