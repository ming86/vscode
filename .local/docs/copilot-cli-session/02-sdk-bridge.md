# SDK Bridge Service

## Overview

The `ICopilotCLISDK` service provides a lazily-initialized bridge to the `@github/copilot/sdk` package. It handles dynamic import, authentication, native shim setup, and logging integration. All session operations flow through this service to obtain the SDK module.

**Source:** `src/extension/chatSessions/copilotcli/node/copilotCli.ts`

## Service Identifiers

```typescript
export const ICopilotCLISDK = createServiceIdentifier<ICopilotCLISDK>('ICopilotCLISDK');
export const ICopilotCLIModels = createServiceIdentifier<ICopilotCLIModels>('ICopilotCLIModels');
export const ICopilotCLIAgents = createServiceIdentifier<ICopilotCLIAgents>('ICopilotCLIAgents');
```

## ICopilotCLISDK Interface

```typescript
export interface ICopilotCLISDK {
    readonly _serviceBrand: undefined;

    // Lazily imports and returns the @github/copilot/sdk module
    getPackage(): Promise<typeof import('@github/copilot/sdk')>;

    // Returns authentication credentials for SDK session creation
    getAuthInfo(): Promise<NonNullable<SessionOptions['authInfo']>>;

    // @deprecated — Maps SDK request IDs to VS Code request details
    getRequestId(sdkRequestId: string): RequestDetails['details'] | undefined;
}
```

## CopilotCLISDK Implementation

### Constructor Dependencies

```typescript
export class CopilotCLISDK implements ICopilotCLISDK {
    constructor(
        @IVSCodeExtensionContext extensionContext,
        @IEnvService envService,
        @ILogService logService,
        @IInstantiationService instantiationService,
        @IAuthenticationService authentService,
        @IConfigurationService configurationService,
    ) { }
}
```

### Lazy Initialization

Logger initialization is eager — triggered in the constructor via a `Lazy<Promise<void>>` wrapper, not deferred to `getPackage()`:

```typescript
// In constructor:
private _initializeLogger = new Lazy<Promise<void>>(() => this.initLogger());
// ...
this._ensureShimsPromise = this.ensureShims();
this._initializeLogger.value.catch((error) => { /* logged */ });
```

### SDK Import Flow

```typescript
public async getPackage(): Promise<typeof import('@github/copilot/sdk')> {
    // 1. Await the shim promise (started eagerly in constructor)
    await this._ensureShimsPromise;

    // 2. Dynamic import of @github/copilot/sdk
    return await import('@github/copilot/sdk');
}
```

### Native Shim Setup

The SDK requires `node-pty` (terminal emulation) and `ripgrep` (search). These are provided by VS Code's bundled natives. The `ensureShims()` method copies them into the SDK's expected location:

```typescript
private async ensureShims(): Promise<void> {
    // Copies node-pty and ripgrep shims to:
    // extension's node_modules/@github/copilot/
    // This allows the SDK to use VS Code's bundled native binaries
}
```

### Authentication

```typescript
public async getAuthInfo(): Promise<NonNullable<SessionOptions['authInfo']>> {
    // Two authentication modes:
    //
    // 1. Standard token auth:
    //    Returns { token: string } from IAuthenticationService
    //
    // 2. HMAC proxy auth (when ConfigKey.Shared.DebugOverrideProxyUrl is set):
    //    Returns HMAC credentials for proxy-mode operation
}
```

The returned `authInfo` is passed directly to `SessionOptions` when creating SDK sessions.

## ICopilotCLIModels Interface

Manages model discovery and selection for CLI sessions.

```typescript
export interface ICopilotCLIModels {
    readonly _serviceBrand: undefined;

    // Resolve model ID by name (case-insensitive)
    resolveModel(modelId: string): Promise<string | undefined>;

    // Get the user's preferred default model
    getDefaultModel(): Promise<string | undefined>;

    // Persist model preference
    setDefaultModel(modelId: string | undefined): Promise<void>;

    // List available models with metadata
    getModels(): Promise<CopilotCLIModelInfo[]>;

    // Register as VS Code language model provider
    registerLanguageModelChatProvider(lm: typeof vscode['lm']): void;
}

export interface CopilotCLIModelInfo {
    readonly id: string;
    readonly name: string;
    readonly multiplier?: number;
    readonly maxInputTokens?: number;
    readonly maxOutputTokens?: number;
    readonly maxContextWindowTokens: number;
    readonly supportsVision?: boolean;
}
```

### Model Resolution

- Models cached on first access (cannot change during session)
- Default model stored in `globalState` under `COPILOT_CLI_MODEL_MEMENTO_KEY`
- First model in SDK list is the platform default
- Resolution is case-insensitive by ID or name

### Language Model Provider

```typescript
public registerLanguageModelChatProvider(lm: typeof vscode['lm']): void {
    // Registers with VS Code LM API as provider 'copilotcli'
    // Exposes models in VS Code's model picker
    // provideLanguageModelChatResponse is a void no-op — models are only usable via the agent loop, not direct LM calls
}
```

## ICopilotCLIAgents Interface

Discovers and manages custom agents for CLI sessions.

```typescript
export interface CLIAgentInfo {
    readonly agent: Readonly<SweCustomAgent>;
    // File URI for prompt-file agents, synthetic copilotcli: URI for SDK agents
    readonly sourceUri: URI;
}

export interface ICopilotCLIAgents {
    readonly _serviceBrand: undefined;
    readonly onDidChangeAgents: Event<void>;

    // Resolve agent by ID, name, or displayName (case-insensitive)
    resolveAgent(agentId: string): Promise<SweCustomAgent | undefined>;

    // List all available agents (SDK + prompt-file)
    getAgents(): Promise<readonly CLIAgentInfo[]>;

    // Get the agent used for a specific session
    getSessionAgent(sessionId: string): Promise<string | undefined>;
}
```

### Agent Sources

Agents are merged from two sources:

1. **SDK Agents** — Discovered via `getCustomAgents()` with workspace folder context
   - URI scheme: `copilotcli://agents/{name}`

2. **Prompt-File Agents** — Discovered from `.github/agents/`, `.vscode/agents/`, etc.
   - URI: actual file path
   - Converted from `ParsedPromptFile` to `SweCustomAgent`

### Per-Session Agent Tracking

```typescript
async trackSessionAgent(sessionId: string, agent: string | undefined): Promise<void> {
    // Stores {agentId?, createdDateTime} per session in workspaceState
    // Key: COPILOT_CLI_SESSION_AGENTS_MEMENTO_KEY
    // Prunes entries older than 7 days automatically
}
```

## SDK Package Dependencies

From `package.json`:

```json
{
    "dependencies": {
        "@github/copilot": "^1.0.12"
    }
}
```

The `@github/copilot/sdk` is accessed via the `/sdk` subpath export of the `@github/copilot` package.

### Key SDK Imports

```typescript
import type { SessionOptions, SweCustomAgent } from '@github/copilot/sdk';
import type { Session, SessionEvent } from '@github/copilot/sdk';
import type { LocalSessionMetadata } from '@github/copilot/sdk';
import { internal } from '@github/copilot/sdk';
// internal.LocalSessionManager — native session persistence
// internal.NoopTelemetryService — telemetry stub
```
