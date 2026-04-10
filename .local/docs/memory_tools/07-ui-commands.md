# VS Code UI Commands

## Overview

Two VS Code commands provide user-facing memory management. Both are registered in the `ToolsContribution` class at `src/extension/tools/vscode-node/tools.ts` and gated behind `config.github.copilot.chat.tools.memory.enabled`.

## `showMemories` — Browse Memory Files

**Command ID:** `github.copilot.chat.tools.memory.showMemories`
**Category:** Chat
**Trigger:** Command Palette → "Chat: Show Memories"

### Behavior

Opens a QuickPick listing all memory files across all three scopes. Selecting a file opens it in the editor.

### Implementation

```typescript
vscode.commands.registerCommand('github.copilot.chat.tools.memory.showMemories', async () => {
    const globalStorageUri = this.extensionContext.globalStorageUri;
    const storageUri = this.extensionContext.storageUri;

    interface MemoryItem extends vscode.QuickPickItem {
        fileUri?: URI;
    }

    const items: MemoryItem[] = [];

    // 1. Collect user-scoped memories
    if (globalStorageUri) {
        const userMemoryUri = URI.joinPath(globalStorageUri, 'memory-tool/memories');
        try {
            const entries = await vscode.workspace.fs.readDirectory(vscode.Uri.from(userMemoryUri));
            const fileEntries = entries.filter(([name, type]) =>
                type === vscode.FileType.File && !name.startsWith('.'));
            if (fileEntries.length > 0) {
                items.push({ label: '/memories', kind: vscode.QuickPickItemKind.Separator });
                for (const [name] of fileEntries) {
                    items.push({
                        label: `$(file) ${name}`,
                        description: 'user',
                        fileUri: URI.joinPath(userMemoryUri, name),
                    });
                }
            }
        } catch { /* Directory may not exist */ }
    }

    // 2. Collect local repo memories (only when CAPI disabled)
    const capiMemoryEnabled = this.configurationService.getExperimentBasedConfig(
        ConfigKey.CopilotMemoryEnabled, this.experimentationService);
    if (storageUri && !capiMemoryEnabled) {
        const repoMemoryUri = URI.joinPath(storageUri, 'memory-tool/memories/repo');
        try {
            const entries = await this.fileSystemService.readDirectory(repoMemoryUri);
            const fileEntries = entries.filter(([name, type]) =>
                type === FileType.File && !name.startsWith('.'));
            if (fileEntries.length > 0) {
                items.push({ label: '/memories/repo', kind: vscode.QuickPickItemKind.Separator });
                for (const [name] of fileEntries) {
                    items.push({
                        label: `$(file) ${name}`,
                        description: 'repo',
                        fileUri: URI.joinPath(repoMemoryUri, name),
                    });
                }
            }
        } catch { /* Directory may not exist */ }
    }

    // 3. Collect session memories (current session only)
    const sessionResource = vscode.window.activeChatPanelSessionResource;
    if (storageUri && sessionResource) {
        const sessionId = extractSessionId(sessionResource.toString());
        const sessionMemoryUri = URI.joinPath(storageUri, 'memory-tool/memories', sessionId);
        try {
            const entries = await vscode.workspace.fs.readDirectory(
                vscode.Uri.from(sessionMemoryUri));
            const fileEntries = entries.filter(([name, type]) =>
                type === vscode.FileType.File && !name.startsWith('.'));
            if (fileEntries.length > 0) {
                items.push({ label: '/memories/session', kind: vscode.QuickPickItemKind.Separator });
                for (const [name] of fileEntries) {
                    items.push({
                        label: `$(file) ${name}`,
                        description: 'session',
                        fileUri: URI.joinPath(sessionMemoryUri, name),
                    });
                }
            }
        } catch { /* Directory may not exist */ }
    }

    // 4. Show QuickPick or "no memories" message
    if (items.length === 0) {
        vscode.window.showInformationMessage('No memories found.');
        return;
    }

    const selected = await vscode.window.showQuickPick(items, {
        title: 'Memory',
        placeHolder: 'Select a memory file to view',
    });

    if (selected?.fileUri) {
        await vscode.commands.executeCommand('vscode.open', vscode.Uri.from(selected.fileUri));
    }
});
```

### QuickPick Structure

```
/memories                     ← Separator
  $(file) notes.md           user
  $(file) debugging.md       user

/memories/repo                ← Separator (only when CAPI disabled)
  $(file) conventions.md     repo

/memories/session             ← Separator
  $(file) plan.md            session
```

### Notes

- Uses `vscode.window.activeChatPanelSessionResource` to get the current chat session for session memory lookup
- Session memory only shows files for the active conversation
- Hidden files (starting with `.`) are excluded
- When CAPI is enabled, repo memories are cloud-stored and not shown in the QuickPick (they have no local files)
- The implementation mixes the VS Code workspace filesystem API (`vscode.workspace.fs`) and the injected `IFileSystemService`. Both resolve to the same underlying implementation in the extension context — the inconsistency is a code style artifact, not a behavioral difference.

## `clearMemories` — Delete All Memory Files

**Command ID:** `github.copilot.chat.tools.memory.clearMemories`
**Category:** Chat
**Trigger:** Command Palette → "Chat: Clear Memories"

### Behavior

Shows a destructive confirmation dialog, then recursively deletes all memory directories.

### Implementation

```typescript
vscode.commands.registerCommand('github.copilot.chat.tools.memory.clearMemories', async () => {
    // 1. Confirmation dialog
    const confirm = await vscode.window.showWarningMessage(
        l10n.t('Are you sure you want to clear all memories? This cannot be undone.'),
        { modal: true },
        l10n.t('Clear All'),
    );
    if (confirm !== l10n.t('Clear All')) return;

    const globalStorageUri = this.extensionContext.globalStorageUri;
    const storageUri = this.extensionContext.storageUri;
    let hasError = false;
    let hasDeleted = false;

    // 2. Delete user-scoped memories (globalStorageUri/memory-tool/memories/)
    if (globalStorageUri) {
        const userMemoryUri = URI.joinPath(globalStorageUri, 'memory-tool/memories');
        try {
            await vscode.workspace.fs.delete(vscode.Uri.from(userMemoryUri), { recursive: true });
            hasDeleted = true;
        } catch (e) {
            if (e instanceof vscode.FileSystemError && e.code === 'FileNotFound') {
                // Nothing to delete
            } else {
                hasError = true;
            }
        }
    }

    // 3. Delete ALL session and local repo memories
    //    (storageUri/memory-tool/memories/ — contains both session dirs and repo/)
    if (storageUri) {
        const sessionMemoryUri = URI.joinPath(storageUri, 'memory-tool/memories');
        try {
            await vscode.workspace.fs.delete(vscode.Uri.from(sessionMemoryUri), { recursive: true });
            hasDeleted = true;
        } catch (e) {
            if (e instanceof vscode.FileSystemError && e.code === 'FileNotFound') {
                // Nothing to delete
            } else {
                hasError = true;
            }
        }
    }

    // 4. Report result
    if (hasError) {
        vscode.window.showErrorMessage('Some memories could not be cleared. Please try again.');
    } else if (!hasDeleted) {
        vscode.window.showInformationMessage('No memories found.');
    } else {
        vscode.window.showInformationMessage('All memories have been cleared.');
    }
});
```

### Deletion Scope

| Directory | Deleted? | Notes |
|-----------|----------|-------|
| `globalStorageUri/memory-tool/memories/` | Yes | Entire user memory tree |
| `storageUri/memory-tool/memories/` | Yes | All session directories + local repo |

This is a complete nuclear option — it removes all local memory state. CAPI-backed repo memories are NOT affected (they live in the cloud).

### Error Handling

- `FileNotFound` errors are silently ignored (nothing to delete)
- Other errors set `hasError = true` but do not abort — both directories are attempted
- Success/error messages are shown after all operations complete
