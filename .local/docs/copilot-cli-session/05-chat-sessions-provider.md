# VS Code Chat Sessions Provider

## Overview

`CopilotCLIChatSessionContentProvider` integrates the Copilot CLI session system with VS Code's proposed chat session APIs. It registers sessions in the VS Code UI, handles session creation, forking, deletion, and content provision. It also manages repository/worktree/isolation dropdowns for session configuration.

The actual chat request handling — model resolution, prompt construction, SDK session lifecycle, worktree commits, PR detection, delegation, and steering coordination — resides in a separate class, `CopilotCLIChatSessionParticipant`.

**Source:** `src/extension/chatSessions/vscode-node/copilotCLIChatSessions.ts`

## URI Scheme

All CLI sessions use the `copilotcli:` URI scheme:

```typescript
namespace SessionIdForCLI {
    export function getResource(sessionId: string): vscode.Uri {
        return vscode.Uri.from({
            scheme: 'copilotcli', path: `/${sessionId}`,
        });
    }

    export function parse(resource: vscode.Uri): string {
        return resource.path.slice(1);  // Strip leading '/'
    }

    export function isCLIResource(resource: vscode.Uri): boolean {
        return resource.scheme === 'copilotcli';
    }
}
```

**Examples:**
- `copilotcli:///2300305f-9dc5-4860-9717-de0fc3813cfc`
- `copilotcli:///a1b2c3d4-e5f6-7890-abcd-ef1234567890`

## VS Code Proposed APIs Used

> ⚠️ **Proposed API** — All `vscode.chat.*` session APIs listed below are proposed VS Code APIs, subject to change or removal without notice.

| API | Purpose |
|-----|---------|
| ⚠️ `vscode.chat.registerChatSessionContentProvider()` | Registers the content provider for a URI scheme |
| ⚠️ `vscode.chat.createChatSessionItemController()` | Creates controller for managing session list items |
| ⚠️ `vscode.chat.createChatParticipant()` | Creates the chat participant that handles requests |
| `controller.items.replace(items)` | Bulk-replace session list |
| `controller.items.add(item)` | Add/update single session |
| `controller.items.delete(resource)` | Remove session from list |
| `controller.createChatSessionItem(resource, label)` | Factory for new session items |
| `controller.createChatSessionInputState(groups)` | Factory for dropdown state |
| `controller.newChatSessionItemHandler` | Handler when user creates new chat |
| `controller.forkHandler` | Handler for forking sessions |
| `controller.getChatSessionInputState` | Provider for session input dropdowns |
| `controller.onDidChangeChatSessionItemState` | Handler for archive/unarchive |

## CopilotCLIChatSessionContentProvider

### Constructor

```typescript
class CopilotCLIChatSessionContentProvider extends Disposable
    implements vscode.ChatSessionContentProvider, ICopilotCLIChatSessionItemProvider {

    constructor(
        @ICopilotCLISessionService sessionService,
        @IChatSessionMetadataStore chatSessionMetadataStore,
        @IChatSessionWorktreeService copilotCLIWorktreeManagerService,
        @IWorkspaceService workspaceService,
        @IGitService gitService,
        @IFolderRepositoryManager folderRepositoryManager,
        @IConfigurationService configurationService,
        @ICustomSessionTitleService customSessionTitleService,
        @IVSCodeExtensionContext context,
        @ICopilotCLISessionTracker sessionTracker,
        @ICopilotCLITerminalIntegration terminalIntegration,
        @IRunCommandExecutionService commandExecutionService,
        @IChatSessionWorkspaceFolderService workspaceFolderService,
        @IOctoKitService octoKitService,
        @ILogService logService,
        @IAgentSessionsWorkspace agentSessionsWorkspace,
        @ICopilotCLIFolderMruService copilotCLIFolderMruService,
    )
}
```

### Session Item Controller

```typescript
// ⚠️ Proposed API
const controller = vscode.chat.createChatSessionItemController(
    'copilotcli',       // Provider ID — the URI scheme for CLI sessions
    async () => {       // Refresh callback
        const sessions = await this.sessionService.getAllSessions(CancellationToken.None);
        const items = await Promise.all(sessions.map(session => this.toChatSessionItem(session)));
        controller.items.replace(items);

        // Set context for empty-state UI
        void this.commandExecutionService.executeCommand('setContext',
            'github.copilot.chat.cliSessionsEmpty', items.length === 0);
    }
);
```

### New Session Handler

```typescript
controller.newChatSessionItemHandler = async (context) => {
    const sessionId = this.sessionService.createNewSessionId();
    const resource = SessionIdForCLI.getResource(sessionId);
    const session = controller.createChatSessionItem(
        resource,
        context.request.prompt ?? context.request.command ?? ''
    );

    // Generate title asynchronously — uses CancellationToken.None (fire-and-forget)
    this.customSessionTitleService.generateSessionTitle(sessionId, context.request, CancellationToken.None)
        .then(() => {
            if (this.controller.items.get(resource)) {
                this.refreshSession({ reason: 'update', sessionId });
            }
        })
        .catch(ex => this.logService.error(ex, 'Failed to generate custom session title'));

    controller.items.add(session);
    this.newSessions.set(resource, session);
    return session;
};
```

### Fork Handler

The fork handler is only assigned when the `ConfigKey.Advanced.CLIForkSessionsEnabled` feature flag is enabled:

```typescript
if (this.configurationService.getConfig(ConfigKey.Advanced.CLIForkSessionsEnabled)) {
    controller.forkHandler = async (sessionResource, request, token) => {
        const sessionId = SessionIdForCLI.parse(sessionResource);
        const folderInfo = await this.folderRepositoryManager
            .getFolderRepository(sessionId, undefined, token);
        const forkedSessionId = await this.sessionService.forkSession({
            sessionId,
            requestId: request?.id,
            workspace: folderInfo,
        }, token);
        const item = await this.sessionService.getSessionItem(forkedSessionId, token);
        if (!item) {
            throw new Error(`Failed to get session item for forked session ${forkedSessionId}`);
        }
        return this.toChatSessionItem(item);
    };
}
```

### Event Wiring

```typescript
// Session deleted → remove from controller
this.sessionService.onDidDeleteSession(async (sessionId) => {
    controller.items.delete(SessionIdForCLI.getResource(sessionId));
});

// Session changed → update in controller
this.sessionService.onDidChangeSession(async (session) => {
    const item = await this.toChatSessionItem(session);
    controller.items.add(item);
});

// Session created (from filesystem watcher) → add to controller
this.sessionService.onDidCreateSession(async (session) => {
    const resource = SessionIdForCLI.getResource(session.id);
    if (!controller.items.get(resource)) {
        const item = await this.toChatSessionItem(session);
        controller.items.add(item);
    }
});

// Archive state change → worktree cleanup/recreation
controller.onDidChangeChatSessionItemState(async (item) => {
    const sessionId = SessionIdForCLI.parse(item.resource);
    if (item.archived) {
        await this.copilotCLIWorktreeManagerService.cleanupWorktreeOnArchive(sessionId);
    } else {
        await this.copilotCLIWorktreeManagerService.recreateWorktreeOnUnarchive(sessionId);
    }
});
```

### Session Item Conversion (`toChatSessionItem`)

Converts an `ICopilotCLISessionItem` into a `vscode.ChatSessionItem` for the controller. The method covers substantially more than basic label/timing mapping:

```typescript
async toChatSessionItem(session: ICopilotCLISessionItem): Promise<vscode.ChatSessionItem> {
    const resource = SessionIdForCLI.getResource(session.id);
    const worktreeProperties = await this.copilotCLIWorktreeManagerService
        .getWorktreeProperties(session.id);
    const workingDirectory = worktreeProperties?.worktreePath
        ? vscode.Uri.file(worktreeProperties.worktreePath)
        : session.workingDirectory;

    // --- Badge (with workspace trust) ---
    // For worktree sessions: shows repo name with $(repo) or $(workspace-untrusted)
    // For workspace sessions: shows folder name with $(folder) or $(workspace-untrusted)
    // Trust is checked via vscode.workspace.isResourceTrusted()

    // --- Changes / Statistics ---
    // Only returned for trusted workspace/worktree folders.
    // Worktree: copilotCLIWorktreeManagerService.getWorktreeChanges(session.id)
    // Workspace: workspaceFolderService.getWorkspaceChanges(session.id)
    // Each change is a ChatSessionChangedFile2 with original/modified URIs and +/- stats.

    // --- Metadata ---
    // Worktree path: metadata includes autoCommit, baseCommit, baseBranchName,
    //   baseBranchProtected, branchName, isolationMode (Worktree), repositoryPath,
    //   worktreePath, pullRequestUrl, pullRequestState, firstCheckpointRef,
    //   baseCheckpointRef, lastCheckpointRef (v2 properties gated on version check)
    // Workspace path: metadata includes isolationMode (Workspace), repositoryPath,
    //   branchName, baseBranchName, workingDirectoryPath, firstCheckpointRef,
    //   lastCheckpointRef (derived from chatSessionMetadataStore.getRequestDetails)

    const item = this.controller.createChatSessionItem(resource, label);
    item.badge = badge;
    item.timing = session.timing;
    item.changes = changes;
    item.status = session.status ?? vscode.ChatSessionStatus.Completed;
    item.metadata = metadata;
    return item;
}
```

## Content Provider Protocol

The `ChatSessionContentProvider` interface (⚠️ Proposed API) requires:

```typescript
interface ChatSessionContentProvider {
    provideChatSessionContent(resource: Uri, token: CancellationToken, context: {
        readonly inputState: ChatSessionInputState;
        readonly sessionOptions: ReadonlyArray<{ optionId: string; value: string | ChatSessionProviderOptionItem }>;
    }): Thenable<ChatSession> | ChatSession;

    // Deprecated optional members:
    onDidChangeChatSessionOptions?: Event<ChatSessionOptionChangeEvent>;
    onDidChangeChatSessionProviderOptions?: Event<void>;
    provideHandleOptionsChange?(resource, updates, token): void;
    provideChatSessionProviderOptions?(token): Thenable<ChatSessionProviderOptions>;
}
```

There is no `resolveSessionRequest` on this interface. Request handling is the responsibility of the `CopilotCLIChatSessionParticipant`, registered as a separate `ChatParticipant` (see below).

### `provideChatSessionContent`

The content provider's main method resolves a session URI into a `ChatSession` object (history + metadata). It handles three cases:

```
provideChatSessionContent(resource, token, context)
    │
    ├── Parse sessionId from resource URI
    │
    ├── Is untitled session? (id starts with 'untitled:' or 'untitled-')
    │   └── Return { history: [], requestHandler: undefined }
    │
    ├── Is new session? (sessionService.isNewSessionId)
    │   ├── Look up in this.newSessions map
    │   ├── Throw if not found
    │   └── Return { history: [], title: session.label, ... }
    │
    └── Is existing session?
        └── provideChatSessionContentForExistingSession(resource, token)
            ├── Fire-and-forget: detectPullRequestOnSessionOpen(sessionId)
            ├── Resolve folder repository
            ├── Parallel fetch: [getSessionHistory, getCustomSessionTitle]
            └── Return { title, history, ... }
```

Performance is logged via `StopWatch` on every invocation.

## CopilotCLIChatSessionParticipant

This class (~750 lines) is the chat request handler. It is registered as a `ChatParticipant` and receives all user messages sent to CLI sessions. The content provider handles session *display*; this class handles session *execution*.

### Constructor and Dependencies

```typescript
class CopilotCLIChatSessionParticipant extends Disposable {
    constructor(
        private readonly contentProvider: CopilotCLIChatSessionContentProvider,
        private readonly promptResolver: CopilotCLIPromptResolver,
        private readonly cloudSessionProvider: CopilotCloudSessionsProvider | undefined,
        private readonly branchNameGenerator: GitBranchNameGenerator,
        @IGitService gitService,
        @ICopilotCLIModels copilotCLIModels,
        @ICopilotCLIAgents copilotCLIAgents,
        @ICopilotCLISessionService sessionService,
        @IChatSessionWorktreeService copilotCLIWorktreeManagerService,
        @IChatSessionWorktreeCheckpointService copilotCLIWorktreeCheckpointService,
        @IChatSessionWorkspaceFolderService workspaceFolderService,
        @ITelemetryService telemetryService,
        @ILogService logService,
        @IPromptsService promptsService,
        @IChatDelegationSummaryService chatDelegationSummaryService,
        @IFolderRepositoryManager folderRepositoryManager,
        @IConfigurationService configurationService,
        @ICopilotCLISDK copilotCLISDK,
        @IChatSessionMetadataStore chatSessionMetadataStore,
        @IOctoKitService octoKitService,
        @IWorkspaceService workspaceService,
    )
}
```

Four explicit dependencies are injected positionally (content provider, prompt resolver, cloud session provider, branch name generator); the remainder are DI-decorated services.

### Registration

The participant is wired into VS Code via:

```typescript
const participant = vscode.chat.createChatParticipant(
    'copilotcli',                               // same scheme as the content provider
    copilotcliChatSessionParticipant.createHandler()  // binds handleRequest
);
// ⚠️ Proposed API
vscode.chat.registerChatSessionContentProvider('copilotcli', contentProvider, participant);
```

The `createHandler()` method returns `this.handleRequest.bind(this)`.

### Request Flow

```
handleRequest(request, context, stream, token)     ← outer handler (steering-aware)
    │
    ├── Start handleRequestImpl(request, context, stream, token) as a promise
    ├── Poll context.yieldRequested every 100ms
    ├── Promise.race([yieldSignal, handleRequestImpl])
    │   └── If yield wins: returns control to VS Code for steering request
    │       (handleRequestImpl continues running in background)
    │
    └── handleRequestImpl(request, context, stream, token)
        │
        ├── Telemetry event: 'copilotcli.chat.invoke'
        ├── Authenticate via copilotCLISDK.getAuthInfo()
        │
        ├── No chatSessionContext?
        │   └── handleDelegationFromAnotherChat(...)
        │       (Creates new session, opens it, streams progress)
        │
        ├── Parse sessionId from resource
        ├── Check _invalidCopilotCLISessionIdsWithErrorMessage
        │
        ├── Parallel resolve: [getModelId, getAgent]
        │   ├── Model: prompt file header → request.model → default model
        │   └── Agent: request.modeInstructions2 → copilotCLIAgents.resolveAgent
        │
        ├── getOrCreateSession(request, context, stream, options, disposables, token)
        │   ├── Resolve workspace/worktree via folderRepositoryManager
        │   ├── New session → sessionService.createSession(...)
        │   ├── Existing session → sessionService.getSession(...)
        │   ├── Set permission level, attach response stream
        │   └── Create baseline checkpoint on first request
        │
        ├── Track pending request in pendingRequestBySession map
        │
        ├── Dispatch by request type:
        │   ├── /delegate command → handleDelegationToCloud(...)
        │   ├── Pre-resolved context → session.handleRequest(prompt, attachments, ...)
        │   ├── Slash command only → session.handleRequest({ command }, ...)
        │   ├── Built-in slash commands → resolvePrompt + session.handleRequest(...)
        │   └── Default → resolvePrompt + session.handleRequest(...)
        │
        ├── commitWorktreeChangesIfNeeded(request, session, token)
        │   ├── Defer if other pending requests exist (steering in flight)
        │   ├── Worktree isolation → copilotCLIWorktreeManagerService.handleRequestCompleted
        │   ├── Workspace isolation → workspaceFolderService.handleRequestCompleted
        │   ├── Create checkpoint via copilotCLIWorktreeCheckpointService
        │   └── handlePullRequestCreated (fire-and-forget)
        │
        └── contentProvider.refreshSession({ reason: 'update', sessionId })
```

### Steering Coordination

When the user sends a new message while a previous request is still in progress:

1. VS Code sets `context.yieldRequested = true` on the running request's context.
2. The outer `handleRequest` polls this flag every 100ms (`CHECK_FOR_STEERING_DELAY`).
3. Once detected, `Promise.race` resolves the yield, returning control to VS Code.
4. The original `handleRequestImpl` continues running in the background — it is **not** cancelled.
5. The `pendingRequestBySession` map tracks all in-flight requests per session. Worktree commit/PR handling is deferred until the *last* pending request completes.

### Key Supporting Methods

| Method | Purpose |
|--------|---------|
| `getModelId(request, token)` | Resolves model: prompt file header → `request.model` → `copilotCLIModels.getDefaultModel()` |
| `getAgent(sessionId, request, token)` | Resolves custom agent from `request.modeInstructions2`; overrides agent tools if prompt file specifies tools |
| `getOrCreateSession(...)` | Initializes workspace info, creates/fetches SDK session, sets permission level, attaches stream |
| `commitWorktreeChangesIfNeeded(...)` | Post-request: commits worktree changes or stages workspace changes, creates checkpoints |
| `handlePullRequestCreated(session)` | Persists PR URL to worktree properties; retries detection with exponential backoff (2s, 4s, 8s) |
| `handleDelegationToCloud(...)` | Delegates to `CopilotCloudSessionsProvider`; warns about uncommitted changes |
| `handleDelegationFromAnotherChat(...)` | Creates a new CLI session from another chat context; summarizes history if present |
| `createModeInstructions(request)` | Extracts and serializes mode instructions for metadata persistence |

## Session Input State (Dropdowns)

The provider supplies dropdown configuration for the session creation UI:

### Option Group IDs

```typescript
const REPOSITORY_OPTION_ID = 'repository';
const BRANCH_OPTION_ID = 'branch';
const ISOLATION_OPTION_ID = 'isolation';
```

### Feature Flags

```typescript
function isBranchOptionFeatureEnabled(config): boolean {
    return config.getConfig(ConfigKey.Advanced.CLIBranchSupport);
}

function isIsolationOptionFeatureEnabled(config): boolean {
    return config.getConfig(ConfigKey.Advanced.CLIIsolationOption);
}
```

### Dropdown Groups

When enabled, the session creation UI shows:

1. **Repository** — Select which workspace folder or repository to use
2. **Branch** — Select or create a branch for isolation
3. **Isolation Mode** — Choose worktree isolation level

## Cloud Sessions Provider

`CopilotCloudSessionsProvider` is a sibling provider (registered separately) that manages cloud-hosted agent sessions:

```typescript
class CopilotCloudSessionsProvider extends Disposable {
    constructor(
        @ICopilotCLISessionService sessionService,
        // ... other dependencies
    )
}
```

This is registered alongside the local CLI sessions provider and handles the `copilot-cloud-agent` session type.

## Terminal Integration

```typescript
// In constructor:
this.terminalIntegration.setSessionDirResolver(terminal =>
    resolveSessionDirsForTerminal(this.sessionTracker, terminal)
);
```

This provides terminal link resolution — when a session runs in an integrated terminal, file paths are resolved relative to the session's working directory.

## Registration Pattern

Both the content provider and participant are registered together during contribution setup:

```typescript
// ⚠️ Proposed API
const participant = vscode.chat.createChatParticipant('copilotcli', handler.createHandler());
vscode.chat.registerChatSessionContentProvider('copilotcli', contentProvider, participant);
```

The provider ID `'copilotcli'` is a string literal used as the URI scheme. It is defined as a class field (`readonly copilotcliSessionType = 'copilotcli'`) on the contribution host and referenced consistently throughout registration.
