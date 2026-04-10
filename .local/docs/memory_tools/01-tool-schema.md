# Tool Schema & Registration

## Tool Declarations (package.json)

Both tools are declared in `package.json` under `contributes.languageModelTools`.

### `copilot_memory`

The primary memory tool. Exposes a file-system-like interface to the LLM with six commands.

```json
{
    "name": "copilot_memory",
    "displayName": "Memory",
    "toolReferenceName": "memory",
    "userDescription": "Manage persistent memory across conversations",
    "when": "config.github.copilot.chat.tools.memory.enabled",
    "modelDescription": "Manage a persistent memory system with three scopes for storing notes and information across conversations.\n\nMemory is organized under /memories/ with three tiers:\n- `/memories/` — User memory: persistent notes that survive across all workspaces and conversations. Store preferences, patterns, and general insights here.\n- `/memories/session/` — Session memory: notes scoped to the current conversation. Store task-specific context and in-progress notes here. Cleared after the conversation ends.\n- `/memories/repo/` — Repository memory: repository-scoped facts stored via Copilot. Only the `create` command is supported for this path.\n\nIMPORTANT: Before creating new memory files, first view the /memories/ directory to understand what already exists. This helps avoid duplicates and maintain organized notes.\n\nCommands:\n- `view`: View contents of a file or list directory contents. Can be used on files or directories (e.g., \"/memories/\" to see all top-level items).\n- `create`: Create a new file at the specified path with the given content. Fails if the file already exists.\n- `str_replace`: Replace an exact string in a file with a new string. The old_str must appear exactly once in the file.\n- `insert`: Insert text at a specific line number in a file. Line 0 inserts at the beginning.\n- `delete`: Delete a file or directory (and all its contents).\n- `rename`: Rename or move a file or directory from path to new_path. Cannot rename across scopes.",
    "inputSchema": {
        "type": "object",
        "properties": {
            "command": {
                "type": "string",
                "enum": ["view", "create", "str_replace", "insert", "delete", "rename"],
                "description": "The operation to perform on the memory file system."
            },
            "path": {
                "type": "string",
                "description": "The absolute path to the file or directory inside /memories/, e.g. \"/memories/notes.md\". Used by all commands except `rename`."
            },
            "file_text": {
                "type": "string",
                "description": "Required for `create`. The content of the file to create."
            },
            "old_str": {
                "type": "string",
                "description": "Required for `str_replace`. The exact string in the file to replace. Must appear exactly once."
            },
            "new_str": {
                "type": "string",
                "description": "Required for `str_replace`. The new string to replace old_str with."
            },
            "insert_line": {
                "type": "number",
                "description": "Required for `insert`. The 0-based line number to insert text at. 0 inserts before the first line."
            },
            "insert_text": {
                "type": "string",
                "description": "Required for `insert`. The text to insert at the specified line."
            },
            "view_range": {
                "type": "array",
                "items": { "type": "number" },
                "minItems": 2,
                "maxItems": 2,
                "description": "Optional for `view`. A two-element array [start_line, end_line] (1-indexed) to view a specific range of lines."
            },
            "old_path": {
                "type": "string",
                "description": "Required for `rename`. The current path of the file or directory to rename."
            },
            "new_path": {
                "type": "string",
                "description": "Required for `rename`. The new path for the file or directory."
            }
        },
        "required": ["command"]
    }
}
```

### `copilot_resolveMemoryFileUri`

A utility tool that translates virtual `/memories/...` paths to real filesystem URIs. Used when the LLM needs an actual URI (e.g., for artifact references).

```json
{
    "name": "copilot_resolveMemoryFileUri",
    "displayName": "Resolve Memory File URI",
    "toolReferenceName": "resolveMemoryFileUri",
    "userDescription": "Resolve a memory file path to its actual URI",
    "modelDescription": "Resolve a memory file path (like /memories/session/plan.md or /memories/repo/notes.md) to its fully qualified URI. Use this when you need the actual URI for a memory file, for example to pass it to setArtifacts. The path must start with /memories/.",
    "tags": [],
    "inputSchema": {
        "type": "object",
        "properties": {
            "path": {
                "type": "string",
                "description": "The memory file path to resolve (e.g. /memories/session/plan.md)."
            }
        },
        "required": ["path"]
    }
}
```

## Tool Set Membership

Both tools are part of the `vscode` tool set:

```json
{
    "name": "vscode",
    "tools": [
        "getProjectSetupInfo",
        "installExtension",
        "memory",
        "newWorkspace",
        "resolveMemoryFileUri",
        "runCommand",
        "switchAgent",
        "vscodeAPI"
    ]
}
```

This means when a `.prompt.md` or `.agent.md` file specifies `tools: ['vscode/memory']`, it references the memory tool via the `<source>/<toolReferenceName>` convention.

## Tool Name Constants

Defined in `src/extension/tools/common/toolNames.ts`:

```typescript
// Short names (toolReferenceName)
enum ToolName {
    Memory = 'memory',
    ResolveMemoryFileUri = 'resolve_memory_file_uri',
}

// Full qualified names (the `name` field from package.json)
enum ContributedToolName {
    Memory = 'copilot_memory',
    ResolveMemoryFileUri = 'copilot_resolveMemoryFileUri',
}

// Category assignment
Memory → ToolCategory.VSCodeInteraction
ResolveMemoryFileUri → ToolCategory.Core
```

## Tool Registration

Both tools self-register via side-effect imports:

```typescript
// memoryTool.tsx (last line)
ToolRegistry.registerTool(MemoryTool);

// resolveMemoryFileUriTool.tsx (last line)
ToolRegistry.registerTool(ResolveMemoryFileUriTool);
```

These modules are imported in `src/extension/tools/node/allTools.ts`:

```typescript
import './memoryTool';
import './resolveMemoryFileUriTool';
```

The `allTools.ts` file is itself imported by `src/extension/tools/vscode-node/tools.ts` (the `ToolsContribution`), which registers them with VS Code's `vscode.lm.registerTool()` API during extension activation.

## Settings

```json
"github.copilot.chat.tools.memory.enabled": {
    "type": "boolean",
    "default": true,
    "markdownDescription": "Enable or disable the memory tool",
    "tags": ["preview"]
},
"github.copilot.chat.copilotMemory.enabled": {
    "type": "boolean",
    "default": false,
    "markdownDescription": "Enable or disable cloud-backed Copilot Memory for repositories",
    "tags": ["preview"]
}
```

Both support experiment-based rollout via `getExperimentBasedConfig()`.

## Commands

```json
{
    "command": "github.copilot.chat.tools.memory.showMemories",
    "title": "Show Memories",
    "category": "Chat"
},
{
    "command": "github.copilot.chat.tools.memory.clearMemories",
    "title": "Clear Memories",
    "category": "Chat"
}
```

Both are in the command palette when `config.github.copilot.chat.tools.memory.enabled` is true.
