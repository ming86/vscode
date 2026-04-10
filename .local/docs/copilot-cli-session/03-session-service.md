# Session Service — ICopilotCLISessionService

## Overview

`ICopilotCLISessionService` is the central service for managing Copilot CLI session lifecycle within the VS Code extension. It lazily initializes the SDK's `LocalSessionManager`, provides ref-counted session wrappers, monitors the filesystem for cross-process changes, and implements session CRUD operations.

**Source:** `src/extension/chatSessions/copilotcli/node/copilotcliSessionService.ts`

## Service Interface

```typescript
export const ICopilotCLISessionService =
    createServiceIdentifier<ICopilotCLISessionService>('ICopilotCLISessionService');

export interface ICopilotCLISessionService {
    readonly _serviceBrand: undefined;

    // Events
    onDidChangeSessions: Event<void>;
    onDidDeleteSession: Event<string>;
    onDidChangeSession: Event<ICopilotCLISessionItem>;
    onDidCreateSession: Event<ICopilotCLISessionItem>;

    // Session metadata
    getSessionWorkingDirectory(sessionId: string): Uri | undefined;
    getSessionItem(sessionId: string, token: CancellationToken): Promise<ICopilotCLISessionItem | undefined>;
    getAllSessions(token: CancellationToken): Promise<readonly ICopilotCLISessionItem[]>;

    // Session ID management
    createNewSessionId(): string;
    isNewSessionId(sessionId: string): boolean;

    // Session CRUD
    deleteSession(sessionId: string): Promise<void>;
    renameSession(sessionId: string, title: string): Promise<void>;
    createSession(options: ICreateSessionOptions, token: CancellationToken): Promise<IReference<ICopilotCLISession>>;
    getSession(options: IGetSessionOptions, token: CancellationToken): Promise<IReference<ICopilotCLISession> | undefined>;
    forkSession(options: { sessionId: string; requestId: string | undefined; workspace: IWorkspaceInfo }, token: CancellationToken): Promise<string>;

    // History
    getChatHistory(options: { sessionId: string; workspace: IWorkspaceInfo }, token: CancellationToken): Promise<(ChatRequestTurn2 | ChatResponseTurn2)[]>;
    tryGetPartialSesionHistory(sessionId: string): Promise<readonly (ChatRequestTurn2 | ChatResponseTurn2)[] | undefined>;
}
```

## Session Item Type

```typescript
export interface ICopilotCLISessionItem {
    readonly id: string;                          // Session UUID
    readonly label: string;                       // Display title
    readonly timing: ChatSessionItem['timing'];   // { created, startTime, endTime? }
    readonly status?: ChatSessionStatus;          // Idle | InProgress | NeedsInput
    readonly workingDirectory?: Uri;              // cwd from workspace.yaml
}
```

## Session Options Types

```typescript
export type ISessionOptions = {
    model?: string;
    workspace: IWorkspaceInfo;
    agent?: SweCustomAgent;
    debugTargetSessionIds?: readonly string[];
    mcpServerMappings?: McpServerMappings;
    additionalWorkspaces?: IWorkspaceInfo[];
};

export type IGetSessionOptions = ISessionOptions & { sessionId: string };
export type ICreateSessionOptions = ISessionOptions & { sessionId?: string };
```

## Constructor Dependencies

```typescript
class CopilotCLISessionService extends Disposable implements ICopilotCLISessionService {
    constructor(
        @ILogService logService,
        @ICopilotCLISDK copilotCLISDK,          // SDK bridge
        @IInstantiationService instantiationService,
        @INativeEnvService nativeEnv,            // userHome path
        @IFileSystemService fileSystem,          // File watcher
        @ICopilotCLIMCPHandler mcpHandler,       // MCP config
        @ICopilotCLIAgents agents,               // Agent discovery
        @IWorkspaceService workspaceService,     // Workspace folders
        @ICustomSessionTitleService customSessionTitleService,
        @IConfigurationService configurationService,
        @ICopilotCLISkills copilotCLISkills,
        @IChatDelegationSummaryService delegationSummaryService,
        @IChatSessionMetadataStore chatSessionMetadataStore,
        @IAgentSessionsWorkspace agentSessionsWorkspace,
        @IChatSessionWorkspaceFolderService workspaceFolderService,
        @IChatSessionWorktreeService worktreeManager,
        @IOTelService otelService,
        @IPromptVariablesService promptVariablesService,
        @IChatDebugFileLoggerService debugFileLogger,
        @IChatPromptFileService chatPromptFileService,
    )
}
```

## LocalSessionManager Initialization

The SDK `LocalSessionManager` is lazily initialized on first use:

```typescript
private _sessionManager: Lazy<Promise<internal.LocalSessionManager>>;

// In constructor:
this._sessionManager = new Lazy<Promise<internal.LocalSessionManager>>(async () => {
    const { internal } = await this.getSDKPackage();

    // Configure OTel environment variables for debug panel
    if (!process.env['COPILOT_OTEL_ENABLED']) {
        process.env['COPILOT_OTEL_ENABLED'] = 'true';
    }
    if (!process.env['OTEL_INSTRUMENTATION_GENAI_CAPTURE_MESSAGE_CONTENT']) {
        process.env['OTEL_INSTRUMENTATION_GENAI_CAPTURE_MESSAGE_CONTENT'] = 'true';
    }

    if (this._otelService.config.enabled) {
        const otelEnv = deriveCopilotCliOTelEnv(this._otelService.config);
        for (const [key, value] of Object.entries(otelEnv)) {
            process.env[key] = value;
        }
        process.env['OTEL_INSTRUMENTATION_GENAI_CAPTURE_MESSAGE_CONTENT'] =
            String(this._otelService.config.captureContent);
    } else {
        // Export to /dev/null so SDK creates OtelSessionTracker for debug panel
        process.env['COPILOT_OTEL_EXPORTER_TYPE'] = 'file';
        process.env['COPILOT_OTEL_FILE_EXPORTER_PATH'] = devNull;
    }

    return new internal.LocalSessionManager({
        telemetryService: new internal.NoopTelemetryService(),
        flushDebounceMs: undefined,
        settings: undefined,
        version: undefined,
    });
});
```

**Key pattern:** The `Lazy<T>` wrapper ensures initialization runs once, regardless of how many concurrent callers request the session manager.

## Session Create Flow

```
createSession(options, token)
    │
    ├── mcpHandler.loadMcpConfig()           → MCP server definitions
    ├── createSessionsOptions(...)           → SessionOptions + agentName
    │   ├── agents.getAgents()               → Custom agents
    │   ├── copilotCLISkills.getSkillsLocations() → Skill directories
    │   └── promptVariablesService.buildTemplateVariablesContext()
    │
    ├── getSessionManager()                  → LocalSessionManager (lazy)
    ├── sessionManager.createSession(opts)   → SDK Session
    ├── _installBridgeIfNeeded()             → OTel bridge for debug panel
    └── createCopilotSession(sdkSession)     → RefCountedSession
```

### SessionOptions Construction

Options are built incrementally with conditionals — only `clientName` is unconditional:

```typescript
// Simplified; structure faithful to source. See createSessionsOptions() for full conditionals.
const allOptions: SessionOptions = {
    clientName: 'vscode',
};

if (workingDirectory) {
    allOptions.workingDirectory = workingDirectory.fsPath;
}
if (options.model) {
    allOptions.model = options.model;
}
if (mcpServers && Object.keys(mcpServers).length > 0) {
    allOptions.mcpServers = mcpServers;
}
if (skillLocations.length > 0) {
    allOptions.skillDirectories = skillLocations.map(uri => uri.fsPath);
}
if (options.agent) {
    allOptions.selectedCustomAgent = options.agent;
}
if (customAgents.length > 0) {
    allOptions.customAgents = customAgents;
}
allOptions.enableStreaming = true;
if (options.copilotUrl) {
    allOptions.copilotUrl = options.copilotUrl;  // SDK endpoint override
}
if (systemMessage) {
    allOptions.systemMessage = systemMessage;    // { mode: 'append', content: ... }
}
allOptions.sessionCapabilities = new Set([
    'plan-mode', 'memory', 'cli-documentation',
    'ask-user', 'interactive-mode', 'system-notifications'
]);
```

## Session Resume Flow (getSession)

```
getSession({ sessionId, ... }, token)
    │
    ├── Acquire per-session Mutex             → Prevent concurrent getSession
    ├── Check _sessionWrappers.get(sessionId) → Return existing ref if cached
    │   ├── session.acquire()                 → Increment ref count
    │   └── session.selectCustomAgent(agent)  → Update agent if needed
    │
    ├── [On cache miss]
    │   ├── getSessionManager()
    │   ├── mcpHandler.loadMcpConfig()
    │   ├── createSessionsOptions(...)
    │   ├── sessionManager.getSession(opts, true)  → Resume from disk
    │   └── createCopilotSession(sdkSession)       → Wrap + ref count
    │
    └── Release Mutex
```

### Mutex Pattern

```typescript
private sessionMutexForGetSession = new Map<string, Mutex>();

public async getSession(opts, token) {
    const lock = this.sessionMutexForGetSession.get(sessionId) ?? new Mutex();
    this.sessionMutexForGetSession.set(sessionId, lock);
    const lockDisposable = await lock.acquire(token);
    try {
        // ... session logic
    } finally {
        lockDisposable?.dispose();
    }
}
```

**Purpose:** Prevents race conditions when multiple VS Code components request the same session simultaneously.

## Reference Counting

```typescript
private _sessionWrappers = new DisposableMap<string, RefCountedSession>();

// Source: line 1242 of copilotcliSessionService.ts
export class RefCountedSession extends RefCountedDisposable implements IReference<CopilotCLISession> { ... }
// Concrete class — not a type alias. Generic parameter is CopilotCLISession (the class), not ICopilotCLISession.
```

### createCopilotSession — Session Wiring

```typescript
private createCopilotSession(sdkSession, workspaceInfo, agentName, sessionManager): RefCountedSession {
    const session = this.instantiationService.createInstance(
        CopilotCLISession, workspaceInfo, agentName, sdkSession, []
    );

    // Debug file logger lifecycle
    this._debugFileLogger.startSession(session.sessionId);
    session.add(toDisposable(() => {
        this._debugFileLogger.endSession(session.sessionId);
    }));

    // Wire OTel bridge + SDK trace context propagation
    session.setBridgeProcessor(this._bridgeProcessor);
    if (sessionManager.otel) {
        session.setSdkTraceContextUpdater((traceparent, tracestate) =>
            sessionManager.otel.updateParentTraceContext(sdkSession.sessionId, traceparent, tracestate));
    }

    // Status change → update session item + fire change event
    session.add(session.onDidChangeStatus(() => {
        this.triggerOnDidChangeSessionItem(sdkSession.sessionId, 'statusChange');
        this._onDidChangeSessions.fire();
    }));

    // Dispose handler: cleanup on session disposal
    session.add(toDisposable(() => {
        this._sessionWrappers.deleteAndLeak(sdkSession.sessionId);
        this.sessionMutexForGetSession.delete(sdkSession.sessionId);
        // Async: abort if running, then close via SDK
        (async () => {
            if (sdkSession.isAbortable()) {
                await sdkSession.abort();
            }
            await sessionManager.closeSession(sdkSession.sessionId);
        })();
    }));

    // Idle timeout: dispose session after inactivity (see lifecycle below)
    session.add(session.onDidChangeStatus(e => {
        if (session.permissionRequested) {
            this.sessionTerminators.deleteAndDispose(session.sessionId);
        } else if (session.status === undefined
            || session.status === ChatSessionStatus.Completed
            || session.status === ChatSessionStatus.Failed) {
            this.sessionTerminators.set(session.sessionId, disposableTimeout(() => {
                session.dispose();
                this.sessionTerminators.deleteAndDispose(session.sessionId);
            }, SESSION_SHUTDOWN_TIMEOUT_MS));
        } else {
            // Session is active — cancel any pending termination
            this.sessionTerminators.deleteAndDispose(session.sessionId);
        }
    }));

    const refCountedSession = new RefCountedSession(session);
    this._sessionWrappers.set(sdkSession.sessionId, refCountedSession);
    return refCountedSession;
}
```

> `RefCountedSession` extends `RefCountedDisposable` — `dispose()` calls `release()`, decrementing the ref count. When the count reaches zero, the underlying `CopilotCLISession` is disposed, which fires the `toDisposable` handlers registered above.

**Session Lifecycle (actual order):**

1. `createSession()` / `getSession()` returns `IReference<ICopilotCLISession>` (a `RefCountedSession`)
2. Caller holds reference; calls `ref.acquire()` for shared access
3. When session status changes to `Completed`, `Failed`, or `undefined` — a `disposableTimeout` (5 minutes) is armed
4. If a new request arrives before the timer fires, the timer is cancelled (status becomes active)
5. If the timer fires, it calls `session.dispose()` — which triggers the registered dispose handlers:
   - `deleteAndLeak` removes from `_sessionWrappers` (without disposing the wrapper itself, to avoid re-entry)
   - Mutex for the session is deleted
   - `sdkSession.abort()` is called if the session is still abortable
   - `sessionManager.closeSession()` flushes and closes the SDK session
6. If a permission prompt is pending, the timer is cancelled to avoid disposing mid-interaction

## Filesystem Monitoring

```typescript
protected monitorSessionFiles() {
    const sessionDir = joinPath(
        this.nativeEnv.userHome, '.copilot', 'session-state'
    );
    const watcher = this.fileSystem.createFileSystemWatcher(
        new RelativePattern(sessionDir, '**/*.jsonl')
    );

    watcher.onDidCreate(async (e) => {
        const sessionId = extractSessionIdFromEventPath(sessionDir, e);
        if (sessionId && this._sessionsBeingCreatedViaFork.has(sessionId)) {
            return;  // Suppress events during fork
        }
        this.triggerSessionsChangeEvent();
        const sessionItem = sessionId
            ? await this.getSessionItemImpl(sessionId, 'disk', CancellationToken.None)
            : undefined;
        if (sessionItem) {
            this._onDidChangeSession.fire(sessionItem);
        }
    });

    watcher.onDidDelete(e => {
        const sessionId = extractSessionIdFromEventPath(sessionDir, e);
        if (sessionId) {
            this._cachedSessionItems.delete(sessionId);
            this._onDidDeleteSession.fire(sessionId);
        }
        this.triggerSessionsChangeEvent();
    });

    watcher.onDidChange((e) => {
        // Guard: suppress during bulk session fetch
        if (this._isGettingSessions > 0) {
            return;
        }

        const sessionId = extractSessionIdFromEventPath(sessionDir, e);
        // Guard: suppress during fork (files are being copied)
        if (sessionId && this._sessionsBeingCreatedViaFork.has(sessionId)) {
            return;
        }
        // Skip if already aware of this session
        if (Array.from(this._sessionWrappers.keys())
            .some(sessionId => e.path.includes(sessionId))) {
            return;
        }
        if (sessionId) {
            this.triggerOnDidChangeSessionItem(sessionId, 'fileSystemChange');
        }
        this.triggerSessionsChangeEvent();
    });
}
```

### Throttled Change Events

```typescript
private readonly _onDidChangeSessionsThrottler =
    this._register(new ThrottledDelayer<void>(500));

private triggerSessionsChangeEvent() {
    if (this._isGettingSessions > 0) return;
    this._onDidChangeSessionsThrottler.trigger(
        () => Promise.resolve(this._onDidChangeSessions.fire())
    );
}
```

**Purpose:** Prevents UI thrashing when the CLI rapidly updates events.jsonl. Changes within 500ms are coalesced into a single event.

## Session Forking

```
forkSession({ sessionId, requestId, workspace }, token)
    │
    ├── Generate new UUID
    ├── Copy session files to new directory
    │   (events.jsonl, workspace.yaml, metadata)
    ├── sessionManager.getSession(newSessionId)
    ├── If requestId specified:
    │   ├── session.truncateToEvent(eventId)
    │   └── Rewrite events.jsonl
    ├── sessionManager.closeSession(newSessionId)
    ├── Set custom title ("Forked: {original}")
    └── Fire onDidCreateSession
```

## Chat History Reconstruction

```typescript
public async getChatHistory({ sessionId, workspace }, token) {
    const sessionManager = await this.getSessionManager();

    // Try existing wrapper first, then open temporarily
    const existingSession = this._sessionWrappers.get(sessionId)?.object?.sdkSession;
    if (existingSession) {
        events = existingSession.getEvents();
    } else {
        const session = await sessionManager.getSession({ sessionId }, false);
        events = session.getEvents();
        await sessionManager.closeSession(sessionId);  // Close immediately
    }

    return buildChatHistoryFromEvents(sessionId, modelId, events, ...);
}
```

The `buildChatHistoryFromEvents()` function (in `copilotCLITools.ts`) converts raw `SessionEvent[]` into VS Code chat turns (`ChatRequestTurn2 | ChatResponseTurn2`).

## Session Visibility Filtering

Sessions are filtered per-workspace so users only see relevant sessions:

```typescript
private async shouldShowSession(sessionId, context): Promise<boolean> {
    // Untitled sessions (not yet persisted) are always visible
    if (isUntitledSessionId(sessionId)) return true;

    // Always show in empty workspace or agent sessions workspace
    if (this.workspaceService.getWorkspaceFolders().length === 0) return true;
    if (this._agentSessionsWorkspace.isAgentSessionsWorkspace) return true;

    // Session tracker: was this session started from the current workspace?
    const sessionTrackerVisibility = this._sessionTracker.shouldShowSession(sessionId);
    if (sessionTrackerVisibility.isWorkspaceSession) return true;

    // Check if session's cwd or gitRoot matches a workspace folder
    if (context && (
        (context.cwd && this.workspaceService.getWorkspaceFolder(URI.file(context.cwd))) ||
        (context.gitRoot && this.workspaceService.getWorkspaceFolder(URI.file(context.gitRoot)))
    )) return true;

    // Check workspace folder service
    const workspaceFolder = await this.workspaceFolderService
        .getSessionWorkspaceFolder(sessionId);
    if (workspaceFolder && this.workspaceService.getWorkspaceFolder(workspaceFolder))
        return true;

    // Check git worktree
    const worktree = await this.worktreeManager.getWorktreeProperties(sessionId);
    if (worktree && this.workspaceService.getWorkspaceFolder(
        URI.file(worktree.repositoryPath)))
        return true;

    // Old global session fallback: show if no specific workspace/worktree data excludes it
    if (sessionTrackerVisibility.isOldGlobalSession
        && !workspaceFolder && !worktree
        && (this.workspaceService.getWorkspaceFolders().length === 0
            || this._agentSessionsWorkspace.isAgentSessionsWorkspace))
        return true;

    return false;
}
```

> `_sessionTracker` is an instance of `CopilotCLISessionWorkspaceTracker` (see below). `isUntitledSessionId()` returns `true` for sessions that have not yet been persisted to disk.

## CopilotCLISessionWorkspaceTracker

Internal helper class that tracks which sessions belong to the current workspace. Used by `shouldShowSession()` to determine visibility.

```typescript
export class CopilotCLISessionWorkspaceTracker {
    private _oldGlobalSessions?: Set<string>;
    private readonly _workspaceSessions = new Set<string>();

    constructor(
        @IFileSystemService fileSystem,
        @IVSCodeExtensionContext context,       // globalStorageUri, workspaceState
        @IWorkspaceService workspaceService,
    )
}
```

**Initialization (lazy):** On first access, loads two JSON files from `globalStorageUri`:

1. **Global sessions file** (`copilot.cli.oldGlobalSessions.json`) — sessions created before per-workspace tracking was introduced. Populated into `_oldGlobalSessions`.
2. **Workspace sessions file** (`copilot.cli.workspaceSessions.<uuid>.json`) — sessions created in the current workspace. UUID is stored in `workspaceState` and generated once per workspace. Populated into `_workspaceSessions`.

If no workspace folders are open, both files point to the same location (global context).

**`shouldShowSession(sessionId)`** returns `{ isOldGlobalSession?: boolean; isWorkspaceSession?: boolean }` — the caller (`shouldShowSession` in the session service) uses these flags in its priority chain to determine whether a session is visible in the current workspace.
