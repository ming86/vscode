# Models and Agents

## Overview

The `ICopilotCLIModels` and `ICopilotCLIAgents` services manage model discovery/selection and custom agent resolution for CLI sessions. Both are defined alongside the SDK bridge in `copilotCli.ts` and interact with the SDK's model/agent APIs.

**Source:** `src/extension/chatSessions/copilotcli/node/copilotCli.ts`

## ICopilotCLIModels

### Interface

```typescript
export const ICopilotCLIModels =
    createServiceIdentifier<ICopilotCLIModels>('ICopilotCLIModels');

export interface ICopilotCLIModels {
    readonly _serviceBrand: undefined;

    resolveModel(modelId: string): Promise<string | undefined>;
    getDefaultModel(): Promise<string | undefined>;
    setDefaultModel(modelId: string | undefined): Promise<void>;
    getModels(): Promise<CopilotCLIModelInfo[]>;
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

```
resolveModel(modelId)
    │
    ├── Get all models from SDK
    ├── Match by ID (case-insensitive)
    ├── Match by name (case-insensitive)
    └── Return resolved model ID or undefined
```

### Default Model Persistence

```typescript
const COPILOT_CLI_MODEL_MEMENTO_KEY = 'github.copilot.cli.sessionModel';

async getDefaultModel(): Promise<string | undefined> {
    // 1. Get available models; first element is always the SDK-designated default
    const models = await this.getModels();
    if (!models.length) { return; }
    const defaultModel = models[0];

    // 2. Read the user's preferred model from the memento (falling back to the default)
    const preferredModelId = this.extensionContext.globalState
        .get<string>(COPILOT_CLI_MODEL_MEMENTO_KEY, defaultModel.id)
        ?.trim()?.toLowerCase();

    // 3. Validate the preferred model still exists in the available set;
    //    fall back to the default if it has been removed upstream
    return models.find(m => m.id.toLowerCase() === preferredModelId)?.id
        ?? defaultModel.id;
}

async setDefaultModel(modelId: string | undefined): Promise<void> {
    await this.extensionContext.globalState.update(
        COPILOT_CLI_MODEL_MEMENTO_KEY, modelId
    );
}
```

### Language Model Provider Registration

```typescript
registerLanguageModelChatProvider(lm: typeof vscode['lm']): void {
    // Registers as VS Code LM provider with family 'copilotcli'
    // Makes SDK models visible in VS Code's model picker
    //
    // IMPORTANT: provideLanguageModelChatResponse() is an empty async
    // function (void no-op) — these models are only usable through
    // the agent loop, not via direct LM API calls
}
```

This allows the VS Code model picker to show CLI-available models without routing through the LM API.

## ICopilotCLIAgents

### Interface

```typescript
export const ICopilotCLIAgents =
    createServiceIdentifier<ICopilotCLIAgents>('ICopilotCLIAgents');

export interface CLIAgentInfo {
    readonly agent: Readonly<SweCustomAgent>;
    readonly sourceUri: URI;
}

export interface ICopilotCLIAgents {
    readonly _serviceBrand: undefined;
    readonly onDidChangeAgents: Event<void>;

    resolveAgent(agentId: string): Promise<SweCustomAgent | undefined>;
    getAgents(): Promise<readonly CLIAgentInfo[]>;
    getSessionAgent(sessionId: string): Promise<string | undefined>;
}
```

### Agent Sources

Agents are discovered from two sources and merged:

```
getAgents()
    │
    ├── SDK agents (via getCustomAgents())
    │   ├── Queried with workspace folder context
    │   └── URI: copilotcli://agents/{name}
    │
    └── Prompt-file agents (via IChatPromptFileService)
        ├── From .github/agents/*.md
        ├── From .vscode/agents/*.md
        ├── From custom locations
        └── URI: actual file path
```

### Agent Resolution

```typescript
async resolveAgent(agentId: string): Promise<SweCustomAgent | undefined> {
    // Pass 1: match prompt-file agents by URI (exact match)
    for (const promptFile of this.chatPromptFileService.customAgentPromptFiles) {
        if (agentId === promptFile.uri.toString()) {
            return this.toCustomAgent(promptFile)?.agent;
        }
    }

    // Pass 2: match all agents (SDK + prompt-file) by name or displayName
    const customAgents = await this.getAgents();
    agentId = agentId.toLowerCase();
    const match = customAgents.find(a =>
        a.agent.name.toLowerCase() === agentId ||
        a.agent.displayName?.toLowerCase() === agentId
    );
    return match ? this.cloneAgent(match.agent) : undefined;
}
```

### Prompt-File to SweCustomAgent Conversion

```typescript
// ParsedPromptFile → CLIAgentInfo (via toCustomAgent)
function toCustomAgent(promptFile: ParsedPromptFile): CLIAgentInfo | undefined {
    const agentName = getAgentFileNameFromFilePath(promptFile.uri);
    const headerName = promptFile.header?.name?.trim();
    const name = headerName === undefined || headerName === ''
        ? agentName : headerName;
    if (!name) { return undefined; }

    const tools = promptFile.header?.tools?.filter(tool => !!tool) ?? [];
    const model = promptFile.header?.model?.[0];

    return {
        agent: {
            name,
            displayName: name,
            description: promptFile.header?.description ?? '',
            tools: tools.length > 0 ? tools : null,
            prompt: async () => promptFile.body?.getContent() ?? '',  // Note: function, not string
            disableModelInvocation: promptFile.header?.disableModelInvocation ?? false,
            ...(model ? { model } : {}),  // Conditionally included
        },
        sourceUri: promptFile.uri,
    };
}
```

### Per-Session Agent Tracking

```typescript
const COPILOT_CLI_SESSION_AGENTS_MEMENTO_KEY = 'github.copilot.cli.sessionAgents';

/**
 * @deprecated Use empty strings to represent default model/agent instead.
 * Left for backward compatibility with state stored by older versions.
 */
export const COPILOT_CLI_DEFAULT_AGENT_ID = '___vscode_default___';

interface SessionAgentEntry {
    agentId?: string;
    createdDateTime: number;
}

// In-memory cache: `private sessionAgents: Record<string, SessionAgentEntry> = {};`

async trackSessionAgent(sessionId: string, agent: string | undefined) {
    // Read from in-memory cache first; fall back to memento
    const details = Object.keys(this.sessionAgents).length
        ? this.sessionAgents
        : this.extensionContext.workspaceState.get<Record<string, SessionAgentEntry>>(
              COPILOT_CLI_SESSION_AGENTS_MEMENTO_KEY, this.sessionAgents
          );

    details[sessionId] = { agentId: agent, createdDateTime: Date.now() };

    // Write to in-memory cache
    this.sessionAgents = details;

    // Prune entries older than 7 days
    const cutoff = Date.now() - (7 * 24 * 60 * 60 * 1000);
    for (const [id, entry] of Object.entries(details)) {
        if (entry.createdDateTime < cutoff) {
            delete details[id];
        }
    }

    // Persist to memento
    await this.extensionContext.workspaceState.update(
        COPILOT_CLI_SESSION_AGENTS_MEMENTO_KEY, details
    );
}

async getSessionAgent(sessionId: string): Promise<string | undefined> {
    const details = this.extensionContext.workspaceState.get<Record<string, SessionAgentEntry>>(
        COPILOT_CLI_SESSION_AGENTS_MEMENTO_KEY, this.sessionAgents
    );
    // In-memory cache takes precedence (may not be persisted yet)
    const agentId = this.sessionAgents[sessionId]?.agentId
        ?? details[sessionId]?.agentId;
    if (agentId === COPILOT_CLI_DEFAULT_AGENT_ID) {
        return '';  // Backward-compat: deprecated sentinel → empty string
    }
    // Two additional defensive branches (largely unreachable in practice):
    // 1. `typeof agentId === 'string'` guard — returns agentId directly
    // 2. Fallback: queries getAgents() and matches by name (handles
    //    edge case where agentId is defined but not a string)
    if (typeof agentId === 'string') {
        return agentId;
    }
    const agents = await this.getAgents();
    return agents.find(a => a.agent.name.toLowerCase() === agentId)?.agent.name;
}
```

## Session Options Integration

When creating a session, models and agents are wired into `SessionOptions`:

```typescript
// In CopilotCLISessionService.createSessionsOptions():

const [agentInfos, skillLocations] = await Promise.all([
    this.agents.getAgents(),
    this.copilotCLISkills.getSkillsLocations(),
]);

const allOptions: SessionOptions = {
    model: options.model,                        // From ICopilotCLIModels
    selectedCustomAgent: options.agent,          // Selected agent
    customAgents: agentInfos.map(i => i.agent),  // All available agents
    skillDirectories: skillLocations.map(u => u.fsPath),
    // ...
};
```

## Agent Mode Instructions

When restoring session history, agent context is preserved:

```typescript
// In CopilotCLISessionService.getChatHistoryImpl():

const agentId = await this._chatSessionMetadataStore.getSessionAgent(sessionId);
const defaultModeInstructions = agentId
    ? this.resolveAgentModeInstructions(agentId, customAgentLookup)
    : undefined;

// Attached to each request turn during history reconstruction
```

### StoredModeInstructions

```typescript
export interface StoredModeInstructions {
    readonly uri?: string;                                        // Agent definition file URI (optional)
    readonly name: string;                                        // Agent display name
    readonly content: string;                                     // Agent instruction body
    readonly metadata?: Record<string, boolean | string | number>; // Arbitrary agent metadata
    readonly isBuiltin?: boolean;                                 // True for built-in agents
}
```

## Custom Agent Selection Flow

```
User selects agent in UI
    │
    ├── New session:
    │   ├── SessionOptions.selectedCustomAgent = agent
    │   └── SessionOptions.customAgents = allAgents
    │
    ├── Existing session:
    │   ├── session.sdkSession.selectCustomAgent(agent.name)
    │   └── (or session.sdkSession.clearCustomAgent())
    │
    └── agents.trackSessionAgent(sessionId, agent.name)
```

## Skills Integration

```typescript
export interface ICopilotCLISkills {
    readonly _serviceBrand: undefined;
    getSkillsLocations(): Uri[];  // Synchronous — returns Uri[], not Promise
}
```

Skills are additional agent capabilities loaded from directories. They are passed to the SDK as `skillDirectories` in `SessionOptions`, allowing the SDK to discover and register skill-based tools.

## ICopilotCLISDK Service

The `ICopilotCLISDK` service abstracts dynamic import of the Copilot CLI SDK package and centralizes authentication. It is architecturally central — injected into both `CopilotCLIModels` and `CopilotCLIAgents`.

```typescript
export interface ICopilotCLISDK {
    readonly _serviceBrand: undefined;
    getPackage(): Promise<typeof import('@github/copilot/sdk')>;
    getAuthInfo(): Promise<NonNullable<SessionOptions['authInfo']>>;
    /** @deprecated */
    getRequestId(sdkRequestId: string): RequestDetails['details'] | undefined;
}
```

See [doc 02](./02-sdk-bridge.md) for implementation details including shim initialization and logger wiring.
