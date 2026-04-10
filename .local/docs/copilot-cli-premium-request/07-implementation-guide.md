# Premium Request Tracking Implementation Guide

## Purpose

Step-by-step guide for implementing premium request tracking in a Copilot-integrated project. Intended for internal GitHub teams or licensed integrators with access to the `@github/copilot` SDK. Covers client-side tracking, API integration, and UI display.

## Prerequisites

- Access to CAPI (Copilot API) via `@github/copilot` SDK
- Understanding of the model multiplier system (see [01-model-multipliers.md](01-model-multipliers.md))
- Understanding of interaction types (see [03-interaction-types.md](03-interaction-types.md))

## Phase 1: Model Billing Metadata

### Step 1.1: Fetch Models with Billing Info

```typescript
import { CopilotClient } from '@github/copilot'; // @github/copilot is an internal GitHub package, not publicly available on npm

const client = new CopilotClient(/* options */);
const models = await client.listModels();

for (const model of models) {
  console.log(`${model.id}: multiplier=${model.billing?.multiplier ?? 0}, premium=${model.billing?.multiplier !== undefined && model.billing.multiplier > 0}`);
}
```

### Step 1.2: Store Billing Info Per-Endpoint

```typescript
interface MyEndpoint {
  modelId: string;
  isPremium: boolean;
  multiplier: number;
  restrictedToSkus?: string[];
}

function createEndpoint(model: ModelInfo): MyEndpoint {
  return {
    modelId: model.id,
    isPremium: model.billing?.multiplier !== undefined && model.billing.multiplier > 0,
    multiplier: model.billing?.multiplier ?? 0,
    restrictedToSkus: undefined, // from model metadata if available
  };
}
```

### Step 1.3: Display Multiplier in UI

```typescript
function getMultiplierLabel(multiplier: number): string {
  if (multiplier === 0) return 'Included';
  return `${multiplier}x`;
}

function getModelTooltip(name: string, multiplier: number): string {
  if (multiplier > 0) {
    return `${name} — Rate is counted at ${multiplier}x.`;
  }
  return name;
}
```

## Phase 2: Quota Tracking Service

### Step 2.1: Define Quota Types

```typescript
interface QuotaInfo {
  entitlement: number;      // Total allowed — Maps to `quota` in the source IChatQuota interface
  used: number;             // Calculated used count
  unlimited: boolean;       // -1 = unlimited
  overageUsed: number;      // Usage beyond entitlement
  overageEnabled: boolean;  // Can use beyond entitlement?
  resetDate: Date;          // When quota resets
}

interface QuotaService {
  readonly quotaExhausted: boolean;
  readonly overagesEnabled: boolean;
  // The actual source uses an internal IHeaders type; this example uses the standard Web API Headers for portability.
  processQuotaHeaders(headers: Headers): void;
  processQuotaSnapshots(snapshots: Record<string, QuotaSnapshot>): void;
  clearQuota(): void;
}
```

### Step 2.2: Parse Quota Headers

> The canonical parsing description is in [02-quota-tracking.md](02-quota-tracking.md).

```typescript
function parseQuotaHeader(headerValue: string): QuotaInfo | null {
  try {
    const params = new URLSearchParams(headerValue);
    const entitlement = parseInt(params.get('ent') || '0', 10);
    const overageUsed = parseFloat(params.get('ov') || '0.0');
    const overageEnabled = params.get('ovPerm') === 'true';
    const percentRemaining = parseFloat(params.get('rem') || '0.0');
    const resetDateStr = params.get('rst');

    const resetDate = resetDateStr
      ? new Date(resetDateStr)
      : new Date(Date.now() + 30 * 24 * 60 * 60 * 1000); // Default: 30 days

    return {
      entitlement,
      used: Math.max(0, entitlement * (1 - percentRemaining / 100)),
      unlimited: entitlement === -1,
      overageUsed,
      overageEnabled,
      resetDate,
    };
  } catch {
    return null;
  }
}

function getQuotaHeader(headers: Headers, isFreeUser: boolean): string | null {
  if (isFreeUser) {
    return headers.get('x-quota-snapshot-chat');
  }
  return headers.get('x-quota-snapshot-premium_models')
      || headers.get('x-quota-snapshot-premium_interactions');
}
```

### Step 2.3: Check Quota Exhaustion

```typescript
function isQuotaExhausted(quota: QuotaInfo): boolean {
  return quota.used >= quota.entitlement
      && !quota.overageEnabled
      && !quota.unlimited;
}
```

## Phase 3: Usage Event Processing

### Step 3.1: Track Per-API-Call Usage

```typescript
interface ModelMetrics {
  requests: { count: number; cost: number };
  usage: {
    inputTokens: number;
    outputTokens: number;
    cacheReadTokens: number;
    cacheWriteTokens: number;
  };
}

interface UsageMetrics {
  totalPremiumRequests: number;  // Accumulated: sum of all costs
  totalUserRequests: number;     // Raw count of user prompts (not multiplied)
  totalApiDurationMs: number;
  modelMetrics: Map<string, ModelMetrics>;
}

function processUsageEvent(metrics: UsageMetrics, event: AssistantUsageEvent): void {
  const cost = event.data.cost ?? 0;
  const model = event.data.model;

  // Note: cost is reported for ALL API calls, but only user-initiated calls
  // (initiator absent or "user") actually count toward premium request billing.
  // Agent-initiated calls (sub-agents, background) have cost=0 or use free models.
  // Assumption: server emits cost=0 for non-user-initiated calls. If this changes, filter by initiator here.
  metrics.totalPremiumRequests += cost;
  metrics.totalApiDurationMs += event.data.duration ?? 0;

  // Only user-initiated calls (initiator absent or "user") are billable.
  // Calls with initiator="agent", "sub-agent", or "mcp-sampling" are NOT billed.
  if (!event.data.initiator || event.data.initiator === 'user') {
    metrics.totalUserRequests += 1;
  }

  // Update per-model metrics
  let modelMetric = metrics.modelMetrics.get(model);
  if (!modelMetric) {
    modelMetric = {
      requests: { count: 0, cost: 0 },
      usage: { inputTokens: 0, outputTokens: 0, cacheReadTokens: 0, cacheWriteTokens: 0 },
    };
    metrics.modelMetrics.set(model, modelMetric);
  }

  modelMetric.requests.count += 1;
  modelMetric.requests.cost += cost;
  modelMetric.usage.inputTokens += event.data.inputTokens ?? 0;
  modelMetric.usage.outputTokens += event.data.outputTokens ?? 0;
  modelMetric.usage.cacheReadTokens += event.data.cacheReadTokens ?? 0;
  modelMetric.usage.cacheWriteTokens += event.data.cacheWriteTokens ?? 0;
}
```

### Step 3.2: Process Quota Snapshots from Usage Events

```typescript
function processUsageQuotaSnapshots(
  quotaService: QuotaService,
  event: AssistantUsageEvent,
): void {
  if (event.data.quotaSnapshots) {
    quotaService.processQuotaSnapshots(event.data.quotaSnapshots);
  }
}
```

## Phase 4: Quota Exhaustion Handling

### Step 4.1: Model Switching on Quota Exhaustion

```typescript
async function selectModel(
  preferredModel: string,
  quotaService: QuotaService,
  endpoints: Map<string, MyEndpoint>,
): Promise<string> {
  const endpoint = endpoints.get(preferredModel);

  // Free model — no switching needed
  if (!endpoint || endpoint.multiplier === 0) {
    return preferredModel;
  }

  // Quota OK or overages enabled
  if (!quotaService.quotaExhausted || quotaService.overagesEnabled) {
    return preferredModel;
  }

  // Switch to fallback (free) model
  console.warn(`Quota exhausted — switching from ${preferredModel} to fallback model`);
  const baseEndpoint = await this.endpointProvider.getChatEndpoint('copilot-base');
  return baseEndpoint?.modelId ?? 'gpt-4o'; // Dynamic resolution preferred; hardcoded fallback only as last resort
}
```

### Step 4.2: Error Handling for Quota Errors

```typescript
function handleQuotaError(statusCode: number, errorCode?: string): string | null {
  if (statusCode === 402) {
    return 'Free plan quota exceeded. Upgrade to a paid plan.';
  }

  switch (errorCode) {
    case 'quota_exceeded':
    case 'free_quota_exceeded':
      return 'Premium request quota exhausted. Enable overages or switch to an included model.';
    case 'overage_limit_reached':
      return 'Overage limit reached. Cannot accrue more paid premium requests.';
    case 'billing_not_configured':
      return 'Billing not configured. Configure billing to use premium models.';
    default:
      return null;
  }
}
```

## Phase 5: UI Display

### Step 5.1: Usage Display Widget

```typescript
function formatUsageDisplay(metrics: UsageMetrics): string {
  const lines: string[] = [];
  lines.push(`Premium Requests: ${metrics.totalPremiumRequests}`);
  lines.push(`User Prompts: ${metrics.totalUserRequests}`);
  lines.push(`API Duration: ${(metrics.totalApiDurationMs / 1000).toFixed(1)}s`);
  lines.push('');
  lines.push('Per-Model Breakdown:');

  for (const [model, m] of metrics.modelMetrics) {
    lines.push(`  ${model}: ${m.requests.count} calls, ${m.requests.cost} PRs, ${m.usage.inputTokens + m.usage.outputTokens} tokens`);
  }

  return lines.join('\n');
}
```

### Step 5.2: Remaining Quota Widget

```typescript
function formatQuotaWidget(quota: QuotaInfo): string {
  if (quota.unlimited) return '∞ remaining';
  const remaining = Math.max(0, quota.entitlement - quota.used);
  const percent = quota.entitlement > 0
    ? (remaining / quota.entitlement * 100).toFixed(0)
    : '0';
  return `${remaining.toFixed(0)}/${quota.entitlement} remaining (${percent}%)`;
}
```

## Phase 6: Integration with copilot-sdk Session Events

### Step 6.1: Subscribe to Usage Events

```typescript
import { Session } from '@github/copilot'; // @github/copilot is an internal GitHub package, not publicly available on npm

const session = new Session(/* options */);

// Illustrative — verify actual SDK event subscription API
session.on('event', (event) => {
  if (event.type === 'assistant.usage') {
    processUsageEvent(myMetrics, event);
    processUsageQuotaSnapshots(myQuotaService, event);
  }

  if (event.type === 'session.shutdown') {
    // WARNING: session.shutdown may not be reliably captured.
    // The SDK's disconnect() clears event handlers after session.destroy RPC,
    // creating a race condition. Prefer real-time tracking via assistant.usage events.
    // Session total — may be unreliable; see 04-usage-events.md for details.
    console.log(`Session ended: ${event.data.totalPremiumRequests} premium requests used`);
    console.log(`Model breakdown:`, event.data.modelMetrics);
  }
});
```

### Step 6.2: Query Quota via RPC

```typescript
const quota = await session.rpc.account.getQuota();
const premiumQuota = quota.quotaSnapshots['premium_interactions'];

if (premiumQuota) {
  console.log(`Used: ${premiumQuota.usedRequests}/${premiumQuota.entitlementRequests}`);
  console.log(`Remaining: ${premiumQuota.remainingPercentage}%`);
  console.log(`Overage: ${premiumQuota.overage}`);
}
```

## Key Implementation Notes

1. **Server-side counting is authoritative** — The client only observes; the server determines actual consumption
2. **Multipliers change** — Always fetch current multipliers from `models.list`, do not hardcode
3. **Sub-agents are NOT separately billed** — they are marked as `initiator=agent` and not charged by the billing system
4. **Background operations are effectively free** — they use multiplier=0 models, so their cost is 0 premium requests
5. **Free plan vs paid plan paths differ** — Different quota headers, different error messages, different model access
6. **Rate limits ≠ quota** — 429 errors are rate limits (temporary), 402 errors are quota exhaustion (monthly)
7. **Test with `/usage`** — Use the CLI's built-in usage command to validate your tracking matches reality

## Related Documentation

- [04-usage-events.md](04-usage-events.md) — Usage event schemas referenced in Phase 3
- [05-consumption-sources.md](05-consumption-sources.md) — Complete catalog of consumption vectors
- [06-quota-api.md](06-quota-api.md) — Quota API details for Phase 6
