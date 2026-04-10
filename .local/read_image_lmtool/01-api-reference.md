# 01 — API Reference

## Core Types

### `vscode.LanguageModelTool<T>`

The interface a tool must implement. Source: `vscode.d.ts` lines ~20925–20990.

```typescript
interface LanguageModelTool<T> {
    /**
     * Invoke the tool with the given input and return a result.
     *
     * The provided {@link LanguageModelToolInvocationOptions.input input}
     * has been validated against the declared `inputSchema`.
     */
    invoke(
        options: LanguageModelToolInvocationOptions<T>,
        token: CancellationToken
    ): ProviderResult<LanguageModelToolResult>;

    /**
     * Called once before a tool is invoked.
     * May show a confirmation message or return extra context.
     */
    prepareInvocation?(
        options: LanguageModelToolInvocationPrepareOptions<T>,
        token: CancellationToken
    ): ProviderResult<PreparedToolInvocation>;
}
```

### `LanguageModelToolResult`

Container for tool output content. Accepts an array of heterogeneous parts.

```typescript
class LanguageModelToolResult {
    content: Array<
        LanguageModelTextPart |
        LanguageModelPromptTsxPart |
        LanguageModelDataPart |
        unknown
    >;

    constructor(
        content: Array<
            LanguageModelTextPart |
            LanguageModelPromptTsxPart |
            LanguageModelDataPart |
            unknown
        >
    );
}
```

### `LanguageModelDataPart`

The critical type for returning binary data (images, JSON, text blobs).

```typescript
class LanguageModelDataPart {
    /**
     * The MIME type of the data.
     */
    mimeType: string;

    /**
     * The data of the part.
     */
    data: Uint8Array;

    /**
     * Factory: create an image data part.
     * @param data — raw image bytes
     * @param mimeType — e.g. 'image/png'
     */
    static image(data: Uint8Array, mimeType: string): LanguageModelDataPart;

    /**
     * Factory: create a JSON data part.
     */
    static json(value: unknown): LanguageModelDataPart;

    /**
     * Factory: create a text data part.
     */
    static text(value: string, mimeType?: string): LanguageModelDataPart;

    constructor(mimeType: string, data: Uint8Array);
}
```

### `LanguageModelToolInvocationOptions<T>`

```typescript
interface LanguageModelToolInvocationOptions<T> {
    /** The validated input conforming to the tool's inputSchema. */
    input: T;

    /** A token to signal cancellation of the tool invocation. */
    tokenizationOptions?: LanguageModelToolTokenizationOptions;
}
```

### `PreparedToolInvocation`

Returned by `prepareInvocation()` to provide a progress message and optional confirmation.

```typescript
interface PreparedToolInvocation {
    /** Message displayed while the tool runs (e.g., "Reading image..."). */
    invocationMessage?: string | MarkdownString;

    /** Past-tense message shown after completion. */
    pastTenseMessage?: string | MarkdownString;

    /** If set, user must confirm before invocation proceeds. */
    confirmationMessages?: {
        title: string;
        message: string | MarkdownString;
    };
}
```

### `vscode.lm.registerTool(name, tool)`

```typescript
namespace lm {
    /**
     * Register a LanguageModelTool. The tool must also be contributed
     * via the `contributes.languageModelTools` section of package.json.
     *
     * @param name — Must match the `name` field in package.json contribution.
     * @param tool — Implementation of the tool.
     * @returns Disposable to unregister the tool.
     */
    export function registerTool<T>(
        name: string,
        tool: LanguageModelTool<T>
    ): Disposable;
}
```

### `vscode.workspace.fs.readFile(uri)`

```typescript
namespace workspace {
    namespace fs {
        /**
         * Read the entire contents of a file.
         * @returns Uint8Array of the file contents.
         * @throws FileNotFound, FileIsADirectory, NoPermissions
         */
        export function readFile(uri: Uri): Thenable<Uint8Array>;
    }
}
```

## Supported Image MIME Types

Based on Copilot Chat's `ChatImageMimeType` enum and `isImageDataPart()` guard:

| Extension | MIME Type | Support Level |
|-----------|-----------|---------------|
| `.png` | `image/png` | Full — primary format |
| `.jpg`, `.jpeg` | `image/jpeg` | Full — primary format |
| `.gif` | `image/gif` | Supported |
| `.webp` | `image/webp` | Supported |
| `.bmp` | `image/bmp` | Supported |
| `.svg` | `image/svg+xml` | Not in `ChatImageMimeType` — not recognised as image by Copilot |
| `.ico` | `image/x-icon` | Not in `ChatImageMimeType` — not recognised |

The `isImageDataPart()` guard in Copilot Chat checks:
1. `part instanceof LanguageModelDataPart`
2. `isChatImageMimeType(part.mimeType)` — must be one of the five enumerated types

Any MIME type outside the five listed above will be treated as a non-image data part and likely rendered as text, not as a visual content block.

## Token Cost Model

Copilot Chat uses a tile-based token cost calculation:

```
cost = 85 + (170 × number_of_512x512_tiles)
```

For a 1024×1024 image: `85 + (170 × 4) = 765 tokens`.

This is calculated by `calculateImageTokenCost()` in the tokenizer module. The cost is deducted from the prompt budget, which means large images reduce space for other context.
