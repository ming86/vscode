# Agent Host Protocol

## Overview

The VS Code platform includes a dedicated **Agent Host** subsystem (`src/vs/platform/agentHost/` in the vscode repo) that provides a second integration pathway for Copilot CLI sessions. Unlike the in-process SDK approach used by the extension, the Agent Host runs the SDK in a separate utility process and communicates via JSON-RPC 2.0.

This document covers the protocol specification, state model, action system, and reconciliation algorithm.

**Source:** `src/vs/platform/agentHost/` (in the vscode repository). The canonical protocol specification is `protocol.md` in that directory; the TypeScript types live in `common/state/protocol/` and are auto-generated from an external `agent-host-protocol` repo via `scripts/sync-agent-host-protocol.ts`.

## Architecture

```
┌─────────────────────────────┐     JSON-RPC 2.0     ┌──────────────────────┐
│  VS Code Renderer Process   │ ◄──────────────────► │  Agent Host Process   │
│                             │                       │                       │
│  AgentHostClient            │   MessagePort /       │  CopilotClient        │
│  ├─ URI subscriptions       │   WebSocket           │  ├─ @github/copilot   │
│  ├─ State rendering         │                       │  ├─ Session manager    │
│  └─ Action dispatch         │                       │  └─ State tree         │
└─────────────────────────────┘                       └──────────────────────┘
```

The protocol is **agent-agnostic**. Two protocol layers exist:

1. **`IAgent` interface** (`common/agentService.ts`) — internal, agent-specific. Each agent backend (CopilotAgent, MockAgent) fires `IAgentProgressEvent`s (raw SDK events: `delta`, `tool_start`, `tool_complete`, etc.).
2. **Sessions state protocol** (`common/state/`) — client-facing, agent-agnostic. An `agentEventMapper.ts` translates raw events into state actions. Clients receive immutable state snapshots and action streams via JSON-RPC.

## Transport

| Environment | Transport | Implementation |
|-------------|-----------|---------------|
| Desktop (Electron) | MessagePort | `IMessagePassingProtocol` via Electron IPC |
| Remote (SSH/WSL) | WebSocket | Standard WebSocket connection |
| Web | WebSocket | Browser WebSocket API |

The entire feature is gated behind `chat.agentHost.enabled` (default `false`), defined as `AgentHostEnabledSettingId` in `agentService.ts`.

## State Model

The protocol uses a **Redux-like immutable state tree** where:
- State is the single source of truth
- Actions are the sole mutation mechanism
- The UI renders from state, never from events directly
- The same reducer code runs on both server and client (enabling write-ahead)

### Root State

Subscribable at `agenthost:/root`. Contains global, lightweight data. **Does not contain the session list** — that is fetched imperatively via the `listSessions` RPC command.

```typescript
// Source: common/state/protocol/state.ts — IRootState
interface IRootState {
    agents: IAgentInfo[];
    activeSessions?: number;
}

interface IAgentInfo {
    provider: string;                            // e.g. 'copilot'
    displayName: string;
    description: string;
    models: ISessionModelInfo[];
    protectedResources?: IProtectedResourceMetadata[];  // RFC 9728 semantics
    customizations?: ICustomizationRef[];
}

interface ISessionModelInfo {
    id: string;
    provider: string;
    name: string;
    maxContextWindow?: number;
    supportsVision?: boolean;
    policyState?: PolicyState;                   // 'enabled' | 'disabled' | 'unconfigured'
}
```

### Session State

Subscribable at the session's URI (e.g. `copilot:/<uuid>`). Contains the full state for a single session.

```typescript
// Source: common/state/protocol/state.ts — ISessionState
interface ISessionState {
    summary: ISessionSummary;
    lifecycle: SessionLifecycle;                 // 'creating' | 'ready' | 'creationFailed'
    creationError?: IErrorInfo;
    serverTools?: IToolDefinition[];
    activeClient?: ISessionActiveClient;
    workingDirectory?: URI;
    turns: ITurn[];
    activeTurn?: IActiveTurn;
    steeringMessage?: IPendingMessage;
    queuedMessages?: IPendingMessage[];
    customizations?: ISessionCustomization[];
}
```

`lifecycle` tracks the asynchronous creation process. When a client creates a session, it picks a URI, sends the `createSession` command, and subscribes immediately. The initial snapshot has `lifecycle: 'creating'`. The server asynchronously initializes the backend and dispatches `session/ready` or `session/creationFailed`.

```typescript
// Source: common/state/protocol/state.ts — ISessionSummary
interface ISessionSummary {
    resource: URI;                              // Session URI
    provider: string;                           // Agent provider ID
    title: string;
    status: SessionStatus;                      // 'idle' | 'in-progress' | 'error'
    createdAt: number;
    modifiedAt: number;
    model?: string;
    workingDirectory?: URI;
}
```

### Turn Types

```typescript
// Source: common/state/protocol/state.ts — ITurn, IActiveTurn
interface ITurn {
    id: string;
    userMessage: IUserMessage;
    responseParts: IResponsePart[];
    usage: IUsageInfo | undefined;
    state: TurnState;                           // 'complete' | 'cancelled' | 'error'
    error?: IErrorInfo;
}

interface IActiveTurn {
    id: string;
    userMessage: IUserMessage;
    responseParts: IResponsePart[];             // includes tool call parts interleaved with text
    usage: IUsageInfo | undefined;
}

interface IUserMessage {
    text: string;
    attachments?: IMessageAttachment[];
}
```

### Tool Call Lifecycle

Tool calls are represented as response parts (`IToolCallResponsePart`) in the stream and follow a state machine:

```
streaming → pending-confirmation → running → completed
                                         → pending-result-confirmation → completed
                                         → cancelled
              → cancelled (denied/skipped)
```

```typescript
// Source: common/state/protocol/state.ts — IToolCallState
type IToolCallState =
    | IToolCallStreamingState                    // LM is streaming parameters
    | IToolCallPendingConfirmationState           // Parameters complete, awaiting user approval
    | IToolCallRunningState                       // Tool is executing
    | IToolCallPendingResultConfirmationState     // Execution done, result awaiting approval
    | IToolCallCompletedState                     // Terminal: tool succeeded or failed
    | IToolCallCancelledState;                    // Terminal: tool was denied/skipped
```

All tool call states share a common `IToolCallBase`:

```typescript
interface IToolCallBase {
    toolCallId: string;
    toolName: string;                           // Internal name (for debugging)
    displayName: string;                        // Human-readable name
    toolClientId?: string;                      // If set, owning client must execute the tool
    _meta?: Record<string, unknown>;            // Provider-specific metadata (e.g. ptyTerminal)
}
```

### Content References

Large content (images, long tool outputs) is **not** inlined in state. A `ContentRef` placeholder is used:

```typescript
// Source: common/state/protocol/state.ts — IContentRef
interface IContentRef {
    uri: URI;                                   // scheme://sessionId/contentId
    sizeHint?: number;
    contentType?: string;
}
```

Clients fetch content separately via the `resourceRead` command. This keeps the state tree small and serializable.

### Session List

The session list can be arbitrarily large and is **not** part of the state tree. Instead:
- Clients fetch it imperatively via `listSessions()` RPC.
- The server sends ephemeral **notifications** (`notify/sessionAdded`, `notify/sessionRemoved`) so clients can update a local cache.
- Notifications are not processed by reducers, not replayed on reconnect. On reconnect, clients re-fetch the list.

### URI Subscriptions

Clients subscribe to state slices via URI patterns:

| URI Pattern | State Slice |
|-------------|------------|
| `agenthost:/root` | Root state (agents and their models, active session count) |
| `copilot:/<sessionId>` | Per-session state (turns, tools, lifecycle) |

The `subscribe(resource)` / `unsubscribe(resource)` mechanism works identically for all resource types. `subscribe` is a JSON-RPC **request** — the snapshot is returned as the response result.

## Action System

Actions are the sole mutation mechanism for subscribable state. They form a discriminated union keyed by `type`. Every action is wrapped in an `IActionEnvelope` for sequencing and origin tracking.

### Action Envelope

```typescript
// Source: common/state/protocol/actions.ts — IActionEnvelope
interface IActionEnvelope {
    readonly action: IStateAction;
    readonly serverSeq: number;                  // Monotonic, assigned by server
    readonly origin: IActionOrigin | undefined;  // undefined = server-originated
    readonly rejectionReason?: string;           // Present when server rejected the action
}

interface IActionOrigin {
    clientId: string;
    clientSeq: number;
}
```

### Root Actions

All server-only. Clients observe them but cannot produce them.

| Type | Payload | Description |
|------|---------|-------------|
| `root/agentsChanged` | `IAgentInfo[]` | Available agent backends or their models changed |
| `root/activeSessionsChanged` | `activeSessions: number` | Active session count changed |

### Session Actions

All scoped to a session URI. Some are server-only; those marked "Client" can be dispatched by clients.

| Type | Origin | Description |
|------|--------|-------------|
| `session/ready` | Server | Session backend initialized successfully |
| `session/creationFailed` | Server | Session backend failed to initialize (`IErrorInfo`) |
| `session/turnStarted` | Client | User sent a message; server starts agent processing |
| `session/delta` | Server | Streaming text chunk, targeted at a response part by `partId` |
| `session/responsePart` | Server | Structured content (markdown, content ref, tool call, reasoning) appended |
| `session/toolCallStart` | Server | Tool call begins — parameters are streaming from the LM |
| `session/toolCallDelta` | Server | Streaming partial parameters for a tool call |
| `session/toolCallReady` | Server | Tool call parameters complete, transitions to pending-confirmation or running |
| `session/toolCallConfirmed` | Client | Client approves or denies a pending tool call |
| `session/toolCallComplete` | Client | Tool execution finished (result provided) |
| `session/toolCallResultConfirmed` | Client | Client approves or denies a tool's result |
| `session/turnComplete` | Server | Turn finished — assistant idle |
| `session/turnCancelled` | Client | Turn aborted; server stops processing |
| `session/error` | Server | Error during turn processing (`IErrorInfo`) |
| `session/titleChanged` | Both | Session title updated |
| `session/usage` | Server | Token usage report for a turn (`IUsageInfo`) |
| `session/reasoning` | Server | Reasoning/thinking text, targeted at a reasoning part by `partId` |
| `session/modelChanged` | Client | Model changed for this session |
| `session/serverToolsChanged` | Server | Server-side tool list replaced |
| `session/activeClientChanged` | Client | Active client for session changed (or unset) |
| `session/activeClientToolsChanged` | Client | Active client's tool list replaced |
| `session/pendingMessageSet` | Client | Steering or queued message set (upsert) |
| `session/pendingMessageRemoved` | Client | Steering or queued message removed |
| `session/queuedMessagesReordered` | Client | Queued message order changed |
| `session/customizationsChanged` | Server | Session customizations replaced |
| `session/customizationToggled` | Client | Client toggled a customization on/off |
| `session/truncated` | Client | Truncates session history |

### Client-Dispatched Actions and Side Effects

When a client dispatches an action, the server applies it to state and also reacts with side effects (e.g. `session/turnStarted` triggers agent processing, `session/turnCancelled` aborts it). This avoids a separate command-to-action translation layer.

| Client Action | Server-Side Effect |
|---|---|
| `session/turnStarted` | Begins agent processing for the new turn |
| `session/toolCallConfirmed` | Unblocks pending tool execution (or cancels it if denied) |
| `session/toolCallComplete` | Provides tool result (for client-provided tools) |
| `session/toolCallResultConfirmed` | Approves or denies a tool's result |
| `session/turnCancelled` | Aborts the in-progress turn |

### Action Flow (Typical Turn)

```
User Action          Client                    Agent Host
    │                   │                           │
    ├── Send prompt ──► │ ── dispatchAction ──────► │ (session/turnStarted)
    │                   │                           ├── Start agent processing
    │                   │ ◄── action ────────────── ├── session/responsePart (markdown)
    │                   │ ◄── action ────────────── ├── session/delta (streaming text)
    │                   │ ◄── action ────────────── ├── session/responsePart (toolCall)
    │                   │ ◄── action ────────────── ├── session/toolCallStart
    │                   │ ◄── action ────────────── ├── session/toolCallDelta
    │                   │ ◄── action ────────────── ├── session/toolCallReady
    │  (user approves)  │ ── dispatchAction ──────► │ (session/toolCallConfirmed)
    │                   │ ◄── action ────────────── ├── session/toolCallComplete
    │                   │ ◄── action ────────────── ├── session/responsePart (markdown)
    │                   │ ◄── action ────────────── ├── session/delta
    │                   │ ◄── action ────────────── ├── session/usage
    │                   │ ◄── action ────────────── ├── session/turnComplete
```

## JSON-RPC Commands (Client to Server)

Commands are JSON-RPC requests — the server returns a result or a JSON-RPC error.

```typescript
// Source: common/state/protocol/messages.ts — ICommandMap
interface ICommandMap {
    'initialize':     { params: IInitializeParams;     result: IInitializeResult };
    'reconnect':      { params: IReconnectParams;      result: IReconnectResult };
    'subscribe':      { params: ISubscribeParams;      result: ISubscribeResult };
    'createSession':  { params: ICreateSessionParams;  result: null };
    'disposeSession': { params: IDisposeSessionParams; result: null };
    'listSessions':   { params: IListSessionsParams;   result: IListSessionsResult };
    'resourceRead':   { params: IResourceReadParams;   result: IResourceReadResult };
    'resourceWrite':  { params: IResourceWriteParams;  result: IResourceWriteResult };
    'resourceList':   { params: IResourceListParams;   result: IResourceListResult };
    'resourceCopy':   { params: IResourceCopyParams;   result: IResourceCopyResult };
    'resourceDelete': { params: IResourceDeleteParams; result: IResourceDeleteResult };
    'resourceMove':   { params: IResourceMoveParams;   result: IResourceMoveResult };
    'fetchTurns':     { params: IFetchTurnsParams;     result: IFetchTurnsResult };
    'authenticate':   { params: IAuthenticateParams;   result: IAuthenticateResult };
}
```

| Command | Description |
|---------|-------------|
| `initialize` | Handshake: negotiate protocol version, provide `clientId`, optional `initialSubscriptions`. Returns snapshots. |
| `reconnect` | Re-establish a dropped connection. Server replays missed actions or provides fresh snapshots. |
| `subscribe` | Subscribe to a URI; returns current state snapshot. |
| `createSession` | Create a session at a client-chosen URI. Params: `session`, `provider?`, `model?`, `workingDirectory?`, `fork?`. |
| `disposeSession` | Dispose a session. Server broadcasts `notify/sessionRemoved`. |
| `listSessions` | Returns `ISessionSummary[]`. |
| `resourceRead` | Fetch content by URI (for `ContentRef` resolution). Supports base64 and UTF-8 encoding. |
| `resourceWrite` | Write content to a file on the server's filesystem. |
| `resourceList` | List directory entries at a file URI on the server's filesystem. |
| `resourceCopy` | Copy a resource between URIs on the server's filesystem. |
| `resourceDelete` | Delete a resource. |
| `resourceMove` | Move (rename) a resource. |
| `fetchTurns` | Lazy-load historical turns for a session. Supports cursor-based pagination (`before`, `limit`). |
| `authenticate` | Push a Bearer token for a protected resource (RFC 6750 semantics). |

### Client to Server Notifications (Fire-and-Forget)

```typescript
// Source: common/state/protocol/messages.ts — IClientNotificationMap
interface IClientNotificationMap {
    'unsubscribe':    { params: IUnsubscribeParams };
    'dispatchAction': { params: IDispatchActionParams };  // { clientSeq, action }
}
```

## Server to Client Notifications

```typescript
// Source: common/state/protocol/messages.ts — IServerNotificationMap
interface IServerNotificationMap {
    'action':       { params: IActionEnvelope };           // State mutation delivery
    'notification': { params: { notification: IProtocolNotification } };  // Ephemeral broadcasts
}
```

### Protocol Notifications

Ephemeral broadcasts **not** part of the state tree. Not processed by reducers, not replayed on reconnect.

```typescript
// Source: common/state/protocol/notifications.ts
enum NotificationType {
    SessionAdded  = 'notify/sessionAdded',      // Payload: ISessionSummary
    SessionRemoved = 'notify/sessionRemoved',    // Payload: session URI
    AuthRequired  = 'notify/authRequired',       // Payload: resource, reason?
}
```

`notify/authRequired` is sent when a previously valid token expires or when the server discovers a new authentication requirement. The client should obtain a fresh token and push it via `authenticate`.

## Connection Handshake

`initialize` is a JSON-RPC **request** — the first message from the client:

```
Client → Server:  { "jsonrpc": "2.0", "id": 1, "method": "initialize",
                    "params": { "protocolVersion": 1, "clientId": "abc", "initialSubscriptions": ["agenthost:/root"] } }

Server → Client:  { "jsonrpc": "2.0", "id": 1, "result":
                    { "protocolVersion": 1, "serverSeq": 42, "snapshots": [...], "defaultDirectory": "file:///home/user" } }
```

`initialSubscriptions` allows the client to subscribe to root state (and any previously-open sessions on reconnect) in the same round-trip as the handshake. If the server does not support the client's protocol version, it returns error code `-32005` (`UnsupportedProtocolVersion`).

### Session Creation Flow

1. Client picks a session URI (e.g. `copilot:/<new-uuid>`)
2. Client sends `createSession(uri, config)` command
3. Client sends `subscribe(uri)` (can be batched)
4. Server creates the session in state with `lifecycle: 'creating'` and sends the subscription snapshot
5. Server asynchronously initializes the agent backend
6. On success: server dispatches `session/ready` action
7. On failure: server dispatches `session/creationFailed` action with error details
8. Server broadcasts `notify/sessionAdded` to all clients

### Authentication Flow

Agents declare protected resources in `IAgentInfo.protectedResources` using RFC 9728 metadata. The client:
1. Reads `protectedResources` from root state
2. Obtains a token from the declared `authorization_servers`
3. Pushes the token via `authenticate({ resource, token })`
4. If the token expires, the server sends `notify/authRequired`; the client re-authenticates

## Write-Ahead Reconciliation

The protocol supports optimistic updates for responsive UI. Clients dispatch actions via `dispatchAction` (a JSON-RPC notification — fire-and-forget). The client applies the action to local state immediately.

### Client-Side State

Each client maintains per-subscription:
- `confirmedState` — last fully server-acknowledged state
- `pendingActions[]` — optimistically applied but not yet echoed by server
- `optimisticState` — `confirmedState` with `pendingActions` replayed on top (computed, not stored)

### Reconciliation Algorithm

When the client receives an `IActionEnvelope` from the server:

1. **Own action echoed**: `origin.clientId === myId` and matches head of `pendingActions` — pop from pending, apply to `confirmedState`
2. **Foreign action**: different origin — apply to `confirmedState`, rebase remaining `pendingActions`
3. **Rejected action**: server echoed with `rejectionReason` present — remove from pending (optimistic effect reverted)
4. Recompute `optimisticState` from `confirmedState` + remaining `pendingActions`

Rebasing is typically straightforward because most session actions are **append-only** (add turn, append delta, add tool call). The rare true conflict (e.g. two clients cancelling the same turn) is resolved by server-wins semantics.

## Protocol Versioning

### Version Constants

```typescript
// Source: common/state/protocol/version/registry.ts
const PROTOCOL_VERSION = 1;      // Current version
const MIN_PROTOCOL_VERSION = 1;  // Oldest supported version
```

Version is bumped when a new feature area requires capability negotiation or behavioral semantics of existing actions change. Adding **optional** fields does not require a bump.

### Compile-Time Compatibility Checks

The upstream `agent-host-protocol` repo's design documents (`protocol.md`, `AGENTS.md`) describe a pattern where each protocol version would have a dedicated type file (`versions/v1.ts`) capturing frozen wire format shapes, with bidirectional `AssertCompatible` checks enforcing backwards compatibility:

```typescript
// Design pattern described in protocol.md — not present in the synced codebase.
// The actual synced code contains only version/registry.ts with ACTION_INTRODUCED_IN
// and PROTOCOL_VERSION constants. No versions/v1.ts or AssertCompatible checks exist
// in the auto-generated code at common/state/protocol/version/.
type AssertCompatible<Frozen, Current extends Frozen> = Frozen extends Current ? true : never;
type _check = AssertCompatible<IV1_TurnStartedAction, ITurnStartedAction>;
```

**Status:** Design intent documented upstream; not yet materialized in the synced protocol types. The version registry (`version/registry.ts`) currently enforces only the exhaustive action-to-version map (`ACTION_INTRODUCED_IN`) — a compile-time guarantee that every action type declares its introduction version.

### Exhaustive Action-to-Version Map

```typescript
// Source: common/state/protocol/version/registry.ts — ACTION_INTRODUCED_IN
const ACTION_INTRODUCED_IN: { readonly [K in IStateAction['type']]: number } = {
    'root/agentsChanged': 1,
    'root/activeSessionsChanged': 1,
    'session/ready': 1,
    'session/creationFailed': 1,
    'session/turnStarted': 1,
    'session/delta': 1,
    'session/responsePart': 1,
    'session/toolCallStart': 1,
    'session/toolCallDelta': 1,
    'session/toolCallReady': 1,
    'session/toolCallConfirmed': 1,
    'session/toolCallComplete': 1,
    'session/toolCallResultConfirmed': 1,
    'session/turnComplete': 1,
    'session/turnCancelled': 1,
    'session/error': 1,
    'session/titleChanged': 1,
    'session/usage': 1,
    'session/reasoning': 1,
    'session/modelChanged': 1,
    'session/serverToolsChanged': 1,
    'session/activeClientChanged': 1,
    'session/activeClientToolsChanged': 1,
    'session/pendingMessageSet': 1,
    'session/pendingMessageRemoved': 1,
    'session/queuedMessagesReordered': 1,
    'session/customizationsChanged': 1,
    'session/customizationToggled': 1,
    'session/truncated': 1,
};
```

The index signature `[K in IStateAction['type']]` means adding a new action to the union without an entry here is a compile error. The server uses this for one-line filtering:

```typescript
function isActionKnownToVersion(action: IStateAction, clientVersion: number): boolean {
    return ACTION_INTRODUCED_IN[action.type] <= clientVersion;
}
```

### Capabilities

The protocol version maps to a `ProtocolCapabilities` interface:

```typescript
// Source: common/state/protocol/version/registry.ts
interface ProtocolCapabilities {
    readonly sessions: true;        // v1 — always present
    readonly tools: true;           // v1 — always present
    readonly permissions: true;     // v1 — always present
}
```

### Forward Compatibility

A newer client connecting to an older server:
1. During handshake, the client learns the server's protocol version from the `initialize` response.
2. The client derives `ProtocolCapabilities` from the server version.
3. The server only sends action types known to the client's declared version (via `isActionKnownToVersion`).
4. As a safety net, clients silently ignore actions with unrecognized `type` values.

### Error Codes

Source uses `AhpErrorCodes.PascalCase` (e.g., `AhpErrorCodes.SessionNotFound`) and `JsonRpcErrorCodes.PascalCase` constants. The `AHP_SNAKE_CASE` names below are descriptive aliases for documentation clarity.

```typescript
// Source: common/state/protocol/errors.ts (via sessionProtocol.ts)

// Standard JSON-RPC 2.0 codes (JsonRpcErrorCodes.*)
const JSON_RPC_PARSE_ERROR          = -32700;  // (source: JsonRpcErrorCodes.ParseError)
const JSON_RPC_INVALID_REQUEST      = -32600;  // (source: JsonRpcErrorCodes.InvalidRequest)
const JSON_RPC_METHOD_NOT_FOUND     = -32601;  // (source: JsonRpcErrorCodes.MethodNotFound)
const JSON_RPC_INVALID_PARAMS       = -32602;  // (source: JsonRpcErrorCodes.InvalidParams)
const JSON_RPC_INTERNAL_ERROR       = -32603;  // (source: JsonRpcErrorCodes.InternalError)

// AHP application codes (AhpErrorCodes.*)
const AHP_SESSION_NOT_FOUND         = -32001;  // (source: AhpErrorCodes.SessionNotFound)
const AHP_PROVIDER_NOT_FOUND        = -32002;  // (source: AhpErrorCodes.ProviderNotFound)
const AHP_SESSION_ALREADY_EXISTS    = -32003;  // (source: AhpErrorCodes.SessionAlreadyExists)
const AHP_TURN_IN_PROGRESS          = -32004;  // (source: AhpErrorCodes.TurnInProgress)
const AHP_UNSUPPORTED_PROTOCOL_VERSION = -32005;  // (source: AhpErrorCodes.UnsupportedProtocolVersion)
const AHP_CONTENT_NOT_FOUND         = -32006;  // (source: AhpErrorCodes.ContentNotFound)
const AHP_AUTH_REQUIRED             = -32007;  // (source: AhpErrorCodes.AuthRequired)
const AHP_NOT_FOUND                 = -32008;  // (source: AhpErrorCodes.NotFound)
const AHP_PERMISSION_DENIED         = -32009;  // (source: AhpErrorCodes.PermissionDenied)
const AHP_ALREADY_EXISTS            = -32010;  // (source: AhpErrorCodes.AlreadyExists)
```

## File Layout

```
src/vs/platform/agentHost/common/state/
├── protocol/                      # Auto-generated from agent-host-protocol repo
│   ├── state.ts                   # Immutable state types (IRootState, ISessionState, ITurn, etc.)
│   ├── actions.ts                 # ActionType enum, action interfaces, IStateAction union
│   ├── action-origin.generated.ts # IRootAction, ISessionAction, IClientSessionAction unions
│   ├── commands.ts                # Command params/results (IInitializeParams, etc.)
│   ├── messages.ts                # ICommandMap, notification maps, typed JSON-RPC messages
│   ├── notifications.ts           # IProtocolNotification union
│   ├── errors.ts                  # Error code enums
│   ├── reducers.ts                # Pure reducer functions
│   └── version/
│       └── registry.ts            # Version constants, ACTION_INTRODUCED_IN, ProtocolCapabilities
├── sessionState.ts                # VS Code re-exports + helpers (createRootState, etc.)
├── sessionActions.ts              # VS Code re-exports + type guards (isRootAction, etc.)
├── sessionReducers.ts             # VS Code reducer wrappers
├── sessionProtocol.ts             # VS Code re-exports + ProtocolError class
├── sessionCapabilities.ts         # Re-exports version constants + ProtocolCapabilities
├── sessionClientState.ts          # Client-side state manager (confirmed + pending + reconciliation)
└── sessionTransport.ts            # Transport abstraction
```

## Relationship to Extension Integration

The agent host protocol is a **parallel pathway** that coexists with the in-process SDK integration:

| Feature | In-Process SDK (Extension) | Agent Host Protocol |
|---------|--------------------------|-------------------|
| Process model | Same process as extension | Separate utility process |
| Communication | Direct function calls | JSON-RPC 2.0 |
| State model | Event-driven (SDK events) | Redux-like state tree |
| Multi-client | Single consumer | Multiple clients supported |
| Used by | Chat extension (primary) | VS Code platform (alternative) |
| Session type | `copilotcli` | `agent-host-copilot` |

The `AgentSessionProviders` enum in `agentSessions.ts` maps well-known types:
- `Local` = `'local'`
- `Background` = `'copilotcli'`
- `Cloud` = `'copilot-cloud-agent'`
- `Claude` = `'claude-code'`
- `Codex` = `'openai-codex'`
- `Growth` = `'copilot-growth'`
- `AgentHostCopilot` = `'agent-host-copilot'`

Both pathways read from and write to the same on-disk session storage (`~/.copilot/session-state/`).
