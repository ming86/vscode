# Language Model Tools Architecture Overview

## Purpose

This document provides a comprehensive architectural overview of the languageModelTools system in the GitHub Copilot Chat extension. These tools enable the language model to interact with the VS Code workspace, perform searches, edit files, and execute various operations on behalf of the user.

## System Architecture

### Registration Flow

```
┌─────────────────────────────────────────────────────────────────┐
│ 1. REGISTRATION PHASE (Extension Activation)                    │
├─────────────────────────────────────────────────────────────────┤
│ Tool Class Definition (e.g., codebaseTool.tsx)                  │
│   - Implements ICopilotTool<T> interface                        │
│   - Defines static toolName = ToolName.Codebase                 │
│   - ToolRegistry.registerTool(CodebaseTool)                     │
│                                                                  │
│ ToolRegistry (toolsRegistry.ts)                                 │
│   - Stores tool constructors in array                           │
│                                                                  │
│ ToolsService instantiation (toolsService.ts)                    │
│   - Lazy-loads tools: ToolRegistry.getTools()                   │
│   - Creates instances via dependency injection                  │
│                                                                  │
│ ToolsContribution (tools.ts)                                    │
│   - Iterates toolsService.copilotTools                          │
│   - Calls vscode.lm.registerTool(contributedName, toolInstance) │
│   - Maps: ToolName.Codebase → "copilot_searchCodebase"          │
└─────────────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────────┐
│ 2. RUNTIME INVOCATION (Model makes tool call)                   │
├─────────────────────────────────────────────────────────────────┤
│ Language Model → "Use tool: copilot_searchCodebase"             │
│                                                                  │
│ VS Code LM API                                                   │
│   - Looks up registered handler for "copilot_searchCodebase"    │
│   - Finds CodebaseTool instance                                 │
│                                                                  │
│ Tool Instance                                                    │
│   - invoke(options, token) called                               │
│   - Executes tool-specific logic                                │
│   - Returns LanguageModelToolResult                             │
│                                                                  │
│ Result flows back to Language Model                             │
└─────────────────────────────────────────────────────────────────┘
```

## Core Components

### 1. Tool Registry System

**File**: `src/extension/tools/toolsRegistry.ts`

The `ToolRegistry` is a singleton that maintains an array of tool class constructors.

```typescript
class ToolRegistry {
    private static tools: ICopilotToolConstructor[] = [];

    static registerTool(tool: ICopilotToolConstructor): void {
        this.tools.push(tool);
    }

    static getTools(): ICopilotToolConstructor[] {
        return this.tools;
    }
}
```

**Key Interfaces**:

```typescript
interface ICopilotToolConstructor {
    new (...args: any[]): ICopilotTool<any>;
    readonly toolName: ToolName;
}

interface ICopilotTool<T> extends vscode.LanguageModelTool<T>, ICopilotToolExtension<T> {}
```

### 2. Tool Service

**File**: `src/extension/tools/toolsService.ts`

The `ToolsService` manages tool instantiation and provides access to registered tools.

```typescript
class ToolsService implements IToolsService {
    private _copilotTools = new Lazy(() =>
        new Map(ToolRegistry.getTools().map(t =>
            [t.toolName, this.instantiationService.createInstance(t)]
        ))
    );

    get copilotTools(): Map<ToolName, ICopilotTool<any>> {
        return this._copilotTools.value;
    }
}
```

**Key Features**:

- Lazy instantiation of tools
- Dependency injection via `IInstantiationService`
- Centralized access point for all tools

### 3. Tool Contribution System

**File**: `src/extension/tools/tools.ts`

The `ToolsContribution` class registers tools with VS Code's Language Model API.

```typescript
export class ToolsContribution extends Disposable {
    constructor(
        @IToolsService private readonly toolsService: IToolsService,
    ) {
        super();

        for (const [name, tool] of toolsService.copilotTools) {
            this._register(
                vscode.lm.registerTool(
                    getContributedToolName(name),
                    tool
                )
            );
        }
    }
}
```

### 4. Name Mapping System

**File**: `src/extension/tools/names.ts`

Two parallel enum systems for internal and external tool names:

```typescript
// Internal names (used in code)
enum ToolName {
    Codebase = 'semantic_search',
    WorkspaceSymbols = 'workspace_symbols',
    // ...
}

// External names (exposed to VS Code API)
enum ContributedToolName {
    Codebase = 'copilot_searchCodebase',
    WorkspaceSymbols = 'copilot_searchWorkspaceSymbols',
    // ...
}
```

Bidirectional mapping functions:

- `getContributedToolName(name: ToolName): ContributedToolName`
- `getToolName(contributedName: ContributedToolName): ToolName`

## Tool Interface Structure

### Base Interface

```typescript
interface ICopilotTool<T> extends vscode.LanguageModelTool<T>, ICopilotToolExtension<T> {
    // From vscode.LanguageModelTool
    invoke(
        options: vscode.LanguageModelToolInvocationOptions<T>,
        token: vscode.CancellationToken
    ): vscode.ProviderResult<vscode.LanguageModelToolResult>;

    // Optional methods from ICopilotToolExtension
    filterEdits?(resource: URI): Promise<IEditFilterData | undefined>;
    provideInput?(promptContext: IBuildPromptContext): Promise<T | undefined>;
    resolveInput?(input: T, promptContext: IBuildPromptContext, mode: CopilotToolMode): Promise<T>;
    alternativeDefinition?(tool: vscode.LanguageModelToolInformation, endpoint?: IChatEndpoint): vscode.LanguageModelToolInformation;
}
```

### Extension Methods

**`filterEdits`**: Pre-filters edits for specific resources
**`provideInput`**: Generates tool input from prompt context
**`resolveInput`**: Resolves/enriches input before invocation
**`alternativeDefinition`**: Provides alternative tool definitions per endpoint

## Tool Declaration in package.json

Tools are declared in the `contributes.languageModelTools` section:

```json
{
  "contributes": {
    "languageModelTools": [
      {
        "name": "copilot_searchCodebase",
        "toolReferenceName": "codebase",
        "displayName": "%copilot.tools.searchCodebase.name%",
        "icon": "$(folder)",
        "userDescription": "%copilot.codebase.tool.description%",
        "modelDescription": "Run a natural language search...",
        "tags": ["codesearch", "vscode_codesearch"],
        "inputSchema": {
          "type": "object",
          "properties": {
            "query": {
              "type": "string",
              "description": "The query to search the codebase for..."
            }
          },
          "required": ["query"]
        }
      }
    ]
  }
}
```

**Key Properties**:

- **name**: External identifier (e.g., `copilot_searchCodebase`)
- **toolReferenceName**: Short reference name for user-facing features
- **displayName**: Localized display name
- **modelDescription**: Detailed description for the language model
- **userDescription**: Description shown to users
- **inputSchema**: JSON Schema defining parameters
- **tags**: Categorization and feature flags
- **when**: Conditional activation expression
- **canBeReferencedInPrompt**: Whether users can explicitly reference the tool

## Tool Categories

### Search Tools

- `copilot_searchCodebase` - Semantic code search
- `copilot_searchWorkspaceSymbols` - Symbol search
- `copilot_listCodeUsages` - Find references/usages
- `copilot_findFiles` - File pattern search
- `copilot_findTextInFiles` - Text/grep search

### File Operation Tools

- `copilot_readFile` - Read file contents
- `copilot_listDirectory` - List directory contents
- `copilot_createFile` - Create new files
- `copilot_createDirectory` - Create directories

### Editing Tools

- `copilot_insertEdit` - Insert code edits
- `copilot_replaceString` - Replace text
- `copilot_multiReplaceString` - Multiple replacements
- `copilot_applyPatch` - Apply diff patches

### Diagnostics Tools

- `copilot_getErrors` - Get compile/lint errors
- `copilot_getChangedFiles` - Get git changes

### Notebook Tools

- `copilot_createNewJupyterNotebook` - Create notebooks
- `copilot_editNotebook` - Edit notebook cells
- `copilot_runNotebookCell` - Execute cells
- `copilot_getNotebookSummary` - Get notebook structure
- `copilot_readNotebookCellOutput` - Read cell outputs

### Workspace Tools

- `copilot_createNewWorkspace` - Initialize projects
- `copilot_getProjectSetupInfo` - Project templates
- `copilot_installExtension` - Install VS Code extensions
- `copilot_runVscodeCommand` - Execute VS Code commands

### Specialized Tools

- `copilot_getVSCodeAPI` - VS Code API documentation
- `copilot_githubRepo` - Search GitHub repositories
- `copilot_fetchWebPage` - Fetch web content
- `copilot_memory` - Persistent memory management
- `copilot_testFailure` - Test failure information
- `copilot_openSimpleBrowser` - Open URLs in editor

### Agent Tools

- `runSubagent` - Launch subagent for autonomous multi-step tasks

## Dependency Injection

Tools use VS Code's dependency injection system via `IInstantiationService`:

```typescript
export class CodebaseTool implements ICopilotTool<ICodebaseToolParams> {
    constructor(
        @IWorkspaceChunkSearchService private readonly workspaceChunkSearchService: IWorkspaceChunkSearchService,
        @IInstantiationService private readonly instantiationService: IInstantiationService,
        @ITelemetryService private readonly telemetryService: ITelemetryService,
        // ... other services
    ) {}
}
```

Common injected services:

- `IWorkspaceChunkSearchService` - Workspace search
- `IFileService` - File operations
- `ITextFileService` - Text file operations
- `IEditorService` - Editor management
- `ITelemetryService` - Telemetry/analytics
- `IAuthenticationService` - Authentication
- `IEndpointProvider` - LLM endpoint access

## Invocation Flow

1. **Model Decision**: Language model determines tool is needed
2. **Tool Call**: Model generates tool call with parameters
3. **VS Code Dispatch**: `vscode.lm` API looks up registered tool by name
4. **Tool Invocation**: Tool's `invoke()` method is called
5. **Execution**: Tool executes its logic using injected services
6. **Result Return**: Tool returns `LanguageModelToolResult`
7. **Model Processing**: Result is provided back to the model

## Result Types

```typescript
class LanguageModelToolResult {
    constructor(
        public readonly content: (vscode.LanguageModelTextPart | vscode.LanguageModelToolResultPart)[],
        public readonly metadata?: { [key: string]: any }
    ) {}
}
```

**Content Types**:

- `LanguageModelTextPart`: Plain text results
- `LanguageModelToolResultPart`: Structured tool output

**Metadata**: Additional context (counts, references, diagnostics)

## Tool Implementation Pattern

Typical tool structure:

```typescript
export class ExampleTool implements ICopilotTool<IExampleToolParams> {
    public static readonly toolName = ToolName.Example;

    constructor(
        @IExampleService private readonly exampleService: IExampleService,
        // ... other dependencies
    ) {}

    async invoke(
        options: vscode.LanguageModelToolInvocationOptions<IExampleToolParams>,
        token: vscode.CancellationToken
    ): Promise<vscode.LanguageModelToolResult> {
        // 1. Validate input
        const { param1, param2 } = options.input;

        // 2. Execute tool logic
        const result = await this.exampleService.doSomething(param1, param2, token);

        // 3. Format results
        const content = formatResults(result);

        // 4. Create references (if applicable)
        const references = createReferences(result);

        // 5. Return result
        return new vscode.LanguageModelToolResult(
            [new vscode.LanguageModelTextPart(content)],
            { count: result.length, references }
        );
    }
}

// Register tool at module load time
ToolRegistry.registerTool(ExampleTool);
```

## Testing Tools

Tool testing typically involves:

1. Mock service dependencies
2. Create test input parameters
3. Invoke tool with mocked services
4. Assert on returned results and side effects

## Configuration

Tools can be conditionally enabled via:

- `when` clauses in package.json
- Configuration settings (e.g., `config.github.copilot.chat.tools.memory.enabled`)
- Tags for feature gating

## Telemetry

Most tools emit telemetry events:

- Invocation counts
- Success/failure rates
- Execution times
- Error details

## Security Considerations

- Tools respect `.copilotignore` files
- File operations validate paths
- Authentication required for external APIs
- Token cancellation for long operations

## Performance Optimization

- Lazy instantiation of services
- Parallel operations where possible
- Caching of expensive computations
- Timeouts and fallbacks
- Incremental results for large operations

## References

For detailed implementation of specific tools, see the individual tool documentation files in this directory.
