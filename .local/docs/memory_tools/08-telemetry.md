# Telemetry Events

## Overview

The memory system emits three telemetry events to track tool invocations and prompt context reads. All events use `sendMSFTTelemetryEvent()`.

## Events

### `memoryToolInvoked`

Emitted by `MemoryTool._sendLocalTelemetry()` on every local (non-CAPI) tool invocation.

| Property | Type | Description |
|----------|------|-------------|
| `command` | string | The operation: `view`, `create`, `str_replace`, `insert`, `delete`, `rename` |
| `scope` | string | Memory scope: `user`, `session`, `repo` |
| `toolOutcome` | string | Normalized outcome: `success`, `error`, `notFound`, `notEnabled` |
| `requestId` | string | The chat request turn ID |
| `model` | string | The model ID that invoked the tool |

```typescript
this.telemetryService.sendMSFTTelemetryEvent('memoryToolInvoked', {
    command,
    scope,
    toolOutcome,
    requestId,
    model: model?.id,
});
```

### `memoryRepoToolInvoked`

Emitted by `MemoryTool._sendRepoTelemetry()` on every CAPI repository memory invocation.

| Property | Type | Description |
|----------|------|-------------|
| `command` | string | The operation attempted |
| `toolOutcome` | string | Normalized outcome: `success`, `error`, `notFound`, `notEnabled` |
| `requestId` | string | The chat request turn ID |
| `model` | string | The model ID |

```typescript
this.telemetryService.sendMSFTTelemetryEvent('memoryRepoToolInvoked', {
    command,
    toolOutcome,
    requestId,
    model: model?.id,
});
```

### `memoryContextRead`

Emitted by `MemoryContextPrompt._sendContextReadTelemetry()` during prompt construction (first turn).

| Property | Type | Classification | Description |
|----------|------|---------------|-------------|
| `hasUserMemory` | string | Dimension | Whether user memory content was loaded (`"true"` / `"false"`) |
| `userMemoryLength` | number | Measurement | String length of user memory content |
| `sessionFileCount` | number | Measurement | Number of session memory files listed |
| `sessionMemoryLength` | number | Measurement | String length of session memory file listing |
| `repoMemoryCount` | number | Measurement | Number of CAPI repository memories fetched |
| `repoMemoryLength` | number | Measurement | String length of formatted repository memories |

```typescript
this.telemetryService.sendMSFTTelemetryEvent('memoryContextRead', {
    hasUserMemory: String(hasUserMemory),
}, {
    userMemoryLength,
    sessionFileCount,
    sessionMemoryLength,
    repoMemoryCount,
    repoMemoryLength,
});
```

## GDPR Annotations

All events include `__GDPR__` comment blocks for compliance tracking:

```typescript
/* __GDPR__
    "memoryToolInvoked" : {
        "owner": "digitarald",
        "comment": "Tracks memory tool invocations for local user, session, and repo scopes",
        "command": { "classification": "SystemMetaData", "purpose": "FeatureInsight" },
        "scope": { "classification": "SystemMetaData", "purpose": "FeatureInsight" },
        "toolOutcome": { "classification": "SystemMetaData", "purpose": "FeatureInsight" },
        "requestId": { "classification": "SystemMetaData", "purpose": "FeatureInsight" },
        "model": { "classification": "SystemMetaData", "purpose": "FeatureInsight" }
    }
*/
```

All properties are classified as `SystemMetaData` with purpose `FeatureInsight`. No user-identifiable or content data is tracked — only metadata about tool usage patterns.
