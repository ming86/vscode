# Copilot getErrors Tool Documentation

## Overview

The `getErrors` tool provides access to compile-time and lint errors from the TypeScript/JavaScript language service and other diagnostic providers in VS Code. This tool enables Copilot to diagnose code issues, suggest fixes, and understand the current error state of the workspace before making changes.

**Primary Use Cases:**
- Pre-validation before code modifications
- Error diagnosis and fix suggestions
- Post-edit validation
- Workspace health assessment
- Targeted error resolution

**Tool Category:** Diagnostic & Analysis

## Tool Declaration

```typescript
{
	name: 'getErrors',
	description: 'Get any compile or lint errors in a specific file or across all files. If the user mentions errors or problems in a file, they may be referring to these. Use the tool to see the same errors that the user is seeing. If the user asks you to analyze all errors, or does not specify a file, use this tool to gather errors for all files. Also use this tool after editing a file to validate the change.',
	inputSchema: {
		type: 'object',
		properties: {
			filePaths: {
				type: 'array',
				items: { type: 'string' },
				description: 'The absolute paths to the files or folders to check for errors. Omit \'filePaths\' when retrieving all errors.'
			}
		}
	}
}
```

## Input Parameters

### filePaths (optional)
- **Type:** `string[]`
- **Description:** Array of absolute file or folder paths to check for errors
- **Behavior:**
  - **When provided:** Returns errors only for specified files/folders
  - **When omitted:** Returns all errors across the entire workspace
  - **Folder support:** Can specify folders to get all errors within them
  - **Path format:** Must be absolute paths (e.g., `/Users/name/project/src/file.ts`)

### Examples

```typescript
// Get all workspace errors
{ }

// Get errors for specific file
{ filePaths: ['/Users/name/project/src/app.ts'] }

// Get errors for multiple files
{ filePaths: ['/Users/name/project/src/app.ts', '/Users/name/project/src/utils.ts'] }

// Get errors for entire folder
{ filePaths: ['/Users/name/project/src'] }
```

## Architecture

### High-Level Flow

```
User Request → getErrors Tool
    ↓
Diagnostic Collection
    ↓
Language Service Query (TypeScript/ESLint/etc.)
    ↓
Filter by File Paths (if specified)
    ↓
Severity Classification
    ↓
Format & Return Results
```

### Component Integration

```
┌─────────────────────────────────────────┐
│         getErrors Tool Handler          │
└──────────────┬──────────────────────────┘
               ↓
┌──────────────────────────────────────────┐
│    VS Code Diagnostic API                │
│    - vscode.languages.getDiagnostics()   │
└──────────────┬───────────────────────────┘
               ↓
┌──────────────────────────────────────────┐
│    Language Service Providers            │
│    - TypeScript Language Service         │
│    - ESLint Extension                    │
│    - Other Diagnostic Providers          │
└──────────────────────────────────────────┘
```

## Internal Implementation

### Core Implementation Location

**File:** `src/extension/tools/builtinTools.ts`

### Implementation Code

```typescript
async function getErrors(args: { filePaths?: string[] }, token: CancellationToken): Promise<string> {
	const diagnostics = vscode.languages.getDiagnostics();

	// Filter by specified paths if provided
	let filteredDiagnostics = diagnostics;
	if (args.filePaths && args.filePaths.length > 0) {
		const targetUris = new Set(args.filePaths.map(p => vscode.Uri.file(p).toString()));
		const targetFolders = args.filePaths
			.filter(p => !p.includes('.'))
			.map(p => vscode.Uri.file(p).toString());

		filteredDiagnostics = diagnostics.filter(([uri, _]) => {
			const uriStr = uri.toString();
			return targetUris.has(uriStr) ||
				   targetFolders.some(folder => uriStr.startsWith(folder));
		});
	}

	// Group by severity
	const errors: string[] = [];
	const warnings: string[] = [];
	const infos: string[] = [];

	for (const [uri, diagnosticList] of filteredDiagnostics) {
		for (const diagnostic of diagnosticList) {
			const message = formatDiagnostic(uri, diagnostic);

			switch (diagnostic.severity) {
				case vscode.DiagnosticSeverity.Error:
					errors.push(message);
					break;
				case vscode.DiagnosticSeverity.Warning:
					warnings.push(message);
					break;
				case vscode.DiagnosticSeverity.Information:
				case vscode.DiagnosticSeverity.Hint:
					infos.push(message);
					break;
			}
		}
	}

	// Format output
	return formatErrorReport(errors, warnings, infos);
}
```

### Diagnostic Formatting

```typescript
function formatDiagnostic(uri: vscode.Uri, diagnostic: vscode.Diagnostic): string {
	const filePath = uri.fsPath;
	const line = diagnostic.range.start.line + 1; // Convert to 1-based
	const column = diagnostic.range.start.character + 1;
	const source = diagnostic.source ? `[${diagnostic.source}]` : '';
	const code = diagnostic.code ? ` (${diagnostic.code})` : '';

	return `${filePath}:${line}:${column} ${source}${code} - ${diagnostic.message}`;
}

function formatErrorReport(errors: string[], warnings: string[], infos: string[]): string {
	const sections: string[] = [];

	if (errors.length > 0) {
		sections.push(`## Errors (${errors.length})\n${errors.join('\n')}`);
	}
	if (warnings.length > 0) {
		sections.push(`## Warnings (${warnings.length})\n${warnings.join('\n')}`);
	}
	if (infos.length > 0) {
		sections.push(`## Information (${infos.length})\n${infos.join('\n')}`);
	}

	if (sections.length === 0) {
		return 'No errors or warnings found.';
	}

	return sections.join('\n\n');
}
```

### Notebook Cell Error Handling

```typescript
// Special handling for notebook cells
function isNotebookCell(uri: vscode.Uri): boolean {
	return uri.scheme === 'vscode-notebook-cell';
}

function formatNotebookCellDiagnostic(uri: vscode.Uri, diagnostic: vscode.Diagnostic): string {
	// Extract notebook URI and cell index from cell URI
	const notebookInfo = parseNotebookCellUri(uri);
	return `${notebookInfo.notebookPath} [Cell ${notebookInfo.cellIndex}]:${diagnostic.range.start.line + 1} - ${diagnostic.message}`;
}
```

## Dependencies

### VS Code APIs

```typescript
import * as vscode from 'vscode';

// Primary API used
vscode.languages.getDiagnostics(): [vscode.Uri, vscode.Diagnostic[]][]

// Diagnostic interfaces
vscode.Diagnostic {
	range: vscode.Range;
	message: string;
	severity: vscode.DiagnosticSeverity;
	source?: string;
	code?: string | number;
	relatedInformation?: vscode.DiagnosticRelatedInformation[];
}

vscode.DiagnosticSeverity {
	Error = 0,
	Warning = 1,
	Information = 2,
	Hint = 3
}
```

### Diagnostic Providers

The tool aggregates diagnostics from multiple sources:
- **TypeScript/JavaScript:** Language service errors
- **ESLint:** Linting violations
- **Custom Extensions:** Any extension providing diagnostics
- **Built-in Validators:** JSON, YAML, Markdown linters

### Internal Dependencies

```typescript
import { ILanguageModelTool } from '../tools/toolsService';
import { CancellationToken } from '../../../platform/cancellation/common/cancellation';
```

## Performance Characteristics

### Response Time
- **All workspace errors:** 100-500ms (depends on project size)
- **Single file errors:** 10-50ms
- **Folder errors:** 50-200ms

### Scalability

```typescript
// Performance optimization for large workspaces
const MAX_DIAGNOSTICS_TO_RETURN = 1000;
const MAX_FILE_COUNT = 500;

function optimizeErrorCollection(diagnostics: [vscode.Uri, vscode.Diagnostic[]][]): string {
	let totalDiagnostics = 0;
	let fileCount = 0;
	const prioritized: string[] = [];

	// Prioritize errors over warnings
	for (const [uri, diags] of diagnostics) {
		const errors = diags.filter(d => d.severity === vscode.DiagnosticSeverity.Error);
		if (errors.length > 0) {
			prioritized.push(...errors.map(d => formatDiagnostic(uri, d)));
			totalDiagnostics += errors.length;
		}
		fileCount++;

		if (totalDiagnostics >= MAX_DIAGNOSTICS_TO_RETURN || fileCount >= MAX_FILE_COUNT) {
			break;
		}
	}

	return prioritized.join('\n');
}
```

### Memory Usage
- **Typical:** 1-5 MB for workspace diagnostics collection
- **Large projects:** Up to 20 MB for 10,000+ files

## Use Cases

### Use Case 1: Pre-Edit Validation

**Scenario:** User asks to refactor a function, Copilot checks for existing errors first.

**Implementation:**
```typescript
// Copilot's reasoning process
async function beforeRefactoring(filePath: string): Promise<void> {
	// Check current error state
	const errors = await getErrors({ filePaths: [filePath] });

	if (errors.includes('Errors (')) {
		// Inform user about existing errors
		console.log('Note: This file has existing errors that should be addressed first');
	}

	// Proceed with refactoring
	await performRefactoring(filePath);
}
```

**Example Dialog:**
```
User: Refactor the calculateTotal function in utils.ts

Copilot: [calls getErrors for utils.ts]
Result: "Errors (2)
utils.ts:45:12 [ts] - Cannot find name 'totla'. Did you mean 'total'?
utils.ts:67:8 [ts] - Type 'string' is not assignable to type 'number'."

Copilot: I notice there are 2 existing TypeScript errors in utils.ts.
Would you like me to fix these first before refactoring, or proceed with
the refactoring and address all issues together?
```

### Use Case 2: Post-Edit Validation

**Scenario:** After making changes, verify no new errors were introduced.

**Implementation:**
```typescript
async function validateAfterEdit(filePath: string, beforeErrors: string): Promise<void> {
	// Wait for language service to update
	await new Promise(resolve => setTimeout(resolve, 500));

	// Get current errors
	const afterErrors = await getErrors({ filePaths: [filePath] });

	// Compare error states
	if (afterErrors !== beforeErrors) {
		if (afterErrors.includes('No errors')) {
			console.log('✓ All errors resolved successfully!');
		} else {
			console.log('⚠ New errors detected. Analyzing...');
			// Suggest fixes
		}
	}
}
```

**Example Dialog:**
```
User: Fix the type error in the login function

Copilot: [calls getErrors before editing]
Before: "Errors (1): auth.ts:23:5 - Type 'null' is not assignable to type 'User'"

[makes edit]

Copilot: [calls getErrors after editing]
After: "No errors or warnings found."

Copilot: ✓ Fixed the type error by updating the return type to 'User | null'
```

### Use Case 3: Workspace Error Analysis

**Scenario:** User asks to fix all TypeScript errors in the project.

**Implementation:**
```typescript
async function analyzeWorkspaceErrors(): Promise<ErrorSummary> {
	// Get all errors
	const allErrors = await getErrors({});

	// Parse and categorize
	const errorsByType = categorizeErrors(allErrors);
	const errorsByFile = groupByFile(allErrors);

	// Prioritize by severity and frequency
	return {
		totalErrors: errorsByType.errors.length,
		totalWarnings: errorsByType.warnings.length,
		mostProblematicFiles: errorsByFile.slice(0, 10),
		commonIssues: identifyPatterns(errorsByType.errors)
	};
}
```

**Example Dialog:**
```
User: What errors do we have in the project?

Copilot: [calls getErrors with no filePaths]

Result shows 47 errors across 12 files. Let me summarize:

**Most Critical Issues:**
1. src/api/users.ts (12 errors)
   - Missing type definitions for API responses
   - Null safety issues

2. src/utils/validation.ts (8 errors)
   - Unreachable code after return statements
   - Unused variables

**Common Patterns:**
- 23 errors related to missing null checks
- 15 errors from implicit 'any' types
- 9 errors from unused imports

Would you like me to fix these systematically, starting with the most
critical files?
```

### Use Case 4: Targeted Error Resolution

**Scenario:** Fix specific error types across multiple files.

**Implementation:**
```typescript
async function fixErrorPattern(pattern: string): Promise<void> {
	const allErrors = await getErrors({});
	const matchingErrors = allErrors
		.split('\n')
		.filter(line => line.includes(pattern));

	// Extract file paths
	const affectedFiles = new Set(
		matchingErrors.map(err => err.split(':')[0])
	);

	// Fix each file
	for (const file of affectedFiles) {
		await fixErrorsInFile(file, pattern);
	}
}
```

**Example Dialog:**
```
User: Fix all "implicit any" errors

Copilot: [calls getErrors]
Found 15 implicit any errors across 5 files. Fixing each one...

[Processes each file with getErrors to verify fixes]

✓ Fixed all 15 errors. All files now have explicit type annotations.
```

### Use Case 5: Error-Driven Code Suggestions

**Scenario:** Use error messages to guide code generation.

**Implementation:**
```typescript
async function suggestFixBasedOnErrors(filePath: string): Promise<string> {
	const errors = await getErrors({ filePaths: [filePath] });

	// Parse error types
	const errorTypes = parseErrorTypes(errors);

	// Generate contextual suggestions
	if (errorTypes.includes('missing-import')) {
		return 'Add missing imports from...';
	}
	if (errorTypes.includes('type-mismatch')) {
		return 'Convert types using...';
	}

	return 'General error fix suggestions...';
}
```

## Comparison with Other Tools

### getErrors vs. read_file

| Aspect | getErrors | read_file |
|--------|-----------|-----------|
| **Purpose** | Get diagnostic information | Read file content |
| **Output** | Structured error list | Raw file text |
| **Performance** | Fast (uses cached diagnostics) | Depends on file size |
| **Best For** | Error analysis | Code inspection |

**When to use getErrors:**
- Before making changes to check error state
- After edits to validate success
- When user mentions "errors" or "problems"

**When to use read_file:**
- Need to see actual code content
- Planning refactoring
- Understanding implementation

### getErrors vs. semantic_search

| Aspect | getErrors | semantic_search |
|--------|-----------|-----------------|
| **Data Source** | Language service diagnostics | Code content |
| **Query Type** | Implicit (all errors) | Explicit query |
| **Response Time** | 10-500ms | 100-2000ms |
| **Best For** | Finding errors | Finding code patterns |

### getErrors vs. grep_search

| Aspect | getErrors | grep_search |
|--------|-----------|-------------|
| **Purpose** | Get language service errors | Search file contents |
| **Accuracy** | High (semantic understanding) | Exact text matching |
| **Context** | Full diagnostic context | Matching lines only |
| **Best For** | TypeScript/JS errors | Finding text patterns |

## Error Handling

### Common Error Scenarios

```typescript
try {
	const errors = await getErrors({ filePaths: ['/invalid/path.ts'] });
} catch (error) {
	// Handle various error conditions
}
```

### Error Cases

1. **Invalid File Path**
   - **Behavior:** Silently ignores invalid paths, returns empty result
   - **Detection:** No errors returned for non-existent files

2. **Language Service Not Ready**
   - **Symptom:** Empty results immediately after file creation
   - **Solution:** Retry after 500ms delay

3. **Too Many Errors**
   - **Symptom:** Truncated results or timeout
   - **Solution:** Use filePaths to narrow scope

4. **Notebook Cell Errors**
   - **Behavior:** Different URI scheme handling
   - **Detection:** Check for `vscode-notebook-cell` scheme

### Retry Logic

```typescript
async function getErrorsWithRetry(
	args: { filePaths?: string[] },
	maxRetries = 2
): Promise<string> {
	for (let attempt = 0; attempt <= maxRetries; attempt++) {
		try {
			const result = await getErrors(args);

			// If language service is still initializing
			if (result.includes('Language service initializing')) {
				if (attempt < maxRetries) {
					await new Promise(resolve => setTimeout(resolve, 1000));
					continue;
				}
			}

			return result;
		} catch (error) {
			if (attempt === maxRetries) throw error;
			await new Promise(resolve => setTimeout(resolve, 500));
		}
	}
}
```

## Configuration

### User Settings (No Direct Configuration)

The `getErrors` tool respects VS Code's diagnostic settings:

```json
{
	// TypeScript diagnostics
	"typescript.validate.enable": true,
	"typescript.reportStyleChecksAsWarnings": true,

	// JavaScript diagnostics
	"javascript.validate.enable": true,

	// Problem matcher settings
	"problems.decorations.enabled": true,
	"problems.showCurrentInStatus": true
}
```

### Extension Settings Impact

```json
{
	// ESLint integration
	"eslint.enable": true,
	"eslint.run": "onType",

	// Other linters
	"python.linting.enabled": true,
	"python.linting.pylintEnabled": true
}
```

### Diagnostic Provider Configuration

The tool aggregates from all active diagnostic providers. To control what's included:

1. **Enable/disable specific linters** in their extension settings
2. **Configure language services** via tsconfig.json, jsconfig.json
3. **Set severity levels** in linter configuration files

## Implementation Blueprint

### Step 1: Basic Error Retrieval

```typescript
// Simple implementation
async function getErrors(filePaths?: string[]): Promise<vscode.Diagnostic[]> {
	const allDiagnostics = vscode.languages.getDiagnostics();

	if (!filePaths || filePaths.length === 0) {
		return allDiagnostics.flatMap(([_, diags]) => diags);
	}

	const targetUris = new Set(filePaths.map(p => vscode.Uri.file(p).toString()));
	return allDiagnostics
		.filter(([uri, _]) => targetUris.has(uri.toString()))
		.flatMap(([_, diags]) => diags);
}
```

### Step 2: Add Severity Filtering

```typescript
interface ErrorReport {
	errors: FormattedDiagnostic[];
	warnings: FormattedDiagnostic[];
	infos: FormattedDiagnostic[];
}

function groupBySeverity(diagnostics: vscode.Diagnostic[]): ErrorReport {
	const report: ErrorReport = { errors: [], warnings: [], infos: [] };

	for (const diag of diagnostics) {
		const formatted = formatDiagnostic(diag);

		switch (diag.severity) {
			case vscode.DiagnosticSeverity.Error:
				report.errors.push(formatted);
				break;
			case vscode.DiagnosticSeverity.Warning:
				report.warnings.push(formatted);
				break;
			default:
				report.infos.push(formatted);
		}
	}

	return report;
}
```

### Step 3: Add Folder Support

```typescript
function matchesPath(
	diagnosticUri: vscode.Uri,
	targetPaths: string[]
): boolean {
	const uriStr = diagnosticUri.toString();

	for (const path of targetPaths) {
		const pathUri = vscode.Uri.file(path);

		// Exact file match
		if (uriStr === pathUri.toString()) {
			return true;
		}

		// Folder match (check if file is within folder)
		if (!path.includes('.')) {
			const folderUri = pathUri.toString() + '/';
			if (uriStr.startsWith(folderUri)) {
				return true;
			}
		}
	}

	return false;
}
```

### Step 4: Format for LLM Consumption

```typescript
function formatForLLM(report: ErrorReport): string {
	const sections: string[] = [];

	if (report.errors.length > 0) {
		sections.push(
			`# Errors (${report.errors.length})`,
			...report.errors.map(e => `- ${e.file}:${e.line}:${e.column} - ${e.message}`)
		);
	}

	if (report.warnings.length > 0) {
		sections.push(
			`\n# Warnings (${report.warnings.length})`,
			...report.warnings.map(w => `- ${w.file}:${w.line}:${w.column} - ${w.message}`)
		);
	}

	if (sections.length === 0) {
		return 'No errors or warnings found.';
	}

	return sections.join('\n');
}
```

## Best Practices

### 1. Always Check Before Major Edits

```typescript
// ✓ Good: Check error state before refactoring
const errorsBefore = await getErrors({ filePaths: [targetFile] });
await performRefactoring(targetFile);
const errorsAfter = await getErrors({ filePaths: [targetFile] });
compareErrorStates(errorsBefore, errorsAfter);

// ✗ Bad: Make changes without checking
await performRefactoring(targetFile);
```

### 2. Scope Appropriately

```typescript
// ✓ Good: Request only what you need
const errors = await getErrors({
	filePaths: ['/project/src/specific-file.ts']
});

// ✗ Bad: Always request all errors when you only need one file
const errors = await getErrors({}); // Returns entire workspace
```

### 3. Wait for Language Service

```typescript
// ✓ Good: Wait after file edits
await editFile(filePath);
await new Promise(resolve => setTimeout(resolve, 500));
const errors = await getErrors({ filePaths: [filePath] });

// ✗ Bad: Check immediately
await editFile(filePath);
const errors = await getErrors({ filePaths: [filePath] }); // May be stale
```

### 4. Handle Large Error Sets

```typescript
// ✓ Good: Process errors in batches
const errors = await getErrors({});
const highPriority = errors.filter(e => e.severity === 'Error');
const topFiles = prioritizeByFrequency(highPriority).slice(0, 10);

// ✗ Bad: Try to fix all errors at once
const errors = await getErrors({});
await fixAllErrors(errors); // May be overwhelming
```

### 5. Provide Context to User

```typescript
// ✓ Good: Explain what you found
const errors = await getErrors({ filePaths: [file] });
if (errors.length > 0) {
	console.log(`Found ${errors.length} errors in ${file}:`);
	console.log(errors);
	console.log('Would you like me to fix these?');
}

// ✗ Bad: Just dump error output
console.log(await getErrors({ filePaths: [file] }));
```

### 6. Combine with Other Tools

```typescript
// ✓ Good: Use getErrors + read_file for context
const errors = await getErrors({ filePaths: [file] });
const content = await readFile(file);
const fixSuggestion = analyzeWithContext(errors, content);

// ✗ Bad: Suggest fixes based only on error messages
const errors = await getErrors({ filePaths: [file] });
suggestFix(errors); // May miss important context
```

## Limitations

### 1. Language Service Dependency

**Limitation:** Only returns errors that language services can detect.

```typescript
// Will detect:
const x: number = 'string'; // ✓ Type error

// Won't detect:
const user = await fetchUser(); // Runtime: user is null
user.name; // ✗ Null reference error (only at runtime)
```

**Workaround:** Combine with code analysis and user-reported issues.

### 2. Timing Sensitivity

**Limitation:** Diagnostics may be stale immediately after file changes.

```typescript
// Problematic sequence:
await writeFile(path, newContent);
const errors = await getErrors({ filePaths: [path] }); // May show old errors
```

**Workaround:** Add delay or use file watcher events.

### 3. Performance with Large Workspaces

**Limitation:** Retrieving all workspace errors can be slow (>1s) in large projects.

```typescript
// Slow in workspace with 10,000+ files:
const allErrors = await getErrors({}); // May take 2-5 seconds
```

**Workaround:** Always specify filePaths when possible.

### 4. No Error Severity Filtering

**Limitation:** Cannot request only errors or only warnings in the API.

```typescript
// Can't do this directly:
const onlyErrors = await getErrors({ severity: 'Error' });

// Must filter after retrieval:
const all = await getErrors({});
const onlyErrors = all.filter(e => e.severity === 'Error');
```

**Workaround:** Parse and filter results after retrieval.

### 5. Limited Error Context

**Limitation:** Diagnostic messages may lack full context for complex issues.

```typescript
// Error message: "Type 'string' is not assignable to type 'number'"
// Missing context: Why is this type expected? What's the source?
```

**Workaround:** Use read_file to get surrounding code context.

### 6. No Historical Tracking

**Limitation:** Cannot see error history or trends.

```typescript
// Can't answer: "Were there errors in this file yesterday?"
// Can only answer: "Are there errors in this file now?"
```

**Workaround:** Store error snapshots externally if tracking is needed.

### 7. Notebook Cell Complexity

**Limitation:** Notebook cell errors use special URI scheme that may be harder to interpret.

```typescript
// Regular file: /Users/name/project/file.ts
// Notebook cell: vscode-notebook-cell:/Users/name/notebook.ipynb#cell_3
```

**Workaround:** Special parsing logic for notebook URIs.

## Summary

The `getErrors` tool is essential for diagnostic workflows in Copilot. It provides real-time access to all language service diagnostics, enabling:

- **Proactive error detection** before making changes
- **Post-edit validation** to ensure no new errors introduced
- **Workspace health analysis** across entire projects
- **Targeted error resolution** for specific files or patterns

**Key Strengths:**
- Fast access to cached diagnostics
- Supports all language service providers
- Flexible scoping (file, folder, workspace)
- Rich diagnostic information with severity levels

**Best Used With:**
- Pre-edit error checking
- Post-edit validation
- Combined with read_file for context
- Iterative error resolution workflows

By following the best practices and understanding its limitations, the `getErrors` tool becomes a powerful diagnostic aid for AI-assisted coding workflows.
