# Copilot getVSCodeAPI Tool Documentation

## Overview

The `getVSCodeAPI` tool provides access to a curated knowledge base of VS Code API documentation, extension development patterns, and best practices. It enables Copilot to answer questions about VS Code extension development, provide accurate API usage examples, and guide users through complex VS Code integration scenarios.

**Primary Use Cases:**
- VS Code extension development assistance
- API usage guidance and examples
- Understanding VS Code's architecture and patterns
- Troubleshooting extension issues
- Learning best practices for VS Code integration

**Tool Category:** Documentation & Knowledge Base

## Tool Declaration

```typescript
{
	name: 'getVSCodeAPI',
	description: 'Query the VS Code API documentation knowledge base. Use this tool when you need information about VS Code extension development, API usage, extension patterns, or best practices. Returns relevant documentation excerpts and code examples.',
	inputSchema: {
		type: 'object',
		properties: {
			query: {
				type: 'string',
				description: 'The search query. Can be an API name, concept, or question about VS Code extension development.'
			},
			apiVersion: {
				type: 'string',
				description: 'VS Code API version to query (e.g., "1.85", "latest"). Defaults to latest.',
				default: 'latest'
			},
			includeExamples: {
				type: 'boolean',
				description: 'Whether to include code examples in the response',
				default: true
			}
		},
		required: ['query']
	}
}
```

## Input Parameters

### query (required)
- **Type:** `string`
- **Description:** Search query for VS Code API documentation
- **Examples:**
  - `"vscode.window.showQuickPick"` - Specific API
  - `"how to create a tree view"` - Conceptual question
  - `"extension activation events"` - Topic area
  - `"workspace folders API"` - Feature area

### apiVersion (optional)
- **Type:** `string`
- **Default:** `'latest'`
- **Description:** Which API version to query
- **Values:**
  - `'latest'` - Most recent stable API
  - `'1.85'` - Specific version
  - `'insiders'` - Pre-release API

### includeExamples (optional)
- **Type:** `boolean`
- **Default:** `true`
- **Description:** Whether to include code examples in results

### Examples

```typescript
// Query specific API
{
	query: 'vscode.window.createTreeView',
	includeExamples: true
}

// Conceptual query
{
	query: 'how to implement a language server',
	apiVersion: 'latest'
}

// Topic area
{
	query: 'webview communication patterns'
}

// Troubleshooting
{
	query: 'activation event not firing'
}
```

## Architecture

### High-Level Flow

```
User Query → getVSCodeAPI Tool
    ↓
Query Processing
    ↓
Knowledge Base Search
    ↓
Semantic Matching
    ↓
Result Ranking
    ↓
Format with Examples
    ↓
Return Documentation
```

### Component Integration

```
┌─────────────────────────────────────────┐
│      getVSCodeAPI Tool Handler          │
│  - Query parsing                        │
│  - Version selection                    │
└──────────────┬──────────────────────────┘
               ↓
┌─────────────────────────────────────────┐
│      Knowledge Base Service             │
│  - Documentation index                  │
│  - Semantic search                      │
│  - Result ranking                       │
└──────────────┬──────────────────────────┘
               ↓
┌─────────────────────────────────────────┐
│      Documentation Sources              │
│  - VS Code API reference                │
│  - Extension guides                     │
│  - Code samples repository              │
│  - Best practices docs                  │
└─────────────────────────────────────────┘
```

### Knowledge Base Structure

```
Knowledge Base
├── API Reference
│   ├── vscode.window.*
│   ├── vscode.workspace.*
│   ├── vscode.languages.*
│   └── vscode.commands.*
├── Extension Guides
│   ├── Getting Started
│   ├── Extension Anatomy
│   ├── Activation Events
│   └── Publishing
├── API Patterns
│   ├── Language Features
│   ├── Webviews
│   ├── Tree Views
│   └── Task Providers
└── Code Examples
    ├── Simple Extensions
    ├── Language Extensions
    └── Complex Patterns
```

## Internal Implementation

### Core Implementation Location

**File:** `src/extension/tools/builtinTools.ts`

### Implementation Code

```typescript
async function getVSCodeAPI(
	args: {
		query: string;
		apiVersion?: string;
		includeExamples?: boolean;
	},
	token: CancellationToken
): Promise<string> {
	const version = args.apiVersion || 'latest';
	const includeExamples = args.includeExamples !== false;

	// Get knowledge base service
	const kbService = getKnowledgeBaseService();

	// Search documentation
	const results = await kbService.search({
		query: args.query,
		version: version,
		includeExamples: includeExamples,
		maxResults: 5
	});

	if (results.length === 0) {
		return `No documentation found for "${args.query}". Try:\n- Using different keywords\n- Checking the query spelling\n- Searching for related APIs`;
	}

	// Format results
	return formatAPIDocumentation(results, includeExamples);
}
```

### Knowledge Base Service

```typescript
interface DocumentationEntry {
	id: string;
	title: string;
	namespace: string; // e.g., 'vscode.window'
	apiName?: string; // e.g., 'showQuickPick'
	description: string;
	signature?: string;
	parameters?: ParameterDoc[];
	returns?: string;
	examples?: CodeExample[];
	relatedAPIs?: string[];
	sinceVersion?: string;
	url: string;
}

interface CodeExample {
	title: string;
	code: string;
	language: string;
	description?: string;
}

class KnowledgeBaseService {
	private index: DocumentationIndex;

	async search(options: SearchOptions): Promise<DocumentationEntry[]> {
		// Parse query
		const queryInfo = this.parseQuery(options.query);

		// Perform semantic search
		const candidates = await this.semanticSearch(
			queryInfo,
			options.version
		);

		// Rank by relevance
		const ranked = this.rankResults(candidates, queryInfo);

		// Return top results
		return ranked.slice(0, options.maxResults || 5);
	}

	private parseQuery(query: string): QueryInfo {
		// Detect query type
		if (query.startsWith('vscode.')) {
			return {
				type: 'api',
				namespace: query.split('.').slice(0, -1).join('.'),
				apiName: query.split('.').pop()!
			};
		}

		if (query.includes('how to')) {
			return { type: 'howto', keywords: extractKeywords(query) };
		}

		return { type: 'general', keywords: extractKeywords(query) };
	}

	private async semanticSearch(
		queryInfo: QueryInfo,
		version: string
	): Promise<DocumentationEntry[]> {
		// Use embeddings for semantic search
		const embedding = await this.embedQuery(queryInfo);

		// Find similar documents
		return await this.index.findSimilar(embedding, {
			version: version,
			threshold: 0.7
		});
	}

	private rankResults(
		candidates: DocumentationEntry[],
		queryInfo: QueryInfo
	): DocumentationEntry[] {
		return candidates.sort((a, b) => {
			// Exact API name match scores highest
			if (queryInfo.type === 'api') {
				if (a.apiName === queryInfo.apiName) return -1;
				if (b.apiName === queryInfo.apiName) return 1;
			}

			// Then by semantic similarity
			return b.score - a.score;
		});
	}
}
```

### Documentation Formatting

```typescript
function formatAPIDocumentation(
	results: DocumentationEntry[],
	includeExamples: boolean
): string {
	const sections: string[] = [];

	sections.push(`# VS Code API Documentation\n`);
	sections.push(`Found ${results.length} relevant result(s):\n`);

	for (const result of results) {
		sections.push(formatEntry(result, includeExamples));
	}

	return sections.join('\n');
}

function formatEntry(
	entry: DocumentationEntry,
	includeExamples: boolean
): string {
	const parts: string[] = [];

	// Title and namespace
	parts.push(`## ${entry.namespace}.${entry.apiName || entry.title}`);

	// Description
	parts.push(entry.description);

	// Signature
	if (entry.signature) {
		parts.push(`\n### Signature\n\`\`\`typescript\n${entry.signature}\n\`\`\``);
	}

	// Parameters
	if (entry.parameters && entry.parameters.length > 0) {
		parts.push('\n### Parameters');
		for (const param of entry.parameters) {
			parts.push(`- **${param.name}** (\`${param.type}\`): ${param.description}`);
		}
	}

	// Returns
	if (entry.returns) {
		parts.push(`\n### Returns\n${entry.returns}`);
	}

	// Examples
	if (includeExamples && entry.examples && entry.examples.length > 0) {
		parts.push('\n### Examples');
		for (const example of entry.examples) {
			parts.push(`\n**${example.title}**`);
			if (example.description) {
				parts.push(example.description);
			}
			parts.push(`\`\`\`${example.language}\n${example.code}\n\`\`\``);
		}
	}

	// Related APIs
	if (entry.relatedAPIs && entry.relatedAPIs.length > 0) {
		parts.push(`\n### Related APIs\n${entry.relatedAPIs.map(api => `- ${api}`).join('\n')}`);
	}

	// Documentation link
	parts.push(`\n[→ Full documentation](${entry.url})`);

	return parts.join('\n');
}
```

### Caching Strategy

```typescript
class DocumentationCache {
	private cache: Map<string, CacheEntry> = new Map();
	private readonly TTL = 60 * 60 * 1000; // 1 hour

	get(query: string, version: string): DocumentationEntry[] | undefined {
		const key = `${query}:${version}`;
		const entry = this.cache.get(key);

		if (!entry) return undefined;

		// Check if expired
		if (Date.now() - entry.timestamp > this.TTL) {
			this.cache.delete(key);
			return undefined;
		}

		return entry.results;
	}

	set(
		query: string,
		version: string,
		results: DocumentationEntry[]
	): void {
		const key = `${query}:${version}`;
		this.cache.set(key, {
			results,
			timestamp: Date.now()
		});
	}
}
```

## Dependencies

### External Services

```typescript
// Documentation sources
const DOC_SOURCES = {
	apiReference: 'https://code.visualstudio.com/api/references/vscode-api',
	extensionGuides: 'https://code.visualstudio.com/api/extension-guides',
	samples: 'https://github.com/microsoft/vscode-extension-samples'
};
```

### Internal Dependencies

```typescript
import { ILanguageModelTool } from '../tools/toolsService';
import { CancellationToken } from '../../../platform/cancellation/common/cancellation';
import { IEmbeddingService } from '../../../platform/embedding/common/embedding';
```

### VS Code API Types

```typescript
// The tool queries documentation about these APIs
import * as vscode from 'vscode';

// Common namespaces documented:
// - vscode.window
// - vscode.workspace
// - vscode.languages
// - vscode.commands
// - vscode.extensions
// - vscode.debug
// - vscode.tasks
// - vscode.scm
// - vscode.comments
// - vscode.authentication
```

## Performance Characteristics

### Response Time
- **Cached queries:** 10-50ms
- **New queries:** 200-1000ms (includes semantic search)
- **Complex queries:** Up to 2000ms

### Search Performance

```typescript
// Performance optimization
const SEARCH_CONFIG = {
	maxResults: 5, // Limit to top results
	semanticThreshold: 0.7, // Similarity threshold
	cacheEnabled: true,
	cacheTTL: 60 * 60 * 1000 // 1 hour
};

async function optimizedSearch(query: string): Promise<DocumentationEntry[]> {
	// Check cache first
	const cached = cache.get(query);
	if (cached) return cached;

	// Perform search with timeout
	const results = await Promise.race([
		kbService.search({ query, maxResults: 5 }),
		timeout(2000)
	]);

	// Cache results
	cache.set(query, results);

	return results;
}
```

### Memory Usage
- **Knowledge base index:** 20-50 MB
- **Per-query cache entry:** 10-50 KB
- **Total cache (100 entries):** ~5 MB

## Use Cases

### Use Case 1: API Usage Query

**Scenario:** User needs to know how to use a specific VS Code API.

**Implementation:**
```typescript
// User asks about QuickPick
const result = await getVSCodeAPI({
	query: 'vscode.window.showQuickPick',
	includeExamples: true
});

// Returns documentation with examples
```

**Example Dialog:**
```
User: How do I show a quick pick menu?

Copilot: [calls getVSCodeAPI]
{ query: 'vscode.window.showQuickPick' }

Result:
## vscode.window.showQuickPick

Shows a selection list allowing multiple selections.

### Signature
```typescript
showQuickPick(items: string[] | Thenable<string[]>,
              options?: QuickPickOptions): Thenable<string | undefined>
```

### Parameters
- **items**: The items to pick from
- **options**: Configures the quick pick behavior

### Example

**Basic Usage**
```typescript
const result = await vscode.window.showQuickPick(
	['Option 1', 'Option 2', 'Option 3'],
	{ placeHolder: 'Select an option' }
);

if (result) {
	vscode.window.showInformationMessage(`Selected: ${result}`);
}
```

Let me show you how to implement this in your extension...
```

### Use Case 2: Conceptual Query

**Scenario:** User wants to understand a concept or pattern.

**Implementation:**
```typescript
// User asks about tree views
const result = await getVSCodeAPI({
	query: 'how to create a tree view',
	includeExamples: true
});
```

**Example Dialog:**
```
User: I want to create a tree view in my extension. How do I do that?

Copilot: [calls getVSCodeAPI]
{ query: 'how to create a tree view' }

Result shows:
- TreeDataProvider interface
- registerTreeDataProvider method
- Complete example with TreeItem

Based on the VS Code API documentation, here's how to create a tree view:

1. **Implement TreeDataProvider**
```typescript
class MyTreeDataProvider implements vscode.TreeDataProvider<MyTreeItem> {
	getTreeItem(element: MyTreeItem): vscode.TreeItem {
		return element;
	}

	getChildren(element?: MyTreeItem): Thenable<MyTreeItem[]> {
		// Return child items
	}
}
```

2. **Register the provider**
```typescript
vscode.window.registerTreeDataProvider('myView', new MyTreeDataProvider());
```

3. **Define your view in package.json**
```json
"views": {
	"explorer": [
		{
			"id": "myView",
			"name": "My Tree View"
		}
	]
}
```

Would you like me to help you implement this for your specific use case?
```

### Use Case 3: Troubleshooting

**Scenario:** User encounters an issue with their extension.

**Implementation:**
```typescript
// User has activation problem
const result = await getVSCodeAPI({
	query: 'extension activation events not working',
	includeExamples: false
});
```

**Example Dialog:**
```
User: My extension isn't activating. The activation event doesn't seem to work.

Copilot: [calls getVSCodeAPI]
{ query: 'extension activation events' }

Result shows activation event types and common issues.

Let me help troubleshoot your activation issue. According to the VS Code API:

**Common Activation Events:**
- `onLanguage:javascript` - Activates when JS file opens
- `onCommand:myextension.myCommand` - Activates when command runs
- `onView:myView` - Activates when view becomes visible
- `*` - Activates on startup (not recommended)

**Common Issues:**
1. **Event not declared in package.json**
   ```json
   "activationEvents": [
   	"onLanguage:javascript"
   ]
   ```

2. **Typo in event name**
   Make sure language ID matches exactly

3. **Command not registered**
   Ensure commands are registered in activate()

Can you share your package.json activationEvents section?
```

### Use Case 4: Best Practices Query

**Scenario:** User wants to know the right way to implement something.

**Implementation:**
```typescript
// User asks about best practices
const result = await getVSCodeAPI({
	query: 'webview communication best practices',
	includeExamples: true
});
```

**Example Dialog:**
```
User: What's the best way to communicate between a webview and extension?

Copilot: [calls getVSCodeAPI]
{ query: 'webview communication patterns' }

According to VS Code's webview documentation, here are the recommended patterns:

**1. Extension → Webview (postMessage)**
```typescript
// Extension side
panel.webview.postMessage({
	command: 'update',
	data: myData
});

// Webview side
window.addEventListener('message', event => {
	const message = event.data;
	switch (message.command) {
		case 'update':
			// Handle update
			break;
	}
});
```

**2. Webview → Extension (message handler)**
```typescript
// Webview side
vscode.postMessage({
	command: 'save',
	data: formData
});

// Extension side
panel.webview.onDidReceiveMessage(message => {
	switch (message.command) {
		case 'save':
			// Handle save
			break;
	}
});
```

**Best Practices:**
- Use typed message interfaces
- Implement error handling for failed messages
- Consider using a message broker pattern for complex communication

Would you like me to help implement this in your extension?
```

### Use Case 5: API Discovery

**Scenario:** User doesn't know which API to use for their task.

**Implementation:**
```typescript
// User has a goal but doesn't know the API
const result = await getVSCodeAPI({
	query: 'how to add custom editor',
	includeExamples: true
});
```

**Example Dialog:**
```
User: I want to create a custom editor for .data files

Copilot: [calls getVSCodeAPI]
{ query: 'custom editor API' }

For custom editors, VS Code provides the **Custom Editor API**:

**Two Types:**

1. **CustomTextEditor** - Text-based custom editor
   - Uses VS Code's text document
   - You control the UI via webview
   - Best for: Configuration files, data formats

2. **CustomEditor** - Fully custom binary editor
   - Complete control over model and UI
   - Best for: Images, binary formats

**For .data files, I recommend CustomTextEditor:**

```typescript
export function activate(context: vscode.ExtensionContext) {
	context.subscriptions.push(
		vscode.window.registerCustomEditorProvider(
			'myextension.dataEditor',
			new DataEditorProvider(),
			{ webviewOptions: { retainContextWhenHidden: true } }
		)
	);
}

class DataEditorProvider implements vscode.CustomTextEditorProvider {
	resolveCustomTextEditor(
		document: vscode.TextDocument,
		webviewPanel: vscode.WebviewPanel
	) {
		// Setup webview
		webviewPanel.webview.html = this.getHtmlForWebview();

		// Handle updates
		const updateWebview = () => {
			webviewPanel.webview.postMessage({
				type: 'update',
				content: document.getText()
			});
		};
	}
}
```

Would you like help implementing the complete editor?
```

## Comparison with Other Tools

### getVSCodeAPI vs. mcp_microsoftdocs_microsoft_docs_search

| Aspect | getVSCodeAPI | microsoft_docs_search |
|--------|--------------|----------------------|
| **Scope** | VS Code API only | All Microsoft docs |
| **Specialization** | Deep VS Code knowledge | Broad coverage |
| **Response Time** | 200-1000ms | 500-2000ms |
| **Examples** | Curated, extension-specific | General documentation |
| **Best For** | Extension development | General Microsoft tech |

### getVSCodeAPI vs. fetchWebPage

| Aspect | getVSCodeAPI | fetchWebPage |
|--------|--------------|--------------|
| **Data Source** | Curated knowledge base | Live web content |
| **Structure** | Structured API docs | Raw HTML/text |
| **Freshness** | Updated periodically | Real-time |
| **Best For** | Known API queries | Latest announcements |

### getVSCodeAPI vs. semantic_search (workspace)

| Aspect | getVSCodeAPI | semantic_search |
|--------|--------------|-----------------|
| **Purpose** | VS Code API docs | User's codebase |
| **Content** | Documentation | Source code |
| **Use Case** | Learning APIs | Finding implementations |
| **Best For** | "How to use X" | "Where did we use X" |

## Error Handling

### Common Error Scenarios

```typescript
try {
	const result = await getVSCodeAPI({ query: 'invalid_api' });
} catch (error) {
	// Handle various error conditions
}
```

### Error Cases

1. **No Results Found**
   ```typescript
   // Query: "nonexistent_api"
   // Returns: "No documentation found for 'nonexistent_api'..."
   ```

2. **Knowledge Base Unavailable**
   ```typescript
   // Network issue or service down
   // Returns: "Unable to access VS Code API documentation. Please try again."
   ```

3. **Invalid API Version**
   ```typescript
   { query: 'showQuickPick', apiVersion: '0.0.1' }
   // Returns: "API version '0.0.1' not found. Available: 1.80, 1.81, ..."
   ```

4. **Query Too Vague**
   ```typescript
   { query: 'extension' }
   // Returns: "Query too broad. Please be more specific. Try: 'extension activation', ..."
   ```

### Retry Logic

```typescript
async function getVSCodeAPIWithRetry(
	args: { query: string },
	maxRetries = 2
): Promise<string> {
	for (let attempt = 0; attempt <= maxRetries; attempt++) {
		try {
			return await getVSCodeAPI(args);
		} catch (error) {
			if (attempt === maxRetries) {
				return `Unable to fetch API documentation after ${maxRetries + 1} attempts. Error: ${error.message}`;
			}
			await new Promise(resolve => setTimeout(resolve, 1000 * (attempt + 1)));
		}
	}
}
```

## Configuration

### User Settings

```json
{
	// Enable/disable API documentation tool
	"github.copilot.tools.vscodeAPI.enabled": true,

	// Default API version
	"github.copilot.tools.vscodeAPI.defaultVersion": "latest",

	// Include examples by default
	"github.copilot.tools.vscodeAPI.includeExamples": true,

	// Maximum results to return
	"github.copilot.tools.vscodeAPI.maxResults": 5,

	// Cache duration (minutes)
	"github.copilot.tools.vscodeAPI.cacheDuration": 60
}
```

### Knowledge Base Updates

```typescript
// Periodic updates of documentation index
const UPDATE_CONFIG = {
	checkInterval: 24 * 60 * 60 * 1000, // 24 hours
	autoUpdate: true,
	sources: [
		'https://code.visualstudio.com/api/references/vscode-api',
		'https://github.com/microsoft/vscode-extension-samples'
	]
};
```

## Implementation Blueprint

### Step 1: Basic Query Handler

```typescript
async function basicGetVSCodeAPI(query: string): Promise<string> {
	// Simple keyword matching
	const docs = getBuiltInDocs();
	const matches = docs.filter(doc =>
		doc.title.toLowerCase().includes(query.toLowerCase()) ||
		doc.description.toLowerCase().includes(query.toLowerCase())
	);

	if (matches.length === 0) {
		return 'No documentation found';
	}

	return formatDocs(matches);
}
```

### Step 2: Add Semantic Search

```typescript
async function semanticGetVSCodeAPI(query: string): Promise<string> {
	// Generate query embedding
	const queryEmbedding = await embeddingService.embed(query);

	// Search index
	const results = await docIndex.findSimilar(queryEmbedding, {
		threshold: 0.7,
		maxResults: 5
	});

	return formatDocs(results);
}
```

### Step 3: Add Examples

```typescript
interface DocWithExamples extends DocumentationEntry {
	examples: CodeExample[];
}

function formatWithExamples(doc: DocWithExamples): string {
	let output = formatBasicDoc(doc);

	if (doc.examples.length > 0) {
		output += '\n\n### Examples\n';
		for (const example of doc.examples) {
			output += `\n**${example.title}**\n`;
			output += `\`\`\`${example.language}\n${example.code}\n\`\`\`\n`;
		}
	}

	return output;
}
```

### Step 4: Add Caching

```typescript
class CachedAPIDocService {
	private cache = new Map<string, CacheEntry>();

	async query(query: string): Promise<string> {
		// Check cache
		const cached = this.cache.get(query);
		if (cached && !this.isExpired(cached)) {
			return cached.result;
		}

		// Fetch fresh results
		const result = await this.fetchDocumentation(query);

		// Cache results
		this.cache.set(query, {
			result,
			timestamp: Date.now()
		});

		return result;
	}

	private isExpired(entry: CacheEntry): boolean {
		const TTL = 60 * 60 * 1000; // 1 hour
		return Date.now() - entry.timestamp > TTL;
	}
}
```

### Step 5: Add Version Support

```typescript
class VersionedDocService {
	private versionedDocs: Map<string, DocumentationEntry[]> = new Map();

	async query(query: string, version: string): Promise<string> {
		// Get docs for specific version
		const docs = this.versionedDocs.get(version) ||
		             this.versionedDocs.get('latest')!;

		// Search within version
		const results = await this.search(query, docs);

		return this.format(results, version);
	}

	async loadVersion(version: string): Promise<void> {
		const docs = await this.fetchDocsForVersion(version);
		this.versionedDocs.set(version, docs);
	}
}
```

## Best Practices

### 1. Be Specific in Queries

```typescript
// ✓ Good: Specific API query
{ query: 'vscode.window.showQuickPick' }
{ query: 'how to create a tree view provider' }

// ✗ Bad: Too vague
{ query: 'window' }
{ query: 'extension' }
```

### 2. Use Conceptual Queries for Learning

```typescript
// ✓ Good: Learning queries
{ query: 'how to implement code completion' }
{ query: 'webview communication patterns' }
{ query: 'language server protocol integration' }

// ✗ Bad: Too specific without context
{ query: 'CompletionItem' } // Better: 'how to provide completions'
```

### 3. Request Examples When Helpful

```typescript
// ✓ Good: Include examples for implementation
{ query: 'custom editor API', includeExamples: true }

// ✗ Bad: Skip examples when you need implementation help
{ query: 'how to create webview', includeExamples: false }
```

### 4. Combine with Code Search

```typescript
// ✓ Good: Use both tools
const apiDocs = await getVSCodeAPI({ query: 'TreeDataProvider' });
const examples = await semantic_search({ query: 'TreeDataProvider implementation' });

// Copilot now has both API docs and real code examples
```

### 5. Handle "No Results" Gracefully

```typescript
// ✓ Good: Provide fallback suggestions
const result = await getVSCodeAPI({ query: userQuery });
if (result.includes('No documentation found')) {
	// Try alternative query
	const alternative = await getVSCodeAPI({
		query: reformulateQuery(userQuery)
	});
}
```

## Limitations

### 1. Documentation Only

**Limitation:** Provides documentation, not actual code inspection.

```typescript
// Can answer: "What parameters does showQuickPick take?"
// Can't answer: "What's the implementation of showQuickPick?"
```

**Workaround:** Combine with source code inspection tools.

### 2. Periodic Updates

**Limitation:** Knowledge base updated periodically, may lag latest VS Code.

```typescript
// May not have: Features from today's Insiders build
// Will have: Stable API from released versions
```

**Workaround:** Use fetchWebPage for bleeding-edge documentation.

### 3. No Real-Time Testing

**Limitation:** Can't test or validate API suggestions.

```typescript
// Provides: Documentation and examples
// Can't: Run code to verify it works
```

**Workaround:** User must test suggested code.

### 4. Limited to VS Code API

**Limitation:** Only covers VS Code extension API, not general TypeScript/Node.js.

```typescript
// Covers: vscode.window.showQuickPick
// Doesn't cover: How to use Express.js or React
```

**Workaround:** Use other documentation tools for non-VS Code topics.

### 5. No Custom Extension APIs

**Limitation:** Only documents official VS Code API, not third-party extensions.

```typescript
// Covers: Official VS Code API
// Doesn't cover: Custom APIs from user's extensions
```

**Workaround:** Use workspace search for user's own code.

## Summary

The `getVSCodeAPI` tool is essential for VS Code extension development assistance. It provides:

- **Quick access** to VS Code API documentation
- **Code examples** for common patterns
- **Conceptual guidance** for extension development
- **Best practices** and recommended patterns
- **Version-specific** documentation

**Key Strengths:**
- Curated, high-quality documentation
- Includes practical code examples
- Semantic search for conceptual queries
- Cached for fast repeated queries
- Version-aware documentation

**Best Used For:**
- Extension development questions
- API usage examples
- Learning VS Code extension patterns
- Troubleshooting extension issues
- Discovering the right API for a task

By leveraging this tool effectively, Copilot can provide expert-level guidance on VS Code extension development without requiring external documentation lookups.
