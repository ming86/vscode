# Request Data Storage: Memory vs Filesystem Analysis

## TL;DR - Quick Answer

**No, request data is NOT automatically stored in the filesystem.** It only exists in memory during the extension session. However, there ARE ways to access it externally:

1. ✅ **Export via UI** - Use built-in commands to save as `.md` / `.json`
2. ✅ **Output Channel Logs** - VS Code saves log URIs to disk automatically
3. ✅ **Custom Extension** - Can access `IRequestLogger` service
4. ✅ **API Interception** - Monitor workspace file changes or commands

---

## Storage Analysis

### Current Storage Model

| Aspect | Details |
|--------|---------|
| **Primary Storage** | `RequestLogger._entries[]` array (memory only) |
| **Capacity** | Max 100 entries (circular buffer) |
| **Lifetime** | Current session only (cleared on reload) |
| **Persistence** | ❌ None by default |
| **Auto-Save** | ❌ No |

### Why No Filesystem Storage?

**Privacy Reasons:**
- Chat logs may contain sensitive code, file paths, terminal output
- Users should explicitly choose to export/save
- Automatic disk writes could expose private data

**Code Comment from requestLoggerImpl.ts:**
```typescript
// No automatic persistence - entries are session-only
this._entries.push(entry);
// keep at most 100 entries
if (this._entries.length > 100) {
  this._entries.shift();
}
```

---

## Ways to Access Request Data

### Method 1: UI Export Commands (Easiest)

**User does this manually:**
- Right-click tree item → "Export Log Item" → Saves to disk
- Right-click tree item → "Export Prompt Archive" → Creates `.tar.gz`
- Right-click tree item → "Export Prompt Logs as JSON" → Saves as `.json`

**Formats available:**
- `.copilotmd` - Markdown with metadata + response
- `.json` - Structured JSON format
- `.request.json` - Raw API request body
- `.tar.gz` - Multiple items archived
- `.chatreplay.json` - Full prompt history

---

### Method 2: Output Channel Logs (Automatic)

**Important Discovery:** The extension logs request URIs to the VS Code output channel, and VS Code automatically persists this!

**Location:**
```
VS Code Output Channel: "GitHub Copilot Chat"
```

**What Gets Logged:**
```
ccreq:abc123de.copilotmd | success | gpt-4o | 145ms | [chat request]
ccreq:latest.copilotmd
```

**VS Code Persistence:**
- Output channels with `{ log: true }` are automatically saved
- Usually stored in: `~/.vscode/output/` directory
- Persists between sessions!

**Code in outputChannelLogTarget.ts:**
```typescript
export class NewOutputChannelLogTarget implements ILogTarget {
  private readonly _outputChannel = window.createOutputChannel(
    OutputChannelName,
    { log: true }  // ← This enables automatic file logging!
  );
}
```

---

### Method 3: Custom Extension (Programmatic Access)

You can create a custom VS Code extension that:

#### Option A: Access the IRequestLogger Service

```typescript
// In your custom extension
import { IRequestLogger } from '@vscode/copilot-chat';

// Get the service
const requestLogger: IRequestLogger = accessor.get(IRequestLogger);

// Get all logged entries
const entries: LoggedInfo[] = requestLogger.getRequests();

// Process entries
for (const entry of entries) {
  if (entry.kind === LoggedInfoKind.Request) {
    const info = entry as ILoggedRequestInfo;
    console.log(info.entry.debugName);
    console.log(info.entry.result);
    // ... do something with the data
  }
}
```

**Limitations:**
- Only works if Copilot Chat extension is active
- Must declare dependency in `package.json`:
  ```json
  "extensionDependencies": ["GitHub.copilot-chat"]
  ```
- Access is only available when extension is loaded

#### Option B: Monitor Virtual Documents

```typescript
// Listen for text document opens
vscode.workspace.onDidOpenTextDocument(doc => {
  if (doc.uri.scheme === 'ccreq') {
    // This is a request log document!
    const content = doc.getText();
    console.log(content); // The rendered markdown

    // Save it somewhere
    const fileName = doc.uri.toString().replace(/:/g, '_');
    await vscode.workspace.fs.writeFile(
      vscode.Uri.file(`/path/to/logs/${fileName}.md`),
      Buffer.from(content)
    );
  }
});
```

**Advantages:**
- Automatic capture when user opens any request
- Works externally (no direct access needed)
- Can save all opened requests

#### Option C: Monitor Commands

```typescript
// Listen for export commands
vscode.commands.onDidExecuteCommand(e => {
  if (e.command === 'github.copilot.chat.debug.exportLogItem') {
    console.log('User exported log item');
    // Could trigger your own processing
  }
});
```

---

### Method 4: Read Output Channel Logs

VS Code saves output channel logs to disk. You can read them:

```typescript
// Find the output log file
const outputDir = path.join(
  os.homedir(),
  '.vscode',
  'output'
);

// List files and find GitHub Copilot Chat logs
const files = await fs.promises.readdir(outputDir);
const chatLogFile = files.find(f =>
  f.includes('GitHub') &&
  f.includes('copilot-chat')
);

// Read the log file
const logPath = path.join(outputDir, chatLogFile);
const content = await fs.promises.readFile(logPath, 'utf-8');

// Parse request URIs
const uriPattern = /ccreq:[^\s]+\.(copilotmd|json|request\.json)/g;
const uris = content.match(uriPattern) || [];

console.log('Found request URIs:', uris);
```

---

### Method 5: Custom IRequestLogger Implementation

If you have access to Copilot Chat source, you could:

1. Fork it and add persistent logging
2. Modify `RequestLogger._addEntry()` to write to disk
3. Or inject logging hook

```typescript
// Pseudo-code: What you could modify
private async _addEntry(entry: LoggedInfo): Promise<boolean> {
  this._entries.push(entry);

  // Add this to make it persistent:
  await this._persistEntry(entry);  // ← NEW

  this._onDidChangeRequests.fire();
  return true;
}

private async _persistEntry(entry: LoggedInfo): Promise<void> {
  const logDir = path.join(os.homedir(), '.copilot-chat-logs');
  await fs.promises.mkdir(logDir, { recursive: true });

  const fileName = `${entry.id}.json`;
  await fs.promises.writeFile(
    path.join(logDir, fileName),
    JSON.stringify(entry, null, 2)
  );
}
```

---

## Comparison: Access Methods

| Method | Ease | Real-time | Persistent | Automatic | External |
|--------|------|-----------|-----------|-----------|----------|
| UI Export | ⭐⭐ | ❌ | ✅ | ❌ | ✅ |
| Output Logs | ⭐⭐⭐ | ✅ | ✅ | ✅ | ✅ |
| IRequestLogger | ⭐⭐⭐⭐ | ✅ | ❌ | ✅ | ❌ |
| Monitor Docs | ⭐⭐⭐ | ✅ | ✅ | ✅ | ✅ |
| Monitor Cmds | ⭐⭐⭐ | ✅ | ❌ | ✅ | ✅ |

**Best for External Extension:** Monitor Documents or Output Logs

---

## Recommended Approach: Custom Extension

Here's a practical example of a custom extension that captures request logs externally:

```typescript
// extension.ts
import * as vscode from 'vscode';
import * as fs from 'fs';
import * as path from 'path';
import * as os from 'os';

const LOG_DIR = path.join(os.homedir(), '.vscode', 'copilot-chat-exports');

export function activate(context: vscode.ExtensionContext) {
  // Ensure log directory exists
  if (!fs.existsSync(LOG_DIR)) {
    fs.mkdirSync(LOG_DIR, { recursive: true });
  }

  // Monitor document opens to capture ccreq: URIs
  vscode.workspace.onDidOpenTextDocument(doc => {
    if (doc.uri.scheme === 'ccreq') {
      captureLogDocument(doc);
    }
  });

  // Also listen for any open/close events
  vscode.workspace.onDidChangeTextDocument(e => {
    if (e.document.uri.scheme === 'ccreq') {
      // Auto-save whenever the document changes
      saveLogDocument(e.document);
    }
  });

  // Register command to manually export all
  context.subscriptions.push(
    vscode.commands.registerCommand('myext.exportAllLogs', async () => {
      vscode.window.showInformationMessage(
        `Logs exported to: ${LOG_DIR}`
      );
    })
  );
}

async function captureLogDocument(doc: vscode.TextDocument) {
  const fileName = doc.uri.toString()
    .replace(/:/g, '_')
    .replace(/\//g, '_');

  const filePath = path.join(LOG_DIR, `${fileName}.md`);
  const content = doc.getText();

  fs.writeFileSync(filePath, content, 'utf-8');
  console.log(`Captured log: ${filePath}`);
}

async function saveLogDocument(doc: vscode.TextDocument) {
  // Similar to above
  captureLogDocument(doc);
}

export function deactivate() {}
```

**package.json for your extension:**
```json
{
  "name": "copilot-chat-log-exporter",
  "displayName": "Copilot Chat Log Exporter",
  "version": "1.0.0",
  "engines": {
    "vscode": "^1.85.0"
  },
  "extensionDependencies": [
    "GitHub.copilot-chat"
  ],
  "activationEvents": [
    "onStartupFinished"
  ],
  "main": "./out/extension.js",
  "contributes": {
    "commands": [
      {
        "command": "myext.exportAllLogs",
        "title": "Export All Copilot Logs"
      }
    ]
  }
}
```

---

## Accessing Output Channel Logs Programmatically

VS Code's `{ log: true }` option in output channels saves to:

```
~/.vscode/output/{Channel ID}/log.txt
```

You can read it:

```typescript
import * as fs from 'fs';
import * as path from 'path';
import * as os from 'os';

async function readCopilotLogs() {
  const outputDir = path.join(
    os.homedir(),
    '.vscode',
    'output'
  );

  // List all output files
  const dirs = fs.readdirSync(outputDir);

  for (const dir of dirs) {
    if (dir.includes('GitHub') || dir.includes('copilot')) {
      const logFile = path.join(outputDir, dir, 'log.txt');

      if (fs.existsSync(logFile)) {
        const content = fs.readFileSync(logFile, 'utf-8');

        // Extract ccreq: URIs
        const uriRegex = /ccreq:[^\s]+\.(copilotmd|json|request\.json)/g;
        const uris = content.match(uriRegex) || [];

        return {
          logFile,
          uris,
          content
        };
      }
    }
  }
}

// Usage
readCopilotLogs().then(result => {
  console.log('Found URIs:', result.uris);
  console.log('Log file:', result.logFile);
});
```

---

## Summary

**Question: Is request data stored in filesystem?**
- **Direct answer:** No, only in memory
- **Session persistence:** None by default
- **BUT:** URIs are logged to VS Code output channel which IS persisted

**Ways to load externally:**
1. ✅ **Use UI exports** - Simplest for one-time exports
2. ✅ **Read output logs** - Automatic, persistent, but just URIs
3. ✅ **Custom extension** - Most flexible, can auto-capture all logs
4. ✅ **Monitor virtual documents** - Real-time capture as user opens them
5. ✅ **Access IRequestLogger** - If you can depend on Copilot Chat extension

**Recommendation:** Create a custom extension that monitors `ccreq:` document opens and auto-saves them to disk. This gives you persistent, automatic logging without modifying Copilot Chat itself.

