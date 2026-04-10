# 06 — Internal Patterns Reference

> This document catalogues how Copilot Chat handles image data from tools internally.
> These patterns are **reference material**, not code to copy. They inform design
> decisions for the 3rd-party extension and explain why certain constraints exist.

## 1. Image Data Part Consumption Pipeline

When a tool returns `LanguageModelDataPart.image()`, Copilot Chat processes it through this pipeline:

```
Tool.invoke()
  → LanguageModelToolResult([DataPart.image(data, mime)])
    → PrimitiveToolResult.onImage(part)
      → Budget check (2.5 MB cumulative)
      → imageDataPartToTSX(part, githubToken, ...)
        → Base64 encode
        → Optional: upload to GitHub (returns URL)
        → <Image> TSX element
          → Wire format: content_part { type: "image_url", image_url: { url: "data:..." } }
```

### `imageDataPartToTSX()` — Core Conversion

Location: `src/extension/prompts/node/panel/toolCalling.tsx`

```typescript
// Simplified from source
async function imageDataPartToTSX(
    part: vscode.LanguageModelDataPart,
    githubToken: string | undefined,
    urlOrRequestMetadata: ...,
    logService: ILogService,
    imageService: IImageService,
): Promise<ImageElement | undefined> {
    const base64 = Buffer.from(part.data).toString('base64');
    const dataUrl = `data:${part.mimeType};base64,${base64}`;

    // Attempt upload for URL-capable models
    if (imageService && githubToken) {
        const uploaded = await imageService.uploadChatImageAttachment(
            part.data, part.mimeType, githubToken
        );
        if (uploaded) {
            return <Image url={uploaded.url} detail="auto" />;
        }
    }

    return <Image url={dataUrl} detail="auto" />;
}
```

### `PrimitiveToolResult.onImage()` — Budget Enforcement

Location: `src/extension/prompts/node/panel/toolCalling.tsx`

```typescript
// Simplified from source
class PrimitiveToolResult {
    private imageBudget = 2.5 * 1024 * 1024;

    async onImage(part: vscode.LanguageModelDataPart) {
        if (this.imageBudget <= 0) {
            this.logService.info('Image budget exhausted');
            return;
        }

        this.imageBudget -= part.data.byteLength;

        const tsx = await imageDataPartToTSX(part, ...);
        if (tsx) {
            this.images.push(tsx);
        }
    }
}
```

## 2. Existing Image-Returning Tools

### `getNotebookCellOutputTool.tsx`

The closest internal precedent for returning images from a tool.

**Key patterns:**

```typescript
// Vision check before returning images
const supportsVision = endpoint?.supportsVision;

// Extract image MIME types from notebook cell outputs
for (const item of output.items) {
    if (item.mime === 'image/png' || item.mime === 'image/jpeg') {
        if (supportsVision) {
            parts.push(vscode.LanguageModelDataPart.image(item.data, item.mime));
        }
    }
}

// Text fallback when vision is unavailable
if (!supportsVision && hasImageOutputs) {
    parts.push(new vscode.LanguageModelTextPart(
        'Cell produced image output but the current model does not support vision.'
    ));
}
```

**Observation:** The internal tool checks `supportsVision` before returning image parts. Our 3rd-party extension cannot access the `endpoint` object, so it always returns the image. The downstream pipeline handles vision gating.

### `runNotebookCellTool.tsx`

Same image extraction pattern as `getNotebookCellOutputTool`, applied after executing a notebook cell.

### `fetchWebPageTool.tsx`

Passes through images embedded in fetched web pages. Uses the same `LanguageModelDataPart.image()` factory.

## 3. Image Type Guard

Location: `src/extension/conversation/common/languageModelChatMessageHelpers.ts`

```typescript
export const enum ChatImageMimeType {
    PNG = 'image/png',
    JPEG = 'image/jpeg',
    GIF = 'image/gif',
    WEBP = 'image/webp',
    BMP = 'image/bmp',
}

function isChatImageMimeType(mime: string): mime is ChatImageMimeType {
    return mime === ChatImageMimeType.PNG
        || mime === ChatImageMimeType.JPEG
        || mime === ChatImageMimeType.GIF
        || mime === ChatImageMimeType.WEBP
        || mime === ChatImageMimeType.BMP;
}

export function isImageDataPart(part: unknown): part is vscode.LanguageModelDataPart {
    return part instanceof vscode.LanguageModelDataPart
        && isChatImageMimeType(part.mimeType);
}
```

**Critical implication:** Any `LanguageModelDataPart` with a MIME type outside these five values will **not** be recognised as an image by Copilot Chat. It will be processed as a generic data part.

## 4. Tool Registration Pattern

Location: `src/extension/tools/common/toolsRegistry.ts`

```typescript
// Internal two-layer registration:
//   Layer 1: Static constructor registry
ToolRegistry.registerTool(GetNotebookCellOutputTool);

//   Layer 2: Runtime bridge (in ToolsContribution)
for (const [name, tool] of toolsService.copilotTools) {
    this._register(vscode.lm.registerTool(getContributedToolName(name), tool));
}
```

**For 3rd-party extensions:** Skip both layers. Call `vscode.lm.registerTool()` directly in `activate()`. The internal `ToolRegistry` and `ICopilotTool<T>` interface are private to Copilot Chat.

## 5. Vision Capability Detection

Location: `src/platform/endpoint/node/chatEndpoint.ts`

```typescript
// Properties on the endpoint object
interface IChatEndpoint {
    supportsVision: boolean;
    maxPromptImages?: number;
    // ...
}
```

Location: `src/platform/endpoint/vscode-node/extChatEndpoint.ts`

```typescript
// Maps model families to vision capability
const supportsImageToText: ReadonlySet<string> = new Set([
    'gpt-4o', 'gpt-4o-mini', 'gpt-4-turbo',
    'claude-3-5-sonnet', 'claude-3-5-haiku', 'claude-3-opus',
    'claude-3-7-sonnet', 'claude-4-sonnet', 'claude-4-opus',
    'gemini-1.5-pro', 'gemini-1.5-flash', 'gemini-2.0-flash',
    // ... and others
]);
```

**For 3rd-party extensions:** This information is not exposed via public API. The extension cannot query whether the current model supports vision. The recommended approach is to always return the image and let the pipeline handle capability detection.

## 6. Automode Vision Fallback

Location: `src/platform/endpoint/node/automodeService.ts`

```typescript
// Simplified from source
private _applyVisionFallback(): void {
    // Detects image references in the conversation
    const hasImages = this._detectImageReferences();
    if (hasImages && !this._currentEndpoint.supportsVision) {
        // Switch to the first vision-capable model
        const visionModel = this._findVisionCapableModel();
        if (visionModel) {
            this._currentEndpoint = visionModel;
        }
    }
}
```

**Implication:** Even if the user's selected model lacks vision, automode may transparently switch to a vision-capable model when images are detected. This means the tool's images have a reasonable chance of reaching a vision-capable model regardless of the user's initial model choice.

## 7. Wire Format

### OpenAI / CAPI Format

Location: `src/platform/networking/common/openai.ts`

```typescript
// Images in the wire format
{
    role: 'tool',
    tool_call_id: '...',
    content: [
        {
            type: 'image_url',
            image_url: {
                url: 'data:image/png;base64,iVBOR...',
                detail: 'auto'
            }
        }
    ]
}
```

### Responses API Format

Location: `src/platform/endpoint/node/responsesApi.ts`

```typescript
// Images in Responses API format
{
    type: 'tool_result',
    tool_use_id: '...',
    content: [
        {
            type: 'input_image',
            source: {
                type: 'base64',
                media_type: 'image/png',
                data: 'iVBOR...'
            }
        }
    ]
}
```

**For 3rd-party extensions:** Wire format serialisation is handled entirely by VS Code and Copilot Chat. The extension only needs to return `LanguageModelDataPart.image()`. The framework handles the rest.

## 8. Implementation Gotchas Summary

| # | Gotcha | Impact | Mitigation |
|---|--------|--------|------------|
| 1 | Image budget is 2.5 MB cumulative | Large images silently dropped | Keep images small; warn in docs |
| 2 | Vision gating is internal | Images on non-vision models have no effect | Cannot control; automode fallback helps |
| 3 | Anthropic: no image URLs | Must use base64 inline | Handled by pipeline; no action needed |
| 4 | `isImageDataPart()` requires exact MIME | Wrong MIME → treated as non-image data | Use exact MIME strings from `ChatImageMimeType` |
| 5 | No public API for vision detection | Cannot check before returning image | Always return; let pipeline handle |
| 6 | `ExtendedLanguageModelToolResult` is proposed | Cannot use `toolResultMessage` etc. | Use standard `LanguageModelToolResult` |
| 7 | Token cost not exposed | Cannot warn about budget impact | Document token cost formula for users |
