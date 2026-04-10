# 04 — Edge Cases, Limitations & Platform Constraints

## Image Processing Pipeline Constraints

These constraints are imposed by Copilot Chat's internal image handling, not by the tool itself. They apply to **any** tool that returns `LanguageModelDataPart.image()`.

### 1. Image Budget: 2.5 MB

`PrimitiveToolResult.onImage()` in `toolCalling.tsx` maintains a cumulative image budget:

```typescript
// Internal to Copilot Chat — approximate logic
const IMAGE_BUDGET = 2.5 * 1024 * 1024; // 2.5 MB
if (currentBudget + imageSize > IMAGE_BUDGET) {
    logService.info('Image budget exceeded, dropping image');
    return; // Image silently dropped
}
```

**Implication:** If the tool returns a 3 MB image, it will be read successfully but **silently dropped** by the pipeline before reaching the model. The tool cannot detect this.

**Mitigation:** The tool enforces a 5 MB limit (generous), but users should be aware that images approaching 2.5 MB may be dropped if other images are also present in the conversation context.

### 2. Vision Capability Gating

Images are only rendered into the model context if the current endpoint has `supportsVision: true`. Without vision support:

- The `imageDataPartToTSX()` function checks `endpoint?.supportsVision`.
- If vision is not supported, the image is not included in the prompt.
- Automode's `_applyVisionFallback()` may automatically switch to a vision-capable model.

**Implication:** On models without vision (e.g., some older GPT models), the tool will execute successfully, but the image data will not reach the model. The model receives an empty or text-only result.

### 3. Anthropic Image URL Exclusion

`modelCanUseMcpResultImageURL()` in `chatModelCapabilities.ts` excludes Anthropic models and "Hidden Model E" from receiving image URLs in tool results:

```typescript
export function modelCanUseMcpResultImageURL(family: string): boolean {
    if (family.startsWith('claude') || family === 'hidden-model-e') {
        return false;
    }
    return true;
}
```

**Implication:** For Anthropic models (Claude family), images in tool results are sent as base64-encoded inline content, never as uploaded URLs. This works correctly but increases payload size.

### 4. Token Cost

Each image consumes prompt tokens:

```
tokens = 85 + (170 × ceil(width / 512) × ceil(height / 512))
```

A 1920×1080 screenshot: `85 + (170 × 4 × 3) = 85 + 2040 = 2125 tokens`.

This is deducted from the prompt budget. In a tight-budget conversation with many code files, a large image may cause other context to be evicted.

## Tool-Level Edge Cases

### 5. Symlinks and Special Files

`workspace.fs.readFile()` follows symlinks transparently. However:

- **Device files** (e.g., `/dev/zero`) will hang or produce infinite data.
- **Named pipes** may block indefinitely.

The `stat()` check for `FileType.File` catches directories but not device files. On typical workspace file systems, this is not a practical concern.

### 6. File Locking

On Windows, if an image file is exclusively locked by another process (e.g., an image editor has it open with exclusive access), `workspace.fs.readFile()` may throw. The tool returns a text error result in this case.

### 7. Remote Workspaces

`workspace.fs.readFile()` works transparently across VS Code remote connections (SSH, WSL, Dev Containers, GitHub Codespaces). The URI scheme is handled by VS Code's virtual file system layer.

However, **performance** may degrade significantly for large images over slow connections. A 5 MB image over SSH on a slow link could take several seconds to transfer.

### 8. Binary File Misidentification

If a non-image file has an image extension (e.g., a `.png` file that is actually a text file), the tool will:

1. Pass the extension check.
2. Read the file.
3. Return it as `LanguageModelDataPart.image()`.
4. The downstream pipeline will attempt to base64-encode it.
5. The model will receive corrupted or blank "image" content.

**Mitigation:** Optional magic-byte validation (see Implementation Notes in 03-implementation.md) would catch this. Not implemented in the base version to keep dependencies minimal.

### 9. Empty Files

A 0-byte `.png` file will pass all checks and be returned as an empty `Uint8Array`. The model will receive an empty image data part, which it will likely describe as "blank" or "could not parse."

### 10. Unicode Paths

`vscode.Uri.file()` handles Unicode paths correctly on all platforms. No special handling is needed for paths containing CJK characters, emoji, or other Unicode code points.

### 11. Workspace Trust

VS Code's workspace trust feature restricts extension capabilities in untrusted workspaces. The tool uses `workspace.fs` which respects trust restrictions. In an untrusted workspace, file access may be limited or denied depending on the extension's trust configuration in `package.json`:

```jsonc
{
    "capabilities": {
        "untrustedWorkspaces": {
            "supported": true,
            "description": "This extension reads image files from the workspace."
        }
    }
}
```

## Model Behavior Considerations

### 12. Tool Selection Reliability

The model may not always choose to invoke `readImage` when it encounters image files. Tool selection depends on:

- **Model capability** — newer models are better at tool selection.
- **`modelDescription` quality** — a precise, well-structured description improves selection.
- **Context** — if the user explicitly mentions "look at the screenshot" the model is more likely to invoke the tool.
- **Competing tools** — if other tools could plausibly handle the request, the model may choose them instead.

### 13. Hallucinated Paths

The model may hallucinate file paths — constructing paths that do not exist. The tool handles this gracefully by returning a text error: "File not found or inaccessible." The model can then ask the user or use a file listing tool to find the correct path.

### 14. Repeated Invocations

In an agentic loop, the model may invoke `readImage` multiple times on the same file. Each invocation re-reads the file from disk. There is no caching. For most image sizes and local file systems, this is negligible, but on remote connections it may be wasteful.

**Future enhancement:** A URI-keyed `Map<string, LanguageModelDataPart>` cache with TTL could avoid redundant reads.

## Platform Compatibility Matrix

| Platform | Support | Notes |
|----------|---------|-------|
| macOS (local) | Full | Native file system access |
| Windows (local) | Full | URI handles drive letters correctly |
| Linux (local) | Full | Native file system access |
| WSL | Full | `workspace.fs` handles `vscode-remote://wsl+` URIs |
| SSH Remote | Full | Performance dependent on connection speed |
| Dev Containers | Full | File system accessed via container mount |
| GitHub Codespaces | Full | Remote file system via Codespaces agent |
| vscode.dev (web) | Partial | Depends on file system provider; `workspace.fs` may not support all operations |
| Virtual file systems | Varies | Works if the FS provider implements `readFile` |
