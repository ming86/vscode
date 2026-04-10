# Transport Layers

## Overview

The Copilot CLI session system supports multiple transport mechanisms depending on the integration pathway and runtime environment. This document catalogs each transport, its configuration, and when it is used.

## Transport Summary

| Transport | Direction | Used By | Environment |
|-----------|-----------|---------|-------------|
| In-process (direct) | Extension ↔ SDK | `ICopilotCLISessionService` | All |
| MessagePort | Renderer ↔ Agent Host | VS Code Desktop | Electron |
| WebSocket | Renderer ↔ Agent Host | VS Code Remote / Web | SSH, WSL, Browser |
| Unix Domain Socket | CLI ↔ IDE | ACP Server Mode | macOS, Linux |
| TCP Socket | CLI ↔ IDE | ACP Server Mode | All platforms |
| stdio | SDK ↔ Child Process | Spawned sessions | All |

## In-Process SDK (Primary)

The extension imports `@github/copilot/sdk` directly and calls methods synchronously within the extension host process.

```
┌──────────────────────────────────────────────┐
│           VS Code Extension Host              │
│                                               │
│  CopilotCLISessionService                     │
│       │                                       │
│       ├── copilotCLISDK.getPackage()          │
│       │   └── import('@github/copilot/sdk')   │
│       │                                       │
│       ├── LocalSessionManager (SDK)           │
│       │   ├── createSession()                 │
│       │   ├── getSession()                    │
│       │   ├── listSessions()                  │
│       │   └── closeSession()                  │
│       │                                       │
│       └── Session (SDK)                       │
│           ├── send()                          │
│           ├── on(event, handler)              │
│           └── abort()                         │
└──────────────────────────────────────────────┘
```

**Configuration:**
```typescript
const sessionManager = new internal.LocalSessionManager({
    telemetryService: new internal.NoopTelemetryService(),
    flushDebounceMs: undefined,
    settings: undefined,
    version: undefined,
});
```

**Characteristics:**
- Zero serialization overhead
- Direct access to SDK internals
- Shares process memory with extension
- Session state persisted to `~/.copilot/session-state/`

## MessagePort (Desktop Agent Host)

Used when the Agent Host runs as an Electron utility process:

```
┌───────────────────┐   MessagePort   ┌──────────────────┐
│  Renderer Process  │ ◄────────────► │  Utility Process   │
│  (VS Code UI)      │                │  (Agent Host)      │
│                    │                │                    │
│  AgentHostClient   │  JSON-RPC 2.0  │  CopilotClient     │
│                    │  (binary msgs) │                    │
└───────────────────┘                └──────────────────┘
```

**Setup:**
1. VS Code main process spawns utility process
2. Electron `MessagePort` pair created
3. One port transferred to renderer, one to utility
4. JSON-RPC messages sent as structured clone (no serialization)

## WebSocket (Remote Agent Host)

Used when VS Code connects to a remote host (SSH, WSL, Codespaces):

```
┌───────────────────┐   WebSocket    ┌──────────────────┐
│  Browser / Client  │ ◄──────────► │  Remote Host       │
│                    │              │                    │
│  AgentHostClient   │  JSON-RPC    │  Agent Host Server │
│                    │  over WS     │  (node process)    │
└───────────────────┘              └──────────────────┘
```

**Characteristics:**
- Full-duplex over single TCP connection
- Works across network boundaries
- Reconnection handling built into VS Code's remote architecture

## Unix Domain Socket (ACP Server Mode)

The CLI binary can expose a JSON-RPC 2.0 server over Unix domain sockets:

```
┌───────────────────┐   Unix Socket   ┌──────────────────┐
│  VS Code Extension │ ◄────────────► │  copilot-cli      │
│                    │                │  (--acp flag)      │
│  reads lock file   │  JSON-RPC 2.0  │  ACP server       │
│  connects to sock  │  + nonce auth  │                    │
└───────────────────┘                └──────────────────┘
```

### Lock File Discovery

```
~/.copilot/ide/{uuid}.lock
```

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
    "workspaceFolders": ["/Users/ming/GitRepos/project"],
    "isTrusted": true
}
```

### Connection Flow

```
1. IDE starts → creates lock file in ~/.copilot/ide/
2. CLI (--acp) → scans lock files, discovers IDE socket
3. CLI connects → sends Authorization header with nonce
4. IDE validates → nonce matches lock file
5. Bidirectional JSON-RPC session established
```

### Multiple Connections

Multiple lock files can exist simultaneously:
- Each IDE window creates its own lock file
- CLI can connect to multiple IDEs
- Stale lock files detected via PID and timestamp

## TCP Socket

> **Note:** TCP transport (`scheme: "tcp"`) is inferred from the code but unverified in practice. All observed lock files use `scheme: "unix"`. The following is speculative.

Alternative to Unix sockets for cross-platform compatibility:

```json
{
    "scheme": "tcp",
    "socketPath": "127.0.0.1:52431",
    "headers": { "Authorization": "Nonce {uuid}" }
}
```

## stdio (Child Process Mode)

The SDK supports spawning as a child process with stdio as transport:

```
┌───────────────────┐   stdin/stdout  ┌──────────────────┐
│  Parent Process    │ ◄────────────► │  SDK Child Process │
│  (VS Code ext)     │                │                    │
│                    │  JSON-RPC 2.0  │  isChildProcess:   │
│                    │  line-delimited│  true              │
└───────────────────┘                └──────────────────┘
```

### Configuration

```typescript
import { joinSession } from "@github/copilot-sdk/extension";

// joinSession reads SESSION_ID from process.env internally
// and creates a CopilotClient({ isChildProcess: true }) internally.
// The caller provides optional config (tools, permission handlers, etc.).
const session = await joinSession({ tools: [myTool] });
```

`joinSession(config?: JoinSessionConfig)` where `JoinSessionConfig` extends `ResumeSessionConfig` (minus `onPermissionRequest`, which is optional with a default no-op handler).

**Characteristics:**
- No network setup required
- Parent process manages lifecycle
- Used for process isolation without network overhead

## Transport Selection Guide

```
Is this the VS Code extension's primary session integration?
├── YES → In-process SDK (direct function calls)
│
├── Is this VS Code platform-level integration?
│   ├── Desktop (Electron) → MessagePort
│   └── Remote/Web → WebSocket
│
├── Is the CLI already running in a terminal?
│   ├── Same machine → Unix Domain Socket (via lock file)
│   └── Remote/cross-platform → TCP Socket
│
└── Need process isolation without network?
    └── stdio (child process mode)
```

## SDK Protocol Version

All transports use the same JSON-RPC 2.0 message format:

```typescript
// sdk-protocol-version.json
{ "version": 3 }
```

### Message Format

The SDK uses dot-separated, hierarchical method names. Server-scoped methods (no session required) include `ping`, `models.list`, `tools.list`, `account.getQuota`, `mcp.config.*`, and `sessionFs.setProvider`. Session-scoped methods are prefixed with `session.` (e.g., `session.model.getCurrent`, `session.plan.read`). Notifications use `session.event`.

```json
// Request (server-scoped)
{"jsonrpc":"2.0","id":1,"method":"models.list","params":{}}

// Request (session-scoped)
{"jsonrpc":"2.0","id":2,"method":"session.model.getCurrent","params":{"sessionId":"abc-123"}}

// Notification
{"jsonrpc":"2.0","method":"session.event","params":{"type":"assistant.message_delta","data":{"deltaContent":"Hello"}}}
```
