# Session Memory Cleanup Service

## Overview

The `MemoryCleanupService` manages the lifecycle of session memory files, automatically deleting files and directories older than 14 days. It runs on extension startup and operates exclusively on session-scoped storage — user memory and local repo memory are not subject to automatic cleanup.

**Source:** `src/extension/tools/common/memoryCleanupService.ts`

## Interface

```typescript
interface IMemoryCleanupService {
    readonly _serviceBrand: undefined;

    /** Marks a memory resource as recently accessed (resets its retention clock). */
    markAccessed(uri: URI): void;

    /** Starts the cleanup scheduler if not already running. */
    start(): void;

    /** Checks if a URI is within the memory storage directory (workspace or global). */
    isMemoryUri(uri: URI): boolean;
}
```

## Constants

```typescript
const RETENTION_PERIOD_MS = 14 * 24 * 60 * 60 * 1000;  // 14 days in milliseconds
const MEMORY_BASE_DIR = 'memory-tool/memories';
```

## Implementation

```typescript
class MemoryCleanupService extends Disposable implements IMemoryCleanupService {
    private readonly baseStorageUri: URI | undefined;       // workspace storage root
    private readonly globalBaseStorageUri: URI | undefined;  // global storage root
    private readonly accessTimestamps = new ResourceMap<number>();  // URI → timestamp
    private started = false;

    constructor(
        @IVSCodeExtensionContext extensionContext,
        @IFileSystemService fileSystem,
        @ILogService logService,
    ) {
        super();
        this.baseStorageUri = extensionContext.storageUri
            ? URI.joinPath(extensionContext.storageUri, MEMORY_BASE_DIR)
            : undefined;
        this.globalBaseStorageUri = extensionContext.globalStorageUri
            ? URI.joinPath(extensionContext.globalStorageUri, MEMORY_BASE_DIR)
            : undefined;
    }
}
```

## Lifecycle

### Startup

Triggered by `MemoryTool` constructor when the memory tool is enabled:

```typescript
// In MemoryTool constructor:
if (this.configurationService.getExperimentBasedConfig(ConfigKey.MemoryToolEnabled, ...)) {
    this.memoryCleanupService.start();
}
```

The `start()` method is idempotent — it runs cleanup once on first call:

```typescript
start(): void {
    if (this.started) return;
    this.started = true;
    this.cleanupStaleResources().catch(err => {
        this.logService.warn(`[MemoryCleanupService] Cleanup error: ${err}`);
    });
}
```

### Access Tracking

Every session-scoped operation in `MemoryTool` calls `markAccessed()`:

```typescript
// In MemoryTool._localView(), _localCreate(), etc:
if (scope === 'session') {
    this.memoryCleanupService.markAccessed(uri);
}
```

This stores the current timestamp in an in-memory `ResourceMap`:

```typescript
markAccessed(uri: URI): void {
    this.accessTimestamps.set(uri, Date.now());
}
```

## Cleanup Algorithm

### `cleanupStaleResources()`

Executed once at startup:

```
1. Check baseStorageUri exists and is a directory
   → No → return (nothing to clean)

2. Read all entries in memory-tool/memories/
3. Filter to directories only, excluding 'repo'
   (These are session ID directories)

4. For each session directory:
   → cleanupSessionDirectory(uri, cutoffTime)

5. For each session directory (second pass):
   → Read its contents
   → If empty → delete the directory
```

### `cleanupSessionDirectory(sessionUri, cutoffTime)`

```
For each entry in the session directory:
  1. Check in-memory timestamp (accessTimestamps map)
     → If exists and >= cutoffTime → skip (still fresh)

  2. Check filesystem mtime (fallback)
     → If mtime >= cutoffTime → update in-memory timestamp, skip

  3. Delete the stale entry
     → Remove from accessTimestamps
     → fileSystem.delete(uri, { recursive: true for directories })
```

### Cutoff Calculation

```typescript
const now = Date.now();
const cutoffTime = now - RETENTION_PERIOD_MS;  // 14 days ago
```

Any file with last access/modification before `cutoffTime` is deleted.

## URI Containment Check

Used by `MemoryTool._resolveUri()` as a security check:

```typescript
isMemoryUri(uri: URI): boolean {
    // Check workspace storage
    if (this.baseStorageUri) {
        const basePath = this.baseStorageUri.path.toLowerCase();
        const uriPath = uri.path.toLowerCase();
        if (uri.scheme === this.baseStorageUri.scheme
            && uriPath.startsWith(basePath)) {
            return true;
        }
    }
    // Check global storage
    if (this.globalBaseStorageUri) {
        const basePath = this.globalBaseStorageUri.path.toLowerCase();
        const uriPath = uri.path.toLowerCase();
        if (uri.scheme === this.globalBaseStorageUri.scheme
            && uriPath.startsWith(basePath)) {
            return true;
        }
    }
    return false;
}
```

This verifies that a resolved URI falls within either the workspace or global memory storage directories, preventing path escape regardless of how the path was constructed.

## What Gets Cleaned

| Storage Area | Cleaned? | Notes |
|-------------|----------|-------|
| Session memory (`/memories/session/`) | Yes | 14-day retention, checked on startup |
| User memory (`/memories/`) | No | Permanent, never auto-deleted |
| Local repo memory (`/memories/repo/`) | No | Explicitly excluded via `name !== 'repo'` filter |

## Service Registration

```typescript
// src/extension/extension/vscode-node/services.ts
builder.define(IMemoryCleanupService, new SyncDescriptor(MemoryCleanupService));
```
