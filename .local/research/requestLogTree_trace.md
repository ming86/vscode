# requestLogTree: Complete Architecture & Trace

## Quick Summary

The `requestLogTree` feature in VS Code Copilot Chat displays chat request logs in a tree view and renders them as markdown files. These are **virtual documents** (not real files on disk) that show the raw API request/response data.

**Key URI Scheme:** `ccreq:` (Chat Content Request)
- Example: `ccreq:abc123def.copilotmd` → Virtual markdown file
- Storage: In-memory in `RequestLogger._entries[]` (max 100 entries)
- Persistence: Session only (unless manually saved to disk)

---

## Architecture Overview

### 1. Tree Data Provider

**File:** `src/extension/log/vscode-node/requestLogTree.ts`

**Classes:**
- `RequestLogTree` (IExtensionContribution)
  - Main entry point
  - Registers tree data provider with VS Code
  - Registers all export/save commands

- `ChatRequestProvider` implements `vscode.TreeDataProvider<TreeItem>`
  - Returns tree structure grouped by chat prompts
  - Filters items (elements, tools, NES requests) via `LogTreeFilters`

- Tree Item Classes
  - `ChatPromptItem`: Groups requests from one user prompt
  - `ChatRequestItem`: Single API request
  - `ToolCallItem`: Tool invocation
  - `ChatElementItem`: Rendered HTML element

### 2. Request Logger Service

**File:** `src/extension/prompt/vscode-node/requestLoggerImpl.ts`

**Class:** `RequestLogger` extends `AbstractRequestLogger`

**Registration:** `src/extension/extension/vscode-node/services.ts`
```typescript
builder.define(IRequestLogger, new SyncDescriptor(RequestLogger));
```

**Key Responsibilities:**
1. Logs all chat requests, tool calls, and rendered elements
2. Registers text document content provider for `ccreq:` scheme
3. Renders markdown content on-demand when URI is opened
4. Stores entries in `_entries[]` array (max 100)

### 3. URI Scheme & Virtual Document Provider

**File:** `src/platform/requestLogger/node/requestLogger.ts`

**Class:** `ChatRequestScheme`

**URI Format:**
```
ccreq:latest.copilotmd          → Latest request (markdown)
ccreq:{id}.copilotmd            → Specific request (markdown)
ccreq:{id}.json                 → Specific request (JSON)
ccreq:{id}.request.json         → Raw API request body
```

**Text Document Provider Registration** (in `requestLoggerImpl.ts`):
```typescript
workspace.registerTextDocumentContentProvider(ChatRequestScheme.chatRequestScheme, {
  onDidChange: Event.map(this.onDidChangeRequests, ...),
  provideTextDocumentContent: (uri) => {
    const { data, format } = ChatRequestScheme.parseUri(uri.toString());
    const entry = this._entries.find(e => e.id === data.id);

    if (format === 'markdown') {
      return this._renderRequestToMarkdown(entry.id, entry.entry);
    } else if (format === 'json') {
      return this._renderToJson(entry);
    } else {
      return this._renderRawRequestToJson(entry);
    }
  }
});
```

### 4. Markdown Rendering

**File:** `src/extension/prompt/vscode-node/requestLoggerImpl.ts`

**Main Function:** `_renderRequestToMarkdown(id: string, entry: LoggedRequest): string`

**Output Structure:**
```markdown
# DebugName - {id}

## Metadata
- url: API endpoint
- model: Claude/GPT-4 etc
- maxPromptTokens: Model limit
- maxResponseTokens: Response limit
- location: Chat/InlineChat/etc
- postOptions: Request settings
- intent: User intent
- startTime, endTime, duration: Timing
- requestId, serverRequestId: Identifiers
- usage: Token counts (if success)
- tools: Available tools (if any)

## Request Messages
### System
[System prompt content]

### User
[User input with file context, diagnostics, etc]

## Prediction
[Optional prediction/draft response]

## Response
### Assistant
[Raw API response or streaming deltas]
[Tool calls with arguments]
[Error messages if failed]
```

**Supporting Functions:**
- `_renderStringMessageToMarkdown()`: Converts strings to markdown
- `_renderDeltasToMarkdown()`: Converts streaming deltas to markdown
- `_renderToolCallToMarkdown()`: Renders tool invocations
- `_renderModelListToMarkdown()`: Renders model list endpoints
- `_renderRawRequestToJson()`: Returns raw API request body

---

## Complete Data Flow

```
┌─────────────────────────────────────────────────────────────┐
│ 1. User sends chat message                                  │
└──────────────────────┬──────────────────────────────────────┘
                       ↓
┌─────────────────────────────────────────────────────────────┐
│ 2. Conversation creates ChatRequest                         │
│    src/extension/conversation/*                             │
└──────────────────────┬──────────────────────────────────────┘
                       ↓
┌─────────────────────────────────────────────────────────────┐
│ 3. IRequestLogger.addEntry() called                         │
│    (requestLoggerImpl.ts)                                    │
└──────────────────────┬──────────────────────────────────────┘
                       ↓
┌─────────────────────────────────────────────────────────────┐
│ 4. Creates LoggedRequestInfo with unique ID                 │
│    Stores in RequestLogger._entries[]                       │
│    Fires onDidChangeRequests event                          │
└──────────────────────┬──────────────────────────────────────┘
                       ↓
┌─────────────────────────────────────────────────────────────┐
│ 5. ChatRequestProvider notified of change                   │
│    Rebuilds tree from _entries                              │
│    Groups by ChatPromptItem                                 │
└──────────────────────┬──────────────────────────────────────┘
                       ↓
┌─────────────────────────────────────────────────────────────┐
│ 6. Tree item displays in "Copilot Chat" panel               │
│    Ready for user interaction                               │
└──────────────────────┬──────────────────────────────────────┘
                       ↓
        ┌──────────────┴──────────────┐
        ↓                             ↓
┌──────────────────────┐    ┌──────────────────────┐
│ User clicks tree     │    │ User right-clicks    │
│ item to view         │    │ to export/save       │
└──────────┬───────────┘    └──────────┬───────────┘
           ↓                           ↓
    Opens URI:               Opens save dialog:
  ccreq:{id}.copilotmd   exportLogItemCommand
           ↓                 saveCurrentMarkdownCommand
┌──────────────────────────────────────────────────────────┐
│ 7. VS Code requests document content for URI              │
│    Text Document Provider invoked                        │
│    provideTextDocumentContent(uri)                       │
└──────────────────────┬──────────────────────────────────┘
                       ↓
┌──────────────────────────────────────────────────────────┐
│ 8. ChatRequestScheme.parseUri() extracts ID/format       │
│    Finds entry in _entries[]                             │
└──────────────────────┬──────────────────────────────────┘
                       ↓
┌──────────────────────────────────────────────────────────┐
│ 9. _renderRequestToMarkdown() generates markdown         │
│    - Includes metadata, request, response                │
│    - Formats with code blocks, headers                   │
│    - Processes deltas/tool calls                         │
└──────────────────────┬──────────────────────────────────┘
                       ↓
┌──────────────────────────────────────────────────────────┐
│ 10. Virtual document displayed in editor                 │
│     (.copilotmd file with raw API data)                  │
│     OR                                                    │
│     Saved to disk at user-selected location              │
└──────────────────────────────────────────────────────────┘
```

---

## File Locations & Key Files

| Component | File Path | Key Export |
|-----------|-----------|-----------|
| **Tree UI** | `src/extension/log/vscode-node/requestLogTree.ts` | `RequestLogTree` (IExtensionContribution) |
| **Logger** | `src/extension/prompt/vscode-node/requestLoggerImpl.ts` | `RequestLogger` (IRequestLogger) |
| **Scheme** | `src/platform/requestLogger/node/requestLogger.ts` | `ChatRequestScheme` (utility class) |
| **Types** | `src/platform/requestLogger/node/requestLogger.ts` | `IRequestLogger`, `ILoggedRequestInfo`, `LoggedRequest` |
| **Registration** | `src/extension/extension/vscode-node/services.ts` | `registerServices()` |
| **Contribution** | `src/extension/extension/vscode-node/contributions.ts` | `asContributionFactory(RequestLogTree)` |

---

## Commands & UI Actions

**Registered in `requestLogTree.ts` constructor:**

| Command ID | Action | Handler |
|-----------|--------|---------|
| `vscode.copilot.chat.showRequestHtmlItem` | Show HTML element in browser | `RequestServer` HTTP server |
| `github.copilot.chat.debug.exportLogItem` | Export selected item as .copilotmd | `exportLogItemCommand` |
| `github.copilot.chat.debug.exportPromptArchive` | Export all items as .tar.gz | `exportPromptArchiveCommand` |
| `github.copilot.chat.debug.exportPromptLogsAsJson` | Export logs as .json | `exportPromptLogsAsJsonCommand` |
| `github.copilot.chat.debug.exportAllPromptLogsAsJson` | Export all prompts as .json | `exportAllPromptLogsAsJsonCommand` |
| `github.copilot.chat.debug.saveCurrentMarkdown` | Save current .copilotmd to disk | `saveCurrentMarkdownCommand` |
| `github.copilot.chat.debug.showRawRequestBody` | Show raw API request as JSON | `showRawRequestBodyCommand` |

---

## Where the Markdown/MD Files Are

### In Memory (Virtual Documents)

**Location:** `RequestLogger._entries[]` array in memory

**Access:** Via `ccreq:` scheme URIs
```
ccreq:abc123.copilotmd       ← Virtual document with markdown
ccreq:abc123.json            ← JSON representation
ccreq:abc123.request.json    ← Raw API request body
```

**How Displayed:**
1. User clicks tree item
2. Tree item command opens: `vscode.Uri.parse(ChatRequestScheme.buildUri({ kind: 'request', id }))`
3. VS Code requests content via text document provider
4. Markdown is generated on-demand and displayed in editor tab
5. File shows as `.copilotmd` virtual document (not on disk)

### On Disk (When Saved)

**Save Locations:**
- User selects via save dialog
- Default: `~/request_id.md` or `~/request_id.copilotmd`
- Can be exported as:
  - Single `.md` file
  - Single `.copilotmd` file
  - `.tar.gz` archive with multiple files
  - `.chatreplay.json` with structured data

**Save Commands:**
- `saveCurrentMarkdownCommand`: Save currently open virtual document
- `exportLogItemCommand`: Export specific tree item
- `exportPromptArchiveCommand`: Export all items in a prompt as archive

---

## Rendering Pipeline

### Step-by-step breakdown:

1. **Request Logged** (in conversation flow)
   ```typescript
   requestLogger.addEntry({
     type: LoggedRequestKind.ChatMLSuccess,
     debugName: 'chat request',
     chatEndpoint: { model, ... },
     chatParams: { messages, ... },
     result: { value: response, ... },
     startTime, endTime, ...
   });
   ```

2. **Entry Created & Stored**
   ```typescript
   const id = generateUuid().substring(0, 8);  // e.g., "abc123de"
   const entry = new LoggedRequestInfo(id, entry, chatRequest);
   this._entries.push(entry);
   this._onDidChangeRequests.fire();
   ```

3. **Tree Updated**
   - `ChatRequestProvider.getChildren()` called
   - Entries converted to tree items
   - Grouped under `ChatPromptItem`

4. **User Opens Request**
   - Clicks on tree item
   - Executes: `vscode.open` with URI `ccreq:abc123de.copilotmd`

5. **Document Provider Called**
   - URI intercepted by text document provider
   - `ChatRequestScheme.parseUri()` extracts ID and format
   - Finds entry in `_entries` array
   - Calls `_renderRequestToMarkdown(id, entry)`

6. **Markdown Generated**
   - Formats metadata (model, timing, tokens, etc)
   - Includes request messages (system, user)
   - Formats response (markdown or deltas)
   - Includes tool calls and errors
   - Applies CSS styles for better display

7. **Displayed in Editor**
   - Virtual document tab appears
   - Shows as `.copilotmd` file
   - User can read, copy, save to disk

---

## Key Insights

### Memory vs Disk
- **Entries stored:** In memory only (`RequestLogger._entries`)
- **Persistence:** Session-based (cleared when extension reloads)
- **Display method:** Virtual documents (not real files)
- **Export method:** Manual save via commands

### URI Scheme Advantages
- Custom `ccreq:` scheme allows custom content provider
- No real files created on disk until explicitly saved
- Content generated on-demand
- Can have multiple formats from same data

### Rendering Performance
- Markdown rendered on-demand (not pre-rendered)
- Stored entries limited to 100 items
- Deltas processed into single message
- Large responses may be truncated

### Filtering & Organization
- Tree filtered by type (elements, tools, NES requests)
- Grouped by chat prompt
- Can be collapsed/expanded
- Real-time updates as new requests arrive

---

## Entry Points for Modification

### To Customize Markdown Output
File: `src/extension/prompt/vscode-node/requestLoggerImpl.ts`
Function: `_renderRequestToMarkdown()` (line 519+)

### To Add New Tree Item Types
File: `src/extension/log/vscode-node/requestLogTree.ts`
Classes: `ChatRequestProvider.logToTreeItem()` and new item class

### To Change Filtering
File: `src/extension/log/vscode-node/requestLogTree.ts`
Class: `LogTreeFilters`

### To Support New Formats
File: `src/platform/requestLogger/node/requestLogger.ts`
Class: `ChatRequestScheme.buildUri()` and `.parseUri()`

File: `src/extension/prompt/vscode-node/requestLoggerImpl.ts`
Method: `provideTextDocumentContent()` in the provider

---

## Summary

The `requestLogTree` is a sophisticated debugging feature that:
1. **Collects** all chat requests, tool calls, and rendered elements
2. **Organizes** them in a tree view grouped by user prompts
3. **Renders** them as markdown via virtual documents (ccreq: scheme)
4. **Stores** them in memory (session-only)
5. **Allows** users to export/save them to disk in various formats

The key innovation is using **virtual documents** instead of real files, allowing rich interactive exploration of chat logs without cluttering the file system.
