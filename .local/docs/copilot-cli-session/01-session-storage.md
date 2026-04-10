# Session Storage Format

## Overview

Copilot CLI sessions are persisted to `~/.copilot/session-state/{sessionId}/` as a collection of files. The storage format is shared between the CLI terminal process, the VS Code extension (via `@github/copilot/sdk`), and the VS Code platform agent host. All three consumers read and write to the same on-disk structure.

## Directory Layout

```
~/.copilot/
├── session-state/
│   ├── {uuid-1}/
│   │   ├── events.jsonl              # Event log (append-only, line-delimited JSON)
│   │   ├── workspace.yaml            # Session metadata
│   │   ├── vscode.metadata.json      # VS Code integration marker (optional)
│   │   ├── inuse.{pid}.lock          # Active CLI process attachment (optional, transient)
│   │   ├── checkpoints/
│   │   │   ├── 001-checkpoint-title.md
│   │   │   ├── 002-checkpoint-title.md
│   │   │   └── index.md               # Checkpoint index (Markdown table)
│   │   ├── files/                    # Session artifacts
│   │   ├── research/                 # Research reports (from /research command)
│   │   └── rewind-snapshots/         # Timeline snapshots for /rewind
│   ├── {uuid-2}/
│   └── ...
│
├── ide/
│   ├── {uuid}.lock                   # IDE connection lock files
│   └── ...
│
├── config.json                       # User configuration
├── mcp-config.json                   # MCP server definitions
├── command-history-state.json        # Command history
└── agents/                           # Custom agent plugins
```

### XDG Compliance

The base directory respects `XDG_STATE_HOME`. The helper functions in `cliHelpers.ts` resolve paths as:

```typescript
function getCopilotHome(): string {
    // Returns $XDG_STATE_HOME/.copilot or ~/.copilot
}

function getCopilotCliStateDir(): string {
    // Returns $XDG_STATE_HOME/.copilot/ide or ~/.copilot/ide
    // Used for IDE lock file directory resolution
}

function getCopilotCLISessionStateDir(): string {
    // Returns getCopilotHome() + '/session-state'
}

function getCopilotCLISessionDir(sessionId: string): string {
    // Returns getCopilotCLISessionStateDir() + '/' + sessionId
}

function getCopilotCLISessionEventsFile(sessionId: string): string {
    // Returns getCopilotCLISessionDir(sessionId) + '/events.jsonl'
}

function getCopilotCLIWorkspaceFile(sessionId: string): string {
    // Returns getCopilotCLISessionDir(sessionId) + '/workspace.yaml'
}
```

**Source:** `src/extension/chatSessions/copilotcli/node/cliHelpers.ts`

## events.jsonl — Event Log

The event log is the canonical record of a session's conversation. It is an append-only, line-delimited JSON file where each line is a single `SessionEvent` object.

### Format

```
{"type":"session.start","data":{...},"id":"evt-uuid-1","timestamp":"2026-03-23T18:48:32.069Z","parentId":null}
{"type":"user.message","data":{...},"id":"evt-uuid-2","timestamp":"2026-03-23T18:48:35.123Z","parentId":"evt-uuid-1"}
{"type":"assistant.turn_start","data":{...},"id":"evt-uuid-3","timestamp":"2026-03-23T18:48:36.000Z","parentId":"evt-uuid-2"}
{"type":"assistant.message","data":{...},"id":"evt-uuid-4","timestamp":"2026-03-23T18:48:42.500Z","parentId":"evt-uuid-3"}
```

### Event Structure

Every event conforms to this base structure:

```typescript
interface SessionEvent {
    type: string;                    // Discriminant (see event types below)
    data: Record<string, unknown>;   // Type-specific payload
    id: string;                      // UUID — unique per event
    timestamp: string;               // ISO 8601 UTC
    parentId: string | null;         // Previous event in the chain (null for session.start)
}
```

### Event Chain

Events form a linked list via `parentId`, establishing causal ordering:

```
session.start [id=evt-001, parentId=null]
    ↓
user.message [id=evt-002, parentId=evt-001]
    ↓
assistant.turn_start [id=evt-003, parentId=evt-002]
    ↓
external_tool.requested [id=evt-004, parentId=evt-003]
    ↓
external_tool.executed [id=evt-005, parentId=evt-004]
    ↓
assistant.message [id=evt-006, parentId=evt-005]
    ↓
assistant.turn_end [id=evt-007, parentId=evt-006]
```

### Core Event Types

| Event Type | Persistence | Purpose |
|-----------|-------------|---------|
| `session.start` | Persisted | Session initialization — sessionId, cwd, version, startTime |
| `session.resume` | Persisted | Resume metadata — event count at resume point |
| `user.message` | Persisted | User-submitted prompt with optional attachments |
| `assistant.message` | Persisted | Final LLM response content |
| `assistant.message_delta` | Ephemeral | Streaming response chunks (not written to events.jsonl) |
| `assistant.turn_start` | Persisted | Turn lifecycle — marks beginning of agent processing |
| `assistant.turn_end` | Persisted | Turn lifecycle — marks end of agent processing |
| `external_tool.requested` | Persisted | Tool invocation request (v3 protocol) |
| `external_tool.executed` | Persisted | Tool execution result |
| `permission.requested` | Persisted | Permission prompt for tool execution |
| `session.compaction_start` | Persisted | Context window compaction initiated |
| `session.compaction_complete` | Persisted | Compaction finished — checkpoint reference |
| `session.shutdown` | Persisted | Session termination with usage statistics |
| `session.idle` | Ephemeral | Idle timeout warning/expiration |
| `session.error` | Persisted | Error state |

### Example: session.start Event

```json
{
    "type": "session.start",
    "data": {
        "sessionId": "2300305f-9dc5-4860-9717-de0fc3813cfc",
        "version": 1,
        "producer": "copilot-agent",
        "copilotVersion": "1.0.10",
        "startTime": "2026-03-23T18:48:32.067Z",
        "context": {
            "cwd": "/Users/ming/GitRepos"
        },
        "alreadyInUse": false
    },
    "id": "4fe0b5be-8fb7-42a8-8d74-298fe7239261",
    "timestamp": "2026-03-23T18:48:32.069Z",
    "parentId": null
}
```

### Reading events.jsonl

Events are consumed through two distinct mechanisms depending on context.

#### SDK Session Path — `getChatHistoryImpl()`

When a session is loaded via the SDK (the primary path for VS Code history rendering), events are retrieved from the in-memory SDK session object — **not** from the file directly:

```typescript
// In CopilotCLISessionService.getChatHistoryImpl()
const existingSession = this._sessionWrappers.get(sessionId)?.object?.sdkSession;
if (existingSession) {
    modelId = await existingSession.getSelectedModel();
    events = existingSession.getEvents();
} else {
    const session = await sessionManager.getSession({ sessionId }, false);
    modelId = await session.getSelectedModel();
    events = session.getEvents();
}
const history = buildChatHistoryFromEvents(sessionId, modelId, events, ...);
```

The SDK session hydrates itself from the on-disk `events.jsonl` internally; callers receive parsed `SessionEvent[]` via `session.getEvents()`.

#### Standalone File-Reading Path — `readSessionEventsFile()`

A separate standalone function reads the file directly using Node.js streaming, used for lightweight queries (e.g., extracting a single event type without loading a full SDK session):

```typescript
// Standalone function outside the service class
async function readSessionEventsFile(
    sessionId: string,
    findFirstEventType?: string
): Promise<SessionEvent[]> {
    const eventsFile = joinPath(
        URI.file(getCopilotCLISessionDir(sessionId)),
        'events.jsonl'
    );
    const events: SessionEvent[] = [];
    const stream = createReadStream(eventsFile.fsPath, { encoding: 'utf8' });
    const reader = createInterface({ input: stream, crlfDelay: Infinity });
    for await (const line of reader) {
        if (line.trim().length === 0) continue;
        const sessionEvent = JSON.parse(line) as SessionEvent;
        events.push(sessionEvent);
        if (findFirstEventType && sessionEvent.type === findFirstEventType) {
            break; // Early exit — avoids reading the entire file
        }
    }
    reader.close();
    stream.close();
    return events;
}
```

This function streams line-by-line via `createReadStream` + `createInterface` (not bulk `readFile`), and supports early termination when a specific event type is found.

**Source:** `src/extension/chatSessions/copilotcli/node/copilotcliSessionService.ts`

## workspace.yaml — Session Metadata

Contains lightweight metadata about the session. Read by the session list UI and the resume picker.

### Format

```yaml
id: 2300305f-9dc5-4860-9717-de0fc3813cfc
cwd: /Users/ming/GitRepos
summary: General Greeting
summary_count: 2
created_at: 2026-03-23T18:48:32.069Z
updated_at: 2026-03-23T18:52:47.584Z
```

### Fields

| Field | Type | Purpose |
|-------|------|---------|
| `id` | string (UUID) | Session identifier — matches directory name |
| `cwd` | string (path) | Working directory at session creation |
| `summary` | string | Auto-generated or user-set session title |
| `summary_count` | number | Number of messages processed before summary generated |
| `created_at` | string (ISO 8601) | Session creation timestamp |
| `updated_at` | string (ISO 8601) | Last activity timestamp |

### SDK Type

The SDK exposes this as `LocalSessionMetadata` (imported from `@github/copilot/sdk`):

```typescript
// SDK external type — shape inferred from usage patterns in the extension
interface LocalSessionMetadata {
    sessionId: string;
    summary?: string;
    context?: SessionContext;  // { cwd?: string; gitRoot?: string; ... }
    startTime: Date;
    modifiedTime: Date;
}
```

The extension accesses `metadata.context?.cwd` for the working directory and calls `metadata.startTime.getTime()` / `metadata.modifiedTime.getTime()` for epoch timestamps. `SessionContext` is imported separately from the SDK.

## vscode.metadata.json — VS Code Marker

An empty or minimal JSON object indicating VS Code has accessed this session:

```json
{}
```

**Purpose:** Allows the CLI to detect VS Code origin and potentially provide IDE-specific behavior. Created by the extension when opening or resuming a session.

## inuse.{pid}.lock — Active Process Attachment

Session directories may contain `inuse.{pid}.lock` files indicating that a CLI process is actively attached to the session. The filename encodes the process ID of the owning CLI process.

### Format

The file content is the PID as plain text (e.g., `3191`).

### Lifecycle

1. **Creation:** The CLI process writes `inuse.{pid}.lock` when it attaches to a session
2. **Content:** Contains the PID of the attached process as a text string
3. **Presence Semantics:** If the file exists, the session is considered in-use by that process
4. **Stale Detection:** The PID can be checked against running processes to detect abandoned locks
5. **Deletion:** Removed when the CLI process detaches or terminates

### Example

```
~/.copilot/session-state/{uuid}/inuse.3191.lock   # Content: "3191"
```

## Checkpoints — Context Compaction

When the conversation's token count approaches the context window limit, the system performs **compaction**:

1. `session.compaction_start` event emitted
2. An auxiliary model summarizes the conversation
3. Summary written to `checkpoints/{n}-title.md`
4. `session.compaction_complete` event emitted with checkpoint reference
5. Subsequent turns use the checkpoint summary as context instead of full history

### Checkpoint Structure

```
checkpoints/
├── index.md                      # Checkpoint index (Markdown table)
├── 001-initial-analysis.md       # First compaction summary
├── 002-implementation-phase.md   # Second compaction summary
└── ...
```

The checkpoint index is a Markdown file with a table mapping sequence numbers to summary files:

```markdown
# Checkpoint History

Checkpoints are listed in chronological order. Checkpoint 1 is the oldest, higher numbers are more recent.

| # | Title | File |
|---|-------|------|
| 1 | Initial greeting and session setup | 001-initial-greeting-and-session-s.md |
| 2 | Initial greeting and introduction | 002-initial-greeting-and-introduct.md |
```

## IDE Lock Files

**Location:** `~/.copilot/ide/{uuid}.lock`

Each active IDE connection creates a lock file encoding the communication endpoint and credentials.

### Lock File Format

```json
{
    "socketPath": "/var/folders/w_/.../mcp-9MHaRK/mcp.sock",
    "scheme": "unix",
    "headers": {
        "Authorization": "Nonce 3f063a9a-855b-4b4c-b866-85225da10675"
    },
    "pid": 16083,
    "ideName": "Visual Studio Code - Insiders",
    "timestamp": 1775284290219,
    "workspaceFolders": [
        "/Users/ming/GitRepos/github-ming86/vscode-copilot-chat",
        "/Users/ming/GitRepos/github-ming86/vscode"
    ],
    "isTrusted": true
}
```

### Fields

| Field | Type | Purpose |
|-------|------|---------|
| `socketPath` | string | Unix domain socket path for JSON-RPC communication |
| `scheme` | `"unix"` or `"tcp"` | Transport protocol |
| `headers.Authorization` | `"Nonce {uuid}"` | Nonce-based bearer token for authentication |
| `pid` | number | IDE process ID |
| `ideName` | string | IDE identification string |
| `timestamp` | number | Lock file creation time (ms since epoch) |
| `workspaceFolders` | string[] | Workspace root paths open in this IDE |
| `isTrusted` | boolean | Workspace trust level |

### Lock File Lifecycle

1. **Creation:** IDE writes lock file on connection to ACP server or session activation
2. **Validation:** CLI ACP server validates nonce against lock file on each request
3. **Stale Detection:** Timestamp enables detection of abandoned lock files
4. **Deletion:** Lock file removed on IDE disconnect or process termination
5. **Multiple Locks:** Multiple IDE windows create separate lock files — concurrent access supported

## Filesystem Monitoring

The extension monitors session state files for changes:

```typescript
// In CopilotCLISessionService.monitorSessionFiles()
// Watches: ~/.copilot/session-state/**/*.jsonl

watcher.onDidCreate → fires triggerSessionsChangeEvent() + _onDidChangeSession (not _onDidCreateSession)
watcher.onDidDelete → removes from cache, fires _onDidDeleteSession + triggerSessionsChangeEvent()
watcher.onDidChange → skips if already tracked or fetching, fires triggerOnDidChangeSessionItem() + triggerSessionsChangeEvent()
```

**Note on `onDidCreate`:** The watcher's `onDidCreate` handler fires `_onDidChangeSession`, not `_onDidCreateSession`. The `_onDidCreateSession` emitter fires only during explicit programmatic session creation (e.g., fork-based session creation). This distinction is important: external session creation by the CLI is surfaced as a *change*, not a *create* event, to avoid duplicating the explicit creation flow.

**Note on path resolution:** The watcher hardcodes `joinPath(this.nativeEnv.userHome, '.copilot', 'session-state')` rather than using the XDG-aware `getCopilotCLISessionStateDir()` helper from `cliHelpers.ts`. This means sessions stored under `$XDG_STATE_HOME/.copilot/session-state/` will not be detected by the filesystem watcher, even though path resolution functions honor `XDG_STATE_HOME`. This is a known inconsistency.

## Global Configuration

### config.json

```json
{
    "banner": "never",
    "renderMarkdown": true,
    "theme": "auto",
    "model": "claude-opus-4.6",
    "trusted_folders": ["/Users/ming/GitRepos/project"],
    "firstLaunchAt": "2026-03-11T00:00:00.000Z"
}
```

### mcp-config.json

```json
{
    "mcpServers": {
        "github": {
            "type": "builtin",
            "tools": ["*"]
        },
        "context7": {
            "type": "local",
            "command": "npx",
            "args": ["-y", "@upstash/context7-mcp"]
        }
    }
}
```

MCP servers defined here are available to all sessions. Workspace-level configuration (`.vscode/mcp.json`, `.mcp.json`) supplements these.
