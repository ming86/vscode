# Interaction Types and Billable Request Classification

## Overview

This document describes the `X-Interaction-Type` header system that governs how CAPI requests are classified for billing. The interaction type determines whether a request counts as a premium request and at what multiplier. Understanding the boundary between what the **extension** controls and what the **SDK** controls is essential — this document calls out that boundary explicitly throughout.

## The InteractionType System

Every API call to the Copilot API (CAPI) includes an `X-Interaction-Type` HTTP header that classifies the request. This header is the primary mechanism for billing determination — it tells the backend whether a given request should be metered against the user's premium request allowance, and at what multiplier.

From the SDK `InteractionType` type definition (`@github/copilot/sdk/index.d.ts`):

```typescript
/** Interaction type for CAPI telemetry, sent as X-Interaction-Type header. */
type InteractionType =
  /** Main agent loop processing a user message */
  "conversation-agent"
  /** Sub-agent invoked by the task tool (explore, code-review, custom agents) */
  | "conversation-subagent"
  /** MCP sampling request executed on behalf of an MCP server */
  | "conversation-sampling"
  /** Background operations (session naming, compaction) */
  | "conversation-background"
  /** First billable request in a user-initiated turn */
  | "conversation-user";
```

> **Note:** The `"conversation-user"` value is defined in the SDK type union but is **never explicitly set** as an `X-Interaction-Type` header by any extension code path. The billing behavior it was meant to represent — charging the first call per user turn — is implemented via the `initiator` field on WebSocket messages instead. See [the PremiumRequestProcessor section below](#the-premiumrequestprocessor-sdk-internal-not-in-extension-source) for details.

> **SDK Reference:** The `InteractionType` union type is defined in the `copilot-sdk` package, which is external to this repository. The schema shown here reflects the SDK at time of writing and cannot be verified from extension source alone.

## CAPI Request Context Headers

Each CAPI request carries a set of correlation headers defined by the SDK `CAPIRequestContext` type definition:

```typescript
type CAPIRequestContext = {
  /** The type of interaction — agent, subagent, sampling, background, or user-initiated. */
  interactionType: InteractionType;
  /** A unique GUID for this agent instance, sent as X-Agent-Task-Id. */
  agentTaskId: string;
  /** The parent agent's task ID, sent as X-Parent-Agent-Id. Only set for subagents. */
  parentAgentTaskId?: string;
  /** The client session ID, sent as X-Client-Session-Id. */
  clientSessionId?: string;
  /** Per-message interaction ID, sent as X-Interaction-Id. */
  interactionId?: string;
  /** The client machine ID, sent as X-Client-Machine-Id. */
  clientMachineId?: string;
};
```

These headers serve three purposes: billing attribution, telemetry correlation, and parent-child relationship tracking for sub-agent hierarchies.

> **Extension vs. SDK headers:** The extension (`src/`) sets only `X-Agent-Task-Id`, `X-Interaction-Type`, and `X-Interaction-Id`. The remaining headers — `X-Parent-Agent-Id`, `X-Client-Session-Id`, and `X-Client-Machine-Id` — are set by the SDK layer and do not appear in extension source code.

## Interaction Type Reference

| Type | Billable | Set By | Trigger |
|------|----------|--------|---------|
| `conversation-user` | YES | Not explicitly set — billing determined by `initiator='user'` field (set by extension on WebSocket messages) | First API call in a user-initiated turn |
| `conversation-agent` | NO | Agent loop | Subsequent API calls within the same agentic loop |
| `conversation-subagent` | **NO** (not separately billed) | Task tool | Sub-agent API calls — marked `initiator=agent`, not charged |
| `conversation-background` | **Effectively NO** (uses free models) | Session management | Uses multiplier=0 models, so cost is 0 |
| `conversation-sampling` | **Server-determined** (not verified from client code) | MCP server handler | MCP server sampling — billing behavior unverified |

## Detailed Breakdown

### 1. `"conversation-user"` — The Primary Billable Request

- **Set by**: The `initiator` field on the WebSocket message (set by the extension)
- **When**: The first API call in a user-initiated turn
- **Billing**: YES — this is the canonical "premium request"
- **Multiplier**: Determined by the model selected (e.g., Sonnet 4 = 1x, Opus 4.5/4.6 = 3x; see [01-model-multipliers.md](01-model-multipliers.md) for the full table)

The actual billing discriminator is the `initiator` field on WebSocket messages. In the WebSocket path (`chatWebSocketManager.ts:604`), the extension sets `initiator: options.userInitiated ? 'user' : 'agent'`. User-initiated calls send `initiator: 'user'`; all agent-initiated calls (including sub-agents and background) send `initiator: 'agent'`.

> **Note:** The `PremiumRequestProcessor` is defined in the SDK type system but is **dead code** — it is never instantiated, called, or used in the current implementation. The billing classification described in earlier versions of this document (header reclassification by `PremiumRequestProcessor`) does not occur in practice. See [the PremiumRequestProcessor section below](#the-premiumrequestprocessor-sdk-internal-not-in-extension-source) for details.

This is the mechanism behind GitHub's documentation claim that "only prompts you send count as premium requests."

### 2. `"conversation-agent"` — Agentic Loop Continuation

- **Set by**: The extension sets `"conversation-agent"` as the interaction type for **all** agent-mode requests. Billing determination is handled by the `initiator` field on WebSocket messages (`chatWebSocketManager.ts:604`), not by interaction type reclassification. The `PremiumRequestProcessor` (originally intended to reclassify the first call) is dead code — see [the dedicated section below](#the-premiumrequestprocessor-sdk-internal-not-in-extension-source).
- **When**: Any API call within an agentic loop
- **Billing**: NO — per GitHub documentation: "actions Copilot takes autonomously to complete your task, such as tool calls, do not [count]"

A single user prompt may trigger 5–15 agentic loop iterations (model call → tool execution → model call → ...). Only the first call is billed; every subsequent model invocation within the same turn carries `"conversation-agent"` and is not metered.

### 3. `"conversation-subagent"` — Sub-Agent Calls

- **Set by**: The task tool when spawning sub-agents (explore, task, general-purpose, custom agents)
- **When**: Any API call originating from a sub-agent context
- **Billing**: **NO — sub-agents are not separately billed.** They are marked as `initiator=agent` (or `initiator=sub-agent`) on the WebSocket message, neither of which is charged by the billing system. Only `initiator=user` calls are metered.
- **Identification**: Each sub-agent receives a unique `X-Agent-Task-Id` and sets `X-Parent-Agent-Id` to the parent's task ID

The SDK also implements a "cost guard" that downgrades sub-agents to free models when the session model is free, further ensuring sub-agent operations do not consume premium requests.

Test evidence confirms this: spawning multiple sub-agents from a single user prompt does not increase the quota delta beyond what the parent's user-initiated calls consume. For example, 7 `sendAndWait` calls producing 8 API calls (1 agent-initiated) resulted in a quota delta of +6 — matching user-initiated calls only, with no additional charge for sub-agent activity.

> **Note:** Model defaults for sub-agents are set by the Copilot CLI agent framework, not the extension.

### 4. `"conversation-background"` — Silent Background Operations

- **Set by**: Session management code
- **When**: Automatic operations not directly tied to user prompts
- **Billing**: **Effectively NO — background operations use multiplier=0 (free) models.** The SDK explicitly selects free models for background tasks (the bundled `app.js` contains `t.filter(r=>r.billing?.multiplier===0)`). Even if technically "billed", the cost is 0 × multiplier = 0 premium requests.

Background operations include:

| Operation | Trigger | Typical Timing |
|-----------|---------|----------------|
| **Session naming** | After the first message in a session | Immediately after turn completes |
| **Conversation compaction / summarization** | Context window reaches ~80% capacity | During or after long turns |

> **Note:** In the extension source, compaction and summarization are the same operation, handled by `BackgroundSummarizer`. They are not separate background tasks.

### 5. `"conversation-sampling"` — MCP Sampling Requests

- **Set by**: MCP server handlers
- **When**: An MCP server invokes the client's sampling capability to perform LLM reasoning
- **Billing**: **Server-determined — billing behavior for MCP sampling has not been verified from client code.** The request flows through CAPI correlation and usage telemetry infrastructure, but whether it is actually metered against the user's premium request allowance is determined server-side and cannot be confirmed from the extension or SDK source alone.

> **Note:** The extension does not set or intercept this interaction type. `conversation-sampling` is handled entirely within the SDK.

## Request Kind in vscode-copilot-chat

The extension uses a `kind` discriminator on request options (defined in `src/platform/networking/common/networking.ts`) to control which `X-Interaction-Type` header is sent:

```typescript
interface IBackgroundRequestOptions {
  readonly kind: 'background';   // → X-Interaction-Type: "conversation-background"
}

interface ISubagentRequestOptions {
  readonly kind: 'subagent';      // → X-Interaction-Type: "conversation-subagent"
}
```

These `requestKindOptions` are passed through the networking layer and translated into the appropriate header before the CAPI call is dispatched.

## The PremiumRequestProcessor (SDK-internal, not in extension source)

The `PremiumRequestProcessor` is defined in the SDK type system but is **dead code** — it is never instantiated, called, or used in the current implementation.

The actual billing determination happens via the `initiator` field on WebSocket messages and server-side logic in CAPI. Calling code does not explicitly set `"conversation-user"` — the billing classification is determined by the `initiator` field value (`'user'` vs `'agent'`), which the extension sets based on `options.userInitiated` at the WebSocket layer.

The responsibilities previously attributed to this processor (detecting the first API call in a turn, reclassifying interaction types) are not performed by any code path in the current implementation. The `X-Interaction-Type` header system remains in place for telemetry and correlation purposes, but billing metering is governed by the `initiator` field.

## Request Flow: Single User Prompt

```
User sends message
    │
    ▼
Turn Start
    │
    ├── API Call #1 (InteractionType: "conversation-user")     ← BILLED × model multiplier
    │   Model responds with tool calls
    │
    ├── Tools execute (bash, grep, edit, etc.)                 ← NOT billed (no API call)
    │
    ├── API Call #2 (InteractionType: "conversation-agent")    ← NOT billed
    │   Model responds with more tool calls
    │
    ├── Tools execute again                                    ← NOT billed
    │
    ├── API Call #3 (InteractionType: "conversation-agent")    ← NOT billed
    │   Model responds with final text (no tool calls)
    │
    └── Turn End
         │
         ├── Session Naming (InteractionType: "conversation-background")  ← EFFECTIVELY FREE (multiplier=0 model)
         │
         └── Maybe Compaction (InteractionType: "conversation-background") ← EFFECTIVELY FREE (multiplier=0 model)
```

**Total for this prompt:** 1 user request (at model multiplier). Background operations use free models and do not consume premium requests.

## Request Flow: With Sub-Agents

```
User sends message
    │
    ▼
Main Agent Turn
    │
    ├── API Call #1 (InteractionType: "conversation-user")          ← BILLED × multiplier
    │   Model decides to use task tool with 3 explore agents
    │
    ├── Sub-Agent 1 (InteractionType: "conversation-subagent")
    │   ├── API Call (initiator=agent)                              ← NOT SEPARATELY BILLED
    │   ├── Tool execution → API Call (agent loop)                  ← NOT billed
    │   └── ...may do multiple agentic iterations
    │
    ├── Sub-Agent 2 (InteractionType: "conversation-subagent")
    │   ├── API Call (initiator=agent)                              ← NOT SEPARATELY BILLED
    │   └── ...
    │
    ├── Sub-Agent 3 (InteractionType: "conversation-subagent")
    │   ├── API Call (initiator=agent)                              ← NOT SEPARATELY BILLED
    │   └── ...
    │
    ├── API Call #2 (InteractionType: "conversation-agent")         ← NOT billed
    │   Model processes sub-agent results, responds to user
    │
    └── Turn End
```

**Total:** 1 × main model multiplier. Sub-agent calls are not separately billed (marked `initiator=agent`). Background operations use free models.

## Correlation Header Summary

Every CAPI request includes the following headers for tracking and billing:

| Header | Source | Set By | Purpose |
|--------|--------|--------|---------|
| `X-Interaction-Type` | `InteractionType` type value | Extension | Billing classification |
| `X-Agent-Task-Id` | Unique GUID per agent instance | Extension | Agent identity |
| `X-Interaction-Id` | Per-message GUID | Extension | Message-level correlation |
| `X-Parent-Agent-Id` | Parent's `agentTaskId` | SDK | Sub-agent lineage (subagents only) |
| `X-Client-Session-Id` | Session identifier | SDK | Session correlation |
| `X-Client-Machine-Id` | Machine identifier | SDK | Device attribution |

## Key Invariants

1. **One `conversation-user` per user-initiated turn** — determined by the `initiator` field on the WebSocket message, not by `PremiumRequestProcessor`.
2. **Agentic loop iterations are free** — `conversation-agent` calls are not metered.
3. **Sub-agents are NOT separately billed** — they use `initiator=agent` and are not charged. The SDK's cost guard may further downgrade them to free models.
4. **Background operations use free models (multiplier=0) and are effectively free** — even if they pass through billing infrastructure, the cost is 0.
5. **Tool execution is never billed** — only LLM API calls carry interaction types; tool invocations (bash, grep, edit) are local operations with no CAPI call.

For per-call usage event tracking, see [04-usage-events.md](04-usage-events.md). For the complete consumption catalog, see [05-consumption-sources.md](05-consumption-sources.md).
