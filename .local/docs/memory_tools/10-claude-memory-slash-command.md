# Claude `/memory` Slash Command

The Claude Agent SDK integration includes a dedicated memory subsystem that manages **CLAUDE.md** files across user, project, and local scopes. Invoked via the `/memory` slash command in chat, it presents a QuickPick of memory file locations, creates files from templates when absent, and opens them in the editor for human authorship. The SDK itself reads these files and injects their content into the system prompt — the VS Code extension handles discovery, change detection, and the editing workflow only.

This subsystem is entirely distinct from the `copilot_memory` tool (which operates a virtual filesystem under `/memories/`) and from the Anthropic native `memory_20250818` server tool. The three share a conceptual domain but no implementation surface.

## File Architecture

All slash command infrastructure resides under the Claude Agent SDK integration directory:

```
src/extension/chatSessions/claude/
├── node/
│   ├── claudeCodeAgent.ts              # SDK Options with settingSources, cwd, path resolvers
│   └── claudeSettingsChangeTracker.ts  # Mtime-based change detection for CLAUDE.md files
└── vscode-node/
    ├── claudeSlashCommandService.ts    # Dispatches slash commands, lazy instantiation
    └── slashCommands/
        ├── index.ts                    # Side-effect imports for registration
        ├── claudeSlashCommandRegistry.ts  # Registry pattern, handler interface
        └── memoryCommand.ts            # The /memory slash command handler
```

## Slash Command Registration

The registration mechanism follows a static registry pattern with side-effect module loading.

### Registry Interface

`claudeSlashCommandRegistry.ts` defines:

- **`IClaudeSlashCommandHandler`** — Interface requiring `commandName`, `description`, optional `commandId`, and an async `handle()` method.
- **`registerClaudeSlashCommand(ctor)`** — Called at module load time as a side effect; pushes the constructor into a module-scoped array.
- **`getClaudeSlashCommandRegistry()`** — Returns all registered constructors.

### Service Instantiation

`ClaudeSlashCommandService` (~140 lines) performs:

1. **Lazy instantiation** — Handlers are constructed on first use via the instantiation service, then cached by `commandName`.
2. **VS Code command registration** — Handlers with a `commandId` property receive a registered VS Code command (e.g., `copilot.claude.memory`).
3. **Dispatch matching** — Checks `request.command` first (VS Code UI routing), then parses `/command args` from raw prompt text as a fallback.

### Side-Effect Loading

`slashCommands/index.ts` imports command modules purely for their registration side effects:

```typescript
// index.ts — 17 lines
// Each import triggers registerClaudeSlashCommand() at module load
import './memoryCommand';
// ... other slash commands
```

## The MemorySlashCommand Class

The handler (232 lines) orchestrates a QuickPick-driven workflow for opening or creating CLAUDE.md files.

### Registration and Metadata

```typescript
export class MemorySlashCommand implements IClaudeSlashCommandHandler {
    readonly commandName = 'memory';
    readonly description = 'Open memory files (CLAUDE.md) for editing';
    readonly commandId = 'copilot.claude.memory';
    // ...
}

registerClaudeSlashCommand(MemorySlashCommand);
```

The `registerClaudeSlashCommand()` call at module scope is the registration side effect.

### `handle()` — Entry Point

```typescript
async handle(
    _args: string,
    stream: vscode.ChatResponseStream | undefined,
    _token: CancellationToken
): Promise<vscode.ChatResult> {
    stream?.markdown(vscode.l10n.t('Opening memory file picker...'));
    this._runPicker().catch(error => {
        this.logService.error('[MemorySlashCommand] Error running memory picker:', error);
        vscode.window.showErrorMessage(
            vscode.l10n.t('Error opening memory file: {0}', error instanceof Error ? error.message : String(error))
        );
    });
    return {};
}
```

The picker runs asynchronously — `handle()` returns immediately with an empty result. Errors surface via `showErrorMessage` rather than propagating to the chat response stream.

### `_runPicker()` — QuickPick Orchestration

Builds the location list, presents `showQuickPick`, and opens the selected file:

```typescript
private async _runPicker(): Promise<void> {
    const locations = this._getMemoryLocations();
    const items = await this._buildQuickPickItems(locations);
    const selected = await vscode.window.showQuickPick(items, {
        title: vscode.l10n.t('Claude Memory'),
        placeHolder: vscode.l10n.t('Select memory file to edit'),
        ignoreFocusOut: true,
    });
    if (selected) {
        await this._openOrCreateMemoryFile(selected.location);
    }
}
```

### `_getMemoryLocations()` — Scope Discovery

Enumerates three scope types across all workspace folders:

| Scope | Path | Notes |
|-------|------|-------|
| `user` | `~/.claude/CLAUDE.md` | Single global file; display path uses `~` abbreviation |
| `project` | `<workspace>/.claude/CLAUDE.md` | Per workspace folder; labeled with folder name in multi-root |
| `local` | `<workspace>/.claude/CLAUDE.local.md` | Per workspace folder; intended for `.gitignore` |

In multi-root workspaces, project and local entries are duplicated per folder with the folder name appended to the label.

### `_buildQuickPickItems()` — Existence-Aware Rendering

Each location is checked for file existence. Items display `$(file)` for existing files and `$(file-add)` with a "(will be created)" suffix for absent ones.

### `_openOrCreateMemoryFile()` — Create-and-Open

```typescript
private async _openOrCreateMemoryFile(location: MemoryLocation): Promise<void> {
    const exists = await this._fileExists(location.path);
    if (!exists) {
        const dir = URI.joinPath(location.path, '..');
        await createDirectoryIfNotExists(this.fileSystemService, dir);
        const template = this._getTemplate(location.type);
        await this.fileSystemService.writeFile(location.path, new TextEncoder().encode(template));
    }
    const doc = await vscode.workspace.openTextDocument(vscode.Uri.file(location.path.fsPath));
    await vscode.window.showTextDocument(doc);
}
```

Directory creation is handled via `createDirectoryIfNotExists` before writing the template.

### `_getTemplate()` — Scope-Specific Templates

Each scope receives a distinct Markdown template:

| Scope | Header | Opening Line |
|-------|--------|-------------|
| `user` | `# User Memory` | "Instructions here apply to all projects." |
| `project` | `# Project Memory` | "Instructions here apply to this project and are shared with team members." |
| `local` | `# Project Memory (Local)` | "Instructions here apply to this project but should not be checked into version control." |

## Scope Model

The three-tier scope model mirrors the Claude CLI's memory hierarchy:

| Scope | File | Version Control | Audience |
|-------|------|-----------------|----------|
| **User** | `~/.claude/CLAUDE.md` | N/A (home directory) | Individual developer, all projects |
| **Project** | `<workspace>/.claude/CLAUDE.md` | Committed | Team-shared project context |
| **Local** | `<workspace>/.claude/CLAUDE.local.md` | Gitignored | Individual developer, single project |

### Multi-Root Workspace Handling

When multiple workspace folders exist:

- A single **user** entry appears (global to the developer).
- **Project** and **local** entries are generated per workspace folder.
- Labels include the folder name: `"Project memory - {folderName}"` vs. `"Project memory"` for single-folder workspaces.

## File Path Resolution

### Paths Exposed in QuickPick

| Scope | Path Construction |
|-------|-------------------|
| User | `URI.joinPath(envService.userHome, '.claude', 'CLAUDE.md')` |
| Project | `URI.joinPath(workspaceFolder, '.claude', 'CLAUDE.md')` |
| Local | `URI.joinPath(workspaceFolder, '.claude', 'CLAUDE.local.md')` |

### Paths Tracked but Not Exposed

The `ClaudeSettingsChangeTracker` registers additional root-level variants for change detection:

```typescript
paths.push(URI.joinPath(folder, 'CLAUDE.md'));       // root-level variant
paths.push(URI.joinPath(folder, 'CLAUDE.local.md')); // root-level variant
```

These four additional paths per workspace folder (`<workspace>/CLAUDE.md`, `<workspace>/CLAUDE.local.md`) are monitored for changes but are **not** surfaced in the `/memory` QuickPick. This asymmetry means the SDK may read root-level CLAUDE.md files, and the extension will detect their modification, but the `/memory` command will not offer to create or open them.

## Settings Change Tracking

`ClaudeSettingsChangeTracker` (211 lines) monitors CLAUDE.md files for external modifications and triggers session restarts when changes are detected.

### Mechanism

1. **`takeSnapshot()`** — Records the mtime of all resolved URIs at a point in time.
2. **`hasChanges()`** — Lazily checks each URI via an async generator; returns `true` on the first file whose mtime differs from the snapshot.
3. **Path resolution** — A registered callback supplies the current set of CLAUDE.md paths (user, project, local, and root-level variants).
4. **Session restart** — When changes are detected, the tracker signals the Claude Agent session to restart, ensuring the SDK re-reads updated memory content.

### Registered Paths

The path resolver registered in `claudeCodeAgent.ts` covers:

```
~/.claude/CLAUDE.md
<workspace>/.claude/CLAUDE.md
<workspace>/.claude/CLAUDE.local.md
<workspace>/CLAUDE.md
<workspace>/CLAUDE.local.md
```

This is a superset of the QuickPick locations — the tracker monitors more paths than the slash command exposes.

## Prompt Integration

The VS Code extension does **not** inject CLAUDE.md content into prompts. The prompt integration pathway is:

1. `claudeCodeAgent.ts` constructs SDK `Options` with `settingSources: ['user', 'project', 'local']` and the correct `cwd` / `additionalDirectories`.
2. The `@anthropic-ai/claude-agent-sdk` reads CLAUDE.md files based on these settings.
3. The SDK injects their content into the system prompt as part of the `claude_code` preset.
4. VS Code's responsibility is limited to: providing correct paths, detecting modifications, and restarting the session when files change.

```typescript
const options: Options = {
    cwd,
    additionalDirectories,
    systemPrompt: { type: 'preset', preset: 'claude_code' },
    settingSources: ['user', 'project', 'local'],
};
```

## Feature Gating

The `/memory` command has no independent feature flag. Its availability is governed entirely by the Claude Agent integration gate:

- **Configuration key:** `config.github.copilot.chat.claudeAgent.enabled`
- **Command ID:** `copilot.claude.memory` (registered in both `contributes.commands` and chat session commands)
- When the Claude Agent is disabled, the entire slash command infrastructure — including `/memory` — is unavailable.

## Comparison with copilot_memory

| Dimension | `/memory` (Claude) | `copilot_memory` |
|-----------|--------------------|--------------------|
| **Storage** | Real files (CLAUDE.md) | Virtual filesystem under `/memories/` |
| **Scopes** | user / project / local | user / session / repo |
| **Persistence** | Indefinite (files on disk) | Session scope cleared after conversation |
| **Access** | Human edits in editor | LLM tool calls |
| **Prompt injection** | SDK reads natively via settingSources | `MemoryContextPrompt` TSX component |
| **Feature gate** | `claudeAgent.enabled` | `tools.memory.enabled` |
| **Version control** | Project is committed; local is gitignored | Repo stored locally, not committed |
| **Creation** | QuickPick with templates | Tool creates files programmatically |
| **Multi-root** | Per-folder project/local entries | Single repo scope across workspace |

## Source File Reference

| File | Path | Lines | Role |
|------|------|-------|------|
| memoryCommand.ts | `src/extension/chatSessions/claude/vscode-node/slashCommands/memoryCommand.ts` | ~230 | Slash command handler with QuickPick workflow |
| claudeSlashCommandRegistry.ts | `src/extension/chatSessions/claude/vscode-node/slashCommands/claudeSlashCommandRegistry.ts` | 104 | Static registry pattern for slash commands |
| claudeSlashCommandService.ts | `src/extension/chatSessions/claude/vscode-node/claudeSlashCommandService.ts` | ~140 | Lazy instantiation, dispatch, VS Code command registration |
| index.ts | `src/extension/chatSessions/claude/vscode-node/slashCommands/index.ts` | 17 | Side-effect imports for registration |
| claudeCodeAgent.ts | `src/extension/chatSessions/claude/node/claudeCodeAgent.ts` | ~490+ | SDK Options construction with settingSources and path resolvers |
| claudeSettingsChangeTracker.ts | `src/extension/chatSessions/claude/node/claudeSettingsChangeTracker.ts` | ~210 | Mtime-based change detection, session restart trigger |

## Replication Notes

1. **Side-effect registration** — The registry pattern relies on module-level `registerClaudeSlashCommand()` calls triggered by `import` statements in `index.ts`. Any new slash command must be imported there or it will not be discovered.

2. **Asymmetric path coverage** — The change tracker monitors root-level CLAUDE.md variants (`<workspace>/CLAUDE.md`) that the QuickPick does not expose. If replicating, decide whether the editing UI should cover the full set of monitored paths.

3. **SDK-owned prompt injection** — The extension deliberately does not read or inject CLAUDE.md content. It provides `settingSources` and `cwd` to the SDK, which handles file discovery and system prompt construction. Replicating this pattern requires the downstream SDK to support an equivalent mechanism.

4. **Fire-and-forget handle()** — The `handle()` method returns immediately while the picker runs asynchronously. Errors are caught and displayed via `showErrorMessage`, not propagated through the chat result. This is intentional — the chat response stream receives only the "Opening memory file picker..." message.

5. **Template encoding** — Files are written via `new TextEncoder().encode(template)`, producing UTF-8. The `createDirectoryIfNotExists` utility handles parent directory creation before the write.

6. **No separate feature flag** — The `/memory` command cannot be independently enabled or disabled. It inherits the `claudeAgent.enabled` gate. Replication into a broader context would require introducing a dedicated flag if independent control is desired.

7. **Mtime-based detection** — `ClaudeSettingsChangeTracker` uses file modification timestamps, not content hashing. This is lightweight but can produce false positives if a file is touched without content changes (e.g., `touch CLAUDE.md`).
