# CAPI Service — Cloud-Backed Repository Memory

## Overview

The `AgentMemoryService` manages repository-scoped memories stored in the cloud via the Copilot API (CAPI). When enabled, `/memories/repo/` paths route to this service instead of local file storage. This allows memories to persist across machines and be shared (implicitly) through the same repository context.

**Source:** `src/extension/tools/common/agentMemoryService.ts`

## Interface

```typescript
interface IAgentMemoryService {
    readonly _serviceBrand: undefined;

    /** Check if Copilot Memory is enabled for the current repository. */
    checkMemoryEnabled(): Promise<boolean>;

    /** Get repo memories from Copilot Memory service. */
    getRepoMemories(limit?: number): Promise<RepoMemoryEntry[] | undefined>;

    /** Store a repo memory to Copilot Memory service. */
    storeRepoMemory(memory: RepoMemoryEntry): Promise<boolean>;
}
```

## Data Model

```typescript
interface RepoMemoryEntry {
    subject: string;           // Topic/entity the fact is about
    fact: string;              // The factual statement
    citations?: string | string[];  // File paths or references supporting the fact
    reason?: string;           // Why this fact is worth remembering
    category?: string;         // Classification (e.g., "testing", "conventions")
}
```

### Citations Format

Citations are polymorphic for backward compatibility:

```typescript
function normalizeCitations(citations: string | string[] | undefined): string[] | undefined {
    if (citations === undefined) return undefined;
    if (typeof citations === 'string') {
        // Legacy format: comma-separated string
        return citations.split(',').map(c => c.trim()).filter(c => c.length > 0);
    }
    return citations;  // Already an array
}
```

### Validation

```typescript
function isRepoMemoryEntry(obj: unknown): obj is RepoMemoryEntry {
    if (typeof obj !== 'object' || obj === null) return false;
    const entry = obj as Record<string, unknown>;

    // Required: subject and fact must be strings
    if (typeof entry.subject !== 'string' || typeof entry.fact !== 'string') return false;

    // Optional: citations must be string or string[]
    if (entry.citations !== undefined) {
        const isString = typeof entry.citations === 'string';
        const isStringArray = Array.isArray(entry.citations)
            && entry.citations.every(c => typeof c === 'string');
        if (!isString && !isStringArray) return false;
    }

    // Optional: reason and category must be strings
    if (entry.reason !== undefined && typeof entry.reason !== 'string') return false;
    if (entry.category !== undefined && typeof entry.category !== 'string') return false;

    return true;
}
```

## Dependencies

```typescript
class AgentMemoryService extends Disposable implements IAgentMemoryService {
    constructor(
        @ILogService private readonly logService,
        @ICAPIClientService private readonly capiClientService,
        @IGitService private readonly gitService,
        @IWorkspaceService private readonly workspaceService,
        @IConfigurationService private readonly configService,
        @IExperimentationService private readonly experimentationService,
        @IAuthenticationService private readonly authenticationService,
    ) { }
}
```

## API Endpoints

All requests go through `ICAPIClientService.makeRequest()` with `RequestType.CopilotAgentMemory`.

> **Caveat:** The exact API URLs are constructed by the CAPI client layer (`ICAPIClientService.makeRequest()` with `RequestType.CopilotAgentMemory`). The paths shown below are inferred from the `RequestType` enum and parameter patterns, not from literal URL strings in the source code. **The actual endpoint format may differ.**

### Enable Check

```
GET /copilot-agent-memory/{nwo}/enabled

Headers:
  Authorization: Bearer <github-oauth-token>

Response: { "enabled": boolean }
```

### Fetch Recent Memories

```
GET /copilot-agent-memory/{nwo}/recent?limit={n}

Headers:
  Authorization: Bearer <github-oauth-token>

Response: Array<{
  subject: string,
  fact: string,
  citations?: string[],
  reason?: string,
  category?: string
}>
```

### Store Memory

```
PUT /copilot-agent-memory/{nwo}

Headers:
  Authorization: Bearer <github-oauth-token>
Content-Type: application/json

Body: {
  "subject": "...",
  "fact": "...",
  "citations": ["..."],
  "reason": "...",
  "category": "...",
  "source": { "agent": "vscode" }
}
```

## Repository NWO Resolution

The service derives the GitHub owner/repo ("name with owner") from git remotes:

```typescript
private async getRepoNwo(): Promise<string | undefined> {
    const workspaceFolders = this.workspaceService.getWorkspaceFolders();
    if (!workspaceFolders || workspaceFolders.length === 0) return undefined;

    const repo = await this.gitService.getRepository(workspaceFolders[0]);
    if (!repo) return undefined;

    for (const remoteUrl of getOrderedRemoteUrlsFromContext(repo)) {
        const repoId = getGithubRepoIdFromFetchUrl(remoteUrl);
        if (repoId) return toGithubNwo(repoId);  // e.g., "microsoft/vscode"
    }
    return undefined;
}
```

**Dependencies:**
- `getOrderedRemoteUrlsFromContext(repo)` — Returns remote URLs prioritised (likely `origin` first)
- `getGithubRepoIdFromFetchUrl(url)` — Parses owner/repo from a GitHub remote URL
- `toGithubNwo(repoId)` — Formats as lowercase `owner/repo`

## Method Implementations

### `checkMemoryEnabled()`

```
1. Check config: CopilotMemoryEnabled experiment flag
   → false → return false
2. Resolve repo NWO from git remotes
   → null → return false
3. Get GitHub OAuth session (silent)
   → null → return false
4. GET /copilot-agent-memory/{nwo}/enabled
   → !ok → return false
5. Parse response: { enabled: boolean }
   → return enabled
```

### `getRepoMemories(limit = 10)`

```
1. checkMemoryEnabled()
   → false → return undefined
2. Resolve repo NWO
   → null → return undefined
3. Get GitHub OAuth session (silent)
   → null → return undefined
4. GET /copilot-agent-memory/{nwo}/recent
   → !ok → return undefined
5. Parse response as Array
   → filter through isRepoMemoryEntry
   → map to RepoMemoryEntry objects
6. return memories (or undefined if empty)
```

### `storeRepoMemory(memory)`

```
1. checkMemoryEnabled()
   → false → return false
2. Resolve repo NWO
   → null → return false
3. Normalize citations to string[]
4. Get GitHub OAuth session (silent)
   → null → return false
5. PUT /copilot-agent-memory/{nwo}
   Body: { subject, fact, citations, reason, category, source: { agent: 'vscode' } }
   → !ok → return false
6. return true
```

## How MemoryTool Uses CAPI

The `MemoryTool._repoCreate()` method bridges the LLM's file-creation interface to the structured CAPI entry format:

```typescript
private async _repoCreate(params: ICreateParams): Promise<MemoryToolResult> {
    const filename = params.path.split('/').pop() || 'memory';
    const pathHint = filename.replace(/\.\w+$/, '');

    let entry: RepoMemoryEntry;
    try {
        // Attempt JSON parse
        const parsed = JSON.parse(params.file_text);
        entry = {
            subject: parsed.subject || pathHint,
            fact: parsed.fact || '',
            citations: parsed.citations || '',
            reason: parsed.reason || '',
            category: parsed.category || pathHint,
        };
    } catch {
        // Plain text fallback
        entry = {
            subject: pathHint,
            fact: params.file_text,
            citations: '',
            reason: 'Stored from memory tool create command.',
            category: pathHint,
        };
    }

    const success = await this.agentMemoryService.storeRepoMemory(entry);
    if (success) return { text: `File created successfully at: ${params.path}`, outcome: 'success' };
    else return { text: 'Error: Failed to store repository memory entry.', outcome: 'error' };
}
```

**Key design decisions:**
- The LLM creates a "file" with structured JSON content → parsed into a `RepoMemoryEntry`
- If the LLM sends plain text → the text becomes the `fact`, filename becomes the `subject`
- Only `create` is supported for CAPI — view/delete/update operations are not available
- The LLM sees the same success message as a local file creation

## How MemoryContextPrompt Uses CAPI

During prompt construction, the `MemoryContextPrompt` calls `getRepoMemories()` to fetch and render cloud-stored facts:

```typescript
const repoMemories = enableCopilotMemory
    ? await this.agentMemoryService.getRepoMemories()
    : undefined;
```

These are rendered as a `<repository_memories>` tag with subject/fact/citations/reason formatting (see [04-prompt-integration.md](04-prompt-integration.md)).

## Service Registration

```typescript
// src/extension/extension/vscode-node/services.ts
builder.define(IAgentMemoryService, new SyncDescriptor(AgentMemoryService));
```
