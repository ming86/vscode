# copilot_findTextInFiles Tool Documentation

## Overview

**Tool Name:** `copilot_findTextInFiles`

**Purpose:** Performs fast text or regex searches across workspace files, returning matching content with context. Essential for finding specific code patterns, string literals, function calls, or any text-based content across the codebase. Supports both plain text and regular expression searches.

**Implementation Location:** `src/extension/tools/codeSearch/findTextInFilesTool.ts`

**Tool Class:** `FindTextInFilesTool`

## Tool Declaration

```json
{
	"type": "function",
	"function": {
		"name": "copilot_findTextInFiles",
		"description": "Do a fast text search in the workspace. Use this tool when you want to search with an exact string or regex. If you are not sure what words will appear in the workspace, prefer using regex patterns with alternation (|) or character classes to search for multiple potential words at once instead of making separate searches. For example, use 'function|method|procedure' to look for all of those words at once. Use includePattern to search within files matching a specific pattern, or in a specific file, using a relative path. Use 'includeIgnoredFiles' to include files normally ignored by .gitignore, other ignore files, and `files.exclude` and `search.exclude` settings. Warning: using this may cause the search to be slower, only set it when you want to search in ignored folders like node_modules or build outputs. Use this tool when you want to see an overview of a particular file, instead of using read_file many times to look for code within a file.",
		"parameters": {
			"type": "object",
			"properties": {
				"query": {
					"type": "string",
					"description": "The pattern to search for in files in the workspace. Use regex with alternation (e.g., 'word1|word2|word3') or character classes to find multiple potential words in a single search. Be sure to set the isRegexp property properly to declare whether it's a regex or plain text pattern. Is case-insensitive."
				},
				"isRegexp": {
					"type": "boolean",
					"description": "Whether the pattern is a regex."
				},
				"includePattern": {
					"type": "string",
					"description": "Search files matching this glob pattern. Will be applied to the relative path of files within the workspace. To search recursively inside a folder, use a proper glob pattern like \"src/folder/**\". Do not use | in includePattern."
				},
				"includeIgnoredFiles": {
					"type": "boolean",
					"description": "Whether to include files that would normally be ignored according to .gitignore, other ignore files and `files.exclude` and `search.exclude` settings. Warning: using this may cause the search to be slower. Only set it when you want to search in ignored folders like node_modules or build outputs."
				},
				"maxResults": {
					"type": "number",
					"description": "The maximum number of results to return. Do not use this unless necessary, it can slow things down. By default, only some matches are returned. If you use this and don't see what you're looking for, you can try again with a more specific query or a larger maxResults."
				}
			},
			"required": [
				"query",
				"isRegexp"
			]
		}
	}
}
```

## Input Parameters

### TypeScript Interface

```typescript
interface FindTextInFilesInput {
	query: string;                    // Required: Search pattern (text or regex)
	isRegexp: boolean;                // Required: Whether query is regex
	includePattern?: string;          // Optional: Glob pattern for files to search
	includeIgnoredFiles?: boolean;    // Optional: Include .gitignore'd files
	maxResults?: number;              // Optional: Maximum results to return
}
```

### Parameter Details

| Parameter | Type | Required | Default | Description |
|-----------|------|----------|---------|-------------|
| `query` | string | Yes | - | Search pattern (text or regex). Case-insensitive. |
| `isRegexp` | boolean | Yes | - | Whether `query` is a regular expression |
| `includePattern` | string | No | `**/*` | Glob pattern to filter files (e.g., `src/**/*.ts`) |
| `includeIgnoredFiles` | boolean | No | `false` | Include files normally ignored (node_modules, .gitignore, etc.) |
| `maxResults` | number | No | 20-50 | Maximum number of matches to return |

### Parameter Examples

```typescript
// Plain text search
{
	"query": "handleRequest",
	"isRegexp": false
}

// Regex search with alternation
{
	"query": "function|method|procedure",
	"isRegexp": true
}

// Search in specific files
{
	"query": "TODO",
	"isRegexp": false,
	"includePattern": "src/**/*.ts"
}

// Search including ignored files
{
	"query": "webpack",
	"isRegexp": false,
	"includeIgnoredFiles": true
}

// Complex regex with limits
{
	"query": "export (class|interface|type)\\s+\\w+",
	"isRegexp": true,
	"includePattern": "**/*.ts",
	"maxResults": 100
}
```

### Regex Pattern Examples

```typescript
// Match function definitions
{
	"query": "function\\s+\\w+\\s*\\(",
	"isRegexp": true
}

// Match imports
{
	"query": "import\\s+.*\\s+from\\s+['\"]",
	"isRegexp": true
}

// Match multiple keywords
{
	"query": "TODO|FIXME|HACK|XXX",
	"isRegexp": true
}

// Match API endpoints
{
	"query": "/(api|v[0-9]+)/[a-z]+",
	"isRegexp": true
}

// Match hex colors
{
	"query": "#[0-9a-fA-F]{6}",
	"isRegexp": true
}
```

## Architecture

### Flow Diagram

```
┌─────────────────────────────────────────────────────────────┐
│                  copilot_findTextInFiles                    │
│                  (FindTextInFilesTool)                      │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
                    ┌─────────────────┐
                    │ Parse Input     │
                    │ - Validate query│
                    │ - Compile regex │
                    └─────────────────┘
                              │
                              ▼
                ┌─────────────────────────────┐
                │ Prepare Search Options      │
                │ - File patterns             │
                │ - Exclude patterns          │
                │ - Search flags              │
                └─────────────────────────────┘
                              │
                              ▼
                ┌─────────────────────────────┐
                │ Execute Text Search         │
                │ - ITextSearchService        │
                │ - Match against pattern     │
                │ - Apply filters             │
                └─────────────────────────────┘
                              │
                              ▼
                ┌─────────────────────────────┐
                │ Collect Matches             │
                │ - Extract match context     │
                │ - Get surrounding lines     │
                │ - Store match metadata      │
                └─────────────────────────────┘
                              │
                              ▼
                ┌─────────────────────────────┐
                │ Group & Organize Results    │
                │ - Group by file             │
                │ - Sort by location          │
                │ - Apply maxResults limit    │
                └─────────────────────────────┘
                              │
                              ▼
                ┌─────────────────────────────┐
                │ Format Output               │
                │ - Create markdown           │
                │ - Add context snippets      │
                │ - Include line numbers      │
                └─────────────────────────────┘
                              │
                              ▼
                    ┌─────────────────┐
                    │ Return Results  │
                    │ (Markdown)      │
                    └─────────────────┘
```

### Component Interaction

```
┌──────────────────┐
│  Language Model  │
│   (Claude/GPT)   │
└────────┬─────────┘
         │ Request: query, isRegexp, options
         ▼
┌──────────────────────────────────┐
│    FindTextInFilesTool           │
│  - validateQuery()               │
│  - executeSearch()               │
│  - extractContext()              │
│  - formatResults()               │
└────────┬─────────────────────────┘
         │
         ├────────────────────┐
         │                    │
         ▼                    ▼
┌──────────────────┐  ┌──────────────────┐
│ ITextSearchSvc   │  │  IFileService    │
│ - textSearch()   │  │  - readFile()    │
│ - fileSearch()   │  │  - stat()        │
└──────────────────┘  └──────────────────┘
         │
         ▼
┌──────────────────┐
│   Regex Engine   │
│   - match()      │
│   - exec()       │
└──────────────────┘
```

## Internal Implementation

### Core Implementation

```typescript
export class FindTextInFilesTool extends LanguageModelTool<FindTextInFilesInput> {
	static readonly ID = 'copilot_findTextInFiles';

	constructor(
		@ITextSearchService private readonly textSearchService: ITextSearchService,
		@IFileService private readonly fileService: IFileService,
		@IWorkspaceContextService private readonly workspaceService: IWorkspaceContextService,
		@IConfigurationService private readonly configService: IConfigurationService
	) {
		super();
	}

	async invoke(
		input: FindTextInFilesInput,
		token: CancellationToken
	): Promise<LanguageModelToolResult> {
		const { query, isRegexp, includePattern, includeIgnoredFiles, maxResults } = input;

		// Step 1: Validate query
		if (!query || query.trim().length === 0) {
			return {
				content: [{
					kind: 'text',
					value: 'Search query cannot be empty'
				}]
			};
		}

		// Step 2: Prepare search options
		const searchOptions = this.prepareSearchOptions(
			query,
			isRegexp,
			includePattern,
			includeIgnoredFiles,
			maxResults
		);

		// Step 3: Execute search
		const matches = await this.executeSearch(searchOptions, token);

		// Step 4: Format results
		if (matches.length === 0) {
			return {
				content: [{
					kind: 'text',
					value: `No matches found for: "${query}"`
				}]
			};
		}

		const markdown = await this.formatResults(query, matches, maxResults);

		return {
			content: [{
				kind: 'text',
				value: markdown
			}]
		};
	}

	private prepareSearchOptions(
		query: string,
		isRegexp: boolean,
		includePattern: string | undefined,
		includeIgnoredFiles: boolean | undefined,
		maxResults: number | undefined
	): ITextSearchOptions {
		const defaultMaxResults = 20;

		// Build search pattern
		const pattern: IPatternInfo = isRegexp
			? {
				pattern: query,
				isRegExp: true,
				isCaseSensitive: false,
				isWordMatch: false
			}
			: {
				pattern: query,
				isRegExp: false,
				isCaseSensitive: false
			};

		// Get exclude patterns
		const excludePattern = includeIgnoredFiles
			? undefined
			: this.getExcludePatterns();

		return {
			pattern,
			includePattern: includePattern || '**/*',
			excludePattern,
			maxResults: maxResults ?? defaultMaxResults,
			previewOptions: {
				matchLines: 1,
				charsPerLine: 1000
			},
			disregardIgnoreFiles: includeIgnoredFiles ?? false,
			disregardExcludeSettings: includeIgnoredFiles ?? false
		};
	}

	private async executeSearch(
		options: ITextSearchOptions,
		token: CancellationToken
	): Promise<TextSearchMatch[]> {
		const workspaceFolders = this.workspaceService.getWorkspace().folders;
		const allMatches: TextSearchMatch[] = [];

		for (const folder of workspaceFolders) {
			if (token.isCancellationRequested) {
				break;
			}

			await this.textSearchService.textSearch(
				options,
				folder.uri,
				match => {
					if (match && this.isTextSearchMatch(match)) {
						allMatches.push(match);
					}

					// Stop if we've hit the limit
					return allMatches.length < (options.maxResults ?? Infinity);
				},
				token
			);

			if (allMatches.length >= (options.maxResults ?? Infinity)) {
				break;
			}
		}

		return allMatches;
	}

	private isTextSearchMatch(
		match: IFileMatch | ITextSearchMatch
	): match is ITextSearchMatch {
		return 'preview' in match && 'ranges' in match;
	}

	private async formatResults(
		query: string,
		matches: TextSearchMatch[],
		maxResults: number | undefined
	): Promise<string> {
		const lines: string[] = [
			`# Matches for \`${query}\``,
			''
		];

		// Group by file
		const byFile = this.groupByFile(matches);

		// Add summary
		const fileCount = byFile.size;
		const matchCount = matches.length;
		const truncated = maxResults && matchCount >= maxResults;

		if (truncated) {
			lines.push(`Found ${matchCount}+ matches in ${fileCount}+ files (showing first ${matchCount}):`);
		} else {
			lines.push(`Found ${matchCount} match(es) in ${fileCount} file(s):`);
		}
		lines.push('');

		// Format each file's matches
		for (const [filePath, fileMatches] of byFile) {
			lines.push(`## ${filePath}`);
			lines.push('');
			lines.push(`${fileMatches.length} match(es)`);
			lines.push('');

			for (let i = 0; i < fileMatches.length; i++) {
				const match = fileMatches[i];
				lines.push(`### Match ${i + 1} (Line ${match.lineNumber})`);
				lines.push('');
				lines.push('```');
				lines.push(this.formatMatchContext(match));
				lines.push('```');
				lines.push('');
			}
		}

		if (truncated) {
			lines.push('*Use maxResults parameter or more specific query to see more matches*');
		}

		return lines.join('\n');
	}

	private formatMatchContext(match: TextSearchMatch): string {
		const lines: string[] = [];
		const preview = match.preview;

		// Calculate line numbers for context
		const matchLine = match.lineNumber;
		const startLine = Math.max(1, matchLine - 2);

		// Get context lines (if available)
		const contextBefore = preview.before || '';
		const contextAfter = preview.after || '';
		const matchText = preview.text;

		// Format with line numbers
		if (contextBefore) {
			const beforeLines = contextBefore.split('\n');
			beforeLines.forEach((line, idx) => {
				const lineNum = startLine + idx;
				lines.push(`  ${lineNum}: ${line}`);
			});
		}

		// Highlight the matching line
		lines.push(`> ${matchLine}: ${matchText}`);

		if (contextAfter) {
			const afterLines = contextAfter.split('\n');
			afterLines.forEach((line, idx) => {
				const lineNum = matchLine + idx + 1;
				lines.push(`  ${lineNum}: ${line}`);
			});
		}

		return lines.join('\n');
	}

	private groupByFile(
		matches: TextSearchMatch[]
	): Map<string, TextSearchMatch[]> {
		const map = new Map<string, TextSearchMatch[]>();

		for (const match of matches) {
			const path = this.getRelativePath(match.uri);
			if (!map.has(path)) {
				map.set(path, []);
			}
			map.get(path)!.push(match);
		}

		return map;
	}

	private getRelativePath(uri: URI): string {
		const workspaceFolder = this.workspaceService.getWorkspaceFolder(uri);
		if (workspaceFolder) {
			const relativePath = uri.fsPath.substring(
				workspaceFolder.uri.fsPath.length + 1
			);
			return relativePath;
		}
		return uri.fsPath;
	}

	private getExcludePatterns(): string[] {
		const filesExclude = this.configService.getValue<Record<string, boolean>>('files.exclude') ?? {};
		const searchExclude = this.configService.getValue<Record<string, boolean>>('search.exclude') ?? {};

		const patterns: string[] = [];

		for (const [pattern, enabled] of Object.entries({...filesExclude, ...searchExclude})) {
			if (enabled) {
				patterns.push(pattern);
			}
		}

		return patterns;
	}
}

interface TextSearchMatch {
	uri: URI;
	lineNumber: number;
	preview: {
		text: string;
		before?: string;
		after?: string;
		matches: Range[];
	};
	ranges: Range[];
}
```

### Advanced Regex Compilation

```typescript
class RegexSearchOptimizer {
	/**
	 * Optimize regex pattern for better performance
	 */
	static optimize(pattern: string): string {
		// Remove unnecessary capturing groups
		pattern = pattern.replace(/\((?!\?)/g, '(?:');

		// Optimize alternation order (most common first)
		// This is a heuristic - in practice, profile and adjust
		pattern = this.optimizeAlternation(pattern);

		return pattern;
	}

	private static optimizeAlternation(pattern: string): string {
		// Extract alternation patterns
		const alternationRegex = /\(([^)]+)\)/g;

		return pattern.replace(alternationRegex, (match, group) => {
			if (!group.includes('|')) {
				return match;
			}

			// Sort alternatives by length (longer first)
			const alternatives = group.split('|');
			alternatives.sort((a, b) => b.length - a.length);

			return `(${alternatives.join('|')})`;
		});
	}

	/**
	 * Validate regex pattern
	 */
	static validate(pattern: string): ValidationResult {
		try {
			new RegExp(pattern);
			return { valid: true };
		} catch (error) {
			return {
				valid: false,
				error: error.message
			};
		}
	}
}
```

### Context Extraction

```typescript
class ContextExtractor {
	private readonly CONTEXT_LINES = 2;
	private readonly MAX_LINE_LENGTH = 1000;

	async extractContext(
		uri: URI,
		lineNumber: number,
		matchRange: Range
	): Promise<MatchContext> {
		const document = await this.fileService.readFile(uri);
		const lines = document.value.toString().split('\n');

		const startLine = Math.max(0, lineNumber - this.CONTEXT_LINES - 1);
		const endLine = Math.min(lines.length - 1, lineNumber + this.CONTEXT_LINES - 1);

		const before: string[] = [];
		const after: string[] = [];

		// Context before match
		for (let i = startLine; i < lineNumber - 1; i++) {
			before.push(this.truncateLine(lines[i]));
		}

		// Context after match
		for (let i = lineNumber; i <= endLine; i++) {
			after.push(this.truncateLine(lines[i]));
		}

		const matchLine = lines[lineNumber - 1];

		return {
			before: before.join('\n'),
			matchLine: this.truncateLine(matchLine),
			after: after.join('\n'),
			lineNumber
		};
	}

	private truncateLine(line: string): string {
		if (line.length <= this.MAX_LINE_LENGTH) {
			return line;
		}

		return line.substring(0, this.MAX_LINE_LENGTH) + '... [truncated]';
	}
}

interface MatchContext {
	before: string;
	matchLine: string;
	after: string;
	lineNumber: number;
}
```

## Dependencies

### Service Dependencies

```typescript
// Core VS Code services required
@ITextSearchService          // Text search across workspace
@IFileService                // File reading operations
@IWorkspaceContextService    // Workspace information
@IConfigurationService       // Access to search settings
```

### Text Search Service

```typescript
interface ITextSearchService {
	/**
	 * Search for text in files
	 */
	textSearch(
		query: ITextQuery,
		folder: URI,
		onProgress: (result: IFileMatch) => boolean,
		token: CancellationToken
	): Promise<ISearchComplete>;
}

interface ITextQuery {
	pattern: IPatternInfo;
	includePattern?: string;
	excludePattern?: string | string[];
	maxResults?: number;
	previewOptions?: ITextSearchPreviewOptions;
	disregardIgnoreFiles?: boolean;
	disregardExcludeSettings?: boolean;
}

interface IPatternInfo {
	pattern: string;
	isRegExp?: boolean;
	isCaseSensitive?: boolean;
	isWordMatch?: boolean;
	isMultiline?: boolean;
}
```

### Configuration Keys

```typescript
// Relevant settings
const searchSettings = {
	exclude: 'search.exclude',
	useIgnoreFiles: 'search.useIgnoreFiles',
	useGlobalIgnoreFiles: 'search.useGlobalIgnoreFiles',
	followSymlinks: 'search.followSymlinks',
	smartCase: 'search.smartCase',
	maxResults: 'search.maxResults'
};

// Example configuration
{
	"search.exclude": {
		"**/node_modules": true,
		"**/dist": true,
		"**/*.min.js": true
	},
	"search.useIgnoreFiles": true,
	"search.useGlobalIgnoreFiles": true,
	"search.followSymlinks": true,
	"search.smartCase": false,
	"search.maxResults": 20000
}
```

## Performance Characteristics

### Time Complexity

| Operation | Complexity | Notes |
|-----------|------------|-------|
| Regex compilation | O(p) | p = pattern length |
| Text search per file | O(n × m) | n = file size, m = pattern complexity |
| Total search | O(N × n × m) | N = number of files |
| Context extraction | O(c) | c = context lines |

### Performance Metrics

```typescript
const benchmarks = {
	plainText: {
		workspace: '10K files',
		pattern: 'function',
		time: '500ms-1s',
		matchesFound: 1000
	},
	simpleRegex: {
		workspace: '10K files',
		pattern: 'function\\s+\\w+',
		time: '1-2s',
		matchesFound: 500
	},
	complexRegex: {
		workspace: '10K files',
		pattern: '(function|method|procedure)\\s+\\w+\\s*\\([^)]*\\)',
		time: '2-5s',
		matchesFound: 750
	},
	withIgnoredFiles: {
		workspace: '50K files (with node_modules)',
		pattern: 'webpack',
		time: '5-10s',
		matchesFound: 2000
	}
};
```

### Optimization Strategies

**1. Use Plain Text When Possible**

```typescript
// ❌ Slower - unnecessary regex
{
	"query": "handleRequest",
	"isRegexp": true
}

// ✅ Faster - plain text
{
	"query": "handleRequest",
	"isRegexp": false
}
```

**2. Scope with includePattern**

```typescript
// ❌ Slow - searches all files
{
	"query": "TODO",
	"isRegexp": false
}

// ✅ Fast - searches only TypeScript files
{
	"query": "TODO",
	"isRegexp": false,
	"includePattern": "**/*.ts"
}
```

**3. Use Specific Patterns**

```typescript
// ❌ Very broad - many false positives
{
	"query": ".*",
	"isRegexp": true
}

// ✅ Specific - relevant matches
{
	"query": "export\\s+(class|interface|type)\\s+\\w+",
	"isRegexp": true
}
```

**4. Combine Alternatives in Single Regex**

```typescript
// ❌ Multiple separate searches
search("TODO");
search("FIXME");
search("HACK");

// ✅ Single search with alternation
{
	"query": "TODO|FIXME|HACK",
	"isRegexp": true
}
```

**5. Avoid includeIgnoredFiles Unless Necessary**

```typescript
// ❌ Slow - searches node_modules, dist, etc.
{
	"query": "webpack",
	"isRegexp": false,
	"includeIgnoredFiles": true
}

// ✅ Fast - respects ignore files
{
	"query": "webpack",
	"isRegexp": false,
	"includeIgnoredFiles": false
}
```

## Use Cases

### 1. Find Function Calls

```typescript
// Query
{
	"query": "calculateTotal\\s*\\(",
	"isRegexp": true,
	"includePattern": "src/**/*.ts"
}

// Finds all calls to calculateTotal function
```

### 2. Find TODO Comments

```typescript
// Query
{
	"query": "TODO|FIXME|XXX|HACK",
	"isRegexp": true
}

// Output
# Matches for `TODO|FIXME|XXX|HACK`

Found 15 match(es) in 8 file(s):

## src/utils/helper.ts

### Match 1 (Line 45)
```
  43:   }
  44:
> 45:   // TODO: Refactor this function
  46:   function processData(data) {
  47:     return data;
```
```

### 3. Find Import Statements

```typescript
// Query
{
	"query": "import\\s+.*\\s+from\\s+['\"]react['\"]",
	"isRegexp": true,
	"includePattern": "**/*.{ts,tsx}"
}

// Finds all React imports
```

### 4. Find API Endpoints

```typescript
// Query
{
	"query": "/(api|v[0-9]+)/[a-z-]+",
	"isRegexp": true,
	"includePattern": "src/**/*.ts"
}

// Finds endpoint patterns like /api/users, /v1/products
```

### 5. Find Configuration Values

```typescript
// Query
{
	"query": "apiKey|secretKey|password",
	"isRegexp": true,
	"includePattern": "**/*.{json,yaml,env}"
}

// Security audit for sensitive keys
```

### 6. Find Deprecated Code

```typescript
// Query
{
	"query": "@deprecated",
	"isRegexp": false,
	"includePattern": "src/**/*.ts"
}

// Finds deprecated methods/classes
```

### 7. Find Error Handling

```typescript
// Query
{
	"query": "catch\\s*\\([^)]*\\)\\s*\\{",
	"isRegexp": true
}

// Finds all catch blocks
```

### 8. Search in Dependencies

```typescript
// Query
{
	"query": "webpack",
	"isRegexp": false,
	"includeIgnoredFiles": true,
	"includePattern": "node_modules/**"
}

// Searches in normally ignored node_modules
```

## Comparison with Other Tools

### vs copilot_searchCodebase

| Aspect | findTextInFiles | searchCodebase |
|--------|-----------------|----------------|
| **Search Method** | Text/regex exact match | Semantic/fuzzy search |
| **Pattern Type** | String or regex | Natural language query |
| **Accuracy** | Exact matches only | Ranked by relevance |
| **Speed** | Very fast | Moderate |
| **Use Case** | Known patterns | Exploratory search |
| **Context** | Line-based | Chunk-based |

**When to use findTextInFiles:**
- Exact string/pattern matching
- Regex-based searches
- Finding specific syntax
- Performance-critical searches

**When to use searchCodebase:**
- Semantic code understanding
- Fuzzy concept searches
- Finding similar implementations
- Exploratory analysis

### vs copilot_listCodeUsages

| Aspect | findTextInFiles | listCodeUsages |
|--------|-----------------|----------------|
| **Target** | Any text | Specific symbols |
| **Method** | Text search | Language service |
| **Accuracy** | String matching | Semantic (symbol-aware) |
| **Noise** | Can include comments, strings | Only code symbols |

**Example:**

```typescript
// findTextInFiles for "count"
// Returns: variables, comments, strings, methods, ALL occurrences

// listCodeUsages for "count"
// Returns: Only the symbol "count" references (semantic)
```

### vs copilot_findFiles

| Aspect | findTextInFiles | findFiles |
|--------|-----------------|-----------|
| **Search Target** | File content | File paths |
| **Performance** | Slower | Faster |
| **Use Case** | Content search | File discovery |

**Workflow combination:**

```typescript
// Step 1: Find files
findFiles({ query: "**/*.config.ts" })

// Step 2: Search within those files
findTextInFiles({
	query: "webpack",
	isRegexp: false,
	includePattern: "**/*.config.ts"
})
```

## Error Handling

### Common Error Scenarios

**1. Invalid Regex Pattern**

```typescript
// Input
{
	"query": "[invalid(",
	"isRegexp": true
}

// Output
Invalid regular expression pattern: Unterminated character class

// Validation
private validateRegex(pattern: string): void {
	try {
		new RegExp(pattern);
	} catch (error) {
		throw new Error(`Invalid regex: ${error.message}`);
	}
}
```

**2. Empty Query**

```typescript
// Input
{
	"query": "",
	"isRegexp": false
}

// Output
Search query cannot be empty
```

**3. No Matches Found**

```typescript
// Input
{
	"query": "nonExistentPattern123",
	"isRegexp": false
}

// Output
No matches found for: "nonExistentPattern123"
```

**4. Search Timeout**

```typescript
// Very complex regex or large workspace
{
	"query": "(a+)+b",  // Catastrophic backtracking
	"isRegexp": true
}

// Protection
const TIMEOUT_MS = 30000;
const timeoutPromise = new Promise((_, reject) =>
	setTimeout(() => reject(new Error('Search timeout')), TIMEOUT_MS)
);

await Promise.race([searchPromise, timeoutPromise]);
```

**5. Pattern Too Broad**

```typescript
// Input
{
	"query": ".",  // Matches everything
	"isRegexp": true
}

// Mitigation: Limit results
Found 1000+ matches (showing first 1000)
*Pattern is very broad - consider using a more specific query*
```

### Error Recovery

```typescript
class RobustFindTextInFilesTool extends FindTextInFilesTool {
	async invoke(
		input: FindTextInFilesInput,
		token: CancellationToken
	): Promise<LanguageModelToolResult> {
		try {
			// Validate regex
			if (input.isRegexp) {
				this.validateRegex(input.query);
			}

			return await super.invoke(input, token);

		} catch (error) {
			return this.handleError(error, input);
		}
	}

	private handleError(
		error: Error,
		input: FindTextInFilesInput
	): LanguageModelToolResult {
		if (error.message.includes('regex') || error.message.includes('pattern')) {
			return {
				content: [{
					kind: 'text',
					value: `Invalid regular expression: ${error.message}\n\n` +
					       `Tip: Try using plain text search (isRegexp: false) or ` +
					       `simplify your regex pattern.`
				}]
			};
		}

		if (error.message.includes('timeout')) {
			return {
				content: [{
					kind: 'text',
					value: `Search timeout. Try:\n` +
					       `- Using a more specific includePattern\n` +
					       `- Simplifying the regex pattern\n` +
					       `- Reducing maxResults`
				}]
			};
		}

		return {
			content: [{
				kind: 'text',
				value: `Search error: ${error.message}`
			}]
		};
	}

	private validateRegex(pattern: string): void {
		try {
			new RegExp(pattern);
		} catch (error) {
			throw new Error(`Invalid regex pattern: ${error.message}`);
		}
	}
}
```

## Configuration

### Tool Settings

```json
{
	// Default maximum results
	"copilot.tools.findTextInFiles.defaultMaxResults": 20,

	// Search timeout (ms)
	"copilot.tools.findTextInFiles.timeout": 30000,

	// Context lines before/after match
	"copilot.tools.findTextInFiles.contextLines": 2,

	// Maximum line length in results
	"copilot.tools.findTextInFiles.maxLineLength": 1000,

	// Respect .gitignore by default
	"copilot.tools.findTextInFiles.respectIgnoreFiles": true,

	// Regex optimization
	"copilot.tools.findTextInFiles.optimizeRegex": true,

	// Enable/disable regex validation
	"copilot.tools.findTextInFiles.validateRegex": true
}
```

### VS Code Search Settings

```json
{
	// Search exclude patterns
	"search.exclude": {
		"**/node_modules": true,
		"**/bower_components": true,
		"**/*.code-search": true,
		"**/dist": true,
		"**/build": true,
		"**/*.min.js": true,
		"**/*.map": true
	},

	// Use .gitignore files
	"search.useIgnoreFiles": true,

	// Use global .gitignore
	"search.useGlobalIgnoreFiles": true,

	// Follow symbolic links
	"search.followSymlinks": true,

	// Smart case sensitivity
	"search.smartCase": false,

	// Maximum file size to search (KB)
	"search.maxFileSize": 1024,

	// Use ripgrep for search
	"search.useRipgrep": true
}
```

## Implementation Blueprint

### Step-by-Step Implementation

**Phase 1: Basic Structure**

```typescript
export class FindTextInFilesTool extends LanguageModelTool<FindTextInFilesInput> {
	static readonly ID = 'copilot_findTextInFiles';

	async invoke(
		input: FindTextInFilesInput,
		token: CancellationToken
	): Promise<LanguageModelToolResult> {
		// Validate input
		this.validateInput(input);

		// Prepare search options
		const options = this.prepareSearchOptions(input);

		// Execute search
		const matches = await this.executeSearch(options, token);

		// Format results
		return this.formatResults(input.query, matches);
	}
}
```

**Phase 2: Input Validation**

```typescript
private validateInput(input: FindTextInFilesInput): void {
	if (!input.query || input.query.trim().length === 0) {
		throw new Error('Search query cannot be empty');
	}

	if (input.isRegexp) {
		try {
			new RegExp(input.query);
		} catch (error) {
			throw new Error(`Invalid regex: ${error.message}`);
		}
	}

	if (input.includePattern) {
		// Validate glob pattern
		this.validateGlobPattern(input.includePattern);
	}
}
```

**Phase 3: Search Execution**

```typescript
private async executeSearch(
	options: ITextSearchOptions,
	token: CancellationToken
): Promise<TextSearchMatch[]> {
	const matches: TextSearchMatch[] = [];
	const folders = this.workspaceService.getWorkspace().folders;

	for (const folder of folders) {
		if (token.isCancellationRequested) break;

		await this.textSearchService.textSearch(
			options,
			folder.uri,
			match => {
				if (this.isTextSearchMatch(match)) {
					matches.push(match);
				}
				return matches.length < (options.maxResults ?? Infinity);
			},
			token
		);
	}

	return matches;
}
```

**Phase 4: Result Formatting**

```typescript
private async formatResults(
	query: string,
	matches: TextSearchMatch[]
): Promise<string> {
	const markdown: string[] = [
		`# Matches for \`${query}\``,
		'',
		`Found ${matches.length} match(es) in ${this.countFiles(matches)} file(s):`,
		''
	];

	const byFile = this.groupByFile(matches);

	for (const [filePath, fileMatches] of byFile) {
		markdown.push(`## ${filePath}`);
		markdown.push('');

		for (const match of fileMatches) {
			markdown.push(`### Line ${match.lineNumber}`);
			markdown.push('```');
			markdown.push(this.formatMatch(match));
			markdown.push('```');
			markdown.push('');
		}
	}

	return markdown.join('\n');
}
```

**Phase 5: Testing**

```typescript
describe('FindTextInFilesTool', () => {
	test('finds plain text', async () => {
		const result = await tool.invoke({
			query: 'function',
			isRegexp: false
		}, token);

		expect(result.content[0].value).toContain('function');
	});

	test('finds regex pattern', async () => {
		const result = await tool.invoke({
			query: 'function\\s+\\w+',
			isRegexp: true
		}, token);

		expect(result.content[0].value).toContain('Matches for');
	});

	test('respects includePattern', async () => {
		const result = await tool.invoke({
			query: 'TODO',
			isRegexp: false,
			includePattern: '**/*.ts'
		}, token);

		// Verify only .ts files in results
	});

	test('handles invalid regex', async () => {
		await expect(
			tool.invoke({
				query: '[invalid',
				isRegexp: true
			}, token)
		).rejects.toThrow('Invalid regex');
	});
});
```

## Best Practices

### 1. Use Regex Alternation for Multiple Patterns

```typescript
// ❌ Multiple searches
search("TODO");
search("FIXME");
search("HACK");

// ✅ Single search
{
	"query": "TODO|FIXME|HACK",
	"isRegexp": true
}
```

### 2. Scope Searches with includePattern

```typescript
// ❌ Search all files
{
	"query": "interface\\s+I\\w+",
	"isRegexp": true
}

// ✅ Search only TypeScript
{
	"query": "interface\\s+I\\w+",
	"isRegexp": true,
	"includePattern": "**/*.ts"
}
```

### 3. Optimize Regex Patterns

```typescript
// ❌ Inefficient - unnecessary capturing
{
	"query": "(function)\\s+(\\w+)",
	"isRegexp": true
}

// ✅ Efficient - non-capturing groups
{
	"query": "(?:function)\\s+(\\w+)",
	"isRegexp": true
}
```

### 4. Use Plain Text When Possible

```typescript
// ❌ Unnecessary regex
{
	"query": "handleRequest",
	"isRegexp": true
}

// ✅ Plain text (faster)
{
	"query": "handleRequest",
	"isRegexp": false
}
```

### 5. Avoid Catastrophic Backtracking

```typescript
// ❌ Dangerous regex
{
	"query": "(a+)+b",  // Exponential time complexity
	"isRegexp": true
}

// ✅ Safe regex
{
	"query": "a+b",
	"isRegexp": true
}
```

### 6. Set Reasonable maxResults

```typescript
// ❌ Too high - memory issues
{
	"query": "function",
	"isRegexp": false,
	"maxResults": 100000
}

// ✅ Reasonable limit
{
	"query": "function",
	"isRegexp": false,
	"maxResults": 100
}
```

## Limitations

### 1. No Semantic Understanding

**Limitation:** Text-based search cannot understand code semantics.

**Example:**
```typescript
// Search for "count"
{
	"query": "count",
	"isRegexp": false
}

// Matches ALL occurrences:
// - Variable: let count = 0;
// - String: "word count"
// - Comment: // count items
// - Method: array.count()
```

**Workaround:** Use `copilot_listCodeUsages` for semantic symbol search

### 2. Regex Complexity Limits

**Limitation:** Very complex regex patterns may timeout or fail.

**Examples:**
```typescript
// Catastrophic backtracking
"(a+)+b"

// Overly complex alternation
"(pattern1|pattern2|...|pattern100)"
```

### 3. Binary Files Not Searched

**Limitation:** Cannot search binary file content.

**Affected:** Images, compiled binaries, compressed files, etc.

### 4. Case Sensitivity Fixed

**Limitation:** Search is case-insensitive by default, cannot be changed per-query.

**Behavior:**
```typescript
// Always case-insensitive
{
	"query": "Function",
	"isRegexp": false
}
// Matches: function, Function, FUNCTION
```

### 5. No Multi-Line Regex by Default

**Limitation:** Regex patterns don't span multiple lines easily.

**Example:**
```typescript
// Won't match function spanning multiple lines well
{
	"query": "function\\s+\\w+\\s*\\([^)]*\\)\\s*{",
	"isRegexp": true
}
```

### 6. Performance with includeIgnoredFiles

**Limitation:** Searching ignored files can be very slow.

**Impact:**
- node_modules: Thousands of files
- dist/build: Large generated files
- .git: Many small files

### 7. Limited Context Size

**Limitation:** Context limited to configured number of lines.

**Default:** Usually 2-3 lines before/after
**Issue:** May not provide enough context for complex code

### 8. No Syntax-Aware Highlighting

**Limitation:** Results are plain text, no syntax highlighting.

**Output:**
```
> 45: function calculateTotal(items) {
```
(No color, no token highlighting)

### 9. Memory with Large Result Sets

**Limitation:** Large result sets consume significant memory.

**Impact:**
- 100 matches: ~50KB
- 1000 matches: ~500KB
- 10000 matches: ~5MB

### 10. No Replace Functionality

**Limitation:** Tool only searches, cannot replace matches.

**Note:** This is by design - replacement requires explicit user confirmation

---

**Related Documentation:**
- [copilot_searchCodebase](./01-copilot-searchCodebase.md)
- [copilot_findFiles](./04-copilot-findFiles.md)
- [copilot_listCodeUsages](./03-copilot-listCodeUsages.md)
- [copilot_readFile](./06-copilot-readFile.md)

**Last Updated:** 2025-01-22
