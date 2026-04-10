# Continuation Mechanism

## Overview

When agent mode reaches the iteration limit, users can opt to **continue** through a confirmation dialog. This creates a new request with an increased iteration limit while preserving conversation context.

## Flow Diagram

```
User Request
    ↓
ToolCallingLoop (i=0, limit=200)
    ↓
Execute iterations (i++ each round)
    ↓
i >= 200 (limit reached)
    ↓
hitToolCallLimit()
    ↓
Show Confirmation Dialog
    ├─ [Continue] → New Request (limit=300)
    └─ [Pause] → Friendly exit message
```

## Step-by-Step Process

### 1. Limit Detection

From `src/platform/chat/toolCallingLoop.ts`:

```typescript
while (true) {
    if (lastResult && i++ >= this.options.toolCallLimit) {
        lastResult = this.hitToolCallLimit(outputStream, lastResult);
        break;
    }
    // ... execute iteration
}
```

When `i >= toolCallLimit`, the loop invokes `hitToolCallLimit()` and breaks.

### 2. Confirmation Creation

```typescript
private hitToolCallLimit(stream: ChatResponseStream | undefined, lastResult: IToolCallSingleResult) {
    if (stream && this.options.onHitToolCallLimit === ToolCallLimitBehavior.Confirm) {
        stream.confirmation(
            l10n.t('Continue to iterate?'),
            messageString,
            { copilotRequestedRoundLimit: Math.round(this.options.toolCallLimit * 3 / 2) } satisfies IToolCallIterationIncrease,
            [l10n.t('Continue'), cancelText()]
        );
    }

    lastResult.chatResult = {
        ...lastResult.chatResult,
        metadata: { ...lastResult.chatResult?.metadata, maxToolCallsExceeded: true }
    };
    return lastResult;
}
```

**New limit calculation:** `Math.round(toolCallLimit * 3 / 2)`

**Result:** 200 → 300, 300 → 450, 450 → 675, etc.

### 3. User Action

#### User Clicks "Continue"

VS Code creates **new ChatRequest** with:
- `prompt`: "Continue"
- `acceptedConfirmationData`: `[{ copilotRequestedRoundLimit: 300 }]`

#### User Clicks "Pause"

Request marked as `isToolCallLimitCancellation`, shows friendly message:
> "Let me know if there's anything else I can help with!"

### 4. Turn Creation

From `src/extension/conversation/conversation.ts`:

```typescript
static fromRequest(id: string | undefined, request: ChatRequest) {
    return new Turn(
        id,
        { message: request.prompt, type: 'user' },
        new ChatVariablesCollection(request.references),
        request.toolReferences.map(InternalToolReference.from),
        request.editedFileEvents,
        request.acceptedConfirmationData,  // ← Contains copilotRequestedRoundLimit
        isToolCallLimitAcceptance(request) || isContinueOnError(request),  // ← isContinuation flag
    );
}
```

**isContinuation flag set:** Marks this turn as a continuation of previous agent work.

### 5. Limit Extraction

From `src/extension/conversation/specialRequestTypes.ts`:

```typescript
export interface IToolCallIterationIncrease {
    copilotRequestedRoundLimit: number;
}

const isToolCallIterationIncrease = (c: unknown): c is IToolCallIterationIncrease =>
    !!(c && typeof (c as IToolCallIterationIncrease).copilotRequestedRoundLimit === 'number');

export const getRequestedToolCallIterationLimit = (request: ChatRequest) =>
    request.acceptedConfirmationData?.find(isToolCallIterationIncrease)?.copilotRequestedRoundLimit;
```

Extracts the increased limit (e.g., 300) from `acceptedConfirmationData`.

### 6. New Loop Creation

From `src/extension/intents/agentIntent.ts`:

```typescript
protected override getIntentHandlerOptions(request: vscode.ChatRequest): IDefaultIntentRequestHandlerOptions {
    return {
        maxToolCallIterations: getRequestedToolCallIterationLimit(request) ??
            this.instantiationService.invokeFunction(getAgentMaxRequests),
        temperature: this.configurationService.getConfig(ConfigKey.Advanced.AgentTemperature) ?? 0,
        overrideRequestLocation: ChatLocation.Agent,
        hideRateLimitTimeEstimate: true
    };
}
```

**Logic:**
- If continuation: `getRequestedToolCallIterationLimit(request)` returns 300
- If initial request: Falls back to `getAgentMaxRequests()` (returns 200)

The extracted limit becomes `maxToolCallIterations` for the new ToolCallingLoop instance.

## Continuation Semantics

### New Request, Not Resume

**Critical distinction:** Continuation is architecturally a **brand new request**, not a paused session resuming.

| Aspect | New Request | Resumed Session |
|--------|-------------|-----------------|
| Request Object | ✓ New ChatRequest created | ✗ Would reuse existing |
| Loop Instance | ✓ New ToolCallingLoop | ✗ Would be same instance |
| Iteration Counter | ✓ Resets to `i=0` | ✗ Would continue from `i=200` |
| Limit Value | ✓ Increased to 300 | ✗ Would stay 200 |

### What Is Preserved

**Conversation Context:**
- Same `sessionId`
- All previous turns in `conversation.turns[]`
- Accumulated tool call results in conversation history
- Tool call rounds available for prompt context

**What Is Reset:**
- Loop iteration counter (`i` starts at 0)
- ToolCallingLoop instance (new object)
- `toolCallResults` map (new empty map)
- `toolCallRounds` array (new empty array, but history preserved in conversation)

From `src/platform/chat/toolCallingLoop.ts`:

```typescript
export abstract class ToolCallingLoop<TOptions extends IToolCallingLoopOptions = IToolCallingLoopOptions> extends Disposable {
    private toolCallResults: Record<string, LanguageModelToolResult2> = Object.create(null);  // Per-instance
    private toolCallRounds: IToolCallRound[] = [];  // Per-instance

    constructor(protected readonly options: TOptions) {
        super();
    }
}
```

Each new request creates a new instance with fresh state.

## Limit Progression

### Multiplicative Growth (1.5× per continuation)

| Continuation # | Calculation | New Limit | Cumulative Iterations |
|----------------|-------------|-----------|----------------------|
| Initial | Config default | 200 | 200 |
| 1st Continue | `200 × 1.5` | 300 | 500 |
| 2nd Continue | `300 × 1.5` | 450 | 950 |
| 3rd Continue | `450 × 1.5` | 675 | 1,625 |
| 4th Continue | `675 × 1.5` | 1,012 | 2,637 |
| 5th Continue | `1,012 × 1.5` | 1,518 | 4,155 |

**Formula:** `newLimit = Math.round(previousLimit × 1.5)`

### No Continuation Cap

**No coded limit exists** on how many times a user can continue.

```typescript
// This check does NOT exist anywhere in the codebase:
if (continuationCount > MAX_CONTINUATIONS) {
    throw new Error('Too many continuations');
}
```

Each continuation simply recalculates: `limit × 1.5`.

## Practical Constraints

While technically unlimited, continuations are constrained by:

### 1. Premium Quota Exhaustion

Each continuation = new turn = new premium request consumed (see [Premium Request Consumption](./premium-request-consumption.md)).

**Example:**
- User with 100 premium requests/month
- Model multiplier: 2
- Each continuation: 2 premium requests

After 50 continuation sessions: quota exhausted.

### 2. Agent Mode Rate Limit

Separate temporal limit on agent mode requests:

Error code: `agent_mode_limit_exceeded`
Message: "Sorry, you have exceeded the agent mode rate limit. Please switch to ask mode and try again later."

### 3. Model Context Window

Conversation history grows with each iteration. Eventually exceeds model's `maxInputTokens`.

### 4. User Patience

Unlikely a user manually clicks "Continue" more than 5-10 times.

### 5. Network/Timeout

Long-running sessions may encounter timeouts or connection issues.

## Cancellation Handling

From `src/extension/conversation/defaultIntentRequestHandler.ts`:

```typescript
async getResult(): Promise<ChatResult> {
    if (isToolCallLimitCancellation(this.request)) {
        this.stream.markdown(l10n.t("Let me know if there's anything else I can help with!"));
        return {};
    }
    // ... normal flow
}
```

**When user clicks "Pause":**
- Request marked with cancellation flag
- Friendly message displayed
- Empty result returned (no error)

## Telemetry Tracking

Continuation requests are tracked separately:

**Request flags:**
- `isSubagent`: false (it's a user-initiated continuation)
- `isContinuation`: true (marks it as continuation turn)

**Debug identifier:**
- `userInitiatedRequest`: false for continuations (iteration 0 check includes `!isContinuation`)

From `src/platform/chat/toolCallingLoop.ts`:

```typescript
userInitiatedRequest: iterationNumber === 0 && !isContinuation && !this.options.request.isSubagent
```

## Best Practices

### For Users

1. **Monitor quota:** Each continuation consumes additional premium requests
2. **Refine prompts:** Consider sending a new message instead of continuing if approach isn't working
3. **Check progress:** Review tool call results before clicking Continue
4. **Use sparingly:** Limit continuation to genuinely complex tasks

### For Developers

1. **Preserve context:** Ensure conversation history is accessible in continued turns
2. **Clear messaging:** Inform users that continuation consumes additional quota
3. **Telemetry:** Track continuation patterns to optimize default limits
4. **Fallback gracefully:** Handle quota exhaustion mid-continuation

## Configuration Link

The confirmation dialog includes a link to settings:

```typescript
[Configure max requests](command:workbench.action.openSettings?["chat.agent.maxRequests"])
```

Clicking this opens VS Code settings to `chat.agent.maxRequests`, allowing users to adjust the initial limit.

## Related Source Files

Key files for researching the continuation mechanism:

### Confirmation and Limit Calculation

- `src/platform/chat/toolCallingLoop.ts` - `hitToolCallLimit()`, confirmation dialog, limit multiplication
- `src/extension/conversation/specialRequestTypes.ts` - `IToolCallIterationIncrease`, `getRequestedToolCallIterationLimit()`

### Request and Turn Creation

- `src/extension/conversation/conversation.ts` - `Turn.fromRequest()`, `isContinuation` flag
- `src/extension/intents/agentIntent.ts` - `getIntentHandlerOptions()`, limit extraction
- `src/extension/conversation/defaultIntentRequestHandler.ts` - Cancellation handling (`isToolCallLimitCancellation`)

### Configuration

- `src/extension/conversation/common/agentConfig.ts` - `getAgentMaxRequests()` fallback for initial requests

## See Also

- [Agent Mode Limits](./agent-mode-limits.md)
- [Premium Request Consumption](./premium-request-consumption.md)
- [Quota and Rate Limiting](./quota-and-rate-limiting.md)
