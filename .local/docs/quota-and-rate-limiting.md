# Quota and Rate Limiting

## Overview

GitHub Copilot employs multiple independent quota and rate limiting mechanisms to ensure fair usage and prevent abuse. Understanding the distinction between these systems is critical for managing agent mode usage.

## Quota Types

### 1. Premium Interactions Quota

**Scope:** Premium model usage across all Copilot features
**Tracking:** Server-side via response headers
**Structure:**

```typescript
interface IChatQuota {
    quota: number;           // Total monthly entitlement
    used: number;            // Premium interactions consumed
    unlimited: boolean;      // Unlimited quota flag
    overageUsed: number;     // Paid requests beyond base quota
    overageEnabled: boolean; // Can accrue additional paid requests
    resetDate: Date;         // When quota resets (monthly)
}
```

**Headers:**
- Free tier: `x-quota-snapshot-chat`
- Premium tier: `x-quota-snapshot-premium_models` or `x-quota-snapshot-premium_interactions`

**URL-encoded format example:**
```
ent=1000&rem=0.75&ovPerm=true&ovCnt=0
```

Parsed as:
- `ent`: Entitlement (total quota)
- `rem`: Percent remaining
- `ovPerm`: Overage permitted
- `ovCnt`: Overage count

### 2. Chat Messages Quota (Free Tier)

**Scope:** Chat requests for Copilot Free users
**Tracking:** `x-quota-snapshot-chat` header
**Reset:** Monthly
**Typical limits:** 50-100 messages/month

### 3. Completions Quota

**Scope:** Inline code completion requests
**Tracking:** Separate from chat/agent mode
**Purpose:** Limits autocomplete suggestions

## Rate Limiting Types

### 1. Global User TPS (Transactions Per Second)

**Pattern:** `global-user(-[^-]+)?-tps-\d{4}-\d{2}-\d{2}`
**Scope:** Per-user across all Copilot features
**Window:** Typically 1 second or 1 minute
**Headers:**
- `x-ratelimit-remaining`: Requests remaining in window
- `x-ratelimit-reset`: Timestamp when window resets

### 2. Agent Mode Specific Rate Limit

**Error code:** `agent_mode_limit_exceeded`
**Scope:** Agent mode requests only
**Purpose:** Prevent abuse of computationally expensive agent iterations

From `src/platform/chat/commonTypes.ts`:

```typescript
if (fetchResult.capiError?.code === 'agent_mode_limit_exceeded') {
    return l10n.t('Sorry, you have exceeded the agent mode rate limit. Please switch to ask mode and try again later.');
}
```

**Characteristics:**
- Distinct from general quota limits
- Temporal (resets after time window)
- Does not affect ask/edit modes
- Likely hourly or daily window

### 3. Model-Specific Rate Limits

**Scope:** Per-model endpoint
**Purpose:** Protect individual model backends
**Behavior:** Auto-fallback to base model when hit

## Quota vs Rate Limit Distinction

| Aspect | Quota | Rate Limit |
|--------|-------|------------|
| **Measure** | Total consumption (cumulative) | Requests per time window (temporal) |
| **Reset** | Monthly (fixed date) | Rolling or fixed time windows |
| **Scope** | Account-wide | User or endpoint-specific |
| **Enforcement** | Hard cap until reset | Temporary throttle |
| **User action** | Enable overages or wait | Wait for window reset |
| **Example** | 1000 requests/month | 10 requests/minute |

## Relationship: Iteration Limit vs Quota

### Independent Systems

The iteration limit (200) and quota system operate independently:

From `src/platform/chat/toolCallingLoop.ts`:

```typescript
if (!result.round.toolCalls.length || result.response.type !== ChatFetchResponseType.Success) {
    break;  // Quota/rate limit errors exit here
}
```

**When quota is exceeded:**
- `ChatFetchResponseType.QuotaExceeded` returned
- Loop terminates immediately
- Iteration count irrelevant

### Iteration Limit Does NOT Guarantee Quota

**Example scenario:**

User with:
- Monthly quota: 100 premium interactions
- Model multiplier: 2
- Effective turns: 50

**Agent mode session:**
- Default iteration limit: 200
- **Quota exhausted at ~50 turns**
- Iteration limit never reached

**Calculation:**
```
Quota_Exhaustion = quota / (model_multiplier × avg_turns_per_task)

100 / (2 × 1) = 50 sessions
```

Even if each session uses only 10 iterations (well under 200 limit), quota exhausted after 50 sessions.

### Quota Can Terminate Before Limit

From `src/extension/intents/agentIntent.ts`:

```typescript
try {
    const result = await this.runOne(outputStream, i, token);
    // ...
} catch (e) {
    if (isCancellationError(e)) {
        break;
    }
    throw e;  // Quota errors propagate here
}
```

**Quota exhaustion triggers immediate exit**, regardless of iteration count.

## Quota Exhaustion Handling

### Auto-Fallback to Base Model

From `src/extension/conversation/chatParticipants.ts`:

```typescript
get quotaExhausted(): boolean {
    return this._quotaInfo.used >= this._quotaInfo.quota
        && !this._quotaInfo.overageEnabled
        && !this._quotaInfo.unlimited;
}
```

**When `quotaExhausted === true`:**
1. Switch to base endpoint (free model)
2. Show warning message
3. Continue execution with free model

### Error Messages by Plan

**Free Tier:**
> "You've reached your monthly chat messages quota. Upgrade to Copilot Pro for unlimited messages and access to premium models."

**Individual Tier:**
> "You've exhausted your premium model quota. Please enable additional paid premium requests, upgrade to Copilot Pro+, or wait for your allowance to renew on {date}."

**Pro Tier:**
> "You've exhausted your premium model quota. Please enable additional paid premium requests or wait for your allowance to renew on {date}."

**Enterprise Tier:**
> "You've exhausted your premium model quota. Please contact your organization administrator or wait for your allowance to renew on {date}."

**Overage Limit Reached:**
> "You cannot accrue additional premium requests at this time. Please contact GitHub Support to continue using Copilot."

## Rate Limit Error Handling

From `src/platform/chat/commonTypes.ts`:

```typescript
function getRateLimitMessage(fetchResult: ChatFetchError, ...): string {
    if (fetchResult.capiError?.code === 'agent_mode_limit_exceeded') {
        return l10n.t('Sorry, you have exceeded the agent mode rate limit. Please switch to ask mode and try again later.');
    }

    if (fetchResult.capiError?.statusCode === 429) {
        const resetTimestamp = Number.parseInt(fetchResult.capiError?.headers?.['x-ratelimit-reset'] || '0', 10);
        const resetDate = resetTimestamp ? new Date(resetTimestamp * 1000) : undefined;

        return l10n.t('Your request was rate-limited. Please wait until {0}.',
            resetDate ? resetDate.toLocaleTimeString() : 'shortly');
    }

    return l10n.t('Your request was rate-limited. Please try again later.');
}
```

**HTTP 429 handling:**
- Extract `x-ratelimit-reset` header
- Calculate time until window resets
- Display user-friendly countdown

## Quota Header Processing

From `src/platform/chat/chatQuotaServiceImpl.ts`:

```typescript
processQuotaHeaders(headers: IHeaders): void {
    const quotaHeader = this._authService.copilotToken?.isFreeUser
        ? headers.get('x-quota-snapshot-chat')
        : headers.get('x-quota-snapshot-premium_models') ||
          headers.get('x-quota-snapshot-premium_interactions');

    if (!quotaHeader) return;

    const params = new URLSearchParams(quotaHeader);
    const entitlement = parseInt(params.get('ent') || '0', 10);
    const percentRemaining = parseFloat(params.get('rem') || '0.0');
    const overageEnabled = params.get('ovPerm') === 'true';
    const overageCount = parseInt(params.get('ovCnt') || '0', 10);

    this._quotaInfo = {
        quota: entitlement,
        used: Math.round(entitlement * (1 - percentRemaining)),
        unlimited: entitlement === 0 && percentRemaining === 1.0,
        overageUsed: overageCount,
        overageEnabled,
        resetDate: new Date(/* parsed from header */)
    };
}
```

**Processed after every language model request.**

## Protection Mechanisms

### 1. Iteration Limit Protects Against

- Runaway loops
- Excessive latency per turn
- UX degradation from long waits
- Compute waste on ineffective approaches

### 2. Quota System Protects Against

- Billing overage
- Unfair resource consumption
- Abuse of premium models
- Service provider costs

### 3. Rate Limits Protect Against

- Burst traffic spikes
- Automated abuse
- Backend overload
- DoS attacks

**All three systems are complementary, not redundant.**

## Quota Monitoring Best Practices

### For Users

1. **Check quota status regularly:**
   - View in Chat panel status bar
   - Monitor after agent mode sessions
   - Track before complex tasks

2. **Use free models for exploration:**
   - Reserve premium for final implementations
   - Switch to free models when quota low

3. **Manage continuations:**
   - Each continuation = new turn = quota consumption
   - Consider refining prompt instead of continuing

4. **Enable overages cautiously:**
   - Understand cost implications
   - Set budgets if available

### For Developers

1. **Display quota prominently:**
   - Show remaining requests in UI
   - Warn when quota low (<10%)

2. **Implement quota-aware features:**
   - Suggest free models when quota exhausted
   - Optimize prompts to reduce turn count

3. **Handle quota errors gracefully:**
   - Auto-fallback to free models
   - Clear user messaging
   - Don't fail silently

4. **Track quota consumption patterns:**
   - Telemetry on quota exhaustion rates
   - Identify high-consumption features
   - Optimize iteration efficiency

## Quota Calculation Examples

### Example 1: Monthly Budget Planning

**User profile:**
- Plan: Individual (100 premium requests/month)
- Model preference: GPT-4 (multiplier=2)
- Effective budget: 50 turns/month

**Usage pattern:**
- 2 agent sessions/day × 30 days = 60 sessions
- Each session averages 1.2 turns (some continuations)
- 60 × 1.2 = 72 turns
- **72 turns × 2 multiplier = 144 requests**

**Result:** Quota exceeded mid-month. Need to:
- Reduce continuations
- Switch to free models for some tasks
- Upgrade to Pro (higher quota)

### Example 2: Within Budget

**User profile:**
- Plan: Pro (1000 premium requests/month)
- Model preference: Claude (multiplier=1)
- Effective budget: 1000 turns/month

**Usage pattern:**
- 10 agent sessions/day × 22 workdays = 220 sessions
- Each session averages 2 turns (some continuations)
- 220 × 2 = 440 turns
- **440 turns × 1 multiplier = 440 requests**

**Result:** Well within quota (56% remaining).

## Special Cases

### BYOK (Bring Your Own Key)

**Custom endpoints:**
- `multiplier: 0` (no quota consumption)
- Bypass Copilot quota system
- Subject to provider's own limits

### Extension-Contributed Models

**Provided by other extensions:**
- No Copilot quota consumption
- Extension manages own rate limiting
- May have separate pricing

### Unlimited Quota

**Enterprise tiers may have:**
- `unlimited: true` flag
- No quota tracking
- Still subject to rate limits

## Related Source Files

Key files for researching quota and rate limiting:

### Quota Implementation

- `src/platform/chat/chatQuotaServiceImpl.ts` - Quota header processing, `processQuotaHeaders()`, quota state management
- `src/platform/chat/chatQuotaService.ts` - Quota service interface

### Error Messages and Handling

- `src/platform/chat/commonTypes.ts` - `getRateLimitMessage()`, error code handling, tier-specific messages
- `src/extension/conversation/chatParticipants.ts` - `quotaExhausted` getter, auto-fallback logic

### Loop Termination

- `src/platform/chat/toolCallingLoop.ts` - Quota/rate limit error handling in loop, immediate exit on `QuotaExceeded`
- `src/extension/intents/agentIntent.ts` - Error propagation in agent intent

### Agent Mode Rate Limiting

- `src/platform/chat/commonTypes.ts` - `agent_mode_limit_exceeded` error code handling

## See Also

- [Agent Mode Limits](./agent-mode-limits.md)
- [Premium Request Consumption](./premium-request-consumption.md)
- [Continuation Mechanism](./continuation-mechanism.md)
- [Tool Call Counting](./tool-call-counting.md)
