# Copilot CLI Session Integration — Architecture Overview

## Purpose

The Copilot CLI Session Integration enables VS Code (and the vscode-copilot-chat extension) to create, resume, and interact with sessions managed by the Copilot CLI (`copilot`). Sessions are persistent, resumable conversation state machines stored on disk at `~/.copilot/session-state/`. The integration allows users to start a session in the CLI terminal, resume it in VS Code, or vice versa — with both clients potentially connected simultaneously.

## Three Integration Pathways

The system comprises three distinct but cooperating integration pathways:

| Pathway | Consumer | SDK Layer | Transport | Session Ownership |
|---------|----------|-----------|-----------|-------------------|
| **In-Process SDK** (Extension) | vscode-copilot-chat extension | `@github/copilot/sdk` `LocalSessionManager` | In-process function calls | Extension creates/resumes sessions directly via SDK |
| **Agent Host Process** (VS Code Core) | VS Code platform (`src/vs/platform/agentHost/`) | `@github/copilot-sdk` `CopilotClient` | JSON-RPC 2.0 over MessagePort (local) or WebSocket (remote) | Dedicated utility process manages sessions |
| **ACP Server Mode** (CLI) | External IDE connections | Copilot CLI binary | JSON-RPC 2.0 over Unix socket | CLI process exposes sessions via `--acp` flag |

The **In-Process SDK** pathway is the primary mechanism used by the vscode-copilot-chat extension. The **Agent Host Process** pathway is the VS Code platform's generic, agent-agnostic infrastructure. The **ACP Server Mode** is the CLI's native multi-client protocol.

## System Architecture

```
┌──────────────────────────────────────────────────────────────────────────────┐
│                          Persistent Storage Layer                            │
│                                                                              │
│  ~/.copilot/session-state/{sessionId}/                                       │
│  ├── events.jsonl          ← Append-only event log (JSONL)                  │
│  ├── workspace.yaml        ← Session metadata (cwd, summary, timestamps)    │
│  ├── vscode.metadata.json  ← VS Code integration marker                     │
│  ├── checkpoints/          ← Context compaction snapshots                   │
│  └── files/                ← Session artifacts                              │
│                                                                              │
│  ~/.copilot/ide/{uuid}.lock   ← IDE connection lock files                   │
│  ~/.copilot/config.json       ← User configuration                          │
│  ~/.copilot/mcp-config.json   ← MCP server definitions                      │
└──────────────────────────────────────────────────────────────────────────────┘
            ▲                          ▲                         ▲
            │                          │                         │
  ┌─────────┴──────────┐    ┌─────────┴──────────┐   ┌─────────┴──────────┐
  │  Copilot CLI        │    │  vscode-copilot-chat│   │  VS Code Platform  │
  │  (Terminal Process)  │    │  (Extension Host)   │   │  (Agent Host)      │
  │                      │    │                     │   │                    │
  │  Interactive I/O     │    │  LocalSessionManager│   │  CopilotClient     │
  │  ACP Server Mode     │    │  (in-process SDK)   │   │  (utility process) │
  │  --acp flag          │    │                     │   │                    │
  │                      │    │  ICopilotCLISession  │   │  IAgent interface  │
  │  JSON-RPC over       │    │  Service             │   │  JSON-RPC over     │
  │  Unix socket         │    │                     │   │  MessagePort/WS    │
  └──────────────────────┘    └─────────────────────┘   └────────────────────┘
            │                          │                         │
            └──────────────────────────┴─────────────────────────┘
                                       │
                              Shared Session State
                           (events.jsonl on disk)
```

## Session Lifecycle

```
┌──────────────┐
│   Create     │  ← UUID assigned, directory created, session.start event
└──────┬───────┘
       ↓
┌──────────────┐
│   Active     │  ← Send/receive messages, tool calls, streaming
└──────┬───────┘
       │
       ├───→ Idle (on disk) ───→ Resume ───→ Active
       │                                       ↑
       ├───→ Compact ──────────────────────────┘
       │     (context window compaction)
       │
       └───→ Shutdown ───→ Persisted (resumable)
```

**Key Properties:**
- Sessions survive process termination, terminal close, and Ctrl+C
- Multiple clients can read/contribute to the same session simultaneously
- Context compaction creates checkpoint summaries when token limits approach
- Events form a linked chain via `parentId` for causal ordering

## Component Map

```
┌────────────────────────────────────────────────────────────────────────────┐
│                    vscode-copilot-chat Extension                           │
│                                                                            │
│  ┌─────────────────────────────┐    ┌──────────────────────────────────┐  │
│  │  ICopilotCLISDK             │    │  ICopilotCLISessionService       │  │
│  │  (copilotCli.ts)            │    │  (copilotcliSessionService.ts)   │  │
│  │                             │    │                                  │  │
│  │  Dynamic import of          │    │  Session CRUD, history,          │  │
│  │  @github/copilot/sdk        │    │  forking, metadata              │  │
│  │  Auth token provider        │    │  LocalSessionManager wrapper    │  │
│  └──────────┬──────────────────┘    └───────────┬────────────────────┘  │
│             │                                    │                       │
│             ▼                                    ▼                       │
│  ┌─────────────────────────────┐    ┌──────────────────────────────────┐  │
│  │  ICopilotCLISession         │    │  CopilotCLIChatSessions          │  │
│  │  (copilotcliSession.ts)     │    │  (copilotCLIChatSessions.ts)     │  │
│  │                             │    │                                  │  │
│  │  Session wrapper            │    │  VS Code ChatSessionItemController│  │
│  │  Stream attachment          │    │  Session list provider           │  │
│  │  Request handling           │    │  Content provider                │  │
│  │  Steering mode              │    │  Fork handler                   │  │
│  └─────────────────────────────┘    └──────────────────────────────────┘  │
│                                                                            │
│  ┌─────────────────────────────┐    ┌──────────────────────────────────┐  │
│  │  ICopilotCLIMCPHandler      │    │  CopilotCliBridgeSpanProcessor   │  │
│  │  (mcpHandler.ts)            │    │  (copilotCliBridgeSpanProcessor) │  │
│  │                             │    │                                  │  │
│  │  MCP server config          │    │  OTel span forwarding           │  │
│  │  Gateway proxy              │    │  Debug panel integration        │  │
│  │  Built-in GitHub server     │    │  Trace ID → Session ID mapping  │  │
│  └─────────────────────────────┘    └──────────────────────────────────┘  │
│                                                                            │
│  ┌─────────────────────────────┐    ┌──────────────────────────────────┐  │
│  │  ICopilotCLIModels          │    │  ICopilotCLIAgents               │  │
│  │  (copilotCli.ts)            │    │  (copilotCli.ts)                 │  │
│  │                             │    │                                  │  │
│  │  Model listing/resolution   │    │  Custom agent discovery          │  │
│  │  Default model persistence  │    │  SDK + prompt-file agents        │  │
│  │  LM provider registration   │    │  Per-session agent tracking      │  │
│  └─────────────────────────────┘    └──────────────────────────────────┘  │
│                                                                            │
│  ┌─────────────────────────────┐                                          │
│  │  ICopilotCLISkills          │                                          │
│  │  (copilotCLISkills.ts)      │                                          │
│  │                             │                                          │
│  │  Skill location discovery   │                                          │
│  │  getSkillsLocations()       │                                          │
│  └─────────────────────────────┘                                          │
└────────────────────────────────────────────────────────────────────────────┘

┌────────────────────────────────────────────────────────────────────────────┐
│                    VS Code Platform (Agent Host)                           │
│                                                                            │
│  ┌─────────────────────────────┐    ┌──────────────────────────────────┐  │
│  │  IAgentService              │    │  SessionStateManager             │  │
│  │  (agentService.ts)          │    │  (sessionStateManager.ts)        │  │
│  │                             │    │                                  │  │
│  │  Agent registration         │    │  Redux-like state tree           │  │
│  │  Session routing            │    │  Pure reducer functions          │  │
│  │  Progress events            │    │  Multi-client reconciliation     │  │
│  └──────────┬──────────────────┘    └──────────────────────────────────┘  │
│             │                                                              │
│             ▼                                                              │
│  ┌─────────────────────────────┐    ┌──────────────────────────────────┐  │
│  │  CopilotAgent               │    │  ProtocolServerHandler           │  │
│  │  (copilotAgent.ts)          │    │  (protocolServerHandler.ts)      │  │
│  │                             │    │                                  │  │
│  │  IAgent implementation      │    │  JSON-RPC request handling       │  │
│  │  CopilotClient wrapper      │    │  Subscribe/unsubscribe           │  │
│  │  SDK ↔ Agent event mapping  │    │  Action dispatch                 │  │
│  └─────────────────────────────┘    └──────────────────────────────────┘  │
└────────────────────────────────────────────────────────────────────────────┘
```

## Feature Flags & Settings

| Setting | Key | Default | Purpose |
|---------|-----|---------|---------|
| Agent Host | `chat.agentHost.enabled` | `false` | Gates the agent host utility process and platform-level integration |
| Agent Host IPC Logging | `chat.agentHost.ipcLoggingEnabled` | `false` | Logs all JSON-RPC traffic for debugging |
| Remote Agent Hosts | `chat.remoteAgentHostsEnabled` | `false` | Enables WebSocket connections to remote agent hosts |
| CLI MCP Servers | `github.copilot.chat.cliMcpServer.enabled` | experiment-based | Gates MCP server forwarding to CLI sessions |
| Sessions Window Override | `chat.experimentalSessionsWindowOverride` | `false` | Forces sessions window mode (agent sessions workspace) |

## File Inventory

| File | Component | Purpose |
|------|-----------|---------|
| `src/extension/chatSessions/copilotcli/node/copilotCli.ts` | `ICopilotCLISDK`, `ICopilotCLIModels`, `ICopilotCLIAgents` | SDK bridge — dynamic import, auth, models, agents |
| `src/extension/chatSessions/copilotcli/node/copilotcliSessionService.ts` | `ICopilotCLISessionService` | Session lifecycle — create, get, fork, history, metadata |
| `src/extension/chatSessions/copilotcli/node/copilotcliSession.ts` | `ICopilotCLISession` | Session wrapper — stream attachment, request handling, steering |
| `src/extension/chatSessions/copilotcli/node/mcpHandler.ts` | `ICopilotCLIMCPHandler` | MCP server configuration and gateway proxy |
| `src/extension/chatSessions/copilotcli/node/copilotCliBridgeSpanProcessor.ts` | `CopilotCliBridgeSpanProcessor` | OTel span forwarding for debug panel |
| `src/extension/chatSessions/copilotcli/node/copilotCLISkills.ts` | `ICopilotCLISkills` | Skill location discovery for SDK sessions |
| `src/extension/chatSessions/copilotcli/node/cliHelpers.ts` | Path helpers | `getCopilotHome()`, `getCopilotCliStateDir()` (`~/.copilot/ide/`), `getCopilotCLISessionStateDir()`, `getCopilotCLISessionDir()`, etc. |
| `src/extension/chatSessions/copilotcli/common/copilotCLITools.ts` | Event/tool processing | `buildChatHistoryFromEvents()`, tool execution handlers |
| `src/extension/chatSessions/vscode-node/copilotCLIChatSessions.ts` | `CopilotCLIChatSessionContentProvider` | VS Code session item controller, content provider |
| `src/extension/chatSessions/vscode-node/copilotCloudSessionsProvider.ts` | `CopilotCloudSessionsProvider` | Cloud agent session provider (delegate to cloud) |
| `src/extension/chatSessions/vscode-node/chatHistoryBuilder.ts` | History builder | Reconstructs chat turns from SDK events |
| `src/extension/chatSessions/common/agentSessionsWorkspace.ts` | `IAgentSessionsWorkspace` | Marker interface for agent sessions workspace |

## Next Documents

| Document | Contents |
|----------|----------|
| [01-session-storage.md](01-session-storage.md) | On-disk storage format — events.jsonl, workspace.yaml, checkpoints, IDE lock files |
| [02-sdk-bridge.md](02-sdk-bridge.md) | SDK bridge service — dynamic import, auth, shims, `ICopilotCLISDK` |
| [03-session-service.md](03-session-service.md) | Session service — `ICopilotCLISessionService`, lifecycle, reference counting |
| [04-session-wrapper.md](04-session-wrapper.md) | Session wrapper — `ICopilotCLISession`, streaming, request handling, steering |
| [05-chat-sessions-provider.md](05-chat-sessions-provider.md) | VS Code integration — session item controller, content provider, proposed API |
| [06-agent-host-protocol.md](06-agent-host-protocol.md) | Agent host protocol — JSON-RPC, state model, actions, reconciliation |
| [07-transport-layers.md](07-transport-layers.md) | Transport mechanisms — stdio, TCP, MessagePort, WebSocket, Unix socket |
| [08-mcp-integration.md](08-mcp-integration.md) | MCP server configuration, gateway proxy, tool remapping |
| [09-event-system.md](09-event-system.md) | Session events — types, chain structure, history reconstruction |
| [10-models-and-agents.md](10-models-and-agents.md) | Model management, custom agents, prompt-file agents |
| [11-implementation-guide.md](11-implementation-guide.md) | Step-by-step guide for implementing your own CLI session integration |
