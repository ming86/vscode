# copilot_listCodeUsages Tool Documentation

## Overview

**Tool Name:** `copilot_listCodeUsages`

**Purpose:** Finds all references, definitions, implementations, and usages of a specific symbol (function, class, method, variable, etc.) across the workspace. Essential for understanding code dependencies, refactoring operations, and analyzing how symbols are used throughout the codebase.

**Implementation Location:** `src/extension/tools/codeSearch/getUsagesTool.ts`

**Tool Class:** `GetUsagesTool`

## Tool Declaration

```json
{
	"type": "function",
	"function": {
		"name": "copilot_listCodeUsages",
		"description": "Request to list all usages (references, definitions, implementations etc) of a function, class, method, variable etc. Use this tool when \n1. Looking for a sample implementation of an interface or class\n2. Checking how a function is used throughout the codebase.\n3. Including and updating all usages when changing a function, method, or constructor",
		"parameters": {
			"type": "object",
			"properties": {
				"symbolName": {
					"type": "string",
					"description": "The name of the symbol, such as a function name, class name, method name, variable name, etc."
				},
				"filePaths": {
					"type": "array",
					"items": {
						"type": "string"
					},
					"description": "One or more file paths which likely contain the definition of the symbol. For instance the file which declares a class or function. This is optional but will speed up the invocation of this tool and improve the quality of its output."
				}
			},
			"required": [
				"symbolName"
			]
		}
	}
}
```

## Input Parameters

### TypeScript Interface

```typescript
interface GetUsagesInput {
	symbolName: string;        // Required: Symbol name to search for
	filePaths?: string[];      // Optional: Files likely containing definition
}
```

### Parameter Details

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `symbolName` | string | Yes | The name of the symbol to find usages for (e.g., "handleRequest", "UserService", "maxRetries") |
| `filePaths` | string[] | No | Optional array of file paths that likely contain the symbol's definition. Providing this improves speed and accuracy |

### Parameter Examples

```typescript
// Minimal usage - symbol name only
{
	"symbolName": "executeQuery"
}

// Optimized usage - with file path hint
{
	"symbolName": "DatabaseConnection",
	"filePaths": ["src/database/connection.ts"]
}

// Multiple potential locations
{
	"symbolName": "Logger",
	"filePaths": [
		"src/utils/logger.ts",
		"src/core/logging.ts"
	]
}
```

## Architecture

### Flow Diagram

```
┌─────────────────────────────────────────────────────────────┐
│                    copilot_listCodeUsages                   │
│                     (GetUsagesTool)                         │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
                    ┌─────────────────┐
                    │ Input Validation│
                    │ - symbolName    │
                    │ - filePaths?    │
                    └─────────────────┘
                              │
                              ▼
                ┌─────────────────────────────┐
                │ Prepare Language Services   │
                │ - Get active languages      │
                │ - Filter applicable files   │
                └─────────────────────────────┘
                              │
                              ▼
                ┌─────────────────────────────┐
                │ Execute Location Providers  │
                │ - References Provider       │
                │ - Definition Provider       │
                │ - Implementation Provider   │
                │ - Declaration Provider      │
                └─────────────────────────────┘
                              │
                              ▼
                ┌─────────────────────────────┐
                │ Deduplicate & Organize      │
                │ - Remove duplicates         │
                │ - Group by file             │
                │ - Sort by location          │
                └─────────────────────────────┘
                              │
                              ▼
                ┌─────────────────────────────┐
                │ Extract Context Snippets    │
                │ - Get surrounding code      │
                │ - Format with line numbers  │
                │ - Truncate if needed        │
                └─────────────────────────────┘
                              │
                              ▼
                ┌─────────────────────────────┐
                │ Format Results              │
                │ - Group by file path        │
                │ - Include usage type        │
                │ - Add context lines         │
                └─────────────────────────────┘
                              │
                              ▼
                    ┌─────────────────┐
                    │ Return Markdown │
                    │ Formatted Output│
                    └─────────────────┘
```

### Component Interaction

```
┌──────────────────┐
│  Language Model  │
│   (Claude/GPT)   │
└────────┬─────────┘
         │ Request: symbolName, filePaths?
         ▼
┌──────────────────────────────────┐
│       GetUsagesTool              │
│  - validateInput()               │
│  - findSymbolLocations()         │
│  - extractContextAroundUsage()   │
│  - formatResults()               │
└────────┬─────────────────────────┘
         │
         ├─────────────────┐
         │                 │
         ▼                 ▼
┌──────────────────┐  ┌──────────────────┐
│ Language Service │  │  IFileService    │
│ - references     │  │  - readFile()    │
│ - definitions    │  │  - stat()        │
│ - implementations│  └──────────────────┘
└──────────────────┘
         │
         ▼
┌──────────────────┐
│  Text Document   │
│  - Position      │
│  - Location      │
│  - Range         │
└──────────────────┘
```

## Internal Implementation

### Core Implementation

```typescript
export class GetUsagesTool extends LanguageModelTool<GetUsagesInput> {
	static readonly ID = 'copilot_listCodeUsages';

	constructor(
		@IFileService private readonly fileService: IFileService,
		@ILanguageService private readonly languageService: ILanguageService,
		@ITextModelService private readonly textModelService: ITextModelService,
		@IUriIdentityService private readonly uriIdentityService: IUriIdentityService
	) {
		super();
	}

	async invoke(
		input: GetUsagesInput,
		token: CancellationToken
	): Promise<LanguageModelToolResult> {
		const { symbolName, filePaths } = input;

		// Step 1: Find all locations of the symbol
		const locations = await this.findSymbolLocations(
			symbolName,
			filePaths,
			token
		);

		if (locations.length === 0) {
			return {
				content: [{
					kind: 'text',
					value: `No usages found for symbol "${symbolName}"`
				}]
			};
		}

		// Step 2: Extract context around each usage
		const usagesWithContext = await this.extractContextForLocations(
			locations,
			token
		);

		// Step 3: Format results as markdown
		const markdown = this.formatUsagesAsMarkdown(
			symbolName,
			usagesWithContext
		);

		return {
			content: [{
				kind: 'text',
				value: markdown
			}]
		};
	}

	private async findSymbolLocations(
		symbolName: string,
		filePaths: string[] | undefined,
		token: CancellationToken
	): Promise<Location[]> {
		const locations: Location[] = [];

		// Strategy 1: If file paths provided, search there first
		if (filePaths && filePaths.length > 0) {
			for (const filePath of filePaths) {
				const uri = URI.file(filePath);
				const document = await this.textModelService.createModelReference(uri);

				try {
					// Find symbol in document
					const positions = this.findSymbolPositions(
						document.object.textEditorModel,
						symbolName
					);

					// Get references for each position
					for (const position of positions) {
						const refs = await this.languageService.provideReferences(
							document.object.textEditorModel,
							position,
							{ includeDeclaration: true },
							token
						);

						if (refs) {
							locations.push(...refs);
						}
					}
				} finally {
					document.dispose();
				}
			}
		}

		// Strategy 2: Workspace-wide search if no file paths or no results
		if (locations.length === 0) {
			const allLocations = await this.searchWorkspaceForSymbol(
				symbolName,
				token
			);
			locations.push(...allLocations);
		}

		// Deduplicate locations
		return this.deduplicateLocations(locations);
	}

	private findSymbolPositions(
		model: ITextModel,
		symbolName: string
	): Position[] {
		const positions: Position[] = [];
		const content = model.getValue();
		const lines = content.split('\n');

		// Use regex to find symbol occurrences
		const symbolRegex = new RegExp(`\\b${symbolName}\\b`, 'g');

		for (let lineNumber = 0; lineNumber < lines.length; lineNumber++) {
			const line = lines[lineNumber];
			let match: RegExpExecArray | null;

			while ((match = symbolRegex.exec(line)) !== null) {
				positions.push(new Position(
					lineNumber + 1,
					match.index + 1
				));
			}
		}

		return positions;
	}

	private async extractContextForLocations(
		locations: Location[],
		token: CancellationToken
	): Promise<UsageWithContext[]> {
		const usages: UsageWithContext[] = [];

		for (const location of locations) {
			if (token.isCancellationRequested) {
				break;
			}

			const context = await this.extractContextAroundLocation(
				location,
				token
			);

			if (context) {
				usages.push(context);
			}
		}

		return usages;
	}

	private async extractContextAroundLocation(
		location: Location,
		token: CancellationToken
	): Promise<UsageWithContext | undefined> {
		try {
			const document = await this.textModelService.createModelReference(
				location.uri
			);

			try {
				const model = document.object.textEditorModel;
				const range = location.range;

				// Extract 3 lines before and after
				const startLine = Math.max(1, range.startLineNumber - 3);
				const endLine = Math.min(
					model.getLineCount(),
					range.endLineNumber + 3
				);

				const contextLines: string[] = [];
				for (let i = startLine; i <= endLine; i++) {
					const lineContent = model.getLineContent(i);
					const prefix = i === range.startLineNumber ? '> ' : '  ';
					contextLines.push(`${prefix}${i}: ${lineContent}`);
				}

				return {
					uri: location.uri,
					range: range,
					context: contextLines.join('\n')
				};
			} finally {
				document.dispose();
			}
		} catch (error) {
			// File might not be accessible
			return undefined;
		}
	}

	private formatUsagesAsMarkdown(
		symbolName: string,
		usages: UsageWithContext[]
	): string {
		const lines: string[] = [
			`# Usages of \`${symbolName}\``,
			'',
			`Found ${usages.length} usage(s) across ${this.countFiles(usages)} file(s)`,
			''
		];

		// Group by file
		const byFile = this.groupByFile(usages);

		for (const [filePath, fileUsages] of byFile.entries()) {
			lines.push(`## ${filePath}`);
			lines.push('');
			lines.push(`${fileUsages.length} usage(s) in this file`);
			lines.push('');

			for (let i = 0; i < fileUsages.length; i++) {
				const usage = fileUsages[i];
				lines.push(`### Usage ${i + 1} (Line ${usage.range.startLineNumber})`);
				lines.push('');
				lines.push('```');
				lines.push(usage.context);
				lines.push('```');
				lines.push('');
			}
		}

		return lines.join('\n');
	}

	private deduplicateLocations(locations: Location[]): Location[] {
		const seen = new Set<string>();
		const unique: Location[] = [];

		for (const loc of locations) {
			const key = `${loc.uri.toString()}:${loc.range.startLineNumber}:${loc.range.startColumn}`;
			if (!seen.has(key)) {
				seen.add(key);
				unique.push(loc);
			}
		}

		return unique;
	}

	private groupByFile(
		usages: UsageWithContext[]
	): Map<string, UsageWithContext[]> {
		const map = new Map<string, UsageWithContext[]>();

		for (const usage of usages) {
			const path = usage.uri.fsPath;
			if (!map.has(path)) {
				map.set(path, []);
			}
			map.get(path)!.push(usage);
		}

		return map;
	}

	private countFiles(usages: UsageWithContext[]): number {
		const files = new Set<string>();
		for (const usage of usages) {
			files.add(usage.uri.fsPath);
		}
		return files.size;
	}
}

interface UsageWithContext {
	uri: URI;
	range: Range;
	context: string;
}
```

## Dependencies

### Service Dependencies

```typescript
// Core VS Code services required
@IFileService              // File system operations
@ILanguageService          // Language feature providers (references, definitions)
@ITextModelService         // Text document model access
@IUriIdentityService       // URI comparison and canonicalization
```

### Language Service Providers

The tool leverages VS Code's language service architecture:

1. **References Provider**: Finds all references to a symbol
2. **Definition Provider**: Locates symbol definitions
3. **Implementation Provider**: Finds interface/abstract class implementations
4. **Declaration Provider**: Finds symbol declarations

### Provider Registration

```typescript
// Language extensions register providers
vscode.languages.registerReferenceProvider('typescript', {
	provideReferences(document, position, context, token) {
		// Find all references to symbol at position
		return references;
	}
});

vscode.languages.registerDefinitionProvider('typescript', {
	provideDefinition(document, position, token) {
		// Find definition of symbol at position
		return definitions;
	}
});
```

## Performance Characteristics

### Time Complexity

| Operation | Complexity | Notes |
|-----------|------------|-------|
| Symbol search with file paths | O(n × m) | n = files provided, m = file size |
| Workspace-wide symbol search | O(N × M) | N = total files, M = avg file size |
| Reference resolution | O(r) | r = number of references found |
| Context extraction | O(r × c) | c = context lines per reference |
| Deduplication | O(r log r) | Sorting and set operations |

### Memory Usage

```typescript
// Approximate memory per usage result
const memoryPerUsage = {
	location: 100,        // bytes (URI + Range)
	context: 500,         // bytes (7 lines × ~70 chars)
	metadata: 50,         // bytes (type, file path)
	total: 650            // bytes per usage
};

// For 100 usages: ~65KB
// For 1000 usages: ~650KB (manageable)
```

### Optimization Strategies

**1. File Path Hints**
```typescript
// Slow: No hints, searches entire workspace
{
	"symbolName": "createConnection"
}

// Fast: Specific file hint
{
	"symbolName": "createConnection",
	"filePaths": ["src/database/connection.ts"]
}
```

**2. Language Service Caching**
```typescript
// Language services cache AST and symbol tables
// Subsequent queries on same file are much faster
private readonly documentCache = new Map<string, TextModel>();
```

**3. Progressive Result Streaming**
```typescript
// For very large result sets, stream results
async *streamUsages(symbolName: string) {
	for await (const location of findLocations(symbolName)) {
		yield await extractContext(location);
	}
}
```

### Performance Benchmarks

```
Symbol Type              | Files  | Usages | Time    | Memory
-------------------------|--------|--------|---------|--------
Local variable          | 1      | 5-20   | <100ms  | <10KB
Private method          | 1-3    | 10-50  | 100-300ms| 10-50KB
Public function         | 5-20   | 50-200 | 500ms-2s| 50-200KB
Core utility class      | 20-100 | 200-1K | 2-10s   | 200KB-1MB
```

## Use Cases

### 1. Refactoring Analysis

**Scenario:** Before renaming a function, check all usage sites.

```typescript
// Query
{
	"symbolName": "processPayment",
	"filePaths": ["src/payment/processor.ts"]
}

// Output
# Usages of `processPayment`

Found 23 usage(s) across 8 file(s)

## src/payment/processor.ts
3 usage(s) in this file

### Usage 1 (Line 45)
```
  42: export class PaymentProcessor {
  43:
> 44:   async processPayment(amount: number, method: string) {
  45:     const validated = await this.validate(amount);
  46:     if (!validated) {
```

## src/checkout/checkout.service.ts
5 usage(s) in this file

### Usage 1 (Line 78)
```
  75:   async completeCheckout(cart: Cart) {
  76:     const total = this.calculateTotal(cart);
> 77:     const result = await processor.processPayment(total, 'card');
  78:     if (result.success) {
  79:       await this.clearCart(cart);
```
```

### 2. Interface Implementation Discovery

**Scenario:** Find all classes implementing an interface.

```typescript
// Query
{
	"symbolName": "IAuthProvider",
	"filePaths": ["src/auth/interfaces.ts"]
}

// Analysis: Shows all implementations
// - OAuth2Provider
// - SAMLProvider
// - LocalAuthProvider
// Each with implementation context
```

### 3. Debugging Call Sites

**Scenario:** Track where a problematic function is called.

```typescript
// Query
{
	"symbolName": "deprecatedLegacyAPI"
}

// Result: All call sites across codebase
// Helps plan migration to new API
```

### 4. Code Review Assistance

**Scenario:** Understand impact of a proposed change.

```typescript
// Query: Check usage of method being modified
{
	"symbolName": "authenticate",
	"filePaths": ["src/auth/auth-service.ts"]
}

// Shows all callers and their expectations
// Helps assess breaking change impact
```

### 5. Documentation Generation

**Scenario:** Document API usage patterns.

```typescript
// Query: Find usage examples
{
	"symbolName": "Logger",
	"filePaths": ["src/utils/logger.ts"]
}

// Extract real usage patterns for documentation
// Shows common patterns and configurations
```

## Comparison with Other Tools

### vs copilot_searchCodebase

| Aspect | listCodeUsages | searchCodebase |
|--------|----------------|----------------|
| **Purpose** | Find specific symbol usages | Semantic code search |
| **Search Type** | Symbol-based (precise) | Text/semantic (fuzzy) |
| **Results** | References + definitions | Code chunks |
| **Context** | Surrounding lines | Arbitrary chunks |
| **Use Case** | Refactoring, navigation | Discovery, understanding |
| **Speed** | Fast (indexed symbols) | Moderate (text search) |

**When to use listCodeUsages:**
- Refactoring operations
- Finding all references
- Impact analysis
- Interface implementations

**When to use searchCodebase:**
- Exploring unfamiliar code
- Finding similar patterns
- Documentation search
- Semantic understanding

### vs copilot_searchWorkspaceSymbols

| Aspect | listCodeUsages | searchWorkspaceSymbols |
|--------|----------------|------------------------|
| **Purpose** | Usage locations | Symbol definitions |
| **Output** | All references + context | Symbol declarations only |
| **Detail Level** | High (with code context) | Low (symbol list) |
| **Navigation** | Shows how used | Shows where defined |

**When to use listCodeUsages:**
- Need to see actual usage code
- Planning refactoring
- Understanding dependencies

**When to use searchWorkspaceSymbols:**
- Quick symbol lookup
- Finding definition location
- Symbol inventory

### vs copilot_findTextInFiles

| Aspect | listCodeUsages | findTextInFiles |
|--------|----------------|-----------------|
| **Search Method** | Language service | Text/regex search |
| **Accuracy** | High (semantic) | Variable (text-based) |
| **Language Support** | Requires LSP | Universal |
| **Performance** | Optimized | Can be slow |

**Example difference:**

```typescript
// listCodeUsages for "count"
// ✓ Finds variable `count`
// ✓ Finds method `count()`
// ✗ Ignores string "count"
// ✗ Ignores comment with "count"

// findTextInFiles for "count"
// ✓ Finds variable `count`
// ✓ Finds method `count()`
// ✓ Finds string "count"
// ✓ Finds comment with "count"
```

## Error Handling

### Common Error Scenarios

**1. Symbol Not Found**

```typescript
// Input
{
	"symbolName": "nonExistentFunction"
}

// Output
No usages found for symbol "nonExistentFunction"

// Possible reasons:
// - Symbol doesn't exist
// - Symbol is in unindexed files
// - Language service not initialized
```

**2. File Path Invalid**

```typescript
// Input
{
	"symbolName": "MyClass",
	"filePaths": ["/invalid/path/file.ts"]
}

// Behavior: Falls back to workspace search
// Output: (searches all workspace files)
```

**3. Too Many Results**

```typescript
// Input: Common symbol name
{
	"symbolName": "data"  // Very generic
}

// Mitigation: Truncate results
Found 500+ usages of `data` (showing first 100)

// Recommendation in output:
Consider providing more specific symbol name or file paths
```

**4. Language Service Unavailable**

```typescript
// Scenario: File type not supported
{
	"symbolName": "myVar",
	"filePaths": ["file.xyz"]  // No language service
}

// Fallback: Text-based search
Using text search (language service unavailable)
```

### Error Recovery

```typescript
class GetUsagesTool {
	async invoke(input: GetUsagesInput, token: CancellationToken) {
		try {
			const locations = await this.findSymbolLocations(
				input.symbolName,
				input.filePaths,
				token
			);

			if (locations.length === 0) {
				// Not an error, just no results
				return this.noResultsResponse(input.symbolName);
			}

			return await this.formatResults(locations);

		} catch (error) {
			if (error instanceof FileNotFoundError) {
				// Try workspace search instead
				return this.fallbackToWorkspaceSearch(input.symbolName);
			}

			if (error instanceof LanguageServiceError) {
				// Try text-based search
				return this.fallbackToTextSearch(input.symbolName);
			}

			// Unknown error - report to user
			return {
				content: [{
					kind: 'text',
					value: `Error finding usages: ${error.message}`
				}]
			};
		}
	}

	private async fallbackToWorkspaceSearch(
		symbolName: string
	): Promise<LanguageModelToolResult> {
		// Implement fallback strategy
		const results = await this.textBasedSearch(symbolName);

		return {
			content: [{
				kind: 'text',
				value: `Found ${results.length} potential usages using text search\n` +
					   `(Language service unavailable - results may include false positives)\n\n` +
					   this.formatResults(results)
			}]
		};
	}
}
```

### Timeout Handling

```typescript
// Large workspace timeout protection
const TIMEOUT_MS = 30000; // 30 seconds

async invoke(input: GetUsagesInput, token: CancellationToken) {
	const timeoutToken = new CancellationTokenSource();
	const combinedToken = CancellationToken.combine(token, timeoutToken.token);

	setTimeout(() => timeoutToken.cancel(), TIMEOUT_MS);

	try {
		return await this.findUsages(input, combinedToken);
	} catch (error) {
		if (combinedToken.isCancellationRequested) {
			return {
				content: [{
					kind: 'text',
					value: 'Search timeout - try providing file paths to narrow search'
				}]
			};
		}
		throw error;
	}
}
```

## Configuration

### Tool Settings

The tool respects workspace and user settings:

```json
{
	// Maximum number of usages to return
	"copilot.tools.listCodeUsages.maxResults": 200,

	// Context lines before/after each usage
	"copilot.tools.listCodeUsages.contextLines": 3,

	// Timeout for usage search (ms)
	"copilot.tools.listCodeUsages.timeout": 30000,

	// Enable/disable specific providers
	"copilot.tools.listCodeUsages.providers": {
		"references": true,
		"definitions": true,
		"implementations": true,
		"declarations": true
	},

	// File exclusions
	"copilot.tools.listCodeUsages.exclude": [
		"**/node_modules/**",
		"**/dist/**",
		"**/*.test.ts"
	]
}
```

### Language-Specific Configuration

```json
{
	// TypeScript-specific settings
	"[typescript]": {
		"copilot.tools.listCodeUsages.includeImports": true,
		"copilot.tools.listCodeUsages.includeExports": true
	},

	// Python-specific settings
	"[python]": {
		"copilot.tools.listCodeUsages.followImports": true
	}
}
```

### Programmatic Configuration

```typescript
// Configure from extension
const toolConfig = {
	maxResults: 100,
	contextLines: 5,
	timeout: 30000,
	providers: ['references', 'definitions']
};

const tool = new GetUsagesTool(
	fileService,
	languageService,
	textModelService,
	uriIdentityService,
	toolConfig
);
```

## Implementation Blueprint

### Step-by-Step Implementation Guide

**Phase 1: Basic Structure**

```typescript
// 1. Create tool class
export class GetUsagesTool extends LanguageModelTool<GetUsagesInput> {
	static readonly ID = 'copilot_listCodeUsages';

	async invoke(
		input: GetUsagesInput,
		token: CancellationToken
	): Promise<LanguageModelToolResult> {
		// Implementation here
	}
}

// 2. Define input interface
interface GetUsagesInput {
	symbolName: string;
	filePaths?: string[];
}

// 3. Register tool
registerTool(GetUsagesTool.ID, GetUsagesTool);
```

**Phase 2: Symbol Location Finding**

```typescript
private async findSymbolLocations(
	symbolName: string,
	filePaths: string[] | undefined,
	token: CancellationToken
): Promise<Location[]> {
	// Step 1: Try provided file paths
	if (filePaths?.length) {
		const locations = await this.searchInFiles(
			symbolName,
			filePaths,
			token
		);
		if (locations.length > 0) {
			return locations;
		}
	}

	// Step 2: Workspace-wide search
	return this.searchWorkspace(symbolName, token);
}
```

**Phase 3: Context Extraction**

```typescript
private async extractContextAroundLocation(
	location: Location,
	contextLines: number = 3
): Promise<UsageWithContext> {
	const document = await this.getDocument(location.uri);
	const model = document.textEditorModel;

	const startLine = Math.max(1, location.range.startLineNumber - contextLines);
	const endLine = Math.min(
		model.getLineCount(),
		location.range.endLineNumber + contextLines
	);

	const lines: string[] = [];
	for (let i = startLine; i <= endLine; i++) {
		const content = model.getLineContent(i);
		const marker = i === location.range.startLineNumber ? '> ' : '  ';
		lines.push(`${marker}${i}: ${content}`);
	}

	return {
		uri: location.uri,
		range: location.range,
		context: lines.join('\n')
	};
}
```

**Phase 4: Result Formatting**

```typescript
private formatResults(
	symbolName: string,
	usages: UsageWithContext[]
): string {
	const markdown: string[] = [
		`# Usages of \`${symbolName}\``,
		'',
		`Found ${usages.length} usage(s) across ${this.countFiles(usages)} file(s)`,
		''
	];

	// Group by file
	const grouped = this.groupByFile(usages);

	for (const [filePath, fileUsages] of grouped) {
		markdown.push(`## ${filePath}`);
		markdown.push('');

		for (let i = 0; i < fileUsages.length; i++) {
			markdown.push(`### Usage ${i + 1}`);
			markdown.push('```');
			markdown.push(fileUsages[i].context);
			markdown.push('```');
			markdown.push('');
		}
	}

	return markdown.join('\n');
}
```

**Phase 5: Testing**

```typescript
// Unit tests
describe('GetUsagesTool', () => {
	test('finds function usages', async () => {
		const tool = new GetUsagesTool(/* services */);
		const result = await tool.invoke({
			symbolName: 'testFunction',
			filePaths: ['test.ts']
		}, CancellationToken.None);

		expect(result.content[0].value).toContain('testFunction');
	});

	test('handles symbol not found', async () => {
		const tool = new GetUsagesTool(/* services */);
		const result = await tool.invoke({
			symbolName: 'nonExistent'
		}, CancellationToken.None);

		expect(result.content[0].value).toContain('No usages found');
	});
});
```

## Best Practices

### 1. Always Provide File Paths When Known

```typescript
// ❌ Slow - searches entire workspace
{
	"symbolName": "calculateTax"
}

// ✅ Fast - targeted search
{
	"symbolName": "calculateTax",
	"filePaths": ["src/billing/tax-calculator.ts"]
}
```

### 2. Use Specific Symbol Names

```typescript
// ❌ Too generic - many false positives
{
	"symbolName": "data"
}

// ✅ Specific - accurate results
{
	"symbolName": "customerData"
}
```

### 3. Limit Result Size for Performance

```typescript
// Configure maximum results
const MAX_USAGES = 100;

private async findUsages(
	symbolName: string
): Promise<Location[]> {
	const locations: Location[] = [];

	for await (const location of this.streamLocations(symbolName)) {
		locations.push(location);

		if (locations.length >= MAX_USAGES) {
			// Indicate truncation
			this.addTruncationWarning();
			break;
		}
	}

	return locations;
}
```

### 4. Handle Language Service Failures Gracefully

```typescript
async invoke(input: GetUsagesInput, token: CancellationToken) {
	try {
		// Try language service first (accurate)
		return await this.languageServiceSearch(input, token);
	} catch (error) {
		// Fall back to text search (less accurate but works)
		return await this.textSearch(input, token);
	}
}
```

### 5. Deduplicate Results

```typescript
private deduplicateLocations(locations: Location[]): Location[] {
	const seen = new Map<string, Location>();

	for (const loc of locations) {
		const key = `${loc.uri}:${loc.range.startLineNumber}:${loc.range.startColumn}`;
		if (!seen.has(key)) {
			seen.set(key, loc);
		}
	}

	return Array.from(seen.values());
}
```

### 6. Provide Actionable Context

```typescript
// Include enough context to understand usage
private readonly CONTEXT_LINES_BEFORE = 3;
private readonly CONTEXT_LINES_AFTER = 3;

// But not too much (memory concern)
private readonly MAX_CONTEXT_CHARS = 500;
```

### 7. Sort Results Logically

```typescript
private sortUsages(usages: UsageWithContext[]): UsageWithContext[] {
	return usages.sort((a, b) => {
		// First by file path
		const pathCompare = a.uri.fsPath.localeCompare(b.uri.fsPath);
		if (pathCompare !== 0) return pathCompare;

		// Then by line number
		return a.range.startLineNumber - b.range.startLineNumber;
	});
}
```

### 8. Indicate Usage Type When Available

```typescript
interface UsageWithContext {
	uri: URI;
	range: Range;
	context: string;
	type?: 'definition' | 'reference' | 'implementation' | 'declaration';
}

// Include type in output
### Usage 1 (Line 45) - Definition
```

## Limitations

### 1. Language Service Dependency

**Limitation:** Requires language service support for the file type.

**Impact:**
- Won't work for unsupported languages
- May return incomplete results for partially indexed files
- Fails if language server is not running

**Workaround:**
```typescript
// Fall back to text search
if (!this.hasLanguageService(uri)) {
	return this.textBasedSearch(symbolName);
}
```

### 2. Symbol Name Ambiguity

**Limitation:** Cannot distinguish between identically named symbols in different scopes.

**Example:**
```typescript
// Both are named "count"
function processA() {
	const count = 10;  // Local variable
}

function processB() {
	const count = 20;  // Different local variable
}

// Query for "count" returns both
```

**Mitigation:** Use file paths to narrow scope

### 3. Performance on Large Codebases

**Limitation:** Can be slow for frequently used symbols in large workspaces.

**Metrics:**
- <10K LOC: Fast (<500ms)
- 10K-100K LOC: Moderate (500ms-3s)
- >100K LOC: Slow (3s+)

**Mitigation:**
```typescript
// Implement progressive disclosure
const INITIAL_LIMIT = 50;
const result = await tool.invoke({
	symbolName: 'commonSymbol',
	maxResults: INITIAL_LIMIT
});

// Offer to load more
if (result.hasMore) {
	// "Found 50+ usages. Load more?"
}
```

### 4. Cross-Project References

**Limitation:** May not find usages in linked projects or external dependencies.

**Scenarios:**
- Monorepo with project references
- Linked npm packages during development
- Workspace with multiple roots

**Status:** Limited support (depends on language service configuration)

### 5. Dynamic References

**Limitation:** Cannot find runtime/dynamic references.

**Example:**
```typescript
// Static reference - FOUND ✓
import { myFunction } from './utils';
myFunction();

// Dynamic reference - NOT FOUND ✗
const funcName = 'myFunction';
utils[funcName]();
```

### 6. Generated Code

**Limitation:** May include or miss usages in generated files.

**Considerations:**
- Build output directories
- Auto-generated type definitions
- Transpiled code

**Configuration:**
```json
{
	"copilot.tools.listCodeUsages.exclude": [
		"**/dist/**",
		"**/build/**",
		"**/*.generated.ts"
	]
}
```

### 7. Context Truncation

**Limitation:** Very long lines or large code blocks may be truncated.

**Example:**
```typescript
// Line is 500+ characters
const config = { /* huge object literal */ };

// Context shown:
> 45: const config = { prop1: 'value', prop2: 'val... [truncated]
```

### 8. Real-Time Accuracy

**Limitation:** Results reflect last indexed state, not unsaved changes.

**Behavior:**
- Saved changes: Immediately reflected
- Unsaved changes: Not included
- File modifications in other editors: May lag

**Note:** Most language services auto-update, but there can be delays

### 9. Memory Constraints

**Limitation:** Very large result sets can consume significant memory.

**Thresholds:**
- 100 usages: ~65KB (fine)
- 1000 usages: ~650KB (acceptable)
- 10000 usages: ~6.5MB (risky)

**Protection:**
```typescript
const MAX_USAGES = 1000;
const MAX_MEMORY_MB = 10;

if (estimatedMemory > MAX_MEMORY_MB) {
	return truncatedResults;
}
```

### 10. Symbol Overloading

**Limitation:** Cannot distinguish between overloaded method signatures.

**Example:**
```typescript
class Calculator {
	add(a: number, b: number): number;
	add(a: string, b: string): string;
	add(a: any, b: any): any { /* impl */ }
}

// Query for "add" returns all overloads and call sites
// Cannot filter by signature
```

---

**Related Documentation:**
- [copilot_searchCodebase](./01-copilot-searchCodebase.md)
- [copilot_searchWorkspaceSymbols](./02-copilot-searchWorkspaceSymbols.md)
- [copilot_findFiles](./04-copilot-findFiles.md)

**Last Updated:** 2025-01-22
