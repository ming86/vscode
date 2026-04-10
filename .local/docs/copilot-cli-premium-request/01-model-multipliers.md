# Model Multipliers and Billing Metadata

## Overview

Each model served by the Copilot API (CAPI) carries billing metadata that determines its cost in premium requests. The core formula:

```
premium_requests_consumed = 1 × multiplier
```

A multiplier of `0` means the model is included at no additional cost on paid plans. A multiplier of `3` means a single interaction consumes three premium requests.

## Model Metadata API Response

From `IModelAPIResponse` in `src/platform/endpoint/common/endpointProvider.ts`:

```typescript
interface IModelAPIResponse {
  id: string;
  vendor: string;
  name: string;
  model_picker_enabled: boolean;
  preview?: boolean;
  is_chat_default: boolean;
  is_chat_fallback: boolean;
  version: string;
  warning_messages?: { code: string; message: string }[];
  info_messages?: { code: string; message: string }[];
  billing?: {
    is_premium: boolean;      // Whether model is premium tier
    multiplier: number;       // Cost multiplier (0 = free, 1 = standard, 3 = expensive)
    restricted_to?: string[]; // SKU restrictions (e.g., ["pro", "pro_plus", "business", "enterprise"])
  };
  capabilities: IChatModelCapabilities | ICompletionModelCapabilities | IEmbeddingModelCapabilities;
  supported_endpoints?: ModelSupportedEndpoint[];
  custom_model?: { key_name: string; owner_name: string };
}
```

The `billing` field is optional — absent billing metadata is treated as non-premium with no cost.

## ChatEndpoint Extraction

From `src/platform/endpoint/node/chatEndpoint.ts`:

```typescript
class ChatEndpoint implements IChatEndpoint {
  public readonly isPremium?: boolean | undefined;
  public readonly multiplier?: number | undefined;
  public readonly restrictedToSkus?: string[] | undefined;

  constructor(public readonly modelMetadata: IChatModelInformation, ...) {
    this.isPremium = modelMetadata.billing?.is_premium;
    this.multiplier = modelMetadata.billing?.multiplier;
    this.restrictedToSkus = modelMetadata.billing?.restricted_to;
  }
}
```

The endpoint propagates billing fields from the API response into the runtime model representation.

## SDK Type Definitions

From copilot-sdk `src/types.ts`:

```typescript
interface ModelBilling {
  multiplier: number;  // Billing cost multiplier relative to the base rate
}

interface ModelInfo {
  id: string;
  name: string;
  capabilities: ModelCapabilities;
  policy?: ModelPolicy;
  billing?: ModelBilling;
  supportedReasoningEfforts?: ReasoningEffort[];
  defaultReasoningEffort?: ReasoningEffort;
}
```

The SDK exposes only the `multiplier` — the `is_premium` and `restricted_to` fields are internal to the VS Code extension layer.

> These types are from the external copilot-sdk and may have evolved. Verify against the SDK's current source.

## Current Model Multiplier Table

> Multiplier values are server-controlled and change over time. The table below is a point-in-time snapshot; always verify against the live `models.list` RPC response.

| Model | is_premium | Multiplier | Restricted To | Notes |
|-------|-----------|------------|---------------|-------|
| GPT-5 mini | false | 0 | — | Included (free on paid plans) |
| GPT-4.1 | false | 0 | — | Included (free on paid plans) |
| GPT-4o | false | 0 | — | Included (free on paid plans) |
| Claude Haiku 4.5 | true | 0.33 | — | ~3 prompts per premium request |
| Claude Sonnet 4 | true | 1 | — | Standard premium rate |
| Claude Sonnet 4.5 | true | 1 | — | Standard premium rate |
| Claude Sonnet 4.6 | true | 1 | — | Standard premium rate (subject to change) |
| GPT-5 | true | 1 | — | Standard premium rate |
| GPT-5.1 | true | 1 | — | Standard premium rate |
| Claude Opus 4.5 | true | 3 | pro, pro_plus, business, enterprise | 1 prompt = 3 premium requests |
| Claude Opus 4.6 | true | 3 | pro, pro_plus, business, enterprise | 1 prompt = 3 premium requests |
| Claude Opus 4.6 (fast) | true | 30 | pro, pro_plus, business, enterprise | Very expensive fast mode |

These multipliers are server-side and change over time. Always use the `models.list` RPC or endpoint provider to obtain current values.

## Auto Model Discount

From `calculateAutoModelInfo` in `src/platform/endpoint/node/autoChatEndpoint.ts` — a standalone module-level helper function, not a method on `AutoChatEndpoint`:

```typescript
// Auto model selection applies a discount to the multiplier
const newMultiplier = Math.round(
  (endpoint.multiplier ?? 0) * (1 - discountPercent) * 100
) / 100;

// Reconstructed billing metadata with discount applied
billing: {
  is_premium: originalModelInfo.billing?.is_premium ?? false,
  multiplier: newMultiplier,
  restricted_to: originalModelInfo.billing?.restricted_to
}
```

- Discount is server-controlled per model via the auto-mode session token (`token.discounted_costs?.[selectedModel.model] || 0`). When absent, the fallback is **0%** (no discount). Observed values are typically ~10% for premium models, but this is not hardcoded.
- Example: Sonnet 4 (1x) becomes Auto Sonnet 4 (0.9x) when server provides a 10% discount.
- Discount is **not** available on the Free plan (server-side invariant — the server omits discount values for Free plan users).

The rounding to two decimal places prevents floating-point drift in billing calculations.

## Extension-Contributed Endpoints

From `src/platform/endpoint/vscode-node/extChatEndpoint.ts`:

```typescript
class ExtensionContributedChatEndpoint implements IChatEndpoint {
  public readonly isPremium: boolean = false;    // Never premium
  public readonly multiplier: number = 0;        // No multiplier (free)
  public readonly isExtensionContributed = true;
}
```

Third-party VS Code language models contributed via the extension API are always free (0x multiplier). They bypass GitHub's billing entirely.

## UI Display

From `src/extension/conversation/vscode-node/languageModelAccess.ts`:

- The model picker categorizes models into: **Premium Models** (`isPremium=true`), **Standard Models** (`isPremium=false`), **Custom Models** (endpoints with `custom_model` set), and **Copilot Models** (shown for Free plan users or when `isPremium` is `undefined`).
- Tooltip text: `"Rate is counted at {multiplier}x."`
- Models with `multiplier=0` are free and do not trigger premium category labels.

## Free Plan vs Paid Plan Billing

| Aspect | Free Plan | Paid Plan |
|--------|-----------|------------|
| Cost per interaction | All models cost 1 premium request | Varies by multiplier |
| Included models | None free; all count against cap | GPT-5 mini, GPT-4.1, GPT-4o (0x) |
| Monthly cap | 50 premium requests | Plan-dependent |
| Quota header | `x-quota-snapshot-chat` | `x-quota-snapshot-premium_models` (fallback: `x-quota-snapshot-premium_interactions`) |
| Auto discount | Not available | Server-provided per model (~10% typical) |

Free plan users see no multiplier differentiation — every model interaction costs exactly one premium request regardless of the underlying model's multiplier value.

## Key Implementation Files

| File | Role |
|------|------|
| `src/platform/endpoint/common/endpointProvider.ts` | `IModelAPIResponse` type definition |
| `src/platform/endpoint/node/chatEndpoint.ts` | Billing field extraction from API response |
| `src/platform/endpoint/node/autoChatEndpoint.ts` | Auto model discount logic |
| `src/platform/endpoint/vscode-node/extChatEndpoint.ts` | Extension-contributed endpoint (always free) |
| `src/extension/conversation/vscode-node/languageModelAccess.ts` | UI display and model picker categorization |
| copilot-sdk `src/types.ts` | `ModelBilling` / `ModelInfo` type definitions |

For how multipliers interact with each consumption source, see [05-consumption-sources.md](05-consumption-sources.md).
