# Tool Call Counting Semantics

## Overview

Understanding how tool calls are counted is essential for managing agent mode iteration limits and quota consumption. This document clarifies the distinction between "iterations" and "tool calls."

## Core Definitions

### Iteration

**One iteration = One complete tool calling round:**

1. **Build prompt** with accumulated context and available tools
2. **Send request** to language model
3. **Receive response** (may contain 0, 1, or multiple tool calls)
4. **Collect tool calls** from streaming response
5. **Execute next iteration** with tool results

### Tool Call

**One tool call = Individual function invocation requested by the LLM:**
- Example: `read_file`, `grep_search`, `insert_edit_into_file`
- Multiple tool calls can occur within a single LLM response
- Executed conceptually in parallel (results all fed back to next iteration)

## Critical Distinction

```
Multiple parallel tool calls = 1 iteration
Sequential tool calls (across responses) = N iterations
```

## Counter Increment Logic

From `src/platform/chat/toolCallingLoop.ts`:

```typescript
public async run(outputStream: ChatResponseStream | undefined, token: CancellationToken | PauseController): Promise<IToolCallLoopResult> {
    let i = 0;  // Iteration counter
    let lastResult: IToolCallSingleResult | undefined;

    while (true) {
        if (lastResult && i++ >= this.options.toolCallLimit) {
            lastResult = this.hitToolCallLimit(outputStream, lastResult);
            break;
        }

        const result = await this.runOne(outputStream, i, token);
        // runOne() executes ONE iteration, may contain multiple tool calls

        this.toolCallRounds.push(result.round);
        if (!result.round.toolCalls.length || result.response.type !== ChatFetchResponseType.Success) {
            break;
        }
    }
}
```

**Key observations:**
- Counter `i` increments **once per `runOne()` call**
- `runOne()` represents one LLM request-response cycle
- `result.round.toolCalls` may contain multiple tool calls
- All tool calls in `result.round.toolCalls` count as **1 iteration**

## Tool Call Collection

### Streaming Aggregation

From `src/platform/chat/toolCallingLoop.ts`:

```typescript
finishedCb: async (text, index, delta) => {
    if (delta.copilotToolCalls) {
        toolCalls.push(...delta.copilotToolCalls.map((call): IToolCall => ({
            ...call,
            id: this.createInternalToolCallId(call.id),
            arguments: call.arguments === '' ? '{}' : call.arguments
        })));
    }
}
```

**Process:**
1. LLM streams response with `delta.copilotToolCalls`
2. All tool calls from deltas are pushed into `toolCalls[]` array
3. Complete array packaged into `ToolCallRound`
4. **Single iteration contains all collected tool calls**

### ToolCallRound Structure

```typescript
round: ToolCallRound.create({
    response: fetchResult.value,
    toolCalls,              // May contain 0, 1, or N tool calls
    toolInputRetry,
    statefulMarker,
    thinking: thinkingItem
})
```

## Counting Rules Table

| Scenario | Tool Calls | Iterations | Notes |
|----------|-----------|-----------|-------|
| LLM requests `read_file` | 1 | 1 | Single tool in response |
| LLM requests `read_file`, `grep_search`, `list_dir` | 3 | 1 | Parallel tools in same response |
| LLM requests `read_file` → next round `grep_search` | 2 | 2 | Sequential across responses |
| LLM requests 5 tools → next round 3 tools | 8 | 2 | Two iterations, total 8 tool calls |
| LLM responds with text only (no tools) | 0 | 1 then exit | Final iteration before loop ends |
| Tool validation fails (retry) | 1 | 1 | Same iteration, `toolInputRetry++` |
| Quota exceeded mid-execution | varies | varies | Loop terminates immediately |

## Example Scenarios

### Scenario 1: Sequential Investigation

```
User: "Find the authentication bug"

Iteration 1:
  LLM requests: [semantic_search("authentication bug")]
  i = 1

Iteration 2:
  LLM requests: [read_file("src/auth.ts")]
  i = 2

Iteration 3:
  LLM requests: [grep_search("password")]
  i = 3

Total: 3 tool calls, 3 iterations
```

### Scenario 2: Parallel Tool Execution

```
User: "Analyze the project structure"

Iteration 1:
  LLM requests: [
    read_project_structure(),
    grep_search("TODO"),
    search_workspace_symbols("class")
  ]
  i = 1

Iteration 2:
  LLM requests: [
    read_file("package.json"),
    read_file("tsconfig.json")
  ]
  i = 2

Total: 5 tool calls, 2 iterations
```

### Scenario 3: Complex Multi-Step Task

```
User: "Refactor the database module"

Iteration 1: [semantic_search("database module")] → i=1
Iteration 2: [read_file("db.ts"), read_file("models.ts"), read_file("queries.ts")] → i=2
Iteration 3: [grep_search("connection")] → i=3
Iteration 4: [insert_edit_into_file("db.ts")] → i=4
Iteration 5: [get_errors("db.ts")] → i=5
Iteration 6: [read_file("db.ts")] → i=6
Iteration 7: [insert_edit_into_file("db.ts")] → i=7
... continues up to iteration 200 ...

Total: ~250 tool calls, 200 iterations (limit reached)
```

## Per-Iteration Tool Call Limit

**No limit exists** on how many tools the LLM can request in a single response.

Evidence:
```typescript
// No check like this exists:
if (toolCalls.length > MAX_TOOLS_PER_ITERATION) {
    throw new Error('Too many tools requested');
}
```

The LLM can theoretically request unlimited tools per iteration, constrained only by:
- Model's own capabilities
- Token budget for tool schemas
- Practical limits of response parsing

## Tool Input Validation Retries

From `src/platform/chat/toolCallingLoop.ts`:

```typescript
const toolInputRetry = isToolInputFailure ?
    (this.toolCallRounds.at(-1)?.toolInputRetry || 0) + 1 : 0;
```

**When tool input validation fails:**
- Same iteration continues
- `toolInputRetry` counter increments (within current round)
- Does **not** increment iteration counter `i`
- Retry is part of the same `runOne()` execution

**Example:**
```
Iteration 5:
  LLM requests: [read_file("invalid/path")]
  Tool validation fails
  toolInputRetry = 1
  LLM corrects: [read_file("src/valid.ts")]
  Tool succeeds
  i = 5 (still same iteration)
```

## All Tools Counted Equally

**No tool-specific exemptions or weights exist.**

| Tool Type | Iteration Count Impact | Special Treatment |
|-----------|----------------------|-------------------|
| Search tools | +0 or +1 (if new iteration) | None |
| Edit tools | +0 or +1 (if new iteration) | None |
| File tools | +0 or +1 (if new iteration) | None |
| Agent tools (runSubagent) | +0 or +1 (if new iteration) | Spawns isolated loop |
| Terminal tools | +0 or +1 (if new iteration) | None |
| Test tools | +0 or +1 (if new iteration) | None |

**Every tool consumes from the same iteration budget.**

## Subagent Tool Calls

**runSubagent creates independent counting context:**

```
Parent Iteration 10: [semantic_search, runSubagent, read_file]
  ├─ parent i = 10
  └─ runSubagent executes:
      ├─ Subagent Iteration 1: [grep_search] → subagent i=1
      ├─ Subagent Iteration 2: [read_file] → subagent i=2
      └─ Returns to parent
Parent Iteration 11: [continues with subagent result]
  └─ parent i = 11
```

**Subagent iterations do NOT count toward parent's iteration limit.**

From `src/vs/workbench/contrib/chat/browser/tools/runSubagentTool.ts`:

```typescript
const agentRequest: IChatAgentRequest = {
    sessionResource: invocation.context.sessionResource,  // Same session
    requestId: `subagent-${Date.now()}`,                  // New request ID
    isSubagent: true,                                     // Marker flag
    // ... creates new ToolCallingLoop with fresh i=0
};
```

## Telemetry Tracking

From `src/extension/conversation/chatParticipantTelemetry.ts`:

**Metrics tracked:**
- `turn`: Conversation turn number
- `toolCallRound`: Current iteration within turn
- `numToolCallIterations`: Total iterations in this turn
- `toolCallRounds`: Array of all rounds with tool call details

**Example telemetry:**
```json
{
    "turn": 5,
    "toolCallRound": 23,
    "numToolCallIterations": 23,
    "toolCallRounds": [
        { "toolCalls": [{"name": "read_file"}] },
        { "toolCalls": [{"name": "grep_search"}, {"name": "semantic_search"}] },
        // ... 21 more rounds
    ]
}
```

## Performance Implications

### High Parallel Tool Count

**Impact on single iteration:**
- More tool execution time
- Higher memory usage for tool results
- Larger prompt for next iteration (more context)

**Does NOT impact:**
- Iteration count (still just 1)
- Premium request count (still just part of current turn)

### Sequential Tool Pattern

**Impact on iteration count:**
- Each sequential tool set = +1 iteration
- Approaches limit faster

**Example:**
- 200 iterations with 1 tool each = 200 tool calls total
- 100 iterations with 2 tools each = 200 tool calls total
- **Same tool calls, different iteration consumption**

## Best Practices

### For LLM Prompt Engineering

**Encourage parallel tool use:**
- "Use multiple tools in parallel when possible"
- Reduces iteration consumption
- Faster overall execution

**Example prompt guidance:**
```
When investigating code, request semantic_search, grep_search, and
read_project_structure together instead of sequentially.
```

### For Tool Implementation

**Make tools parallelizable:**
- Avoid side effects that require sequential execution
- Design tools to operate independently
- Return comprehensive results to reduce follow-up iterations

### For Quota Management

**Monitor iteration patterns:**
- Tools per iteration ratio
- Sequential vs parallel tool usage
- Identify inefficient patterns

## Iteration Counter Reset

**Counter resets only when:**
1. New ToolCallingLoop instance created
2. New user request (including continuations)
3. Subagent invocation (isolated counter)

**Counter does NOT reset:**
- During same request execution
- After tool validation failures
- On quota errors (loop terminates instead)

## Related Source Files

Key files for researching tool call counting semantics:

### Core Loop Implementation

- `src/platform/chat/toolCallingLoop.ts` - Counter increment logic, `run()`, `runOne()`, tool call collection

### Tool Call Aggregation

- `src/platform/chat/toolCallingLoop.ts` - Streaming aggregation (`finishedCb`), `ToolCallRound.create()`

### Subagent Independence

- `src/vs/workbench/contrib/chat/browser/tools/runSubagentTool.ts` (VS Code repository) - Subagent invocation, isolated loop creation

### Telemetry

- `src/extension/conversation/chatParticipantTelemetry.ts` - Turn, iteration, and tool call round tracking

### Tool Validation

- `src/platform/chat/toolCallingLoop.ts` - `toolInputRetry` handling for validation failures

## See Also

- [Agent Mode Limits](./agent-mode-limits.md)
- [Premium Request Consumption](./premium-request-consumption.md)
- [Continuation Mechanism](./continuation-mechanism.md)
