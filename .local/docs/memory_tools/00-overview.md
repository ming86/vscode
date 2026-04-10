# Memory Tools — Architecture Overview

## Purpose

The Memory Tools system provides a persistent, scoped note-taking capability that allows Copilot's LLM to read and write structured files across three distinct scopes — user, session, and repository. Memory is injected into the LLM's system prompt on each conversation so that Copilot can build on prior interactions and stored knowledge.

## Three Parallel Subsystems

The memory feature comprises three distinct but coexisting subsystems:

| Subsystem | Scope | Provider | Storage | Prompt Injection |
|-----------|-------|----------|---------|-----------------|
| **File-based `copilot_memory` tool** | All providers | VS Code tool framework | Local filesystem (virtual `/memories/` paths) | `MemoryContextPrompt` component |
| **Anthropic native memory (`memory_20250818`)** | BYOK Anthropic only | Anthropic Messages API beta | Server-managed (transparent) | Same `MemoryContextPrompt` (prompt layer unchanged) |
| **Claude `/memory` slash command** | Claude Agent SDK sessions | `@anthropic-ai/claude-agent-sdk` | Real CLAUDE.md files on disk | SDK reads files natively |

The file-based tool (documents 01–08) is the primary system. The Anthropic native tool (document 09) replaces the transport layer for supported Claude models. The Claude `/memory` command (document 10) is an entirely separate subsystem within the Claude Agent integration.

## Scopes

| Scope | Virtual Path | Persistence | Physical Storage | Auto-loaded in Prompt |
|-------|-------------|-------------|------------------|----------------------|
| **User** | `/memories/` | Permanent (cross-workspace, cross-session) | `globalStorageUri/memory-tool/memories/` | First 200 lines of all files |
| **Session** | `/memories/session/` | Current conversation only (14-day TTL) | `storageUri/memory-tool/memories/<sessionId>/` | File listing only (names, not content) |
| **Repository (local)** | `/memories/repo/` | Workspace-scoped | `storageUri/memory-tool/memories/repo/` | File listing only |
| **Repository (CAPI)** | `/memories/repo/` | Cloud-backed via Copilot API | GitHub Copilot Agent Memory API | Fetched and rendered with subject/fact/citations |

## Component Map

```
┌─────────────────────────────────────────────────────────────────────┐
│                         package.json                                │
│  ┌─────────────────────┐  ┌──────────────────────────────┐         │
│  │  copilot_memory      │  │  copilot_resolveMemoryFileUri│         │
│  │  (tool schema)       │  │  (tool schema)               │         │
│  └──────────┬──────────┘  └──────────────┬───────────────┘         │
│             │  when: config.github.       │                         │
│             │  copilot.chat.tools.        │                         │
│             │  memory.enabled             │                         │
└─────────────┼─────────────────────────────┼─────────────────────────┘
              │                             │
              ▼                             ▼
┌─────────────────────────────┐  ┌──────────────────────────────┐
│  MemoryTool                 │  │  ResolveMemoryFileUriTool    │
│  (memoryTool.tsx)           │  │  (resolveMemoryFileUriTool.tsx)│
│                             │  │                              │
│  Commands: view, create,    │  │  Resolves /memories/... path │
│  str_replace, insert,       │  │  → real filesystem URI       │
│  delete, rename             │  │                              │
└──────┬──────────┬───────────┘  └──────────────────────────────┘
       │          │
       │          ▼
       │  ┌───────────────────────┐
       │  │  IAgentMemoryService  │
       │  │  (agentMemoryService) │
       │  │                       │
       │  │  CAPI repo memory:    │
       │  │  GET /enabled         │
       │  │  GET /recent          │
       │  │  PUT /                │
       │  └───────────────────────┘
       │
       ▼
┌──────────────────────────┐     ┌──────────────────────────────┐
│  IFileSystemService      │     │  IMemoryCleanupService       │
│  (read/write/delete)     │     │  (memoryCleanupService)      │
│                          │     │                              │
│  Local disk operations   │     │  14-day retention for        │
│  for user/session/repo   │     │  session files               │
└──────────────────────────┘     │  Runs on startup             │
                                 └──────────────────────────────┘

┌──────────────────────────────────────────────────────────────────┐
│                     Prompt Integration                            │
│                                                                  │
│  ┌──────────────────────────┐  ┌──────────────────────────────┐ │
│  │  MemoryInstructionsPrompt│  │  MemoryContextPrompt         │ │
│  │  Rendered in SystemMessage│  │  Rendered in UserMessage     │ │
│  │  (when base instructions │  │  (first turn only)           │ │
│  │   are not omitted)       │  │                              │ │
│  │                          │  │                              │ │
│  │  Scope descriptions,     │  │  <userMemory> content        │ │
│  │  usage guidelines,       │  │  <sessionMemory> file list   │ │
│  │  repo memory criteria    │  │  <repoMemory> file list/CAPI │ │
│  └──────────────────────────┘  └──────────────────────────────┘ │
└──────────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────────┐
│                     VS Code UI Commands                          │
│                                                                  │
│  showMemories  — QuickPick to browse all memory files            │
│  clearMemories — Destructive delete of all memory directories    │
└──────────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────────┐
│               Anthropic Native Memory (BYOK only)                │
│                                                                  │
│  modelSupportsMemory()          Is Claude Haiku 4.5+, Sonnet 4+,│
│  isAnthropicMemoryToolEnabled() or Opus 4+?                      │
│                                                                  │
│  AnthropicLMProvider intercepts tool name === 'memory'           │
│  → replaces with { type: 'memory_20250818' }                    │
│  → adds beta: 'context-management-2025-06-27'                   │
│  → server manages memory state transparently                    │
│                                                                  │
│  Prompt layer unchanged — MemoryContextPrompt is provider-agnostic│
└──────────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────────┐
│               Claude /memory Slash Command                       │
│                                                                  │
│  MemorySlashCommand → QuickPick → open/create CLAUDE.md          │
│                                                                  │
│  Scopes:  ~/.claude/CLAUDE.md          (user)                   │
│           <workspace>/.claude/CLAUDE.md     (project)            │
│           <workspace>/.claude/CLAUDE.local.md  (local)           │
│                                                                  │
│  SDK reads CLAUDE.md natively via settingSources config          │
│  ClaudeSettingsChangeTracker monitors mtime → session restart    │
│  Gated by: config.github.copilot.chat.claudeAgent.enabled       │
└──────────────────────────────────────────────────────────────────┘
```

## Feature Flags

| Setting | Key | Default | Purpose |
|---------|-----|---------|---------|
| Memory Tool | `github.copilot.chat.tools.memory.enabled` | `true` (experiment-based) | Gates the `copilot_memory` tool, memory prompt injection, and UI commands |
| Copilot Memory | `github.copilot.chat.copilotMemory.enabled` | `false` (experiment-based) | Gates cloud-backed (CAPI) repo memory |

### Feature Interaction Matrix

| MemoryToolEnabled | CopilotMemoryEnabled | Anthropic Model | Behavior |
|:-:|:-:|:-:|---|
| true | false | No | Local user + session + local repo memory (file-based tool) |
| true | true | No | Local user + session + **CAPI** repo memory (file-based tool) |
| true | false | Yes | Local user + session + local repo memory (**native `memory_20250818` tool** replaces transport for `memory`-named tools; `copilot_memory` unaffected) |
| true | true | Yes | Local user + session + CAPI repo memory (native transport for `memory` tools) |
| false | true | Any | Only CAPI repo memory in context (no tool available) |
| false | false | Any | Memory completely disabled |

**Note:** The Claude `/memory` slash command operates independently — it is gated by `claudeAgent.enabled` and is unaffected by the above matrix.

## File Inventory

| File | Component | Purpose |
|------|-----------|---------|
| `package.json` | Tool schemas, settings, commands | Declaration layer — tool contracts, UI contributions |
| `src/extension/tools/node/memoryTool.tsx` | `MemoryTool` | Core tool implementation — all 6 commands, path routing, scope resolution |
| `src/extension/tools/node/resolveMemoryFileUriTool.tsx` | `ResolveMemoryFileUriTool` | Translates virtual `/memories/` paths to real filesystem URIs |
| `src/extension/tools/node/memoryContextPrompt.tsx` | `MemoryContextPrompt`, `MemoryInstructionsPrompt` | Prompt components injecting memory content and instructions |
| `src/extension/tools/common/agentMemoryService.ts` | `AgentMemoryService` | CAPI client for cloud-backed repository memory |
| `src/extension/tools/common/memoryCleanupService.ts` | `MemoryCleanupService` | 14-day TTL cleanup for session memory files |
| `src/extension/tools/vscode-node/tools.ts` | `ToolsContribution` | VS Code command implementations (showMemories, clearMemories) |
| `src/extension/tools/common/toolNames.ts` | `ToolName` enum | Tool name constants |
| `src/extension/tools/node/allTools.ts` | Side-effect imports | Triggers tool registration |
| `src/extension/extension/vscode-node/services.ts` | Service registration | DI bindings for memory services |
| `src/extension/prompts/node/agent/agentPrompt.tsx` | `AgentPrompt` | Renders memory prompts into system/user messages |
| `src/platform/configuration/common/configurationService.ts` | Config keys | Feature flag definitions |
| `src/platform/networking/common/anthropic.ts` | `modelSupportsMemory()`, `isAnthropicMemoryToolEnabled()` | Anthropic native memory feature gates |
| `src/extension/byok/vscode-node/anthropicProvider.ts` | `AnthropicLMProvider` | Intercepts `memory` tool and registers as `memory_20250818` type |
| `src/extension/chatSessions/claude/vscode-node/slashCommands/memoryCommand.ts` | `MemorySlashCommand` | Claude `/memory` slash command handler |
| `src/extension/chatSessions/claude/vscode-node/claudeSlashCommandService.ts` | `ClaudeSlashCommandService` | Dispatches Claude slash commands |
| `src/extension/chatSessions/claude/node/claudeCodeAgent.ts` | `ClaudeCodeAgent` | SDK Options with CLAUDE.md path resolvers |
| `src/extension/chatSessions/claude/node/claudeSettingsChangeTracker.ts` | `ClaudeSettingsChangeTracker` | Mtime-based CLAUDE.md change detection |
| `.local/docs/memory_tools/11-user-memory-deep-dive.md` | Documentation | User-scope memory deep dive — operations, path resolution, prompt injection, storage, gating, telemetry, edge cases |

## Next Documents

| Document | Contents |
|----------|----------|
| [01-tool-schema.md](01-tool-schema.md) | Complete tool JSON schemas and registration |
| [02-memory-tool.md](02-memory-tool.md) | `MemoryTool` class — full implementation reference |
| [03-storage-layout.md](03-storage-layout.md) | Physical storage layout, path resolution, session ID extraction |
| [04-prompt-integration.md](04-prompt-integration.md) | How memory is injected into LLM prompts |
| [05-capi-service.md](05-capi-service.md) | Cloud-backed repository memory via Copilot API |
| [06-cleanup-service.md](06-cleanup-service.md) | Session memory lifecycle and stale file cleanup |
| [07-ui-commands.md](07-ui-commands.md) | VS Code commands — showMemories and clearMemories |
| [08-telemetry.md](08-telemetry.md) | Telemetry events and properties |
| [09-anthropic-native-memory.md](09-anthropic-native-memory.md) | Anthropic `memory_20250818` server-side tool integration |
| [10-claude-memory-slash-command.md](10-claude-memory-slash-command.md) | Claude `/memory` slash command and CLAUDE.md file management |
| [11-user-memory-deep-dive.md](11-user-memory-deep-dive.md) | User Memory Deep Dive — user-scope tool operations, path resolution, prompt injection, UI commands, storage, feature gating, telemetry, edge cases |
