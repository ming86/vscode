# copilot_searchWorkspaceSymbols Tool

## Overview

**Tool Name**: `copilot_searchWorkspaceSymbols` (internal: `workspace_symbols`)

**Purpose**: Search the workspace for code symbols (classes, functions, variables, etc.) using language service providers.

**Primary File**: `src/extension/tools/vscode-node/searchWorkspaceSymbolsTool.tsx`

**Tool Class**: `SearchWorkspaceSymbolsTool`

## Tool Declaration

```json
{
  "name": "copilot_searchWorkspaceSymbols",
  "toolReferenceName": "symbols",
  "displayName": "Search Workspace Symbols",
  "icon": "$(symbol)",
  "modelDescription": "Search the user's workspace for code symbols using language services. Use this tool when the user is looking for a specific symbol in their workspace.",
  "inputSchema": {
    "type": "object",
    "properties": {
      "symbolName": {
        "type": "string",
        "description": "The symbol to search for, such as a function name, class name, or variable name."
      }
    },
    "required": ["symbolName"]
  }
}
```

## Input Parameters

```typescript
interface ISearchWorkspaceSymbolsTool {
    symbolName: string;  // Required: symbol to search for
}
```

## Architecture

### High-Level Flow

```
Symbol Query
    ↓
SearchWorkspaceSymbolsTool.invoke()
    ↓
ILanguageFeaturesService.getWorkspaceSymbols()
    ↓
VS Code Command: vscode.executeWorkspaceSymbolProvider
    ↓
Language Extension Providers (TypeScript, Python, etc.)
    ↓
Results Filtered & Ranked
    ↓
Format & Return (top 20)
```

## Internal Implementation

### Tool Invocation

```typescript
async invoke(
    options: vscode.LanguageModelToolInvocationOptions<ISearchWorkspaceSymbolsTool>,
    token: CancellationToken
): Promise<vscode.LanguageModelToolResult> {
    // 1. Query language services
    const symbols = await this.languageFeaturesService.getWorkspaceSymbols(
        options.input.symbolName
    );

    checkCancellation(token);

    // 2. Render results
    const result = await renderPromptElementJSON(
        this.instantiationService,
        WorkspaceSymbolSearchOutput,
        { symbols },
        options.tokenizationOptions,
        token
    );

    // 3. Create tool result with message
    const toolResult = new ExtendedLanguageModelToolResult([
        new LanguageModelPromptTsxPart(result)
    ]);

    const query = `\`${options.input.symbolName}\``;
    toolResult.toolResultMessage = symbols.length === 0 ?
        new MarkdownString(l10n.t`Searched for ${query}, no results`) :
        symbols.length === 1 ?
            new MarkdownString(l10n.t`Searched for ${query}, 1 result`) :
            new MarkdownString(l10n.t`Searched for ${query}, ${symbols.length} results`);

    return toolResult;
}
```

### Language Services Integration

**Service Method**:

```typescript
// From languageFeaturesServicesImpl.ts
async getWorkspaceSymbols(query: string): Promise<vscode.SymbolInformation[]> {
    return await vscode.commands.executeCommand<vscode.SymbolInformation[]>(
        'vscode.executeWorkspaceSymbolProvider',
        query
    );
}
```

**How It Works**:

1. VS Code's command system dispatches to all registered workspace symbol providers
2. Each language extension (TypeScript, Python, Java, etc.) provides symbols matching the query
3. Providers use language-specific indexes and parsing
4. Results are automatically ranked by VS Code based on:
   - Exact matches (highest priority)
   - Prefix matches
   - Substring matches
   - Symbol kind relevance
5. Typical limit: ~256 results from VS Code API

### Symbol Types Supported

All `vscode.SymbolKind` types:

| Category | Types |
|----------|-------|
| **Modules** | File, Module, Namespace, Package |
| **Classes** | Class, Interface, Struct |
| **Functions** | Function, Method, Constructor |
| **Variables** | Variable, Constant, Property, Field |
| **Enums** | Enum, EnumMember |
| **Other** | String, Number, Boolean, Array, Object, Key, Null, Event, Operator, TypeParameter |

### Result Structure

Each symbol result contains:

```typescript
interface vscode.SymbolInformation {
    name: string;                    // Symbol name
    kind: vscode.SymbolKind;        // Symbol type
    containerName?: string;          // Parent symbol (e.g., class for method)
    location: vscode.Location;       // File URI + Range
}
```

**Example Result**:

```
Symbol: authenticate
Kind: Function
Container: AuthService
Location: src/auth/service.ts:42-58
```

### Result Formatting

**Rendering Component**: `WorkspaceSymbolSearchOutput`

The tool limits output to **top 20 results** to preserve token budget.

**Output Format**:

```xml
<symbolResult priority="100">
  <symbol name="AuthService" kind="Class" container="" />
  <location uri="file:///workspace/src/auth/service.ts" range="15:0-45:1" />
</symbolResult>
<symbolResult priority="90">
  <symbol name="authenticate" kind="Method" container="AuthService" />
  <location uri="file:///workspace/src/auth/service.ts" range="25:4-35:5" />
</symbolResult>
```

**Priority Ranking**: Earlier results have higher priority (100, 90, 80, ...) for token budgeting.

## Dependencies

### Core Services

1. **ILanguageFeaturesService**: Bridge to VS Code language services
2. **IInstantiationService**: Dependency injection
3. **IPromptPathRepresentationService**: Path formatting for display

### Language Extensions

The tool relies on language extensions providing symbol providers:

- **TypeScript/JavaScript**: Built-in VS Code extension
- **Python**: Python extension
- **Java**: Java language support
- **C/C++**: C/C++ extension
- **Go**: Go extension
- **Rust**: rust-analyzer
- **Others**: Any extension implementing `WorkspaceSymbolProvider`

## Search Behavior

### Query Matching

**Case-Insensitive**: Queries are case-insensitive by default

**Match Types**:

| Query | Matches | Priority |
|-------|---------|----------|
| `Auth` | `AuthService`, `Authenticator` | Prefix (high) |
| `auth` | `isAuthenticated`, `authToken` | Substring (medium) |
| `AS` | `AuthService` (camelCase) | Acronym (medium) |

**CamelCase Matching**: Many providers support fuzzy matching

- Query: `AS` → Matches: `AuthService`, `AppState`
- Query: `gtd` → Matches: `getTestData`

### Ranking Algorithm

Providers typically rank by:

1. **Exact match** (highest)
2. **Prefix match**
3. **Word boundary match** (camelCase)
4. **Substring match**
5. **Fuzzy match** (lowest)

Additional factors:

- **Symbol kind**: Classes/functions ranked higher than variables
- **Recency**: Recently used symbols boosted
- **Workspace location**: Same-project symbols prioritized

## Performance Characteristics

### Execution Time

- **Fast queries**: < 100ms (cached index)
- **Cold start**: 200-500ms (building index)
- **Large workspaces**: May take 1-2s

### Scalability

- **Small workspaces** (< 100 files): Instant
- **Medium workspaces** (100-1000 files): Fast (< 500ms)
- **Large workspaces** (> 1000 files): Depends on language indexing

### Optimization Techniques

1. **Pre-built Indexes**: Language servers maintain symbol indexes
2. **Incremental Updates**: Indexes update on file changes
3. **Result Caching**: Recent queries cached by VS Code
4. **Top-N Limiting**: Tool caps at 20 results for token efficiency
5. **Lazy Loading**: Symbols loaded on-demand, not all upfront

## Use Cases

### 1. Finding Class Definitions

**Query**: `UserService`

**Result**: All classes/interfaces named UserService

### 2. Locating Functions

**Query**: `calculateTotal`

**Result**: All functions/methods with that name

### 3. Finding Variables

**Query**: `API_KEY`

**Result**: All constant/variable declarations

### 4. Discovering Interfaces

**Query**: `IUser`

**Result**: Interface definitions (TypeScript convention)

### 5. Fuzzy Search

**Query**: `usrAuth` (abbreviated)

**Result**: `UserAuthenticator`, `userAuthService`

## Comparison with Other Tools

### vs. copilot_searchCodebase

| Feature | searchWorkspaceSymbols | searchCodebase |
|---------|------------------------|----------------|
| **Search Target** | Symbols (names) | Content (semantics) |
| **Technology** | Language services | Embeddings |
| **Speed** | Fast (indexed) | Slower (AI) |
| **Accuracy** | Exact names | Conceptual |
| **Use Case** | "Find class X" | "Find auth code" |

### vs. copilot_listCodeUsages

| Feature | searchWorkspaceSymbols | listCodeUsages |
|---------|------------------------|----------------|
| **Purpose** | Find symbols | Find usages |
| **Input** | Symbol name | Symbol name + hint |
| **Output** | Definitions | Defs + Refs + Impls |

### vs. copilot_findTextInFiles

| Feature | searchWorkspaceSymbols | findTextInFiles |
|---------|------------------------|-----------------|
| **Parsing** | Yes (AST-based) | No (text-based) |
| **Accuracy** | High (understands code) | Lower (matches text) |
| **Speed** | Fast (pre-indexed) | Medium (on-demand) |
| **Flexibility** | Symbol names only | Any text/regex |

## Error Handling

### Common Scenarios

1. **No Results**: Returns empty array with message
2. **Query Too Short**: Still executes (some providers have min length)
3. **Language Server Down**: Returns empty (graceful degradation)
4. **Cancellation**: Throws `CancellationError`

### Error Messages

- "Searched for `symbolName`, no results"
- "Searched for `symbolName`, N results"

## Configuration

No tool-specific settings. Behavior controlled by:

- **Language extensions**: Must be installed and activated
- **Workspace folders**: Must have workspace open
- **`files.exclude`**: Excluded files not indexed
- **Language-specific settings**: e.g., `typescript.workspaceSymbols`

## Special Features

### Container Name Context

Includes parent symbol for disambiguation:

```
Symbol: render
Container: UserComponent  ← Disambiguates from other "render" methods
```

### Multi-Workspace Support

Searches across all workspace folders in multi-root workspaces.

### Cross-Language

Single query searches all languages:

- Query: `User`
- Results: TypeScript class + Python class + Java class

## Implementation Blueprint

To implement this tool in another project:

### 1. Minimal Implementation

```typescript
class SymbolSearchTool {
    async invoke(symbolName: string): Promise<Symbol[]> {
        // Use editor's symbol provider API
        return await editor.workspace.findSymbols(symbolName);
    }
}
```

### 2. Integration Points

**Required APIs**:

- Workspace symbol provider registration
- Symbol query execution
- Language server protocol (LSP) support

**Example LSP Request**:

```json
{
  "jsonrpc": "2.0",
  "method": "workspace/symbol",
  "params": {
    "query": "AuthService"
  }
}
```

### 3. Result Formatting

```typescript
interface SymbolResult {
    name: string;
    kind: 'class' | 'function' | 'variable' | ...;
    containerName?: string;
    location: {
        file: string;
        line: number;
        column: number;
    };
}
```

## Best Practices

1. **Specific Queries**: Use full or near-full names for best results
2. **CamelCase Shortcuts**: Use acronyms for long names (e.g., `UAS` for `UserAuthService`)
3. **Partial Names**: Prefix matching works well (e.g., `Auth` finds `AuthService`)
4. **Language Context**: Include language-specific prefixes (e.g., `I` for interfaces in TypeScript)
5. **Follow-up Queries**: Use `listCodeUsages` after finding symbol for more details

## Limitations

1. **Requires Language Support**: Language extension must be installed
2. **Index Freshness**: Results may lag behind recent edits
3. **Name-Based Only**: Cannot search by symbol type alone
4. **Top Results**: Limited to 20 results (full results may be 100+)
5. **No Filtering**: Cannot filter by file pattern or directory
6. **No Sorting Options**: Ranking determined by language provider

## Future Enhancements

- Symbol type filtering (e.g., "only classes")
- Workspace scope filtering
- Regex support for symbol names
- Include/exclude patterns
- Custom ranking by recency or frequency
- Symbol relationship queries (e.g., "find all subclasses")
