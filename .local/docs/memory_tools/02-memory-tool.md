# MemoryTool — Implementation Reference

## Class Overview

`MemoryTool` implements `ICopilotTool<MemoryToolParams>` and is the core handler invoked when the LLM calls the `copilot_memory` tool. It manages all six file operations across three storage scopes, routing to either local filesystem or CAPI endpoints.

**Source:** `src/extension/tools/node/memoryTool.tsx`

## Type Definitions

```typescript
type MemoryScope = 'user' | 'session' | 'repo';

interface IViewParams {
    command: 'view';
    path: string;
    view_range?: [number, number];
}

interface ICreateParams {
    command: 'create';
    path: string;
    file_text: string;
}

interface IStrReplaceParams {
    command: 'str_replace';
    path: string;
    old_str: string;
    new_str: string;
}

interface IInsertParams {
    command: 'insert';
    path: string;
    insert_line: number;
    insert_text?: string;
    new_str?: string;  // Models sometimes send this instead of insert_text
}

interface IDeleteParams {
    command: 'delete';
    path: string;
}

interface IRenameParams {
    command: 'rename';
    old_path?: string;
    new_path: string;
    path?: string;  // Models sometimes send this instead of old_path
}

type MemoryToolParams = IViewParams | ICreateParams | IStrReplaceParams
    | IInsertParams | IDeleteParams | IRenameParams;

type MemoryToolOutcome = 'success' | 'error' | 'notFound' | 'notEnabled';

interface MemoryToolResult {
    text: string;
    outcome: MemoryToolOutcome;
}
```

## Constants

```typescript
const MEMORY_BASE_DIR = 'memory-tool/memories';
const REPO_PATH_PREFIX = '/memories/repo';
const SESSION_PATH_PREFIX = '/memories/session';
```

## Injected Dependencies

| Service | Purpose |
|---------|---------|
| `IFileSystemService` | Read/write/delete file operations |
| `IAgentMemoryService` | CAPI-backed repo memory operations |
| `IMemoryCleanupService` | Session file access tracking and TTL cleanup |
| `IVSCodeExtensionContext` | Provides `globalStorageUri` and `storageUri` for path resolution |
| `ILogService` | Structured logging |
| `IConfigurationService` | Feature flag evaluation |
| `IExperimentationService` | Experiment-based config resolution |
| `ITelemetryService` | Event tracking |

## Constructor

On construction, if the memory tool is enabled, the cleanup service is started:

```typescript
constructor(...services) {
    if (this.configurationService.getExperimentBasedConfig(
        ConfigKey.MemoryToolEnabled, this.experimentationService)) {
        this.memoryCleanupService.start();
    }
}
```

## Entry Point: `invoke()`

```typescript
async invoke(options, _token): Promise<LanguageModelToolResult> {
    const params = options.input;
    const sessionResource = options.chatSessionResource?.toString();
    const resultText = await this._dispatch(params, sessionResource,
        options.chatRequestId, options.model);
    return new LanguageModelToolResult([new LanguageModelTextPart(resultText)]);
}
```

The tool receives `chatSessionResource` (a `vscode-chat-session://local/<sessionId>` URI) from the VS Code chat framework, which is used to scope session memory to the current conversation.

## Dispatch Logic: `_dispatch()`

The routing algorithm:

1. Extract path from params (handle `rename` using `old_path` or `path`)
2. Validate path with `validatePath()`
3. Route based on path prefix:
   - `/memories/repo/*` → check CAPI enablement → either `_dispatchRepoCAPI` or `_dispatchLocal('repo')`
   - `/memories/session/*` → `_dispatchLocal('session')`
   - `/memories/*` (anything else) → `_dispatchLocal('user')`

```typescript
private async _dispatch(params, sessionResource?, requestId?, model?): Promise<string> {
    const path = params.command === 'rename'
        ? (params.old_path ?? params.path) : params.path;

    if (!path) return 'Error: Missing required path parameter.';

    const pathError = validatePath(path);
    if (pathError) return pathError;

    if (isRepoPath(path)) {
        const capiEnabled = await this.agentMemoryService.checkMemoryEnabled();
        if (capiEnabled) {
            const result = await this._dispatchRepoCAPI(params, path);
            return result.text;
        }
        // Fall back to local repo memory
        const result = await this._dispatchLocal(params, 'repo', sessionResource);
        return result.text;
    }

    const scope: MemoryScope = isSessionPath(path) ? 'session' : 'user';
    const result = await this._dispatchLocal(params, scope, sessionResource);
    return result.text;
}
```

## Path Validation: `validatePath()`

Security-critical function that prevents path traversal:

```typescript
function validatePath(path: string): string | undefined {
    // Must start with /memories/
    if (!normalizePath(path).startsWith('/memories/')) {
        return 'Error: All memory paths must start with /memories/';
    }
    // No .. traversal
    if (path.includes('..')) {
        return 'Error: Path traversal is not allowed';
    }
    // No . segments (e.g., /memories/./etc)
    const segments = path.split('/').filter(s => s.length > 0);
    if (segments.some(s => s === '.')) {
        return 'Error: Path traversal is not allowed';
    }
    // First segment must be "memories"
    if (segments[0] !== 'memories') {
        return 'Error: All memory paths must start with /memories/';
    }
    return undefined;
}
```

## Scope Detection

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
```

## Local Operations

### `_dispatchLocal()`

Routes to the appropriate handler based on command:

```typescript
switch (params.command) {
    case 'view':      return this._localView(params.path, params.view_range, scope, sessionResource);
    case 'create':    return this._localCreate(params, scope, sessionResource);
    case 'str_replace': return this._localStrReplace(params, scope, sessionResource);
    case 'insert':    return this._localInsert(params, scope, sessionResource);
    case 'delete':    return this._localDelete(params.path, scope, sessionResource);
    case 'rename':    return this._localRename(params, scope, sessionResource);
}
```

### `view` Command

1. If viewing `/memories/` root with user scope → delegates to `_localViewMergedRoot()` (merges all scopes)
2. Resolves URI, stats the path
3. If directory → `_listDirectory()` (recursive up to 2 levels, excludes hidden files and `repo/` dir). `_listDirectory` is called for scope-level directory views (e.g., `/memories/session/`). The merged root view (`/memories/`) is handled separately by `_localViewMergedRoot`, which has its own listing logic combining user, session, and repo scopes.
4. If file → reads content, formats with line numbers
5. If `view_range` provided → validates range, returns the requested slice

**Output format (file):**
```
Here's the content of /memories/notes.md with line numbers:
     1	# Notes
     2	- Important thing
     3	- Another thing
```

**Output format (directory):**
```
Here are the files and directories up to 2 levels deep in /memories/, excluding hidden items:
preferences/
  42	/memories/preferences/coding.md
123	/memories/notes.md
```

### `_localViewMergedRoot()` — Root Directory Listing

When the LLM views `/memories/`, this method merges content from all three scopes into a single listing:

1. **User files** — Lists files from `globalStorageUri/memory-tool/memories/`
2. **Session files** — Lists files from the current session's directory (always shows `/memories/session/` even if empty)
3. **Repo files** — Lists files from local repo storage (only when CAPI is disabled)

Hidden files (starting with `.`) are excluded from all listings.

### `create` Command

1. Resolves URI from virtual path
2. **Checks if file already exists** → returns error if it does
3. Creates parent directory if needed (`createDirectoryIfNotExists`)
4. Writes file content
5. Marks session files as accessed for cleanup tracking

### `str_replace` Command

1. Reads file content
2. Searches for all occurrences of `old_str`
3. **Exactly one match required** — zero matches returns "not found", multiple returns "ambiguous"
4. Replaces the single occurrence and writes back
5. Returns a snippet showing the edit context (4 lines before/after)

### `insert` Command

1. Reads file content, splits into lines
2. Validates `insert_line` is within `[0, nLines]`
3. Uses `splice()` to insert new lines at the specified position
4. Writes back and returns snippet

Note: Handles model quirk where `new_str` is sent instead of `insert_text`:
```typescript
const insertText = params.insert_text ?? params.new_str;
```

### `delete` Command

1. Checks path exists via `stat()`
2. Deletes with `{ recursive: true }` (supports both files and directories)

### `rename` Command

1. Validates both source and destination paths
2. **Prevents cross-scope renames** — checks that old and new paths resolve to the same scope
3. Verifies source exists and destination does not exist
4. Creates destination parent directory if needed
5. Performs rename via `fileSystemService.rename()`

## CAPI Operations (Repository Memory)

### `_dispatchRepoCAPI()`

Only supports `create`. All other commands return an error:
```
Error: The '<command>' operation is not supported for repository memories at <path>.
Only 'create' is allowed for /memories/repo/.
```

### `_repoCreate()`

1. Extracts a hint from the filename (e.g., `/memories/repo/testing.json` → `"testing"`)
2. Attempts to parse `file_text` as JSON → extracts `subject`, `fact`, `citations`, `reason`, `category`
3. If JSON parsing fails → treats entire `file_text` as a plain text fact
4. Calls `agentMemoryService.storeRepoMemory(entry)`

## UI Messages: `prepareInvocation()`

Returns localized messages for the VS Code chat UI:

| Command | In-progress | Past tense |
|---------|-------------|------------|
| `view` | "Reading memory {file}" | "Read memory {file}" |
| `create` | "Creating memory file {file}" | "Created memory file {file}" |
| `str_replace` | "Updating memory file {file}" | "Updated memory file {file}" |
| `insert` | "Inserting into memory file {file}" | "Inserted into memory file {file}" |
| `delete` | "Deleting memory {file}" | "Deleted memory {file}" |
| `rename` | "Renaming memory {file}" | "Renamed memory {file}" |

Directory-level operations show verb only without a file widget.

## Output Formatting Helpers

### `formatFileContent(path, content)`
```
Here's the content of <path> with line numbers:
     1	<line 1>
     2	<line 2>
```

### `makeSnippet(fileContent, editLine, path)`
Shows 4 lines before and after the edit point:
```
The memory file has been edited. Here's the result of running `cat -n` on a snippet of <path>:
     5	context line
     6	context line
     7	THE EDITED LINE
     8	context line
     9	context line
```

## ResolveMemoryFileUriTool

A simpler companion tool that translates `/memories/...` paths to actual filesystem URIs.

**Purpose:** When the LLM needs to reference a memory file by its real URI (e.g., for `setArtifacts`), it calls this tool to resolve the virtual path.

**Implementation:** Same path resolution logic as `MemoryTool._resolveUri()`:
- `/memories/session/**` → `storageUri/memory-tool/memories/<sessionId>/**`
- `/memories/repo/**` → `storageUri/memory-tool/memories/repo/**`
- `/memories/**` → `globalStorageUri/memory-tool/memories/**`

Returns the resolved URI as a string via `LanguageModelTextPart`.
