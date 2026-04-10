# GitHub Copilot Image Features — Exhaustive Audit

**Date:** 2026-02-26
**Codebase versions:** Local clones of `vscode-copilot-chat` and `vscode`

---

## Executive Summary

Image features in GitHub Copilot Chat remain **preview/experimental** as of this audit. There is no evidence in either codebase of a GA (general availability) milestone being reached or imminent. Organisations that disable preview features via the Copilot token policy (`editor_preview_features = '0'`) will have all image functionality silently suppressed.

Three independent gating layers must all permit images before they function:

1. **Organisational policy** — server-side Copilot token flag
2. **Proposed VS Code API** — `chatReferenceBinaryData` must be enabled
3. **Model capability** — the selected model must declare `vision` support

If any single layer blocks, images are either silently dropped from prompts or displayed with a warning in the UI.

---

## Table of Contents

1. [Complete Feature Inventory](#1-complete-feature-inventory)
2. [Gating Architecture](#2-gating-architecture)
3. [Model Support](#3-model-support)
4. [API Dependency Map](#4-api-dependency-map)
5. [Key Code Paths](#5-key-code-paths)
6. [GA Status & Timeline](#6-ga-status--timeline)
7. [Workaround Analysis](#7-workaround-analysis)
8. [Supported Formats](#8-supported-formats)

---

## 1. Complete Feature Inventory

### User-Facing Features

| # | Feature | Source | Status | Gate(s) |
|---|---------|--------|--------|---------|
| 1 | **Paste image into chat** | VS Code core: `chatPasteProviders.ts` | Preview | `chatReferenceBinaryData` proposed API |
| 2 | **Drag-and-drop image** | VS Code core: `chatDragAndDrop.ts` | Preview | `chatReferenceBinaryData` proposed API |
| 3 | **Attach image from clipboard** (context menu) | VS Code core: `chatContext.ts` | Preview | `supportsImageAttachments` + model `vision` capability |
| 4 | **Screenshot window** | VS Code core: `chatContext.ts` | Preview | `supportsImageAttachments` + model `vision` + `IHostService.getScreenshot()` |
| 5 | **Attach image file** | VS Code core: `chatAttachmentModel.ts` | Preview | File extension match → `asImageVariableEntry` |
| 6 | **Send elements to chat** (Simple Browser) | VS Code core: `simpleBrowserEditorOverlay.ts` | Experimental | `chat.sendElementsToChat.attachImages` setting (default: `false`) |
| 7 | **Alt text generation** (inline chat) | Copilot Chat: `inlineChatCodeActions.ts` | Preview | — |

### Internal/Pipeline Features

| # | Feature | Source | Status | Description |
|---|---------|--------|--------|-------------|
| 8 | **Image rendering in prompts** | Copilot Chat: `image.tsx` | Preview | Encodes user-attached images as base64 or uploaded URL |
| 9 | **Image upload via CAPI** | Copilot Chat: `imageServiceImpl.ts` | Experimental | Uploads binary data to GitHub CAPI endpoint, returns URL |
| 10 | **Historical image replay** | Copilot Chat: `image.tsx` | Preview | Re-renders previously processed images from conversation history |
| 11 | **Tool result images** | Copilot Chat: `toolCalling.tsx` | Preview | Converts `LanguageModelDataPart` images from tool results; 2.5 MB budget |
| 12 | **Vision auto-fallback** | Copilot Chat: `automodeService.ts` | Preview | Auto-mode switches to vision-capable model when images detected |
| 13 | **Image count validation** | Copilot Chat: `chatEndpoint.ts` | Preview | Enforces `maxPromptImages` limit (Gemini family) |
| 14 | **Image token cost calc** | Copilot Chat: `tokenizer.ts` | Stable (internal) | OpenAI tile-based token pricing |
| 15 | **Image dimension extraction** | Copilot Chat: `imageUtils.ts` | Stable (internal) | PNG, JPEG, GIF, WebP dimension parsers |
| 16 | **Image resize** | VS Code core: `chatImageUtils.ts` | Internal | Canvas-based resize using OpenAI dimensioning algorithm; max 30 MB input |
| 17 | **BYOK vision config** | Copilot Chat: `configurationService.ts` | Experimental | User-declared `vision: boolean` on custom model configs |
| 18 | **MCP result image URL gate** | Copilot Chat: `chatModelCapabilities.ts` | Preview | Anthropic family excluded from receiving image URLs in MCP results |
| 19 | **Fetch web page images** | Copilot Chat: `fetchWebPageTool.tsx` | Preview | `fetchWebPage` tool passes images through if `supportsImageToText` |
| 20 | **OpenAI Responses API images** | Copilot Chat: `responsesApi.ts` | Preview | Converts to `input_image` format; handles `image_generation_call` outputs |
| 21 | **Anthropic Messages API images** | Copilot Chat: `messagesApi.ts` | Preview | Base64 only; restricted to JPEG, PNG, GIF, WebP |
| 22 | **Extension API image parts** | Copilot Chat: `extChatEndpoint.ts` | Preview | Converts to `LanguageModelDataPart` for extension-contributed models |

### readImage Tool

The `readImage` tool does **not** exist in VS Code core or the Copilot Chat extension. It is a separate third-party extension (`lmtools-read-image`). The closest core equivalent is the `fetchPage` tool in VS Code, which can return image data from URLs.

---

## 2. Gating Architecture

### Gate A — Organisational Policy Token (Server-Side)

| Attribute | Detail |
|-----------|--------|
| **Mechanism** | `isEditorPreviewFeaturesEnabled()` — checks Copilot token value `editor_preview_features` |
| **Location** | `copilotToken.ts` (~line 210) |
| **Default** | `true` (enabled) — returns `false` only when `editor_preview_features === '0'` |
| **Effect** | When disabled: `Image` and `HistoricalImage` components silently omit all image content from prompts |
| **Log message** | "Copilot preview features are disabled by organizational policy" via `contextKeys.contribution.ts` |
| **UI indicator** | `previewFeaturesDisabledContextKey` context key → warning icon on image attachments: "Vision is disabled by your organization" |

**This is the gate that affects your organisation.**

### Gate B — `chatReferenceBinaryData` Proposed API (VS Code Platform)

| Attribute | Detail |
|-----------|--------|
| **Mechanism** | `isProposedApiEnabled(ext, 'chatReferenceBinaryData')` |
| **Enforced in** | `chatAttachmentResolveService.ts`, `chatDragAndDrop.ts`, `chatPasteProviders.ts` |
| **Effect** | If no installed extension declares this proposed API, all image attachment UI (paste, drag-drop, file attach) is disabled |
| **Current state** | Copilot Chat extension declares it; without Copilot, image attachment does not function |

### Gate C — Model Vision Capability

| Attribute | Detail |
|-----------|--------|
| **Mechanism** | `modelMetadata.capabilities.supports.vision` (CAPI) / `capabilities.supportsImageToText` (proposed API) / `capabilities.imageInput` (stable API) |
| **Effect** | Soft gate — images are attached but flagged with warning if model lacks vision. Auto-mode attempts fallback. |
| **Auto-fallback** | `automodeService.ts` `_applyVisionFallback()` iterates available models, selects first with `supportsVision` |

### Gate D — Image Upload Experiment

| Attribute | Detail |
|-----------|--------|
| **Key** | `EnableChatImageUpload` (`chat.imageUpload.enabled`) |
| **Config type** | `ExperimentBased` — A/B experiment assignment |
| **Default** | `true` |
| **Tags** | `["experimental", "onExp"]` |
| **Effect** | Controls whether images are uploaded to CAPI (URL) or sent inline (base64). Upload is preferred when enabled. |

### Composite Gate Logic

```
Image content appears in LLM prompt IFF:
  Gate A: isEditorPreviewFeaturesEnabled() === true
  AND Gate C: supportsVision === true

Image attachment UI functional IFF:
  Gate B: chatReferenceBinaryData proposed API is enabled by installed extension

Image upload (vs inline base64):
  Gate D: EnableChatImageUpload experiment assignment
```

---

## 3. Model Support

### Vision Capability Detection

| Method | Source | Scope |
|--------|--------|-------|
| Server metadata: `capabilities.supports.vision` | `chatEndpoint.ts` | CAPI models |
| Stable API: `LanguageModelChatCapabilities.imageInput` | `vscode.d.ts` | All extension-visible models |
| Proposed API: `LanguageModelChat.capabilities.supportsImageToText` | Proposed `languageModelCapabilities` | Per-instance check (proposed) |
| Extension metadata: `capabilities.vision` | `languageModels.ts` | Internal model registration |
| BYOK user config: `vision: boolean` | `configurationService.ts` | Azure Models / Custom OAI Models |

### Model-Specific Constraints

| Model Family | Image URLs | MCP Result Image URLs | Max Images | Format Restrictions |
|-------------|------------|----------------------|------------|---------------------|
| OpenAI (GPT-4o, etc.) | Yes | Yes | Unlimited | All supported formats |
| Anthropic (Claude) | Yes | **No** (excluded) | Unlimited | JPEG, PNG, GIF, WebP only (no BMP) |
| Gemini | Yes | Yes | `maxPromptImages` from server | All supported formats |
| "HiddenModelE" | Yes | **No** (excluded) | Unknown | Unknown |
| BYOK models | Yes | Depends on declaration | Unlimited | Depends on provider |

### Auto-Mode Fallback

When auto-model selection encounters image attachments and the chosen model lacks vision:
1. `hasImage()` detects `image/*` MIME types in attachments
2. `_applyVisionFallback()` iterates registered models
3. Selects first model with `supportsVision === true`
4. Falls back silently — no user notification

---

## 4. API Dependency Map

### Stable APIs (Graduated to `vscode.d.ts`)

| API | Purpose |
|-----|---------|
| `LanguageModelDataPart` | Generic binary data part with `.image()`, `.json()`, `.text()` factories |
| `LanguageModelDataPart.image(data, mime)` | Construct image data part from raw bytes |
| `LanguageModelChatCapabilities.imageInput` | Boolean — model accepts image input |
| `LanguageModelChatMessage.User([...dataParts])` | User messages can contain data parts |

### Proposed APIs (Must Graduate for Full GA)

| Proposed API File | Key Type | Purpose | GA Impact |
|-------------------|----------|---------|-----------|
| `vscode.proposed.chatReferenceBinaryData.d.ts` | `ChatReferenceBinaryData` | Image attachments reaching extensions | **Primary blocker** — without this, chat UI image features do not function |
| `vscode.proposed.languageModelCapabilities.d.ts` | `supportsImageToText` | Per-model-instance vision check | Secondary — `imageInput` on stable API covers basic case |
| `vscode.proposed.languageModelToolResultAudience.d.ts` | `LanguageModelDataPart2` | Audience routing for data parts | Enrichment — not a blocker |
| `vscode.proposed.chatProvider.d.ts` | `LanguageModelResponsePart2` | Models emitting images in responses | Provider-side; not user-facing blocker |
| `vscode.proposed.languageModelThinkingPart.d.ts` | `LanguageModelChatMessage2` | Extended message content with data parts | Enrichment |

---

## 5. Key Code Paths

### User Attaches Image (Full Path)

```
User action (paste/drag/drop/attach/screenshot)
  → VS Code core: chatPasteProviders.ts / chatDragAndDrop.ts / chatContext.ts / chatAttachmentModel.ts
    → Gate B check: chatReferenceBinaryData proposed API enabled?
    → chatAttachmentModel.ts: IImageVariableEntry created (kind: 'image')
    → chatImageUtils.ts: resizeImage() — canvas resize, max 30 MB
    → chatAttachmentResolveService.ts: resolveImageEditorAttachContext()
    → Extension host: extHostTypeConverters.ts → ChatReferenceBinaryData
  → Copilot Chat extension: image.tsx: Image.render()
    → Gate A check: isEditorPreviewFeaturesEnabled()?
    → Gate C check: supportsVision?
    → Gate D check: EnableChatImageUpload → upload vs base64
    → imageServiceImpl.ts: upload to CAPI (if upload enabled)
    → imageUtils.ts: getImageDimensions() + getMimeType()
    → tokenizer.ts: calculateImageTokenCost()
    → Output: ChatCompletionContentPartKind.Image
  → API serialization layer:
    → chatEndpoint.ts (CAPI / OpenAI Chat Completions)
    → responsesApi.ts (OpenAI Responses API)
    → messagesApi.ts (Anthropic Messages API)
    → extChatEndpoint.ts (Extension-contributed models)
```

### Tool Returns Image

```
Tool execution returns LanguageModelDataPart
  → Copilot Chat: toolCalling.tsx: imageDataPartToTSX()
    → modelCanUseMcpResultImageURL() — Anthropic excluded
    → 2.5 MB inline budget enforcement
    → Upload or inline decision
    → Output: Image content in prompt
```

### Auto-Mode Vision Fallback

```
Auto-model selection triggered
  → automodeService.ts: hasImage() detects image attachments
  → Model lacks vision?
    → _applyVisionFallback(): iterate models, select first with supportsVision
    → Switch model silently
```

---

## 6. GA Status & Timeline

### Current Status: PREVIEW

**Evidence supporting preview classification:**

1. **Organisational kill-switch exists.** `isEditorPreviewFeaturesEnabled()` is specifically designed for preview features that organisations may suppress. Its existence is definitional.

2. **Proposed API dependency.** `ChatReferenceBinaryData` remains in `vscode.proposed.chatReferenceBinaryData.d.ts`. Until this graduates to `vscode.d.ts`, the entire image attachment UI is gated behind proposed API enablement.

3. **Experimental tags in package.json.** The `chat.imageUpload.enabled` setting carries `["experimental", "onExp"]` tags, indicating experiment-controlled rollout.

4. **Experiment-based configuration.** `EnableChatImageUpload` uses `ConfigType.ExperimentBased`, meaning its value is determined by A/B experiment assignment.

5. **No GA markers found.** Neither codebase contains comments, changelog entries, or documentation asserting GA readiness for image features.

### Partial Progress Toward GA

Not all image APIs are proposed. Two critical types have already graduated:

| API | Status | Implication |
|-----|--------|-------------|
| `LanguageModelDataPart` | **Stable** | Extensions can construct and consume image data parts today |
| `LanguageModelChatCapabilities.imageInput` | **Stable** | Models can declare image support via stable API |
| `ChatReferenceBinaryData` | **Proposed** | Chat UI → extension image flow still gated |
| `supportsImageToText` | **Proposed** | Per-instance capability check still gated |

The graduation of `LanguageModelDataPart` and `imageInput` to stable suggests active progress, but `ChatReferenceBinaryData` remains the critical blocker.

### Timeline Estimate

No explicit timeline was found in either codebase. Based on the evidence:

- The feature has been in preview since at least early 2025 (consistent with user's report of March 2025)
- Core data types have graduated — indicating active stabilisation work
- The remaining proposed API (`ChatReferenceBinaryData`) is structurally simple
- No "do not ship" or "blocked by" markers were found

**Assessment:** The feature is on a graduation trajectory, but no committed date exists in the code.

---

## 7. Workaround Analysis

### For Organisations with `editor_preview_features = '0'`

| Approach | Feasibility | Risk |
|----------|-------------|------|
| **Request org admin to enable preview features** | Most direct path. The Copilot token's `editor_preview_features` value is set at the organisation level in GitHub settings. | Policy decision — depends on org risk tolerance. |
| **Wait for GA graduation** | No code changes required. When `ChatReferenceBinaryData` graduates and the `isEditorPreviewFeaturesEnabled()` gate is removed or bypassed for GA features, images will work. | Timeline unknown. |
| **Use the Extension API directly** | Extensions can construct `LanguageModelDataPart.image()` and include it in `LanguageModelChatMessage.User()` via the stable API. A custom extension could read images and inject them into model requests without using the chat attachment UI. | Requires custom development. Bypasses the chat UI entirely. The stable `imageInput` capability check works, but the model must actually support it. |
| **BYOK with vision-capable model** | Configure a bring-your-own-key model with `vision: true`. Image *processing* may still work if the preview gate only affects the chat UI attachment flow rather than the raw API path. | Untested — the `isEditorPreviewFeaturesEnabled()` check in `image.tsx` applies regardless of model source. BYOK does not bypass Gate A. |
| **Use VS Code Insiders with `--enable-proposed-api`** | Enables proposed APIs but does **not** bypass the Copilot token's `editor_preview_features` check. | Does not solve the core problem (Gate A). |

### Assessment

The organisational policy gate (Gate A) is enforced in the extension's prompt construction layer (`image.tsx`), not in the VS Code UI layer. This means:

- Even if you attach an image successfully in the UI, the extension will silently strip it from the prompt when `editor_preview_features === '0'`.
- The only reliable workaround is requesting the org admin to enable preview features, or waiting for the feature to exit preview.
- Custom extension development using the stable `LanguageModelDataPart` API could work but requires building an alternative to the built-in chat attachment flow.

---

## 8. Supported Image Formats

| Format | MIME Type | Dimension Parsing | Chat Attachment | Anthropic | OpenAI | Extension API |
|--------|-----------|-------------------|-----------------|-----------|--------|---------------|
| PNG | `image/png` | Yes | Yes | Yes | Yes | Yes |
| JPEG | `image/jpeg` | Yes | Yes | Yes | Yes | Yes |
| GIF | `image/gif` | Yes | Yes (current frame only) | Yes | Yes | Yes |
| WebP | `image/webp` | Yes | Yes | Yes | Yes | Yes |
| BMP | `image/bmp` | No | Paste only (not in attachment allowlist) | No | Yes | Yes |
| TIFF | `image/tiff` | No | Paste only | No | Unknown | Unknown |

**Notes:**
- GIF attachments are marked `OmittedState.Partial` — only the current frame is sent to the model
- BMP is in `ChatImageMimeType` enum but **not** in `CHAT_ATTACHABLE_IMAGE_MIME_TYPES`
- Maximum raw file size before resize: **30 MB** (VS Code core)
- Inline budget for tool result images: **2.5 MB** (Copilot Chat extension)
- Image resize targets: OpenAI's tile-based dimensioning algorithm (2048x2048 max, 768px short side, 512px tiles)

---

## Appendix: File Reference Index

### VS Code Core (`vscode/`)

| File | Relevance |
|------|-----------|
| `src/vscode-dts/vscode.d.ts` | Stable `LanguageModelDataPart`, `imageInput` definitions |
| `src/vscode-dts/vscode.proposed.chatReferenceBinaryData.d.ts` | `ChatReferenceBinaryData` — primary proposed API blocker |
| `src/vscode-dts/vscode.proposed.languageModelCapabilities.d.ts` | `supportsImageToText` proposed capability |
| `src/vs/workbench/contrib/chat/browser/widget/input/editor/chatPasteProviders.ts` | Paste image handling |
| `src/vs/workbench/contrib/chat/browser/widget/chatDragAndDrop.ts` | Drag-and-drop image handling |
| `src/vs/workbench/contrib/chat/browser/actions/chatContext.ts` | Clipboard image + screenshot actions |
| `src/vs/workbench/contrib/chat/browser/attachments/chatAttachmentModel.ts` | Image attachment model |
| `src/vs/workbench/contrib/chat/browser/attachments/chatAttachmentWidgets.ts` | Image attachment UI widget |
| `src/vs/workbench/contrib/chat/browser/attachments/chatAttachmentResolveService.ts` | Image attachment resolution |
| `src/vs/workbench/contrib/chat/browser/chatImageUtils.ts` | Image resize utilities |
| `src/vs/workbench/contrib/chat/common/model/chatModel.ts` | MIME type definitions |
| `src/vs/workbench/contrib/chat/common/languageModels.ts` | Model capabilities including vision |
| `src/vs/workbench/contrib/chat/common/attachments/chatVariableEntries.ts` | `IImageVariableEntry` definition |
| `src/vs/workbench/api/common/extHostTypeConverters.ts` | Extension host image serialization |
| `src/vs/workbench/services/chat/common/chatEntitlementService.ts` | Preview features disabled check |

### Copilot Chat Extension (`vscode-copilot-chat/`)

| File | Relevance |
|------|-----------|
| `src/extension/prompts/node/panel/image.tsx` | Core image rendering in prompts — Gate A + C enforcement |
| `src/platform/image/node/imageServiceImpl.ts` | CAPI image upload service |
| `src/platform/endpoint/node/automodeService.ts` | Vision auto-fallback |
| `src/platform/endpoint/node/chatEndpoint.ts` | Image count validation, vision metadata |
| `src/platform/endpoint/common/chatModelCapabilities.ts` | MCP result image URL gating |
| `src/platform/endpoint/node/responsesApi.ts` | OpenAI Responses API image handling |
| `src/platform/endpoint/node/messagesApi.ts` | Anthropic Messages API image handling |
| `src/platform/endpoint/vscode-node/extChatEndpoint.ts` | Extension model image handling |
| `src/platform/tokenizer/node/tokenizer.ts` | Image token cost calculation |
| `src/platform/authentication/common/copilotToken.ts` | `isEditorPreviewFeaturesEnabled()` — Gate A |
| `src/platform/configuration/common/configurationService.ts` | `EnableChatImageUpload` experiment config |
| `src/util/common/imageUtils.ts` | Image dimension extraction, MIME detection |
| `src/extension/prompts/node/panel/toolCalling.tsx` | Tool result image processing |
| `src/extension/inlineChat/vscode-node/inlineChatCodeActions.ts` | Alt text generation |
| `src/extension/tools/vscode-node/fetchWebPageTool.tsx` | Web page image passthrough |
| `src/extension/contextKeys/vscode-node/contextKeys.contribution.ts` | Preview features context key |

---

## 9. readImage LM Tool — Failure Analysis

### Background

A custom VS Code language model tool (`readImage`) was developed to bypass the organisational preview features gate. It registered via `vscode.lm.registerTool('readImage', ...)`, read image files with `vscode.workspace.fs.readFile()`, and returned `LanguageModelDataPart.image(data, mime)` in tool results.

### Why It Previously Worked

The tool exploited a gap between two enforcement layers:

1. **Gate A (client)** checks `isEditorPreviewFeaturesEnabled()` only in `image.tsx` — the component that renders *user-attached* images into prompts. Tool result images flow through a completely separate path: `toolCalling.tsx` → `PrimitiveToolResult.onImage()` → `imageDataPartToTSX()`. This path uses the base `Image` element from `@vscode/prompt-tsx`, **not** the gated `Image` class from `image.tsx`.

2. **Gate B (server)** did not exist. CAPI trusted the client to enforce policy.

The result: images from tools reached the model despite the organisational policy.

### Why It Stopped Working

**Server-side enforcement was added.** No client-side code changes were made to the tool result pipeline — `isEditorPreviewFeaturesEnabled()` is still not checked in `toolCalling.tsx`. Instead, CAPI now inspects the `editor_preview_features` claim in the bearer token and rejects requests containing image content parts (`image_url`, `input_image`, `image` types) when the value is `'0'`.

This is a **server-side policy enforcement** correction. The gap was unintentional; the server now mirrors the client-side gate.

### Evidence

- `isEditorPreviewFeaturesEnabled()` is called in exactly 2 locations: `image.tsx` lines 54 and 81. Neither is in the tool result path.
- `toolCalling.tsx` `PrimitiveToolResult.onImage()` (line 662) has no preview features check.
- `imageDataPartToTSX()` (line 448) has no preview features check.
- `chatEndpoint.ts` `createRequestBody()` has no image filtering based on preview features.
- The serialization layer (`openai.ts`, `responsesApi.ts`, `messagesApi.ts`) forwards all image content parts without policy checks.
- Conclusion: the rejection occurs at the CAPI server, not in any client code.

---

## 10. Workaround Analysis — All Approaches Evaluated

### Summary Matrix

| # | Approach | Bypasses Gate A (Client) | Bypasses Gate B (Server) | Requires API Keys | Verdict |
|---|----------|:---:|:---:|:---:|---------|
| 1 | Chat Provider API | No (user images) / Yes (tool images) | Only if non-CAPI | Yes | **Impractical** |
| 2 | BYOK / ExtChatEndpoint | No | Yes (for BYOK) | Yes, and BYOK disabled for org users | **Blocked** |
| 3 | Base64 in text part | Yes | Yes | No | **Blocked** (model can't parse) |
| 4 | Content part type manipulation | N/A | N/A | No | **Blocked** (protocol violation) |
| 5 | File reference injection (`#file:image.png`) | No | N/A | No | **Blocked** (same gated path) |
| 6 | Third-party extension model provider | No (user) / Yes (tool) | Yes | Via extension | **Theoretically possible** |
| 7 | Personal Copilot account | Yes | Yes | No | **Viable** (policy risk) |
| 8 | Fine-grained GitHub config | N/A | N/A | No | **Blocked** (doesn't exist) |
| 9 | MCP server returning text descriptions | N/A (text, not images) | N/A (text, not images) | Depends | **Viable** (degraded fidelity) |
| 10 | Request org admin to enable preview features | Yes | Yes | No | **Most direct** |

### Detailed Analysis

#### 1. Chat Provider API

A custom `LanguageModelChatProvider` receives raw API calls including `LanguageModelDataPart` images. However, user-attached images are stripped by Gate A before they reach any provider. Tool result images bypass Gate A, but the provider needs its own model backend — which subsumes the BYOK problem.

#### 2. BYOK / Extension-Contributed Endpoint

BYOK models bypass CAPI entirely (requests go to the configured endpoint). However, BYOK is gated by `copilotToken.isInternal || copilotToken.isIndividual` — **organisation-managed users are explicitly excluded**. Furthermore, Gate A still applies for user-attached images.

#### 3. Base64 in Text Part

Encodes image as base64 string in a `LanguageModelTextPart`. Bypasses both gates (the content is plain text). However, **LLMs cannot interpret raw base64 image data from text**. Vision models require structured image content parts to activate their vision pathways. A base64 string is just a long meaningless string.

#### 4. Content Part Type Manipulation

The serialization pipeline (`openai.ts`, `responsesApi.ts`, `messagesApi.ts`) produces strictly typed content parts. Non-standard types would cause protocol errors or be silently dropped.

#### 5. File Reference Injection

`#file:image.png` resolves through `ChatAttachmentResolveService` → `resolveImageEditorAttachContext()` → `ChatReferenceBinaryData` → handled by the gated `Image` class in `image.tsx`. Same path as drag-and-drop.

#### 6. Third-Party Extension Model Provider

The most technically interesting option. A third-party extension registering a `LanguageModelChatProvider` would receive images from tool results (which bypass Gate A) and could send them to its own backend (bypassing CAPI/Gate B). Requirements:
- A VS Code marketplace extension providing a vision-capable model
- The extension must handle image `LanguageModelDataPart` in requests
- User-attached images are still blocked by Gate A — only tool-generated images work

#### 7. Personal Copilot Account

Sign in with a personal GitHub account with an individual Copilot subscription. The token would lack `editor_preview_features=0`. Both gates open. **Policy risk:** may violate employment agreements that require use of the organisation account for work.

#### 8. Fine-Grained GitHub Configuration

The `editor_preview_features` flag is binary — all preview features or none. No per-feature granularity exists. The linked documentation (`https://aka.ms/github-copilot-org-enable-features`) confirms all-or-nothing.

#### 9. MCP Server with Text Descriptions

An MCP server that runs a local vision model (or calls an external vision API) to analyse the image and returns a text description. Copilot receives plain text, not image data — no gates apply. Tradeoffs:
- Loses direct visual analysis fidelity
- Adds latency and cost (two model calls)
- Works well for: text in images, error screenshots, code screenshots
- Works poorly for: UI layout critique, color/styling analysis, spatial reasoning

#### 10. Request Org Admin

The most direct, zero-risk technical solution. Request that the organisation enable preview features. The exact setting path is documented at `https://aka.ms/github-copilot-org-enable-features`.

### Recommended Path

For most users in this situation, the priority order is:

1. **Request org admin** (#10) — zero technical risk, full feature access
2. **MCP text-description server** (#9) — if admin request is denied, provides degraded but functional image understanding
3. **Third-party model extension** (#6) — if a suitable extension exists on the marketplace, provides full image capabilities for tool-generated images
4. **Wait for GA** (#N/A) — when image features graduate from preview, the organisational gate ceases to apply
