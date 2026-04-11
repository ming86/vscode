# System Architecture

## Overview

The system comprises four architectural layers and two external actors. Each layer has a distinct responsibility boundary, and communication between layers follows well-defined contracts.

- **Extension Layer** (`extensions/copilot/`) — Owns the Copilot-specific integration logic. Manages the MCP server that external processes connect to, wraps the Copilot SDK to conduct API conversations, and handles lock file lifecycle for cross-process discovery. This layer translates between the Copilot-specific world and the provider-agnostic contracts expected by the layers beneath it.

- **Sessions Layer** (`src/vs/sessions/`) — Owns cross-cutting session infrastructure. Defines the `ISessionsProvider` contract, hosts the provider registry (`ISessionsProvidersService`), aggregates providers into a unified management surface (`ISessionsManagementService`), and houses provider implementations such as `CopilotChatSessionsProvider`. This layer is the glue between extension-contributed providers and the workbench UI.

- **Workbench Layer** (`src/vs/workbench/contrib/chat/`) — Owns the user-facing session experience. Processes extension points to discover available session types, maintains the resolved session list (with temporal grouping), and renders chat conversations through a generic widget and renderer pipeline. This layer is deliberately ignorant of Copilot-specific details.

- **Platform Layer** (`src/vs/platform/agentHost/`) — Owns session state management and persistence. Runs in a dedicated utility process (the agent host), manages the authoritative state tree for each session, persists file-edit data via SQLite, and communicates with the renderer through a JSON-RPC state protocol.

The two **external actors** — the Copilot CLI process and the GitHub Copilot API — sit outside the VS Code process boundary. The CLI connects inward via MCP; the SDK connects outward to the API.

## Layer Diagram

```mermaid
graph TD
    subgraph External
        CLI["Copilot CLI Process"]
        GHAPI["GitHub Copilot API"]
    end

    subgraph Extension Layer
        CONTRIB["CopilotCLIContrib<br/>─────────────────<br/>MCP server orchestrator,<br/>lock file lifecycle, CLI tools"]
        MCP_SERVER["InProcHttpServer (MCP)<br/>─────────────────<br/>Unix domain socket listener,<br/>MCP protocol handler"]
        CLI_SVC["CopilotCLISessionService<br/>─────────────────<br/>Session factory, manages<br/>active CLI sessions"]
        CLI_SESSION["CopilotCLISession<br/>─────────────────<br/>SDK wrapper, maps SDK events<br/>to chat progress tokens"]
        LOCK_MGR["Lock File Manager<br/>─────────────────<br/>Creates/removes lock files<br/>for process discovery"]
    end

    subgraph Sessions Layer
        SESSIONS_MGMT["ISessionsManagementService<br/>─────────────────<br/>Aggregates providers,<br/>manages active session"]
        SESSIONS_PROVIDERS["ISessionsProvidersService<br/>─────────────────<br/>Provider registry"]
        COPILOT_PROVIDER["CopilotChatSessionsProvider<br/>─────────────────<br/>Factory for CopilotCLISession,<br/>RemoteNewSession, AgentSessionAdapter"]
    end

    subgraph Workbench Layer
        CHAT_SESSIONS_SVC["ChatSessionsService<br/>─────────────────<br/>Extension point processor,<br/>content providers, item controllers"]
        AGENT_SESSIONS_SVC["AgentSessionsService<br/>─────────────────<br/>Resolves providers into<br/>unified session list"]
        AGENT_SESSIONS_MODEL["AgentSessionsModel<br/>─────────────────<br/>Section grouping:<br/>Pinned / Today / Yesterday / Week / Older / Archived"]
        AGENT_SESSIONS_CTRL["AgentSessionsControl<br/>─────────────────<br/>Tree widget for session list"]
        CHAT_VIEW_PANE["ChatViewPane<br/>─────────────────<br/>Sidebar view container"]
        CHAT_WIDGET["ChatWidget<br/>─────────────────<br/>Conversation renderer"]
        CHAT_WIDGET_SVC["ChatWidgetService<br/>─────────────────<br/>Widget lifecycle manager"]
        SESSION_PICKERS["Session Type Pickers<br/>─────────────────<br/>Quick-pick UI for new sessions"]
        CONTINUATION["Continuation Actions<br/>─────────────────<br/>Resume / switch / delete"]
    end

    subgraph Platform Layer
        AGENT_SVC["IAgentService<br/>─────────────────<br/>IPC contract to agent host"]
        STATE_MGR["AgentHostStateManager<br/>─────────────────<br/>Authoritative state tree,<br/>action application"]
        SESSION_DATA_SVC["SessionDataService<br/>─────────────────<br/>Per-session data dir,<br/>ref-counted SQLite connections"]
        SESSION_DB["SessionDatabase<br/>─────────────────<br/>SQLite tables for<br/>file-edit persistence"]
        STATE_PROTOCOL["State Protocol (JSON-RPC)<br/>─────────────────<br/>Bidirectional state sync"]
        COPILOT_AGENT["CopilotAgentSession<br/>─────────────────<br/>Agent host session instance"]
    end

    CONTRIB -->|"Creates &<br/>starts"| MCP_SERVER
    CONTRIB -->|"Creates"| LOCK_MGR
    CLI -->|"Reads lock file<br/>to find socket path"| LOCK_MGR
    CLI -->|"Connects via<br/>Unix domain socket"| MCP_SERVER
    MCP_SERVER --> CLI_SVC
    CLI_SVC --> CLI_SESSION
    CLI_SESSION -->|"API requests"| GHAPI
    CLI_SESSION -->|"Response parts via<br/>ChatResponseStream"| CHAT_WIDGET
    COPILOT_PROVIDER -->|"Implements<br/>ISessionsProvider"| SESSIONS_PROVIDERS
    SESSIONS_MGMT -->|"Queries"| SESSIONS_PROVIDERS
    COPILOT_PROVIDER -->|"Contributes items via"| CHAT_SESSIONS_SVC
    CHAT_SESSIONS_SVC --> AGENT_SESSIONS_SVC
    AGENT_SESSIONS_SVC --> AGENT_SESSIONS_MODEL
    AGENT_SESSIONS_MODEL --> AGENT_SESSIONS_CTRL
    AGENT_SESSIONS_CTRL --> CHAT_VIEW_PANE
    CHAT_VIEW_PANE --> CHAT_WIDGET
    CHAT_WIDGET --> CHAT_WIDGET_SVC
    SESSION_PICKERS --> AGENT_SESSIONS_SVC
    CONTINUATION --> AGENT_SESSIONS_SVC
    AGENT_SVC --> STATE_MGR
    STATE_MGR --> STATE_PROTOCOL
    STATE_MGR --> COPILOT_AGENT
    SESSION_DATA_SVC --> SESSION_DB
    COPILOT_AGENT --> SESSION_DATA_SVC
```

## Service Hierarchy

| Service | Layer | Responsibility |
|---------|-------|----------------|
| `ISessionsManagementService` | Sessions (`vs/sessions`) | Aggregates all registered providers into a unified session management surface. Manages the active session reference and persists the last-selected session across restarts. |
| `ISessionsProvidersService` | Sessions (`vs/sessions`) | Registry for `ISessionsProvider` instances. Providers register here; consumers query here. Decouples provider implementation from consumer code. |
| `CopilotChatSessionsProvider` | Sessions (`vs/sessions`) | Copilot-specific provider implementation. Factory for three session subtypes: `CopilotCLISession` (external CLI handoff), `RemoteNewSession` (fresh cloud session), and `AgentSessionAdapter` (wraps agent-host sessions). |
| `IChatSessionsService` | Workbench (`contrib/chat`) | Extension point processor. Manages content providers (how session items produce content), item controllers (how items respond to actions), and session options (filtering, sorting, display). |
| `IAgentSessionsService` | Workbench (`contrib/chat`) | Manages `AgentSessionsModel` — the resolved, flattened, and sorted session list consumed by the UI. Coordinates between multiple providers to produce a single coherent list. |
| `AgentSessionsModel` | Workbench (`contrib/chat`) | Data model backing the session list tree widget. Groups sessions into temporal sections (Pinned, Today, Yesterday, Week, Older, Archived). Also supports capped grouping (More) and repository grouping. Emits change events for reactive UI updates. |
| `IAgentService` | Platform (`agentHost`) | IPC contract between the renderer/extension host and the agent host utility process. All state mutations and queries flow through this interface. |
| `ISessionDataService` | Platform (`agentHost`) | Manages per-session data directories and ref-counted SQLite database connections. Ensures database handles are shared when multiple consumers access the same session's data. |
| `SessionDatabase` | Platform (`agentHost`) | SQLite implementation for persisting file-edit operations within a session. Stores edit history (with before/after file content), and session metadata (custom titles, model info, working directory). |

## Data Flow

The following sequence diagram traces a complete flow: the CLI process connects to VS Code, a conversation is relayed to the GitHub API, and the response is rendered in the chat panel.

```mermaid
sequenceDiagram
    participant CLI as Copilot CLI
    participant FS as File System (Lock File)
    participant MCP as MCP Server<br/>(Extension Host)
    participant SVC as CopilotCLISessionService
    participant SDK as CopilotCLISession<br/>(SDK Wrapper)
    participant API as GitHub Copilot API
    participant AGENT as AgentSessionsService
    participant MODEL as AgentSessionsModel
    participant WIDGET as ChatWidget
    participant RENDERER as Chat Renderers

    Note over CLI,FS: Phase 1 — Discovery
    CLI->>FS: Read lock file to obtain socket path
    FS-->>CLI: Socket path (Unix domain socket)

    Note over CLI,MCP: Phase 2 — Connection
    CLI->>MCP: Connect via MCP protocol (Unix socket)
    MCP->>SVC: Route MCP session-start request
    SVC->>SDK: Create CopilotCLISession instance
    SDK-->>SVC: Session ready

    Note over CLI,API: Phase 3 — Conversation Relay
    CLI->>MCP: Send user message via MCP
    MCP->>SVC: Forward to active session
    SVC->>SDK: Relay message to SDK
    SDK->>API: HTTP request (conversation turn)
    API-->>SDK: Streaming response (SSE chunks)

    Note over SDK,RENDERER: Phase 4 — Event Mapping & Rendering
    SDK->>SDK: Map SDK events to response parts<br/>(markdownContent, toolInvocation, etc.)
    SDK->>WIDGET: Push response parts via ChatResponseStream
    WIDGET->>RENDERER: Render turn content<br/>(markdown, tool results, references)

    Note over CLI,RENDERER: Phase 5 — Session Persistence
    SDK->>API: Session ID persisted server-side
    AGENT->>MODEL: Session appears in list<br/>(grouped by date)
    Note right of MODEL: Session survives restart:<br/>reloaded from provider on next activation
```

## Design Principles

### 1. Agent-Agnostic Protocol

The state protocol — the JSON-RPC contract between the agent host and the renderer — makes no reference to Copilot, CLI, or any specific agent implementation. It defines:

- A **state shape**: a typed, serializable tree describing session content, turn history, and pending operations.
- An **action vocabulary**: a finite set of typed mutations (append turn, update tool status, mark complete).
- **Reconciliation rules**: how optimistic updates in the renderer are confirmed or rolled back by the host.

Any agent that conforms to this protocol can participate in the session system. The Copilot extension is merely the first (and currently only) provider.

### 2. Extension Point Model for Session Types

Session types are not hard-coded. They are registered declaratively through the `ISessionsProvidersService` registry:

- A provider implements `ISessionsProvider` and registers itself at activation time.
- The workbench discovers available session types by querying the registry.
- New session types (e.g., a hypothetical local-LLM provider) require only a new provider registration — no workbench modifications.

This follows VS Code's established contribution-point pattern: the platform defines the shape; extensions fill it.

### 3. Redux-Like State Synchronization

Session state follows an immutable-state, action-stream model reminiscent of Redux:

- **Immutable state tree.** The authoritative state lives in the agent host process. The renderer holds a read-only projection.
- **Action streams.** All mutations are expressed as typed actions dispatched from the renderer to the host.
- **Write-ahead reconciliation.** The renderer applies actions optimistically for responsiveness. The host confirms or rejects each action. On rejection, the renderer rolls back to the last confirmed state.

This architecture permits a responsive UI (no round-trip latency for local interactions) without sacrificing consistency (the host remains authoritative).

### 4. Dual Communication Channels

Two channels serve distinct integration scenarios:

| Channel | Transport | Use Case |
|---------|-----------|----------|
| **In-process SDK** | Direct function calls within the extension host | Sessions originating inside VS Code (new chat, agent-initiated) |
| **MCP (Model Context Protocol)** | Unix domain socket, JSON-RPC framing | Sessions arriving from external processes (CLI, other editors) |

The MCP channel exists because the CLI runs in a separate process — possibly a separate machine over SSH. It uses the Model Context Protocol, which provides a standardized framing for tool/resource exchange between AI-capable processes.

Both channels converge at `CopilotCLISessionService`, which normalizes them into a single internal session representation.

### 5. File-System-Based Discovery

Cross-process discovery uses lock files rather than a service registry:

1. When VS Code starts, it writes a lock file to a well-known directory (`~/.copilot/ide/<uuid>.lock`, with mode `0o600`).
2. The lock file contains a JSON object with connection metadata: `socketPath` (Unix domain socket), `scheme`, `headers` (authentication), `pid`, `ideName`, `timestamp`, `workspaceFolders`, and `isTrusted`. The socket path is the primary discovery datum; the remaining fields allow the CLI to select among multiple instances and authenticate.
3. The CLI reads the lock file to discover where to connect.
4. When VS Code exits, the lock file is removed (or becomes stale and is garbage-collected).

This approach is deliberately simple:
- No daemon process required.
- No network ports to manage or conflict with.
- Graceful degradation: if VS Code is not running, no lock file exists, and the CLI knows immediately.
- Multiple VS Code windows produce multiple lock files; the CLI can choose which to connect to.

### 6. Provider-Agnostic Renderers

The chat rendering pipeline operates on abstract content types — `IChatMarkdownContent`, `IChatToolInvocation`, `IChatContentReference` — not on provider-specific data structures. The renderers:

- Receive typed content tokens from the session model.
- Render each token according to its type (markdown block, tool call card, file reference pill).
- Never inspect the *source* of a token (CLI vs. in-editor vs. agent host).

This means a CLI-originated session renders identically to an in-editor session, because the rendering layer cannot distinguish them.

### 7. State-Based Rendering (toolKind, displayName)

Tool invocations are rendered based on their declared metadata — specifically `toolKind` and `displayName` — rather than raw tool names or internal identifiers. This provides:

- **Stable rendering** across tool renames or version changes.
- **User-comprehensible labels** ("Search files" rather than `ripgrep_search_v2`).
- **Consistent iconography** (tool kind determines the icon family).

The `toolKind` is a coarse classifier (currently `terminal` or `subagent`) that controls visual treatment. The `displayName` is a human-readable label. Together they form a stable rendering contract that is independent of the tool's internal implementation.

## Process Architecture

Five distinct processes participate in the system. Understanding which code runs where is essential for reasoning about communication overhead, failure isolation, and debugging.

```mermaid
graph LR
    subgraph Electron Main Process
        MAIN["Main Process<br/>─────────────────<br/>Window management,<br/>process lifecycle,<br/>spawns utility processes"]
    end

    subgraph Agent Host Utility Process
        AGENT_HOST["CopilotAgent<br/>─────────────────<br/>Authoritative state management,<br/>action application,<br/>SQLite persistence"]
    end

    subgraph Extension Host Process
        EXT_HOST["Copilot Extension<br/>─────────────────<br/>MCP server (Unix socket),<br/>SDK session management,<br/>lock file lifecycle"]
    end

    subgraph Renderer Process
        RENDERER["Chat Panel UI<br/>─────────────────<br/>ChatWidget, session list,<br/>renderers, user input"]
    end

    subgraph External
        CLI_PROC["Copilot CLI<br/>─────────────────<br/>Separate process,<br/>connects via MCP"]
    end

    MAIN -->|"spawns"| AGENT_HOST
    MAIN -->|"spawns"| EXT_HOST
    MAIN -->|"spawns"| RENDERER
    CLI_PROC -->|"MCP over<br/>Unix socket"| EXT_HOST
    EXT_HOST -->|"SDK events /<br/>service calls"| AGENT_HOST
    AGENT_HOST -->|"State protocol<br/>(JSON-RPC)"| RENDERER
    EXT_HOST -->|"Chat progress<br/>tokens"| RENDERER
```

### Main Process (Electron)

The Electron main process is the lifecycle orchestrator. It:

- Manages window creation and destruction.
- Spawns the agent host as a **utility process** (a Chromium `UtilityProcess`, isolated from the renderer's event loop).
- Spawns extension host processes.
- Routes IPC between processes.

The main process does **not** run any Copilot-specific logic. It is purely infrastructure.

### Agent Host Utility Process

The agent host runs in a dedicated utility process, isolated from both the renderer and the extension host. It:

- Holds the **authoritative state tree** for all active sessions.
- Applies actions received from the renderer via the state protocol.
- Manages `SessionDataService` for per-session SQLite databases.
- Runs `CopilotAgentSession` instances that encapsulate session-level logic.

Running in a utility process ensures that SQLite I/O and state computation do not block the UI thread.

### Extension Host Process

The extension host runs the Copilot extension, which:

- Starts the **MCP server** on a Unix domain socket, listening for external CLI connections.
- Manages **lock files** that advertise the socket path to external processes.
- Creates and manages **SDK sessions** (`CopilotCLISession`) that wrap the Copilot SDK's conversation API.
- Translates SDK events into chat progress tokens that the renderer can consume.

The extension host is the bridge between the Copilot-specific world (SDK, API, CLI protocol) and the provider-agnostic workbench.

### Renderer Process

The renderer process runs the chat panel UI:

- `ChatViewPane` hosts the sidebar view.
- `AgentSessionsControl` renders the session list as a tree widget.
- `ChatWidget` renders the active conversation.
- Chat renderers transform abstract content tokens into DOM elements.

The renderer holds a **read-only projection** of session state. All mutations are dispatched as actions to the agent host; the renderer never modifies state directly.

### External CLI Process

The Copilot CLI is a fully independent process — it may be a different binary, a different language runtime, or running on a remote machine (connected via SSH). It:

- Discovers VS Code by reading lock files from the file system.
- Connects to the MCP server via Unix domain socket.
- Sends conversation context (user messages, tool results) using the MCP protocol.
- Receives acknowledgments and can monitor session progress.

The CLI has no direct access to VS Code internals. All interaction is mediated through the MCP protocol, which provides a clean integration boundary.
