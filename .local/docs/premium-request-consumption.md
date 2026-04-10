# Premium Request Consumption Model

## Overview

GitHub Copilot premium requests are billed **per user turn**, not per tool calling iteration. Understanding this model is critical for quota management in agent mode.

## Core Billing Equation

```
Premium_Requests_Consumed = (User_Turns) × (Model_Multiplier)
```

**NOT:**
```
Premium_Requests_Consumed ≠ (Iterations) × (Model_Multiplier)  ❌ INCORRECT
```

## What Constitutes "One Premium Request"

**One premium request = One complete conversation turn:**
- User sends a message
- Agent responds (with 0 to N tool calling iterations)
- Response completes

**Regardless of:**
- Number of tool calls executed
- Number of iterations (could be 1, 100, or 200)
- How long the response took
- How many files were edited

## Model Multiplier System

Premium models consume quota at different rates based on a `multiplier` field in their metadata.

### Multiplier Definition

From `src/platform/chat/chatEndpoint.ts`:

```typescript
interface IChatModelInformation {
    billing?: {
        is_premium: boolean;
        multiplier: number;        // Cost factor per turn
        restricted_to?: string[];  // SKU restrictions
    };
}
```

### Multiplier Examples

| Model Type | Multiplier | Quota Per Turn |
|------------|-----------|----------------|
| Free models (o1-mini, etc.) | 0 | 0 (unlimited) |
| Base premium models | 1 | 1 request |
| Advanced models | 2 | 2 requests |
| High-cost models | 5 | 5 requests |

**UI Tooltip** (from `src/extension/commands/chatModelsWidget.ts`):
> "Every chat message counts {multiplier} towards your premium model request quota"

"Chat message" = user turn, not individual LLM iteration.

## Quota Structure

From `src/platform/chat/chatQuotaServiceImpl.ts`:

```typescript
interface IChatQuota {
    quota: number;           // Total monthly entitlement
    used: number;            // Premium interactions consumed
    unlimited: boolean;      // Unlimited quota flag
    overageUsed: number;     // Additional paid requests beyond quota
    overageEnabled: boolean; // Can accrue paid requests
    resetDate: Date;         // When quota resets
}
```

**Quota types:**
- **Free tier:** `x-quota-snapshot-chat` header
- **Premium tier:** `x-quota-snapshot-premium_models` or `x-quota-snapshot-premium_interactions`

**Server-side tracking:** Authoritative quota state returned in response headers from `copilot_internal/user` endpoint.

## Iteration Consumption Examples

### Scenario 1: Single Ask Mode Message

```
User: "Explain this code"
Agent: Responds with explanation (no tools)

Iterations: 0
Premium requests consumed: 1 × multiplier
```

### Scenario 2: Agent Mode with 10 Iterations

```
User: "Fix the authentication bug"
Agent: 10 tool calling iterations
  - Iteration 1: semantic_search
  - Iteration 2: read_file (3 files)
  - Iteration 3: grep_search
  - Iterations 4-8: More investigation
  - Iteration 9: insert_edit_into_file
  - Iteration 10: get_errors, read_file

Iterations: 10
Premium requests consumed: 1 × multiplier
```

**All 10 iterations = single premium request.**

### Scenario 3: Agent Mode with 200 Iterations

```
User: "Refactor the entire authentication system"
Agent: 200 tool calling iterations (reaches limit)

Iterations: 200
Premium requests consumed: 1 × multiplier
```

**Even at the limit, still just 1 premium request.**

### Scenario 4: Continuation (New Turn)

```
User: "Refactor the entire authentication system"
Agent: 200 iterations, hits limit, shows confirmation

User: [Clicks "Continue"]
Agent: 100 more iterations (new limit: 300)

Total iterations: 300
Total turns: 2 (initial + continuation)
Premium requests consumed: 2 × multiplier
```

**Continuation creates new turn = additional premium request.**

### Scenario 5: Multiple Continuations

```
Turn 1: 200 iterations → 1 × multiplier
Turn 2 (continue): 300 iterations → 1 × multiplier
Turn 3 (continue): 450 iterations → 1 × multiplier
Turn 4 (continue): 675 iterations → 1 × multiplier

Total iterations: 1,625
Total turns: 4
Premium requests consumed: 4 × multiplier
```

With `multiplier=2`: **8 premium requests total**.

## Quota Consumption Table

| User Action | Iterations | Turns | Premium Requests (multiplier=1) | Premium Requests (multiplier=2) |
|-------------|-----------|-------|----------------------------------|----------------------------------|
| Ask mode question | 0 | 1 | 1 | 2 |
| Agent: 10 iterations | 10 | 1 | 1 | 2 |
| Agent: 200 iterations | 200 | 1 | 1 | 2 |
| Agent: 200 iter + 1 continue | 500 | 2 | 2 | 4 |
| Agent: 200 + 3 continues | 1,625 | 4 | 4 | 8 |
| Free model (multiplier=0) | Any | Any | 0 | 0 |

## Why Iterations Don't Count Individually

### Architectural Evidence

From `src/platform/chat/toolCallingLoop.ts`:

Tool call iterations are internal execution loops within a single conversation turn. They represent:
- Autonomous agent decision-making
- Tool selection and invocation
- Result processing and continuation

**All part of the same "response" to the user's message.**

### Telemetry Tracking

From `src/extension/conversation/chatParticipantTelemetry.ts`:

**Separate metrics:**
- `turn` / `externalTurnNumber`: Conversation turn count (correlates to premium requests)
- `toolCallRound`: Current iteration within turn
- `numToolCallIterations`: Total iterations in this turn

**Premium requests are counted by turn number, not iteration count.**

## Quota Exhaustion Scenarios

### During Agent Mode Execution

From `src/platform/chat/commonTypes.ts`:

When quota is exhausted mid-iteration:
1. Language model API returns quota error
2. `ChatFetchResponseType.Success` becomes `ChatFetchResponseType.QuotaExceeded`
3. Tool calling loop breaks immediately
4. User sees quota exhaustion message

**Effect:** Agent stops even if iteration limit (200) not reached.

### Error Messages by Tier

**Free Tier:**
> "You've reached your monthly chat messages quota. Upgrade to Copilot Pro for unlimited messages and access to premium models."

**Individual Tier:**
> "You've exhausted your premium model quota. Please enable additional paid premium requests, upgrade to Copilot Pro+, or wait for your allowance to renew on {date}."

**Pro Tier:**
> "You've exhausted your premium model quota. Please enable additional paid premium requests or wait for your allowance to renew on {date}."

**Enterprise Tier:**
> Contact organization administrator message

### Overage Limit Reached

When paid overages are exhausted:
> "You cannot accrue additional premium requests at this time. Please contact GitHub Support to continue using Copilot."

## Quota Protection via Iteration Limit

### Independent Systems

The iteration limit (200) and quota system are **separate mechanisms**:

| Purpose | Iteration Limit | Quota System |
|---------|----------------|--------------|
| **What it prevents** | Runaway loops, excessive latency | Billing overage, abuse |
| **Enforcement** | Client-side (extension) | Server-side (API) |
| **Scope** | Per-turn iterations | Account-wide requests |
| **User control** | Can extend via confirmation | Cannot extend (except paid) |

**Relationship:** Iteration limit **does not guarantee quota availability**.

### Quota Can Be Exhausted Before Limit

**Example calculation:**

User with:
- Monthly quota: 100 premium interactions
- Model multiplier: 2
- Each turn consumes: 2 requests

**Scenario:**
- 50 agent mode sessions (any iteration count)
- 50 sessions × 2 requests/session = 100 requests
- **Quota exhausted** regardless of iterations per session

If user continues agent sessions:
- Each continuation = +2 requests
- After 25 sessions with 1 continuation each = quota exhausted

## Auto-Fallback to Base Model

From `src/extension/conversation/chatParticipants.ts`:

When premium quota is exhausted:
1. Extension checks `quotaExhausted` status
2. Auto-switches to base endpoint (`copilotToken.endpoint`)
3. Shows warning message with option to enable paid requests

```typescript
get quotaExhausted(): boolean {
    return this._quotaInfo.used >= this._quotaInfo.quota
        && !this._quotaInfo.overageEnabled
        && !this._quotaInfo.unlimited;
}
```

**User impact:** Agent mode continues with free model instead of failing.

## BYOK and Extension Models

**Bring Your Own Key (BYOK) models:**
- Custom endpoints configured by users
- `multiplier: 0` (no quota consumption)
- Unlimited iterations subject only to iteration limit

**Extension-contributed models:**
- Provided by other VS Code extensions
- Bypass Copilot quota system
- Subject to extension's own rate limits

## Best Practices

### For Users

1. **Use free models for exploration:** Reserve premium models for final implementations
2. **Monitor quota usage:** Check remaining allowance in Chat panel
3. **Avoid unnecessary continuations:** Each continuation = new premium request
4. **Enable overages carefully:** Paid overages accrue costs beyond subscription

### For Developers

1. **Display quota status:** Show remaining requests in UI
2. **Warn before expensive operations:** Alert users before high-iteration tasks
3. **Optimize prompts:** Reduce iterations needed for common tasks
4. **Support free models:** Ensure agent mode works with multiplier=0 models

## Quota Calculation Examples

### Example 1: Daily Heavy Usage

User with:
- 1,000 premium requests/month
- Model multiplier: 1
- ~33 requests/day budget

**Usage pattern:**
- 10 ask mode questions/day: 10 requests
- 5 agent mode sessions (avg 50 iterations): 5 requests
- 2 continuations: 2 requests
- **Daily total: 17 requests**

**Monthly: 510 requests (well within quota)**

### Example 2: Aggressive Agent Use

User with:
- 100 premium requests/month
- Model multiplier: 2
- Effective: 50 turns/month

**Usage pattern:**
- 20 agent sessions with 2 continuations each
- 20 sessions × 3 turns × 2 multiplier = 120 requests
- **Quota exceeded after ~16 sessions**

**Recommendation:** Use free models or reduce continuations.

## Related Source Files

Key files for researching premium request consumption:

### Quota Structure and Tracking

- `src/platform/chat/chatQuotaServiceImpl.ts` - Quota tracking, header processing, `IChatQuota` interface
- `src/platform/chat/chatQuotaService.ts` - Quota service interface definitions

### Model Billing

- `src/platform/chat/chatEndpoint.ts` - `IChatModelInformation`, billing multiplier definition
- `src/extension/commands/chatModelsWidget.ts` - UI tooltip for multiplier display

### Telemetry and Turn Tracking

- `src/extension/conversation/chatParticipantTelemetry.ts` - Turn vs iteration tracking
- `src/platform/chat/toolCallingLoop.ts` - Iteration execution within turns

### Error Handling

- `src/platform/chat/commonTypes.ts` - Quota exhaustion error messages by tier
- `src/extension/conversation/chatParticipants.ts` - Auto-fallback to base model logic

## See Also

- [Agent Mode Limits](./agent-mode-limits.md)
- [Continuation Mechanism](./continuation-mechanism.md)
- [Quota and Rate Limiting](./quota-and-rate-limiting.md)
- [Tool Call Counting](./tool-call-counting.md)
