# 03 — Implementation

## Source Location

The complete implementation lives in the `lmtools-read-image` repository:

- **Tool**: `/Users/ming/GitRepos/github-ming86/lmtools-read-image/src/tools/readImageTool.ts`
- **Types**: `/Users/ming/GitRepos/github-ming86/lmtools-read-image/src/types/index.ts`
- **Entry point**: `/Users/ming/GitRepos/github-ming86/lmtools-read-image/src/extension.ts`
- **Manifest**: `/Users/ming/GitRepos/github-ming86/lmtools-read-image/package.json`

## Implementation Summary

```typescript
import * as vscode from 'vscode';
import * as path from 'path';

// ---------------------------------------------------------------------------
// Constants
// ---------------------------------------------------------------------------

const TOOL_NAME = 'readImage';

/**
 * Maximum file size in bytes. Copilot Chat enforces a 2.5 MB image budget
 * (when upload is not available) and the LanguageModelDataPart documentation
 * suggests a 5 MB ceiling. We use 5 MB here; the downstream pipeline will
 * handle further budget enforcement.
 */
const MAX_FILE_SIZE_BYTES = 5 * 1024 * 1024; // 5 MB

/**
 * Supported image MIME types. These match the `ChatImageMimeType` enum
 * used internally by Copilot Chat's `isImageDataPart()` guard.
 *
 * Only these five types are recognised as visual content by the
 * imageDataPartToTSX() pipeline. Anything else is treated as a
 * non-image data part and rendered as text.
 */
const MIME_MAP: ReadonlyMap<string, string> = new Map([
    ['.png', 'image/png'],
    ['.jpg', 'image/jpeg'],
    ['.jpeg', 'image/jpeg'],
    ['.gif', 'image/gif'],
    ['.webp', 'image/webp'],
    ['.bmp', 'image/bmp'],
]);

const SUPPORTED_EXTENSIONS = Array.from(MIME_MAP.keys()).join(', ');

// ---------------------------------------------------------------------------
// Input Schema Type
// ---------------------------------------------------------------------------

interface ReadImageInput {
    filePath: string;
}

// ---------------------------------------------------------------------------
// Tool Implementation
// ---------------------------------------------------------------------------

class ReadImageTool implements vscode.LanguageModelTool<ReadImageInput> {

    async invoke(
        options: vscode.LanguageModelToolInvocationOptions<ReadImageInput>,
        token: vscode.CancellationToken
    ): Promise<vscode.LanguageModelToolResult> {
        const { filePath } = options.input;

        // 1. Resolve the URI
        const uri = this.resolveUri(filePath);

        // 2. Validate the file extension
        const ext = path.extname(uri.fsPath).toLowerCase();
        const mime = MIME_MAP.get(ext);
        if (!mime) {
            return this.errorResult(
                `Unsupported image format: "${ext}". Supported formats: ${SUPPORTED_EXTENSIONS}`
            );
        }

        // 3. Check cancellation before I/O
        if (token.isCancellationRequested) {
            return this.errorResult('Operation cancelled.');
        }

        // 4. Stat the file to check existence and size before reading
        let stat: vscode.FileStat;
        try {
            stat = await vscode.workspace.fs.stat(uri);
        } catch (err) {
            return this.errorResult(
                `File not found or inaccessible: ${filePath}`
            );
        }

        if (stat.type !== vscode.FileType.File) {
            return this.errorResult(
                `Path is not a file: ${filePath}`
            );
        }

        if (stat.size > MAX_FILE_SIZE_BYTES) {
            const sizeMB = (stat.size / (1024 * 1024)).toFixed(1);
            return this.errorResult(
                `File too large: ${sizeMB} MB. Maximum supported size is ${MAX_FILE_SIZE_BYTES / (1024 * 1024)} MB.`
            );
        }

        // 5. Read the file
        let data: Uint8Array;
        try {
            data = await vscode.workspace.fs.readFile(uri);
        } catch (err) {
            return this.errorResult(
                `Failed to read file: ${filePath}. ${err instanceof Error ? err.message : String(err)}`
            );
        }

        // 6. Check cancellation after I/O
        if (token.isCancellationRequested) {
            return this.errorResult('Operation cancelled.');
        }

        // 7. Return the image as a data part
        return new vscode.LanguageModelToolResult([
            vscode.LanguageModelDataPart.image(data, mime),
        ]);
    }

    prepareInvocation(
        options: vscode.LanguageModelToolInvocationPrepareOptions<ReadImageInput>,
        _token: vscode.CancellationToken
    ): vscode.ProviderResult<vscode.PreparedToolInvocation> {
        const fileName = path.basename(options.input.filePath);
        return {
            invocationMessage: `Reading image: ${fileName}`,
            pastTenseMessage: `Read image: ${fileName}`,
            // Optional: require user confirmation before reading.
            // Uncomment the following to enable confirmation dialogs:
            //
            // confirmationMessages: {
            //     title: 'Read Image',
            //     message: `Allow Copilot to read the image file "${fileName}"?`,
            // },
        };
    }

    // -----------------------------------------------------------------------
    // Private helpers
    // -----------------------------------------------------------------------

    /**
     * Resolve a file path string to a VS Code URI.
     *
     * If the path is absolute, it is used directly. If relative, it is
     * resolved against the first workspace folder.
     */
    private resolveUri(filePath: string): vscode.Uri {
        if (path.isAbsolute(filePath)) {
            return vscode.Uri.file(filePath);
        }

        // Resolve relative paths against workspace root
        const workspaceFolders = vscode.workspace.workspaceFolders;
        if (workspaceFolders && workspaceFolders.length > 0) {
            return vscode.Uri.joinPath(workspaceFolders[0].uri, filePath);
        }

        // Fallback: treat as absolute
        return vscode.Uri.file(filePath);
    }

    /**
     * Produce a text-only error result.
     *
     * Returning an error as a LanguageModelToolResult (rather than throwing)
     * allows the model to read the error message and adapt its behavior.
     * Thrown exceptions surface as opaque failures in the chat UI.
     */
    private errorResult(message: string): vscode.LanguageModelToolResult {
        return new vscode.LanguageModelToolResult([
            new vscode.LanguageModelTextPart(message),
        ]);
    }
}

// ---------------------------------------------------------------------------
// Extension Lifecycle
// ---------------------------------------------------------------------------

export function activate(context: vscode.ExtensionContext): void {
    const tool = new ReadImageTool();
    context.subscriptions.push(
        vscode.lm.registerTool(TOOL_NAME, tool)
    );
}

export function deactivate(): void {
    // Disposal is handled by the subscription registered in activate().
}
```

## Implementation Notes

### 1. Error Handling Strategy

Errors are returned as `LanguageModelTextPart` within a `LanguageModelToolResult`, **not** thrown as exceptions. This is deliberate:

- **Thrown exceptions** surface as opaque "tool failed" messages in the chat UI. The model cannot read the error and cannot adapt.
- **Error text in results** lets the model read the failure reason and take corrective action (e.g., choose a different file, inform the user).

This matches the pattern used by Copilot Chat's internal tools.

### 2. Path Resolution

The model typically provides absolute paths (derived from workspace context, file listings, etc.). However, the tool also handles relative paths by resolving against the first workspace folder.

`vscode.Uri.file()` is preferred over string concatenation for cross-platform compatibility. On Windows, `Uri.file()` correctly handles drive letters and backslashes.

### 3. File Size Guard

Two layers of size protection exist:

1. **This tool:** Rejects files > 5 MB before reading. This prevents loading absurdly large files into memory.
2. **Copilot Chat pipeline:** `PrimitiveToolResult.onImage()` enforces a 2.5 MB budget across all images in a single tool result. Images exceeding the budget are silently dropped with a log message.

The tool's 5 MB limit is generous because the downstream pipeline provides additional enforcement.

### 4. MIME Detection

MIME type is determined by file extension, not by magic bytes. This is:

- **Simpler** — no dependency on a binary inspection library.
- **Sufficient** — image files in workspaces virtually always have correct extensions.
- **Consistent** — matches how web browsers and VS Code's built-in editor determine image types.

If magic-byte detection is desired for robustness, the first few bytes could be inspected after reading. Common signatures:

| Format | Magic Bytes |
|--------|-------------|
| PNG | `89 50 4E 47` |
| JPEG | `FF D8 FF` |
| GIF | `47 49 46 38` |
| WebP | `52 49 46 46 ... 57 45 42 50` |
| BMP | `42 4D` |

### 5. Cancellation Token

The `CancellationToken` is checked both before and after the `readFile` call. `workspace.fs.readFile()` does not accept a cancellation token, so cancellation during the read itself is not possible — but the check after reading prevents returning stale results if the request was cancelled while I/O was in flight.

### 6. No Confirmation Dialog by Default

The `confirmationMessages` in `prepareInvocation` is commented out. Image reading is a non-destructive, read-only operation. Adding a confirmation dialog would interrupt the agentic loop for every image, which defeats the purpose of autonomous operation.

If the extension is used in security-sensitive contexts (e.g., reading images from untrusted repositories), the confirmation can be enabled by uncommenting those lines.

## Variant: Multi-Image Support

If the model should be able to read multiple images in a single invocation (e.g., comparing two screenshots), the input schema and implementation can be extended:

```typescript
interface ReadImagesInput {
    filePaths: string[];
}
```

```jsonc
// package.json inputSchema
{
    "type": "object",
    "properties": {
        "filePaths": {
            "type": "array",
            "items": { "type": "string" },
            "description": "Array of absolute paths to image files.",
            "maxItems": 5
        }
    },
    "required": ["filePaths"]
}
```

```typescript
async invoke(options, token) {
    const parts: (vscode.LanguageModelDataPart | vscode.LanguageModelTextPart)[] = [];
    for (const filePath of options.input.filePaths) {
        // ... validate and read each file
        parts.push(vscode.LanguageModelDataPart.image(data, mime));
    }
    return new vscode.LanguageModelToolResult(parts);
}
```

This is a natural extension but adds complexity. The single-file variant is recommended as the starting point; the model can invoke the tool multiple times for multiple images.

## Variant: Workspace-Relative Path Resolution with Multi-Root

For multi-root workspaces, a more sophisticated path resolution strategy:

```typescript
private resolveUri(filePath: string): vscode.Uri {
    if (path.isAbsolute(filePath)) {
        return vscode.Uri.file(filePath);
    }

    const folders = vscode.workspace.workspaceFolders ?? [];
    for (const folder of folders) {
        const candidate = vscode.Uri.joinPath(folder.uri, filePath);
        // We cannot stat synchronously, so prefer the first folder.
        // The model should provide absolute paths for multi-root workspaces.
        return candidate;
    }

    return vscode.Uri.file(filePath);
}
```

In practice, the model almost always provides absolute paths because Copilot Chat's workspace context gives it absolute file listings.
