# Implementation Guide

> Last updated: 2026-04-21

A step-by-step guide for recreating the Copilot CLI session integration feature in another project. Each step is self-contained with rationale, interfaces, schemas, and pseudocode. No prior reading of the VS Code source is required.

> **Scope Note:** This document describes VS Code's internal implementation using `@github/copilot/sdk` (VS Code's bundled SDK package). For standalone webapp implementation using the public `@github/copilot-sdk` package, see [06-webapp-extraction-guide.md](./06-webapp-extraction-guide.md). The SDK event types and API methods documented here apply to both packages — they share the same underlying SDK.

---

## Prerequisites

| Dependency | Purpose |
|---|---|
| Chat panel UI | Renders conversation turns (user prompts, assistant responses, tool calls) |
| Node.js runtime | SDK integration and MCP server execution |
| `@github/copilot/sdk` | Copilot API access — session lifecycle, streaming, tool dispatch |
| `@modelcontextprotocol/sdk` | MCP server implementation (StreamableHTTPServerTransport) |
| `express` | HTTP server bound to a Unix domain socket |
| SQLite library (e.g. `@vscode/sqlite3` or `better-sqlite3`) | Per-session persistence of turns and file edits |
| File system access | Lock files, event logs, session directories |
| Session list UI | Sidebar or panel widget for browsing, filtering, and opening sessions |

---

## Step 1: Define the Session Data Model

### Rationale

Three distinct shapes serve three distinct consumers: the **list view** needs lightweight summaries, the **chat widget** needs full state, and the **persistence layer** needs serializable metadata. Conflating them produces either over-fetching in lists or under-specification in the editor.

### Interfaces

```typescript
// --- Session identification ---

/**
 * URI scheme encodes the session type (e.g. "copilotcli", "copilotcloud").
 * This avoids a separate "type" field and makes routing unambiguous.
 */
type SessionUri = string; // e.g. "copilotcli:///untitled-a1b2c3d4"

// --- Lightweight summary (list view) ---

interface SessionSummary {
  readonly uri: SessionUri;
  readonly title: string;
  readonly status: SessionStatus;
  readonly createdAt: number;       // epoch ms
  readonly lastActivityAt: number;  // epoch ms
  readonly providerName: string;    // e.g. "Copilot CLI"
  readonly providerIcon: string;    // icon identifier or URI
  readonly fileChangeCount: number;
  readonly isPinned: boolean;
  readonly isArchived: boolean;
  readonly isRead: boolean;
  readonly workspaceFolders: string[];
}

type SessionStatus =
  | 'active'     // turn in progress
  | 'idle'       // waiting for user input
  | 'completed'  // finished successfully
  | 'error'      // terminal failure
  | 'archived';

// --- Full session state (active chat widget) ---

interface SessionState {
  readonly uri: SessionUri;
  readonly title: Observable<string>;
  readonly status: Observable<SessionStatus>;
  readonly turns: Observable<Turn[]>;
  readonly activeTurn: Observable<Turn | undefined>;
  readonly lifecycle: SessionLifecycle;
  readonly tools: ToolDefinition[];
  readonly customizations: SessionCustomizations;
  readonly fileEdits: Observable<FileEditRecord[]>;
}

type SessionLifecycle =
  | 'uninitialized'
  | 'initializing'
  | 'ready'
  | 'disposed';

// --- Turn model ---

interface Turn {
  readonly id: string;
  readonly role: 'user' | 'assistant';
  readonly content: TurnContent[];
  readonly toolCalls: ToolCall[];
  readonly status: TurnStatus;
  readonly timestamp: number;
}

type TurnStatus = 'pending' | 'streaming' | 'complete' | 'error' | 'cancelled';

interface TurnContent {
  readonly kind: 'text' | 'code' | 'image' | 'reference';
  readonly value: string;
  readonly language?: string;       // for 'code' kind
  readonly mimeType?: string;       // for 'image' kind
}

interface ToolCall {
  readonly id: string;
  readonly name: string;
  readonly arguments: Record<string, unknown>;
  readonly result?: string;
  readonly status: 'pending' | 'running' | 'complete' | 'error';
}

// --- File edit records ---

interface FileEditRecord {
  readonly turnId: string;
  readonly toolCallId: string;
  readonly filePath: string;
  readonly editType: 'create' | 'edit' | 'rename' | 'delete';
  readonly originalPath?: string;   // for renames
  readonly beforeContent?: Uint8Array;
  readonly afterContent?: Uint8Array;
  readonly addedLines: number;
  readonly removedLines: number;
}

// --- Session metadata (persisted key-value pairs) ---

interface SessionMetadata {
  readonly title: string;
  readonly providerName: string;
  readonly createdAt: number;
  readonly lastActivityAt: number;
  readonly workspaceFolders: string[];
  readonly modelId?: string;
}
```

### Design Decisions

1. **Observable pattern.** `title`, `status`, `turns`, and `fileEdits` are observable so the UI reacts to changes without polling. Use whatever reactive primitive your framework provides (RxJS `BehaviorSubject`, Svelte stores, Vue refs, etc.).

2. **URI-based identification.** The URI scheme (`copilotcli:///...`) encodes session type. Routing logic becomes a scheme check rather than a type enum lookup. Parsing is trivial; serialization is a string.

3. **Summary vs. State separation.** The list view loads `SessionSummary` objects — no turns, no file content. The chat widget loads `SessionState` on demand. This keeps the list responsive even with thousands of sessions.

---

## Step 2: Implement Session Persistence

### Architecture: Three Layers

```
┌─────────────────────────────────────────┐
│         Layer 3: In-Memory State        │
│  (active sessions, observable, fast)    │
├─────────────────────────────────────────┤
│    Layer 2: Application State Storage   │
│  (UI preferences, selected model, etc) │
├─────────────────────────────────────────┤
│      Layer 1: Per-Session SQLite DB     │
│  (turns, file edits, metadata, durable)│
└─────────────────────────────────────────┘
```

### Layer 1: Per-Session SQLite Database

Each session gets its own database file at:

```
<userDataPath>/agentSessionData/<sessionId>/session.db
```

This isolation simplifies backup, deletion, and garbage collection — removing a session is `rm -rf <sessionDir>`.

#### Schema

```sql
-- Turns table: one row per conversation turn
CREATE TABLE turns (
    id TEXT PRIMARY KEY NOT NULL
);

-- File edits (v1): before/after snapshots for each file touched by a tool call
CREATE TABLE file_edits (
    turn_id TEXT NOT NULL REFERENCES turns(id) ON DELETE CASCADE,
    tool_call_id TEXT NOT NULL,
    file_path TEXT NOT NULL,
    before_content BLOB NOT NULL,
    after_content BLOB NOT NULL,
    added_lines INTEGER,
    removed_lines INTEGER,
    PRIMARY KEY (tool_call_id, file_path)
);

-- Session metadata (added in migration v2): key-value pairs
CREATE TABLE session_metadata (
    key TEXT PRIMARY KEY NOT NULL,
    value TEXT NOT NULL
);

-- File edits v3 (migration v3): adds edit_type and original_path,
-- relaxes NOT NULL on before_content/after_content for create/delete ops
-- ALTER TABLE file_edits ADD COLUMN edit_type TEXT NOT NULL DEFAULT 'edit';
-- ALTER TABLE file_edits ADD COLUMN original_path TEXT;
-- (Actual migration recreates the table; shown here as logical additions.)
```

#### Connection Management

Use ref-counting to share a single connection per session across multiple consumers. VS Code uses `@vscode/sqlite3` (async, callback-based) wrapped in a `ReferenceCollection`; the pseudocode below illustrates the pattern with a synchronous API for clarity:

```typescript
class SessionDatabase {
  private static readonly connections = new Map<string, { db: Database; refCount: number }>();

  static acquire(sessionId: string): Database {
    let entry = this.connections.get(sessionId);
    if (!entry) {
      const dbPath = path.join(dataDir, 'agentSessionData', sessionId, 'session.db');
      fs.mkdirSync(path.dirname(dbPath), { recursive: true });
      const db = new Database(dbPath);
      db.pragma('foreign_keys = ON');
      this.runMigrations(db);
      entry = { db, refCount: 0 };
      this.connections.set(sessionId, entry);
    }
    entry.refCount++;
    return entry.db;
  }

  static release(sessionId: string): void {
    const entry = this.connections.get(sessionId);
    if (!entry) return;
    entry.refCount--;
    if (entry.refCount <= 0) {
      entry.db.close();
      this.connections.delete(sessionId);
    }
  }

  private static runMigrations(db: Database): void {
    db.exec(`
      CREATE TABLE IF NOT EXISTS turns (id TEXT PRIMARY KEY NOT NULL);
      CREATE TABLE IF NOT EXISTS file_edits (
        turn_id TEXT NOT NULL REFERENCES turns(id) ON DELETE CASCADE,
        tool_call_id TEXT NOT NULL,
        file_path TEXT NOT NULL,
        before_content BLOB NOT NULL,
        after_content BLOB NOT NULL,
        added_lines INTEGER,
        removed_lines INTEGER,
        PRIMARY KEY (tool_call_id, file_path)
      );
      CREATE TABLE IF NOT EXISTS session_metadata (
        key TEXT PRIMARY KEY NOT NULL,
        value TEXT NOT NULL
      );
    `);
    // Note: VS Code uses a versioned migration system (PRAGMA user_version)
    // to evolve the schema over time. A production implementation should
    // track applied migrations rather than relying on IF NOT EXISTS.
  }
}
```

#### Garbage Collection

On startup, scan the sessions directory and remove orphaned entries. VS Code's actual implementation performs **immediate** cleanup with no grace period — any directory not in the known set is removed:

```typescript
async function garbageCollectSessions(dataDir: string, knownSessionIds: Set<string>): Promise<void> {
  const sessionsDir = path.join(dataDir, 'agentSessionData');
  if (!fs.existsSync(sessionsDir)) return;

  for (const entry of fs.readdirSync(sessionsDir, { withFileTypes: true })) {
    if (entry.isDirectory() && !knownSessionIds.has(entry.name)) {
      fs.rmSync(path.join(sessionsDir, entry.name), { recursive: true, force: true });
    }
  }
}
```

### Layer 2: Application State Storage

Store lightweight UI preferences using whatever key-value mechanism your application provides (localStorage, a settings file, a global SQLite database):

```typescript
interface AppSessionState {
  lastSelectedSessionUri: string | undefined;
  activeProviderId: string | undefined;
  selectedModelId: string | undefined;
  sidebarWidth: number;
  filterQuery: string;
}
```

This layer holds nothing about session *content* — only about how the user has configured their view.

### Layer 3: In-Memory State (Active Sessions)

For sessions currently displayed in the chat widget, maintain an in-memory state tree with an action-based update model. This enables undo, optimistic updates, and multi-client synchronization.

```typescript
// Actions describe state transitions
type SessionAction =
  | { type: 'addTurn'; turn: Turn }
  | { type: 'updateTurn'; turnId: string; patch: Partial<Turn> }
  | { type: 'setStatus'; status: SessionStatus }
  | { type: 'addFileEdit'; edit: FileEditRecord }
  | { type: 'setTitle'; title: string };

// ActionEnvelope wraps actions with sequencing metadata
interface ActionEnvelope {
  readonly action: SessionAction;
  readonly serverSeq: number;       // monotonically increasing sequence
  readonly origin?: {               // client origin for conflict resolution
    clientId: string;
    clientSeq: number;
  };
  readonly rejectionReason?: string; // set when server rejects a client action
}

// Reducer: pure function from state + action → new state
function sessionReducer(state: SessionState, envelope: ActionEnvelope): SessionState {
  const { action } = envelope;
  switch (action.type) {
    case 'addTurn':
      return { ...state, turns: [...state.turns, action.turn] };
    case 'updateTurn':
      return {
        ...state,
        turns: state.turns.map(t =>
          t.id === action.turnId ? { ...t, ...action.patch } : t
        ),
      };
    case 'setStatus':
      return { ...state, status: action.status };
    case 'addFileEdit':
      return { ...state, fileEdits: [...state.fileEdits, action.edit] };
    case 'setTitle':
      return { ...state, title: action.title };
    default:
      return state;
  }
}
```

**Write-ahead reconciliation** for multi-client scenarios: local actions are applied optimistically and tagged with an `origin` containing the `clientId` and a monotonic `clientSeq`. When the server confirms with a `serverSeq`, remove the local pending action and apply the canonical version. If the server sets `rejectionReason`, discard the local action and rebase remaining pending actions on top of the server state.

---

## Step 3: Implement the MCP Server

The MCP server allows an external CLI process to connect to your IDE and exchange tool calls, editor state, and session lifecycle events.

### 3a. HTTP Server on a Unix Domain Socket

```typescript
import express from 'express';
import { randomUUID } from 'crypto';
import fs from 'fs';
import os from 'os';
import path from 'path';

function createMcpServer(): { app: express.Application; socketPath: string; nonce: string } {
  const app = express();
  const nonce = randomUUID();

  // Socket in a private temp directory
  // VS Code convention: temp dir with `ide-mcp-` prefix.
  // The standalone webapp uses `$XDG_RUNTIME_DIR/copilot/mcp-{uuid}/mcp.sock` instead (see doc 06).
  const socketDir = fs.mkdtempSync(path.join(os.tmpdir(), 'ide-mcp-'));
  fs.chmodSync(socketDir, 0o700);
  const socketPath = path.join(socketDir, 'mcp.sock');

  // Body parsing with 10 MB limit for large tool results
  app.use(express.json({ limit: '10mb' }));

  return { app, socketPath, nonce };
}
```

### 3b. Nonce-Based Authentication

The nonce is generated once at server startup and written into the lock file (Step 4). Only processes that can read the lock file — which requires file system access to the user's home directory — can authenticate.

```typescript
function authMiddleware(nonce: string): express.RequestHandler {
  return (req, res, next) => {
    const auth = req.headers.authorization;
    if (auth !== `Nonce ${nonce}`) {
      res.status(401).json({ error: 'Invalid or missing nonce' });
      return;
    }
    next();
  };
}
```

### 3c. MCP Tool Definitions

Register the following tools with the MCP server. Each tool corresponds to a capability the CLI process may invoke.

| Tool | Parameters | Returns | Purpose |
|---|---|---|---|
| `get_vscode_info` | (none) | IDE name, version, app root, language, machine ID, session ID, URI scheme, shell | CLI identifies the host IDE |
| `get_selection` | (none) | file path, file URL, text, selection range, `current` boolean | CLI reads current editor selection |
| `get_diagnostics` | `{ uri?: string }` | diagnostic entries | CLI reads language errors/warnings |
| `open_diff` | `{ original_file_path, new_file_contents, tab_name }` | `success`, `result`: `SAVED` or `REJECTED`, `trigger`, `tab_name`, `message` (blocks until user action) | Show diff view, block until user decides |
| `close_diff` | `{ tab_name }` | `success`, `already_closed`, `tab_name`, `message` | Programmatically close a diff tab |
| `update_session_name` | `{ name }` | `success: true` | Update session title in the list |

```typescript
import { McpServer } from '@modelcontextprotocol/sdk/server/mcp.js';
import { z } from 'zod';

function registerTools(server: McpServer, context: IdeContext): void {
  server.tool(
    'get_vscode_info',
    'Returns information about the IDE (name, version, workspace)',
    {},
    async () => ({
      content: [{
        type: 'text',
        text: JSON.stringify({
          name: context.ideName,
          version: context.ideVersion,
          workspaceFolders: context.workspaceFolders,
        }),
      }],
    })
  );

  server.tool(
    'get_selection',
    'Returns the current editor text selection',
    {},
    async () => {
      const selection = context.getSelection();
      return {
        content: [{
          type: 'text',
          text: JSON.stringify(selection ?? { text: '', uri: '', range: null }),
        }],
      };
    }
  );

  server.tool(
    'get_diagnostics',
    'Returns language diagnostics (errors, warnings)',
    { uri: z.string().optional().describe('File URI to filter diagnostics') },
    async ({ uri }) => {
      const diagnostics = context.getDiagnostics(uri);
      return {
        content: [{
          type: 'text',
          text: JSON.stringify(diagnostics),
        }],
      };
    }
  );

  server.tool(
    'open_diff',
    'Opens a diff view comparing original file content with new content. Blocks until user accepts, rejects, or closes.',
    {
      original_file_path: z.string().describe('Path to the original file'),
      new_file_contents: z.string().describe('The new file contents to compare against'),
      tab_name: z.string().describe('Name for the diff tab'),
    },
    async ({ original_file_path, new_file_contents, tab_name }) => {
      const result = await context.openDiffAndWait(original_file_path, new_file_contents, tab_name);
      return {
        content: [{
          type: 'text',
          text: JSON.stringify({
            success: true,
            result: result.status,        // 'SAVED' | 'REJECTED'
            trigger: result.trigger,
            tab_name,
            message: result.status === 'SAVED'
              ? `User accepted changes for ${original_file_path}`
              : `User rejected changes for ${original_file_path}`,
          }),
        }],
      };
    }
  );

  server.tool(
    'close_diff',
    'Programmatically closes a diff tab by name',
    { tab_name: z.string().describe('The tab name of the diff to close') },
    async ({ tab_name }) => {
      const result = await context.closeDiff(tab_name);
      return {
        content: [{
          type: 'text',
          text: JSON.stringify({
            success: result.success,
            already_closed: result.alreadyClosed,
            tab_name,
            message: result.alreadyClosed
              ? `Tab "${tab_name}" was already closed`
              : `Closed tab "${tab_name}"`,
          }),
        }],
      };
    }
  );

  server.tool(
    'update_session_name',
    'Updates the display name of the current session',
    { name: z.string().describe('New session name') },
    async ({ name }) => {
      context.updateSessionName(name);
      return { content: [{ type: 'text', text: 'OK' }] };
    }
  );
}
```

### 3d. Push Notifications

The MCP server pushes IDE state changes to the CLI process so it can react to user activity.

```typescript
function setupNotifications(server: McpServer, context: IdeContext): Disposable[] {
  const disposables: Disposable[] = [];

  // Selection changes — debounced (200ms) to avoid noise during cursor movement
  let selectionTimer: NodeJS.Timeout | undefined;
  disposables.push(
    context.onSelectionChanged((selection) => {
      clearTimeout(selectionTimer);
      selectionTimer = setTimeout(() => {
        server.sendNotification('selection_changed', {
          text: selection.text,
          filePath: selection.filePath,
          fileUrl: selection.fileUrl,
          selection: selection.selection,  // { start, end, isEmpty }
          current: selection.current,
        });
      }, 200);
    })
  );

  // Diagnostics changes — debounced to avoid noise during rapid typing
  let diagnosticsTimer: NodeJS.Timeout | undefined;
  disposables.push(
    context.onDiagnosticsChanged((changedUris: string[]) => {
      clearTimeout(diagnosticsTimer);
      diagnosticsTimer = setTimeout(() => {
        const uris = changedUris.map(uri => ({
          uri,
          diagnostics: context.getDiagnostics(uri),
        }));
        server.sendNotification('diagnostics_changed', { uris });
      }, 200); // VS Code uses 200ms debounce for both selection and diagnostics
    })
  );

  return disposables;
}
```

### 3e. Session Multiplexing

A single MCP server handles multiple concurrent sessions. Each session gets its own `StreamableHTTPServerTransport`.

```typescript
import { StreamableHTTPServerTransport } from '@modelcontextprotocol/sdk/server/streamableHttp.js';

class McpSessionRouter {
  private transports = new Map<string, StreamableHTTPServerTransport>();

  async handleRequest(req: express.Request, res: express.Response): Promise<void> {
    const sessionId = this.extractSessionId(req);

    if (req.method === 'POST' && this.isInitializeRequest(req.body)) {
      if (this.transports.has(sessionId)) {
        res.status(409).json({ error: 'Session already initialized' });
        return;
      }
      const transport = new StreamableHTTPServerTransport({ sessionIdGenerator: () => sessionId });
      this.transports.set(sessionId, transport);
      await transport.handleRequest(req, res);
      return;
    }

    const transport = this.transports.get(sessionId);
    if (!transport) {
      res.status(404).json({ error: 'Session not found' });
      return;
    }

    if (req.method === 'DELETE') {
      await transport.handleRequest(req, res);
      this.transports.delete(sessionId);
      return;
    }

    await transport.handleRequest(req, res);
  }

  private extractSessionId(req: express.Request): string {
    // Session ID from MCP or custom header
    return (req.headers['mcp-session-id'] as string) ?? req.headers['x-copilot-session-id'] as string ?? 'default';
  }

  private isInitializeRequest(body: unknown): boolean {
    return typeof body === 'object' && body !== null && (body as any).method === 'initialize';
  }
}
```

### Full Server Assembly

```typescript
function startMcpServer(context: IdeContext): { socketPath: string; nonce: string; dispose: () => void } {
  const { app, socketPath, nonce } = createMcpServer();
  const mcpServer = new McpServer({ name: 'ide-mcp', version: '1.0.0' });
  const router = new McpSessionRouter();

  registerTools(mcpServer, context);
  const notificationDisposables = setupNotifications(mcpServer, context);

  app.use(authMiddleware(nonce));
  app.post('/mcp', (req, res) => router.handleRequest(req, res));
  app.get('/mcp', (req, res) => router.handleRequest(req, res));
  app.delete('/mcp', (req, res) => router.handleRequest(req, res));

  const server = app.listen(socketPath);

  return {
    socketPath,
    nonce,
    dispose: () => {
      notificationDisposables.forEach(d => d.dispose());
      server.close();
      try { fs.unlinkSync(socketPath); } catch {}
    },
  };
}
```

---

## Step 4: Implement Lock File Discovery

Lock files allow external CLI processes to discover running IDE instances. Each IDE instance writes a lock file on startup; CLI processes scan the directory to find a target.

### 4a. Lock File Format and Location

```
~/.copilot/ide/<uuid>.lock
```

```typescript
interface LockFileInfo {
  socketPath: string;                    // path to Unix domain socket
  scheme: string;                         // 'unix' or 'pipe'
  headers: Record<string, string>;       // { Authorization: 'Nonce <uuid>' }
  pid: number;                           // IDE process ID
  ideName: string;                       // e.g. "Visual Studio Code"
  timestamp: number;                     // epoch ms, creation time
  workspaceFolders: string[];            // absolute paths
  isTrusted: boolean;                    // workspace trust status
}
```

#### Writing the Lock File

```typescript
function writeLockFile(info: LockFileInfo): string {
  const lockDir = path.join(os.homedir(), '.copilot', 'ide');
  fs.mkdirSync(lockDir, { recursive: true, mode: 0o700 });

  const lockId = randomUUID();
  const lockPath = path.join(lockDir, `${lockId}.lock`);
  const content = JSON.stringify(info, null, 2);

  // Atomic write: temp file → rename
  const tempPath = `${lockPath}.tmp`;
  fs.writeFileSync(tempPath, content, { mode: 0o600 });
  fs.renameSync(tempPath, lockPath);

  return lockPath;
}
```

#### Cleaning Stale Locks at Startup

```typescript
function cleanStaleLockFiles(): void {
  const lockDir = path.join(os.homedir(), '.copilot', 'ide');
  if (!fs.existsSync(lockDir)) return;

  for (const file of fs.readdirSync(lockDir)) {
    if (!file.endsWith('.lock')) continue;
    const lockPath = path.join(lockDir, file);
    try {
      const info: LockFileInfo = JSON.parse(fs.readFileSync(lockPath, 'utf-8'));
      if (!isProcessAlive(info.pid)) {
        fs.unlinkSync(lockPath);
      }
    } catch {
      // Corrupt lock file — remove it
      try { fs.unlinkSync(lockPath); } catch {}
    }
  }
}

function isProcessAlive(pid: number): boolean {
  try {
    process.kill(pid, 0); // signal 0: existence check, no actual signal
    return true;
  } catch {
    return false;
  }
}
```

### 4b. Update on Workspace Change

When the user opens or closes a workspace folder, update the lock file:

```typescript
function updateLockFile(lockPath: string, workspaceFolders: string[]): void {
  const content = fs.readFileSync(lockPath, 'utf-8');
  const info: LockFileInfo = JSON.parse(content);
  info.workspaceFolders = workspaceFolders;
  info.timestamp = Date.now();

  const tempPath = `${lockPath}.tmp`;
  fs.writeFileSync(tempPath, JSON.stringify(info, null, 2), { mode: 0o600 });
  fs.renameSync(tempPath, lockPath);
}
```

### 4c. Delete on Shutdown

```typescript
function deleteLockFile(lockPath: string): void {
  try { fs.unlinkSync(lockPath); } catch {}
}

// Register with your application's shutdown hook:
process.on('exit', () => deleteLockFile(lockPath));
process.on('SIGTERM', () => { deleteLockFile(lockPath); process.exit(0); });
process.on('SIGINT', () => { deleteLockFile(lockPath); process.exit(0); });
```

---

## Step 5: Wrap the Copilot SDK

### 5a. Session Wrapper Class

The SDK's `Session` object emits a stream of events. The wrapper class translates these into your application's turn model.

```typescript
import { Session as CopilotSession, SessionEvent } from '@github/copilot/sdk';

class CopilotSessionWrapper {
  private session: CopilotSession;
  private turns: Turn[] = [];
  private activeTurn: Turn | undefined;
  private readonly onTurnUpdate = new EventEmitter<Turn>();
  private readonly onStatusChange = new EventEmitter<SessionStatus>();

  constructor(session: CopilotSession) {
    this.session = session;
    this.bindEvents();
  }

  private bindEvents(): void {
    this.session.on('user.message', (event) => {
      const turn: Turn = {
        id: event.data.turnId,
        role: 'user',
        content: [{ kind: 'text', value: event.data.prompt }],
        toolCalls: [],
        status: 'complete',
        timestamp: event.timestamp, // ISO 8601 string
      };
      this.turns.push(turn);
      this.onTurnUpdate.fire(turn);
    });

    this.session.on('assistant.turn_start', (event) => {
      const turn: Turn = {
        id: event.data.turnId,
        role: 'assistant',
        content: [],
        toolCalls: [],
        status: 'streaming',
        timestamp: event.timestamp, // ISO 8601 string
      };
      this.activeTurn = turn;
      this.turns.push(turn);
      this.onTurnUpdate.fire(turn);
    });

    this.session.on('assistant.message_delta', (event) => {
      if (!this.activeTurn) return;
      this.appendContent(this.activeTurn, event.data.delta);
      this.onTurnUpdate.fire(this.activeTurn);
    });

    this.session.on('tool.execution_start', (event) => {
      if (!this.activeTurn) return;
      const toolCall: ToolCall = {
        id: event.data.toolCallId,
        name: event.data.toolName,
        arguments: event.data.arguments,
        status: 'running',
      };
      this.activeTurn.toolCalls.push(toolCall);
      this.onTurnUpdate.fire(this.activeTurn);
    });

    this.session.on('tool.execution_complete', (event) => {
      if (!this.activeTurn) return;
      const tc = this.activeTurn.toolCalls.find(t => t.id === event.data.toolCallId);
      if (tc) {
        (tc as any).result = event.data.result;
        (tc as any).status = 'complete';
      }
      this.onTurnUpdate.fire(this.activeTurn);
    });

    this.session.on('assistant.turn_end', (event) => {
      if (this.activeTurn) {
        (this.activeTurn as any).status = 'complete';
        this.onTurnUpdate.fire(this.activeTurn);
        this.activeTurn = undefined;
      }
    });

    this.session.on('session.error', (event) => {
      if (this.activeTurn) {
        (this.activeTurn as any).status = 'error';
        this.onTurnUpdate.fire(this.activeTurn);
        this.activeTurn = undefined;
      }
      this.onStatusChange.fire('error');
    });
  }

  private appendContent(turn: Turn, delta: string): void {
    const last = turn.content[turn.content.length - 1];
    if (last && last.kind === 'text') {
      (last as any).value += delta;
    } else {
      turn.content.push({ kind: 'text', value: delta });
    }
  }

  async send(options: {
    prompt: string;
    attachments?: Attachment[];
    mode?: 'interactive' | 'autopilot' | 'plan';
  }): Promise<void> {
    if (options.mode) {
      await this.session.rpc.mode.set({ mode: options.mode });
    }
    await this.session.send({
      prompt: options.prompt,
      attachments: options.attachments ?? [],
    });
  }

  dispose(): void {
    this.session.disconnect();
  }
}
```

### 5b. Ref-Counted Session Management

Multiple consumers (chat widget, file watcher, MCP handler) may need the same session simultaneously. Ref-counting prevents premature disposal.

```typescript
class SharedSession {
  private refCount = 0;
  private idleTimer: NodeJS.Timeout | undefined;
  private static readonly IDLE_TIMEOUT_MS = 300_000; // 5 minutes (matches SESSION_SHUTDOWN_TIMEOUT_MS)

  constructor(
    readonly sessionId: string,
    readonly wrapper: CopilotSessionWrapper,
    private readonly onDispose: (id: string) => void,
  ) {}

  acquire(): CopilotSessionWrapper {
    this.refCount++;
    clearTimeout(this.idleTimer);
    return this.wrapper;
  }

  release(): void {
    this.refCount--;
    if (this.refCount <= 0) {
      this.refCount = 0;
      this.idleTimer = setTimeout(() => {
        this.wrapper.disconnect();
        this.onDispose(this.sessionId);
      }, SharedSession.IDLE_TIMEOUT_MS);
    }
  }
}

class SessionManager {
  private sessions = new Map<string, SharedSession>();

  create(sessionId: string, sdkSession: CopilotSession): SharedSession {
    const wrapper = new CopilotSessionWrapper(sdkSession);
    const shared = new SharedSession(sessionId, wrapper, (id) => this.sessions.delete(id));
    this.sessions.set(sessionId, shared);
    return shared;
  }

  get(sessionId: string): SharedSession | undefined {
    return this.sessions.get(sessionId);
  }

  async fork(sourceId: string, newId: string): Promise<SharedSession> {
    const source = this.sessions.get(sourceId);
    if (!source) throw new Error(`Session ${sourceId} not found`);
    // Create a new SDK session with the source's history as context
    const newSdkSession = await createSdkSession({ forkFrom: sourceId });
    return this.create(newId, newSdkSession);
  }

  delete(sessionId: string): void {
    const session = this.sessions.get(sessionId);
    if (session) {
      session.wrapper.disconnect();
      this.sessions.delete(sessionId);
    }
  }

  rename(sessionId: string, newTitle: string): void {
    const session = this.sessions.get(sessionId);
    if (session) {
      // Title update is handled separately via session metadata, not via send()
      // Directly update the session metadata
    }
  }
}
```

### 5c. History Reconstruction from Event Logs

CLI sessions persist their event stream to `events.jsonl`. Reconstruction reads this file and replays events into turn objects.

```typescript
interface SessionEventLine {
  id: string;
  type: string;
  timestamp: string; // ISO 8601
  parentId?: string;
  data: Record<string, unknown>;
}

async function reconstructHistory(eventsPath: string): Promise<Turn[]> {
  const turns: Turn[] = [];
  let currentTurn: Turn | undefined;

  const fileContent = await fs.promises.readFile(eventsPath, 'utf-8');
  const lines = fileContent.split('\n').filter(line => line.trim().length > 0);

  for (const line of lines) {
    let event: SessionEventLine;
    try {
      event = JSON.parse(line);
    } catch {
      // Partial or corrupt line — skip gracefully
      continue;
    }

    switch (event.type) {
      case 'user.message':
        turns.push({
          id: event.data.turnId as string,
          role: 'user',
          content: [{ kind: 'text', value: event.data.prompt as string }],
          toolCalls: [],
          status: 'complete',
          timestamp: event.timestamp,
        });
        break;

      case 'assistant.turn_start':
        currentTurn = {
          id: event.data.turnId as string,
          role: 'assistant',
          content: [],
          toolCalls: [],
          status: 'streaming',
          timestamp: event.timestamp,
        };
        turns.push(currentTurn);
        break;

      case 'assistant.message_delta':
        if (currentTurn) {
          const last = currentTurn.content[currentTurn.content.length - 1];
          if (last?.kind === 'text') {
            (last as any).value += event.data.delta;
          } else {
            currentTurn.content.push({ kind: 'text', value: event.data.delta as string });
          }
        }
        break;

      case 'assistant.turn_end':
        if (currentTurn) {
          (currentTurn as any).status = 'complete';
          currentTurn = undefined;
        }
        break;

      case 'session.error':
        if (currentTurn) {
          (currentTurn as any).status = 'error';
          currentTurn = undefined;
        }
        break;
    }
  }

  return turns;
}
```

**Optimization: Early termination.** When reconstructing only to read the session title, scan for the first `update_session_name` event and stop:

```typescript
async function findFirstEvent(eventsPath: string, eventType: string): Promise<SessionEventLine | undefined> {
  const stream = fs.createReadStream(eventsPath, { encoding: 'utf-8' });
  let buffer = '';
  for await (const chunk of stream) {
    buffer += chunk;
    const lines = buffer.split('\n');
    buffer = lines.pop() ?? '';
    for (const line of lines) {
      if (!line.trim()) continue;
      try {
        const event: SessionEventLine = JSON.parse(line);
        if (event.type === eventType) {
          stream.destroy();
          return event;
        }
      } catch { continue; }
    }
  }
  return undefined;
}
```

---

## Step 6: Implement the Session List UI

### 6a. Tree/List Widget Structure

Sessions are grouped by time. The grouping is computed from `lastActivityAt` on each session summary.

```typescript
type SessionGroup = 'pinned' | 'today' | 'yesterday' | 'week' | 'older' | 'archived';

function groupSessions(sessions: SessionSummary[]): Map<SessionGroup, SessionSummary[]> {
  const now = new Date();
  const todayStart = new Date(now.getFullYear(), now.getMonth(), now.getDate()).getTime();
  const yesterdayStart = todayStart - 86_400_000;
  const weekStart = todayStart - 7 * 86_400_000;

  const groups = new Map<SessionGroup, SessionSummary[]>();
  for (const group of ['pinned', 'today', 'yesterday', 'week', 'older', 'archived'] as SessionGroup[]) {
    groups.set(group, []);
  }

  for (const session of sessions) {
    if (session.isPinned) {
      groups.get('pinned')!.push(session);
    } else if (session.isArchived) {
      groups.get('archived')!.push(session);
    } else if (session.lastActivityAt >= todayStart) {
      groups.get('today')!.push(session);
    } else if (session.lastActivityAt >= yesterdayStart) {
      groups.get('yesterday')!.push(session);
    } else if (session.lastActivityAt >= weekStart) {
      groups.get('week')!.push(session);
    } else {
      groups.get('older')!.push(session);
    }
  }

  // Sort within each group: most recent first
  for (const items of groups.values()) {
    items.sort((a, b) => b.lastActivityAt - a.lastActivityAt);
  }

  return groups;
}
```

Each list item renders:

```
┌──────────────────────────────────────────────┐
│ [Provider Icon] Session Title          2m ago │
│ [Status Icon]   3 file changes               │
└──────────────────────────────────────────────┘
```

### 6b. Data Source

```typescript
class SessionListDataSource {
  private sessions: SessionSummary[] = [];
  private readonly onChange = new EventEmitter<void>();

  constructor(private readonly providers: SessionProvider[]) {
    // Subscribe to all providers for live updates
    for (const provider of providers) {
      provider.onSessionsChanged(() => this.refresh());
    }
  }

  async refresh(): Promise<void> {
    const allSessions: SessionSummary[] = [];
    for (const provider of this.providers) {
      const sessions = await provider.listSessions();
      allSessions.push(...sessions);
    }
    this.sessions = allSessions;
    this.onChange.fire();
  }

  getGrouped(filterQuery?: string): Map<SessionGroup, SessionSummary[]> {
    let filtered = this.sessions;
    if (filterQuery) {
      const q = filterQuery.toLowerCase();
      filtered = filtered.filter(s =>
        s.title.toLowerCase().includes(q) ||
        s.providerName.toLowerCase().includes(q)
      );
    }
    return groupSessions(filtered);
  }
}
```

### 6c. Session Type Metadata

```typescript
interface SessionTypeDescriptor {
  readonly id: string;              // matches URI scheme
  readonly name: string;            // e.g. "Copilot CLI"
  readonly icon: string;            // icon identifier
  readonly description: string;
  readonly isFirstParty: boolean;
  readonly canContinueIn: string[]; // target type IDs this type can delegate to
}

const SESSION_TYPES: SessionTypeDescriptor[] = [
  {
    id: 'copilotcli',
    name: 'Copilot CLI',
    icon: 'terminal',
    description: 'Background agent sessions from the Copilot CLI',
    isFirstParty: true,
    canContinueIn: ['copilotlocal', 'copilotcloud'],
  },
  {
    id: 'copilotlocal',
    name: 'Copilot Chat',
    icon: 'comment-discussion',
    description: 'Local IDE chat sessions',
    isFirstParty: true,
    canContinueIn: ['copilotcli', 'copilotcloud'],
  },
];

function getSessionType(uri: SessionUri): SessionTypeDescriptor | undefined {
  const scheme = new URL(uri).protocol.replace(':', '');
  return SESSION_TYPES.find(t => t.id === scheme);
}
```

---

## Step 7: Implement Session Loading

### 7a. Opening a Session from the List

```typescript
async function openSession(uri: SessionUri, target: 'sidebar' | 'editor'): Promise<void> {
  // 1. Mark as read
  await markSessionRead(uri);

  // 2. Determine session type and activate the corresponding provider
  const sessionType = getSessionType(uri);
  if (!sessionType) throw new Error(`Unknown session type for URI: ${uri}`);
  const provider = getProviderForType(sessionType.id);
  await provider.activate();

  // 3. Load session model (or reuse if already loaded)
  const shared = sessionManager.get(extractSessionId(uri));
  let wrapper: CopilotSessionWrapper;
  if (shared) {
    wrapper = shared.acquire();
  } else {
    // Load from persistence
    const sessionId = extractSessionId(uri);
    const eventsPath = getEventsPath(sessionId);
    const turns = await reconstructHistory(eventsPath);
    const sdkSession = await createSdkSession({ sessionId, history: turns });
    const newShared = sessionManager.create(sessionId, sdkSession);
    wrapper = newShared.acquire();
  }

  // 4. Show in chat widget at the requested location
  if (target === 'sidebar') {
    sidebarChatWidget.setSession(wrapper);
  } else {
    editorChatWidget.open(wrapper);
  }
}

function extractSessionId(uri: SessionUri): string {
  return new URL(uri).pathname.replace(/^\//, '');
}
```

### 7b. Cross-Process Session Opening

When an external CLI process creates a new session, the IDE must detect and open it. This uses IPC (or a custom message channel) for explicit requests.

```typescript
// IPC handler: external process requests opening a session
ipcMain.on('open-session', (event, uri: string) => {
  openSession(uri, 'sidebar');
});

// File watcher detection (see Step 10): new events.jsonl file appears
fileWatcher.onNewSession((sessionId) => {
  const uri = `copilotcli:///${sessionId}`;
  // Show notification or auto-open based on user preference
  showSessionNotification(uri);
});
```

### 7c. New Session Creation

```typescript
function createNewSession(type: string): SessionUri {
  const sessionId = `untitled-${randomUUID()}`;
  const uri = `${type}:///${sessionId}`;
  return uri;
}

// Usage
const uri = createNewSession('copilotcli');
// → "copilotcli:///untitled-a1b2c3d4-..."
```

---

## Step 8: Implement Session Continuation

### 8a. Delegation Flow

"Continue in..." transfers context from one session type to another — for example, from a local chat session to a CLI background agent.

```typescript
async function continueSessionIn(
  sourceUri: SessionUri,
  targetType: string,
): Promise<SessionUri> {
  // 1. Validate delegation is permitted
  const sourceType = getSessionType(sourceUri);
  if (!sourceType?.canContinueIn.includes(targetType)) {
    throw new Error(`Cannot continue ${sourceType?.id} session in ${targetType}`);
  }

  // 2. Extract repository information
  const repoInfo = await extractRepoInfo(sourceUri);

  // 3. Create new session of target type
  const targetUri = createNewSession(targetType);

  // 4. Build context summary from source session
  const sourceSession = sessionManager.get(extractSessionId(sourceUri));
  const contextSummary = buildContextSummary(sourceSession);

  // 5. Initialize target session with context
  const targetSdkSession = await createSdkSession({
    sessionId: extractSessionId(targetUri),
    initialContext: contextSummary,
    repository: repoInfo,
  });
  sessionManager.create(extractSessionId(targetUri), targetSdkSession);

  return targetUri;
}

async function extractRepoInfo(uri: SessionUri): Promise<RepoInfo | undefined> {
  // Try multiple sources in priority order:
  // 1. Git remote from workspace folders
  // 2. Session metadata
  // 3. Package.json repository field
  const sessionId = extractSessionId(uri);
  const metadata = await loadSessionMetadata(sessionId);

  for (const folder of metadata?.workspaceFolders ?? []) {
    const nwo = await getGitRemoteNwo(folder);
    if (nwo) return { nwo, rootPath: folder };
  }

  return undefined;
}

interface RepoInfo {
  nwo: string;       // "owner/repo" (name with owner)
  rootPath: string;
}
```

### 8b. Delegation Rules

```typescript
const DELEGATION_RULES: Record<string, string[]> = {
  copilotlocal: ['copilotcli', 'copilotcloud'],
  copilotcli: ['copilotlocal', 'copilotcloud'],
  copilotcloud: ['copilotlocal', 'copilotcli'],  // CLI delegation requires git repo (NWO)
};

function canDelegateTo(sourceType: string, targetType: string): boolean {
  return DELEGATION_RULES[sourceType]?.includes(targetType) ?? false;
}
```

Constraints:
- Delegating to a cloud session requires a git repository (NWO must be resolvable).
- Delegating preserves the conversation summary, not the full turn history.
- The source session remains open and browsable after delegation.

---

## Step 9: Implement CLI-Specific Rendering Behaviors

CLI sessions differ from local IDE sessions in several rendering aspects. Implement these overrides based on session type.

```typescript
function getRendererOverrides(sessionType: string): RendererOverrides {
  if (sessionType !== 'copilotcli') return {};

  return {
    // 9a. Suppress the IDE's own file-changes summary panel.
    //     CLI sessions report file changes inline via tool call results.
    suppressFileChangesSummary: true,

    // 9b. Working set: return empty entries.
    //     File changes are tracked at the session level, not per-turn.
    getWorkingSetEntries: () => [],

    // 9c. Auto-accept streaming edits without showing diff approval UI.
    //     The CLI process manages its own acceptance flow.
    autoAcceptStreamingEdits: true,

    // 9d. Hide untitled CLI sessions from auxiliary views (e.g. "Recent" panel).
    //     They appear only when they have meaningful content.
    filterFromAuxiliaryViews: (session: SessionSummary) =>
      session.title.startsWith('untitled-'),

    // 9e. Status display: show "Background Agent" for active CLI sessions.
    getStatusLabel: (status: SessionStatus) =>
      status === 'active' ? 'Background Agent' : defaultStatusLabel(status),
  };
}

interface RendererOverrides {
  suppressFileChangesSummary?: boolean;
  getWorkingSetEntries?: () => WorkingSetEntry[];
  autoAcceptStreamingEdits?: boolean;
  filterFromAuxiliaryViews?: (session: SessionSummary) => boolean;
  getStatusLabel?: (status: SessionStatus) => string;
}
```

---

## Step 10: Implement File System Watcher

The file system watcher detects when external CLI processes create, update, or remove session event logs. This is the cross-process synchronization mechanism.

### 10a. Watch Configuration

```typescript
// Use your platform's file system watcher. VS Code uses its built-in
// FileSystemWatcher; external projects may use chokidar or node:fs.watch.
import { watch, FSWatcher } from 'chokidar'; // or equivalent

const SESSION_STATE_GLOB = path.join(os.homedir(), '.copilot', 'session-state', '**', '*.jsonl');
const THROTTLE_MS = 500; // VS Code uses 500ms ThrottledDelayer for file watcher events

class SessionFileWatcher {
  private watcher: FSWatcher;
  private throttleTimers = new Map<string, NodeJS.Timeout>();
  private readonly onSessionCreated = new EventEmitter<string>();
  private readonly onSessionUpdated = new EventEmitter<string>();
  private readonly onSessionDeleted = new EventEmitter<string>();

  start(): void {
    this.watcher = watch(SESSION_STATE_GLOB, {
      ignoreInitial: true,
      awaitWriteFinish: { stabilityThreshold: 200 },
    });

    this.watcher.on('add', (filePath) => {
      const sessionId = this.extractSessionId(filePath);
      if (sessionId) this.onSessionCreated.fire(sessionId);
    });

    this.watcher.on('change', (filePath) => {
      const sessionId = this.extractSessionId(filePath);
      if (!sessionId) return;

      // Throttle rapid writes
      clearTimeout(this.throttleTimers.get(sessionId));
      this.throttleTimers.set(sessionId, setTimeout(() => {
        this.throttleTimers.delete(sessionId);
        this.onSessionUpdated.fire(sessionId);
      }, THROTTLE_MS));
    });

    this.watcher.on('unlink', (filePath) => {
      const sessionId = this.extractSessionId(filePath);
      if (sessionId) this.onSessionDeleted.fire(sessionId);
    });
  }

  // 10b. Extract session ID from event file path
  private extractSessionId(filePath: string): string | undefined {
    // Expected path: ~/.copilot/session-state/<sessionId>/events.jsonl
    const parts = filePath.split(path.sep);
    const stateIdx = parts.indexOf('session-state');
    if (stateIdx >= 0 && stateIdx + 1 < parts.length) {
      return parts[stateIdx + 1];
    }
    return undefined;
  }

  stop(): void {
    this.watcher?.close();
    for (const timer of this.throttleTimers.values()) {
      clearTimeout(timer);
    }
    this.throttleTimers.clear();
  }
}
```

### 10c. Integration with Session List

```typescript
// Wire the file watcher to the session list data source
const watcher = new SessionFileWatcher();
watcher.onSessionCreated.on((sessionId) => {
  sessionListDataSource.refresh();
});
watcher.onSessionUpdated.on((sessionId) => {
  sessionListDataSource.refresh();
});
watcher.onSessionDeleted.on((sessionId) => {
  sessionListDataSource.refresh();
});
watcher.start();
```

---

## Step 11: Implement the Agent Host State Protocol (Optional)

This step is necessary only if you support multiple connected clients viewing or interacting with the same session simultaneously (e.g., desktop IDE + web IDE, or multiple windows).

### 11a. State Model

```typescript
// Root state: list of agents with model information
interface AgentHostRootState {
  agents: AgentInfo[];
  defaultModel: ModelInfo;
}

interface AgentInfo {
  id: string;
  name: string;
  description: string;
  modelId: string;
}

interface ModelInfo {
  id: string;
  name: string;
  provider: string;
}

// Session state: full detail for a subscribed session
interface AgentHostSessionState {
  summary: SessionSummary;
  lifecycle: SessionLifecycle;
  turns: TurnReference[];    // lightweight references; content fetched separately
  activeTurn: TurnReference | undefined;
  tools: ToolDefinition[];
}

// Turn references contain metadata but not full content
interface TurnReference {
  id: string;
  role: 'user' | 'assistant';
  status: TurnStatus;
  timestamp: number;
  contentUri: string;        // fetch full content via this URI
}
```

### 11b. Communication Protocol

JSON-RPC 2.0 over MessagePort (desktop, same-machine) or WebSocket (remote, cross-machine).

```typescript
// Client → Server
interface AgentHostRequest {
  jsonrpc: '2.0';
  id: number;
  method: AgentHostMethod;
  params: Record<string, unknown>;
}

type AgentHostMethod =
  | 'createSession'
  | 'listSessions'
  | 'subscribe'
  | 'unsubscribe'
  | 'dispatchAction'  // client → server: state mutation (notification)
  | 'fetchTurns'
  | 'resourceRead'    // fetch large content by URI
  | 'sendMessage'
  | 'cancelTurn';

// Server → Client (notifications)
interface AgentHostNotification {
  jsonrpc: '2.0';
  method: 'state/update';
  params: {
    uri: string;
    envelope: ActionEnvelope;
  };
}

// Subscription model: URI-based
// Subscribe to "session://<id>" to receive state updates for that session
// Subscribe to "sessions://list" to receive list-level updates
```

### 11c. Command Implementations

```typescript
class AgentHostServer {
  private subscriptions = new Map<string, Set<ClientConnection>>();

  async handleRequest(client: ClientConnection, request: AgentHostRequest): Promise<unknown> {
    switch (request.method) {
      case 'createSession':
        return this.createSession(request.params as { type: string });

      case 'listSessions':
        return this.listSessions();

      case 'subscribe': {
        const uri = request.params.uri as string;
        if (!this.subscriptions.has(uri)) {
          this.subscriptions.set(uri, new Set());
        }
        this.subscriptions.get(uri)!.add(client);
        // Return current state snapshot
        return this.getStateSnapshot(uri);
      }

      case 'unsubscribe': {
        const uri = request.params.uri as string;
        this.subscriptions.get(uri)?.delete(client);
        return null;
      }

      case 'fetchTurns': {
        const { sessionId, offset, limit } = request.params as {
          sessionId: string; offset: number; limit: number;
        };
        return this.fetchTurns(sessionId, offset, limit);
      }

      case 'resourceRead': {
        const { uri } = request.params as { uri: string };
        return this.resourceRead(uri);
      }

      case 'dispatchAction': {
        // Notification: client-originated state mutation
        const { action, clientId, clientSeq } = request.params as {
          action: SessionAction; clientId: string; clientSeq: number;
        };
        this.dispatchAction(action, { clientId, clientSeq });
        return null;
      }

      case 'sendMessage': {
        const { sessionId, content } = request.params as {
          sessionId: string; content: string;
        };
        return this.sendMessage(sessionId, content);
      }

      case 'cancelTurn': {
        const { sessionId, turnId } = request.params as {
          sessionId: string; turnId: string;
        };
        return this.cancelTurn(sessionId, turnId);
      }

      default:
        throw new Error(`Unknown method: ${request.method}`);
    }
  }

  // Broadcast state change to all subscribers of a URI
  private broadcastUpdate(uri: string, envelope: ActionEnvelope): void {
    const subscribers = this.subscriptions.get(uri);
    if (!subscribers) return;
    const notification: AgentHostNotification = {
      jsonrpc: '2.0',
      method: 'state/update',
      params: { uri, envelope },
    };
    for (const client of subscribers) {
      client.send(notification);
    }
  }
}
```

### 11d. Content References

Large content (assistant responses with extensive code, file diffs) is stored by URI reference rather than inlined in state updates. This prevents payload bloat in notifications.

```typescript
// Turn reference in a state update — lightweight
{
  id: 'turn-42',
  role: 'assistant',
  status: 'complete',
  timestamp: 1700000000000,
  contentUri: 'content://session/abc123/turn/turn-42'
}

// Client fetches full content separately when the turn is visible
const content = await agentHostClient.resourceRead('content://session/abc123/turn/turn-42');
// → { kind: 'text', value: '...(full response text)...' }
```

---

## Architecture Decisions to Consider

| Decision | VS Code's Choice | Alternatives | Trade-offs |
|---|---|---|---|
| Transport for external CLI | Unix socket + MCP | Named pipes, TCP, gRPC | Unix sockets are fast and local-only; TCP requires auth hardening; gRPC adds a build dependency |
| Session persistence | SQLite per session | Single database, flat files | Per-session isolation simplifies deletion and prevents cross-session corruption; single DB scales better for queries across sessions |
| State synchronization | Redux-like actions | CRDT, operational transform | Actions are simple and debuggable; CRDTs handle true concurrent edits but add significant complexity |
| Discovery mechanism | Lock files | mDNS, registry, env variables | Lock files are portable and require no daemon; mDNS is fragile across networks; env variables require parent-child process relationship |
| Authentication | Nonce in lock file | Certificates, tokens, OAuth | Nonce is zero-config; security depends on file permissions (sufficient for local-only); certificates are overkill for same-machine IPC |
| Session identification | URI scheme encodes type | Separate type field | URI scheme enables routing via standard URL parsing; separate field requires carrying a tuple everywhere |
| UI integration | Extension point model | Hardcoded, plugin API | Extension points allow third-party session providers; hardcoded is simpler for single-provider applications |
| File edit storage | Before/after BLOBs | Diffs, patches | BLOBs enable instant undo without applying a reverse diff; diffs save storage but require a patch engine |

---

## Common Pitfalls

### 1. Lock File Race Conditions

**Problem:** Two IDE instances starting simultaneously may both attempt to clean the same stale lock file, or an IDE crash may leave an orphaned lock.

**Mitigation:**
- Write to a temp file, then atomically rename (`fs.renameSync`). This prevents partial reads.
- Check PID liveness with `process.kill(pid, 0)` before removing a lock.
- Use a unique filename (`<uuid>.lock`) to prevent collisions between instances.

### 2. Event Log Corruption

**Problem:** A crash during a write to `events.jsonl` leaves a partial JSON line.

**Mitigation:**
- Each line must be independently parseable. Wrap `JSON.parse` in a try/catch per line.
- Never rely on the last line being complete.
- Append with `\n` *after* the JSON, not before, so truncation produces an incomplete last line rather than a blank line followed by partial data.

### 3. Session Multiplexing Conflicts

**Problem:** A CLI process sends a duplicate `initialize` request for an already-active session.

**Mitigation:**
- Return HTTP 409 if a transport already exists for the session ID.
- The CLI should interpret 409 as "reconnect to existing session" and skip initialization.

### 4. Idle Session Cleanup

**Problem:** A session is disposed while a background operation (e.g., file write) is still pending.

**Mitigation:**
- Use ref-counting. Every consumer that holds a reference must explicitly `release()`.
- The idle timeout (300 seconds) only starts when `refCount` reaches zero.
- Log a warning if `release()` is called more times than `acquire()`.

### 5. Cross-Platform Socket Paths

**Problem:** Unix domain sockets work on macOS and Linux. Windows requires named pipes.

**Mitigation:**
- Abstract the transport behind an interface:
  ```typescript
  interface TransportConfig {
    scheme: 'unix' | 'pipe';
    path: string;
  }
  function createTransportConfig(): TransportConfig {
    if (process.platform === 'win32') {
      return { scheme: 'pipe', path: `\\\\.\\pipe\\ide-mcp-${randomUUID()}` };
    }
    const dir = fs.mkdtempSync(path.join(os.tmpdir(), 'ide-mcp-'));
    fs.chmodSync(dir, 0o700);
    return { scheme: 'unix', path: path.join(dir, 'mcp.sock') };
  }
  ```
- Store the `scheme` in the lock file so the CLI knows which transport to use.

### 6. File Watcher Throttling

**Problem:** Rapid writes to `events.jsonl` during an active CLI session produce dozens of change events per second, causing UI thrashing.

**Mitigation:**
- Throttle change events with a 200–500ms window. Only the last event in a window fires the refresh. VS Code uses a 500ms `ThrottledDelayer` for file watcher events and a separate `RunOnceScheduler` for summary notification batching.
- Use `awaitWriteFinish` (if available in your file watcher library) to delay until the file is stable.

### 7. Content Size Management

**Problem:** A single assistant response may contain megabytes of code output, bloating state notifications.

**Mitigation:**
- Use `ContentRef` (URI references) for any content exceeding a threshold (e.g., 4 KB).
- Inline small content directly in state updates.
- Fetch large content on demand when the turn scrolls into view.

### 8. Auto-Accept Timing

**Problem:** Auto-accepting a streaming edit before it is fully written produces a partial file.

**Mitigation:**
- Track the edit stream's completion status. Auto-accept only after the `assistant.turn_end` event for the tool call that produced the edit.
- For multi-file edits within a single tool call, wait for all files to be written before accepting any.

---

## Testing Strategy

### Unit Tests

| Layer | What to Test | Technique |
|---|---|---|
| Persistence | Schema migration, CRUD operations, ref-counting | In-memory SQLite (`':memory:'`). VS Code uses `@vscode/sqlite3` with a `TestableSessionDatabase` that supports `ejectDb()`/`fromDb()` for reopen tests. |
| Session model | Reducer correctness, observable emissions, turn construction | Pure function tests |
| Lock file | Write, read, stale detection, atomic rename | Temp directory fixtures |
| Event reconstruction | Full replay, partial lines, early termination | Fixture `.jsonl` files (see `src/vs/platform/agentHost/test/node/test-cases/`) |
| Session grouping | Time-based bucketing, pinned/archived handling | Frozen clock (`Date.now` mock) |

### Integration Tests

> **Note:** VS Code core tests use `suite`/`test` with Node's `assert` module. The Copilot extension uses vitest (`describe`/`it`/`expect`). The examples below use vitest style for readability; adapt to your project's test framework.

```typescript
describe('MCP Connection Lifecycle', () => {
  it('startup → connect → tool call → disconnect', async () => {
    // 1. Start MCP server
    const { socketPath, nonce, dispose } = startMcpServer(mockContext);

    // 2. Connect as CLI client
    const client = new McpClient(socketPath, nonce);
    await client.initialize();

    // 3. Call a tool
    const info = await client.callTool('get_vscode_info', {});
    expect(JSON.parse(info)).toHaveProperty('name');

    // 4. Disconnect
    await client.close();
    dispose();
  });
});
```

### End-to-End Tests

```
1. Create session            → verify URI returned, session appears in list
2. Send message              → verify user turn created, streaming response received
3. Receive tool call         → verify tool call rendered, result returned to SDK
4. Response completes        → verify assistant turn status is 'complete'
5. Close and reload          → verify session loads from persistence with full history
6. Open from list            → verify chat widget displays reconstructed turns
7. Continue in another type  → verify new session created with context, source still browsable
```

### File Watcher Tests

```typescript
describe('Cross-Process Sync', () => {
  it('detects new session from external process', async () => {
    const watcher = new SessionFileWatcher();
    const created = new Promise<string>(resolve => watcher.onSessionCreated.on(resolve));
    watcher.start();

    // Simulate external CLI process creating a session
    const sessionDir = path.join(os.homedir(), '.copilot', 'session-state', 'test-session');
    fs.mkdirSync(sessionDir, { recursive: true });
    fs.writeFileSync(path.join(sessionDir, 'events.jsonl'), '{"id":"evt-1","type":"user.message","timestamp":"2025-01-01T00:00:00.000Z","data":{}}\n');

    const sessionId = await created;
    expect(sessionId).toBe('test-session');

    watcher.stop();
    fs.rmSync(sessionDir, { recursive: true });
  });
});
```

### Lock File Tests

```typescript
describe('Lock File Discovery', () => {
  it('creates, discovers, and cleans stale locks', () => {
    // Write a lock file
    const lockPath = writeLockFile({
      socketPath: '/tmp/test.sock',
      scheme: 'unix',
      headers: { Authorization: 'Nonce test-nonce' },
      pid: process.pid,
      ideName: 'Test IDE',
      timestamp: Date.now(),
      workspaceFolders: ['/home/user/project'],
      isTrusted: true,
    });

    // Discover it
    const locks = discoverLockFiles();
    expect(locks).toContainEqual(expect.objectContaining({ pid: process.pid }));

    // Simulate stale lock (PID that does not exist)
    const staleLockPath = writeLockFile({
      ...locks[0],
      pid: 999999, // almost certainly not running
    });

    cleanStaleLockFiles();

    // Stale lock should be removed; live lock should remain
    expect(fs.existsSync(staleLockPath)).toBe(false);
    expect(fs.existsSync(lockPath)).toBe(true);

    // Cleanup
    fs.unlinkSync(lockPath);
  });
});
```
