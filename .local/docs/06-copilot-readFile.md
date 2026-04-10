# copilot_readFile Tool Documentation

## Overview

**Tool Name:** `copilot_readFile`

**Purpose:** Reads and returns the complete contents of a specific file from the workspace. Essential for accessing source code, configuration files, documentation, and any text-based file content. Provides the foundation for code analysis, understanding implementation details, and generating context-aware responses.

**Implementation Location:** `src/extension/tools/codeSearch/readFileTool.ts`

**Tool Class:** `ReadFileTool`

## Tool Declaration

```json
{
	"type": "function",
	"function": {
		"name": "copilot_readFile",
		"description": "Read the contents of a file. Line numbers are 1-indexed. This tool will truncate its output at 2000 lines and may be called repeatedly with offset and limit parameters to read larger files in chunks.",
		"parameters": {
			"type": "object",
			"properties": {
				"filePath": {
					"type": "string",
					"description": "The absolute path of the file to read."
				},
				"offset": {
					"type": "number",
					"description": "Optional: the 1-based line number to start reading from. Only use this if the file is too large to read at once. If not specified, the file will be read from the beginning."
				},
				"limit": {
					"type": "number",
					"description": "Optional: the maximum number of lines to read. Only use this together with `offset` if the file is too large to read at once."
				}
			},
			"required": [
				"filePath"
			]
		}
	}
}
```

## Input Parameters

### TypeScript Interface

```typescript
interface ReadFileInput {
	filePath: string;      // Required: Absolute path to file
	offset?: number;       // Optional: 1-based line number to start from
	limit?: number;        // Optional: Maximum lines to read
}
```

### Parameter Details

| Parameter | Type | Required | Default | Description |
|-----------|------|----------|---------|-------------|
| `filePath` | string | Yes | - | Absolute path to the file (e.g., `/workspace/src/app.ts`) |
| `offset` | number | No | 1 | 1-based line number to start reading from |
| `limit` | number | No | 2000 | Maximum number of lines to read |

### Parameter Examples

```typescript
// Read entire file (up to 2000 lines)
{
	"filePath": "/workspace/src/app.ts"
}

// Read specific line range
{
	"filePath": "/workspace/src/utils/helpers.ts",
	"offset": 50,
	"limit": 100
}

// Read first 500 lines
{
	"filePath": "/workspace/package.json",
	"limit": 500
}

// Read from line 1000 to 1500
{
	"filePath": "/workspace/src/large-file.ts",
	"offset": 1000,
	"limit": 500
}

// Configuration file
{
	"filePath": "/workspace/.eslintrc.json"
}
```

### Path Formats

```typescript
// Absolute paths (preferred)
"/workspace/src/app.ts"
"/Users/user/project/file.js"
"C:\\Users\\user\\project\\file.js"

// URI format (supported)
"file:///workspace/src/app.ts"

// Relative paths (resolved to workspace)
"src/app.ts"  // Resolved to /workspace/src/app.ts
"./file.js"   // Resolved to /workspace/file.js
```

## Architecture

### Flow Diagram

```
┌─────────────────────────────────────────────────────────────┐
│                     copilot_readFile                        │
│                      (ReadFileTool)                         │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
                    ┌─────────────────┐
                    │ Validate Input  │
                    │ - Check path    │
                    │ - Verify params │
                    └─────────────────┘
                              │
                              ▼
                ┌─────────────────────────────┐
                │ Resolve File Path           │
                │ - Convert to URI            │
                │ - Handle relative paths     │
                │ - Normalize separators      │
                └─────────────────────────────┘
                              │
                              ▼
                ┌─────────────────────────────┐
                │ Check File Existence        │
                │ - stat() file               │
                │ - Verify readable           │
                │ - Check file type           │
                └─────────────────────────────┘
                              │
                              ▼
                ┌─────────────────────────────┐
                │ Read File Content           │
                │ - IFileService.readFile()   │
                │ - Decode to UTF-8           │
                │ - Split into lines          │
                └─────────────────────────────┘
                              │
                              ▼
                ┌─────────────────────────────┐
                │ Apply Range Selection       │
                │ - Calculate start line      │
                │ - Calculate end line        │
                │ - Extract line range        │
                └─────────────────────────────┘
                              │
                              ▼
                ┌─────────────────────────────┐
                │ Format Output               │
                │ - Add line numbers          │
                │ - Create markdown           │
                │ - Add metadata              │
                └─────────────────────────────┘
                              │
                              ▼
                    ┌─────────────────┐
                    │ Return Content  │
                    │ (Markdown)      │
                    └─────────────────┘
```

### Component Interaction

```
┌──────────────────┐
│  Language Model  │
│   (Claude/GPT)   │
└────────┬─────────┘
         │ Request: filePath, offset?, limit?
         ▼
┌──────────────────────────────────┐
│       ReadFileTool               │
│  - validatePath()                │
│  - resolveURI()                  │
│  - readFileContent()             │
│  - formatOutput()                │
└────────┬─────────────────────────┘
         │
         ├─────────────────┐
         │                 │
         ▼                 ▼
┌──────────────────┐  ┌──────────────────┐
│  IFileService    │  │ IWorkspaceContext│
│  - readFile()    │  │  - resolve()     │
│  - stat()        │  │  - rootPath      │
│  - exists()      │  └──────────────────┘
└──────────────────┘
         │
         ▼
┌──────────────────┐
│   File System    │
│   - Read ops     │
│   - Encoding     │
└──────────────────┘
```

## Internal Implementation

### Core Implementation

```typescript
export class ReadFileTool extends LanguageModelTool<ReadFileInput> {
	static readonly ID = 'copilot_readFile';
	private static readonly MAX_LINES = 2000;
	private static readonly DEFAULT_LIMIT = 2000;

	constructor(
		@IFileService private readonly fileService: IFileService,
		@IWorkspaceContextService private readonly workspaceService: IWorkspaceContextService,
		@IUriIdentityService private readonly uriIdentityService: IUriIdentityService
	) {
		super();
	}

	async invoke(
		input: ReadFileInput,
		token: CancellationToken
	): Promise<LanguageModelToolResult> {
		const { filePath, offset, limit } = input;

		// Step 1: Validate and resolve file path
		const uri = this.resolveFileURI(filePath);
		if (!uri) {
			return {
				content: [{
					kind: 'text',
					value: `Invalid file path: ${filePath}`
				}]
			};
		}

		// Step 2: Check file exists
		try {
			const stat = await this.fileService.stat(uri);
			if (stat.isDirectory) {
				return {
					content: [{
						kind: 'text',
						value: `Path is a directory, not a file: ${filePath}`
					}]
				};
			}
		} catch (error) {
			return {
				content: [{
					kind: 'text',
					value: `File not found: ${filePath}`
				}]
			};
		}

		// Step 3: Read file content
		let content: string;
		try {
			const fileContent = await this.fileService.readFile(uri);
			content = fileContent.value.toString();
		} catch (error) {
			return {
				content: [{
					kind: 'text',
					value: `Error reading file: ${error.message}`
				}]
			};
		}

		// Step 4: Split into lines
		const lines = content.split('\n');
		const totalLines = lines.length;

		// Step 5: Apply offset and limit
		const startLine = Math.max(1, offset ?? 1);
		const readLimit = limit ?? ReadFileTool.DEFAULT_LIMIT;
		const endLine = Math.min(totalLines, startLine + readLimit - 1);

		// Validate range
		if (startLine > totalLines) {
			return {
				content: [{
					kind: 'text',
					value: `Offset ${startLine} exceeds file length (${totalLines} lines)`
				}]
			};
		}

		// Step 6: Extract lines (convert to 0-indexed)
		const selectedLines = lines.slice(startLine - 1, endLine);

		// Step 7: Format output
		const markdown = this.formatOutput(
			uri,
			selectedLines,
			startLine,
			endLine,
			totalLines
		);

		return {
			content: [{
				kind: 'text',
				value: markdown
			}]
		};
	}

	private resolveFileURI(filePath: string): URI | undefined {
		try {
			// Handle URI format
			if (filePath.startsWith('file://')) {
				return URI.parse(filePath);
			}

			// Handle absolute paths
			if (path.isAbsolute(filePath)) {
				return URI.file(filePath);
			}

			// Handle relative paths
			const workspaceFolder = this.workspaceService.getWorkspace().folders[0];
			if (workspaceFolder) {
				const absolutePath = path.join(workspaceFolder.uri.fsPath, filePath);
				return URI.file(absolutePath);
			}

			return undefined;
		} catch (error) {
			return undefined;
		}
	}

	private formatOutput(
		uri: URI,
		lines: string[],
		startLine: number,
		endLine: number,
		totalLines: number
	): string {
		const relativePath = this.getRelativePath(uri);
		const output: string[] = [];

		// Header
		output.push(`# ${relativePath}`);
		output.push('');

		// Metadata
		if (startLine > 1 || endLine < totalLines) {
			output.push(`*Showing lines ${startLine}-${endLine} of ${totalLines} total*`);
			output.push('');
		} else {
			output.push(`*${totalLines} lines*`);
			output.push('');
		}

		// Content with line numbers
		output.push('```');
		for (let i = 0; i < lines.length; i++) {
			const lineNumber = startLine + i;
			output.push(`${lineNumber}: ${lines[i]}`);
		}
		output.push('```');

		// Pagination info
		if (endLine < totalLines) {
			const remainingLines = totalLines - endLine;
			output.push('');
			output.push(`*${remainingLines} more lines available. Use offset=${endLine + 1} to continue reading.*`);
		}

		return output.join('\n');
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
}
```

### Chunked Reading for Large Files

```typescript
class ChunkedFileReader {
	private static readonly CHUNK_SIZE = 2000; // lines

	async readInChunks(
		tool: ReadFileTool,
		filePath: string,
		processor: (chunk: string[], startLine: number) => Promise<void>
	): Promise<void> {
		let offset = 1;
		let hasMore = true;

		while (hasMore) {
			const result = await tool.invoke({
				filePath,
				offset,
				limit: ChunkedFileReader.CHUNK_SIZE
			}, CancellationToken.None);

			const content = result.content[0].value;
			const lines = this.extractLines(content);

			await processor(lines, offset);

			// Check if there are more lines
			hasMore = content.includes('more lines available');
			offset += ChunkedFileReader.CHUNK_SIZE;
		}
	}

	private extractLines(markdown: string): string[] {
		// Extract code block content
		const match = markdown.match(/```\n([\s\S]*?)\n```/);
		if (!match) return [];

		const codeBlock = match[1];
		return codeBlock.split('\n').map(line => {
			// Remove line numbers
			return line.replace(/^\d+:\s/, '');
		});
	}
}
```

### Binary File Detection

```typescript
class SafeFileReader extends ReadFileTool {
	async invoke(
		input: ReadFileInput,
		token: CancellationToken
	): Promise<LanguageModelToolResult> {
		const uri = this.resolveFileURI(input.filePath);
		if (!uri) {
			return this.errorResult('Invalid file path');
		}

		// Check if file is binary
		if (await this.isBinaryFile(uri)) {
			return {
				content: [{
					kind: 'text',
					value: `Cannot read binary file: ${input.filePath}\n\n` +
					       `File appears to be a binary file (image, executable, etc.)`
				}]
			};
		}

		return super.invoke(input, token);
	}

	private async isBinaryFile(uri: URI): Promise<boolean> {
		try {
			// Read first 8KB to detect binary content
			const sample = await this.fileService.readFile(uri, { length: 8192 });
			const buffer = sample.value;

			// Check for null bytes (common in binary files)
			for (let i = 0; i < buffer.byteLength; i++) {
				if (buffer[i] === 0) {
					return true;
				}
			}

			// Check file extension
			const binaryExtensions = [
				'.exe', '.dll', '.so', '.dylib',
				'.png', '.jpg', '.jpeg', '.gif', '.bmp',
				'.pdf', '.zip', '.tar', '.gz',
				'.mp3', '.mp4', '.avi', '.mov'
			];

			const ext = path.extname(uri.fsPath).toLowerCase();
			return binaryExtensions.includes(ext);

		} catch {
			return false;
		}
	}
}
```

### Encoding Detection

```typescript
class EncodingAwareFileReader extends ReadFileTool {
	private async detectEncoding(uri: URI): Promise<string> {
		// Check VS Code's file encoding
		const encoding = this.configService.getValue<string>(
			'files.encoding',
			{ resource: uri }
		);

		if (encoding) {
			return encoding;
		}

		// Try to detect from BOM
		const sample = await this.fileService.readFile(uri, { length: 4 });
		const bom = sample.value;

		// UTF-8 BOM
		if (bom[0] === 0xEF && bom[1] === 0xBB && bom[2] === 0xBF) {
			return 'utf8bom';
		}

		// UTF-16 LE BOM
		if (bom[0] === 0xFF && bom[1] === 0xFE) {
			return 'utf16le';
		}

		// UTF-16 BE BOM
		if (bom[0] === 0xFE && bom[1] === 0xFF) {
			return 'utf16be';
		}

		// Default to UTF-8
		return 'utf8';
	}

	async invoke(
		input: ReadFileInput,
		token: CancellationToken
	): Promise<LanguageModelToolResult> {
		const uri = this.resolveFileURI(input.filePath);
		const encoding = await this.detectEncoding(uri);

		// Read with detected encoding
		const content = await this.fileService.readFile(uri, { encoding });

		// Continue with normal processing
		return this.processContent(content.value.toString(), input);
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
@IConfigurationService       // Optional: encoding settings
```

### File Service Methods

```typescript
interface IFileService {
	/**
	 * Read file content
	 */
	readFile(
		uri: URI,
		options?: {
			encoding?: string;
			length?: number;
			position?: number;
		}
	): Promise<IFileContent>;

	/**
	 * Get file metadata
	 */
	stat(uri: URI): Promise<IFileStatWithMetadata>;

	/**
	 * Check if file exists
	 */
	exists(uri: URI): Promise<boolean>;
}

interface IFileContent {
	value: VSBuffer;
	encoding: string;
	etag: string;
	mtime: number;
}
```

### Configuration Keys

```typescript
// File encoding settings
const encodingSettings = {
	defaultEncoding: 'files.encoding',
	autoGuessEncoding: 'files.autoGuessEncoding',
	eol: 'files.eol'
};

// Example configuration
{
	"files.encoding": "utf8",
	"files.autoGuessEncoding": true,
	"files.eol": "\n"
}
```

## Performance Characteristics

### Time Complexity

| Operation | Complexity | Notes |
|-----------|------------|-------|
| Path resolution | O(1) | URI parsing |
| File existence check | O(1) | File system stat |
| File read | O(n) | n = file size |
| Line splitting | O(n) | n = file size |
| Range extraction | O(m) | m = selected lines |
| Formatting | O(m) | m = output lines |

### Performance Metrics

```typescript
const performanceData = {
	small: {
		size: '<100 lines',
		readTime: '<50ms',
		memory: '<10KB'
	},
	medium: {
		size: '100-1000 lines',
		readTime: '50-200ms',
		memory: '10-100KB'
	},
	large: {
		size: '1000-10000 lines',
		readTime: '200-1000ms',
		memory: '100KB-1MB'
	},
	veryLarge: {
		size: '>10000 lines',
		readTime: '1s+',
		memory: '1MB+',
		recommendation: 'Use chunked reading'
	}
};
```

### Memory Usage

```typescript
// Approximate memory per file read
const memoryEstimate = {
	fileContent: 'fileSize bytes',
	lineArray: 'fileSize + (lineCount × 40) bytes',
	formatted: 'selectedLines × 80 bytes',
	total: 'Roughly 2-3× file size'
};

// Example: 1000-line file, 50KB
// - Content: 50KB
// - Lines array: 50KB + 40KB = 90KB
// - Formatted: 40KB
// - Total: ~180KB
```

### Optimization Strategies

**1. Use Chunked Reading for Large Files**

```typescript
// ❌ Read entire large file
{
	"filePath": "/workspace/large-file.ts"  // 10000 lines
}

// ✅ Read in chunks
{
	"filePath": "/workspace/large-file.ts",
	"offset": 1,
	"limit": 500
}
// Then read next chunk with offset: 501
```

**2. Read Only Needed Lines**

```typescript
// ❌ Read entire file when only need specific function
{
	"filePath": "/workspace/utils.ts"  // 5000 lines
}

// ✅ Read specific range (if known)
{
	"filePath": "/workspace/utils.ts",
	"offset": 450,
	"limit": 50
}
```

**3. Cache Frequently Accessed Files**

```typescript
class CachedReadFileTool extends ReadFileTool {
	private cache = new Map<string, CacheEntry>();
	private readonly CACHE_TTL = 5000; // 5 seconds

	async invoke(input: ReadFileInput, token: CancellationToken) {
		const cacheKey = this.getCacheKey(input);
		const cached = this.cache.get(cacheKey);

		if (cached && !this.isExpired(cached)) {
			return cached.result;
		}

		const result = await super.invoke(input, token);
		this.cache.set(cacheKey, {
			result,
			timestamp: Date.now()
		});

		return result;
	}
}
```

## Use Cases

### 1. Read Source File

```typescript
// Query
{
	"filePath": "/workspace/src/app.ts"
}

// Output
# src/app.ts

*523 lines*

```
1: import express from 'express';
2: import { config } from './config';
3:
4: const app = express();
5:
6: app.get('/', (req, res) => {
7:   res.send('Hello World');
8: });
...
```

### 2. Read Configuration File

```typescript
// Query
{
	"filePath": "/workspace/package.json"
}

// Output shows package.json content with line numbers
```

### 3. Read Specific Function

```typescript
// Combined with search to find function location
// Step 1: Find function with findTextInFiles
// Step 2: Read that section
{
	"filePath": "/workspace/src/utils.ts",
	"offset": 150,
	"limit": 50
}
```

### 4. Read Large File in Chunks

```typescript
// First chunk
{
	"filePath": "/workspace/large-file.ts",
	"offset": 1,
	"limit": 1000
}

// Second chunk
{
	"filePath": "/workspace/large-file.ts",
	"offset": 1001,
	"limit": 1000
}
```

### 5. Read Documentation

```typescript
// Query
{
	"filePath": "/workspace/README.md"
}

// Returns markdown content with line numbers
```

### 6. Read Test File

```typescript
// Query
{
	"filePath": "/workspace/src/utils.test.ts"
}

// Analyze test cases
```

### 7. Read Build Configuration

```typescript
// Query
{
	"filePath": "/workspace/tsconfig.json"
}

// Understand TypeScript configuration
```

### 8. Read Environment File

```typescript
// Query
{
	"filePath": "/workspace/.env.example"
}

// See environment variable structure
```

## Comparison with Other Tools

### vs copilot_findTextInFiles

| Aspect | readFile | findTextInFiles |
|--------|----------|-----------------|
| **Purpose** | Read entire file | Search for text patterns |
| **Output** | Full file content | Matching snippets |
| **Use Case** | File review | Pattern discovery |
| **Speed** | Fast | Moderate |
| **Context** | Complete file | Match surroundings |

**Workflow:**
```typescript
// 1. Search for pattern
findTextInFiles({ query: "handleRequest" })
// Result: Found in src/handlers.ts, line 45

// 2. Read full context
readFile({ filePath: "/workspace/src/handlers.ts" })
// Result: Complete file content
```

### vs copilot_listDirectory

| Aspect | readFile | listDirectory |
|--------|----------|---------------|
| **Target** | File content | Directory contents |
| **Output** | Text content | File/folder list |
| **Use** | Read data | Navigate structure |

**Complementary use:**
```typescript
// 1. List directory
listDirectory({ path: "/workspace/src" })
// Result: [app.ts, utils.ts, config.ts]

// 2. Read specific file
readFile({ filePath: "/workspace/src/app.ts" })
// Result: File content
```

### vs copilot_searchCodebase

| Aspect | readFile | searchCodebase |
|--------|----------|----------------|
| **Precision** | Exact file access | Semantic search |
| **Input** | Known file path | Query/concept |
| **Output** | Complete file | Relevant chunks |

**When to use each:**
- **readFile**: Known file, need full content
- **searchCodebase**: Exploring, finding relevant code

## Error Handling

### Common Error Scenarios

**1. File Not Found**

```typescript
// Input
{
	"filePath": "/workspace/nonexistent.ts"
}

// Output
File not found: /workspace/nonexistent.ts
```

**2. Path is Directory**

```typescript
// Input
{
	"filePath": "/workspace/src"
}

// Output
Path is a directory, not a file: /workspace/src
```

**3. Invalid Offset**

```typescript
// Input
{
	"filePath": "/workspace/app.ts",  // 500 lines
	"offset": 1000
}

// Output
Offset 1000 exceeds file length (500 lines)
```

**4. Permission Denied**

```typescript
// Scenario: No read permission

// Error handling
try {
	const content = await this.fileService.readFile(uri);
} catch (error) {
	if (error.code === 'EACCES') {
		return {
			content: [{
				kind: 'text',
				value: `Permission denied: ${filePath}`
			}]
		};
	}
	throw error;
}
```

**5. Binary File**

```typescript
// Input
{
	"filePath": "/workspace/image.png"
}

// Output
Cannot read binary file: /workspace/image.png

File appears to be a binary file (image, executable, etc.)
```

**6. File Too Large**

```typescript
// Input
{
	"filePath": "/workspace/huge-file.log"  // 100MB
}

// Output (with automatic chunking)
# huge-file.log

*Showing lines 1-2000 of 1000000 total*

```
1: [First log line]
...
2000: [Line 2000]
```

*999998 more lines available. Use offset=2001 to continue reading.*
```

### Error Recovery Strategies

```typescript
class RobustReadFileTool extends ReadFileTool {
	async invoke(
		input: ReadFileInput,
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
		input: ReadFileInput
	): LanguageModelToolResult {
		// File not found
		if (error.code === 'ENOENT') {
			return this.suggestAlternatives(input.filePath);
		}

		// Permission denied
		if (error.code === 'EACCES') {
			return {
				content: [{
					kind: 'text',
					value: `Permission denied: ${input.filePath}\n\n` +
					       `Check file permissions or try a different file.`
				}]
			};
		}

		// Encoding error
		if (error.message.includes('encoding')) {
			return {
				content: [{
					kind: 'text',
					value: `Encoding error: ${input.filePath}\n\n` +
					       `File may be binary or use unsupported encoding.`
				}]
			};
		}

		// Generic error
		return {
			content: [{
				kind: 'text',
				value: `Error reading file: ${error.message}`
			}]
		};
	}

	private suggestAlternatives(filePath: string): LanguageModelToolResult {
		// Suggest similar files
		const directory = path.dirname(filePath);
		const filename = path.basename(filePath);

		return {
			content: [{
				kind: 'text',
				value: `File not found: ${filePath}\n\n` +
				       `Try:\n` +
				       `- Check the file path spelling\n` +
				       `- Use copilot_findFiles to locate the file\n` +
				       `- List directory: copilot_listDirectory({ path: "${directory}" })`
			}]
		};
	}
}
```

## Configuration

### Tool Settings

```json
{
	// Maximum lines per read
	"copilot.tools.readFile.maxLines": 2000,

	// Default encoding
	"copilot.tools.readFile.encoding": "utf8",

	// Auto-detect encoding
	"copilot.tools.readFile.autoDetectEncoding": true,

	// Binary file detection
	"copilot.tools.readFile.detectBinary": true,

	// Maximum file size (bytes)
	"copilot.tools.readFile.maxFileSize": 10485760, // 10MB

	// Cache read files
	"copilot.tools.readFile.cacheEnabled": true,
	"copilot.tools.readFile.cacheTTL": 5000,

	// Line number formatting
	"copilot.tools.readFile.showLineNumbers": true,
	"copilot.tools.readFile.lineNumberPadding": true
}
```

### VS Code File Settings

```json
{
	// Default file encoding
	"files.encoding": "utf8",

	// Auto-guess encoding
	"files.autoGuessEncoding": true,

	// End of line character
	"files.eol": "\n",

	// Large file size warning (MB)
	"files.largeFileOptimizations": true
}
```

## Implementation Blueprint

### Step-by-Step Implementation

**Phase 1: Basic Structure**

```typescript
export class ReadFileTool extends LanguageModelTool<ReadFileInput> {
	static readonly ID = 'copilot_readFile';
	static readonly MAX_LINES = 2000;

	async invoke(
		input: ReadFileInput,
		token: CancellationToken
	): Promise<LanguageModelToolResult> {
		// Validate input
		const uri = this.resolveFileURI(input.filePath);
		if (!uri) {
			return this.errorResult('Invalid file path');
		}

		// Read file
		const content = await this.readFile(uri);

		// Apply range
		const lines = this.applyRange(
			content,
			input.offset,
			input.limit
		);

		// Format output
		return this.formatOutput(uri, lines);
	}
}
```

**Phase 2: Path Resolution**

```typescript
private resolveFileURI(filePath: string): URI | undefined {
	try {
		// URI format
		if (filePath.startsWith('file://')) {
			return URI.parse(filePath);
		}

		// Absolute path
		if (path.isAbsolute(filePath)) {
			return URI.file(filePath);
		}

		// Relative path
		const workspace = this.workspaceService.getWorkspace();
		if (workspace.folders.length > 0) {
			const root = workspace.folders[0].uri.fsPath;
			return URI.file(path.join(root, filePath));
		}

		return undefined;
	} catch {
		return undefined;
	}
}
```

**Phase 3: File Reading**

```typescript
private async readFile(uri: URI): Promise<string> {
	// Check exists
	const exists = await this.fileService.exists(uri);
	if (!exists) {
		throw new Error('File not found');
	}

	// Check is file
	const stat = await this.fileService.stat(uri);
	if (stat.isDirectory) {
		throw new Error('Path is a directory');
	}

	// Read content
	const fileContent = await this.fileService.readFile(uri);
	return fileContent.value.toString();
}
```

**Phase 4: Range Application**

```typescript
private applyRange(
	content: string,
	offset?: number,
	limit?: number
): string[] {
	const lines = content.split('\n');
	const totalLines = lines.length;

	const start = Math.max(1, offset ?? 1);
	const count = limit ?? ReadFileTool.MAX_LINES;
	const end = Math.min(totalLines, start + count - 1);

	// Extract range (convert to 0-indexed)
	return lines.slice(start - 1, end);
}
```

**Phase 5: Output Formatting**

```typescript
private formatOutput(
	uri: URI,
	lines: string[],
	startLine: number,
	totalLines: number
): string {
	const output: string[] = [];

	// Header
	output.push(`# ${this.getRelativePath(uri)}`);
	output.push('');
	output.push(`*${totalLines} lines*`);
	output.push('');

	// Content
	output.push('```');
	lines.forEach((line, idx) => {
		output.push(`${startLine + idx}: ${line}`);
	});
	output.push('```');

	return output.join('\n');
}
```

**Phase 6: Testing**

```typescript
describe('ReadFileTool', () => {
	test('reads entire file', async () => {
		const result = await tool.invoke({
			filePath: '/workspace/test.ts'
		}, token);

		expect(result.content[0].value).toContain('test.ts');
	});

	test('reads with offset and limit', async () => {
		const result = await tool.invoke({
			filePath: '/workspace/test.ts',
			offset: 10,
			limit: 5
		}, token);

		const content = result.content[0].value;
		expect(content).toContain('10:');
		expect(content).toContain('14:');
	});

	test('handles file not found', async () => {
		const result = await tool.invoke({
			filePath: '/workspace/nonexistent.ts'
		}, token);

		expect(result.content[0].value).toContain('not found');
	});
});
```

## Best Practices

### 1. Use Absolute Paths

```typescript
// ✅ Absolute path (clear, unambiguous)
{
	"filePath": "/workspace/src/app.ts"
}

// ⚠️ Relative path (depends on workspace root)
{
	"filePath": "src/app.ts"
}
```

### 2. Read Large Files in Chunks

```typescript
// ❌ Try to read 50,000-line file at once
{
	"filePath": "/workspace/large.log"
}

// ✅ Read in manageable chunks
{
	"filePath": "/workspace/large.log",
	"offset": 1,
	"limit": 1000
}
```

### 3. Combine with Search Tools

```typescript
// Workflow: Find then Read
// Step 1: Find the file
await findFiles({ query: "**/*config.ts" });

// Step 2: Read the file
await readFile({ filePath: "/workspace/config/app.config.ts" });
```

### 4. Cache for Repeated Access

```typescript
// If reading same file multiple times, use caching
const cache = new Map();

async function cachedRead(filePath: string) {
	if (cache.has(filePath)) {
		return cache.get(filePath);
	}

	const result = await readFile({ filePath });
	cache.set(filePath, result);
	return result;
}
```

### 5. Handle Errors Gracefully

```typescript
try {
	const result = await readFile({ filePath });
	// Process result
} catch (error) {
	// Suggest alternatives
	console.log('File not found. Try:');
	console.log('- Use findFiles to locate it');
	console.log('- Check path spelling');
}
```

## Limitations

### 1. 2000-Line Truncation

**Limitation:** Files are automatically truncated at 2000 lines.

**Mitigation:** Use `offset` and `limit` for chunked reading.

### 2. Binary Files Not Supported

**Limitation:** Cannot read binary file content.

**Affected:** Images, executables, compiled binaries, PDFs, etc.

### 3. No Syntax Highlighting

**Limitation:** Output is plain text with line numbers.

**No:** Color coding, token highlighting, semantic colors

### 4. Memory Consumption

**Limitation:** Entire file loaded into memory.

**Impact:**
- 1MB file: 2-3MB memory
- 10MB file: 20-30MB memory
- 100MB file: May fail

### 5. No Real-Time Updates

**Limitation:** Returns snapshot at read time.

**Behavior:** Changes after read not reflected

### 6. Line Ending Handling

**Limitation:** All line endings normalized to `\n`.

**Behavior:**
- Windows (`\r\n`) → `\n`
- Mac (`\r`) → `\n`
- Unix (`\n`) → `\n`

### 7. Encoding Assumptions

**Limitation:** Assumes UTF-8 by default.

**Issues:** May misread files in other encodings

### 8. No Partial Line Reading

**Limitation:** Always reads complete lines.

**Cannot:** Read from character position within line

### 9. Performance with Network Files

**Limitation:** Slow with remote/network file systems.

**Impact:** Additional latency for network operations

### 10. No Write Capability

**Limitation:** Read-only tool, cannot modify files.

**By Design:** Prevents accidental modifications

---

**Related Documentation:**
- [copilot_findFiles](./04-copilot-findFiles.md)
- [copilot_findTextInFiles](./05-copilot-findTextInFiles.md)
- [copilot_listDirectory](./07-copilot-listDirectory.md)
- [copilot_searchCodebase](./01-copilot-searchCodebase.md)

**Last Updated:** 2025-01-22
