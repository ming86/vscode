# Complete Catalog of Premium Request Consumption Sources

## Overview

This document catalogs every activity related to premium request consumption in the Copilot CLI / VS Code system. Understanding the billing model is critical for explaining unexpected consumption and optimizing usage.

> See [03-interaction-types.md](03-interaction-types.md) for interaction type definitions.

**Key finding:** Only **user-initiated prompts** are billed as premium requests (PRs). Sub-agent calls, background agents, session naming, and conversation compaction are either marked with non-billable initiator values or use free (0x multiplier) models, and do not contribute to premium request consumption.

**Billing formula:** `Premium requests consumed = Σ(model_multiplier) for each user-initiated prompt`

The billing system distinguishes calls by the `initiator` field in usage events. Only calls with `initiator=user` (or absent initiator, which defaults to user) are charged. Calls with `initiator=agent` or `initiator=sub-agent` are not billed.

| Model | Multiplier | Notes |
|-------|-----------|-------|
| GPT-5 mini, GPT-4.1, GPT-4o | 0x | Included models — free |
| Claude Haiku 4.5 | 0.33x | |
| Claude Sonnet 4 / 4.5 / 4.6 | 1x | |
| GPT-5 / GPT-5.1 | 1x | See [01-model-multipliers.md](01-model-multipliers.md) for the canonical list |
| Claude Opus 4.5 / 4.6 | 3x | |
| Any model via "Auto" selection | Variable discount on base | Server-controlled discount applied client-side in `calculateAutoModelInfo()` (`autoChatEndpoint.ts`); the server's `cost` field reflects the resulting charge. Discount is server-provided per model (typically ~10% for premium models; fallback is 0% when absent). |

> See [01-model-multipliers.md](01-model-multipliers.md) for the complete canonical table; this is an abbreviated subset.

> **Free plan exception:** Free plan users are charged 1 premium request per interaction regardless of model multiplier. The multiplier table above applies only to paid plans. (See also [00-overview.md](00-overview.md) and [01-model-multipliers.md](01-model-multipliers.md).)

## Source 1: User Prompts (Primary Consumption Source)

**InteractionType:** `"conversation-user"` *(CAPI-side interaction type — not present in client extension code, set by SDK/server)*
**Billing:** `model_multiplier` per user-initiated prompt

Each message the user types and sends to the model constitutes the sole significant billable event. Billing is determined server-side by CAPI based on the `initiator` field in the request: only calls where the initiator is `"user"` (or absent, which defaults to user) are charged. The `cost` field in `assistant.usage` events reflects the server-determined charge.

Subsequent agentic loop iterations within the same user turn (tool calls, retries, follow-up reasoning) are **not** separately billed — these are marked with `initiator=agent` and carry zero billing weight.

> **Verified by test:** 7 `sendAndWait` calls producing 8 total API calls (1 agent-initiated) resulted in a quota delta of exactly +6, matching the number of user-initiated prompts only. Similarly, 2 `sendAndWait` calls producing 5 total API calls resulted in a quota delta of exactly +2.

### Billing Logic

```
1. User sends message (initiator=user)
2. CAPI determines billing for the user turn:
   → initiator=user → bill model_multiplier
   → initiator=agent (agentic loop iterations, tool calls) → no bill
   → initiator=sub-agent (sub-agent calls) → no bill
3. The `cost` field in the assistant.usage event reflects the charge
```

### Examples

| Action | Model | Multiplier | Premium Requests |
|--------|-------|-----------|-----------------|
| Send "Hello" | Sonnet 4 | 1x | 1 |
| Send "Hello" | Opus 4.6 | 3x | 3 |
| Send "Hello" | GPT-4.1 | 0x | 0 |
| Send "Hello" (Auto selects Sonnet 4) | Sonnet 4 via Auto | Variable discount | Varies (e.g., 0.9 at typical ~10% server-provided discount) |
| Send prompt that triggers 5 tool calls and 3 agentic iterations | Sonnet 4 | 1x | 1 (iterations are free) |

## Source 2: Sub-Agents via Task Tool (NOT Separately Billed)

**InteractionType:** `"conversation-subagent"`
**Billing:** Not separately billed — sub-agent API calls are marked with `initiator=agent` or `initiator=sub-agent`, which are non-billable initiator values.

When the main agent uses the `task` tool to spawn sub-agents (explore, task, general-purpose, custom agents), each sub-agent makes independent API calls. However, these calls are **not charged as premium requests** because:

1. The `initiator` field is set to `agent` or `sub-agent`, and the billing system only charges calls with `initiator=user`.
2. The SDK implements a "cost guard" that may downgrade sub-agent models to free (0x multiplier) alternatives, further ensuring zero cost.

> **Verified by test:** Sessions with many sub-agents and multiple API calls show no additional premium request consumption beyond the user-initiated prompts. The `/usage` command confirms zero delta from sub-agent activity.

### Sub-Agent Types and Default Models

| Agent Type | Default Model | Typical Multiplier | Billing Impact |
|-----------|---------------|-------------------|----------------|
| `explore` | Haiku 4.5 | 0.33x | Not billed (initiator=agent) |
| `task` | Haiku 4.5 | 0.33x | Not billed (initiator=agent) |
| `general-purpose` | Sonnet 4 | 1x | Not billed (initiator=agent) |
| `helper-agent` | Varies (user-configured) | Varies | Not billed (initiator=agent) |
| `helper-subagent` | Varies (user-configured) | Varies | Not billed (initiator=agent) |

> **Note:** The `model` parameter on the `task` tool allows overriding the default model per invocation. However, since sub-agent calls are not billed regardless of model, the override affects quality and speed — not cost.

### Tracking

Sub-agent usage events include `parentToolCallId` linking back to the parent invocation:

```typescript
// Conceptual illustration (not a formal SDK type)
interface SubAgentUsageEvent {
    interactionType: "conversation-subagent";
    initiator: "agent" | "sub-agent";  // Non-billable initiator value
    parentToolCallId: string;  // Links to parent task tool invocation
    agentType: string;         // "explore" | "task" | "general-purpose" | custom
    model: string;
    turnIndex: number;  // (conceptual field — no direct evidence in extension source)
}
```

## Source 3: Sub-Agents in Background Mode (mode: "background") (NOT Separately Billed)

**InteractionType:** `"conversation-subagent"` (same non-billing path as regular sub-agents)
**Billing:** Not separately billed — background agents are sub-agents with non-billable initiator values (`initiator=agent`). They use the same default models as synchronous sub-agents (see Source 2).

Agents launched with `mode: "background"` in the task tool run independently and make their own API calls while the user continues working. However, these agents are **not separately billed** because:

1. Their API calls are marked with non-billable initiator values (`initiator=agent`), the same mechanism as synchronous sub-agents (Source 2).
2. They use the same default models as their synchronous counterparts: `explore`/`task` default to Haiku (0.33x), `general-purpose` to Sonnet (1x).
3. The billing system does not charge calls with non-user initiator values, regardless of the model's multiplier.

> **Verified by test:** User sessions with many background agents running concurrently show no additional premium request consumption in `/usage` output.

### Tracking

Background agents are listed in the `session.idle` event:

```typescript
backgroundTasks?: {
    agents: {
        agentId: string;
        agentType: string;
        description?: string;
    }[];
    shells: {
        shellId: string;
        description?: string;
    }[];
};
```

> **SDK caveat:** The `backgroundTasks` schema above is illustrative. The exact structure of `session.idle` event payloads is determined by the Copilot SDK/server and is not directly verifiable from the VS Code extension source code. The actual schema may differ.

## Source 4: Session Naming (Effectively Free)

**InteractionType:** `"conversation-background"`
**Billing:** Effectively free — the SDK uses free (0x multiplier) models for session naming.

After the first message in a session, the system automatically generates a session name/title using an LLM call. This call uses a free model and does not consume premium requests.

### Behavior

```
1. User sends first message in a new session
2. System triggers background LLM call to generate session title
   → Model: free model (0x multiplier), determined by SDK
   → Bill: 0 PRs
3. Title appears in session list
```

**Controllability:** Not user-controllable. Fires automatically on first message. No billing impact.

## Source 5: Conversation Compaction (Effectively Free)

**InteractionType:** `"conversation-background"`
**Billing:** Effectively free — the SDK uses free (0x multiplier) models for compaction operations.

When the context window reaches certain thresholds, the system automatically summarizes the conversation to free up space. Each compaction is an LLM API call, but it uses a free model and does **not** consume premium requests.

### Trigger Thresholds

From `src/extension/intents/node/agentIntent.ts`:

| Context Utilization | Behavior | Phase |
|--------------------|----------|-------|
| < 80% | No compaction | — |
| >= 80% | Kicks off async background compaction for future use | Post-render |
| >= 85% | Sets inline summarization flag on the prompt (no separate call) | Pre-render |
| >= 95% | Blocks and waits for compaction to complete before proceeding | Pre-render |
| >= 95% (post-render) | After rendering, if context still >= 95%, blocks on compaction before next turn | Post-render |

### Telemetry

Tracked via `backgroundSummarizationApplied` event:

```typescript
// Illustrative interface (not a formal SDK type)
interface BackgroundSummarizationEvent {
    trigger: "preRender" | "preRenderBlocked" | "postRenderBlocked" | "budgetExceededWaited" | "budgetExceededReady";
    outcome: "applied" | "noResult";
    contextRatio: number;  // How full the context window was (0.0–1.0)
}
```

> **Impact:** While long conversations trigger multiple compaction rounds, these do **not** silently consume premium requests. Compaction uses free models. The primary cost concern with long conversations remains the user-initiated prompts themselves, not the compaction overhead.

## Source 6: MCP Sampling (Server-Determined — Not Verified from Client Code)

**InteractionType:** `"conversation-sampling"` *(CAPI-side interaction type — not present in client extension code, set by SDK/server)*
**Billing:** Server-determined. The billing behavior of MCP sampling calls has not been independently verified from the client extension source code.

When an MCP server requests the client to sample the LLM (i.e., an MCP tool that needs to perform its own reasoning step), the sampling call may be billed as a premium request. However, the actual billing treatment — whether these are charged at full model multiplier, use free models, or are marked with a non-billable initiator — is determined server-side and cannot be confirmed from the VS Code extension source alone.

> **From SDK docs:** "Uses the sampling client pattern for proper CAPI correlation, billing, and usage telemetry."

**When:** Only when MCP servers explicitly request sampling via the `sampling/createMessage` protocol method.

**Impact:** Typically rare. The billing impact depends on server-side implementation details not visible in the client extension code.

## Source 7: Copilot Cloud Agent Sessions

**InteractionType:** Dedicated SKU (`copilot_cloud_agent_premium_requests`) *(server-side concept; this SKU identifier is not referenced in the VS Code extension source code)*
**Billing:** 1 premium request per session (from dedicated SKU)

Each cloud agent session (triggered by assigning a task or issue via the GitHub UI) consumes one premium request from a **separate** tracking SKU.

> **Note:** This uses a separate quota bucket from the main `premium_interactions` bucket. Cloud agent premium requests do not reduce the user's chat premium request allowance, and vice versa.

## Source 8: Inline Suggestions with Premium Models

**Billing:** Based on `completions` quota (separate from chat)

If premium models are configured for inline code completions, they consume from the `completions` quota bucket, not the chat `premium_interactions` bucket.

> **Note:** Most users rely on included models for completions, making this a rare consumption vector. It only applies when the user has explicitly opted into premium completion models.

## Source 9: Code Review

**Billing:** Premium requests consumed per review interaction

Using Copilot Code Review with premium models consumes premium requests. Each review interaction (e.g., requesting a review on a PR diff) constitutes a billable event.

## Source 10: Copilot Extensions / Spaces

**Billing:** Premium requests consumed based on model used

Third-party Copilot extensions may consume premium requests when they use non-included models.

**Exception:** Extension-contributed endpoints registered via the VS Code API are assigned a `0x` multiplier and are free.

## Scenario Analysis: "Why Did I Consume 6 PRs for 1 Prompt?"

**Given:** User sends 1 prompt using Opus 4.6 (3x multiplier). Dashboard shows 6 premium requests consumed.

### Scenario A: Two Rapid User Turns (Most Likely)

```
User prompt #1:          1 × 3  =  3 PRs  (previous turn, not yet reflected in quota)
User prompt #2:          1 × 3  =  3 PRs  (current turn)
                                   ──────
Total:                             6 PRs
```

The most common explanation. Users often send a follow-up message before the quota display updates from the previous turn.

### Scenario B: Quota Propagation Lag

```
Costs from earlier sessions or turns propagating to quota after a delay.
Quota updates are eventually consistent — can take seconds to minutes.
A single prompt cost 3 PRs, but 3 PRs from an earlier session arrived simultaneously.
```

### Scenario C: Quota Rounding and Accumulation

```
Multiple earlier fractional costs accumulating and rounding:
  Earlier Haiku turns:   3 × 0.33  =  0.99 PRs (rounds to 1)
  Earlier Sonnet turns:  2 × 1     =  2.00 PRs
  Current Opus turn:     1 × 3     =  3.00 PRs
                                      ──────
Apparent total since last check:       6 PRs
```

> **Note:** Sub-agents, background agents, session naming, and compaction are **not** the cause. These operations use non-billable initiator values or free models and do not contribute to premium request consumption. If you see unexpected consumption, look first at (a) multiple user turns, (b) quota propagation delay, or (c) accumulated costs from earlier sessions.

## Optimization Strategies

| Strategy | Mechanism | Savings |
|----------|-----------|---------|
| Use included models for routine tasks | GPT-5 mini, GPT-4.1, GPT-4o carry 0x multiplier | 100% on those turns |
| Use Auto model selection | Variable multiplier discount on Auto-selected calls (server-provided, typically ~10%) | Varies per call |
| Prefer Haiku for explore agents | 0.33x vs 1x (Sonnet) or 3x (Opus) if sub-agents were ever billed | Informational — sub-agents are currently free |
| Keep conversations short | Reduces overall token consumption and context pressure | Improves response quality; no direct PR savings from compaction (compaction is free) |
| Monitor with `/usage` command | Real-time consumption breakdown | Awareness, not direct savings |
| Be deliberate with user prompts | Each user-sent message is billed at the model's multiplier | Fewer turns = fewer PRs; combine requests where feasible |

## Summary of Consumption Vectors

| # | Source | InteractionType | Trigger | Billed? | Notes |
|---|--------|----------------|---------|---------|-------|
| 1 | User prompts | `conversation-user` | User sends message | **Yes** — model_multiplier per prompt | Only source with verified billing impact |
| 2 | Sub-agents | `conversation-subagent` | Agent spawns task tool | **No** — initiator=agent, not billed | SDK cost guard may also downgrade to free models |
| 3 | Background agents | `conversation-subagent` | Agent spawns background task | **No** — `initiator=agent`, not billed | Same mechanism as Source 2; uses same default models as synchronous sub-agents |
| 4 | Session naming | `conversation-background` | First message in session | **No** — uses free models (0x) | Automatic, not user-controllable |
| 5 | Compaction | `conversation-background` | Context window > 80% full | **No** — uses free models (0x) | Automatic, not user-controllable |
| 6 | MCP sampling | `conversation-sampling` | MCP server requests sampling | **Unverified** — server-determined | Not confirmed from client code |
| 7 | Cloud agent sessions | Dedicated SKU | Task/issue assignment | **Yes** — separate bucket | Does not reduce chat PR allowance |
| 8 | Inline suggestions | `completions` bucket | Typing in editor | **Yes** — separate bucket | Only with premium completion models |
| 9 | Code review | Per review interaction | Requesting review | **Yes** | Premium model dependent |
| 10 | Extensions / Spaces | Per model used | Extension invocation | **Yes** | Except 0x extension endpoints |
