# requestLogTree - Quick Reference Guide

## TL;DR - What is it?

`requestLogTree` is a VS Code extension feature that:
- **Collects** all AI chat requests/responses in memory
- **Displays** them in a tree view (Copilot Chat panel)
- **Renders** them as virtual markdown documents (`ccreq:` scheme)
- **Allows** exporting to disk in multiple formats

---

## Key Files You Need to Know

### 1. Tree Display
📍 `src/extension/log/vscode-node/requestLogTree.ts`
- Shows the UI in "Copilot Chat" tree panel
- Handles all export/save commands
- Classes: `RequestLogTree`, `ChatRequestProvider`, tree item classes

### 2. Request Logger (Core Service)
📍 `src/extension/prompt/vscode-node/requestLoggerImpl.ts`
- Logs all API requests/responses
- Registered as `IRequestLogger` service
- Text document provider for virtual documents
- Main rendering function: `_renderRequestToMarkdown()`

### 3. URI Scheme
📍 `src/platform/requestLogger/node/requestLogger.ts`
- Defines `ccreq:` URI scheme
- `ChatRequestScheme` class with URI builder/parser
- Supported formats: `.copilotmd`, `.json`, `.request.json`

### 4. Service Registration
📍 `src/extension/extension/vscode-node/services.ts`
- Registers `RequestLogger` as `IRequestLogger`

---

## Virtual Document URI Format

### How it Works
```
ccreq:{id}.{format}
│     │    │
│     │    └─ copilotmd | json | request.json
│     └────── 8-char UUID
└─────────── Custom scheme
```

### Examples
```
ccreq:latest.copilotmd          ← Latest request in markdown
ccreq:abc123de.copilotmd        ← Specific request in markdown
ccreq:abc123de.json             ← Specific request as JSON
ccreq:abc123de.request.json     ← Raw API request body
```

### Key Point
**These are NOT real files on disk!** They are:
- Virtual documents (only exist in VS Code editor)
- Content generated on-demand
- Rendered from in-memory data in `RequestLogger._entries[]`

---

## What Gets Rendered in Markdown

When you open a `.copilotmd` file, you see:

```markdown
# Request Name - id

## Metadata
- API endpoint
- Model name (Claude, GPT-4, etc)
- Token limits and usage
- Request timing
- Request/Response IDs
- Tool configurations

## Request Messages
### System
[System prompt]

### User
[User input with context]

## Prediction (optional)
[Draft response if prediction was used]

## Response
### Assistant
[The actual AI response]

[Tool calls if any]
[Error info if failed]
```

---

## Data Flow One-Liner

**Chat API Request** → **Logged in RequestLogger** → **Grouped in Tree** → **User clicks** → **Virtual Doc Provider** → **Renders Markdown** → **Displayed in Editor**

---

## Key Commands

| Context | Command | What it Does |
|---------|---------|------------|
| Tree item | `exportLogItemCommand` | Save single item as `.md`/`.copilotmd` |
| Open .copilotmd | `saveCurrentMarkdownCommand` | Save current virtual doc to disk |
| Tree item | `exportPromptArchiveCommand` | Export all items as `.tar.gz` |
| Tree item | `exportPromptLogsAsJsonCommand` | Export as structured `.json` |
| Tree | `exportAllPromptLogsAsJsonCommand` | Export all prompts as `.json` |

---

## Memory & Storage

### In-Memory Storage
- **Location:** `RequestLogger._entries[]` array
- **Max entries:** 100 (circular buffer)
- **Lifetime:** Current extension session only
- **Format:** Stored as `LoggedRequestInfo` / `LoggedToolCall` / `LoggedElementInfo` objects

### On-Disk Storage
- **Default location:** User's home directory (`~/`)
- **Formats:** `.md`, `.copilotmd`, `.json`, `.request.json`, `.tar.gz`
- **Action:** Manual save via commands (NOT automatic)

**Key point:** Nothing is saved to disk unless the user explicitly exports/saves it.

---

## The Three Types of Logged Items

### 1. Chat Request (`LoggedRequestInfo`)
- The API call to the model
- Includes request messages and response
- Shows metadata (tokens, timing, model, etc)
- Most common item in the tree

### 2. Tool Call (`LoggedToolCall`)
- When the model uses a tool
- Shows tool name and arguments
- Shows tool response

### 3. Element (`LoggedElementInfo`)
- Rendered HTML from prompt-tsx
- Token usage information
- Can be displayed in browser (not markdown)

---

## How Filtering Works

Tree items can be hidden/shown via filter buttons:

| Filter | Type | What it hides |
|--------|------|---------------|
| Elements | HTML Elements | `<PromptElement/>` rendered items |
| Tools | Tool Calls | Invocations of tools/functions |
| NES Requests | Specific requests | Requests from NES provider |

**Default:** All shown. Filtered via `LogTreeFilters` class.

---

## Where to Make Changes

### Modify Markdown Format
**File:** `src/extension/prompt/vscode-node/requestLoggerImpl.ts`
**Function:** `_renderRequestToMarkdown()` (line ~519)

**What to change:**
- Add new sections to markdown output
- Change formatting of metadata
- Customize response rendering
- Add new information fields

### Add New Commands
**File:** `src/extension/log/vscode-node/requestLogTree.ts`
**Location:** In `RequestLogTree` constructor

**What to add:**
- New `vscode.commands.registerCommand()`
- Export format handlers
- Filtering options

### Change URI Scheme
**File:** `src/platform/requestLogger/node/requestLogger.ts`
**Class:** `ChatRequestScheme`

**What to modify:**
- Add new file extensions
- Change URI format
- Add new parse patterns

### Customize Tree Display
**File:** `src/extension/log/vscode-node/requestLogTree.ts`
**Class:** `ChatRequestProvider` or tree item classes

**What to change:**
- Tree structure/hierarchy
- Item labels and descriptions
- Icon assignments
- Context menus

---

## Debugging Tips

### Find where requests are logged
```bash
grep -r "requestLogger.addEntry\|logModelListCall\|logToolCall" src/
```

### Trace URI parsing
Look at `ChatRequestScheme.parseUri()` in `requestLogger.ts`

### See what markdown gets generated
The function `_renderRequestToMarkdown()` is the single source of truth for markdown output

### Check tree structure
`ChatRequestProvider.getChildren()` builds the tree from `_entries[]`

### Monitor in-memory entries
Set breakpoint in `RequestLogger._addEntry()` to see all logged items

---

## Important Implementation Details

1. **Lazy Rendering**
   - Markdown is NOT pre-rendered and stored
   - It's generated on-demand when the virtual doc is opened
   - This saves memory for large responses

2. **Circular Buffer**
   - `_entries[]` limited to 100 items
   - Oldest entry is dropped when limit exceeded
   - FIFO (first in, first out)

3. **Event-Driven Updates**
   - Tree updates via `onDidChangeRequests` event
   - UI automatically refreshes when new requests arrive
   - Filter changes trigger tree rebuild

4. **No Real Files**
   - The `.copilotmd` files shown in editor are NOT on disk
   - They're generated by text document provider
   - Only saved when user explicitly exports them

5. **Scheme Registration**
   - `ccreq:` scheme is custom to this extension
   - Intercepted by registered text document provider
   - Allows content generation from memory

---

## Export Formats Explained

| Format | Extension | Contains | Best For |
|--------|-----------|----------|----------|
| Markdown | `.md` / `.copilotmd` | Formatted markdown | Reading/sharing |
| JSON | `.json` | Structured data | Parsing/analysis |
| Raw Request | `.request.json` | Raw API request | Debugging API calls |
| Archive | `.tar.gz` | Multiple items | Batch export |
| Replay | `.chatreplay.json` | Full prompt history | Replaying chats |

---

## Common Questions

**Q: Where are the .copilotmd files stored?**
A: Nowhere by default! They're virtual documents. Only saved to disk when you export them.

**Q: Can I access the chat logs programmatically?**
A: Yes, via `IRequestLogger.getRequests()` which returns `LoggedInfo[]` array.

**Q: How long are logs kept?**
A: While the extension is running (session only). Max 100 entries.

**Q: Can I change what gets logged?**
A: Yes, modify what calls `requestLogger.addEntry()` in the codebase.

**Q: Why not just save to disk automatically?**
A: Privacy - chat may contain sensitive code/data. Manual export lets users choose.

**Q: Can I customize the markdown output?**
A: Yes! Edit `_renderRequestToMarkdown()` in `requestLoggerImpl.ts`.

