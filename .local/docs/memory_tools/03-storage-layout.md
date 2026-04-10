# Storage Layout & Path Resolution

## Physical Storage Locations

The memory system maps virtual `/memories/` paths to real filesystem locations using two VS Code extension storage URIs:

| URI Source | VS Code API | Persistence | Typical macOS Path |
|-----------|-------------|-------------|-------------------|
| `globalStorageUri` | `context.globalStorageUri` | Permanent, cross-workspace | `~/Library/Application Support/Code/User/globalStorage/<extensionId>/` |
| `storageUri` | `context.storageUri` | Per-workspace | `~/Library/Application Support/Code/User/workspaceStorage/<workspaceHash>/<extensionId>/` |

## Directory Structure

```
globalStorageUri/
└── memory-tool/
    └── memories/                        ← User scope root
        ├── notes.md                     ← /memories/notes.md
        ├── debugging.md                 ← /memories/debugging.md
        └── patterns/
            └── typescript.md            ← /memories/patterns/typescript.md

storageUri/
└── memory-tool/
    └── memories/
        ├── <sessionId-1>/               ← Session scope (conversation 1)
        │   ├── plan.md                  ← /memories/session/plan.md
        │   └── progress.md              ← /memories/session/progress.md
        ├── <sessionId-2>/               ← Session scope (conversation 2)
        │   └── notes.md
        └── repo/                        ← Local repo scope
            ├── conventions.md           ← /memories/repo/conventions.md
            └── build-commands.md        ← /memories/repo/build-commands.md
```

## Base Directory Constant

All storage uses a fixed subdirectory name:

```typescript
const MEMORY_BASE_DIR = 'memory-tool/memories';
```

This constant appears in four files:
- `memoryTool.tsx`
- `memoryContextPrompt.tsx`
- `resolveMemoryFileUriTool.tsx`
- `memoryCleanupService.ts`

## Path Resolution Algorithm

The `_resolveUri(memoryPath, scope, sessionResource?)` method translates virtual paths to filesystem URIs:

### User Scope

```
Input:  /memories/notes.md
        scope = 'user'

Segments: ['memories', 'notes.md']
Relative: segments.slice(1) → ['notes.md']

Result: globalStorageUri / MEMORY_BASE_DIR / notes.md
        → globalStorageUri/memory-tool/memories/notes.md
```

### Session Scope

```
Input:  /memories/session/plan.md
        scope = 'session'
        sessionResource = 'vscode-chat-session://local/abc123'

Segments: ['memories', 'session', 'plan.md']
Relative: segments.slice(2) → ['plan.md']
SessionId: extractSessionId(sessionResource) → 'abc123'

Result: storageUri / MEMORY_BASE_DIR / <sessionId> / plan.md
        → storageUri/memory-tool/memories/abc123/plan.md
```

If no `sessionResource` is provided, the session ID segment is omitted:
```
Result: storageUri / MEMORY_BASE_DIR / plan.md
```

### Repository Scope (Local)

```
Input:  /memories/repo/conventions.md
        scope = 'repo'

Segments: ['memories', 'repo', 'conventions.md']
Relative: segments.slice(2) → ['conventions.md']

Result: storageUri / MEMORY_BASE_DIR / repo / conventions.md
        → storageUri/memory-tool/memories/repo/conventions.md
```

### Complete Implementation

```typescript
private _resolveUri(memoryPath: string, scope: MemoryScope, sessionResource?: string): URI {
    const pathError = validatePath(memoryPath);
    if (pathError) throw new Error(pathError);

    const segments = memoryPath.split('/').filter(s => s.length > 0);

    if (scope === 'session') {
        const storageUri = this.extensionContext.storageUri;
        if (!storageUri) throw new Error('No workspace storage available.');

        const relativeSegments = segments.slice(2);  // skip 'memories', 'session'
        const baseUri = URI.from(storageUri);

        let resolved: URI;
        if (sessionResource) {
            const sessionId = extractSessionId(sessionResource);
            resolved = relativeSegments.length > 0
                ? URI.joinPath(baseUri, MEMORY_BASE_DIR, sessionId, ...relativeSegments)
                : URI.joinPath(baseUri, MEMORY_BASE_DIR, sessionId);
        } else {
            resolved = relativeSegments.length > 0
                ? URI.joinPath(baseUri, MEMORY_BASE_DIR, ...relativeSegments)
                : URI.joinPath(baseUri, MEMORY_BASE_DIR);
        }

        // Safety check: verify resolved path is within memory storage
        if (!this.memoryCleanupService.isMemoryUri(resolved)) {
            throw new Error('Resolved path escapes the memory storage directory.');
        }
        return resolved;
    }

    if (scope === 'repo') {
        const storageUri = this.extensionContext.storageUri;
        if (!storageUri) throw new Error('No workspace storage available.');

        const relativeSegments = segments.slice(2);  // skip 'memories', 'repo'
        const baseUri = URI.from(storageUri);
        const resolved = relativeSegments.length > 0
            ? URI.joinPath(baseUri, MEMORY_BASE_DIR, 'repo', ...relativeSegments)
            : URI.joinPath(baseUri, MEMORY_BASE_DIR, 'repo');

        if (!this.memoryCleanupService.isMemoryUri(resolved)) {
            throw new Error('Resolved path escapes the memory storage directory.');
        }
        return resolved;
    }

    // User scope
    const relativeSegments = segments.slice(1);  // skip 'memories'
    const globalStorageUri = this.extensionContext.globalStorageUri;
    if (!globalStorageUri) throw new Error('No global storage available.');

    return relativeSegments.length > 0
        ? URI.joinPath(globalStorageUri, MEMORY_BASE_DIR, ...relativeSegments)
        : URI.joinPath(globalStorageUri, MEMORY_BASE_DIR);
}
```

## Session ID Extraction

Session IDs are derived from the chat session resource URI:

```typescript
export function extractSessionId(sessionResource: string): string {
    const parsed = URI.parse(sessionResource);
    // Extract the last path segment as the session ID
    const segments = parsed.path.replace(/^\//, '').split('/');
    const raw = segments[segments.length - 1] || parsed.authority || 'unknown';
    // Sanitize to only safe characters for a directory name
    return raw.replace(/[^a-zA-Z0-9_.-]/g, '_');
}
```

**Input:** `vscode-chat-session://local/abc123-def456`
**Output:** `abc123-def456`

The sanitization replaces any character not in `[a-zA-Z0-9_.-]` with an underscore, ensuring the result is safe for use as a directory name on all platforms.

## Security: Path Containment

Two mechanisms prevent path traversal:

### 1. `validatePath()` (pre-resolution)

Rejects paths containing `..`, `.` segments, or those not starting with `/memories/`.

### 2. `isMemoryUri()` (post-resolution)

The `MemoryCleanupService.isMemoryUri()` method verifies that a resolved URI falls within the expected storage directories:

```typescript
isMemoryUri(uri: URI): boolean {
    if (this.baseStorageUri) {
        const basePath = this.baseStorageUri.path.toLowerCase();
        if (uri.scheme === this.baseStorageUri.scheme
            && uri.path.toLowerCase().startsWith(basePath)) {
            return true;
        }
    }
    if (this.globalBaseStorageUri) {
        const basePath = this.globalBaseStorageUri.path.toLowerCase();
        if (uri.scheme === this.globalBaseStorageUri.scheme
            && uri.path.toLowerCase().startsWith(basePath)) {
            return true;
        }
    }
    return false;
}
```

This post-resolution check is applied only to **session** and **repo** scopes — the user scope branch does **not** call `isMemoryUri()`. The asymmetry is intentional: session and repo paths resolve under `storageUri`, which is per-workspace and could theoretically be influenced by workspace configuration, so the extra containment check is warranted. User scope resolves to `globalStorageUri`, which is a fixed, extension-controlled location outside any workspace, making the pre-resolution `validatePath()` check sufficient on its own.

## Path Prefix Constants

```typescript
const REPO_PATH_PREFIX = '/memories/repo';
const SESSION_PATH_PREFIX = '/memories/session';
```

Helper functions for scope detection:

```typescript
function isRepoPath(path: string): boolean {
    return path === REPO_PATH_PREFIX || path.startsWith(REPO_PATH_PREFIX + '/');
}

function isSessionPath(path: string): boolean {
    return path === SESSION_PATH_PREFIX || path.startsWith(SESSION_PATH_PREFIX + '/');
}

function isMemoriesRoot(path: string): boolean {
    return normalizePath(path) === '/memories/';
}

function normalizePath(path: string): string {
    return path.endsWith('/') ? path : path + '/';
}
```
