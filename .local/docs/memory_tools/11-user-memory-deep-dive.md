# User Memory Scope — Exhaustive Deep Dive

## 1. Overview

User memory is the default storage scope for the `copilot_memory` tool. Any path under `/memories/` that does not match `/memories/session/` or `/memories/repo/` routes to user scope. It provides permanent, cross-workspace, cross-session note storage on the local filesystem.

**Key characteristics:**
- Persistent across all workspaces and all conversations
- No cloud sync — local filesystem only
- No TTL, no automatic cleanup, no expiration
- No access tracking (unlike session scope)
- Last-write-wins (no file locking)

User memory is the mechanism by which Copilot accumulates long-term knowledge about a user's preferences, patterns, and working conventions.

---

## 2. Storage

### Physical Locations by OS

| OS | Path |
|----|------|
| **macOS** | `~/.vscode/data/User/globalStorage/github.copilot-chat/memory-tool/memories/` |
| **Windows** | `%APPDATA%\Code\User\globalStorage\github.copilot-chat\memory-tool\memories\` |
| **Linux** | `~/.config/Code/User/globalStorage/github.copilot-chat/memory-tool/memories/` |

All paths resolve from VS Code's `globalStorageUri` for the `github.copilot-chat` extension, with `memory-tool/memories` appended.

### Encoding and Limits

| Property | Value |
|----------|-------|
| Encoding | UTF-8 (`TextDecoder`/`TextEncoder`) |
| File size limits | None enforced |
| Subdirectory support | Full — tool operations work at arbitrary depth |
| File locking | None — last-write-wins semantics |
| Hidden file handling | Excluded from directory listings and prompt injection |

---

## 3. Scope Determination

Routing occurs in `_dispatch()` (memoryTool.tsx L231). The decision tree:

```
validatePath(path)
    │
    ├─ isRepoPath(path)?     → repo scope
    │
    ├─ isSessionPath(path)?  → session scope
    │
    └─ (default)             → user scope
```

The user scope is the **fall-through** — it handles everything not explicitly claimed by repo or session. There is no `isUserPath()` predicate. The path constants:

```typescript
const REPO_PATH_PREFIX = '/memories/repo';
const SESSION_PATH_PREFIX = '/memories/session';
```

Any path under `/memories/` that does not begin with either prefix is user-scoped. This includes:
- `/memories/notes.md`
- `/memories/patterns/typescript.md`
- `/memories/debugging.md`
- `/memories/` itself (the root view)

---

## 4. Path Resolution

### `_resolveUri()` for User Scope

**Source:** `memoryTool.tsx` L404–414

```typescript
relativeSegments = segments.slice(1);  // Skip 'memories'
const globalStorageUri = this.extensionContext.globalStorageUri;
if (!globalStorageUri) {
    throw new Error('No global storage available. User memory operations require global storage.');
}
const resolved = relativeSegments.length > 0
    ? URI.joinPath(globalStorageUri, MEMORY_BASE_DIR, ...relativeSegments)
    : URI.joinPath(globalStorageUri, MEMORY_BASE_DIR);
return resolved;
```

### Worked Example

Input path: `/memories/patterns/typescript.md`

| Step | Value |
|------|-------|
| `segments` | `['memories', 'patterns', 'typescript.md']` |
| `relativeSegments` (`segments.slice(1)`) | `['patterns', 'typescript.md']` |
| URI construction | `URI.joinPath(globalStorageUri, 'memory-tool/memories', 'patterns', 'typescript.md')` |
| **Resolved** | `<globalStorageUri>/memory-tool/memories/patterns/typescript.md` |

### Security Asymmetry

**Session and repo scopes** perform post-resolution validation via `isMemoryUri()` — checking that the resolved URI falls within the expected storage directory.

**User scope does NOT perform this check.** It relies solely on `validatePath()` rejecting `..` and `.` path segments before resolution. This is a structural asymmetry: user scope trusts pre-resolution validation alone, while other scopes add a post-resolution belt-and-braces check.

---

## 5. Tool Operations

All six `copilot_memory` commands support user scope. Each subsection details the complete behavioral specification.

### 5.1 view

**Source:** `memoryTool.tsx` L303

#### Root View (`/memories/` with scope `'user'`)

When `isMemoriesRoot(path)` is true and scope is `'user'`, the view command delegates to `_localViewMergedRoot()`. This method produces a unified listing combining:
- User-scope files and directories
- Session-scope files
- Repo-scope files (when CAPI is disabled)

This merged root provides a single-pane view of all local memory content.

#### Single File View

1. Resolves URI via `_resolveUri()`
2. Stats the path
3. If **directory** → delegates to `_listDirectory()`
4. If **file** → reads content, decodes UTF-8, formats with 1-indexed line numbers (6-character left-padded)

#### `_listDirectory()` Behavior

- Maximum recursion depth: **2**
- Excludes hidden files (names starting with `.`)
- Excludes the `repo/` directory
- Sort order: directories first, then files
- Shows file sizes in listing

#### `view_range` Parameter

- **1-based line indexing**
- Validates bounds against file content
- Returns the requested slice with line numbers

#### Not Found

Returns `"No memories found in <path>."` with outcome `'notFound'`.

#### Access Tracking

`markAccessed()` is **NOT called** for user scope. Only session scope tracks access for TTL purposes.

---

### 5.2 create

**Source:** `memoryTool.tsx` L425

1. Resolves URI
2. Stats the path — if the file **already exists**, returns a hard error: `"Error: File <path> already exists"`
3. Calls `createDirectoryIfNotExists()` for the parent directory
   - Race-safe implementation: stat → create → re-stat on conflict
4. Encodes content as UTF-8 via `TextEncoder`
5. Writes via `fileSystemService.writeFile()`

**No file size limit is enforced.**

**No overwrite:** The create command refuses to overwrite existing files. To modify content, use `str_replace` or `insert`.

`markAccessed()` is **NOT called** for user scope.

---

### 5.3 str_replace

**Source:** `memoryTool.tsx` L451

1. Resolves URI, reads file content
2. **File not found** → `"The path <path> does not exist."` with outcome `'notFound'`
3. Searches using `content.indexOf()` in a loop — **exact string match, case-sensitive**
4. Match count evaluation:
   - **Zero matches** → error message advising verbatim match requirement
   - **Multiple matches** → error listing the line numbers of each occurrence
   - **Exactly one match** → performs `content.replace(old_str, new_str)`, writes back
5. Returns a snippet of ±4 lines around the replacement via `makeSnippet()`

The uniqueness requirement is strict: the `old_str` must appear exactly once in the file. Partial matches, case-insensitive matches, or regex patterns are not supported.

---

### 5.4 insert

**Source:** `memoryTool.tsx` L482

1. Accepts **either** `insert_text` or `new_str` as the content parameter
   - This dual-parameter support handles model inconsistency — some models send `new_str` instead of the schema-defined `insert_text`
2. File must exist (error if not found)
3. **0-based line indexing:**
   - `0` = insert before the first line
   - `nLines` = insert after the last line
   - Valid range: `[0, nLines]`
4. Uses `Array.splice()` for insertion
5. Returns a snippet centered on `insert_line + 1`

The 0-based indexing is notable because `view` and `view_range` use 1-based indexing. This asymmetry is a known inconsistency in the interface.

---

### 5.5 delete

**Source:** `memoryTool.tsx` L513

1. Stats the path — if not found, returns an error
2. Deletes with `{ recursive: true }` — handles both files and directories
3. **No confirmation** at the tool level — deletion is immediate

**Warning:** Deleting `/memories/` via this command would wipe all user memory. There is no guard against root-level deletion beyond `validatePath()` accepting the path as valid.

---

### 5.6 rename

**Source:** `memoryTool.tsx` L526

1. Accepts `old_path`, or falls back to `path` if `old_path` is not provided
2. Validates `new_path` via `validatePath()`
3. **Cross-scope restriction:** Derives the scope of `new_path` and errors if it differs from the source scope. A user-scope file cannot be renamed into session or repo scope.
4. Source must exist, destination must **NOT** exist
5. Creates the parent directory of the destination if needed
6. Performs `fileSystemService.rename()`

---

## 6. Prompt Injection

### `getUserMemoryContent()`

**Source:** `memoryContextPrompt.tsx` L107

The function that reads user memory files and formats them for inclusion in the system prompt.

#### Algorithm

1. Checks `globalStorageUri` — if undefined, returns `undefined`
2. Constructs `memoryDirUri = globalStorageUri + 'memory-tool/memories'`
3. Stats the directory — if it is not a directory or stat fails, returns `undefined`
4. Reads directory entries, filters to: `type === FileType.File && !name.startsWith('.')`
5. Iterates files, pushing `## <filename>` header then all content lines
6. **200-line cap is TOTAL across all files** (not per-file). Once `lines.length >= MAX_USER_MEMORY_LINES`, the loop breaks. A final `slice(0, 200)` truncates any overshoot.
7. Returns the joined string, or `undefined` if no content

#### Subdirectory Gap

**Subdirectories are NOT recursed.** Only top-level files in the memory directory are read. This means:

- `/memories/notes.md` → **included** in prompt
- `/memories/patterns/typescript.md` → **NOT included** in prompt
- `/memories/debugging/errors.md` → **NOT included** in prompt

The tool itself supports full subdirectory operations, but the prompt injection layer does not. Files stored in subdirectories are invisible to the LLM's automatic context unless explicitly viewed via the `view` command.

#### File Ordering

File ordering is **filesystem-dependent** — no explicit sort is applied. The order in which `readDirectory()` returns entries determines which files consume the 200-line budget first. This is non-deterministic across platforms and filesystems.

### Context Tag Format

```xml
<userMemory>
The following are your persistent user memory notes. These persist across all
workspaces and conversations.

## notes.md
- Prefer tabs over spaces

## debugging.md
- Always check compilation before running tests
</userMemory>
```

### Empty State

When no user memory files exist:
```
"No user preferences or notes saved yet. Use the copilot_memory tool to store persistent notes under /memories/."
```

### First-Turn Only

User memory content is rendered **only on the first turn** of a conversation, gated by the `isNewChat` check in `agentPrompt.tsx`. Subsequent turns within the same conversation do not re-inject user memory.

---

## 7. UI Commands

### `showMemories`

**Source:** `tools.ts`

- Lists only **top-level files** from `globalStorageUri/memory-tool/memories`
- Excludes hidden files
- Does **NOT** recurse subdirectories
- Separator: `/memories`
- Description: `'user'`

This exhibits the same subdirectory gap as prompt injection — files organized in folders are not surfaced.

### `clearMemories`

**Source:** `tools.ts`

- Deletes `globalStorageUri/memory-tool/memories` recursively (all user memory)
- **Also** deletes `storageUri/memory-tool/memories` (session + local repo memory)
- Uses `l10n.t('Clear All')` for confirmation string comparison
- This is a **nuclear operation** — it wipes all three local scopes, not just user memory

---

## 8. Feature Gating

| Property | Value |
|----------|-------|
| Setting | `github.copilot.chat.tools.memory.enabled` |
| Default | `true` |
| Source | `ExperimentBased` |
| Settable by | User (VS Code settings) |
| Tag | `preview` |
| `when` clause | `config.github.copilot.chat.tools.memory.enabled` |

### Behavior When Disabled

- Prompt injection skips the `<userMemory>` block
- Files remain on disk — disabling does not delete stored memories
- `invoke()` has **no internal guard** — the `when` clause on the tool registration prevents the LLM from calling the tool, but the implementation does not independently check the setting
- Public availability: **Preview — enabled for all users by default**

---

## 9. Telemetry

### `memoryToolInvoked`

Fires on every user-scope tool operation.

| Property | Type | Values |
|----------|------|--------|
| `command` | string | `'view'`, `'create'`, `'str_replace'`, `'insert'`, `'delete'`, `'rename'` |
| `scope` | string | `'user'` |
| `toolOutcome` | string | `'success'`, `'error'`, `'notFound'` |
| `requestId` | string | Copilot request identifier |
| `model` | string | Model identifier |

### `memoryContextRead`

Fires during prompt construction when user memory is evaluated for injection.

| Property | Type | Description |
|----------|------|-------------|
| `hasUserMemory` | string | Whether user memory content was found |
| `userMemoryLength` | measurement | Length of the user memory content |

---

## 10. Edge Cases

### `globalStorageUri` Undefined

| Component | Behavior |
|-----------|----------|
| Tool operations | Throws error: `"No global storage available. User memory operations require global storage."` |
| Prompt injection | Returns `undefined` — `<userMemory>` block omitted |
| UI commands | Skips user section entirely |

### Filesystem Errors

All filesystem errors are caught by the outer `try/catch` in `_dispatchLocal()`. Errors are logged via `ILogService` and returned as error strings to the LLM. No exceptions propagate to the caller.

### Very Large Files

Files are read entirely into memory with no size check. The 200-line cap limits prompt injection only — the tool itself will happily read, replace, or insert into files of arbitrary size. A sufficiently large file could cause memory pressure during read operations.

### Non-Existent Memory Directory

| Command | Behavior |
|---------|----------|
| `view` | Returns `"No memories found in <path>."` |
| `create` | Auto-creates parent directories as needed |
| `str_replace` | Returns file-not-found error |
| `insert` | Returns file-not-found error |
| `delete` | Returns path-not-found error |

---

## 11. Notable Behaviors

Seven key behavioral observations that distinguish user memory from other scopes or reveal non-obvious implementation details:

1. **Subdirectory gap:** The tool supports nested files at arbitrary depth, but prompt injection and `showMemories` only surface top-level files. Content stored in subdirectories is invisible to the LLM unless explicitly requested via `view`.

2. **200-line budget is aggregate:** The line cap applies across all user memory files combined, not per file. A single 200-line file will consume the entire budget, leaving all other files unrepresented in the prompt.

3. **No access tracking:** Unlike session scope (which calls `markAccessed()` to reset TTL), user scope does not track access. Files persist unconditionally with no lifecycle management.

4. **No `isMemoryUri()` security check:** Session and repo scopes validate resolved URIs against expected directories post-resolution. User scope does not. Security relies solely on `validatePath()` rejecting `..` and `.` segments before resolution.

5. **`clearMemories` is nuclear:** The UI command deletes all three local scopes (user, session, local repo) — not just user memory. The naming is misleading.

6. **Tool registration is unconditional:** The tool is registered at module load time regardless of the feature gate setting. The `when` clause on the registration prevents invocation, but the code is always loaded.

7. **File ordering is non-deterministic:** Prompt injection iterates files in filesystem-returned order with no sort. Which files consume the 200-line budget first is platform-dependent and unpredictable.

---

## 12. Source File Reference

| File | Relevant Lines | Purpose |
|------|---------------|---------|
| `src/extension/tools/node/memoryTool.tsx` | L231 | `_dispatch()` — scope routing |
| `src/extension/tools/node/memoryTool.tsx` | L303 | `view` command implementation |
| `src/extension/tools/node/memoryTool.tsx` | L404–414 | `_resolveUri()` — user scope path resolution |
| `src/extension/tools/node/memoryTool.tsx` | L425 | `create` command implementation |
| `src/extension/tools/node/memoryTool.tsx` | L451 | `str_replace` command implementation |
| `src/extension/tools/node/memoryTool.tsx` | L482 | `insert` command implementation |
| `src/extension/tools/node/memoryTool.tsx` | L513 | `delete` command implementation |
| `src/extension/tools/node/memoryTool.tsx` | L526 | `rename` command implementation |
| `src/extension/context/memoryContextPrompt.tsx` | L107 | `getUserMemoryContent()` — prompt injection |
| `src/extension/context/agentPrompt.tsx` | — | `isNewChat` gating for memory injection |
| `src/extension/tools/tools.ts` | — | `showMemories` and `clearMemories` UI commands |
| `package.json` | — | Tool schema, feature gate setting |

---

## 13. Replication Notes

Key implementation facts for anyone reimplementing or extending user memory:

1. **Path routing is fall-through.** There is no explicit `isUserPath()` check. Any valid path under `/memories/` that is not repo or session is user-scoped.

2. **`MEMORY_BASE_DIR` is `'memory-tool/memories'`.** This is appended to `globalStorageUri` — not `storageUri` (which is workspace-scoped).

3. **`globalStorageUri` is the anchor.** If it is undefined, all user memory operations fail. Test environments must provide it.

4. **UTF-8 throughout.** All reads use `TextDecoder`, all writes use `TextEncoder`. No BOM handling, no encoding detection.

5. **`createDirectoryIfNotExists()` is race-safe.** It uses a stat → create → re-stat pattern to handle concurrent directory creation without errors.

6. **`str_replace` uses `String.indexOf()` in a loop** to count occurrences, not a regex. The match is exact and case-sensitive. Replacement uses `String.replace()` (first occurrence only, but uniqueness is pre-validated).

7. **`insert` uses 0-based indexing** while `view_range` uses 1-based. This asymmetry is intentional (matching the tool schema) but confusing.

8. **`delete` with `{ recursive: true }`** will remove entire directory trees. There is no depth limit or safeguard beyond `validatePath()`.

9. **Prompt injection reads only top-level files.** Subdirectories are fully supported by the tool but invisible to automatic context. Users relying on organized folder structures will find their content excluded from the prompt.

10. **The 200-line cap uses `>=` for the break condition and `slice(0, 200)` as a safety net.** A file that pushes the count past 200 mid-read will include some extra lines before the slice truncates.

11. **Telemetry fires unconditionally** for tool invocations. There is no sampling or throttling for the `memoryToolInvoked` event.

12. **The feature gate is `ExperimentBased` with default `true`.** This means it can be remotely toggled via experimentation, but ships enabled for all users in preview.
