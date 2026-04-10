# Anthropic Native Memory Tool Integration

The Copilot Chat extension implements a parallel memory subsystem for BYOK Anthropic providers that replaces the client-side file-based `copilot_memory` tool with Anthropic's server-managed `memory_20250818` protocol. When enabled, the `AnthropicLMProvider` intercepts the standard `memory` tool declaration and substitutes it with a native tool type, delegating memory persistence entirely to the Anthropic Messages API. The server manages memory state transparently — no special response block types appear in the stream handler, and no client-side storage is involved.

## Model Capability Check

The `modelSupportsMemory()` function in `src/platform/networking/common/anthropic.ts` gates native memory support by model identifier. The function normalizes the model ID (lowercase, dots replaced with hyphens) and matches against a whitelist of Claude 4+ model families:

```typescript
export function modelSupportsMemory(modelId: string): boolean {
	const normalized = modelId.toLowerCase().replace(/\./g, '-');
	return normalized.startsWith('claude-haiku-4-5') ||
		normalized.startsWith('claude-sonnet-4-6') ||
		normalized.startsWith('claude-sonnet-4-5') ||
		normalized.startsWith('claude-sonnet-4') ||
		normalized.startsWith('claude-opus-4-6') ||
		normalized.startsWith('claude-opus-4-5') ||
		normalized.startsWith('claude-opus-4-1') ||
		normalized.startsWith('claude-opus-4');
}
```

The normalization step (`replace(/\./g, '-')`) ensures that variant identifiers such as `claude-sonnet-4.5` and `claude-sonnet-4-5` resolve identically. The `startsWith` matching permits version suffixes (e.g., `claude-sonnet-4-5-20250620`) without maintaining an exhaustive list.

## Feature Gating

`isAnthropicMemoryToolEnabled()` composes the model capability check with an experiment-based configuration flag:

```typescript
export function isAnthropicMemoryToolEnabled(
	endpoint: IChatEndpoint | string,
	configurationService: IConfigurationService,
	experimentationService: IExperimentationService,
): boolean {
	const effectiveModelId = typeof endpoint === 'string' ? endpoint : endpoint.model;
	if (!modelSupportsMemory(effectiveModelId)) {
		return false;
	}
	return configurationService.getExperimentBasedConfig(ConfigKey.MemoryToolEnabled, experimentationService);
}
```

The configuration key is defined in `src/platform/configuration/common/configurationService.ts`:

```typescript
export const MemoryToolEnabled = defineSetting<boolean>(
	'chat.tools.memory.enabled', ConfigType.ExperimentBased, true);
```

Two conditions must hold for native memory activation:

| Condition | Source | Effect if false |
|-----------|--------|-----------------|
| Model is in the `modelSupportsMemory` whitelist | `anthropic.ts` | Returns `false` immediately |
| `chat.tools.memory.enabled` resolves to `true` via experiment service | `configurationService.ts` | Returns `false` |

The same `MemoryToolEnabled` config key gates both the native Anthropic memory and the file-based `copilot_memory` tool.

## Tool Registration

Within `AnthropicLMProvider` (`src/extension/byok/vscode-node/anthropicProvider.ts`), the tool registration loop intercepts any tool named `'memory'` and, when native memory is enabled, replaces it with the Anthropic-native tool type:

```typescript
const memoryToolEnabled = isAnthropicMemoryToolEnabled(
	model.id, this._configurationService, this._experimentationService);

// ...

let hasMemoryTool = false;
for (const tool of (options.tools ?? [])) {
	if (tool.name === 'memory' && memoryToolEnabled) {
		hasMemoryTool = true;
		tools.push({
			name: 'memory',
			type: 'memory_20250818'
		} as Anthropic.Beta.BetaMemoryTool20250818);
		continue;
	}
	// ... standard tool registration ...
}
```

Key mechanics:

- The interception matches on tool name `'memory'`, not `'copilot_memory'`. These are distinct tool identifiers.
- When intercepted, the tool declaration is replaced with `type: 'memory_20250818'` — an Anthropic-specific server tool type.
- The `continue` skips standard tool serialization for this entry, preventing a duplicate declaration.
- If `memoryToolEnabled` is `false`, the `memory` tool falls through to standard registration.

## Beta Protocol

The `context-management-2025-06-27` beta flag is appended to the request when native memory (or context management) is active:

```typescript
if (hasMemoryTool || contextManagement) {
	betas.push('context-management-2025-06-27');
}
```

This beta header is required by the Anthropic Messages API to enable the `memory_20250818` tool type. Without it, the server rejects the tool declaration. The flag is shared with the context management feature — either capability triggers its inclusion.

## Relationship to File-Based Memory

The native Anthropic memory and the file-based `copilot_memory` tool are **not mutually exclusive**. They operate as parallel subsystems with different scopes and activation conditions:

| Dimension | Anthropic Native Memory | File-Based `copilot_memory` |
|-----------|------------------------|-----------------------------|
| **Provider scope** | BYOK Anthropic (`AnthropicLMProvider`) only | All providers (GitHub-hosted, BYOK OpenAI, BYOK Anthropic, etc.) |
| **Tool name** | `memory` | `copilot_memory` |
| **Tool type** | `memory_20250818` (Anthropic server tool) | Standard function tool (client-side) |
| **Storage** | Server-managed by Anthropic | Client-side filesystem |
| **Protocol** | Anthropic Messages API beta | VS Code Language Model Tool API |
| **Stream handling** | Transparent — no special response blocks | Standard tool-use response blocks |
| **Config gate** | `chat.tools.memory.enabled` + model whitelist | `chat.tools.memory.enabled` |
| **Activation** | Model in whitelist AND config enabled AND BYOK Anthropic provider | Config enabled (any provider) |

Both systems share the `MemoryToolEnabled` configuration gate, but native memory additionally requires a whitelisted model and the BYOK Anthropic provider path.

## Prompt Layer Invariance

The prompt layer — specifically `MemoryContextPrompt` (`memoryContextPrompt.tsx`) — contains **zero** Anthropic-specific code. Memory context injection into the system prompt is identical regardless of whether the underlying provider uses native Anthropic memory or file-based storage. The prompt component is unaware of which memory subsystem is active; it operates purely on the memory content surface, not the storage mechanism.

This separation is deliberate: the native memory integration is confined entirely to the provider layer (`AnthropicLMProvider`), while prompt construction remains provider-agnostic.

## Source File Reference

| File | Path | Key Elements |
|------|------|-------------|
| anthropic.ts | `src/platform/networking/common/anthropic.ts` | `modelSupportsMemory()` (~L233–243), `isAnthropicMemoryToolEnabled()` (~L288–298) |
| anthropicProvider.ts | `src/extension/byok/vscode-node/anthropicProvider.ts` | Tool interception (~L142–170), beta flag (~L260–263) |
| configurationService.ts | `src/platform/configuration/common/configurationService.ts` | `MemoryToolEnabled` config key definition |

## Replication Notes

1. **Model whitelist is prefix-based.** Adding support for a new Claude model family requires a single `startsWith` clause in `modelSupportsMemory()`. The dot-to-hyphen normalization must be preserved.

2. **Tool name interception is `'memory'`, not `'copilot_memory'`.** These are distinct tools. The native path intercepts only `'memory'`; the file-based tool uses `'copilot_memory'` and is never intercepted.

3. **The beta flag `context-management-2025-06-27` is mandatory.** The Anthropic API rejects `memory_20250818` tool types without this header. It is shared with the context management feature.

4. **The two memory systems can coexist.** A BYOK Anthropic request with a whitelisted model may carry both a native `memory` tool (type `memory_20250818`) and a standard `copilot_memory` function tool in the same request.

5. **No client-side stream handling is required for native memory.** The Anthropic server manages memory state entirely. No special content block types (beyond standard tool use) appear in the response stream.

6. **Configuration is experiment-gated.** `MemoryToolEnabled` uses `ConfigType.ExperimentBased` with a default of `true`, meaning the experimentation service can override the local setting.

7. **Provider isolation is strict.** Only `AnthropicLMProvider` implements native memory interception. Other BYOK providers (OpenAI, etc.) and the GitHub-hosted path are unaffected.
