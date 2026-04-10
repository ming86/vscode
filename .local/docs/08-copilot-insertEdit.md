# GitHub Copilot Tool: insertEdit

## Overview

The `insertEdit` tool provides a sophisticated mechanism for inserting code or text at specific locations within a file by leveraging AI-powered code mapping. Unlike simple line-based insertion, this tool uses semantic understanding to locate the precise insertion point, making it robust against whitespace variations and minor code changes.

**Key Capabilities:**
- AI-powered location matching using code mappers
- Semantic understanding of code context
- Whitespace-insensitive matching
- Support for both before/after insertion modes
- Integration with LSP-based code intelligence

**Primary Use Cases:**
- Adding new methods to classes
- Inserting import statements
- Adding configuration entries
- Inserting code blocks at specific logical points

## Tool Declaration

```typescript
interface InsertEditTool {
	name: 'insertEdit';
	description: 'Insert text at a specific location in a file using semantic code mapping';
	parameters: {
		filePath: string;          // Absolute path to the file
		locationToInsertAfter: string;   // Code snippet to locate (insert AFTER this)
		codeToInsert: string;      // Text to insert
		explanation: string;       // Brief description of the operation
	}
}
```

## Input Parameters

### filePath
- **Type:** `string`
- **Required:** Yes
- **Format:** Absolute file path
- **Example:** `/Users/ming/project/src/util/helper.ts`
- **Validation:** File must exist, must be accessible

### locationToInsertAfter
- **Type:** `string`
- **Required:** Yes
- **Purpose:** Anchor text that defines where to insert
- **Matching:** Semantic, whitespace-flexible
- **Best Practice:** Include 2-5 lines of unique context
- **Example:**
```typescript
export class ConfigManager {
	private config: Config;
	// Insert new method after this line
}
```

### codeToInsert
- **Type:** `string`
- **Required:** Yes
- **Format:** Raw text with proper indentation
- **Indentation:** Should match surrounding code style
- **Newlines:** Include leading/trailing newlines as needed
- **Example:**
```typescript

	public getConfig(): Config {
		return this.config;
	}
```

### explanation
- **Type:** `string`
- **Required:** Yes
- **Purpose:** User-facing description
- **Best Practice:** Brief, action-oriented
- **Example:** "Add getConfig method to ConfigManager class"

## Architecture

### System Flow Diagram

```
┌─────────────────────────────────────────────────────────────────┐
│                        insertEdit Tool                           │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│                     Parameter Validation                         │
│  • Validate filePath exists and is accessible                   │
│  • Validate locationToInsertAfter is non-empty                  │
│  • Validate codeToInsert is non-empty                           │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│                    Read File Content                             │
│  • Load complete file into memory                               │
│  • Parse into lines with metadata                               │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│                  Code Mapper Selection                           │
│  • Detect file language from extension                          │
│  • Select appropriate mapper:                                   │
│    - TypeScript/JavaScript → Tree-sitter mapper                 │
│    - Python → Tree-sitter mapper                                │
│    - Generic text → Fuzzy matcher                               │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│                  Location Matching Phase                         │
│  1. Parse file with language-specific parser                    │
│  2. Build AST representation                                     │
│  3. Extract code blocks and their positions                      │
│  4. Match locationToInsertAfter using:                           │
│     • Exact string matching (if possible)                       │
│     • Whitespace-normalized matching                            │
│     • Semantic similarity scoring                               │
│     • Context-aware fuzzy matching                              │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│                  Insertion Point Calculation                     │
│  • Determine line number after matched location                 │
│  • Analyze indentation context                                  │
│  • Calculate proper indentation for new code                    │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│                    Code Insertion                                │
│  • Split file content at insertion point                        │
│  • Apply indentation to codeToInsert                            │
│  • Merge: before + codeToInsert + after                        │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│                    Write Modified File                           │
│  • Atomic write operation                                       │
│  • Preserve file permissions                                    │
│  • Maintain file encoding                                       │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│                    Success Response                              │
│  • Return success status                                        │
│  • Include line number where inserted                           │
│  • Log operation for telemetry                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Code Mapper Architecture

```
┌────────────────────────────────────────────────────────────────┐
│                        Code Mapper System                       │
└────────────────────────────────────────────────────────────────┘
                              ↓
              ┌───────────────┴───────────────┐
              ↓                               ↓
┌──────────────────────────┐   ┌──────────────────────────┐
│   Tree-sitter Mapper     │   │    Fuzzy Text Mapper     │
│  (Language-specific)     │   │   (Language-agnostic)    │
└──────────────────────────┘   └──────────────────────────┘
              ↓                               ↓
┌──────────────────────────┐   ┌──────────────────────────┐
│ • Parse to AST           │   │ • Line-by-line analysis  │
│ • Extract code blocks    │   │ • String similarity      │
│ • Semantic matching      │   │ • Levenshtein distance   │
│ • Context awareness      │   │ • N-gram matching        │
└──────────────────────────┘   └──────────────────────────┘
```

## Internal Implementation

### Algorithm: Location Matching

```typescript
function findInsertionPoint(
	fileContent: string,
	locationToInsertAfter: string,
	language: string
): number {
	const mapper = selectCodeMapper(language);

	// Step 1: Parse file content
	const ast = mapper.parse(fileContent);
	const blocks = mapper.extractBlocks(ast);

	// Step 2: Normalize search text
	const normalizedSearch = normalizeWhitespace(locationToInsertAfter);

	// Step 3: Try exact match first (performance optimization)
	const exactMatch = fileContent.indexOf(locationToInsertAfter);
	if (exactMatch !== -1) {
		return lineNumberFromOffset(fileContent, exactMatch + locationToInsertAfter.length);
	}

	// Step 4: Semantic matching with scoring
	let bestMatch = null;
	let bestScore = 0;

	for (const block of blocks) {
		const score = calculateSimilarity(
			normalizedSearch,
			normalizeWhitespace(block.text)
		);

		if (score > bestScore && score > SIMILARITY_THRESHOLD) {
			bestMatch = block;
			bestScore = score;
		}
	}

	if (!bestMatch) {
		throw new Error('Could not find matching location in file');
	}

	// Step 5: Calculate insertion point (after the matched block)
	return bestMatch.endLine;
}

function calculateSimilarity(text1: string, text2: string): number {
	// Token-based similarity (better for code than character-based)
	const tokens1 = tokenize(text1);
	const tokens2 = tokenize(text2);

	const intersection = tokens1.filter(t => tokens2.includes(t)).length;
	const union = new Set([...tokens1, ...tokens2]).size;

	return intersection / union; // Jaccard similarity
}
```

### Algorithm: Indentation Detection

```typescript
function detectAndApplyIndentation(
	fileContent: string,
	insertionLine: number,
	codeToInsert: string
): string {
	const lines = fileContent.split('\n');

	// Step 1: Analyze surrounding context
	const contextLines = lines.slice(
		Math.max(0, insertionLine - 3),
		Math.min(lines.length, insertionLine + 3)
	);

	// Step 2: Detect indentation style
	const indentStyle = detectIndentStyle(contextLines);
	// Returns: { type: 'tabs' | 'spaces', size: number }

	// Step 3: Determine base indentation level
	const baseIndent = detectIndentLevel(lines[insertionLine]);

	// Step 4: Apply indentation to new code
	const indentedCode = codeToInsert
		.split('\n')
		.map(line => {
			if (line.trim() === '') return line;
			return createIndent(baseIndent, indentStyle) + line;
		})
		.join('\n');

	return indentedCode;
}

function detectIndentStyle(lines: string[]): IndentStyle {
	let tabCount = 0;
	let spaceCount = 0;
	const spaceSizes = new Map<number, number>();

	for (const line of lines) {
		const match = line.match(/^(\s+)/);
		if (!match) continue;

		const indent = match[1];
		if (indent.includes('\t')) {
			tabCount++;
		} else {
			spaceCount++;
			spaceSizes.set(indent.length, (spaceSizes.get(indent.length) || 0) + 1);
		}
	}

	if (tabCount > spaceCount) {
		return { type: 'tabs', size: 1 };
	}

	// Find most common space count (likely 2 or 4)
	const mostCommon = Array.from(spaceSizes.entries())
		.sort((a, b) => b[1] - a[1])[0];

	return { type: 'spaces', size: mostCommon?.[0] || 4 };
}
```

## Dependencies

### Internal Services
- **IFileService**: File system operations
- **ILanguageService**: Language detection and parsing
- **ICodeMapperService**: Semantic code mapping
- **ITextModelService**: Text buffer management

### External Libraries
- **tree-sitter**: Language parsing for TypeScript, Python, etc.
- **tree-sitter-typescript**: TypeScript grammar
- **tree-sitter-python**: Python grammar
- **string-similarity**: Text similarity algorithms

### VS Code APIs
- `vscode.workspace.fs`: File system access
- `vscode.languages`: Language detection
- `vscode.TextDocument`: Document management

## Performance Characteristics

### Time Complexity
- **File Reading**: O(n) where n = file size
- **AST Parsing**: O(n) for most languages
- **Location Matching**: O(m × k) where:
  - m = number of code blocks in file
  - k = length of search text
- **Insertion**: O(n) for string concatenation
- **Total**: O(n) for typical cases

### Space Complexity
- **Memory Usage**: 2-3x file size during operation
  - Original content: 1x
  - AST representation: 1-2x
  - Modified content: 1x
- **Peak Usage**: ~10-50 MB for typical files (< 10k lines)

### Performance Benchmarks
```
File Size    Parse Time    Match Time    Total Time
─────────────────────────────────────────────────────
< 1 KB       < 1 ms        < 1 ms        5-10 ms
10 KB        2-5 ms        1-2 ms        10-20 ms
100 KB       20-50 ms      5-10 ms       50-100 ms
1 MB         200-500 ms    20-50 ms      300-600 ms
```

## Use Cases

### Use Case 1: Adding a Method to a Class

```typescript
// Tool invocation
{
	filePath: '/project/src/services/userService.ts',
	locationToInsertAfter: `export class UserService {
	constructor(private db: Database) {}

	async getUser(id: string): Promise<User> {
		return this.db.users.findById(id);
	}`,
	codeToInsert: `

	async updateUser(id: string, data: Partial<User>): Promise<User> {
		const user = await this.getUser(id);
		return this.db.users.update(id, { ...user, ...data });
	}`,
	explanation: 'Add updateUser method to UserService'
}

// Result: Method inserted after getUser with proper indentation
```

### Use Case 2: Adding Import Statements

```typescript
{
	filePath: '/project/src/components/Dashboard.tsx',
	locationToInsertAfter: `import React from 'react';
import { useAuth } from '../hooks/useAuth';`,
	codeToInsert: `
import { useSettings } from '../hooks/useSettings';`,
	explanation: 'Add useSettings import'
}
```

### Use Case 3: Inserting Configuration Entry

```typescript
{
	filePath: '/project/config/database.json',
	locationToInsertAfter: `{
	"host": "localhost",
	"port": 5432,`,
	codeToInsert: `
	"pool": {
		"min": 2,
		"max": 10
	},`,
	explanation: 'Add connection pool configuration'
}
```

### Use Case 4: Adding Test Case

```typescript
{
	filePath: '/project/test/userService.test.ts',
	locationToInsertAfter: `describe('UserService', () => {
	it('should get user by id', async () => {
		// test implementation
	});`,
	codeToInsert: `

	it('should update user data', async () => {
		const userId = '123';
		const updates = { name: 'John Doe' };
		const result = await service.updateUser(userId, updates);
		expect(result.name).toBe('John Doe');
	});`,
	explanation: 'Add test case for updateUser method'
}
```

## Comparison with Other Tools

### insertEdit vs replaceString

| Feature | insertEdit | replaceString |
|---------|-----------|---------------|
| **Primary Use** | Adding new content | Modifying existing content |
| **Matching** | Semantic code mapping | 4-strategy text matching |
| **Position** | After anchor point | Exact location replacement |
| **Indentation** | Auto-detected | User-provided |
| **Best For** | New methods, imports | Bug fixes, refactoring |

### insertEdit vs applyPatch

| Feature | insertEdit | applyPatch |
|---------|-----------|------------|
| **Operation Type** | Single insertion | Multiple changes |
| **Format** | Simple parameters | Unified diff format |
| **Context** | Minimal (anchor only) | Rich (surrounding lines) |
| **Complexity** | Low | High |
| **Use Case** | Quick additions | Complex refactorings |

### When to Use insertEdit

**✅ Use insertEdit when:**
- Adding a single new code block
- Location is easily identifiable
- Minimal context needed
- Quick, focused addition

**❌ Don't use insertEdit when:**
- Making multiple changes (use multiReplaceString or applyPatch)
- Replacing existing code (use replaceString)
- Complex refactoring needed (use applyPatch)
- Location is ambiguous (use replaceString with more context)

## Error Handling

### Error Scenarios and Responses

#### 1. File Not Found
```typescript
{
	error: 'FileNotFound',
	message: 'File does not exist: /path/to/file.ts',
	filePath: '/path/to/file.ts',
	suggestion: 'Verify the file path is correct or create the file first using createFile'
}
```

#### 2. Location Not Found
```typescript
{
	error: 'LocationNotFound',
	message: 'Could not find matching location in file',
	searchText: 'export class UserService...',
	suggestion: 'Provide more context or use a different anchor point',
	similarMatches: [
		{ text: '...', line: 42, similarity: 0.75 },
		{ text: '...', line: 156, similarity: 0.68 }
	]
}
```

#### 3. Ambiguous Location
```typescript
{
	error: 'AmbiguousLocation',
	message: 'Multiple locations match the search text',
	matches: [
		{ line: 42, context: '...' },
		{ line: 156, context: '...' }
	],
	suggestion: 'Provide more unique context to identify the exact location'
}
```

#### 4. Invalid File Encoding
```typescript
{
	error: 'EncodingError',
	message: 'File encoding is not supported: utf-16be',
	filePath: '/path/to/file.ts',
	detectedEncoding: 'utf-16be',
	supportedEncodings: ['utf-8', 'ascii', 'iso-8859-1']
}
```

#### 5. Permission Denied
```typescript
{
	error: 'PermissionDenied',
	message: 'Cannot write to file: permission denied',
	filePath: '/path/to/file.ts',
	suggestion: 'Check file permissions or run with appropriate privileges'
}
```

### Error Recovery Strategies

```typescript
async function insertEditWithRetry(params: InsertEditParams): Promise<void> {
	try {
		await insertEdit(params);
	} catch (error) {
		if (error.type === 'LocationNotFound' && error.similarMatches) {
			// Strategy 1: Try best similar match
			console.log('Trying similar match with 75% similarity...');
			await insertEditAtLine(params.filePath, error.similarMatches[0].line, params.codeToInsert);
		} else if (error.type === 'AmbiguousLocation') {
			// Strategy 2: Use first match with warning
			console.warn('Multiple matches found, using first occurrence');
			await insertEditAtLine(params.filePath, error.matches[0].line, params.codeToInsert);
		} else {
			throw error; // Unrecoverable
		}
	}
}
```

## Configuration

### VS Code Settings

```json
{
	"copilot.tools.insertEdit.enabled": true,
	"copilot.tools.insertEdit.codeMapper": "tree-sitter",
	"copilot.tools.insertEdit.similarityThreshold": 0.8,
	"copilot.tools.insertEdit.maxFileSize": 1048576,
	"copilot.tools.insertEdit.autoIndent": true,
	"copilot.tools.insertEdit.fallbackToFuzzy": true
}
```

### Configuration Options

| Setting | Type | Default | Description |
|---------|------|---------|-------------|
| `enabled` | boolean | true | Enable/disable the tool |
| `codeMapper` | string | "tree-sitter" | Code mapping strategy |
| `similarityThreshold` | number | 0.8 | Minimum match similarity (0-1) |
| `maxFileSize` | number | 1MB | Maximum file size to process |
| `autoIndent` | boolean | true | Automatically detect and apply indentation |
| `fallbackToFuzzy` | boolean | true | Fall back to fuzzy matching if semantic matching fails |

## Implementation Blueprint

### Step 1: Tool Registration

```typescript
// src/extension/tools/insertEditTool.ts
import { Tool } from '../api/Tool';
import { IFileService } from '../../platform/files/common/files';
import { ICodeMapperService } from '../../platform/codeMapper/codeMapper';

export class InsertEditTool implements Tool {
	static readonly ID = 'insertEdit';

	constructor(
		@IFileService private fileService: IFileService,
		@ICodeMapperService private codeMapper: ICodeMapperService
	) {}

	getDefinition(): ToolDefinition {
		return {
			name: 'insertEdit',
			description: 'Insert code at a specific location using semantic matching',
			parameters: {
				type: 'object',
				properties: {
					filePath: {
						type: 'string',
						description: 'Absolute path to the file'
					},
					locationToInsertAfter: {
						type: 'string',
						description: 'Code snippet to locate (insert after this)'
					},
					codeToInsert: {
						type: 'string',
						description: 'Text to insert'
					},
					explanation: {
						type: 'string',
						description: 'Brief description of the operation'
					}
				},
				required: ['filePath', 'locationToInsertAfter', 'codeToInsert', 'explanation']
			}
		};
	}

	async execute(params: InsertEditParams): Promise<ToolResult> {
		// Implementation
	}
}
```

### Step 2: Code Mapper Service

```typescript
// src/platform/codeMapper/codeMapperService.ts
export class CodeMapperService implements ICodeMapperService {
	private mappers = new Map<string, ICodeMapper>();

	constructor() {
		this.registerMapper('typescript', new TreeSitterTypeScriptMapper());
		this.registerMapper('javascript', new TreeSitterJavaScriptMapper());
		this.registerMapper('python', new TreeSitterPythonMapper());
		this.registerMapper('*', new FuzzyTextMapper()); // Fallback
	}

	async findLocation(
		fileContent: string,
		searchText: string,
		language: string
	): Promise<Location> {
		const mapper = this.mappers.get(language) || this.mappers.get('*')!;
		return mapper.findMatch(fileContent, searchText);
	}
}
```

### Step 3: Execution Logic

```typescript
async execute(params: InsertEditParams): Promise<ToolResult> {
	// 1. Validate parameters
	this.validateParams(params);

	// 2. Read file
	const content = await this.fileService.readFile(params.filePath);

	// 3. Find insertion point
	const language = detectLanguage(params.filePath);
	const location = await this.codeMapper.findLocation(
		content,
		params.locationToInsertAfter,
		language
	);

	// 4. Apply indentation
	const indentedCode = this.applyIndentation(
		content,
		location.line,
		params.codeToInsert
	);

	// 5. Insert code
	const lines = content.split('\n');
	lines.splice(location.line + 1, 0, indentedCode);
	const newContent = lines.join('\n');

	// 6. Write file
	await this.fileService.writeFile(params.filePath, newContent);

	return {
		success: true,
		insertedAtLine: location.line + 1,
		explanation: params.explanation
	};
}
```

## Best Practices

### 1. Provide Sufficient Context

**❌ Bad: Too little context**
```typescript
{
	locationToInsertAfter: 'export class UserService {'
}
```

**✅ Good: Unique, identifiable context**
```typescript
{
	locationToInsertAfter: `export class UserService {
	constructor(private db: Database) {}

	async getUser(id: string): Promise<User> {
		return this.db.users.findById(id);
	}`
}
```

### 2. Include Proper Indentation

**❌ Bad: No indentation**
```typescript
{
	codeToInsert: `
async updateUser(id: string, data: Partial<User>): Promise<User> {
return this.db.users.update(id, data);
}`
}
```

**✅ Good: Proper indentation**
```typescript
{
	codeToInsert: `

	async updateUser(id: string, data: Partial<User>): Promise<User> {
		return this.db.users.update(id, data);
	}`
}
```

### 3. Use Semantic Anchors

**❌ Bad: Generic comment**
```typescript
{
	locationToInsertAfter: '// Methods'
}
```

**✅ Good: Specific code block**
```typescript
{
	locationToInsertAfter: `private validateInput(data: unknown): boolean {
		return schema.validate(data);
	}`
}
```

### 4. Handle Edge Cases

```typescript
// Check if file exists before inserting
const fileExists = await fileService.exists(filePath);
if (!fileExists) {
	throw new Error('File does not exist. Create it first with createFile.');
}

// Verify the location exists before attempting insertion
try {
	await insertEdit(params);
} catch (error) {
	if (error.type === 'LocationNotFound') {
		// Provide helpful error message with suggestions
		console.error('Could not find location. Similar matches:', error.similarMatches);
	}
}
```

### 5. Batch Multiple Insertions

**❌ Bad: Multiple individual insertions**
```typescript
await insertEdit({ /* insert method 1 */ });
await insertEdit({ /* insert method 2 */ });
await insertEdit({ /* insert method 3 */ });
```

**✅ Good: Single operation or use multiReplaceString**
```typescript
// Option 1: Insert all at once
await insertEdit({
	locationToInsertAfter: '...',
	codeToInsert: `
	method1() { }

	method2() { }

	method3() { }`
});

// Option 2: Use multiReplaceString for complex scenarios
await multiReplaceString({ /* multiple changes */ });
```

## Limitations

### 1. Single Insertion Point
- Only inserts at one location per invocation
- For multiple insertions, must call tool multiple times or use multiReplaceString
- Each call reparses the file (performance impact)

### 2. Context Sensitivity
- Requires unique, identifiable anchor text
- Ambiguous locations will fail or match incorrectly
- Cannot handle extremely dynamic code that changes frequently

### 3. Language Support
- Full semantic support limited to languages with Tree-sitter grammars
- Falls back to fuzzy text matching for unsupported languages
- May have reduced accuracy for complex language features

### 4. File Size Constraints
- Default maximum file size: 1 MB
- Larger files have significant performance degradation
- Memory usage scales with file size

### 5. Indentation Detection
- Auto-indentation may fail with mixed tabs/spaces
- May not match project-specific style guides
- Manual verification recommended for critical code

### 6. Concurrent Modifications
- Not safe for concurrent edits to the same file
- No file locking mechanism
- Last write wins (potential data loss)

### 7. No Undo Support
- Tool does not maintain edit history
- Relies on VS Code's undo stack
- Manual backup recommended for critical changes

## Related Tools

- **replaceString**: For modifying existing code
- **multiReplaceString**: For multiple simultaneous changes
- **applyPatch**: For complex multi-file refactorings
- **createFile**: For creating new files before insertion

## Version History

- **v1.0**: Initial implementation with tree-sitter support
- **v1.1**: Added fuzzy fallback matcher
- **v1.2**: Improved indentation detection
- **v1.3**: Enhanced error messages with similar matches
- **v2.0**: Added support for Python and additional languages

## References

- Code Mapper Architecture: `src/platform/codeMapper/`
- Tree-sitter Integration: `src/platform/parser/`
- File Service: `src/platform/files/`
- Tool Framework: `src/extension/tools/`
