# Implementation Guide

## Overview

This guide provides a step-by-step walkthrough for implementing Copilot CLI session integration in your own VS Code extension or application. It covers the minimum viable integration (reading existing sessions), progresses through full CRUD operations, and concludes with advanced features like MCP integration and multi-transport support.

## Prerequisites

- Node.js 20+
- VS Code Extension API (or equivalent host)
- Access to `@github/copilot` npm package (includes `/sdk` subpath export)
- GitHub authentication token

## Phase 1: Session Storage Access (Read-Only)

### Step 1: Resolve Session Directory

```typescript
import * as os from 'node:os';
import * as path from 'node:path';

function getCopilotHome(): string {
    const xdgState = process.env['XDG_STATE_HOME'];
    const base = xdgState || os.homedir();
    return path.join(base, '.copilot');
}

function getSessionStateDir(): string {
    return path.join(getCopilotHome(), 'session-state');
}

function getSessionDir(sessionId: string): string {
    return path.join(getSessionStateDir(), sessionId);
}

function getEventsFile(sessionId: string): string {
    return path.join(getSessionDir(sessionId), 'events.jsonl');
}
```

### Step 2: List Sessions

```typescript
import * as fs from 'node:fs/promises';
import * as yaml from 'yaml';  // or js-yaml

interface SessionMetadata {
    id: string;
    cwd: string;
    summary: string;
    summary_count: number;
    created_at: string;
    updated_at: string;
}

async function listSessions(): Promise<SessionMetadata[]> {
    const stateDir = getSessionStateDir();
    const entries = await fs.readdir(stateDir, { withFileTypes: true });

    const sessions: SessionMetadata[] = [];
    for (const entry of entries) {
        if (!entry.isDirectory()) continue;

        const workspacePath = path.join(stateDir, entry.name, 'workspace.yaml');
        try {
            const content = await fs.readFile(workspacePath, 'utf-8');
            const metadata = yaml.parse(content) as SessionMetadata;
            sessions.push(metadata);
        } catch {
            // Skip sessions without workspace.yaml
        }
    }

    return sessions.sort((a, b) =>
        new Date(b.updated_at).getTime() - new Date(a.updated_at).getTime()
    );
}
```

### Step 3: Read Session Events

```typescript
interface SessionEvent {
    type: string;
    data: Record<string, unknown>;
    id: string;
    timestamp: string;
    parentId: string | null;
}

async function readSessionEvents(sessionId: string): Promise<SessionEvent[]> {
    const eventsPath = getEventsFile(sessionId);
    const content = await fs.readFile(eventsPath, 'utf-8');
    return content
        .split('\n')
        .filter(line => line.trim())
        .map(line => JSON.parse(line));
}

// Extract conversation history
function extractConversation(events: SessionEvent[]): Array<{
    role: 'user' | 'assistant';
    content: string;
}> {
    const messages: Array<{ role: 'user' | 'assistant'; content: string }> = [];

    for (const event of events) {
        if (event.type === 'user.message' && typeof event.data.content === 'string') {
            messages.push({ role: 'user', content: event.data.content });
        }
        if (event.type === 'assistant.message' && typeof event.data.content === 'string') {
            messages.push({ role: 'assistant', content: event.data.content });
        }
    }

    return messages;
}
```

### Step 4: Watch for Changes

```typescript
import * as chokidar from 'chokidar';  // or VS Code FileSystemWatcher

function watchSessions(
    onChange: (sessionId: string) => void,
    onCreate: (sessionId: string) => void,
    onDelete: (sessionId: string) => void,
): { dispose(): void } {
    const stateDir = getSessionStateDir();
    const watcher = chokidar.watch(
        path.join(stateDir, '**', 'events.jsonl'),
        { ignoreInitial: true }
    );

    let changeTimeout: NodeJS.Timeout | undefined;

    watcher.on('add', filePath => {
        const sessionId = path.basename(path.dirname(filePath));
        onCreate(sessionId);
    });

    watcher.on('change', filePath => {
        const sessionId = path.basename(path.dirname(filePath));
        // Throttle change events (500ms)
        clearTimeout(changeTimeout);
        changeTimeout = setTimeout(() => onChange(sessionId), 500);
    });

    watcher.on('unlink', filePath => {
        const sessionId = path.basename(path.dirname(filePath));
        onDelete(sessionId);
    });

    return {
        dispose() {
            watcher.close();
            clearTimeout(changeTimeout);
        }
    };
}
```

## Phase 2: SDK Integration (Full CRUD)

### Step 5: Import SDK

```typescript
// Dynamic import (recommended for lazy loading)
async function getSDK() {
    return await import('@github/copilot/sdk');
}

// Key exports you'll need:
// - internal.LocalSessionManager
// - internal.NoopTelemetryService
// - Session
// - SessionOptions
// - SessionEvent
// - SendOptions
// - Attachment
// - SweCustomAgent
```

### Step 6: Initialize Session Manager

```typescript
async function createSessionManager() {
    const { internal } = await getSDK();

    // Configure OTel (optional)
    process.env['COPILOT_OTEL_ENABLED'] = 'true';

    return new internal.LocalSessionManager({
        telemetryService: new internal.NoopTelemetryService(),
        flushDebounceMs: undefined,
        settings: undefined,
        version: undefined,
    });
}
```

**Important:** Keep a single `LocalSessionManager` instance for the entire lifecycle. Use lazy initialization:

```typescript
let _manager: Promise<LocalSessionManager> | undefined;
function getSessionManager() {
    if (!_manager) {
        _manager = createSessionManager();
    }
    return _manager;
}
```

### Step 7: Create Session

```typescript
async function createSession(options: {
    workingDirectory: string;
    model?: string;
    authToken: string;
}) {
    const manager = await getSessionManager();

    const session = await manager.createSession({
        clientName: 'my-app',
        workingDirectory: options.workingDirectory,
        model: options.model,
        enableStreaming: true,
        authInfo: { token: options.authToken },
        sessionCapabilities: new Set([
            'plan-mode', 'memory', 'interactive-mode',
            'cli-documentation', 'ask-user', 'system-notifications',
        ]),
    });

    return session;
}
```

### Step 8: Resume Session

```typescript
async function resumeSession(sessionId: string, authToken: string) {
    const manager = await getSessionManager();

    const session = await manager.getSession({
        sessionId,
        clientName: 'my-app',
        enableStreaming: true,
        authInfo: { token: authToken },
    }, true);  // true = resume

    return session;
}
```

### Step 9: Send Messages and Handle Events

```typescript
async function sendMessage(session: Session, prompt: string) {
    // Register event listeners BEFORE sending
    const disposables: Array<() => void> = [];

    disposables.push(session.on('assistant.message_delta', (event) => {
        // Streaming text
        process.stdout.write(event.data.deltaContent as string);
    }));

    disposables.push(session.on('tool.execution_start', (event) => {
        console.log(`[Tool] ${event.data.toolName}: starting`);
    }));

    disposables.push(session.on('tool.execution_complete', (event) => {
        console.log(`[Tool] complete: ${event.data.success ? 'ok' : 'error'}`);
    }));

    disposables.push(session.on('permission.requested', async (event) => {
        // Auto-approve or prompt user
        session.respondToPermission(event.data.requestId, {
            kind: 'approved',
        });
    }));

    disposables.push(session.on('session.error', (event) => {
        console.error(`[Error] ${event.data.message}`);
    }));

    // Send the message
    try {
        await session.send({
            prompt,
            attachments: [],
            agentMode: 'interactive',  // or 'autopilot' or 'plan'
        });
    } finally {
        // Clean up listeners
        disposables.forEach(dispose => dispose());
    }
}
```

### Step 10: Implement Reference Counting

```typescript
class RefCountedSession {
    private _refCount = 1;
    private _disposed = false;

    constructor(
        public readonly session: Session,
        private readonly _onDispose: () => void,
    ) {}

    acquire(): this {
        if (this._disposed) throw new Error('Session disposed');
        this._refCount++;
        return this;
    }

    release(): void {
        if (--this._refCount === 0) {
            this._disposed = true;
            this._onDispose();
        }
    }
}

// Usage:
const sessions = new Map<string, RefCountedSession>();

function getOrCreateSession(sessionId: string): RefCountedSession {
    const existing = sessions.get(sessionId);
    if (existing) return existing.acquire();

    const ref = new RefCountedSession(session, () => {
        sessions.delete(sessionId);
        // Schedule delayed shutdown
        setTimeout(() => manager.closeSession(sessionId), 300_000);
    });
    sessions.set(sessionId, ref);
    return ref;
}
```

## Phase 3: VS Code Integration

> **Proposed API Requirements:** Phases 3-5 rely on proposed VS Code APIs. Your `package.json` must include:
> ```json
> "enabledApiProposals": [
>     "chatSessionsProvider@3",
>     "chatSessionCustomizationProvider"
> ]
> ```
> These APIs are unstable. Test against VS Code Insiders and pin to a proposal version.

### Architecture: Provider vs. Participant

The VS Code integration splits into two complementary registrations:

```
┌─────────────────────────────┐     ┌──────────────────────────────┐
│  ChatSessionContentProvider │     │      Chat Participant        │
│                             │     │                              │
│  • Lists sessions for UI    │     │  • Handles user requests     │
│  • Provides history/metadata│     │  • Routes to SDK session     │
│  • Session lifecycle events │     │  • Manages streaming output  │
└─────────────────────────────┘     └──────────────────────────────┘
         ▲                                    ▲
         │                                    │
         └──── Same provider ID ──────────────┘
```

The **content provider** is passive (data retrieval); the **participant** is active (request handling). Both register against the same provider ID so VS Code can associate them.

### Step 11: Register Chat Session Provider

> **Warning — Proposed API:** `vscode.chat.createChatSessionItemController` is a proposed API. Your extension must declare `"chatSessionsProvider@3"` in the `enabledApiProposals` array of `package.json`. Proposed APIs are unstable and may change between VS Code releases.

```typescript
// Using VS Code proposed APIs:
const controller = vscode.chat.createChatSessionItemController(
    'my-provider-id',
    async () => {
        // Refresh handler — called when UI needs updated session list
        const sessions = await listSessions();
        controller.items.replace(sessions.map(toSessionItem));
    }
);

// New session handler
controller.newChatSessionItemHandler = async (context) => {
    const sessionId = crypto.randomUUID();
    const resource = vscode.Uri.from({
        scheme: 'my-sessions',
        path: `/${sessionId}`,
    });
    return controller.createChatSessionItem(resource, context.request.prompt ?? '');
};
```

### Step 12: Implement Content Provider

> **Warning — Proposed API:** `vscode.chat.registerChatSessionContentProvider` is a proposed API requiring `"chatSessionsProvider@3"` in `enabledApiProposals`. See Step 11 warning.

```typescript
class MySessionContentProvider implements vscode.ChatSessionContentProvider {
    async provideChatSessionItems(token: vscode.CancellationToken) {
        const sessions = await listSessions();
        return sessions.map(session => ({
            resource: vscode.Uri.from({
                scheme: 'my-sessions',
                path: `/${session.id}`,
            }),
            label: session.summary,
            timing: {
                created: new Date(session.created_at).getTime(),
                startTime: new Date(session.created_at).getTime(),
                endTime: new Date(session.updated_at).getTime(),
            },
        }));
    }
}

// Register:
vscode.chat.registerChatSessionContentProvider(
    'my-provider-id',
    new MySessionContentProvider()
);
```

## Phase 4: MCP Integration

MCP (Model Context Protocol) servers extend the session's tool capabilities. The SDK handles MCP transport, tool discovery, and invocation — the host only needs to supply configuration.

### Step 13: Configure MCP Servers

Two transport types are supported:
- **`http`**: Remote MCP servers accessed via HTTP. Requires URL and optional auth headers.
- **`local`**: Local processes spawned as child processes. Requires command and args.

Tool names from MCP servers are normalized (lowercase, valid characters). Agent tool remapping may apply — the SDK can remap tool names from an agent's `tools` list to the actual MCP tool names.

```typescript
async function createSessionWithMCP(workingDirectory: string, authToken: string) {
    const manager = await getSessionManager();

    const session = await manager.createSession({
        clientName: 'my-app',
        workingDirectory,
        enableStreaming: true,
        authInfo: { token: authToken },
        mcpServers: {
            'github': {
                type: 'http',
                url: 'https://api.github.com/mcp',
                headers: { Authorization: `Bearer ${authToken}` },
                tools: ['*'],
                displayName: 'GitHub',
            },
            'my-local-server': {
                type: 'local',
                command: 'npx',
                args: ['-y', 'my-mcp-server'],
                tools: ['*'],
            },
        },
    });

    return session;
}
```

## Phase 5: IDE Lock File (ACP Integration)

> **Note on Phases 3-5:** These phases depend heavily on VS Code proposed APIs (`chatSessionsProvider@3`, `chatSessionCustomizationProvider`) declared in `enabledApiProposals`. The provider/participant architecture splits responsibilities: the **content provider** supplies session metadata and history to the Chat panel, while the **chat participant** handles incoming requests and routes them through the SDK session wrapper. Both register against the same provider ID. Proposed APIs are subject to change — pin to a specific proposal version (e.g., `@3`) and test against VS Code Insiders builds.

### Step 14: Create Lock File for ACP Connection

```typescript
import * as net from 'node:net';

async function createIDELockFile(options: {
    socketPath: string;
    workspaceFolders: string[];
}): Promise<{ dispose(): void }> {
    const lockId = crypto.randomUUID();
    const nonce = crypto.randomUUID();
    const lockPath = path.join(getCopilotHome(), 'ide', `${lockId}.lock`);

    await fs.mkdir(path.dirname(lockPath), { recursive: true });

    const lockData = {
        socketPath: options.socketPath,
        scheme: 'unix',
        headers: { Authorization: `Nonce ${nonce}` },
        pid: process.pid,
        ideName: 'My IDE',
        timestamp: Date.now(),
        workspaceFolders: options.workspaceFolders,
        isTrusted: true,
    };

    await fs.writeFile(lockPath, JSON.stringify(lockData, null, 2));

    return {
        async dispose() {
            await fs.unlink(lockPath).catch(() => {});
        },
    };
}
```

## Error Handling

The happy-path examples above omit failure modes that production integrations must address.

### SDK Import Failures

The SDK package (`@github/copilot/sdk`) is dynamically imported and requires native shims (`node-pty`, `ripgrep`). Guard the import:

```typescript
async function getSDKSafe() {
    try {
        return await import('@github/copilot/sdk');
    } catch (error) {
        // Native shim missing, package not installed, or version mismatch
        console.error('Failed to load Copilot SDK:', error);
        return undefined;
    }
}
```

### Authentication Failures

- Call `session.setAuthInfo()` before each `send()` if the token may have expired.
- Handle 401/403 from the SDK by refreshing the token via `IAuthenticationService` and retrying once.
- If the user has no active GitHub session (`!authService.anyGitHubSession`), return empty results from model/agent queries rather than throwing.

### Session Corruption

- `events.jsonl` may be truncated (crash during write). Skip unparseable lines rather than rejecting the entire file.
- `workspace.yaml` may be absent for sessions created by older CLI versions. Treat as metadata-less.

### Concurrent Access

- The CLI and VS Code may write to the same session simultaneously. Use the SDK's `LocalSessionManager` as the single writer; never write `events.jsonl` directly.
- For in-memory caches (e.g., `sessionAgents`), read from cache first, fall back to memento, write to both — as shown in `trackSessionAgent`.

### Token Refresh

Auth tokens expire. The production implementation refreshes via `ICopilotCLISDK.getAuthInfo()` before each SDK call that requires authentication. See doc 02 for the `getAuthInfo()` flow.

## Edit Tracking and Checkpoints

The session wrapper maintains an `ExternalEditTracker` per request to coordinate file edits with VS Code's undo/redo system.

```
permission.requested (file edit)
    │
    ├── editTracker.trackEdit(toolCallId, [fileUri], stream)
    │     └── Calls stream.externalEdit() → VS Code begins tracking
    │
    └── tool.execution_complete
          └── editTracker.completeEdit(toolCallId)
                └── Signals VS Code that the edit batch is done
```

A parallel `toolIdEditMap` (`Map<string, Promise<string | undefined>>`) records the mapping from each tool call ID to its VS Code edit group ID. This map is persisted in `IChatSessionMetadataStore` alongside request details, enabling undo-by-tool-call in the Chat panel.

**Checkpoint behavior:** Edit tracking integrates with VS Code's checkpoint system — each completed edit group becomes an undoable checkpoint in the session timeline. The `toolIdEditMap` stored in request metadata allows the UI to map "Undo this tool's changes" to the correct checkpoint.

## Steering and Immediate Mode

When a session is already busy (`InProgress` or `NeedsInput`), incoming messages are treated as *steering* requests rather than new conversation turns.

```typescript
// In handleRequest():
const isAlreadyBusy = this._status === ChatSessionStatus.InProgress
    || this._status === ChatSessionStatus.NeedsInput;

if (isAlreadyBusy) {
    return this._handleRequestSteering(input, attachments, modelId,
        previousRequestSnapshot, token);
}
```

### How Steering Works

1. The steering prompt is sent to the SDK with `mode: 'immediate'` (via `sendRequestInternal`), injecting it into the running conversation as additional user context.
2. The SDK `send()` returns quickly — it enqueues rather than processes.
3. The steering promise awaits *both* the steering send and the original in-flight request (`Promise.all([previousRequestPromise, steeringSend])`), ensuring the caller sees correct session state when the promise settles.
4. The `previousRequest` field is a chained promise that serializes all requests against a session.

### Implementation Notes

- Always snapshot `this.previousRequest` *before* extending the chain, to avoid circular awaits.
- Cancellation during steering aborts the SDK session (`this._sdkSession.abort()`).
- Steering does not create a new OTel span — it piggybacks on the original request's trace.

## Exit Plan Mode Handling

When the SDK operates in plan mode, it emits `exit_plan_mode.requested` events to transition to execution. The handler has two branches depending on the permission level.

### Autopilot Mode (Auto-Approval)

```
exit_plan_mode.requested
    │
    ├── If recommendedAction is in the action list → select it
    ├── Else if 'autopilot' available → select 'autopilot'
    ├── Else if 'autopilot_fleet' available → select 'autopilot_fleet'
    ├── Else if 'interactive' available → select 'interactive'
    ├── Else if 'exit_only' available → select 'exit_only'
    └── Else → approve with autoApproveEdits: true (no specific action)
```

### Interactive Mode (User Confirmation)

When not in autopilot, the handler presents a UI prompt via `IUserQuestionHandler.askUserQuestion`:

```typescript
// Action types from the SDK:
type ActionType = 'autopilot' | 'interactive' | 'exit_only' | 'autopilot_fleet';

// User sees labeled options; freeform text input is also allowed.
// If the user provides freeform text → respondToExitPlanMode({ approved: false, feedback: text })
// If the user selects an action → respondToExitPlanMode({ approved: true, selectedAction })
// If the user cancels → respondToExitPlanMode({ approved: false })
```

**Configuration:** The entire `exit_plan_mode.requested` handler is gated behind `ConfigKey.Advanced.CLIPlanExitModeEnabled`. When disabled, no handler is attached and the SDK falls back to its default behavior.

## Worktree Management

The `IChatSessionWorktreeService` manages Git worktrees for isolated agent sessions. Each session can optionally operate in its own worktree to avoid polluting the user's working directory.

```typescript
export interface IChatSessionWorktreeService {
    createWorktree(repositoryPath: Uri, stream?: ChatResponseStream,
        baseBranch?: string, branchName?: string
    ): Promise<ChatSessionWorktreeProperties | undefined>;

    getWorktreeProperties(sessionId: string): Promise<ChatSessionWorktreeProperties | undefined>;
    getWorktreeProperties(folder: Uri): Promise<ChatSessionWorktreeProperties | undefined>;
    setWorktreeProperties(sessionId: string, properties: string | ChatSessionWorktreeProperties): Promise<void>;
    getWorktreeRepository(sessionId: string): Promise<RepoContext | undefined>;
    getWorktreePath(sessionId: string): Promise<Uri | undefined>;
}
```

The session service uses `worktreeManager.getWorktreeProperties(sessionId)` during session filtering to determine whether a session's worktree belongs to the current workspace. This affects session visibility — a session is shown if its worktree's repository path matches a workspace folder, even when the session's own `workspaceFolder` does not.

## Testing Checklist

### Storage Layer
- [ ] Reads `~/.copilot/session-state/` directory listing
- [ ] Parses `workspace.yaml` YAML format
- [ ] Parses `events.jsonl` line-delimited JSON
- [ ] Handles missing or corrupted files gracefully
- [ ] Respects `XDG_STATE_HOME` environment variable

### Session Lifecycle
- [ ] Creates new session via SDK
- [ ] Resumes existing session by ID
- [ ] Handles concurrent access (mutex/locking)
- [ ] Cleans up sessions on dispose
- [ ] Reference counting works correctly (acquire/release)
- [ ] Delayed shutdown after last reference released

### Event Processing
- [ ] Receives `assistant.message_delta` streaming events
- [ ] Handles `tool.execution_start/complete` pairs
- [ ] Responds to `permission.requested` events
- [ ] Handles `session.error` gracefully
- [ ] Supports steering (`mode: 'immediate'`) while session is busy
- [ ] Steering promise resolves only after original request completes

### Exit Plan Mode
- [ ] Auto-approves in autopilot mode with correct action selection cascade
- [ ] Presents UI prompt in interactive mode
- [ ] Handles freeform feedback (approved: false + feedback)
- [ ] Respects `CLIPlanExitModeEnabled` configuration gate

### Edit Tracking
- [ ] `ExternalEditTracker.trackEdit()` called before file edit proceeds
- [ ] `completeEdit()` called on `tool.execution_complete`
- [ ] `toolIdEditMap` persisted in request metadata
- [ ] Undo-by-tool-call maps to correct checkpoint

### Worktree Management
- [ ] Sessions with worktrees visible when repo matches workspace folder
- [ ] `getWorktreeProperties` returns correct worktree for session ID

### Filesystem Monitoring
- [ ] Detects new sessions created by CLI
- [ ] Detects session deletions
- [ ] Detects session changes (new events)
- [ ] Throttles change events (500ms)
- [ ] Filters sessions by workspace folder

### MCP Integration
- [ ] Passes MCP server config to SDK
- [ ] Tool name normalization (lowercase, valid chars)
- [ ] Agent tool remapping works
- [ ] Built-in GitHub server authentication

### VS Code Integration (if applicable)
- [ ] Session list shows in Chat panel
- [ ] Creating new session works
- [ ] Resuming existing session works
- [ ] Session forking preserves history
- [ ] Session deletion removes from UI
- [ ] Status badges update correctly
- [ ] Terminal integration resolves file paths

## Common Pitfalls

1. **Single LocalSessionManager** — Never create multiple instances. Use lazy singleton.
2. **OTel environment** — Set `COPILOT_OTEL_ENABLED=true` before first SDK call or spans are lost.
3. **Event listeners before send** — Always register listeners before calling `session.send()`.
4. **Steering vs. new turn** — Check `session.status` before sending; use `mode: 'immediate'` if busy.
5. **File watcher throttling** — The CLI writes events rapidly; throttle to 500ms minimum.
6. **SDK session closure** — Always close sessions via `sessionManager.closeSession()` — don't rely on GC.
7. **Auth token refresh** — Call `session.setAuthInfo()` before each send if token may have expired.
8. **Native shims** — SDK requires `node-pty` and `ripgrep` binaries; ensure they're available.
