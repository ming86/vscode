# copilot_listDirectory Tool Documentation

## Overview

**Tool Name:** `copilot_listDirectory`

**Purpose:** Lists all files and subdirectories within a specified directory, providing a snapshot of the directory structure. Essential for workspace navigation, understanding project organization, exploring folder contents, and discovering available files and subdirectories.

**Implementation Location:** `src/extension/tools/codeSearch/listDirTool.ts`

**Tool Class:** `ListDirTool`

## Tool Declaration

```json
{
	"type": "function",
	"function": {
		"name": "copilot_listDirectory",
		"description": "List the contents of a directory. Result will have the name of the child. If the name ends in /, it's a folder, otherwise a file",
		"parameters": {
			"type": "object",
			"properties": {
				"path": {
					"type": "string",
					"description": "The absolute path to the directory to list."
				}
			},
			"required": [
				"path"
			]
		}
	}
}
```

## Input Parameters

### TypeScript Interface

```typescript
interface ListDirectoryInput {
	path: string;      // Required: Absolute path to directory
}
```

### Parameter Details

| Parameter | Type | Required | Default | Description |
|-----------|------|----------|---------|-------------|
| `path` | string | Yes | - | Absolute path to the directory (e.g., `/workspace/src`) |

### Parameter Examples

```typescript
// List workspace root
{
	"path": "/workspace"
}

// List source directory
{
	"path": "/workspace/src"
}

// List nested directory
{
	"path": "/workspace/src/components/ui"
}

// List configuration directory
{
	"path": "/workspace/config"
}

// List with relative path (resolved to workspace)
{
	"path": "src"
}
```

### Path Formats

```typescript
// Absolute paths (preferred)
"/workspace/src"
"/Users/user/project/src"
"C:\\Users\\user\\project\\src"

// URI format (supported)
"file:///workspace/src"

// Relative paths (resolved to workspace root)
"src"           // Resolved to /workspace/src
"./src"         // Resolved to /workspace/src
"src/utils"     // Resolved to /workspace/src/utils
```

## Architecture

### Flow Diagram

```
┌─────────────────────────────────────────────────────────────┐
│                   copilot_listDirectory                     │
│                     (ListDirTool)                           │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
                    ┌─────────────────┐
                    │ Validate Input  │
                    │ - Check path    │
                    │ - Parse format  │
                    └─────────────────┘
                              │
                              ▼
                ┌─────────────────────────────┐
                │ Resolve Directory Path      │
                │ - Convert to URI            │
                │ - Handle relative paths     │
                │ - Normalize separators      │
                └─────────────────────────────┘
                              │
                              ▼
                ┌─────────────────────────────┐
                │ Check Directory Exists      │
                │ - stat() directory          │
                │ - Verify is directory       │
                │ - Check permissions         │
                └─────────────────────────────┘
                              │
                              ▼
                ┌─────────────────────────────┐
                │ Read Directory Contents     │
                │ - IFileService.readdir()    │
                │ - Get all children          │
                │ - Retrieve metadata         │
                └─────────────────────────────┘
                              │
                              ▼
                ┌─────────────────────────────┐
                │ Classify Entries            │
                │ - Identify files            │
                │ - Identify directories      │
                │ - Get file metadata         │
                └─────────────────────────────┘
                              │
                              ▼
                ┌─────────────────────────────┐
                │ Sort & Organize             │
                │ - Directories first         │
                │ - Alphabetical order        │
                │ - Apply formatting          │
                └─────────────────────────────┘
                              │
                              ▼
                ┌─────────────────────────────┐
                │ Format Output               │
                │ - Create list               │
                │ - Add directory markers     │
                │ - Include metadata          │
                └─────────────────────────────┘
                              │
                              ▼
                    ┌─────────────────┐
                    │ Return Results  │
                    │ (Markdown List) │
                    └─────────────────┘
```

### Component Interaction

```
┌──────────────────┐
│  Language Model  │
│   (Claude/GPT)   │
└────────┬─────────┘
         │ Request: path
         ▼
┌──────────────────────────────────┐
│       ListDirTool                │
│  - validatePath()                │
│  - resolveURI()                  │
│  - listContents()                │
│  - formatOutput()                │
└────────┬─────────────────────────┘
         │
         ├─────────────────┐
         │                 │
         ▼                 ▼
┌──────────────────┐  ┌──────────────────┐
│  IFileService    │  │ IWorkspaceContext│
│  - readdir()     │  │  - resolve()     │
│  - stat()        │  │  - rootPath      │
└──────────────────┘  └──────────────────┘
         │
         ▼
┌──────────────────┐
│   File System    │
│   - Directory    │
│   - List ops     │
└──────────────────┘
```

## Internal Implementation

### Core Implementation

```typescript
export class ListDirTool extends LanguageModelTool<ListDirectoryInput> {
	static readonly ID = 'copilot_listDirectory';

	constructor(
		@IFileService private readonly fileService: IFileService,
		@IWorkspaceContextService private readonly workspaceService: IWorkspaceContextService,
		@IUriIdentityService private readonly uriIdentityService: IUriIdentityService
	) {
		super();
	}

	async invoke(
		input: ListDirectoryInput,
		token: CancellationToken
	): Promise<LanguageModelToolResult> {
		const { path } = input;

		// Step 1: Validate and resolve path
		const uri = this.resolveDirectoryURI(path);
		if (!uri) {
			return {
				content: [{
					kind: 'text',
					value: `Invalid directory path: ${path}`
				}]
			};
		}

		// Step 2: Check directory exists and is a directory
		try {
			const stat = await this.fileService.stat(uri);
			if (!stat.isDirectory) {
				return {
					content: [{
						kind: 'text',
						value: `Path is not a directory: ${path}`
					}]
				};
			}
		} catch (error) {
			return {
				content: [{
					kind: 'text',
					value: `Directory not found: ${path}`
				}]
			};
		}

		// Step 3: Read directory contents
		let entries: [string, FileType][];
		try {
			entries = await this.fileService.readdir(uri);
		} catch (error) {
			return {
				content: [{
					kind: 'text',
					value: `Error reading directory: ${error.message}`
				}]
			};
		}

		// Step 4: Format and return results
		if (entries.length === 0) {
			return {
				content: [{
					kind: 'text',
					value: `Directory is empty: ${path}`
				}]
			};
		}

		const markdown = this.formatOutput(uri, entries);

		return {
			content: [{
				kind: 'text',
				value: markdown
			}]
		};
	}

	private resolveDirectoryURI(dirPath: string): URI | undefined {
		try {
			// Handle URI format
			if (dirPath.startsWith('file://')) {
				return URI.parse(dirPath);
			}

			// Handle absolute paths
			if (path.isAbsolute(dirPath)) {
				return URI.file(dirPath);
			}

			// Handle relative paths
			const workspaceFolder = this.workspaceService.getWorkspace().folders[0];
			if (workspaceFolder) {
				const absolutePath = path.join(workspaceFolder.uri.fsPath, dirPath);
				return URI.file(absolutePath);
			}

			return undefined;
		} catch (error) {
			return undefined;
		}
	}

	private formatOutput(
		uri: URI,
		entries: [string, FileType][]
	): string {
		const relativePath = this.getRelativePath(uri);
		const lines: string[] = [
			`# Contents of \`${relativePath}\``,
			''
		];

		// Sort entries: directories first, then alphabetically
		const sortedEntries = this.sortEntries(entries);

		// Count directories and files
		const dirCount = sortedEntries.filter(([_, type]) =>
			type === FileType.Directory
		).length;
		const fileCount = sortedEntries.length - dirCount;

		lines.push(`${dirCount} director(ies), ${fileCount} file(s)`);
		lines.push('');

		// List entries
		for (const [name, type] of sortedEntries) {
			if (type === FileType.Directory) {
				lines.push(`- ${name}/`);
			} else {
				lines.push(`- ${name}`);
			}
		}

		return lines.join('\n');
	}

	private sortEntries(
		entries: [string, FileType][]
	): [string, FileType][] {
		return entries.sort((a, b) => {
			// Directories before files
			if (a[1] === FileType.Directory && b[1] !== FileType.Directory) {
				return -1;
			}
			if (a[1] !== FileType.Directory && b[1] === FileType.Directory) {
				return 1;
			}

			// Alphabetical within same type
			return a[0].localeCompare(b[0], undefined, {
				numeric: true,
				sensitivity: 'base'
			});
		});
	}

	private getRelativePath(uri: URI): string {
		const workspaceFolder = this.workspaceService.getWorkspaceFolder(uri);
		if (workspaceFolder) {
			const relativePath = uri.fsPath.substring(
				workspaceFolder.uri.fsPath.length + 1
			);
			return relativePath || '.';
		}
		return uri.fsPath;
	}
}

enum FileType {
	File = 1,
	Directory = 2,
	SymbolicLink = 64
}
```

### Enhanced Listing with Metadata

```typescript
class EnhancedListDirTool extends ListDirTool {
	private async getDetailedEntries(
		uri: URI,
		entries: [string, FileType][]
	): Promise<DetailedEntry[]> {
		const detailed: DetailedEntry[] = [];

		for (const [name, type] of entries) {
			const childUri = URI.joinPath(uri, name);

			try {
				const stat = await this.fileService.stat(childUri);

				detailed.push({
					name,
					type,
					size: stat.size,
					mtime: stat.mtime,
					isHidden: name.startsWith('.'),
					extension: type === FileType.File ? path.extname(name) : undefined
				});
			} catch {
				// If stat fails, include basic info
				detailed.push({
					name,
					type,
					isHidden: name.startsWith('.')
				});
			}
		}

		return detailed;
	}

	private formatDetailedOutput(
		relativePath: string,
		entries: DetailedEntry[]
	): string {
		const lines: string[] = [
			`# Contents of \`${relativePath}\``,
			''
		];

		// Group by type
		const directories = entries.filter(e => e.type === FileType.Directory);
		const files = entries.filter(e => e.type === FileType.File);

		// Directories section
		if (directories.length > 0) {
			lines.push(`## Directories (${directories.length})`);
			lines.push('');
			for (const dir of directories) {
				lines.push(`- ${dir.name}/`);
			}
			lines.push('');
		}

		// Files section
		if (files.length > 0) {
			lines.push(`## Files (${files.length})`);
			lines.push('');
			for (const file of files) {
				const sizeStr = file.size !== undefined
					? ` (${this.formatSize(file.size)})`
					: '';
				lines.push(`- ${file.name}${sizeStr}`);
			}
		}

		return lines.join('\n');
	}

	private formatSize(bytes: number): string {
		if (bytes < 1024) return `${bytes}B`;
		if (bytes < 1024 * 1024) return `${(bytes / 1024).toFixed(1)}KB`;
		return `${(bytes / (1024 * 1024)).toFixed(1)}MB`;
	}
}

interface DetailedEntry {
	name: string;
	type: FileType;
	size?: number;
	mtime?: number;
	isHidden?: boolean;
	extension?: string;
}
```

### Recursive Directory Listing

```typescript
class RecursiveListDirTool extends ListDirTool {
	async listRecursive(
		dirPath: string,
		maxDepth: number = 3
	): Promise<DirectoryTree> {
		const uri = this.resolveDirectoryURI(dirPath);
		if (!uri) {
			throw new Error('Invalid directory path');
		}

		return this.buildTree(uri, 0, maxDepth);
	}

	private async buildTree(
		uri: URI,
		currentDepth: number,
		maxDepth: number
	): Promise<DirectoryTree> {
		if (currentDepth >= maxDepth) {
			return {
				name: path.basename(uri.fsPath),
				path: uri.fsPath,
				type: 'directory',
				children: []
			};
		}

		const entries = await this.fileService.readdir(uri);
		const children: DirectoryTree[] = [];

		for (const [name, type] of entries) {
			const childUri = URI.joinPath(uri, name);

			if (type === FileType.Directory) {
				// Recursively list subdirectory
				const subtree = await this.buildTree(
					childUri,
					currentDepth + 1,
					maxDepth
				);
				children.push(subtree);
			} else {
				children.push({
					name,
					path: childUri.fsPath,
					type: 'file'
				});
			}
		}

		return {
			name: path.basename(uri.fsPath),
			path: uri.fsPath,
			type: 'directory',
			children
		};
	}

	private formatTree(tree: DirectoryTree, indent: string = ''): string {
		const lines: string[] = [];

		const marker = tree.type === 'directory' ? '📁' : '📄';
		const displayName = tree.type === 'directory'
			? `${tree.name}/`
			: tree.name;

		lines.push(`${indent}${marker} ${displayName}`);

		if (tree.children) {
			const sortedChildren = tree.children.sort((a, b) => {
				// Directories first
				if (a.type === 'directory' && b.type === 'file') return -1;
				if (a.type === 'file' && b.type === 'directory') return 1;
				return a.name.localeCompare(b.name);
			});

			for (let i = 0; i < sortedChildren.length; i++) {
				const child = sortedChildren[i];
				const isLast = i === sortedChildren.length - 1;
				const newIndent = indent + (isLast ? '    ' : '│   ');
				const prefix = isLast ? '└── ' : '├── ';

				const childLines = this.formatTree(child, newIndent);
				lines.push(prefix + childLines.split('\n')[0].trim());

				// Add child's children
				const restLines = childLines.split('\n').slice(1);
				lines.push(...restLines.map(line => newIndent + line));
			}
		}

		return lines.join('\n');
	}
}

interface DirectoryTree {
	name: string;
	path: string;
	type: 'file' | 'directory';
	children?: DirectoryTree[];
}
```

### Filtered Directory Listing

```typescript
class FilteredListDirTool extends ListDirTool {
	async invoke(
		input: ListDirectoryInput & {
			filter?: {
				extensions?: string[];
				excludeHidden?: boolean;
				excludePatterns?: string[];
			};
		},
		token: CancellationToken
	): Promise<LanguageModelToolResult> {
		const uri = this.resolveDirectoryURI(input.path);
		const entries = await this.fileService.readdir(uri!);

		// Apply filters
		const filtered = this.applyFilters(
			entries,
			input.filter
		);

		return this.formatOutput(uri!, filtered);
	}

	private applyFilters(
		entries: [string, FileType][],
		filter?: {
			extensions?: string[];
			excludeHidden?: boolean;
			excludePatterns?: string[];
		}
	): [string, FileType][] {
		let result = entries;

		// Filter by extension
		if (filter?.extensions) {
			result = result.filter(([name, type]) => {
				if (type === FileType.Directory) return true;
				const ext = path.extname(name);
				return filter.extensions!.includes(ext);
			});
		}

		// Exclude hidden files
		if (filter?.excludeHidden) {
			result = result.filter(([name]) => !name.startsWith('.'));
		}

		// Exclude patterns
		if (filter?.excludePatterns) {
			result = result.filter(([name]) => {
				return !filter.excludePatterns!.some(pattern => {
					return minimatch(name, pattern);
				});
			});
		}

		return result;
	}
}
```

## Dependencies

### Service Dependencies

```typescript
// Core VS Code services required
@IFileService                // File system operations
@IWorkspaceContextService    // Workspace path resolution
@IUriIdentityService         // URI comparison and normalization
```

### File Service Methods

```typescript
interface IFileService {
	/**
	 * Read directory contents
	 * Returns array of [name, type] tuples
	 */
	readdir(uri: URI): Promise<[string, FileType][]>;

	/**
	 * Get file/directory metadata
	 */
	stat(uri: URI): Promise<IFileStatWithMetadata>;

	/**
	 * Check if path exists
	 */
	exists(uri: URI): Promise<boolean>;
}

interface IFileStatWithMetadata {
	type: FileType;
	mtime: number;
	ctime: number;
	size: number;
	isDirectory: boolean;
	isFile: boolean;
	isSymbolicLink: boolean;
}

enum FileType {
	Unknown = 0,
	File = 1,
	Directory = 2,
	SymbolicLink = 64
}
```

## Performance Characteristics

### Time Complexity

| Operation | Complexity | Notes |
|-----------|------------|-------|
| Path resolution | O(1) | URI parsing |
| Directory stat | O(1) | Single file system call |
| Read directory | O(n) | n = number of entries |
| Sort entries | O(n log n) | Sorting algorithm |
| Format output | O(n) | Linear iteration |
| **Total** | **O(n log n)** | Dominated by sorting |

### Performance Metrics

```typescript
const performanceData = {
	small: {
		entries: '<50',
		time: '<50ms',
		memory: '<1KB'
	},
	medium: {
		entries: '50-500',
		time: '50-200ms',
		memory: '1-10KB'
	},
	large: {
		entries: '500-5000',
		time: '200-1000ms',
		memory: '10-100KB'
	},
	veryLarge: {
		entries: '>5000',
		time: '>1s',
		memory: '>100KB',
		note: 'Consider pagination or filtering'
	}
};
```

### Memory Usage

```typescript
// Approximate memory per directory listing
const memoryEstimate = {
	perEntry: {
		name: '~20 bytes',
		type: '1 byte',
		metadata: '~40 bytes (if included)',
		total: '~61 bytes per entry'
	},
	example: {
		small: '50 entries × 61 bytes = ~3KB',
		medium: '500 entries × 61 bytes = ~30KB',
		large: '5000 entries × 61 bytes = ~300KB'
	}
};
```

### Optimization Strategies

**1. Avoid Recursive Listing Unless Needed**

```typescript
// ❌ Slow - recursively lists all subdirectories
await listDirectoryRecursive("/workspace");

// ✅ Fast - lists only current directory
await listDirectory({ path: "/workspace" });
```

**2. Filter Results Early**

```typescript
// ❌ Fetch all, then filter in client
const all = await listDirectory({ path: "/workspace" });
const tsFiles = all.filter(name => name.endsWith('.ts'));

// ✅ Filter during listing (if supported)
const tsFiles = await listDirectory({
	path: "/workspace",
	filter: { extensions: ['.ts'] }
});
```

**3. Cache Directory Listings**

```typescript
// Cache frequently accessed directories
const cache = new Map<string, CacheEntry>();
const CACHE_TTL = 5000; // 5 seconds

async function cachedListDirectory(path: string) {
	const cached = cache.get(path);
	if (cached && Date.now() - cached.timestamp < CACHE_TTL) {
		return cached.result;
	}

	const result = await listDirectory({ path });
	cache.set(path, { result, timestamp: Date.now() });
	return result;
}
```

**4. Paginate Large Directories**

```typescript
// For directories with thousands of entries
async function paginatedList(
	path: string,
	pageSize: number = 100
): AsyncGenerator<Entry[]> {
	const allEntries = await listDirectory({ path });

	for (let i = 0; i < allEntries.length; i += pageSize) {
		yield allEntries.slice(i, i + pageSize);
	}
}
```

## Use Cases

### 1. Explore Workspace Structure

```typescript
// Query
{
	"path": "/workspace"
}

// Output
# Contents of `.`

3 director(ies), 5 file(s)

- src/
- test/
- node_modules/
- .gitignore
- package.json
- README.md
- tsconfig.json
- vite.config.ts
```

### 2. List Source Directory

```typescript
// Query
{
	"path": "/workspace/src"
}

// Output
# Contents of `src`

2 director(ies), 3 file(s)

- components/
- utils/
- app.ts
- config.ts
- index.ts
```

### 3. Explore Nested Directory

```typescript
// Query
{
	"path": "/workspace/src/components"
}

// Shows all components
```

### 4. Check Configuration Directory

```typescript
// Query
{
	"path": "/workspace/config"
}

// Lists all configuration files
```

### 5. Navigate Unknown Project

```typescript
// Step 1: List root
await listDirectory({ path: "/workspace" });

// Step 2: Explore subdirectory
await listDirectory({ path: "/workspace/src" });

// Step 3: Continue navigation
await listDirectory({ path: "/workspace/src/components" });
```

### 6. Find Available Test Directories

```typescript
// Query
{
	"path": "/workspace/test"
}

// Shows test organization structure
```

### 7. Check Build Output

```typescript
// Query
{
	"path": "/workspace/dist"
}

// Lists compiled/built files
```

### 8. Verify Package Structure

```typescript
// Query
{
	"path": "/workspace/packages"
}

// In monorepo, shows all packages
```

## Comparison with Other Tools

### vs copilot_findFiles

| Aspect | listDirectory | findFiles |
|--------|---------------|-----------|
| **Scope** | Single directory | Recursive workspace search |
| **Depth** | One level | All levels |
| **Pattern** | No filtering | Glob patterns |
| **Output** | All entries | Matching files only |
| **Speed** | Very fast | Moderate |

**Complementary use:**

```typescript
// listDirectory: Structure overview
await listDirectory({ path: "/workspace/src" });
// Result: components/, utils/, app.ts, config.ts

// findFiles: Pattern-based search
await findFiles({ query: "src/**/*.test.ts" });
// Result: All test files under src/
```

### vs copilot_readFile

| Aspect | listDirectory | readFile |
|--------|---------------|----------|
| **Target** | Directory contents | File content |
| **Output** | List of entries | File text |
| **Use Case** | Navigation | Reading |

**Typical workflow:**

```typescript
// Step 1: List directory
const entries = await listDirectory({ path: "/workspace/src" });
// Result: [app.ts, config.ts, ...]

// Step 2: Read specific file
const content = await readFile({ filePath: "/workspace/src/app.ts" });
// Result: File content
```

### vs copilot_searchCodebase

| Aspect | listDirectory | searchCodebase |
|--------|---------------|----------------|
| **Purpose** | Structure exploration | Content discovery |
| **Method** | File system listing | Semantic search |
| **Knowledge** | What files exist | What code contains |

### vs File Explorer

| Aspect | listDirectory | VS Code Explorer |
|--------|---------------|------------------|
| **Interface** | Programmatic | Visual UI |
| **Context** | AI conversation | Manual browsing |
| **Automation** | Easy to automate | Manual interaction |
| **Filtering** | Code-based | Extension-based |

## Error Handling

### Common Error Scenarios

**1. Directory Not Found**

```typescript
// Input
{
	"path": "/workspace/nonexistent"
}

// Output
Directory not found: /workspace/nonexistent
```

**2. Path is File, Not Directory**

```typescript
// Input
{
	"path": "/workspace/package.json"  // This is a file
}

// Output
Path is not a directory: /workspace/package.json
```

**3. Permission Denied**

```typescript
// Scenario: No read permission

// Error handling
try {
	const entries = await this.fileService.readdir(uri);
} catch (error) {
	if (error.code === 'EACCES') {
		return {
			content: [{
				kind: 'text',
				value: `Permission denied: ${path}`
			}]
		};
	}
	throw error;
}
```

**4. Empty Directory**

```typescript
// Input
{
	"path": "/workspace/empty-dir"
}

// Output
Directory is empty: /workspace/empty-dir
```

**5. Invalid Path Format**

```typescript
// Input
{
	"path": "invalid<>path"
}

// Output
Invalid directory path: invalid<>path
```

### Error Recovery Strategies

```typescript
class RobustListDirTool extends ListDirTool {
	async invoke(
		input: ListDirectoryInput,
		token: CancellationToken
	): Promise<LanguageModelToolResult> {
		try {
			return await super.invoke(input, token);
		} catch (error) {
			return this.handleError(error, input);
		}
	}

	private handleError(
		error: Error,
		input: ListDirectoryInput
	): LanguageModelToolResult {
		// Directory not found
		if (error.code === 'ENOENT') {
			return this.suggestAlternatives(input.path);
		}

		// Permission denied
		if (error.code === 'EACCES') {
			return {
				content: [{
					kind: 'text',
					value: `Permission denied: ${input.path}\n\n` +
					       `You may not have read access to this directory.`
				}]
			};
		}

		// Not a directory
		if (error.message.includes('not a directory')) {
			return {
				content: [{
					kind: 'text',
					value: `Path is a file, not a directory: ${input.path}\n\n` +
					       `Try copilot_readFile to read the file content.`
				}]
			};
		}

		// Generic error
		return {
			content: [{
				kind: 'text',
				value: `Error listing directory: ${error.message}`
			}]
		};
	}

	private suggestAlternatives(dirPath: string): LanguageModelToolResult {
		const parentDir = path.dirname(dirPath);
		const dirName = path.basename(dirPath);

		return {
			content: [{
				kind: 'text',
				value: `Directory not found: ${dirPath}\n\n` +
				       `Suggestions:\n` +
				       `- Check spelling and path format\n` +
				       `- List parent directory: copilot_listDirectory({ path: "${parentDir}" })\n` +
				       `- Search for directory: copilot_findFiles({ query: "**/${dirName}" })`
			}]
		};
	}
}
```

## Configuration

### Tool Settings

```json
{
	// Sort order
	"copilot.tools.listDirectory.sortOrder": "directoriesFirst",

	// Show hidden files
	"copilot.tools.listDirectory.showHidden": true,

	// Include metadata
	"copilot.tools.listDirectory.includeMetadata": false,

	// Maximum entries to return
	"copilot.tools.listDirectory.maxEntries": 1000,

	// Exclude patterns
	"copilot.tools.listDirectory.exclude": [
		"node_modules",
		".git",
		"dist",
		"build"
	],

	// Cache enabled
	"copilot.tools.listDirectory.cacheEnabled": true,
	"copilot.tools.listDirectory.cacheTTL": 5000
}
```

### VS Code File Settings

```json
{
	// Files to exclude from Explorer
	"files.exclude": {
		"**/.git": true,
		"**/.DS_Store": true,
		"**/node_modules": true
	},

	// Sort order in Explorer
	"explorer.sortOrder": "type",

	// Show hidden files
	"files.exclude": {
		"**/.*": false  // Show hidden files
	}
}
```

## Implementation Blueprint

### Step-by-Step Implementation

**Phase 1: Basic Structure**

```typescript
export class ListDirTool extends LanguageModelTool<ListDirectoryInput> {
	static readonly ID = 'copilot_listDirectory';

	async invoke(
		input: ListDirectoryInput,
		token: CancellationToken
	): Promise<LanguageModelToolResult> {
		// Resolve path
		const uri = this.resolveDirectoryURI(input.path);

		// Verify is directory
		await this.verifyDirectory(uri);

		// List contents
		const entries = await this.listContents(uri);

		// Format output
		return this.formatOutput(uri, entries);
	}
}
```

**Phase 2: Path Resolution**

```typescript
private resolveDirectoryURI(dirPath: string): URI {
	// URI format
	if (dirPath.startsWith('file://')) {
		return URI.parse(dirPath);
	}

	// Absolute path
	if (path.isAbsolute(dirPath)) {
		return URI.file(dirPath);
	}

	// Relative path
	const workspace = this.workspaceService.getWorkspace();
	if (workspace.folders.length > 0) {
		const root = workspace.folders[0].uri.fsPath;
		return URI.file(path.join(root, dirPath));
	}

	throw new Error('No workspace folder open');
}
```

**Phase 3: Directory Verification**

```typescript
private async verifyDirectory(uri: URI): Promise<void> {
	const stat = await this.fileService.stat(uri);

	if (!stat.isDirectory) {
		throw new Error('Path is not a directory');
	}
}
```

**Phase 4: List Contents**

```typescript
private async listContents(
	uri: URI
): Promise<[string, FileType][]> {
	const entries = await this.fileService.readdir(uri);
	return this.sortEntries(entries);
}

private sortEntries(
	entries: [string, FileType][]
): [string, FileType][] {
	return entries.sort((a, b) => {
		// Directories first
		if (a[1] === FileType.Directory && b[1] !== FileType.Directory) {
			return -1;
		}
		if (a[1] !== FileType.Directory && b[1] === FileType.Directory) {
			return 1;
		}
		// Alphabetical
		return a[0].localeCompare(b[0]);
	});
}
```

**Phase 5: Output Formatting**

```typescript
private formatOutput(
	uri: URI,
	entries: [string, FileType][]
): string {
	const lines: string[] = [];
	const relativePath = this.getRelativePath(uri);

	// Header
	lines.push(`# Contents of \`${relativePath}\``);
	lines.push('');

	// Summary
	const dirs = entries.filter(([_, type]) =>
		type === FileType.Directory
	).length;
	const files = entries.length - dirs;
	lines.push(`${dirs} director(ies), ${files} file(s)`);
	lines.push('');

	// Entries
	for (const [name, type] of entries) {
		const marker = type === FileType.Directory ? '/' : '';
		lines.push(`- ${name}${marker}`);
	}

	return lines.join('\n');
}
```

**Phase 6: Testing**

```typescript
describe('ListDirTool', () => {
	test('lists directory contents', async () => {
		const result = await tool.invoke(
			{ path: '/workspace/src' },
			token
		);

		expect(result.content[0].value).toContain('Contents of');
	});

	test('sorts directories first', async () => {
		const result = await tool.invoke(
			{ path: '/workspace' },
			token
		);

		const content = result.content[0].value;
		// Directories should appear before files
		const srcIndex = content.indexOf('src/');
		const pkgIndex = content.indexOf('package.json');
		expect(srcIndex).toBeLessThan(pkgIndex);
	});

	test('handles directory not found', async () => {
		const result = await tool.invoke(
			{ path: '/nonexistent' },
			token
		);

		expect(result.content[0].value).toContain('not found');
	});

	test('handles file path', async () => {
		const result = await tool.invoke(
			{ path: '/workspace/package.json' },
			token
		);

		expect(result.content[0].value).toContain('not a directory');
	});
});
```

## Best Practices

### 1. Use Absolute Paths

```typescript
// ✅ Absolute path (clear)
{
	"path": "/workspace/src"
}

// ⚠️ Relative path (ambiguous)
{
	"path": "src"
}
```

### 2. Navigate Incrementally

```typescript
// Good exploration pattern
// Step 1: List root
await listDirectory({ path: "/workspace" });

// Step 2: Drill down
await listDirectory({ path: "/workspace/src" });

// Step 3: Further exploration
await listDirectory({ path: "/workspace/src/components" });
```

### 3. Combine with Other Tools

```typescript
// Pattern: List → Read
// Step 1: List directory
const listing = await listDirectory({ path: "/workspace/config" });

// Step 2: Read specific config file
await readFile({ filePath: "/workspace/config/app.json" });
```

### 4. Cache for Repeated Access

```typescript
// Cache directory listings that don't change often
const cache = new Map();

async function getCachedListing(path: string) {
	if (!cache.has(path)) {
		cache.set(path, await listDirectory({ path }));
	}
	return cache.get(path);
}
```

### 5. Handle Empty Directories

```typescript
const result = await listDirectory({ path: "/workspace/empty" });

if (result.content[0].value.includes('empty')) {
	console.log('Directory has no contents');
}
```

## Limitations

### 1. Single Level Only

**Limitation:** Only lists immediate children, not recursive.

**Workaround:** Call multiple times for nested exploration, or use recursive implementation

### 2. No Filtering Options

**Limitation:** Returns all entries, no built-in filtering.

**Impact:** May return many irrelevant entries

**Workaround:** Post-process results or implement filtered variant

### 3. No Metadata by Default

**Limitation:** Basic listing doesn't include file sizes, dates, etc.

**Available:** File type (file vs directory) only

**Enhancement:** Implement enhanced variant with metadata

### 4. Performance with Large Directories

**Limitation:** Slow with thousands of entries.

**Threshold:**
- <100 entries: Fast
- 100-1000: Moderate
- >1000: Consider pagination

### 5. No Symbolic Link Resolution

**Limitation:** Symbolic links shown but not followed by default.

**Behavior:** Symlinks marked with FileType.SymbolicLink

### 6. No Pattern Matching

**Limitation:** Cannot filter by glob patterns directly.

**Alternative:** Use `copilot_findFiles` for pattern-based search

### 7. Network File System Latency

**Limitation:** Slower on remote/network file systems.

**Impact:** Additional network round-trip time

### 8. No Sorting Options

**Limitation:** Fixed sort order (directories first, then alphabetical).

**No Control:** Cannot change sort criteria per request

### 9. Unicode Handling

**Limitation:** May have issues with certain unicode filenames.

**Depends On:** File system and encoding support

### 10. No Permissions Info

**Limitation:** Doesn't show file permissions or ownership.

**Available:** Only existence and type

---

**Related Documentation:**
- [copilot_readFile](./06-copilot-readFile.md)
- [copilot_findFiles](./04-copilot-findFiles.md)
- [copilot_findTextInFiles](./05-copilot-findTextInFiles.md)
- [copilot_searchCodebase](./01-copilot-searchCodebase.md)

**Last Updated:** 2025-01-22
