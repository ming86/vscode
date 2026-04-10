# Copilot CLI Premium Request System — Overview

## Purpose

The Premium Request system governs how advanced Copilot model usage is metered and billed. Each user-initiated prompt that reaches the Copilot API (CAPI) constitutes a billable event. The cost of that event is determined by a **model multiplier** — a numeric weight assigned to each model by CAPI's model metadata API. The extension tracks consumption via quota headers and snapshot events returned from CAPI, surfacing warnings and hard stops when the user's monthly allowance is exhausted or approaching exhaustion.

## What Are Premium Requests?

A **premium request** is the billable unit consumed when using Copilot with advanced models. Key characteristics:

| Property | Detail |
|----------|--------|
| **Reset cadence** | Monthly — 1st of each month at 00:00:00 UTC |
| **Carry-over** | None — unused requests expire at reset |
| **Overage** | Some plans permit limited overage; controlled by `overage_permitted` flag in quota snapshot |
| **Tracking** | Server-side via CAPI; reported to the extension through response headers and event payloads |

### Plan Allowances

| Plan | Monthly Premium Requests | Notes |
|------|--------------------------|-------|
| **Free** | 50 | Every model costs 1 PR per interaction (no included models) |
| **Pro** | 300 | Included models (GPT-5 mini, GPT-4.1, GPT-4o) cost 0 PRs |
| **Pro+** | 1,500 | Same included-model benefit as Pro |
| **Business** | 300 per user | Organization-managed quota |
| **Enterprise** | 1,000 per user | Organization-managed quota |

## The Premium Request Formula

```
Premium Requests Consumed = User Prompts × Model Multiplier
```

Billing is **per user prompt**, not per API call. Only calls marked with `initiator=user` on the WebSocket message (set in `chatWebSocketManager.ts:604` via `initiator: options.userInitiated ? 'user' : 'agent'`) are billed. Subsequent agentic loop iterations, sub-agent calls, and background operations use `initiator=agent` and are not separately charged.

The `X-Interaction-Type` header provides additional classification, but the `initiator` field is the actual billing discriminator:

| Interaction Type | Billing (server-side) | Description |
|-----------------|----------------------|-------------|
| `conversation-user` | **Billable** | User-initiated prompt — the only billing-relevant type |
| `conversation-agent` | Not billable | Agentic loop continuation — tool calls, follow-up reasoning within the same turn |
| `conversation-subagent` | **Not separately billed** | Sub-agent API calls — marked `initiator=agent`, not charged; SDK cost guard may also force free models |
| `conversation-background` | **Effectively free** | Background operations (naming, compaction) use multiplier=0 models |
| `conversation-sampling` | **Server-determined** | MCP sampling — billing behavior not verified from client code |

> **Note:** Billing classification is determined server-side. The `initiator` field on WebSocket messages (`'user'` vs `'agent'`) is the actual billing discriminator, not the `X-Interaction-Type` header alone. The extension sets the header via the networking layer in `networking.ts` and `chatMLFetcher.ts` based on request kind (`subagent`, `background`, or default agent flow); the server uses the `initiator` field to decide whether to bill.
>
> **Evidence:** The `PremiumRequestProcessor` class is defined in the SDK but never instantiated — it is dead code. Verified billing tests show quota deltas matching only `initiator=user` calls (e.g., 7 sendAndWait calls producing 8 API calls with 1 agent-initiated, measured quota delta = +6, matching user calls only).

> **See [03-interaction-types.md](03-interaction-types.md) for the complete breakdown.**

## Model Multiplier System

Each model exposes a `billing` object from the CAPI model metadata API:

```typescript
billing?: {
  is_premium: boolean;
  multiplier: number;
  restricted_to?: string[];
}
```

The `multiplier` determines cost per billable request:

| Model | Multiplier | Cost per Prompt (Paid Plan) |
|-------|------------|----------------------------|
| GPT-5 mini | 0× | 0 PRs (included) |
| GPT-4.1 | 0× | 0 PRs (included) |
| GPT-4o | 0× | 0 PRs (included) |
| Claude Haiku 4.5 | 0.33× | 0.33 PRs |
| Claude Sonnet family (4, 4.5, 4.6) | 1× | 1 PR |
| Claude Opus 4.5 | 3× | 3 PRs |
| Claude Opus 4.6 | 3× | 3 PRs |
| GPT-5 | 1× | 1 PR |

> See [01-model-multipliers.md](01-model-multipliers.md) for the complete list including all GPT-5 variants.

### Auto Model Discount

When the user selects **Auto** model (delegating model choice to Copilot), a discount is applied to the multiplier. The discount is server-controlled per model via the auto-mode session token (`token.discounted_costs?.[selectedModel.model]`). When absent, the fallback is **0%** (no discount). Observed values are typically ~10% for premium models, but this is not hardcoded.

> See [01-model-multipliers.md](01-model-multipliers.md) for the discount calculation.

### Free Plan Exception

Free plan users are charged **1 premium request per interaction** regardless of model multiplier. There are no included models on the Free plan.

## Sources of Premium Consumption

Only user-initiated prompts consume premium requests. Other API call types are either not billed or effectively free:

| Source | Interaction Type | Billed? | Notes |
|--------|-----------------|---------|-------|
| **User prompt** | `conversation-user` | **Yes** | Primary and typically sole consumption source — each prompt × model multiplier |
| **Sub-agents** | `conversation-subagent` | **No** | Marked `initiator=agent`; SDK cost guard downgrades to free models when session model is free |
| **Background operations** | `conversation-background` | **Effectively no** | Always use multiplier=0 (free) models; even if technically "billed", cost is 0 |
| **MCP sampling** | `conversation-sampling` | **Server-determined** | Billing behavior not verified from client code |
| **Background agents** | `conversation-subagent` | **No** | Same as sub-agents — marked `initiator=agent`, not separately charged |
| **Copilot Cloud Agent sessions** | Dedicated SKU | **Flat** | Each session = 1 premium request in dedicated `premium_requests` SKU (separate billing mechanism) |

## Why Consumption Exceeds Expectations

A common scenario: "I sent 1 prompt but 6 premium requests were consumed."

Because billing is per user prompt and sub-agents/background operations are not separately billed, the most likely explanations involve multiple user turns or quota propagation, not hidden background costs.

**Scenario A — Two rapid user turns (most likely):**

| Event | Calculation | PRs |
|-------|-------------|-----|
| First user prompt (Opus 4.6) | 1 × 3 | 3 |
| Second user prompt (Opus 4.6) | 1 × 3 | 3 |
| **Total** | | **6** |

The user may not recall both turns, or the second turn may have been triggered by a retry or edit-and-resubmit.

**Scenario B — Quota propagation lag:**

Quota snapshots are eventually consistent. Costs from earlier sessions or turns may still be propagating when `/usage` is checked. A delta observed after a single prompt may include costs from prior activity that had not yet been reflected in the snapshot.

> **Note:** Sub-agents, background compaction, and session naming do **not** add to the premium request count. They use `initiator=agent` (not billed) or multiplier=0 models (zero cost). Verified by discriminator tests: many sub-agents and compaction events produced zero additional quota consumption beyond user-initiated calls.

## Architecture Overview

```
┌──────────────────────────────────────────────────────────────────────────┐
│                          User Sends Prompt                               │
└──────────────────────┬───────────────────────────────────────────────────┘
                       │
                       ▼
┌──────────────────────────────────────────────────────────────────────────┐
│                     Networking Layer                                      │
│                                                                          │
│  networking.ts / chatMLFetcher.ts                                        │
│                                                                          │
│  Determines interaction type from request kind:                          │
│    kind === 'subagent'    → 'conversation-subagent'                      │
│    kind === 'background'  → 'conversation-background'                    │
│    else → passes through the intent value as-is                          │
│                                                                          │
│  networking.ts:                                                          │
│    (same — passes through the raw intent value)                          │
│  chatMLFetcher.ts:                                                       │
│    else → sets undefined (omits header, deferring to the SDK)            │
│                                                                          │
│  Sets header: X-Interaction-Type: <type>                                 │
│                                                                          │
│  WebSocket path (chatWebSocketManager.ts:604):                           │
│    initiator: options.userInitiated ? 'user' : 'agent'                   │
│    ← This is the actual billing discriminator sent on WS messages        │
└──────────────────────┬───────────────────────────────────────────────────┘
                       │
                       ▼
┌──────────────────────────────────────────────────────────────────────────┐
│                     CAPI (Copilot API)                                    │
│                                                                          │
│  Receives request with:                                                  │
│    X-Interaction-Type: "conversation-user" | "conversation-subagent" ... │
│    initiator: "user" | "agent"  (on WebSocket messages)                  │
│                                                                          │
│  Returns response with quota information:                                │
│    Headers:                                                              │
│      x-quota-snapshot-premium_models    (URL-encoded quota state)        │
│      x-quota-snapshot-premium_interactions  (fallback)                   │
│      x-quota-snapshot-chat              (Free plan users)                │
│                                                                          │
│    Payload:                                                              │
│      copilot_quota_snapshots in response.completed event                 │
└──────────────────────┬───────────────────────────────────────────────────┘
                       │
                       ▼
┌──────────────────────────────────────────────────────────────────────────┐
│                     ChatQuotaService                                      │
│                                                                          │
│  chatQuotaServiceImpl.ts                                                 │
│                                                                          │
│  processQuotaHeaders(headers):                                           │
│    Parses URL-encoded header value → decodes keys:                       │
│      ent      → entitlement (total allowance)                            │
│      ov       → overage_count                                            │
│      ovPerm   → overage_permitted (boolean)                              │
│      rem      → percent_remaining                                        │
│      rst      → reset_date (ISO 8601)                                    │
│                                                                          │
│  processQuotaSnapshots(snapshots):                                       │
│    Looks up premium_models first, falls back to premium_interactions     │
│                                                                          │
│  Emits quota state → consumed by UI for warnings / hard stops            │
└──────────────────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────────────────┐
│                     Endpoint Layer                                        │
│                                                                          │
│  ChatEndpoint (chatEndpoint.ts)                                          │
│    isPremium?: boolean       ← from modelMetadata.billing.is_premium     │
│    multiplier?: number       ← from modelMetadata.billing.multiplier     │
│    restrictedToSkus?: string[] ← from modelMetadata.billing.restricted_to│
│                                                                          │
│  AutoChatEndpoint (autoChatEndpoint.ts)                                  │
│    Extends CopilotChatEndpoint (which extends ChatEndpoint)              │
│    Discount computed by calculateAutoModelInfo() — standalone function   │
│    Formula: multiplier × (1 - discountPercent)                           │
│    discountPercent is server-provided per model; fallback is 0 (none)    │
└──────────────────────────────────────────────────────────────────────────┘
```

## Quota Snapshot Structure

The `QuotaSnapshot` type represents the state of a user's quota as reported by CAPI:

```typescript
// chatQuotaService.ts
interface QuotaSnapshot {
  readonly entitlement: string;        // String representation of the entitlement count, "-1" for unlimited
  readonly percent_remaining: number;  // 0–100, percentage of quota remaining
  readonly overage_permitted: boolean;
  readonly overage_count: number;
  readonly reset_date?: string;        // ISO 8601 — next reset timestamp, if applicable
}
```

Quota snapshots are keyed by SKU name. The primary keys are:

| SKU Key | Description |
|---------|-------------|
| `premium_models` | Premium model usage (primary) |
| `premium_interactions` | Premium interactions (fallback) |
| `completions` | Code completions quota |
| `chat` | Chat quota (Free plan users) |

## Key Source Files

### Extension (vscode-copilot-chat)

| File | Component | Purpose |
|------|-----------|---------|
| `src/platform/chat/common/chatQuotaService.ts` | `IChatQuotaService`, `QuotaSnapshot`, `CopilotUserQuotaInfo` | Service interface and quota types |
| `src/platform/chat/common/chatQuotaServiceImpl.ts` | `ChatQuotaService` | Implementation — header parsing, snapshot processing, quota state emission |
| `src/platform/endpoint/common/endpointProvider.ts` | `IModelAPIResponse` | Type definition for model metadata including `billing` object |
| `src/platform/endpoint/node/chatEndpoint.ts` | `ChatEndpoint` | Endpoint with `isPremium`, `multiplier`, `restrictedToSkus` properties |
| `src/platform/endpoint/node/autoChatEndpoint.ts` | `AutoChatEndpoint` | Auto model selection with billing multiplier discount calculation |
| `src/platform/chat/common/commonTypes.ts` | `ChatLocation`, `ChatFetchResponseType`, `ChatFetchError` | Enums, error type unions, and helpers used across quota error handling and response classification paths |
| `src/platform/networking/common/networking.ts` | Networking layer | Sets `X-Interaction-Type` header based on request kind |
| `src/extension/prompt/node/chatMLFetcher.ts` | `ChatMLFetcher` | Alternative interaction type header injection path |
| `src/platform/github/common/githubAPI.ts` | `SessionInfo` | Contains `premium_requests` field for Cloud Agent session tracking |

### SDK (copilot-sdk — referenced, not in this repository)

| File | Component | Purpose |
|------|-----------|---------|
| `nodejs/src/generated/session-events.ts` | `assistant.usage` event | Token usage reporting per turn |
| `nodejs/src/generated/rpc.ts` | `AccountGetQuotaResult`, `models.list` | Quota RPC and model listing with billing metadata |
| `nodejs/src/types.ts` | `ModelBilling`, `ModelInfo` | Type definitions for model billing information |

**Note:** The SDK source files are not present in this repository. The `chat-lib/` directory contains only build configuration. SDK types are referenced via compiled dependencies.

## Relationship to Other Systems

```
┌─────────────────────────────┐     ┌──────────────────────────────┐
│  Premium Request System      │     │  Memory Tools                │
│  (this document set)         │     │  (.local/docs/memory_tools/) │
│                              │     │                              │
│  Governs: cost metering,     │     │  Governs: persistent notes,  │
│  quota tracking, billing     │     │  context injection           │
│  multipliers                 │     │                              │
└──────────────┬──────────────┘     └──────────────────────────────┘
               │
               │  Quota exhaustion affects
               ▼
┌──────────────────────────────┐     ┌──────────────────────────────┐
│  Session System              │     │  Copilot Cloud Agent         │
│  (.local/docs/copilot-cli-  │     │  Sessions                    │
│   session/)                  │     │                              │
│                              │     │  Each session = 1 PR in      │
│  assistant.usage events      │     │  dedicated premium_requests  │
│  session.shutdown metrics    │     │  SKU                         │
└──────────────────────────────┘     └──────────────────────────────┘
```

## Document Index

| Document | Contents |
|----------|----------|
| [00-overview.md](00-overview.md) | This file — system overview, architecture, source inventory |
| [01-model-multipliers.md](01-model-multipliers.md) | Model billing metadata, multiplier system, auto model discount logic |
| [02-quota-tracking.md](02-quota-tracking.md) | Quota headers, snapshots, exhaustion handling, reset mechanics |
| [03-interaction-types.md](03-interaction-types.md) | `InteractionType` system — what counts as billable, header injection paths |
| [04-usage-events.md](04-usage-events.md) | `assistant.usage` events, session metrics, token reporting |
| [05-consumption-sources.md](05-consumption-sources.md) | Comprehensive catalog of all sources that consume premium requests |
| [06-quota-api.md](06-quota-api.md) | `account.getQuota` RPC, quota monitoring, and proactive quota checks |
| [07-implementation-guide.md](07-implementation-guide.md) | How to implement premium request tracking in new features |
