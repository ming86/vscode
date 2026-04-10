# copilot_findFiles Tool Documentation

## Overview

**Tool Name:** `copilot_findFiles`

**Purpose:** Searches for files in the workspace by glob pattern. Returns file paths matching specified patterns, enabling file discovery, navigation, and workspace structure analysis. Essential for locating configuration files, finding files by type, or exploring directory structures.

**Implementation Location:** `src/extension/tools/codeSearch/findFilesTool.ts`

**Tool Class:** `FindFilesTool`

## Tool Declaration

```json
{
	"type": "function",
	"function": {
		"name": "copilot_findFiles",
		"description": "Search for files in the workspace by glob pattern. This only returns the paths of matching files. Use this tool when you know the exact filename pattern of the files you're searching for. Glob patterns match from the root of the workspace folder. Examples:\n- **/*.{js,ts} to match all js/ts files in the workspace.\n- src/** to match all files under the top-level src folder.\n- **/foo/**/*.js to match all js files under any foo folder in the workspace.",
		"parameters": {
			"type": "object",
			"properties": {
				"query": {
					"type": "string",
					"description": "Search for files with names or paths matching this glob pattern."
				},
				"maxResults": {
					"type": "number",
					"description": "The maximum number of results to return. Do not use this unless necessary, it can slow things down. By default, only some matches are returned. If you use this and don't see what you're looking for, you can try again with a more specific query or a larger maxResults."
				}
			},
			"required": [
				"query"
			]
		}
	}
}
```

## Input Parameters

### TypeScript Interface

```typescript
interface FindFilesInput {
	query: string;           // Required: Glob pattern to match files
	maxResults?: number;     // Optional: Maximum number of results (default varies)
}
```

### Parameter Details

| Parameter | Type | Required | Default | Description |
|-----------|------|----------|---------|-------------|
| `query` | string | Yes | - | Glob pattern to match file paths (e.g., `**/*.ts`, `src/**/test/*.js`) |
| `maxResults` | number | No | 20-100 | Maximum number of file paths to return. Higher values may impact performance |

### Glob Pattern Syntax

**Basic Patterns:**
```typescript
// Exact filename
"package.json"

// Wildcard
"*.ts"              // All TypeScript files in root
"*.{js,ts}"         // All JS or TS files in root

// Double star (recursive)
"**/*.ts"           // All TypeScript files anywhere
"src/**/*.test.ts"  // All test files under src/

// Directory patterns
"src/**"            // All files under src/
"**/test/**"        // All files in any test/ directory

// Negation
"**/*.ts"           // Find: All TS files
"!**/*.test.ts"     // (negation requires separate query)

// Character classes
"src/[abc]*.ts"     // Files starting with a, b, or c
"data/file[0-9].json" // file0.json, file1.json, etc.
```

### Parameter Examples

```typescript
// Find all TypeScript files
{
	"query": "**/*.ts"
}

// Find configuration files
{
	"query": "**/tsconfig*.json"
}

// Find test files with limit
{
	"query": "**/*.test.ts",
	"maxResults": 50
}

// Find files in specific directory
{
	"query": "src/components/**/*.tsx"
}

// Find multiple file types
{
	"query": "**/*.{js,jsx,ts,tsx}"
}
```

## Architecture

### Flow Diagram

```
┌─────────────────────────────────────────────────────────────┐
│                      copilot_findFiles                      │
│                      (FindFilesTool)                        │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
                    ┌─────────────────┐
                    │ Validate Input  │
                    │ - Parse pattern │
                    │ - Check syntax  │
                    └─────────────────┘
                              │
                              ▼
                ┌─────────────────────────────┐
                │ Prepare Search Parameters   │
                │ - Normalize glob pattern    │
                │ - Apply maxResults limit    │
                │ - Set search options        │
                └─────────────────────────────┘
                              │
                              ▼
                ┌─────────────────────────────┐
                │ Execute File Search         │
                │ - Use IFileService          │
                │ - Match against pattern     │
                │ - Apply exclude filters     │
                └─────────────────────────────┘
                              │
                              ▼
                ┌─────────────────────────────┐
                │ Filter & Sort Results       │
                │ - Remove excluded paths     │
                │ - Sort by relevance/path    │
                │ - Apply maxResults limit    │
                └─────────────────────────────┘
                              │
                              ▼
                ┌─────────────────────────────┐
                │ Convert to Relative Paths   │
                │ - Strip workspace root      │
                │ - Normalize separators      │
                │ - Format consistently       │
                └─────────────────────────────┘
                              │
                              ▼
                ┌─────────────────────────────┐
                │ Format Results              │
                │ - Create file path list     │
                │ - Add metadata if needed    │
                │ - Apply formatting          │
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
         │ Request: query, maxResults?
         ▼
┌──────────────────────────────────┐
│       FindFilesTool              │
│  - validateGlobPattern()         │
│  - searchFiles()                 │
│  - filterResults()               │
│  - formatOutput()                │
└────────┬─────────────────────────┘
         │
         ├─────────────────┐
         │                 │
         ▼                 ▼
┌──────────────────┐  ┌──────────────────┐
│  IFileService    │  │ IWorkspaceContext│
│  - findFiles()   │  │  - rootPath      │
│  - stat()        │  │  - excludes      │
└──────────────────┘  └──────────────────┘
         │
         ▼
┌──────────────────┐
│  Glob Matcher    │
│  - minimatch     │
│  - picomatch     │
└──────────────────┘
```

## Internal Implementation

### Core Implementation

```typescript
export class FindFilesTool extends LanguageModelTool<FindFilesInput> {
	static readonly ID = 'copilot_findFiles';

	constructor(
		@IFileService private readonly fileService: IFileService,
		@IWorkspaceContextService private readonly workspaceService: IWorkspaceContextService,
		@IConfigurationService private readonly configService: IConfigurationService
	) {
		super();
	}

	async invoke(
		input: FindFilesInput,
		token: CancellationToken
	): Promise<LanguageModelToolResult> {
		const { query, maxResults } = input;

		// Step 1: Validate glob pattern
		if (!this.isValidGlobPattern(query)) {
			return {
				content: [{
					kind: 'text',
					value: `Invalid glob pattern: "${query}"`
				}]
			};
		}

		// Step 2: Get workspace folders
		const workspaceFolders = this.workspaceService.getWorkspace().folders;
		if (workspaceFolders.length === 0) {
			return {
				content: [{
					kind: 'text',
					value: 'No workspace folder open'
				}]
			};
		}

		// Step 3: Search for files
		const files = await this.searchFiles(
			query,
			workspaceFolders,
			maxResults,
			token
		);

		// Step 4: Format and return results
		if (files.length === 0) {
			return {
				content: [{
					kind: 'text',
					value: `No files found matching pattern: "${query}"`
				}]
			};
		}

		const markdown = this.formatResults(query, files, maxResults);

		return {
			content: [{
				kind: 'text',
				value: markdown
			}]
		};
	}

	private isValidGlobPattern(pattern: string): boolean {
		// Check for invalid characters or malformed patterns
		if (!pattern || pattern.trim().length === 0) {
			return false;
		}

		// Basic validation
		try {
			// Test pattern compilation
			new RegExp(pattern.replace(/\*/g, '.*'));
			return true;
		} catch {
			return false;
		}
	}

	private async searchFiles(
		pattern: string,
		workspaceFolders: readonly IWorkspaceFolder[],
		maxResults: number | undefined,
		token: CancellationToken
	): Promise<URI[]> {
		const defaultMaxResults = 100;
		const limit = maxResults ?? defaultMaxResults;

		// Get exclude patterns from settings
		const excludePatterns = this.getExcludePatterns();

		const allFiles: URI[] = [];

		for (const folder of workspaceFolders) {
			if (token.isCancellationRequested) {
				break;
			}

			// Search in this workspace folder
			const files = await this.fileService.findFiles(
				folder.uri,
				{
					includePattern: pattern,
					excludePattern: excludePatterns,
					maxResults: limit - allFiles.length,
					useDefaultExcludes: true
				},
				token
			);

			allFiles.push(...files);

			if (allFiles.length >= limit) {
				break;
			}
		}

		return allFiles;
	}

	private getExcludePatterns(): string[] {
		// Get files.exclude and search.exclude settings
		const filesExclude = this.configService.getValue<Record<string, boolean>>('files.exclude') ?? {};
		const searchExclude = this.configService.getValue<Record<string, boolean>>('search.exclude') ?? {};

		const patterns: string[] = [];

		// Combine exclude patterns
		for (const [pattern, enabled] of Object.entries(filesExclude)) {
			if (enabled) {
				patterns.push(pattern);
			}
		}

		for (const [pattern, enabled] of Object.entries(searchExclude)) {
			if (enabled && !patterns.includes(pattern)) {
				patterns.push(pattern);
			}
		}

		return patterns;
	}

	private formatResults(
		query: string,
		files: URI[],
		maxResults: number | undefined
	): string {
		const lines: string[] = [
			`# Files matching \`${query}\``,
			''
		];

		// Add result count
		const hasMore = maxResults && files.length >= maxResults;
		if (hasMore) {
			lines.push(`Found ${files.length}+ files (showing first ${files.length}):`);
		} else {
			lines.push(`Found ${files.length} file(s):`);
		}
		lines.push('');

		// Get workspace root for relative paths
		const workspaceRoot = this.workspaceService.getWorkspace().folders[0]?.uri;

		// List files
		for (const file of files) {
			const relativePath = workspaceRoot 
				? this.getRelativePath(workspaceRoot, file)
				: file.fsPath;
			lines.push(`- ${relativePath}`);
		}

		if (hasMore) {
			lines.push('');
			lines.push('*Use a more specific pattern or increase maxResults to see more files*');
		}

		return lines.join('\n');
	}

	private getRelativePath(root: URI, file: URI): string {
		const rootPath = root.fsPath;
		const filePath = file.fsPath;

		if (filePath.startsWith(rootPath)) {
			return filePath.substring(rootPath.length + 1);
		}

		return filePath;
	}
}
```

### Advanced Pattern Matching

```typescript
class GlobPatternMatcher {
	constructor(private readonly pattern: string) {}

	matches(filePath: string): boolean {
		// Convert glob to regex
		const regex = this.globToRegex(this.pattern);
		return regex.test(filePath);
	}

	private globToRegex(glob: string): RegExp {
		let pattern = glob;

		// Escape special regex characters except glob wildcards
		pattern = pattern.replace(/[.+^${}()|[\]\\]/g, '\\$&');

		// Convert glob patterns to regex
		pattern = pattern.replace(/\*\*/g, '§DOUBLESTAR§');
		pattern = pattern.replace(/\*/g, '[^/]*');
		pattern = pattern.replace(/§DOUBLESTAR§/g, '.*');
		pattern = pattern.replace(/\?/g, '[^/]');

		// Handle {a,b} alternatives
		pattern = pattern.replace(/\{([^}]+)\}/g, (_, alternatives) => {
			return `(${alternatives.split(',').join('|')})`;
		});

		return new RegExp(`^${pattern}$`);
	}
}
```

### Optimization with Caching

```typescript
class FindFilesToolWithCache extends FindFilesTool {
	private readonly cache = new Map<string, CacheEntry>();
	private readonly CACHE_TTL = 5000; // 5 seconds

	async invoke(
		input: FindFilesInput,
		token: CancellationToken
	): Promise<LanguageModelToolResult> {
		const cacheKey = this.getCacheKey(input);
		const cached = this.cache.get(cacheKey);

		if (cached && !this.isCacheExpired(cached)) {
			return cached.result;
		}

		const result = await super.invoke(input, token);

		this.cache.set(cacheKey, {
			result,
			timestamp: Date.now()
		});

		return result;
	}

	private getCacheKey(input: FindFilesInput): string {
		return `${input.query}:${input.maxResults ?? 'default'}`;
	}

	private isCacheExpired(entry: CacheEntry): boolean {
		return Date.now() - entry.timestamp > this.CACHE_TTL;
	}
}

interface CacheEntry {
	result: LanguageModelToolResult;
	timestamp: number;
}
```

## Dependencies

### Service Dependencies

```typescript
// Core VS Code services required
@IFileService                    // File system operations and glob matching
@IWorkspaceContextService        // Workspace folder information
@IConfigurationService           // Access to exclude settings
@IUriIdentityService (optional)  // URI comparison and normalization
```

### File Service Methods

```typescript
interface IFileService {
	/**
	 * Find files matching patterns
	 */
	findFiles(
		root: URI,
		options: {
			includePattern?: string | string[];
			excludePattern?: string | string[];
			maxResults?: number;
			useDefaultExcludes?: boolean;
		},
		token: CancellationToken
	): Promise<URI[]>;

	/**
	 * Get file stats
	 */
	stat(uri: URI): Promise<IFileStatWithMetadata>;
}
```

### Configuration Keys

```typescript
// Relevant VS Code settings
const configKeys = {
	filesExclude: 'files.exclude',       // File Explorer exclusions
	searchExclude: 'search.exclude',     // Search exclusions
	watcherExclude: 'files.watcherExclude' // File watcher exclusions
};

// Example settings
{
	"files.exclude": {
		"**/.git": true,
		"**/node_modules": true,
		"**/.DS_Store": true
	},
	"search.exclude": {
		"**/dist": true,
		"**/build": true,
		"**/*.min.js": true
	}
}
```

## Performance Characteristics

### Time Complexity

| Operation | Complexity | Notes |
|-----------|------------|-------|
| Pattern compilation | O(p) | p = pattern length |
| File tree traversal | O(n) | n = total files in workspace |
| Pattern matching per file | O(p × f) | f = file path length |
| Overall search | O(n × p × f) | Can be optimized with indexing |

### Performance Metrics

```typescript
// Benchmark results
const performanceData = {
	small: {
		workspace: '<1K files',
		pattern: '**/*.ts',
		time: '<100ms',
		memory: '<5MB'
	},
	medium: {
		workspace: '1K-10K files',
		pattern: '**/*.{js,ts}',
		time: '100-500ms',
		memory: '5-20MB'
	},
	large: {
		workspace: '10K-100K files',
		pattern: 'src/**/*.tsx',
		time: '500ms-3s',
		memory: '20-100MB'
	},
	veryLarge: {
		workspace: '>100K files',
		pattern: '**/*',
		time: '3s+',
		memory: '100MB+'
	}
};
```

### Optimization Strategies

**1. Use Specific Patterns**

```typescript
// ❌ Slow - matches everything
{
	"query": "**/*"
}

// ✅ Fast - specific pattern
{
	"query": "src/components/**/*.tsx"
}
```

**2. Limit Results Early**

```typescript
// ❌ Fetches all, then limits
async function searchFiles(pattern: string): Promise<URI[]> {
	const all = await findAllFiles(pattern);
	return all.slice(0, 100);
}

// ✅ Limits during search
async function searchFiles(pattern: string, limit: number): Promise<URI[]> {
	const results: URI[] = [];
	for await (const file of iterateFiles(pattern)) {
		results.push(file);
		if (results.length >= limit) break;
	}
	return results;
}
```

**3. Leverage Excludes**

```typescript
// Respect default excludes
const options = {
	includePattern: query,
	excludePattern: [
		'**/node_modules/**',
		'**/dist/**',
		'**/.git/**'
	],
	useDefaultExcludes: true  // Use VS Code's default excludes
};
```

**4. Parallel Multi-Folder Search**

```typescript
async function searchMultipleFolders(
	pattern: string,
	folders: IWorkspaceFolder[]
): Promise<URI[]> {
	// Search folders in parallel
	const searches = folders.map(folder =>
		this.searchInFolder(folder, pattern)
	);

	const results = await Promise.all(searches);
	return results.flat();
}
```

### Memory Management

```typescript
// Stream results to avoid loading all files in memory
async function* streamFileMatches(
	pattern: string
): AsyncGenerator<URI> {
	for await (const file of walkFileTree()) {
		if (matchesPattern(file, pattern)) {
			yield file;
		}
	}
}

// Usage
const results: URI[] = [];
for await (const file of streamFileMatches('**/*.ts')) {
	results.push(file);
	if (results.length >= maxResults) break;
}
```

## Use Cases

### 1. Find Configuration Files

**Scenario:** Locate all TypeScript configuration files.

```typescript
// Query
{
	"query": "**/tsconfig*.json"
}

// Output
# Files matching `**/tsconfig*.json`

Found 5 file(s):

- tsconfig.json
- tsconfig.base.json
- src/tsconfig.json
- test/tsconfig.json
- packages/core/tsconfig.json
```

### 2. Locate Test Files

**Scenario:** Find all test files in a specific directory.

```typescript
// Query
{
	"query": "src/**/*.test.ts",
	"maxResults": 50
}

// Output
# Files matching `src/**/*.test.ts`

Found 23 file(s):

- src/utils/helpers.test.ts
- src/services/auth.test.ts
- src/components/Button.test.ts
...
```

### 3. Find Files by Extension

**Scenario:** List all React components.

```typescript
// Query
{
	"query": "**/*.{jsx,tsx}"
}

// Output lists all JSX/TSX files
```

### 4. Explore Directory Structure

**Scenario:** List all files in a specific folder.

```typescript
// Query
{
	"query": "docs/**"
}

// Shows all files under docs/
```

### 5. Find Multiple File Types

**Scenario:** Locate all JavaScript and TypeScript files.

```typescript
// Query
{
	"query": "src/**/*.{js,jsx,ts,tsx}"
}

// Returns all JS/TS source files
```

### 6. Find Files Matching Name Pattern

**Scenario:** Find all files starting with "index".

```typescript
// Query
{
	"query": "**/index.*"
}

// Output
# Files matching `**/index.*`

Found 12 file(s):

- index.ts
- src/index.ts
- src/components/index.ts
- src/utils/index.ts
...
```

### 7. Locate Build Artifacts

**Scenario:** Find all built JavaScript files.

```typescript
// Query
{
	"query": "dist/**/*.js"
}

// Lists compiled output files
```

### 8. Find Package Files

**Scenario:** Locate all package.json files in a monorepo.

```typescript
// Query
{
	"query": "**/package.json"
}

// Output
# Files matching `**/package.json`

Found 15 file(s):

- package.json
- packages/core/package.json
- packages/utils/package.json
- packages/ui/package.json
...
```

## Comparison with Other Tools

### vs copilot_searchCodebase

| Aspect | findFiles | searchCodebase |
|--------|-----------|----------------|
| **Search Target** | File paths/names | File content |
| **Search Method** | Glob patterns | Text/semantic search |
| **Results** | File paths only | Content chunks |
| **Speed** | Very fast | Moderate |
| **Use Case** | File discovery | Content discovery |
| **Precision** | Exact pattern match | Fuzzy/semantic match |

**When to use findFiles:**
- Finding files by name/path pattern
- Listing files of specific type
- Exploring directory structure
- Quick file lookups

**When to use searchCodebase:**
- Finding code by content
- Searching for implementations
- Understanding code patterns
- Content-based discovery

### vs copilot_findTextInFiles

| Aspect | findFiles | findTextInFiles |
|--------|-----------|-----------------|
| **Search Basis** | Filename/path | File content |
| **Pattern Type** | Glob patterns | Text/regex |
| **Output** | File list | Content matches |
| **Performance** | Fastest | Slower |
| **Context** | No content | With content snippets |

**Example comparison:**

```typescript
// findFiles - returns paths
{
	"query": "**/*.config.js"
}
// Result: List of config file paths

// findTextInFiles - returns content
{
	"query": "webpack.config",
	"isRegexp": false
}
// Result: Files containing "webpack.config" with context
```

### vs copilot_readFile

| Aspect | findFiles | readFile |
|--------|-----------|----------|
| **Purpose** | Discover files | Read file content |
| **Input** | Pattern | Specific path |
| **Output** | Multiple paths | Single file content |
| **Usage** | Discovery phase | Access phase |

**Typical workflow:**

```typescript
// Step 1: Find files
findFiles({ query: "**/*.config.ts" })
// Result: [config/app.config.ts, config/db.config.ts, ...]

// Step 2: Read specific file
readFile({ filePath: "config/app.config.ts" })
// Result: File content
```

### vs copilot_listDirectory

| Aspect | findFiles | listDirectory |
|--------|-----------|---------------|
| **Scope** | Recursive (workspace) | Single directory |
| **Pattern** | Glob patterns | No filtering |
| **Depth** | Any depth | One level |
| **Output** | Filtered list | Full directory listing |

**When to use each:**

```typescript
// findFiles - pattern-based, recursive
{
	"query": "src/**/test/*.ts"
}
// Finds all test .ts files under any src/ subdirectory

// listDirectory - complete listing, single level
{
	"path": "/workspace/src"
}
// Lists all items directly in src/ (files and folders)
```

## Error Handling

### Common Error Scenarios

**1. Invalid Glob Pattern**

```typescript
// Input
{
	"query": "[invalid"  // Unclosed bracket
}

// Output
Invalid glob pattern: "[invalid"

// Validation
private isValidGlobPattern(pattern: string): boolean {
	try {
		// Test pattern compilation
		minimatch.makeRe(pattern);
		return true;
	} catch {
		return false;
	}
}
```

**2. No Workspace Folder**

```typescript
// Input
{
	"query": "**/*.ts"
}

// Output (when no folder open)
No workspace folder open

// Check
if (workspaceFolders.length === 0) {
	return noWorkspaceError();
}
```

**3. No Matching Files**

```typescript
// Input
{
	"query": "**/*.xyz"  // Non-existent extension
}

// Output
No files found matching pattern: "**/*.xyz"

// This is not an error, just empty results
```

**4. Pattern Too Broad**

```typescript
// Input
{
	"query": "**/*",      // Matches everything
	"maxResults": 10
}

// Output (with warning)
Found 10+ files (showing first 10):
...

*Use a more specific pattern or increase maxResults to see more files*
```

**5. Permission Denied**

```typescript
// Scenario: No read access to directory

// Error handling
try {
	const files = await this.fileService.findFiles(root, options, token);
	return files;
} catch (error) {
	if (error.code === 'EACCES') {
		return {
			content: [{
				kind: 'text',
				value: `Permission denied accessing: ${root.fsPath}`
			}]
		};
	}
	throw error;
}
```

### Error Recovery Strategies

```typescript
class RobustFindFilesTool extends FindFilesTool {
	async invoke(
		input: FindFilesInput,
		token: CancellationToken
	): Promise<LanguageModelToolResult> {
		try {
			return await super.invoke(input, token);
		} catch (error) {
			// Try to provide helpful fallback
			return this.handleError(error, input);
		}
	}

	private async handleError(
		error: Error,
		input: FindFilesInput
	): Promise<LanguageModelToolResult> {
		// Pattern too complex
		if (error.message.includes('pattern')) {
			return {
				content: [{
					kind: 'text',
					value: `Pattern error: Try simplifying the glob pattern.\n` +
					       `Example: "src/**/*.ts" instead of complex patterns`
				}]
			};
		}

		// Timeout
		if (error.message.includes('timeout')) {
			return {
				content: [{
					kind: 'text',
					value: `Search timeout. Try:\n` +
					       `- Using a more specific pattern\n` +
					       `- Reducing maxResults\n` +
					       `- Searching in a subdirectory`
				}]
			};
		}

		// Generic error
		return {
			content: [{
				kind: 'text',
				value: `Error searching files: ${error.message}`
			}]
		};
	}
}
```

### Timeout Protection

```typescript
const SEARCH_TIMEOUT_MS = 10000; // 10 seconds

async function searchWithTimeout(
	pattern: string,
	token: CancellationToken
): Promise<URI[]> {
	const timeoutToken = new CancellationTokenSource();
	const combinedToken = CancellationToken.combine(token, timeoutToken.token);

	const timeoutHandle = setTimeout(() => {
		timeoutToken.cancel();
	}, SEARCH_TIMEOUT_MS);

	try {
		return await this.searchFiles(pattern, combinedToken);
	} finally {
		clearTimeout(timeoutHandle);
		timeoutToken.dispose();
	}
}
```

## Configuration

### Tool Settings

```json
{
	// Maximum results to return by default
	"copilot.tools.findFiles.defaultMaxResults": 100,
	
	// Timeout for file search (ms)
	"copilot.tools.findFiles.timeout": 10000,
	
	// Use default VS Code excludes
	"copilot.tools.findFiles.useDefaultExcludes": true,
	
	// Additional exclude patterns
	"copilot.tools.findFiles.exclude": [
		"**/node_modules/**",
		"**/dist/**",
		"**/.git/**"
	],
	
	// Enable pattern validation
	"copilot.tools.findFiles.validatePatterns": true,
	
	// Cache search results
	"copilot.tools.findFiles.cacheResults": true,
	"copilot.tools.findFiles.cacheTTL": 5000
}
```

### VS Code File Settings

```json
{
	// Files to exclude from workspace
	"files.exclude": {
		"**/.git": true,
		"**/.svn": true,
		"**/.hg": true,
		"**/CVS": true,
		"**/.DS_Store": true,
		"**/Thumbs.db": true,
		"**/node_modules": true
	},
	
	// Files to exclude from search
	"search.exclude": {
		"**/node_modules": true,
		"**/bower_components": true,
		"**/*.code-search": true,
		"**/dist": true,
		"**/build": true
	},
	
	// Files to exclude from file watcher
	"files.watcherExclude": {
		"**/.git/objects/**": true,
		"**/.git/subtree-cache/**": true,
		"**/node_modules/*/**": true,
		"**/.hg/store/**": true
	}
}
```

### Programmatic Configuration

```typescript
// Configure tool instance
const config: FindFilesConfig = {
	defaultMaxResults: 50,
	timeout: 5000,
	excludePatterns: [
		'**/test/**',
		'**/*.test.ts'
	],
	validatePatterns: true
};

const tool = new FindFilesTool(
	fileService,
	workspaceService,
	configService,
	config
);

// Override per-invocation
const result = await tool.invoke(
	{
		query: '**/*.ts',
		maxResults: 200  // Override default
	},
	token
);
```

## Implementation Blueprint

### Step-by-Step Implementation Guide

**Phase 1: Basic Tool Structure**

```typescript
// 1. Create tool class
export class FindFilesTool extends LanguageModelTool<FindFilesInput> {
	static readonly ID = 'copilot_findFiles';

	constructor(
		@IFileService private readonly fileService: IFileService,
		@IWorkspaceContextService private readonly workspaceService: IWorkspaceContextService
	) {
		super();
	}

	async invoke(
		input: FindFilesInput,
		token: CancellationToken
	): Promise<LanguageModelToolResult> {
		// Implementation
	}
}

// 2. Define input interface
interface FindFilesInput {
	query: string;
	maxResults?: number;
}

// 3. Register tool
registerLanguageModelTool(FindFilesTool.ID, FindFilesTool);
```

**Phase 2: Pattern Validation**

```typescript
private validatePattern(pattern: string): ValidationResult {
	// Check for empty pattern
	if (!pattern || pattern.trim().length === 0) {
		return {
			valid: false,
			error: 'Pattern cannot be empty'
		};
	}

	// Check for invalid characters
	const invalidChars = /[<>:"|]/;
	if (invalidChars.test(pattern)) {
		return {
			valid: false,
			error: 'Pattern contains invalid characters'
		};
	}

	// Test pattern compilation
	try {
		minimatch.makeRe(pattern);
		return { valid: true };
	} catch (error) {
		return {
			valid: false,
			error: `Invalid glob pattern: ${error.message}`
		};
	}
}

interface ValidationResult {
	valid: boolean;
	error?: string;
}
```

**Phase 3: File Search Implementation**

```typescript
private async searchFiles(
	pattern: string,
	maxResults: number,
	token: CancellationToken
): Promise<URI[]> {
	const workspaceFolders = this.workspaceService.getWorkspace().folders;
	const allResults: URI[] = [];

	for (const folder of workspaceFolders) {
		if (token.isCancellationRequested) {
			break;
		}

		// Search in this folder
		const results = await this.fileService.findFiles(
			folder.uri,
			{
				includePattern: pattern,
				excludePattern: this.getExcludePatterns(),
				maxResults: maxResults - allResults.length,
				useDefaultExcludes: true
			},
			token
		);

		allResults.push(...results);

		if (allResults.length >= maxResults) {
			break;
		}
	}

	return allResults;
}
```

**Phase 4: Result Formatting**

```typescript
private formatResults(
	pattern: string,
	files: URI[],
	maxResults?: number
): string {
	const lines: string[] = [];

	// Header
	lines.push(`# Files matching \`${pattern}\``);
	lines.push('');

	// Count
	const truncated = maxResults && files.length >= maxResults;
	if (truncated) {
		lines.push(`Found ${files.length}+ files (showing first ${files.length}):`);
	} else {
		lines.push(`Found ${files.length} file(s):`);
	}
	lines.push('');

	// File list
	const root = this.workspaceService.getWorkspace().folders[0]?.uri;
	for (const file of files) {
		const relativePath = this.toRelativePath(root, file);
		lines.push(`- ${relativePath}`);
	}

	// Truncation notice
	if (truncated) {
		lines.push('');
		lines.push('*Use maxResults parameter to see more files*');
	}

	return lines.join('\n');
}

private toRelativePath(root: URI | undefined, file: URI): string {
	if (!root) {
		return file.fsPath;
	}

	const rootPath = root.fsPath + '/';
	const filePath = file.fsPath;

	if (filePath.startsWith(rootPath)) {
		return filePath.substring(rootPath.length);
	}

	return filePath;
}
```

**Phase 5: Testing**

```typescript
describe('FindFilesTool', () => {
	let tool: FindFilesTool;
	let fileService: IFileService;
	let workspaceService: IWorkspaceContextService;

	beforeEach(() => {
		fileService = createMockFileService();
		workspaceService = createMockWorkspaceService();
		tool = new FindFilesTool(fileService, workspaceService);
	});

	test('finds TypeScript files', async () => {
		const result = await tool.invoke(
			{ query: '**/*.ts' },
			CancellationToken.None
		);

		expect(result.content[0].value).toContain('.ts');
	});

	test('respects maxResults', async () => {
		const result = await tool.invoke(
			{ query: '**/*', maxResults: 10 },
			CancellationToken.None
		);

		const lines = result.content[0].value.split('\n');
		const fileLines = lines.filter(l => l.startsWith('- '));
		expect(fileLines.length).toBeLessThanOrEqual(10);
	});

	test('handles invalid pattern', async () => {
		const result = await tool.invoke(
			{ query: '[invalid' },
			CancellationToken.None
		);

		expect(result.content[0].value).toContain('Invalid');
	});

	test('handles no results', async () => {
		const result = await tool.invoke(
			{ query: '**/*.nonexistent' },
			CancellationToken.None
		);

		expect(result.content[0].value).toContain('No files found');
	});
});
```

## Best Practices

### 1. Use Specific Patterns

```typescript
// ❌ Too broad - slow and many results
{
	"query": "**/*"
}

// ✅ Specific - fast and relevant
{
	"query": "src/components/**/*.tsx"
}
```

### 2. Leverage Extension Alternatives

```typescript
// ❌ Less efficient
{
	"query": "**/file.{js,jsx,ts,tsx}"
}

// ✅ More efficient if you know the type
{
	"query": "**/file.tsx"
}
```

### 3. Use Appropriate maxResults

```typescript
// ❌ Requesting too many
{
	"query": "**/*.ts",
	"maxResults": 10000
}

// ✅ Reasonable limit
{
	"query": "**/*.ts",
	"maxResults": 50
}
```

### 4. Narrow Search Scope

```typescript
// ❌ Workspace-wide search
{
	"query": "**/*.config.js"
}

// ✅ Scoped to specific directory
{
	"query": "src/**/*.config.js"
}
```

### 5. Combine with Other Tools

```typescript
// Pattern: Find then Read
// Step 1: Find config files
const files = await findFiles({
	query: "**/tsconfig.json"
});

// Step 2: Read specific file
const content = await readFile({
	filePath: files[0]
});
```

### 6. Handle Empty Results Gracefully

```typescript
async function findOrFallback(pattern: string): Promise<string[]> {
	const result = await findFiles({ query: pattern });

	if (result.content[0].value.includes('No files found')) {
		// Try alternative pattern
		return findFiles({ query: alternativePattern });
	}

	return parseFileList(result);
}
```

### 7. Cache When Appropriate

```typescript
// For repeated searches, use caching
const cache = new Map<string, URI[]>();

async function cachedFindFiles(query: string): Promise<URI[]> {
	if (cache.has(query)) {
		return cache.get(query)!;
	}

	const result = await findFiles({ query });
	cache.set(query, result);

	return result;
}
```

## Limitations

### 1. No Content Search

**Limitation:** Only matches file paths, not file content.

**Example:**
```typescript
// ✗ Cannot find files containing "TODO"
// ✓ Can find files named "TODO.txt"
{
	"query": "**/TODO*"
}
```

**Workaround:** Use `copilot_findTextInFiles` for content search

### 2. Glob Pattern Limitations

**Limitation:** Some advanced glob features may not be supported.

**Examples:**
```typescript
// Not all shells' extended glob supported
"**/{!(test|spec)}/**/*.ts"  // May not work

// Negation often requires separate query
"!**/node_modules/**"  // Handled via excludes instead
```

### 3. Performance on Large Workspaces

**Limitation:** Broad patterns can be slow in large workspaces.

**Impact:**
- 100K+ files: Several seconds for `**/*`
- Network/remote filesystems: Additional latency

**Mitigation:**
```typescript
// Use specific patterns
"src/**/*.ts"  // Instead of "**/*.ts"

// Limit results
{
	"query": "**/*.ts",
	"maxResults": 50
}
```

### 4. Excluded Files Not Searchable

**Limitation:** Files matching exclude patterns are not found.

**Example:**
```json
// settings.json
{
	"search.exclude": {
		"**/dist/**": true
	}
}

// This will not find files in dist/
{
	"query": "dist/**/*.js"
}
```

**Workaround:** Adjust exclude settings temporarily or use file system directly

### 5. Symbolic Link Handling

**Limitation:** Behavior with symbolic links depends on VS Code settings.

**Settings:**
```json
{
	"search.followSymlinks": true  // Follow symlinks (default)
}
```

**Issues:**
- Circular symlinks can cause infinite loops
- Performance impact with many symlinks

### 6. Case Sensitivity

**Limitation:** Case sensitivity depends on file system.

**Behavior:**
- Windows: Case-insensitive
- macOS: Case-insensitive (usually)
- Linux: Case-sensitive

**Example:**
```typescript
// On Windows: matches File.ts, file.ts, FILE.ts
// On Linux: matches only file.ts
{
	"query": "**/file.ts"
}
```

### 7. No Sorting or Ranking

**Limitation:** Results are not ranked by relevance.

**Behavior:**
- Order depends on file system traversal
- No scoring or ranking applied

**Workaround:** Implement custom sorting post-search

### 8. Pattern Complexity Limits

**Limitation:** Very complex patterns may fail or timeout.

**Example:**
```typescript
// May be too complex
{
	"query": "**/{a,b,c}/**/[0-9]*{x,y,z}.{js,ts,jsx,tsx}"
}
```

**Best Practice:** Keep patterns simple and specific

### 9. No Regex Support

**Limitation:** Only glob patterns, not regular expressions.

**Not Supported:**
```typescript
// Regex syntax doesn't work
{
	"query": "**/(test|spec)\\.ts$"  // ✗ Won't work
}

// Use glob instead
{
	"query": "**/{test,spec}.ts"  // ✓ Works
}
```

### 10. Memory with Large Result Sets

**Limitation:** Large result sets consume memory.

**Impact:**
- 1K files: ~100KB
- 10K files: ~1MB
- 100K files: ~10MB+

**Mitigation:**
```typescript
// Use maxResults to limit memory
{
	"query": "**/*",
	"maxResults": 100
}
```

---

**Related Documentation:**
- [copilot_findTextInFiles](./05-copilot-findTextInFiles.md)
- [copilot_readFile](./06-copilot-readFile.md)
- [copilot_listDirectory](./07-copilot-listDirectory.md)
- [copilot_searchCodebase](./01-copilot-searchCodebase.md)

**Last Updated:** 2025-01-22
