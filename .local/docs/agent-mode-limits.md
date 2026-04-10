# Agent Mode Iteration Limits

## Overview

Agent mode in vscode-copilot-chat implements a **soft iteration limit** with user confirmation to prevent runaway tool calling loops while allowing flexible continuation for complex tasks.

## Configuration

### Primary Setting

**Configuration Key:** `chat.agent.maxRequests`
**Default Value:** 200 iterations
**Type:** `number`
**Scope:** Per-turn tool calling iterations

### Resolution Logic

From `src/extension/conversation/common/agentConfig.ts`:

```typescript
export function getAgentMaxRequests(accessor: ServicesAccessor): number {
    const configuredValue = configurationService.getNonExtensionConfig<number>('chat.agent.maxRequests') ?? 200;

    const experimentLimit = experimentationService.getTreatmentVariable<number>('chatAgentMaxRequestsLimit');
    if (typeof experimentLimit === 'number' && experimentLimit > 0) {
        return Math.min(configuredValue, experimentLimit);
    }

    return configuredValue;
}
```

**Priority:**
1. User setting `chat.agent.maxRequests`
2. Capped by experiment variable `chatAgentMaxRequestsLimit` (if active)
3. Fallback to 200 for simulation tests

## What Constitutes an "Iteration"

**One iteration = One complete tool calling round:**
1. Build prompt with accumulated context
2. Send request to language model
3. Receive response (may contain multiple tool calls)
4. Execute all tool calls from response
5. Collect results

**Critical distinction:**
- Multiple parallel tool calls in one response = **1 iteration**
- Sequential tool calls across responses = **N iterations**

### Example Scenario

```
Iteration 1: LLM responds with [read_file, grep_search, list_dir] → i=1
Iteration 2: LLM responds with [insert_edit_into_file] → i=2
Iteration 3: LLM responds with [get_errors, read_file] → i=3

Total: 6 tool calls consumed 3 iterations
```

## Enforcement Mechanism

### Loop Check

From `src/platform/chat/toolCallingLoop.ts`:

```typescript
public async run(outputStream: ChatResponseStream | undefined, token: CancellationToken | PauseController): Promise<IToolCallLoopResult> {
    let i = 0;
    let lastResult: IToolCallSingleResult | undefined;

    while (true) {
        if (lastResult && i++ >= this.options.toolCallLimit) {
            lastResult = this.hitToolCallLimit(outputStream, lastResult);
            break;
        }

        const result = await this.runOne(outputStream, i, token);
        this.toolCallRounds.push(result.round);

        if (!result.round.toolCalls.length || result.response.type !== ChatFetchResponseType.Success) {
            break;
        }
    }
}
```

**Counter behavior:**
- Local variable `i` starts at 0
- Increments after each `runOne()` completion
- Post-increment semantics: `i++` increments after comparison
- First iteration (i=0) always executes

## Limit Behaviors

### ToolCallLimitBehavior.Confirm (Default for Agent Mode)

When limit is reached, displays confirmation dialog to user:

**Title:** "Continue to iterate?"

**Message:** "Copilot has been working on this problem for a while. It can continue to iterate, or you can send a new message to refine your prompt. [Configure max requests](settings link)."

**Actions:**
- **Continue** → Increases limit by 50% and creates new request
- **Pause** → Stops execution gracefully

### ToolCallLimitBehavior.Stop

Immediately terminates when limit is reached. Sets `maxToolCallsExceeded: true` in metadata.

**Used when:** `confirmOnMaxToolIterations: false` in intent options

## Hard Limit vs Soft Limit

### Classification: **Soft Limit**

The iteration limit is **not a hard cap**. Users can extend it indefinitely through confirmation dialogs.

**Evidence:**
- No maximum continuation count check exists in codebase
- Each continuation recalculates limit as `previous × 1.5`
- No coded ceiling on how many times user can click "Continue"

**Practical constraints:**
- Premium quota exhaustion
- Agent mode rate limits
- Model context window size
- User patience
- Network/timeout issues

## Tool Type Treatment

**All tools are counted equally:**

| Tool Type | Counted in Iteration? | Special Exemption? |
|-----------|----------------------|-------------------|
| Search tools (semantic_search, grep_search) | ✓ Yes | ✗ No |
| Edit tools (insert_edit_into_file, apply_patch) | ✓ Yes | ✗ No |
| File tools (read_file, create_file) | ✓ Yes | ✗ No |
| Agent tools (runSubagent, manage_todo_list) | ✓ Yes | ✗ No |
| Terminal tools (run_in_terminal) | ✓ Yes | ✗ No |

**No tool-specific limits exist.** The iteration counter treats all tool invocations uniformly.

## Subagent Independence

**runSubagent calls create isolated execution contexts:**

| Aspect | Parent Agent | Subagent |
|--------|--------------|----------|
| Session ID | `session-abc` | `session-abc` (same) |
| Request ID | `parent-request` | `subagent-12345` (new) |
| Iteration Counter | Parent's `i` variable | **New `i` variable** |
| toolCallLimit | 200 | 200 (fresh budget) |
| Conversation History | Full parent history | Empty `[]` |

**Result:** Subagent gets independent 200-iteration budget, does not consume parent's iterations.

## Intent-Specific Limits

| Intent | File | Max Iterations Source |
|--------|------|----------------------|
| `Intent.Agent` | agentIntent.ts | `getAgentMaxRequests()` |
| `Intent.AskAgent` | askAgentIntent.ts | `getAgentMaxRequests()` |
| `Intent.Edit2` | editCodeIntent2.ts | `getAgentMaxRequests()` |
| `Intent.NotebookEditor` | notebookEditorIntent.ts | `getAgentMaxRequests()` |
| Default | defaultIntentRequestHandler.ts | 15 (hardcoded) |

**All agent-related intents share the same configuration source.**

## Metadata Tracking

When limit is reached, result metadata is updated:

```typescript
{
    ...lastResult.chatResult,
    metadata: {
        ...lastResult.chatResult?.metadata,
        maxToolCallsExceeded: true
    }
}
```

This flag persists regardless of whether user continues or pauses.

## Related Configuration

| Setting | Path | Default | Purpose |
|---------|------|---------|---------|
| `maxRequests` | `chat.agent.maxRequests` | 200 | Iteration limit |
| `temperature` | `github.copilot.chat.advanced.agentTemperature` | 0 | Response determinism |
| `summarizeThreshold` | `github.copilot.chat.advanced.summarizeAgentConversationHistoryThreshold` | `modelMaxPromptTokens` | Token budget trigger |
| `confirmOnMaxToolIterations` | (intent-specific) | `true` | Enable confirmation dialog |

## Telemetry

**Location identifier:** `panel/editAgent` or `editingSessionAgent` in Chat Debug panel

**Tracked separately:**
- `turn` / `externalTurnNumber`: Conversation turn count
- `toolCallRound`: Iteration number within current turn
- `numToolCallIterations`: Total iterations in turn
- Tool call rounds stored in `IResultMetadata.toolCallRounds[]`

## Related Source Files

Key files for researching agent mode iteration limits:

### Configuration and Limits

- `src/extension/conversation/common/agentConfig.ts` - Limit resolution logic (`getAgentMaxRequests()`)
- `src/extension/intents/agentIntent.ts` - Agent intent implementation and options
- `src/extension/intents/askAgentIntent.ts` - Ask agent intent with limit configuration
- `src/extension/intents/editCodeIntent2.ts` - Edit intent with limit configuration
- `src/extension/intents/notebookEditorIntent.ts` - Notebook editor intent with limits
- `src/extension/conversation/defaultIntentRequestHandler.ts` - Default handler with fallback limit (15)

### Loop Implementation

- `src/platform/chat/toolCallingLoop.ts` - Core loop implementation, counter logic, limit enforcement
- `src/extension/conversation/conversation.ts` - Conversation and turn management

### Location and Telemetry

- `src/extension/conversation/conversation.ts` - ChatLocation enum definition
- `src/extension/conversation/chatParticipantTelemetry.ts` - Telemetry tracking

## See Also

- [Continuation Mechanism](./continuation-mechanism.md)
- [Premium Request Consumption](./premium-request-consumption.md)
- [Tool Call Counting](./tool-call-counting.md)
