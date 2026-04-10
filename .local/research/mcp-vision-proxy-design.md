# MCP Vision Proxy — Design Document

**Date:** 2026-02-26
**Type:** Design Document (implementation plan for an MCP server)
**Status:** Draft

---

## 1. Problem Statement

The user's organisation has `editor_preview_features = '0'` in the GitHub Copilot token. This triggers two enforcement layers:

- **Client (Gate A):** `isEditorPreviewFeaturesEnabled()` silently strips image content parts from prompts in `image.tsx`
- **Server (Gate B):** CAPI rejects API requests containing `image_url`, `input_image`, or `image` content parts

These gates block *all* image-to-model paths — user attachments, tool results, and MCP image responses alike. Any approach that includes image content parts in the CAPI request is structurally blocked.

## 2. Solution Architecture

An MCP server that acts as a **vision analysis proxy**. It accepts an image reference (file path or URL), sends the raw image to an external vision API, and returns the analysis as **text only**. No image content parts reach CAPI.

```
┌─────────────────────────────────────────────────┐
│                  Copilot Chat                    │
│                                                  │
│  LLM decides to call "analyze_image" tool        │
│  Input: { imagePath: "/path/to/screenshot.png",  │
│           prompt: "Extract all error messages" }  │
│                                                  │
│  ┌────────────────────────────────────────────┐  │
│  │ VS Code MCP Client (stdio transport)       │  │
│  │ tools/call → JSON-RPC                      │  │
│  └─────────────────┬──────────────────────────┘  │
│                    │                              │
└────────────────────┼──────────────────────────────┘
                     │ stdio
                     ▼
┌─────────────────────────────────────────────────┐
│            MCP Vision Proxy Server               │
│                                                  │
│  1. Read image from disk / URL                   │
│  2. Select provider (OpenAI / Gemini / Claude)   │
│  3. Send image + analysis prompt to vision API   │
│  4. Receive structured text analysis             │
│  5. Return { type: "text", text: "..." }         │
│                                                  │
│  No image data returned to Copilot.              │
│  Only text analysis flows back through CAPI.     │
│                                                  │
└──────────────────┬──────────────────────────────┘
                   │ HTTPS
                   ▼
┌─────────────────────────────────────────────────┐
│         External Vision API                      │
│  (OpenAI GPT-4o / Gemini / Claude / any OAI-    │
│   compatible endpoint)                           │
└─────────────────────────────────────────────────┘
```

### Why This Works

| Layer | Impact |
|-------|--------|
| Gate A (client) | Not triggered — no image content parts in tool result |
| Gate B (server) | Not triggered — CAPI request contains only text |
| MCP protocol | Text-only `CallToolResult` — fully supported |
| External API | Direct HTTPS from MCP server process — CAPI not involved |

---

## 3. API Contract

### Tool: `analyze_image`

**Description:** Reads an image from a local file path or URL and returns a structured text analysis using an external vision model.

#### Input Schema

```json
{
  "type": "object",
  "properties": {
    "imagePath": {
      "type": "string",
      "description": "Absolute path to a local image file, or an HTTP(S) URL"
    },
    "prompt": {
      "type": "string",
      "description": "Optional analysis instruction. Examples: 'Extract all text', 'Describe the UI layout', 'Identify error messages'. Defaults to structured analysis if omitted."
    },
    "detail": {
      "type": "string",
      "enum": ["low", "high", "auto"],
      "description": "Image detail level for the vision model. 'low' uses fewer tokens. Default: 'auto'."
    }
  },
  "required": ["imagePath"]
}
```

#### Output Format

The tool returns a single `TextContent` block containing a structured analysis:

```json
{
  "content": [{
    "type": "text",
    "text": "## Image Analysis\n\n### Text Content\n...\n\n### Visual Elements\n...\n\n### Layout\n...\n\n### Notable Details\n..."
  }]
}
```

When the user provides a custom `prompt`, the analysis follows that instruction instead of the default structured format.

#### Error Cases

```json
{
  "content": [{ "type": "text", "text": "Error: File not found: /path/to/image.png" }],
  "isError": true
}
```

---

## 4. Provider Abstraction

The server supports multiple vision API providers through a common interface. **The recommended provider for the user's environment is Azure OpenAI** (see Section 4.1).

### 4.1 Recommended Configuration: Azure OpenAI

The user has Azure OpenAI API keys available. This is the path of least resistance:

- Azure OpenAI's Chat Completions API supports vision-capable models (GPT-4o, GPT-4o-mini)
- The request goes directly to the Azure endpoint — CAPI is not involved
- No additional dependencies beyond Node.js built-in `fetch()`

**VS Code MCP configuration:**

```jsonc
{
  "mcp": {
    "servers": {
      "vision-proxy": {
        "type": "stdio",
        "command": "node",
        "args": ["${userHome}/.vscode-mcp/vision-proxy/out/index.js"],
        "env": {
          "VISION_PROVIDER": "azure",
          "AZURE_API_KEY": "${env:AZURE_OPENAI_API_KEY}",
          "AZURE_ENDPOINT": "https://<your-resource>.openai.azure.com",
          "AZURE_DEPLOYMENT": "gpt-4o",
          "AZURE_API_VERSION": "2024-12-01-preview"
        }
      }
    }
  }
}
```

**Azure OpenAI request format:**

```
POST https://<resource>.openai.azure.com/openai/deployments/<deployment>/chat/completions?api-version=2024-12-01-preview
Headers:
  api-key: <AZURE_API_KEY>
  Content-Type: application/json

Body:
{
  "messages": [{
    "role": "user",
    "content": [
      { "type": "text", "text": "<analysis prompt>" },
      { "type": "image_url", "image_url": { "url": "data:image/png;base64,...", "detail": "auto" } }
    ]
  }],
  "max_tokens": 4096
}
```

**Why not Codex CLI:** While the Codex CLI could serve as a vision backend via subprocess invocation, the direct API approach is simpler, faster (no process spawn overhead), and more reliable (stable JSON API contract vs CLI output parsing). The MCP server remains a single Node.js process with zero external tool dependencies.

### Provider Interface

```typescript
interface VisionProvider {
  name: string;
  analyzeImage(params: {
    imageData: Buffer;
    mimeType: string;
    prompt: string;
    detail: 'low' | 'high' | 'auto';
  }): Promise<string>;
}
```

### Supported Providers

| Provider | API Format | Endpoint | Auth |
|----------|-----------|----------|------|
| **OpenAI** | Chat Completions with `image_url` content part | `https://api.openai.com/v1/chat/completions` | `Authorization: Bearer $OPENAI_API_KEY` |
| **Azure OpenAI** | Same as OpenAI, different endpoint | `https://{resource}.openai.azure.com/...` | `api-key: $AZURE_API_KEY` |
| **Google Gemini** | `generateContent` with `inlineData` | `https://generativelanguage.googleapis.com/v1beta/models/{model}:generateContent` | `x-goog-api-key: $GEMINI_API_KEY` |
| **Anthropic Claude** | Messages API with `image` source | `https://api.anthropic.com/v1/messages` | `x-api-key: $ANTHROPIC_API_KEY` |
| **OpenAI-compatible** | Same as OpenAI, custom base URL | User-configured | User-configured |

### Provider Selection

Determined by environment variable `VISION_PROVIDER`:

| Value | Provider | Required Environment Variables |
|-------|----------|-------------------------------|
| `openai` | OpenAI GPT-4o | `OPENAI_API_KEY`, optional `OPENAI_MODEL` |
| `azure` | Azure OpenAI | `AZURE_API_KEY`, `AZURE_ENDPOINT`, `AZURE_DEPLOYMENT` |
| `gemini` | Google Gemini | `GEMINI_API_KEY`, optional `GEMINI_MODEL` |
| `anthropic` | Anthropic Claude | `ANTHROPIC_API_KEY`, optional `ANTHROPIC_MODEL` |
| `custom` | Any OpenAI-compatible | `CUSTOM_API_KEY`, `CUSTOM_BASE_URL`, optional `CUSTOM_MODEL` |

### Default Models

| Provider | Default Model |
|----------|--------------|
| OpenAI | `gpt-4o` |
| Gemini | `gemini-2.0-flash` |
| Anthropic | `claude-sonnet-4-20250514` |
| Custom | `gpt-4o` |

---

## 5. Server Configuration

### VS Code Settings (`settings.json` or `.vscode/mcp.json`)

```jsonc
{
  "mcp": {
    "servers": {
      "vision-proxy": {
        "type": "stdio",
        "command": "node",
        "args": ["${userHome}/.vscode-mcp/vision-proxy/out/index.js"],
        "env": {
          "VISION_PROVIDER": "openai",
          "OPENAI_API_KEY": "${env:OPENAI_API_KEY}"
        }
      }
    }
  }
}
```

Alternative configurations:

```jsonc
// Gemini
{
  "env": {
    "VISION_PROVIDER": "gemini",
    "GEMINI_API_KEY": "${env:GEMINI_API_KEY}"
  }
}

// Azure OpenAI
{
  "env": {
    "VISION_PROVIDER": "azure",
    "AZURE_API_KEY": "${env:AZURE_API_KEY}",
    "AZURE_ENDPOINT": "https://myresource.openai.azure.com",
    "AZURE_DEPLOYMENT": "gpt-4o"
  }
}

// Self-hosted / OpenAI-compatible
{
  "env": {
    "VISION_PROVIDER": "custom",
    "CUSTOM_API_KEY": "sk-...",
    "CUSTOM_BASE_URL": "http://localhost:11434/v1",
    "CUSTOM_MODEL": "llava"
  }
}
```

---

## 6. Default System Prompt

When the user does not provide a custom `prompt`, the server uses a structured analysis system prompt:

```
You are an image analysis assistant. Analyze the provided image and return a structured report with the following sections:

## Text Content
Extract ALL visible text, preserving structure (headings, lists, code blocks, error messages).

## Visual Elements
Describe key visual elements: icons, buttons, graphs, diagrams, UI components.

## Layout
Describe the spatial arrangement and visual hierarchy.

## Notable Details
Highlight anything unusual, errors, warnings, or items likely to be of interest to a software developer.

Be concise and factual. Do not speculate about context not visible in the image.
```

When a custom `prompt` is provided, it replaces the system prompt entirely. The user's prompt becomes the sole instruction for the vision model.

---

## 7. Security Considerations

| Concern | Mitigation |
|---------|------------|
| **Path traversal** | Canonicalize paths with `path.resolve()`. Optionally restrict to workspace directories. |
| **URL SSRF** | Restrict to `https://` URLs only (no `file://`, `ftp://`, `data://`). Optionally allowlist domains. |
| **API key exposure** | Keys are in environment variables, not in code or tool arguments. VS Code MCP `env` field supports `${env:VAR}` references. |
| **File size** | Enforce maximum file size (e.g., 20 MB) before reading. |
| **MIME validation** | Verify file magic bytes match expected image formats before sending to vision API. |
| **Output sanitization** | Vision model output is plain text — no execution risk. But truncate if excessively large (e.g., >50 KB). |

---

## 8. Project Structure

```
mcp-vision-proxy/
├── package.json
├── tsconfig.json
├── src/
│   ├── index.ts              # Entry point: create server, connect stdio transport
│   ├── server.ts             # McpServer setup + tool registration
│   ├── analyzeImage.ts       # Tool handler: read image, call provider, return text
│   ├── imageReader.ts        # Read image from file path or URL, validate, return Buffer
│   ├── providers/
│   │   ├── types.ts          # VisionProvider interface
│   │   ├── openai.ts         # OpenAI / Azure OpenAI / OpenAI-compatible provider
│   │   ├── gemini.ts         # Google Gemini provider
│   │   ├── anthropic.ts      # Anthropic Claude provider
│   │   └── factory.ts        # Provider factory: select based on VISION_PROVIDER env var
│   └── util/
│       ├── mimeDetect.ts     # Magic-byte MIME type detection
│       └── config.ts         # Environment variable reading + validation
└── out/                      # Compiled output
```

### Dependencies

```json
{
  "dependencies": {
    "@modelcontextprotocol/sdk": "^1.26.0",
    "zod": "^3.23.0"
  },
  "devDependencies": {
    "typescript": "~5.8.0",
    "@types/node": "^22.0.0"
  }
}
```

No external HTTP libraries required — Node.js built-in `fetch` (available since Node 18) handles all HTTP requests.

---

## 9. Data Flow (Detailed)

### Happy Path

```
1. LLM generates tool call: analyze_image({ imagePath: "/Users/me/screenshot.png" })
2. VS Code MCP client sends JSON-RPC tools/call over stdio
3. MCP server receives invocation
4. imageReader.ts:
   a. path.resolve(imagePath) → canonicalized absolute path
   b. fs.stat() → check size < 20 MB
   c. fs.readFile() → Buffer
   d. mimeDetect(buffer) → "image/png"
5. factory.ts: select provider based on VISION_PROVIDER env
6. provider.analyzeImage({ imageData, mimeType, prompt, detail })
   a. Construct vision API request with base64-encoded image
   b. POST to provider endpoint
   c. Parse response → analysis text string
7. Return to MCP client:
   { content: [{ type: "text", text: "## Image Analysis\n\n### Text Content\n..." }] }
8. VS Code MCP client delivers to Copilot Chat as IToolResult (text parts only)
9. PrimitiveToolResult.render() appends text to prompt
10. CAPI request contains only text content parts → no gate triggered
```

### URL Path

Same as above, but step 4 uses `fetch(url)` instead of `fs.readFile()`. The `imageReader.ts` module detects `https://` prefix and routes accordingly.

### Error Paths

| Error | Step | Result |
|-------|------|--------|
| File not found | 4b | `{ isError: true, content: [{ type: "text", text: "Error: File not found: ..." }] }` |
| File too large | 4b | `{ isError: true, content: [{ type: "text", text: "Error: File exceeds 20 MB limit" }] }` |
| Unsupported format | 4d | `{ isError: true, content: [{ type: "text", text: "Error: Unsupported image format" }] }` |
| Vision API error | 6b | `{ isError: true, content: [{ type: "text", text: "Error: Vision API returned: ..." }] }` |
| Missing API key | 5 | `{ isError: true, content: [{ type: "text", text: "Error: OPENAI_API_KEY not set" }] }` |

---

## 10. Usage Patterns

### Direct Analysis

User types in Copilot Chat:
> "Analyze the screenshot at ~/Desktop/error.png and tell me what error occurred"

The LLM calls `analyze_image({ imagePath: "~/Desktop/error.png", prompt: "What error is shown in this screenshot?" })`.

### Code Generation from UI Screenshot

> "Look at ~/mockup.png and generate HTML/CSS that matches it"

The LLM calls `analyze_image({ imagePath: "~/mockup.png", prompt: "Describe every UI element, its position, styling, colors, and text content in detail" })`, then uses the analysis to generate code.

### Error Diagnosis

> "Read the terminal screenshot at /tmp/build-error.png and help me fix the build errors"

The LLM calls `analyze_image({ imagePath: "/tmp/build-error.png", prompt: "Extract all error messages and stack traces exactly as shown" })`.

### Chained with Other Tools

The LLM can call `analyze_image` to understand an image, then use the text analysis to inform subsequent tool calls (file edits, terminal commands, etc.).

---

## 11. Limitations & Tradeoffs

| Dimension | Native Image Support | MCP Vision Proxy |
|-----------|---------------------|------------------|
| **Fidelity** | Model sees raw pixels | Model receives text description from intermediary |
| **Spatial reasoning** | Full | Degraded — depends on intermediary's description quality |
| **Follow-up questions** | Can reference same image | Must re-analyse (or cache description) |
| **Cost** | 1 model call | 2 model calls (vision API + Copilot) |
| **Latency** | Single round-trip | Additional vision API call (typically 2-8s) |
| **Privacy** | Image goes to CAPI (GitHub) | Image goes to chosen vision API provider |
| **Token budget** | Image tiles consume tokens in main context | Only text analysis consumes tokens (typically much smaller) |
| **Works with org policy** | No | **Yes** |

### Well-Served Use Cases

- Extracting text from screenshots (error messages, logs, code)
- Understanding simple diagrams and flowcharts
- Reading UI elements and their labels
- Identifying icons, buttons, and navigation structures

### Poorly-Served Use Cases

- Precise color analysis (hex values)
- Pixel-level spatial positioning
- Comparing two images for differences
- Subtle visual defects or anti-aliasing issues

---

## 12. Future Considerations

| Enhancement | Value | Complexity |
|-------------|-------|------------|
| **Caching** | Cache analysis results by image hash to avoid re-analysis on follow-up questions | Low |
| **Multiple images** | Accept an array of image paths for batch analysis | Low |
| **Clipboard integration** | Accept `"clipboard"` as imagePath, read from system clipboard | Medium |
| **Local model fallback** | Add a `local` provider using Ollama/LLaVA for offline use | Medium |
| **OCR specialization** | Dedicated OCR tool for high-precision text extraction (Tesseract) | Medium |
| **Image comparison** | Two-image diff analysis tool | Medium |
| **VS Code extension wrapper** | Wrap as a VS Code extension for easier installation and settings GUI | Medium |

---

## Appendix A: Provider API Formats

### OpenAI (Chat Completions)

```json
{
  "model": "gpt-4o",
  "messages": [{
    "role": "user",
    "content": [
      { "type": "text", "text": "<system prompt + user prompt>" },
      { "type": "image_url", "image_url": { "url": "data:image/png;base64,...", "detail": "auto" } }
    ]
  }],
  "max_tokens": 4096
}
```

### Google Gemini

```json
{
  "contents": [{
    "parts": [
      { "text": "<system prompt + user prompt>" },
      { "inlineData": { "mimeType": "image/png", "data": "<base64>" } }
    ]
  }]
}
```

### Anthropic Claude (Messages API)

```json
{
  "model": "claude-sonnet-4-20250514",
  "max_tokens": 4096,
  "messages": [{
    "role": "user",
    "content": [
      { "type": "text", "text": "<system prompt + user prompt>" },
      { "type": "image", "source": { "type": "base64", "media_type": "image/png", "data": "<base64>" } }
    ]
  }]
}
```

---

## Appendix B: MIME Detection by Magic Bytes

| Format | Magic Bytes | Hex |
|--------|-------------|-----|
| PNG | `\x89PNG\r\n\x1a\n` | `89 50 4E 47 0D 0A 1A 0A` |
| JPEG | `\xFF\xD8\xFF` | `FF D8 FF` |
| GIF | `GIF87a` or `GIF89a` | `47 49 46 38 37/39 61` |
| WebP | `RIFF....WEBP` | `52 49 46 46 xx xx xx xx 57 45 42 50` |
| BMP | `BM` | `42 4D` |
