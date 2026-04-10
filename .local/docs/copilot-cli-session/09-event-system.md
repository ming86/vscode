# Event System

## Overview

The session event system is the bridge between the SDK's internal conversation loop and the VS Code chat UI. Events flow from the SDK `Session` object via `.on()` listeners to the extension's response stream. This document covers event types, processing logic, and history reconstruction.

**Sources:**
- `src/extension/chatSessions/copilotcli/common/copilotCLITools.ts` — Event processing, history reconstruction, tool types
- `src/extension/chatSessions/copilotcli/node/copilotcliSession.ts` — Event listener registration

## SDK Event Interface

```typescript
// From @github/copilot/sdk
interface SessionEvent {
    type: string;
    data: Record<string, unknown>;
    id: string;                      // UUID
    timestamp: string;               // ISO 8601
    parentId: string | null;         // Chain link
}

// Event listener registration
session.on(eventType: string, handler: (event: SessionEvent) => void): () => void;
session.on('*', handler): () => void;  // Wildcard — all events
```

## Event Types Reference

### Lifecycle Events

| Event Type | Data Fields | UI Effect |
|-----------|------------|-----------|
| `session.start` | `sessionId, version, producer, copilotVersion, startTime, context` | None — initialization |
| `session.resume` | `eventCount` | None — resume metadata |
| `session.title_changed` | `title` | Updates session label |
| `session.error` | `errorType, message, stack` | Error markdown in stream |
| `session.idle` | | Status → Idle |
| `session.shutdown` | `usage` | Session cleanup |
| `session.compaction_start` | | Progress indicator |
| `session.compaction_complete` | `checkpointRef` | Updates context |

### Message Events

| Event Type | Data Fields | UI Effect |
|-----------|------------|-----------|
| `user.message` | `content, attachments?` | Captures SDK request ID |
| `assistant.turn_start` | `turnId` | Marks beginning of assistant turn (used in transcript) |
| `assistant.message` | `messageId, content, parentToolCallId?` | Markdown to stream |
| `assistant.message_delta` | `messageId, deltaContent, parentToolCallId?` | Streaming markdown |
| `assistant.usage` | `inputTokens, outputTokens` | Token usage reporting |
| `assistant.turn_end` | `turnId` | Marks end of assistant turn (used in transcript) |

### Tool Events

| Event Type | Data Fields | UI Effect |
|-----------|------------|-----------|
| `tool.execution_start` | `toolCallId, toolName, arguments, parentToolCallId?` | Tool invocation UI part |
| `tool.execution_complete` | `toolCallId, success, result?, error?` | Tool result rendering |

### Permission Events

| Event Type | Data Fields | UI Effect |
|-----------|------------|-----------|
| `permission.requested` | `requestId, permissionRequest` | Permission dialog |
| `exit_plan_mode.requested` | `requestId, actions, recommendedAction` | Plan approval dialog |
| `user_input.requested` | `requestId, question, choices?, allowFreeform` | User question UI |

### Subagent Events

| Event Type | Data Fields | UI Effect |
|-----------|------------|-----------|
| `subagent.started` | `toolCallId, agentDisplayName, agentDescription` | Enriches tool part |
| `subagent.completed` | `toolCallId, agentDisplayName` | Logging |
| `subagent.failed` | `toolCallId, agentDisplayName` | Logging |

### Hook Events

| Event Type | Data Fields | UI Effect |
|-----------|------------|-----------|
| `hook.start` | `hookInvocationId, hookType, input` | OTel enrichment |
| `hook.end` | `hookInvocationId, hookType, success, output?, error?` | OTel enrichment |

## Tool Type Definitions

### ToolInfo Union

`ToolInfo` is a union of all recognized tool shapes. Each member is a typed interface with a literal `toolName` discriminant and a typed `arguments` object.

```typescript
// Actual definition from copilotCLITools.ts (lines 405-414):
export type ToolInfo =
    | StringReplaceEditorTool   // toolName: 'str_replace_editor' (legacy composite — wraps create/view/edit/str_replace/insert)
    | EditTool                  // toolName: 'edit' | 'str_replace'
    | CreateTool                // toolName: 'create'
    | ViewTool                  // toolName: 'view'
    | InsertTool                // toolName: 'insert'
    | ShellTool                 // toolName: 'bash' | 'powershell'
    | WriteShellTool            // toolName: 'write_bash' | 'write_powershell'
    | ReadShellTool             // toolName: 'read_bash' | 'read_powershell'
    | StopShellTool             // toolName: 'stop_bash' | 'stop_powershell'
    | ListShellTool             // toolName: 'list_bash' | 'list_powershell'
    | GrepTool                  // toolName: 'grep' | 'rg'
    | GLobTool                  // toolName: 'glob'  (note: casing matches source)
    | ReportIntentTool          // toolName: 'report_intent'
    | ThinkTool                 // toolName: 'think'
    | ReportProgressTool        // toolName: 'report_progress'
    | SearchCodeSubagentTool    // toolName: 'search_code_subagent'
    | ReplyToCommentTool        // toolName: 'reply_to_comment'
    | CodeReviewTool            // toolName: 'code_review'
    | WebFetchTool              // toolName: 'web_fetch'
    | UpdateTodoTool            // toolName: 'update_todo'
    | WebSearchTool             // toolName: 'web_search'
    | ShowFileTool              // toolName: 'show_file'
    | FetchCopilotCliDocumentationTool  // toolName: 'fetch_copilot_cli_documentation'
    | ProposeWorkTool           // toolName: 'propose_work'
    | TaskCompleteTool          // toolName: 'task_complete'
    | AskUserTool               // toolName: 'ask_user'
    | SkillTool                 // toolName: 'skill'
    | TaskTool                  // toolName: 'task'
    | ListAgentsTool            // toolName: 'list_agents'
    | ReadAgentTool             // toolName: 'read_agent'
    | WriteAgentTool            // toolName: 'write_agent'
    | ExitPlanModeTool          // toolName: 'exit_plan_mode'
    | SqlTool                   // toolName: 'sql'
    | LspTool                   // toolName: 'lsp'
    | CreatePullRequestTool     // toolName: 'create_pull_request'
    | DependencyCheckerTool     // toolName: 'gh-advisory-database'
    | StoreMemoryTool           // toolName: 'store_memory'
    | ParallelValidationTool    // toolName: 'parallel_validation'
    | ApplyPatchTool            // toolName: 'apply_patch'
    | McpReloadTool             // toolName: 'mcp_reload'
    | McpValidateTool           // toolName: 'mcp_validate'
    | ToolSearchTool            // toolName: 'tool_search_tool_regex'
    | CodeQLCheckerTool;        // toolName: 'codeql_checker'
```

### ToolCall and UnknownToolCall

`ToolCall` is an **intersection type** — it extends any `ToolInfo` member with session-level metadata. `UnknownToolCall` is a **separate type** for unrecognized tools, not part of the `ToolInfo` union.

```typescript
// Intersection — not a discriminated union
export type ToolCall = ToolInfo & {
    toolCallId: string;
    mcpServerName?: string | undefined;
    mcpToolName?: string | undefined;
};

// Separate fallback type for tools not in ToolInfo
export type UnknownToolCall = {
    toolName: string;
    arguments: unknown;
    toolCallId: string;
};
```

## Event Processing Functions

### processToolExecutionStart()

Converts `tool.execution_start` events into VS Code response parts:

```typescript
export function processToolExecutionStart(
    event: ToolExecutionStartEvent,
    pendingToolInvocations: Map<string, [ChatToolInvocationPart | ChatResponseMarkdownPart | ChatResponseThinkingProgressPart, toolData: ToolCall, parentToolCallId: string | undefined]>,
    workingDirectory?: URI,
): ChatToolInvocationPart | ChatResponseMarkdownPart | ChatResponseThinkingProgressPart | undefined {
    // Delegates to createCopilotCLIToolInvocation for UI part creation
    // Resolves parentToolCallId chain via resolveRootSubagentId
    // Stores in pendingToolInvocations map for later completion
}
```

**Dispatch logic:**
- **Edit tools** (`create`, `edit`, `str_replace`, `insert`) → Text edit parts with file URIs
- **Shell tools** (`bash`, `powershell`) → Terminal tool invocation parts
- **Search tools** (`grep`, `glob`, `view`) → Standard tool invocation parts
- **Think tool** (toolName `'think'`) → `ChatResponseThinkingProgressPart`
- **Unrecognized tools** (not in `ToolFriendlyNameAndHandlers`) → fallthrough branch; if `mcpServerName` and `mcpToolName` properties are present on the tool call, rendered as MCP-specific invocation parts; otherwise rendered as generic unknown tool parts

### processToolExecutionComplete()

Converts `tool.execution_complete` events into completion parts. Returns a **3-tuple** (not 2-tuple): the response part, the tool data, and the parent tool call ID (if nested under a subagent).

```typescript
export function processToolExecutionComplete(
    event: ToolExecutionCompleteEvent,
    pendingToolInvocations: Map<string, [ChatToolInvocationPart | ChatResponseMarkdownPart | ChatResponseThinkingProgressPart, toolData: ToolCall, parentToolCallId: string | undefined]>,
    logger: ILogger,
    workingDirectory?: URI,
): [ChatToolInvocationPart | ChatResponseMarkdownPart | ChatResponseThinkingProgressPart, toolData: ToolCall, parentToolCallId: string | undefined] | undefined {
    // Completes the pending invocation part
    // Updates tool result data (isComplete, isError, isConfirmed)
    // Returns undefined if no pending invocation found
}
```

### enrichToolInvocationWithSubagentMetadata()

Updates a pending tool invocation with richer metadata from a `subagent.started` event. The `task` tool's own arguments carry minimal info; this function backfills display name and description onto the `ChatSubagentToolInvocationData`.

```typescript
export function enrichToolInvocationWithSubagentMetadata(
    toolCallId: string,
    agentDisplayName: string,
    agentDescription: string | undefined,
    pendingToolInvocations: Map<string, [ChatToolInvocationPart | ChatResponseMarkdownPart | ChatResponseThinkingProgressPart, toolData: ToolCall, parentToolCallId: string | undefined]>
): void;
```

### createCopilotCLIToolInvocation()

Factory function that maps a `ToolCall` to the appropriate VS Code response part. Dispatches based on `toolName` against the `ToolFriendlyNameAndHandlers` registry. Returns `undefined` for `report_intent` and `show_file`; returns `ChatResponseThinkingProgressPart` for `think`; returns `ChatResponseMarkdownPart` for `task_complete` with a summary.

```typescript
export function createCopilotCLIToolInvocation(
    data: { toolCallId: string; toolName: string; arguments?: unknown; mcpServerName?: string; mcpToolName?: string },
    editId?: string,
    workingDirectory?: URI,
    logger?: ILogger,
): ChatToolInvocationPart | ChatResponseMarkdownPart | ChatResponseThinkingProgressPart | undefined;
```

### resolveRootSubagentId() *(internal)*

Walks the `parentToolCallId` chain through `pendingToolInvocations` to find the root (top-level) subagent tool call ID. Prevents infinite loops via a visited set. Not exported — used internally by `processToolExecutionStart`.

```typescript
function resolveRootSubagentId(
    parentToolCallId: string,
    pendingToolInvocations: Map<string, [ChatToolInvocationPart | ChatResponseMarkdownPart | ChatResponseThinkingProgressPart, toolData: ToolCall, parentToolCallId: string | undefined]>
): string;
```

## History Reconstruction

### buildChatHistoryFromEvents()

Converts raw `SessionEvent[]` into VS Code chat turns:

```typescript
export function buildChatHistoryFromEvents(
    sessionId: string,
    modelId: string | undefined,
    events: readonly SessionEvent[],
    getVSCodeRequestId: (sdkRequestId: string) => RequestIdDetails | undefined,
    delegationSummaryService: IChatDelegationSummaryService,
    logger: ILogger,
    workingDirectory?: URI,
    defaultModeInstructionsForLastRequest?: StoredModeInstructions,
): (ChatRequestTurn2 | ChatResponseTurn2)[] {
    // Processes events sequentially:
    // 1. user.message → ChatRequestTurn2
    // 2. assistant.message / tool events → ChatResponseTurn2
    // 3. Groups response parts per turn
    // 4. Maps SDK request IDs to VS Code request IDs
}
```

### Event Chain to Turn Mapping

```
Events:                          Chat Turns:
user.message [A]            →   ChatRequestTurn2 { prompt: "..." }
assistant.turn_start [B]    →   ChatResponseTurn2 {
tool.execution_start [C]    →     parts: [
tool.execution_complete [D] →       ChatToolInvocationPart,
assistant.message_delta [E] →       ChatResponseMarkdownPart,
assistant.message [F]       →     ]
assistant.turn_end [G]      →   }
user.message [H]            →   ChatRequestTurn2 { prompt: "..." }
...
```

### Partial History (for unopened sessions)

```typescript
// In CopilotCLISessionService:
public async tryGetPartialSesionHistory(sessionId: string) {
    // Reads events.jsonl directly (without opening SDK session)
    const events = await readSessionEventsFile(sessionId);
    return buildChatHistoryFromEvents(sessionId, undefined, events, ...);
}
```

This provides fast read-only access to session history for the session list preview.

## Event Chain Integrity

Events are linked via `parentId` forming a singly-linked list:

```typescript
// Conceptual illustration — not actual code in the codebase.
// Validates that events form a singly-linked list via parentId.
function validateEventChain(events: SessionEvent[]): boolean {
    for (let i = 1; i < events.length; i++) {
        if (events[i].parentId !== events[i - 1].id) {
            return false;  // Chain broken
        }
    }
    return events[0].parentId === null;  // Root has no parent
}
```

### Fork Truncation

When forking a session, the event chain is truncated at a specific event:

```typescript
await session.truncateToEvent(eventId);
const events = session.getEvents();
const contents = Buffer.from(
    events.map(e => JSON.stringify(e)).join(EOL) + EOL
);
await fileSystem.writeFile(eventsFile, contents);
```

This preserves chain integrity by rewriting the events.jsonl with only the prefix up to the truncation point.

## Stripping Reminders

The SDK sometimes wraps user prompts with XML tags. The extension strips these before display. The function removes all seven tag types:

```typescript
export function stripReminders(content: string): string {
    // Strips the following XML wrappers (in order):
    // 1. <reminder>...</reminder>
    // 2. <attachments>...</attachments>
    // 3. <userRequest>...</userRequest>
    // 4. <context>...</context>
    // 5. <current_datetime>...</current_datetime>
    // 6. <pr_metadata .../> (self-closing)
    // 7. <user_query .../> (self-closing)
    // Result is trimmed.
}
```
