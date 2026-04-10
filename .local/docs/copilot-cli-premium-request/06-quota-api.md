# Quota API and Monitoring

## Overview

The premium request quota system is observable through multiple channels: a dedicated JSON-RPC method (`account.getQuota`), HTTP response headers on every API call, SSE events at response completion, and a user-info endpoint consulted at authentication time. Together these provide real-time, per-period, and per-session quota visibility across the CLI and VS Code extension.

## Schema Reconciliation Note

Quota data surfaces in three distinct schemas, each with a slightly different field set:

| Source | Schema | Key differences |
|--------|--------|-----------------|
| RPC `account.getQuota` result | `AccountGetQuotaResult.quotaSnapshots[type]` | Simpler schema: `entitlementRequests`, `usedRequests`, `remainingPercentage`, `overage`, `overageAllowedWithExhaustedQuota`, `resetDate` |
| Per-request `assistant.usage` events (SDK) | `event.data.quotaSnapshots[type]` | Richer schema that may include `isUnlimitedEntitlement` and `usageAllowedWithExhaustedQuota` alongside the standard fields |
| SSE `copilot_quota_snapshots` in `response.completed` events | `QuotaSnapshots` (see `src/platform/chat/common/chatQuotaService.ts`) | Uses `entitlement` (string), `percent_remaining`, `overage_permitted`, `overage_count`, `reset_date` |

Consumers must normalize across these schemas. The extension's `ChatQuotaService` (implemented in `src/platform/chat/common/chatQuotaServiceImpl.ts`) handles this in `processQuotaHeaders` (header-based) and `processQuotaSnapshots` (SSE-based), both converging into the unified `IChatQuota` shape.

## account.getQuota RPC Method

The SDK exposes a JSON-RPC method for querying current quota status on demand.

**Source:** `copilot-sdk/nodejs/src/generated/rpc.ts`

### Result Schema

```typescript
export interface AccountGetQuotaResult {
  /**
   * Quota snapshots keyed by type (e.g., chat, completions, premium_interactions)
   */
  quotaSnapshots: {
    [quotaType: string]: {
      /** Number of requests included in the entitlement */
      entitlementRequests: number;
      /** Number of requests used so far this period */
      usedRequests: number;
      /** Percentage of entitlement remaining */
      remainingPercentage: number;
      /** Number of overage requests made this period */
      overage: number;
      /** Whether pay-per-request usage is allowed when quota is exhausted */
      overageAllowedWithExhaustedQuota: boolean;
      /** Date when the quota resets (ISO 8601) */
      resetDate?: string;
    };
  };
}
```

### Usage via SDK RPC Client

```typescript
const client = createRpcClient(connection);
const quota = await client.account.getQuota();
// Returns:
// {
//   quotaSnapshots: {
//     premium_interactions: {
//       entitlementRequests: 300,
//       usedRequests: 42,
//       remainingPercentage: 86.0,
//       overage: 0,
//       overageAllowedWithExhaustedQuota: true,
//       resetDate: "2025-11-01T00:00:00Z"
//     }
//   }
// }
```

### Implementation Status

From the SDK test file (`test/e2e/rpc.test.ts`):

- `account.getQuota()` is defined in the RPC schema
- May not be fully implemented in all CLI versions — verify against the version in use

## models.list RPC Method

Returns available models with billing metadata, including multipliers that determine premium request cost.

```typescript
const models = await client.models.list();
// Each model includes:
// {
//   id: "claude-opus-4.6",
//   name: "Claude Opus 4.6",
//   billing: { multiplier: 3 },
//   ...
// }
```

This is the mechanism by which the CLI and VS Code obtain current model multipliers at runtime. See [01-model-multipliers.md](01-model-multipliers.md) for the multiplier table.

## Quota Monitoring in vscode-copilot-chat

Three monitoring channels feed quota state into the extension: HTTP response headers, SSE completion events, and the user-info authentication endpoint.

### Real-Time Header Monitoring

Every API response includes quota headers parsed by `ChatQuotaService`:

```
x-quota-snapshot-premium_models: ent=300&ov=0.0&ovPerm=true&rem=85.5&rst=2025-11-01
```

Or, depending on the SKU:

```
x-quota-snapshot-premium_interactions: ent=300&ov=0.0&ovPerm=true&rem=85.5&rst=2025-11-01
```

| Header Parameter | Meaning |
|-----------------|---------|
| `ent` | Entitlement (total requests in period) |
| `ov` | Overage count |
| `ovPerm` | Overage permitted (`true`/`false`) |
| `rem` | Remaining percentage |
| `rst` | Reset date (ISO 8601 date portion) |

### SSE Event Monitoring

The `response.completed` SSE event includes `copilot_quota_snapshots` with the post-request quota state.

> **Note:** The snapshot key may be either `premium_models` or `premium_interactions` depending on the SKU. The extension resolves this via fallback: `snapshots['premium_models'] ?? snapshots['premium_interactions']` (see `src/platform/chat/common/chatQuotaServiceImpl.ts`). Free-plan users receive the `chat` key instead.

```json
{
  "type": "response.completed",
  "copilot_quota_snapshots": {
    "premium_models": {
      "entitlement": "300",
      "percent_remaining": 85.5,
      "overage_permitted": true,
      "overage_count": 0,
      "reset_date": "2025-11-01T00:00:00Z"
    }
  }
}
```

This is the authoritative post-turn snapshot, reflecting the quota impact of the request that just completed.

### Token-Level Quota (User Info Endpoint)

The `/copilot_internal/user` endpoint returns user info including quota data (defined as `CopilotUserQuotaInfo` in `src/platform/chat/common/chatQuotaService.ts`). This is distinct from the authentication token itself, which is obtained from `/copilot_internal/v2/token`.

> **Documentation alias:** `QuotaBucket` is not a named type in the source. It represents the anonymous inline type used for each quota bucket in `CopilotUserQuotaInfo`. The actual source repeats the inline object shape for each key (`chat`, `completions`, `premium_interactions`).

```typescript
// Source: src/platform/chat/common/chatQuotaService.ts
// See [02-quota-tracking.md](02-quota-tracking.md) for the canonical definition and detailed field documentation.
export interface CopilotUserQuotaInfo {
  quota_reset_date?: string;
  quota_snapshots?: {
    chat: {
      quota_id: string;
      entitlement: number;
      remaining: number;
      unlimited: boolean;
      overage_count: number;
      overage_permitted: boolean;
      percent_remaining: number;
    };
    completions: { /* same shape */ };
    premium_interactions: { /* same shape */ };
  };
}
```

This provides the initial quota state before any requests are made in the session.

## Quota Context Keys (VS Code)

**Source:** `src/extension/contextKeys/vscode-node/contextKeys.contribution.ts`

The extension exposes quota state as VS Code context keys, consumed by `when` clauses throughout the UI:

- **`github.copilot.chat.quotaExceeded`** — Set when `isFreeUser && (limited_user_quotas?.chat ?? 1) <= 0` — the `?? 1` default means absent quota data is treated as non-exhausted. This reflects free-plan chat exhaustion only, not premium quota. Source: `src/extension/contextKeys/vscode-node/contextKeys.contribution.ts`, which reads `copilotToken.isChatQuotaExceeded`.
- **`github.copilot.completions.quotaExceeded`** — Analogous key for Free plan completions quota (defined in `src/extension/completions-core/vscode-node/lib/src/openai/fetch.ts`)

These context keys are consumed exclusively by VS Code `when` clauses for UI purposes (enabling/disabling UI elements, showing banners). They are **not** consulted by runtime logic such as model fallback.

| Purpose | Mechanism |
|---------|-----------|
| Enable/disable UI elements | `when` clauses grey out controls when the context key is `true` |
| Quota exceeded warnings | Surfaces banners and notifications for Free plan users |
| Base model fallback | **Not driven by this context key.** The `switchToBaseModel` method in `src/extension/conversation/vscode-node/chatParticipants.ts` consults `IChatQuotaService.quotaExhausted` directly (the premium quota signal), not the context key. It also checks `overagesEnabled` — if overages are permitted, no model switch occurs even when quota is exhausted. |

See [02-quota-tracking.md](02-quota-tracking.md) for the full tracking lifecycle and header-based monitoring details.

## SessionInfo Premium Tracking

**Source:** `src/platform/github/common/githubAPI.ts`

```typescript
interface SessionInfo {
  id: string;
  state: 'completed' | 'in_progress' | 'failed' | 'queued';
  premium_requests: number;  // Count of premium requests used in this session
  // Only premium-request-relevant fields shown. See `src/platform/github/common/githubAPI.ts` for the full interface.
}
```

Used for tracking premium consumption per cloud agent session. The `premium_requests` field accumulates across all turns within a single session, including sub-agent turns.

## The /usage Slash Command (CLI)

Added in copilot-cli v0.0.333 (from release notes — not verifiable from extension source). Displays real-time consumption metrics for the current session.

```
/usage
```

### Output Includes

| Metric | Description |
|--------|-------------|
| Premium request usage | Total consumed this session |
| Per-model token consumption | Breakdown by model (input/output tokens) |
| Code change metrics | Lines added/removed, files modified |
| Session duration | Wall-clock time since session start |

Since v0.0.422 (from release notes — not verifiable from extension source): Also includes token consumption from sub-agents.

This information is also displayed automatically at session conclusion (the `session.shutdown` event).

## Premium Request Analytics Dashboard

GitHub provides an organization/enterprise-level analytics dashboard:

- Accessible via **GitHub.com > Copilot Settings > Usage**
- Breaks down usage by model, feature, and user
- Shows separate SKU tracking for Copilot Chat, Spark, and Cloud Agent
- Generally available since September 2025

### Usage Report Download

Organizations can download CSV usage reports containing:

| Column Category | Content |
|----------------|---------|
| Request counts | All premium requests (within and beyond allowance) |
| Per-user breakdown | Individual consumption by GitHub handle |
| Per-model breakdown | Requests and tokens grouped by model identifier |
| Temporal data | Daily granularity for trend analysis |

## Monitoring Best Practices

1. **Use `/usage` frequently** — Check consumption during long or multi-turn sessions
2. **Watch the quota widget** — CLI surfaces remaining requests in status output
3. **Review `assistant.usage` events** — Each event contains real-time quota snapshots
4. **Check `session.shutdown` event** — Final aggregated metrics after session ends
5. **Monitor `modelMetrics`** — Per-model breakdown reveals which models consume the most budget
6. **Set billing alerts** — Use GitHub billing settings to configure alerts at 75%, 90%, and 100% thresholds
7. **Enable streamer mode** — `/streamer-mode` or `/on-air` suppresses quota details during broadcasts
