# Usage Events and Session Metrics

## Overview

The copilot-sdk tracks premium request consumption through two primary event types emitted during and after a session:

| Event | Timing | Persistence | Purpose |
|-------|--------|-------------|---------|
| `assistant.usage` | After every LLM API call | Ephemeral (not written to `events.jsonl`) | Per-call metrics, quota snapshots, billing data |
| `session.shutdown` | Session end | Persisted to `events.jsonl` | Aggregated totals for the entire session |

Both are defined in `copilot-sdk/nodejs/src/generated/session-events.ts`.

## `assistant.usage` Event (Per-API-Call)

Emitted after every LLM API call. Marked `ephemeral: true`, meaning it is available only in real-time and is **not** persisted to `events.jsonl`.

> **SDK Reference:** The `assistant.usage` event schema is defined in the `copilot-sdk` package (`session-events.ts`), which is external to this repository. The schema shown here reflects the SDK at time of writing and cannot be verified from extension source alone. The extension itself only accesses `inputTokens` and `outputTokens` from these events.

### Type Definition

From `session-events.ts` (the `type: "assistant.usage"` definition):

```typescript
{
  id: string;              // UUID v4
  timestamp: string;       // ISO 8601
  parentId: string | null;
  ephemeral: true;         // NOT persisted to events.jsonl
  type: "assistant.usage";
  data: {
    /** Model identifier used for this API call */
    model: string;
    /** Number of input tokens consumed */
    inputTokens?: number;
    /** Number of output tokens produced */
    outputTokens?: number;
    /** Number of tokens read from prompt cache */
    cacheReadTokens?: number;
    /** Number of tokens written to prompt cache */
    cacheWriteTokens?: number;
    /** Model multiplier cost for billing purposes */
    cost?: number;
    /** Duration of the API call in milliseconds */
    duration?: number;
    /** Time to first token in milliseconds (TTFT) */
    ttftMs?: number;
    /** Average inter-token latency in milliseconds */
    interTokenLatencyMs?: number;
    /** What initiated this API call (absent for user-initiated calls) */
    initiator?: string;
    /** Completion ID from the model provider */
    apiCallId?: string;
    /** GitHub request tracing ID (x-github-request-id header) */
    providerCallId?: string;
    /** Parent tool call ID when usage originates from a sub-agent */
    parentToolCallId?: string;

    /** Per-request quota snapshots */
    quotaSnapshots?: {
      [quotaType: string]: {
        isUnlimitedEntitlement: boolean;
        entitlementRequests: number;
        usedRequests: number;
        usageAllowedWithExhaustedQuota: boolean;
        overage: number;
        overageAllowedWithExhaustedQuota: boolean;
        remainingPercentage: number;
        resetDate?: string;
      };
    };

    /** Per-request CAPI billing data */
    copilotUsage?: {
      tokenDetails: {
        batchSize: number;    // Number of tokens in this billing batch
        costPerBatch: number; // Cost per batch of tokens
        tokenCount: number;   // Total token count for this entry
        tokenType: string;    // "input" or "output"
      }[];
      totalNanoAiu: number;   // Total cost in nano-AIU (AI Units)
    };

    /** Reasoning effort level used, if applicable */
    reasoningEffort?: string;
  };
}
```

### `quotaSnapshots` — SDK vs Extension Property Mapping

The `quotaSnapshots` field within `assistant.usage` uses property names that differ from the extension's `QuotaSnapshot` type (`src/platform/chat/common/chatQuotaService.ts`). The extension does **not** consume `quotaSnapshots` from `assistant.usage` events directly; instead, it processes structurally similar snapshots from the SSE completion response (`copilot_quota_snapshots`), which use the extension's own schema.

| SDK event (`assistant.usage`) | Extension `QuotaSnapshot` | Notes |
|-------------------------------|---------------------------|-------|
| `isUnlimitedEntitlement: boolean` | `entitlement: string` | Extension uses `"-1"` string for unlimited |
| `entitlementRequests: number` | (derived) | Extension parses `entitlement` string to integer |
| `usedRequests: number` | (derived) | Extension computes from `entitlement * (1 - percent_remaining / 100)` |
| `remainingPercentage: number` | `percent_remaining: number` | Same semantics, different name |
| `overage: number` | `overage_count: number` | Same semantics, different name |
| `overageAllowedWithExhaustedQuota: boolean` | `overage_permitted: boolean` | Same semantics, different name |
| `usageAllowedWithExhaustedQuota: boolean` | (not mapped) | No extension equivalent |
| `resetDate?: string` | `reset_date?: string` | camelCase vs snake_case |

The extension selects the quota key by plan type: Free plan users use `snapshots['chat']`; paid plan users use `snapshots['premium_models']` (falling back to `snapshots['premium_interactions']`).

### Key Fields for Premium Tracking

| Field | Significance |
|-------|-------------|
| **`initiator`** | **The primary billing discriminator.** Determines whether this call counts toward premium request consumption — see [Initiator Values](#initiator-values) and [Billing Discriminator](#billing-discriminator) below |
| `cost` | The model multiplier applied to this request (e.g., `3` for Opus 4.6, `0.33` for Haiku 4.5). Emitted for ALL calls, but only contributes to billing when `initiator` is absent or `"user"` |
| `parentToolCallId` | Links sub-agent API calls back to the parent tool invocation |
| `quotaSnapshots` | Real-time quota status **after** this request was consumed — see [mapping table above](#quotasnapshots--sdk-vs-extension-property-mapping) |
| `copilotUsage.totalNanoAiu` | Actual billing cost in nano-AIU (based on the naming convention (nano = 10⁻⁹); the exact conversion factor is not documented in source) |

> For what makes a request billable, see [03-interaction-types.md](03-interaction-types.md). For the complete catalog of consumption sources, see [05-consumption-sources.md](05-consumption-sources.md).

> **Note:** Quota data appears in different shapes depending on context — the RPC `getQuota` result, the per-request `quotaSnapshots` within `assistant.usage` events, and the SSE completion snapshot each carry overlapping but structurally distinct representations. Cross-check the relevant schema when switching between sources.

### Initiator Values

| Value | Meaning |
|-------|---------|
| `undefined` (absent) | User-initiated — main agent responding to a user prompt — **BILLED** |
| `"user"` | Explicitly user-initiated (used by the WebSocket path) — **BILLED** |
| `"agent"` | Agent-initiated (agentic loop iterations, sub-agents, background operations via `chatWebSocketManager.ts`) — **NOT billed** |
| `"sub-agent"` | Task tool sub-agent API call — **NOT billed** |
| `"mcp-sampling"` | MCP server sampling request — server-determined (SDK/server concept — not present in extension source) |

### Billing Discriminator

The `initiator` field is the primary billing discriminator. Only events where `initiator` is absent (or `initiator="user"`) count toward premium request consumption. Events with any other `initiator` value — including `initiator="agent"`, `initiator="sub-agent"`, and `initiator="mcp-sampling"` — are **not** counted toward premium requests.

This was verified by controlled testing: sessions with many agent-initiated calls showed no additional premium request consumption beyond user-initiated calls. Background operations further reinforce this pattern by using free models (multiplier=0), making them effectively free regardless of initiator.

## `session.shutdown` Event (Session Totals)

(SDK-internal event — not consumed by the extension directly)

Emitted when the session ends. Contains aggregated metrics for the entire session. Unlike `assistant.usage`, this event **is** persisted to `events.jsonl`.

### Type Definition

From `session-events.ts` (the `type: "session.shutdown"` definition):

```typescript
{
  type: "session.shutdown";
  data: {
    shutdownType: "routine" | "error";
    errorReason?: string;

    /** Total number of premium API requests used during the session */
    totalPremiumRequests: number;
    /** Cumulative time spent in API calls during the session, in milliseconds */
    totalApiDurationMs: number;
    /** Unix timestamp (milliseconds) when the session started */
    sessionStartTime: number;

    /** Aggregate code change metrics */
    codeChanges: {
      linesAdded: number;
      linesRemoved: number;
      filesModified: string[];
    };

    /** Per-model usage breakdown, keyed by model identifier */
    modelMetrics: {
      [modelId: string]: {
        requests: {
          count: number; // Total API requests to this model
          cost: number;  // Cumulative cost multiplier (count × multiplier)
        };
        usage: {
          inputTokens: number;
          outputTokens: number;
          cacheReadTokens: number;
          cacheWriteTokens: number;
        };
      };
    };

    currentModel?: string;
    currentTokens?: number;
    systemTokens?: number;
    conversationTokens?: number;
    toolDefinitionsTokens?: number;
  };
}
```

### Understanding `modelMetrics`

The `modelMetrics` map provides a per-model breakdown of consumption:

- `requests.count` — Number of API calls to that model
- `requests.cost` — Cumulative premium requests consumed (`count × multiplier`)

#### Worked Example

```json
{
  "claude-opus-4.6": {
    "requests": { "count": 2, "cost": 6 },
    "usage": {
      "inputTokens": 45000,
      "outputTokens": 8000,
      "cacheReadTokens": 12000,
      "cacheWriteTokens": 3000
    }
  },
  "claude-haiku-4.5": {
    "requests": { "count": 4, "cost": 1.32 },
    "usage": {
      "inputTokens": 20000,
      "outputTokens": 4000,
      "cacheReadTokens": 0,
      "cacheWriteTokens": 0
    }
  }
}
```

Reading: Opus 4.6 made 2 API calls at 3x multiplier = 6 premium requests. Haiku 4.5 made 4 calls at 0.33x = 1.32 premium requests. Session total: **7.32 premium requests**.

> **Caveat:** `requests.count` includes **all** API calls (both user-initiated and agent-initiated). The `requests.cost` reflects the server-determined cost multiplied across all calls. In practice, only user-initiated calls (`initiator` absent or `initiator="user"`) contribute to actual billing. For example, if the 2 Opus calls comprised 1 user-initiated + 1 agent-initiated, the actual billing would be **3 premium requests**, not 6.

## `UsageMetrics` Interface (SDK)

> **SDK Reference:** The `UsageMetrics` interface is defined in the `copilot-sdk` package (`index.d.ts`), which is external to this repository. The schema shown here reflects the SDK at time of writing and cannot be verified from extension source alone.

From SDK (`index.d.ts`; line numbers are version-specific):

```typescript
interface UsageMetrics {
  /** Accumulated billing total: sum of all (cost × multiplier) */
  totalPremiumRequests: number;
  /** Raw count of user-initiated requests (not multiplied) */
  totalUserRequests: number;
  totalApiDurationMs: number;
  sessionStartTime: number;
  codeChanges: CodeChangeMetrics;
  modelMetrics: Map<string, ModelMetrics>;
  currentModel?: string;
  lastCallInputTokens: number;  // From most recent main-agent call
  lastCallOutputTokens: number; // From most recent main-agent call
}
```

> **Reliability note:** `totalPremiumRequests` from the `session.shutdown` event may be unreliable. The SDK's `disconnect()` method clears event handlers after sending `session.destroy`, creating a race condition that often prevents the shutdown event from being captured. Prefer in-session `assistant.usage` tracking via `UsageMetricsTracker` for accurate totals.

### Key Distinction

| Field | What It Counts | Multiplied? |
|-------|---------------|-------------|
| `totalPremiumRequests` | Billing total (the number that matters for quota) | Yes — each call weighted by model multiplier |
| `totalUserRequests` | Raw count of user prompts | No — straight count |

Example: 2 user prompts on Opus 4.6 (3x multiplier) yields `totalUserRequests = 2`, `totalPremiumRequests = 6`.

## `UsageMetricsTracker`

From SDK (`index.d.ts`; line numbers are version-specific):

```typescript
class UsageMetricsTracker {
  private _metrics: UsageMetrics;

  get metrics(): UsageMetrics;

  /** Process a session event to update running totals */
  processEvent(event: SessionEvent): void;
}
```

This class consumes `assistant.usage` events as they arrive and maintains running totals. It is the single source of truth for in-session premium request accounting.

## `UsageInfoEvent` (Context Window Monitoring)

From SDK (`index.d.ts`; line numbers are version-specific):

```typescript
type UsageInfoEvent = {
  kind: "usage_info";
  turn: number;           // Current turn number
  tokenLimit: number;     // Max context window tokens
  currentTokens: number;  // Current token count
  messagesLength: number; // Number of messages
  systemTokens?: number;
  conversationTokens?: number;
  toolDefinitionsTokens?: number;
  isInitial?: boolean;    // First event in session?
};
```

Emitted per-turn for context window monitoring. This event concerns token budget, **not** billing — it is distinct from the premium request tracking events above.

## Persisted vs Ephemeral Events

| Event | Ephemeral | Written to `events.jsonl` | Availability |
|-------|-----------|--------------------------|--------------|
| `assistant.usage` | Yes | No | Real-time only |
| `session.shutdown` | No | Yes | After session ends |

The `session.shutdown` event (which includes usage metrics) is the mechanism by which metrics are persisted to `events.jsonl`.

## The `/usage` Command

The copilot-cli `/usage` command reads `UsageMetrics` from the `UsageMetricsTracker` and displays:

1. Total premium requests consumed
2. Per-model breakdown (requests, cost, tokens)
3. Code change metrics (lines added/removed, files modified)
4. Session duration
5. Token consumption from sub-agents (since v0.0.422)
