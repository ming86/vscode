# Session Wrapper — ICopilotCLISession

## Overview

`ICopilotCLISession` wraps a single SDK `Session` object and manages the lifecycle of a conversation turn. It handles stream attachment, request routing (normal vs. steering), event processing, permission handling, tool invocation rendering, and OTel instrumentation.

**Source:** `src/extension/chatSessions/copilotcli/node/copilotcliSession.ts`

## Interface

```typescript
export interface ICopilotCLISession extends IDisposable {
    readonly sessionId: string;
    readonly title?: string;
    readonly createdPullRequestUrl: string | undefined;
    readonly onDidChangeTitle: vscode.Event<string>;
    readonly status: vscode.ChatSessionStatus | undefined;
    readonly onDidChangeStatus: vscode.Event<vscode.ChatSessionStatus | undefined>;
    readonly workspace: IWorkspaceInfo;
    readonly additionalWorkspaces: IWorkspaceInfo[];
    readonly pendingPrompt: string | undefined;

    attachStream(stream: vscode.ChatResponseStream): IDisposable;
    setPermissionLevel(level: string | undefined): void;
    handleRequest(
        request: {
            id: string;
            toolInvocationToken: ChatParticipantToolToken;
            sessionResource?: vscode.Uri;
        },
        input: CopilotCLISessionInput,
        attachments: Attachment[],
        modelId: string | undefined,
        authInfo: NonNullable<SessionOptions['authInfo']>,
        token: vscode.CancellationToken
    ): Promise<void>;
    addUserMessage(content: string): void;
    addUserAssistantMessage(content: string): void;
    getSelectedModelId(): Promise<string | undefined>;
}
```

## Input Types

```typescript
export type CopilotCLICommand = 'compact' | 'plan' | 'fleet';

export type CopilotCLISessionInput =
    | { readonly prompt: string }
    | { readonly prompt?: string; readonly command: CopilotCLICommand };

export const builtinSlashSCommands = {
    commit: '/commit',
    sync: '/sync',
    merge: '/merge',
    createPr: '/create-pr',
    createDraftPr: '/create-draft-pr',
    updatePr: '/update-pr',
};
```

## Constructor Dependencies

```typescript
class CopilotCLISession extends DisposableStore implements ICopilotCLISession {
    constructor(
        private readonly _workspaceInfo: IWorkspaceInfo,
        private readonly _agentName: string | undefined,
        private readonly _sdkSession: Session,       // SDK Session object
        private readonly _additionalWorkspaces: IWorkspaceInfo[],
        @ILogService logService,
        @IWorkspaceService workspaceService,
        @IChatSessionMetadataStore chatSessionMetadataStore,
        @IInstantiationService instantiationService,
        @IRequestLogger requestLogger,
        @ICopilotCLIImageSupport imageSupport,
        @IToolsService toolsService,
        @IUserQuestionHandler userQuestionHandler,
        @IConfigurationService configurationService,
        @IOTelService otelService,
    )
}
```

## Stream Attachment

A session can have at most one attached response stream at a time:

```typescript
attachStream(stream: vscode.ChatResponseStream): IDisposable {
    this._stream = stream;
    return toDisposable(() => {
        if (this._stream === stream) {
            this._stream = undefined;
        }
    });
}
```

The stream receives all output: markdown text, tool invocations, progress updates, and errors.

## Request Handling

### Request Routing

```
handleRequest(request, input, attachments, modelId, authInfo, token)
    │
    ├── Update model and auth if needed
    │
    ├── Check: is session already busy?
    │   ├── YES → _handleRequestSteering() [inject into running conversation]
    │   └── NO  → _handleRequestImpl()     [start new turn]
    │
    └── Chain onto previousRequest promise
```

### Normal Request Flow (_handleRequestImpl)

```
_handleRequestImpl(request, input, attachments, modelId, token)
    │
    ├── Start OTel span: 'invoke_agent copilotcli'
    │   └── Attributes: agent_name, session_id, model_id
    │
    ├── Register SDK event listeners:
    │   ├── '*'                    → Debug logging (all events)
    │   ├── 'permission.requested' → Permission UI + respondToPermission()
    │   ├── 'exit_plan_mode.requested' → Plan approval UI
    │   ├── 'user_input.requested' → User question UI
    │   ├── 'session.title_changed' → Update title display
    │   ├── 'user.message'         → Capture SDK request ID
    │   ├── 'assistant.usage'      → Token usage reporting
    │   ├── 'assistant.message_delta' → Streaming markdown output
    │   ├── 'assistant.message'    → Non-streamed assistant text
    │   ├── 'tool.execution_start' → Tool invocation UI parts
    │   ├── 'tool.execution_complete' → Tool result rendering
    │   ├── 'session.error'        → Error display
    │   ├── 'subagent.started/completed/failed' → Subagent tracking
    │   └── 'hook.start/end'       → Hook OTel enrichment
    │
    ├── sendRequestInternal(input, attachments, false)
    │   └── _sdkSession.send({ prompt, attachments, agentMode })
    │
    ├── Update status: InProgress → Completed/Failed
    └── End OTel span
```

### Steering Request Flow

When a user sends a message while the session is already processing:

```typescript
private async _handleRequestSteering(
    input, attachments, modelId, previousRequestPromise, token
) {
    // Send with mode: 'immediate' — injected into running conversation
    await Promise.all([
        previousRequestPromise,
        this.sendRequestInternal(input, attachments, true, logStartTime)
    ]);
}
```

The steering message resolves only when **both** the injection and the original request complete.

### Send Options

```typescript
private async sendRequestInternal(input, attachments, steering, logStartTime) {
    if ('command' in input && input.command !== 'plan') {
        switch (input.command) {
            case 'compact':
                this._stream?.progress(l10n.t('Compacting conversation...'));
                await this._sdkSession.initializeAndValidateTools();
                this._sdkSession.currentMode = 'interactive';
                const result = await this._sdkSession.compactHistory();
                if (result.success) {
                    this._stream?.markdown(l10n.t('Compacted conversation.'));
                } else {
                    this._stream?.markdown(l10n.t('Unable to compact conversation.'));
                }
                break;
            case 'fleet':
                await this._startFleetAndWaitForIdle(input);
                break;
        }
    } else {
        // Mode selection: plan is checked first, then autopilot/interactive
        if ('command' in input && input.command === 'plan') {
            this._sdkSession.currentMode = 'plan';
        } else if (this._permissionLevel === 'autopilot') {
            this._sdkSession.currentMode = 'autopilot';
        } else {
            this._sdkSession.currentMode = 'interactive';
        }

        const sendOptions: SendOptions = {
            prompt: input.prompt ?? '',
            attachments,
            agentMode: this._sdkSession.currentMode,
        };
        if (steering) {
            sendOptions.mode = 'immediate';
        }
        await this._sdkSession.send(sendOptions);
    }
}
```

**Key structural detail:** `compact` and `fleet` are terminal commands — they do **not** fall through to `send()`. The `plan` command sets the mode and proceeds to `send()` alongside the standard autopilot/interactive selection.

## SDK Event Processing

### Streaming Text

```typescript
this._sdkSession.on('assistant.message_delta', (event) => {
    if (typeof event.data.deltaContent === 'string' && event.data.deltaContent.length) {
        flushPendingInvocationMessages();
        // Skip sub-agent markdown
        if (event.data.parentToolCallId) return;
        chunkMessageIds.add(event.data.messageId);
        this._stream?.markdown(event.data.deltaContent);
    }
});
```

### Tool Execution

```typescript
this._sdkSession.on('tool.execution_start', (event) => {
    toolCalls.set(event.data.toolCallId, event.data);

    if (isCopilotCliEditToolCall(event.data)) {
        // Edit tools handled differently — no UI until complete
        editToolIds.add(event.data.toolCallId);
    } else {
        const responsePart = processToolExecutionStart(event, pendingToolInvocations, workingDir);
        if (responsePart instanceof ChatToolInvocationPart) {
            responsePart.enablePartialUpdate = true;
            if (isCopilotCLIToolThatCouldRequirePermissions(event)) {
                // Delay display until permission resolved
                toolCallWaitingForPermissions.push([responsePart, event.data]);
            } else {
                this._stream?.push(responsePart);
            }
        }
    }
});

this._sdkSession.on('tool.execution_complete', (event) => {
    const [responsePart] = processToolExecutionComplete(
        event, pendingToolInvocations, this.logService, workingDir
    );
    if (responsePart) {
        flushPendingInvocationMessageForToolCallId(event.data.toolCallId);
        this._stream?.push(responsePart);
    }
});
```

### Permission Handling

```typescript
this._sdkSession.on('permission.requested', async (event) => {
    const response = await this.requestPermission(
        event.data.permissionRequest,
        editTracker,
        (toolCallId) => toolCalls.get(toolCallId),
        token
    );
    flushPendingInvocationMessageForToolCallId(event.data.permissionRequest.toolCallId);
    this._sdkSession.respondToPermission(event.data.requestId, response);
});
```

#### Permission Auto-Approval Decision Tree

The private `requestPermission` method (source lines ~912-1018) implements a multi-stage auto-approval decision tree before falling through to interactive UI prompting. Key heuristics, evaluated in order:

1. **Permission level short-circuit** — `autoApprove` or `autopilot` permission levels approve unconditionally.
2. **Read requests** — auto-approved when the target file is:
   - A trusted image (`isTrustedImage`)
   - Inside the session workspace (`isFileFromSessionWorkspace`)
   - Inside any VS Code workspace folder (`getWorkspaceFolder`)
   - Inside the session state directory (`~/.copilot/session-state/{sessionId}/`)
   - An explicitly attached file
3. **Write requests** — auto-approved when:
   - Isolation mode is enabled and the file is within the working directory
   - The file is a workspace file and does not require protected-file edit confirmation (`requiresFileEditconfirmation`)
   - The file is in the working directory (non-isolation) and passes the protected-file check
   - The file is within the session state directory
4. **Fallback** — delegates to the interactive `requestPermission` helper (from `permissionHelpers.ts`), which surfaces the VS Code permission UI. Denial returns `{ kind: 'denied-interactively-by-user' }`.

### User Input Handling

```typescript
this._sdkSession.on('user_input.requested', async (event) => {
    if (this._permissionLevel === 'autopilot') {
        // Auto-respond in autopilot mode
        this._sdkSession.respondToUserInput(event.data.requestId, {
            answer: 'The user is not available to respond...',
            wasFreeform: true,
        });
        return;
    }
    const answer = await this._userQuestionHandler.askUserQuestion(
        userInputRequest, this._toolInvocationToken, token
    );
    this._sdkSession.respondToUserInput(event.data.requestId, {
        answer: answer.freeText || answer.selected.join(', '),
        wasFreeform: !!answer.freeText,
    });
});
```

## Promise Chain Serialization

Requests are serialized via a promise chain to prevent interleaving:

```typescript
private previousRequest: Promise<unknown> = Promise.resolve();

public async handleRequest(...) {
    const previousRequestSnapshot = this.previousRequest;

    const handled = this._requestLogger.captureInvocation(capturingToken, async () => {
        if (isAlreadyBusyWithAnotherRequest) {
            return this._handleRequestSteering(input, attachments, modelId,
                previousRequestSnapshot, token);
        } else {
            return this._handleRequestImpl(request, input, attachments, modelId, token);
        }
    });

    this.previousRequest = this.previousRequest.then(() => handled);
    return handled;
}
```

## OTel Instrumentation

```typescript
private async _handleRequestImpl(...) {
    return this._otelService.startActiveSpan(
        'invoke_agent copilotcli',
        {
            kind: SpanKind.INTERNAL,
            attributes: {
                [GenAiAttr.OPERATION_NAME]: GenAiOperationName.INVOKE_AGENT,
                [GenAiAttr.AGENT_NAME]: 'copilotcli',
                [GenAiAttr.PROVIDER_NAME]: 'github',
                [GenAiAttr.CONVERSATION_ID]: this.sessionId,
                [CopilotChatAttr.SESSION_ID]: this.sessionId,
                [CopilotChatAttr.CHAT_SESSION_ID]: this.sessionId,
            },
        },
        async span => {
            // Register trace so bridge processor can inject CHAT_SESSION_ID
            // into SDK-originated spans that share this trace
            const traceCtx = span.getSpanContext();
            if (traceCtx && this._bridgeProcessor) {
                this._bridgeProcessor.registerTrace(traceCtx.traceId, this.sessionId);
            }
            // Propagate trace context to SDK
            if (traceCtx && this._updateSdkTraceContext) {
                const traceparent = `00-${traceCtx.traceId}-${traceCtx.spanId}-01`;
                this._updateSdkTraceContext(traceparent);
            }
            try {
                // ... handle request
            } finally {
                if (traceCtx && this._bridgeProcessor) {
                    this._bridgeProcessor.unregisterTrace(traceCtx.traceId);
                }
            }
        },
    );
}
```

### Bridge Span Processor

The `CopilotCliBridgeSpanProcessor` injects into the SDK's OTel `TracerProvider` to forward SDK-native spans to the VS Code debug panel:

```typescript
// Installed once after first session creation:
const api = require('@opentelemetry/api');
const globalProvider = api.trace.getTracerProvider();
const delegate = globalProvider._delegate ?? globalProvider;
const processorArray = delegate._activeSpanProcessor._spanProcessors;
processorArray.push(new CopilotCliBridgeSpanProcessor(this._otelService));
```

## Status Transitions

```
Initial: undefined
    │
    ├── handleRequest() → InProgress
    │   ├── turn complete → Completed
    │   └── error → Failed
    │
    ├── steering request → stays InProgress (no change)
    │
    └── NeedsInput (set by SDK events)
```

```typescript
// On request start:
this._status = ChatSessionStatus.InProgress;
this._statusChange.fire(this._status);

// On completion:
this._status = ChatSessionStatus.Completed;
this._statusChange.fire(this._status);

// On error:
this._status = ChatSessionStatus.Failed;
this._statusChange.fire(this._status);
```

## Cancellation

```typescript
disposables.add(token.onCancellationRequested(() => {
    this._sdkSession.abort();
}));
disposables.add(toDisposable(() => this._sdkSession.abort()));
```

Cancellation triggers `sdkSession.abort()` which terminates the current turn and any pending tool executions.
