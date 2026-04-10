# MCP Integration

## Overview

The MCP (Model Context Protocol) integration enables Copilot CLI sessions to use VS Code's registered MCP servers. This is achieved through two mechanisms: a **gateway proxy** that bridges VS Code's MCP infrastructure to the SDK, and a **tool name remapping** system that translates between agent-friendly names and gateway identifiers.

**Source:** `src/extension/chatSessions/copilotcli/node/mcpHandler.ts`

## Service Interface

```typescript
export const ICopilotCLIMCPHandler =
    createServiceIdentifier<ICopilotCLIMCPHandler>('ICopilotCLIMCPHandler');

export interface ICopilotCLIMCPHandler {
    readonly _serviceBrand: undefined;
    loadMcpConfig(): Promise<{
        mcpConfig: Record<string, MCPServerConfig> | undefined;
        disposable: IDisposable;
    }>;
}
```

## Types

```typescript
// User-facing display label of an MCP server (from VS Code settings)
export type MCPDisplayName = string;

// Short server name as used in agent definition files
export type MCPServerName = string;

// Mapping from friendly MCP server names → VS Code MCP server display labels
export type McpServerMappings = Map<MCPServerName, MCPDisplayName>;

// MCP server config passed to SDK Session
export type MCPServerConfig = NonNullable<Session['mcpServers']>[string];
```

### MCPServerConfig Shape

```typescript
interface MCPServerConfig {
    type: 'http' | 'local' | 'builtin';
    url?: string;              // HTTP endpoint for gateway-proxied servers
    command?: string;          // For local servers
    args?: string[];           // For local servers
    tools?: string[];          // Tool filter — ['*'] for all
    headers?: Record<string, string>;  // Auth headers
    displayName?: string;      // Human-readable name
    isDefaultServer?: boolean; // Built-in server flag
}
```

## MCP Configuration Flow

```
loadMcpConfig()
    │
    ├── Is Sessions Window?
    │   └── YES → loadMcpConfigWithGateway()
    │
    ├── Is CLIMCPServerEnabled?
    │   └── YES → loadMcpConfigWithGateway()
    │
    └── NO → addBuiltInGitHubServer() only
```

### Gateway Mode

When enabled, all MCP servers from VS Code are proxied through a gateway:

```typescript
private async loadMcpConfigWithGateway() {
    const gateway = await this.mcpService.startMcpGateway(
        URI.from({
            scheme: 'copilot-cli',
            path: `mcp-gateway-${generateUuid()}`
        })
    );

    for (const server of gateway.servers) {
        const serverId = this.normalizeServerName(server.label)
            ?? `vscode-mcp-server-${Object.keys(mcpConfig).length}`;
        mcpConfig[serverId] = {
            type: 'http',
            url: server.address.toString(),  // Local HTTP URL
            tools: ['*'],
            displayName: server.label,
        };
    }

    return { mcpConfig, disposable: gateway };
}
```

**Key detail:** The gateway starts an HTTP server per MCP server. The SDK connects to these local HTTP endpoints instead of directly managing the MCP server processes.

### Server Name Normalization

```typescript
private normalizeServerName(originalName: string): string | undefined {
    // SDK requires tool/server names to match /[a-z0-9_-]/gi
    let normalized = originalName.toLowerCase().replace(/[^a-z0-9_-]/gi, '_');
    normalized = normalized.replace(/^_+|_+$/g, '');
    return normalized || undefined;
}
```

**Examples:**
- `"GitHub"` → `"github"`
- `"My Server (v2)"` → `"my_server__v2_"` → `"my_server__v2"`
- `"Context7"` → `"context7"`

### Built-in GitHub Server

```typescript
private async addBuiltInGitHubServer(config) {
    const githubId = this.normalizeServerName('gitHub');
    if (!githubId) {
        return;
    }

    // Conflict detection: skip if an HTTP GitHub server with auth headers exists
    if (config[githubId] && config[githubId].type === 'http') {
        if (Object.keys(config[githubId].headers || {}).length > 0) {
            return;
        }
    }

    const definitionProvider = new GitHubMcpDefinitionProvider(
        this.configurationService,
        this.authenticationService,
        this.logService
    );

    const definitions = definitionProvider.provideMcpServerDefinitions();
    const definition = definitions[0];
    const resolvedDefinition = await definitionProvider
        .resolveMcpServerDefinition(definition, {} as CancellationToken);

    config[githubId] = {
        type: 'http',
        url: resolvedDefinition.uri.toString(),
        isDefaultServer: true,
        headers: resolvedDefinition.headers,  // Contains auth token
        tools: ['*'],
        displayName: 'GitHub',
    };
}
```

## Tool Name Remapping

### Problem

Custom agent files reference MCP tools as `<friendly-server-name>/<tool-name>`:
```yaml
# .github/agents/my-agent.md
tools:
  - context7/resolve-library-id
  - github/create_issue
```

But the SDK expects `<gateway-name>/<tool-name>` where gateway names are the normalized keys in the MCP config:
```
context7/resolve-library-id  →  context7/resolve-library-id  (same)
My Server/my-tool            →  my_server/my-tool            (remapped)
```

### Building the Mapping

```typescript
export function buildMcpServerMappings(
    tools: ReadonlyMap<LanguageModelToolInformation, boolean>
): McpServerMappings {
    const mappings = new Map<string, string>();

    for (const [tool] of tools) {
        // Structural typing guard: source must have a `name` property (MCP-backed tool)
        if (!tool.source || !hasKey(tool.source, { name: true }) || !tool.fullReferenceName) {
            continue;
        }

        const slashIndex = tool.fullReferenceName.lastIndexOf('/');
        if (slashIndex > 0) {
            const serverName = tool.fullReferenceName.substring(0, slashIndex);
            if (serverName && !mappings.has(serverName) && tool.source.label) {
                mappings.set(serverName, tool.source.label);
            }
        }
    }

    return mappings;
}
```

**Input:** VS Code's `LanguageModelToolInformation` (has `fullReferenceName` like `"github/create_issue"` and `source.label` like `"GitHub"`)

**Output:** `Map<MCPServerName, MCPDisplayName>` (e.g., `"github" → "GitHub"`)

### Applying the Remapping

```typescript
export function remapCustomAgentTools(
    customAgents: SweCustomAgent[],
    mcpServerMappings: McpServerMappings,
    mcpServers: SessionOptions['mcpServers'],
    selectedAgent: SweCustomAgent | undefined,
): void {
    // Early return for empty mappings or missing server config
    if (!mcpServerMappings.size || !mcpServers) {
        return;
    }

    // Build reverse lookup: displayName → gatewayName
    const displayNameToGatewayName = new Map<string, string>();
    for (const [gatewayName, config] of Object.entries(mcpServers)) {
        if (config.displayName) {
            displayNameToGatewayName.set(config.displayName, gatewayName);
        }
    }

    const agentsToRemap = selectedAgent ? [...customAgents, selectedAgent] : customAgents;
    for (const agent of agentsToRemap) {
        if (!agent.tools?.length) {
            continue;  // Skip agents with no tools
        }
        for (let i = 0; i < agent.tools.length; i++) {
            const tool = agent.tools[i];
            const slashIndex = tool.lastIndexOf('/');
            if (slashIndex < 1) {
                continue;  // No valid server/tool separator
            }
            const serverName = tool.substring(0, slashIndex);
            const toolName = tool.substring(slashIndex + 1);
            if (!serverName || !toolName) {
                continue;  // Empty server name or tool name
            }

            // Resolve: friendlyName → displayName → gatewayName
            const displayName = mcpServerMappings.get(serverName);
            const gatewayName = displayName
                ? displayNameToGatewayName.get(displayName)
                : displayNameToGatewayName.get(serverName);

            if (gatewayName) {
                agent.tools[i] = `${gatewayName}/${toolName}`;
            }
        }
    }
}
```

### Remapping Chain

```
Agent file tool ref: "My Server/my-tool"
         │
         ├── mcpServerMappings: "My Server" → "My Server" (displayName)
         │
         ├── mcpServers config: { "my_server": { displayName: "My Server" } }
         │
         ├── displayNameToGatewayName: "My Server" → "my_server"
         │
         └── Result: "my_server/my-tool"
```

## Integration with Session Creation

```typescript
// In CopilotCLISessionService.createSessionsOptions():

if (options.mcpServerMappings?.size && customAgents && options.mcpServers) {
    remapCustomAgentTools(
        customAgents,
        options.mcpServerMappings,
        options.mcpServers,
        options.agent
    );
}

const allOptions: SessionOptions = {
    mcpServers: mcpServers,
    customAgents: customAgents,  // Already remapped
    // ...
};
```

## Configuration Flags

| Config Key | Default | Effect |
|------------|---------|--------|
| `ConfigKey.Advanced.CLIMCPServerEnabled` | `true` | Enable MCP server forwarding via gateway |
| `chat.experimentalSessionsWindowOverride` | `false` | Force gateway mode in sessions window |

## MCP Config Sources

The SDK merges MCP configuration from multiple sources:

1. **VS Code gateway** — Servers registered in VS Code (via `mcpHandler.loadMcpConfig()`)
2. **`~/.copilot/mcp-config.json`** — Global CLI MCP configuration
3. **Workspace `.mcp.json`** — Per-workspace MCP servers
4. **`.vscode/mcp.json`** — VS Code workspace MCP servers

The gateway approach (source 1) is preferred because it reuses VS Code's MCP server lifecycle management, authentication, and process supervision.
