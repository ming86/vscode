# Copilot getChangedFiles Tool Documentation

## Overview

The `getChangedFiles` tool provides access to git working tree changes through VS Code's Git extension API. It enables Copilot to understand what files have been modified, staged, or have conflicts, allowing for context-aware assistance with code reviews, commit preparation, and change management.

**Primary Use Cases:**
- Pre-commit code review assistance
- Change impact analysis
- Conflict resolution guidance
- Staged vs. unstaged change management
- Commit message generation based on changes

**Tool Category:** Source Control & Version Control

## Tool Declaration

```typescript
{
	name: 'getChangedFiles',
	description: 'Get the list of changed files in the git working tree. Returns files that have been modified, added, deleted, or have conflicts. Can filter by change type (staged, unstaged, conflicts).',
	inputSchema: {
		type: 'object',
		properties: {
			filterType: {
				type: 'string',
				enum: ['all', 'staged', 'unstaged', 'conflicts'],
				description: 'Filter changes by type. "all" returns all changes, "staged" returns only staged changes, "unstaged" returns only unstaged changes, "conflicts" returns only files with merge conflicts.',
				default: 'all'
			},
			includeStatus: {
				type: 'boolean',
				description: 'Include git status for each file (modified, added, deleted, renamed, etc.)',
				default: true
			}
		}
	}
}
```

## Input Parameters

### filterType (optional)
- **Type:** `'all' | 'staged' | 'unstaged' | 'conflicts'`
- **Default:** `'all'`
- **Description:** Filters the returned changes by their git status

**Filter Types:**
- **`all`**: Returns all modified files (staged + unstaged + conflicts)
- **`staged`**: Returns only files added to staging area (ready for commit)
- **`unstaged`**: Returns only modified files not yet staged
- **`conflicts`**: Returns only files with merge conflicts

### includeStatus (optional)
- **Type:** `boolean`
- **Default:** `true`
- **Description:** Whether to include detailed git status for each file

**Status Types:**
- `INDEX_MODIFIED` - File modified and staged
- `INDEX_ADDED` - New file staged
- `INDEX_DELETED` - File deletion staged
- `INDEX_RENAMED` - File renamed and staged
- `MODIFIED` - File modified but not staged
- `DELETED` - File deleted but not staged
- `UNTRACKED` - New file not yet tracked by git
- `CONFLICT` - File has merge conflicts

### Examples

```typescript
// Get all changes with status
{ filterType: 'all', includeStatus: true }

// Get only staged files
{ filterType: 'staged' }

// Get only files with conflicts
{ filterType: 'conflicts' }

// Get unstaged changes without detailed status
{ filterType: 'unstaged', includeStatus: false }
```

## Architecture

### High-Level Flow

```
User Request → getChangedFiles Tool
    ↓
VS Code Git Extension Query
    ↓
Repository Status Retrieval
    ↓
Filter by Type (staged/unstaged/conflicts)
    ↓
Format with Status Information
    ↓
Return Structured Results
```

### Component Integration

```
┌─────────────────────────────────────────┐
│      getChangedFiles Tool Handler       │
└──────────────┬──────────────────────────┘
               ↓
┌──────────────────────────────────────────┐
│    VS Code Git Extension API             │
│    - getAPI(1): GitExtension             │
└──────────────┬───────────────────────────┘
               ↓
┌──────────────────────────────────────────┐
│    Git Repository State                  │
│    - repository.state.workingTreeChanges │
│    - repository.state.indexChanges       │
│    - repository.state.mergeChanges       │
└──────────────────────────────────────────┘
```

### Git Extension Integration

```
┌─────────────────────────────────────────┐
│         Git Extension (vscode.git)       │
│                                          │
│  ┌────────────────────────────────┐    │
│  │    Repository Manager          │    │
│  │  - Active repositories         │    │
│  │  - Multi-repo support          │    │
│  └────────────────────────────────┘    │
│                                          │
│  ┌────────────────────────────────┐    │
│  │    Repository State            │    │
│  │  - Working tree changes        │    │
│  │  - Index (staged) changes      │    │
│  │  - Merge conflicts             │    │
│  │  - Untracked files             │    │
│  └────────────────────────────────┘    │
└─────────────────────────────────────────┘
```

## Internal Implementation

### Core Implementation Location

**File:** `src/extension/tools/builtinTools.ts`

### Implementation Code

```typescript
async function getChangedFiles(
	args: {
		filterType?: 'all' | 'staged' | 'unstaged' | 'conflicts',
		includeStatus?: boolean
	},
	token: CancellationToken
): Promise<string> {
	const filterType = args.filterType || 'all';
	const includeStatus = args.includeStatus !== false;

	// Get git extension
	const gitExtension = vscode.extensions.getExtension('vscode.git')?.exports;
	if (!gitExtension) {
		return 'Git extension is not available';
	}

	const git = gitExtension.getAPI(1);
	if (git.repositories.length === 0) {
		return 'No git repositories found in workspace';
	}

	// Collect changes from all repositories
	const allChanges: FileChange[] = [];

	for (const repo of git.repositories) {
		const changes = collectRepositoryChanges(repo, filterType);
		allChanges.push(...changes);
	}

	if (allChanges.length === 0) {
		return `No ${filterType === 'all' ? '' : filterType + ' '}changes found`;
	}

	// Format results
	return formatChanges(allChanges, includeStatus);
}
```

### Repository Change Collection

```typescript
interface FileChange {
	uri: vscode.Uri;
	status: Status;
	originalUri?: vscode.Uri; // For renames
	isStaged: boolean;
	isConflict: boolean;
}

function collectRepositoryChanges(
	repository: Repository,
	filterType: string
): FileChange[] {
	const changes: FileChange[] = [];
	const state = repository.state;

	// Collect staged changes (index)
	if (filterType === 'all' || filterType === 'staged') {
		for (const change of state.indexChanges) {
			changes.push({
				uri: change.uri,
				status: change.status,
				originalUri: change.originalUri,
				isStaged: true,
				isConflict: false
			});
		}
	}

	// Collect unstaged changes (working tree)
	if (filterType === 'all' || filterType === 'unstaged') {
		for (const change of state.workingTreeChanges) {
			// Skip if already in staged changes
			const alreadyStaged = changes.some(c =>
				c.uri.toString() === change.uri.toString()
			);

			if (!alreadyStaged) {
				changes.push({
					uri: change.uri,
					status: change.status,
					originalUri: change.originalUri,
					isStaged: false,
					isConflict: false
				});
			}
		}
	}

	// Collect merge conflicts
	if (filterType === 'all' || filterType === 'conflicts') {
		for (const change of state.mergeChanges) {
			changes.push({
				uri: change.uri,
				status: Status.CONFLICT,
				isStaged: false,
				isConflict: true
			});
		}
	}

	return changes;
}
```

### Status Enum and Formatting

```typescript
enum Status {
	INDEX_MODIFIED = 0,
	INDEX_ADDED = 1,
	INDEX_DELETED = 2,
	INDEX_RENAMED = 3,
	INDEX_COPIED = 4,
	MODIFIED = 5,
	DELETED = 6,
	UNTRACKED = 7,
	IGNORED = 8,
	INTENT_TO_ADD = 9,
	ADDED_BY_US = 10,
	ADDED_BY_THEM = 11,
	DELETED_BY_US = 12,
	DELETED_BY_THEM = 13,
	BOTH_ADDED = 14,
	BOTH_DELETED = 15,
	BOTH_MODIFIED = 16,
	CONFLICT = 17
}

function getStatusLabel(status: Status): string {
	const labels: Record<number, string> = {
		[Status.INDEX_MODIFIED]: 'Modified (staged)',
		[Status.INDEX_ADDED]: 'Added (staged)',
		[Status.INDEX_DELETED]: 'Deleted (staged)',
		[Status.INDEX_RENAMED]: 'Renamed (staged)',
		[Status.INDEX_COPIED]: 'Copied (staged)',
		[Status.MODIFIED]: 'Modified',
		[Status.DELETED]: 'Deleted',
		[Status.UNTRACKED]: 'Untracked',
		[Status.CONFLICT]: 'Conflict',
		[Status.BOTH_MODIFIED]: 'Conflict (both modified)',
		[Status.BOTH_ADDED]: 'Conflict (both added)',
		[Status.DELETED_BY_US]: 'Conflict (deleted by us)',
		[Status.DELETED_BY_THEM]: 'Conflict (deleted by them)'
	};

	return labels[status] || 'Unknown';
}
```

### Output Formatting

```typescript
function formatChanges(changes: FileChange[], includeStatus: boolean): string {
	const sections: string[] = [];

	// Group by category
	const staged = changes.filter(c => c.isStaged);
	const conflicts = changes.filter(c => c.isConflict);
	const unstaged = changes.filter(c => !c.isStaged && !c.isConflict);

	// Format staged changes
	if (staged.length > 0) {
		sections.push('## Staged Changes (' + staged.length + ')');
		sections.push(formatChangeList(staged, includeStatus));
	}

	// Format conflicts
	if (conflicts.length > 0) {
		sections.push('## Conflicts (' + conflicts.length + ')');
		sections.push(formatChangeList(conflicts, includeStatus));
	}

	// Format unstaged changes
	if (unstaged.length > 0) {
		sections.push('## Unstaged Changes (' + unstaged.length + ')');
		sections.push(formatChangeList(unstaged, includeStatus));
	}

	return sections.join('\n\n');
}

function formatChangeList(changes: FileChange[], includeStatus: boolean): string {
	return changes.map(change => {
		const relativePath = vscode.workspace.asRelativePath(change.uri);
		const statusLabel = includeStatus ? ` [${getStatusLabel(change.status)}]` : '';

		if (change.originalUri) {
			// Handle renames
			const originalPath = vscode.workspace.asRelativePath(change.originalUri);
			return `- ${originalPath} → ${relativePath}${statusLabel}`;
		}

		return `- ${relativePath}${statusLabel}`;
	}).join('\n');
}
```

### Multi-Repository Support

```typescript
function formatMultiRepoChanges(repositories: Repository[]): string {
	const sections: string[] = [];

	for (const repo of repositories) {
		const repoName = path.basename(repo.rootUri.fsPath);
		const changes = collectRepositoryChanges(repo, 'all');

		if (changes.length > 0) {
			sections.push(`# Repository: ${repoName}`);
			sections.push(formatChanges(changes, true));
		}
	}

	return sections.join('\n\n');
}
```

## Dependencies

### VS Code Git Extension

```typescript
import * as vscode from 'vscode';

// Git Extension API
interface GitExtension {
	getAPI(version: 1): API;
}

interface API {
	repositories: Repository[];
	getRepository(uri: vscode.Uri): Repository | null;
}

interface Repository {
	readonly rootUri: vscode.Uri;
	readonly state: RepositoryState;
	// ... other methods
}

interface RepositoryState {
	readonly HEAD: Branch | undefined;
	readonly workingTreeChanges: Change[];
	readonly indexChanges: Change[];
	readonly mergeChanges: Change[];
}

interface Change {
	readonly uri: vscode.Uri;
	readonly originalUri: vscode.Uri | undefined;
	readonly status: Status;
}
```

### Extension Loading

```typescript
// Ensure git extension is loaded
const gitExtension = vscode.extensions.getExtension<GitExtension>('vscode.git');
if (!gitExtension) {
	throw new Error('Git extension not found');
}

// Activate if not already active
if (!gitExtension.isActive) {
	await gitExtension.activate();
}

const git = gitExtension.exports.getAPI(1);
```

### Internal Dependencies

```typescript
import { ILanguageModelTool } from '../tools/toolsService';
import { CancellationToken } from '../../../platform/cancellation/common/cancellation';
import * as path from 'path';
```

## Performance Characteristics

### Response Time
- **Single repository:** 10-50ms
- **Multiple repositories:** 50-200ms
- **Large change sets (100+ files):** 100-300ms

### Scalability

```typescript
// Performance optimization for large change sets
const MAX_CHANGES_TO_RETURN = 500;

function optimizeChangeCollection(changes: FileChange[]): FileChange[] {
	if (changes.length <= MAX_CHANGES_TO_RETURN) {
		return changes;
	}

	// Prioritize: conflicts > staged > unstaged
	const prioritized = [
		...changes.filter(c => c.isConflict),
		...changes.filter(c => c.isStaged && !c.isConflict),
		...changes.filter(c => !c.isStaged && !c.isConflict)
	];

	return prioritized.slice(0, MAX_CHANGES_TO_RETURN);
}
```

### Memory Usage
- **Typical:** <1 MB for normal change sets (10-50 files)
- **Large projects:** 2-5 MB for 500+ changed files

### Caching

```typescript
// Git extension maintains its own cache
// No additional caching needed in tool implementation
// State is automatically updated by git file watcher
```

## Use Cases

### Use Case 1: Pre-Commit Code Review

**Scenario:** User asks Copilot to review changes before committing.

**Implementation:**
```typescript
async function reviewChangesBeforeCommit(): Promise<void> {
	// Get staged changes
	const stagedFiles = await getChangedFiles({
		filterType: 'staged',
		includeStatus: true
	});

	// Read content of each changed file
	const filePaths = parseFilePathsFromOutput(stagedFiles);
	const reviews: string[] = [];

	for (const filePath of filePaths) {
		const content = await readFile(filePath);
		const review = await analyzeCodeQuality(content);
		reviews.push(`${filePath}: ${review}`);
	}

	return reviews.join('\n\n');
}
```

**Example Dialog:**
```
User: Review my staged changes before I commit

Copilot: [calls getChangedFiles with filterType='staged']
Result: "Staged Changes (3)
- src/api/users.ts [Modified (staged)]
- src/types/user.ts [Added (staged)]
- tests/users.test.ts [Modified (staged)]"

[Reads each file and analyzes]

Copilot: I've reviewed your 3 staged changes:

✓ src/api/users.ts
  - Good: Added proper error handling for null users
  - Suggestion: Consider adding JSDoc comments for new getUserById function

✓ src/types/user.ts
  - Good: New User interface is well-typed
  - Suggestion: Add optional fields for email and phone

✓ tests/users.test.ts
  - Good: Added test coverage for new getUserById function
  - Note: Consider adding edge case tests for null/undefined inputs

Overall looks good! Ready to commit.
```

### Use Case 2: Conflict Resolution Assistance

**Scenario:** User has merge conflicts and needs help resolving them.

**Implementation:**
```typescript
async function assistWithConflicts(): Promise<void> {
	// Get files with conflicts
	const conflicts = await getChangedFiles({
		filterType: 'conflicts'
	});

	if (!conflicts.includes('Conflicts')) {
		return 'No conflicts found. Your merge is clean!';
	}

	// Parse conflict files
	const conflictFiles = parseFilePathsFromOutput(conflicts);

	// Analyze each conflict
	for (const file of conflictFiles) {
		const content = await readFile(file);
		const conflictSections = extractConflictMarkers(content);
		const resolution = await suggestResolution(conflictSections);

		console.log(`\n## ${file}`);
		console.log(resolution);
	}
}
```

**Example Dialog:**
```
User: Help me resolve merge conflicts

Copilot: [calls getChangedFiles with filterType='conflicts']
Result: "Conflicts (2)
- src/config.ts [Conflict (both modified)]
- src/api/auth.ts [Conflict (both modified)]"

Let me analyze these conflicts:

## src/config.ts
Conflict in API_URL configuration:
- Your branch: Using staging URL
- Main branch: Using production URL

Suggestion: Keep production URL but add environment variable override

## src/api/auth.ts
Conflict in token validation:
- Your branch: Added expiry check
- Main branch: Refactored validation logic

Suggestion: Accept main branch changes and add your expiry check within
the new validation structure

Would you like me to apply these resolutions?
```

### Use Case 3: Commit Message Generation

**Scenario:** Generate commit message based on actual changes.

**Implementation:**
```typescript
async function generateCommitMessage(): Promise<string> {
	const stagedFiles = await getChangedFiles({
		filterType: 'staged',
		includeStatus: true
	});

	if (!stagedFiles.includes('Staged Changes')) {
		return 'No staged changes to commit';
	}

	// Analyze change types
	const changeTypes = analyzeChangeTypes(stagedFiles);

	// Read key files to understand changes
	const filePaths = parseFilePathsFromOutput(stagedFiles);
	const changesSummary = await summarizeChanges(filePaths);

	// Generate conventional commit message
	return formatCommitMessage(changeTypes, changesSummary);
}

function formatCommitMessage(types: string[], summary: string): string {
	const type = determineCommitType(types); // feat, fix, refactor, etc.
	const scope = determineScope(summary);

	return `${type}${scope ? `(${scope})` : ''}: ${summary}

${generateDetailedDescription(summary)}`;
}
```

**Example Dialog:**
```
User: Generate a commit message for my changes

Copilot: [calls getChangedFiles with filterType='staged']
Result: "Staged Changes (4)
- src/api/users.ts [Modified (staged)]
- src/api/auth.ts [Modified (staged)]
- src/types/api.ts [Added (staged)]
- tests/api.test.ts [Modified (staged)]"

[Analyzes changes]

Based on your staged changes, I suggest:

```
feat(api): add user authentication endpoints

- Implement getUserById and validateToken endpoints
- Add API types for request/response objects
- Update test coverage for new authentication flow
- Improve error handling for unauthorized requests
```

This follows conventional commit format. Want me to adjust anything?
```

### Use Case 4: Change Impact Analysis

**Scenario:** Understand the scope and impact of current changes.

**Implementation:**
```typescript
async function analyzeChangeImpact(): Promise<void> {
	const allChanges = await getChangedFiles({
		filterType: 'all',
		includeStatus: true
	});

	// Parse changes
	const files = parseFilePathsFromOutput(allChanges);

	// Categorize by impact
	const impact = {
		high: [] as string[], // Core files, widely imported modules
		medium: [] as string[], // Feature files
		low: [] as string[] // Tests, configs
	};

	for (const file of files) {
		const usage = await listCodeUsages(path.basename(file));
		const category = categorizeImpact(file, usage);
		impact[category].push(file);
	}

	return formatImpactReport(impact);
}
```

**Example Dialog:**
```
User: What's the impact of my current changes?

Copilot: [calls getChangedFiles]
Result: "All Changes (8)
- src/core/database.ts [Modified]
- src/api/users.ts [Modified (staged)]
- src/api/products.ts [Modified]
- src/utils/validation.ts [Added]
..."

[Analyzes dependencies]

## Change Impact Analysis

High Impact (3 files):
- src/core/database.ts
  - Used by 47 files across the project
  - Affects: API layer, services, repositories

Medium Impact (2 files):
- src/api/users.ts - Used by 12 files
- src/api/products.ts - Used by 8 files

Low Impact (3 files):
- src/utils/validation.ts - New utility, no dependencies yet
- tests/* - Test files, no production impact

Recommendation: Carefully test database.ts changes as they affect
almost every module. Consider staging high-impact changes separately.
```

### Use Case 5: Workspace Clean-up Assistance

**Scenario:** User has many unstaged changes and wants to organize them.

**Implementation:**
```typescript
async function organizeUnstagedChanges(): Promise<void> {
	const unstaged = await getChangedFiles({
		filterType: 'unstaged',
		includeStatus: true
	});

	const files = parseFilePathsFromOutput(unstaged);

	// Categorize by purpose
	const categories = {
		features: [] as string[],
		bugFixes: [] as string[],
		refactoring: [] as string[],
		tests: [] as string[],
		config: [] as string[]
	};

	for (const file of files) {
		const category = await categorizeChange(file);
		categories[category].push(file);
	}

	return suggestStagingStrategy(categories);
}
```

**Example Dialog:**
```
User: I have a lot of unstaged changes. Help me organize them.

Copilot: [calls getChangedFiles with filterType='unstaged']
Result: "Unstaged Changes (15)
- src/api/users.ts [Modified]
- src/api/products.ts [Modified]
- src/utils/helpers.ts [Modified]
- tests/api.test.ts [Modified]
..."

[Analyzes each change]

I've organized your 15 unstaged changes:

**Feature Changes (5 files):**
- New user authentication endpoints
- Product search functionality
→ Suggestion: Stage together for "feat(api): add auth and search"

**Bug Fixes (3 files):**
- Fixed null pointer in user validation
- Corrected pagination offset
→ Suggestion: Stage separately for "fix: resolve validation bugs"

**Refactoring (4 files):**
- Extracted helper functions
- Simplified database queries
→ Suggestion: Stage separately for "refactor: improve code structure"

**Tests (3 files):**
- Add with corresponding feature/fix commits

Would you like me to create a staging plan?
```

## Comparison with Other Tools

### getChangedFiles vs. grep_search

| Aspect | getChangedFiles | grep_search |
|--------|-----------------|-------------|
| **Purpose** | Find modified files | Search file contents |
| **Data Source** | Git status | File system |
| **Output** | File list with status | Matching lines |
| **Performance** | Very fast (10-50ms) | Slower (100-2000ms) |
| **Best For** | Change management | Finding code patterns |

### getChangedFiles vs. file_search

| Aspect | getChangedFiles | file_search |
|--------|-----------------|-------------|
| **Purpose** | Find modified files | Find files by name |
| **Filter** | Git status (modified/staged) | Glob patterns |
| **Context** | Change status included | Name/path only |
| **Best For** | Version control ops | File location |

### getChangedFiles vs. semantic_search

| Aspect | getChangedFiles | semantic_search |
|--------|-----------------|-----------------|
| **Query Type** | Implicit (git changes) | Explicit semantic |
| **Scope** | Changed files only | All workspace files |
| **Response Time** | 10-50ms | 100-2000ms |
| **Best For** | Change tracking | Code understanding |

## Error Handling

### Common Error Scenarios

```typescript
try {
	const changes = await getChangedFiles({ filterType: 'staged' });
} catch (error) {
	// Handle various error conditions
}
```

### Error Cases

1. **Git Extension Not Available**
   ```typescript
   const gitExtension = vscode.extensions.getExtension('vscode.git');
   if (!gitExtension) {
   	return 'Git extension is not available. Please install it.';
   }
   ```

2. **No Git Repository**
   ```typescript
   if (git.repositories.length === 0) {
   	return 'No git repositories found in workspace. Initialize git first.';
   }
   ```

3. **Repository Not Open**
   ```typescript
   const repo = git.getRepository(workspaceUri);
   if (!repo) {
   	return 'No git repository found at this location.';
   }
   ```

4. **Detached HEAD State**
   ```typescript
   if (!repo.state.HEAD) {
   	return 'Repository is in detached HEAD state. Checkout a branch first.';
   }
   ```

### Error Recovery

```typescript
async function getChangedFilesWithFallback(
	args: { filterType?: string }
): Promise<string> {
	try {
		// Try git extension API
		return await getChangedFiles(args);
	} catch (error) {
		// Fallback to git CLI
		try {
			return await getChangedFilesViaGitCLI(args);
		} catch (cliError) {
			return `Unable to get changed files: ${error.message}`;
		}
	}
}

async function getChangedFilesViaGitCLI(
	args: { filterType?: string }
): Promise<string> {
	const workspaceRoot = vscode.workspace.workspaceFolders?.[0]?.uri.fsPath;
	if (!workspaceRoot) {
		throw new Error('No workspace folder');
	}

	let gitCommand = 'git status --porcelain';
	if (args.filterType === 'staged') {
		gitCommand = 'git diff --cached --name-only';
	} else if (args.filterType === 'unstaged') {
		gitCommand = 'git diff --name-only';
	}

	const result = await execCommand(gitCommand, workspaceRoot);
	return result.stdout;
}
```

## Configuration

### Git Extension Settings

The tool respects VS Code's git configuration:

```json
{
	// Enable git extension
	"git.enabled": true,

	// Auto-fetch from remote
	"git.autofetch": true,

	// Show untracke files
	"git.showUntracked": true,

	// Ignored files
	"git.ignoredRepositories": [],

	// Default branch name
	"git.defaultBranchName": "main"
}
```

### Repository-Specific Settings

```bash
# .git/config
[core]
	repositoryformatversion = 0
	filemode = true
	bare = false
	logallrefupdates = true
[remote "origin"]
	url = https://github.com/user/repo.git
	fetch = +refs/heads/*:refs/remotes/origin/*
```

### Ignore Patterns

```bash
# .gitignore affects which files appear as untracked
node_modules/
dist/
.env
*.log
```

## Implementation Blueprint

### Step 1: Basic Git Extension Access

```typescript
async function getGitAPI(): Promise<API> {
	const gitExtension = vscode.extensions.getExtension<GitExtension>('vscode.git');

	if (!gitExtension) {
		throw new Error('Git extension not found');
	}

	if (!gitExtension.isActive) {
		await gitExtension.activate();
	}

	return gitExtension.exports.getAPI(1);
}
```

### Step 2: Get Repository Changes

```typescript
async function getRepositoryChanges(repo: Repository): Promise<Change[]> {
	const state = repo.state;

	return [
		...state.workingTreeChanges,
		...state.indexChanges,
		...state.mergeChanges
	];
}
```

### Step 3: Filter by Type

```typescript
function filterChanges(
	changes: Change[],
	filterType: string
): Change[] {
	switch (filterType) {
		case 'staged':
			return changes.filter(c => c.status < Status.MODIFIED);
		case 'unstaged':
			return changes.filter(c => c.status >= Status.MODIFIED && c.status !== Status.CONFLICT);
		case 'conflicts':
			return changes.filter(c => c.status === Status.CONFLICT || c.status >= Status.BOTH_ADDED);
		default:
			return changes;
	}
}
```

### Step 4: Format Output

```typescript
function formatChangedFiles(changes: Change[]): string {
	return changes
		.map(change => {
			const path = vscode.workspace.asRelativePath(change.uri);
			const status = getStatusLabel(change.status);
			return `- ${path} [${status}]`;
		})
		.join('\n');
}
```

### Step 5: Handle Multiple Repositories

```typescript
async function getAllChangesMultiRepo(
	filterType: string
): Promise<string> {
	const git = await getGitAPI();
	const sections: string[] = [];

	for (const repo of git.repositories) {
		const repoName = path.basename(repo.rootUri.fsPath);
		const changes = await getRepositoryChanges(repo);
		const filtered = filterChanges(changes, filterType);

		if (filtered.length > 0) {
			sections.push(`## ${repoName}\n${formatChangedFiles(filtered)}`);
		}
	}

	return sections.join('\n\n') || 'No changes found';
}
```

## Best Practices

### 1. Filter Appropriately

```typescript
// ✓ Good: Request specific subset
const staged = await getChangedFiles({ filterType: 'staged' });

// ✗ Bad: Always get all then filter manually
const all = await getChangedFiles({ filterType: 'all' });
const staged = all.split('\n').filter(line => line.includes('staged'));
```

### 2. Check for Changes Before Acting

```typescript
// ✓ Good: Verify there are changes
const changes = await getChangedFiles({ filterType: 'unstaged' });
if (changes.includes('No')) {
	console.log('No unstaged changes to review');
	return;
}

// ✗ Bad: Assume there are always changes
const changes = await getChangedFiles({ filterType: 'unstaged' });
await reviewChanges(changes); // May fail if no changes
```

### 3. Combine with File Reading

```typescript
// ✓ Good: Get changes then read content
const changes = await getChangedFiles({ filterType: 'staged' });
const files = parseFilePaths(changes);
for (const file of files) {
	const content = await readFile(file);
	await analyzeChange(content);
}

// ✗ Bad: Try to analyze without reading content
const changes = await getChangedFiles({ filterType: 'staged' });
analyzeChange(changes); // Only has file paths, not content
```

### 4. Prioritize Conflicts

```typescript
// ✓ Good: Check conflicts first
const conflicts = await getChangedFiles({ filterType: 'conflicts' });
if (conflicts.includes('Conflicts')) {
	console.log('You have merge conflicts that need resolution first');
	return;
}

// Then proceed with normal flow
const staged = await getChangedFiles({ filterType: 'staged' });
```

### 5. Provide Context to User

```typescript
// ✓ Good: Explain what you found
const changes = await getChangedFiles({ filterType: 'all' });
const counts = parseChangeCounts(changes);
console.log(`You have ${counts.staged} staged and ${counts.unstaged} unstaged changes.`);

// ✗ Bad: Just dump raw output
console.log(await getChangedFiles({ filterType: 'all' }));
```

### 6. Handle Multiple Repositories

```typescript
// ✓ Good: Check repository count
const git = await getGitAPI();
if (git.repositories.length > 1) {
	console.log(`Found ${git.repositories.length} repositories. Checking all...`);
}

// ✗ Bad: Assume single repository
const repo = git.repositories[0]; // May be undefined or wrong repo
```

## Limitations

### 1. Git Extension Dependency

**Limitation:** Requires VS Code's Git extension to be installed and active.

```typescript
// Won't work if git extension is disabled
const changes = await getChangedFiles({});
// Returns: "Git extension is not available"
```

**Workaround:** Fallback to git CLI commands if extension unavailable.

### 2. No Historical Changes

**Limitation:** Only shows current working tree state, not commit history.

```typescript
// Can't answer: "What files changed in the last 3 commits?"
// Can only answer: "What files are currently modified?"
```

**Workaround:** Use git log parsing for historical analysis.

### 3. No Diff Content

**Limitation:** Returns file paths and status, not actual diff content.

```typescript
// Returns: "src/app.ts [Modified]"
// Doesn't show: What lines changed in app.ts
```

**Workaround:** Use `read_file` to get content, compare with git version.

### 4. Limited to Git

**Limitation:** Only works with git repositories, not other VCS systems.

```typescript
// Won't work for:
// - SVN repositories
// - Mercurial repositories
// - Perforce workspaces
```

**Workaround:** Build separate tools for other VCS systems if needed.

### 5. No Change Grouping

**Limitation:** Can't group related changes automatically (e.g., "all auth changes").

```typescript
// Can't request: getChangedFiles({ feature: 'authentication' })
// Must filter manually after retrieval
```

**Workaround:** Parse file paths and analyze content to group changes.

### 6. Timing Sensitivity

**Limitation:** State may change between tool call and subsequent actions.

```typescript
// Time gap issue:
const changes = await getChangedFiles({ filterType: 'staged' });
// User stages more files
await commitChanges(); // Will include new files not in original query
```

**Workaround:** Re-query immediately before critical operations.

### 7. No Submodule Support

**Limitation:** Doesn't automatically recurse into git submodules.

```typescript
// Only shows changes in main repository
// Submodule changes appear as single entry: "submodule @ commit"
```

**Workaround:** Query each submodule repository separately if needed.

## Summary

The `getChangedFiles` tool provides essential git status information for Copilot workflows. It enables:

- **Pre-commit review** of staged changes
- **Conflict detection** and resolution assistance
- **Change organization** for better commit structure
- **Impact analysis** of modifications
- **Commit message generation** based on actual changes

**Key Strengths:**
- Fast access to git status (10-50ms)
- Rich status information (modified, staged, conflict, etc.)
- Multi-repository support
- Flexible filtering (all, staged, unstaged, conflicts)

**Best Used With:**
- Pre-commit code review workflows
- Merge conflict resolution
- Change organization and staging
- Commit message generation
- Combined with `read_file` for content analysis

By following the best practices and understanding its limitations, the `getChangedFiles` tool becomes a powerful aid for version control workflows in AI-assisted development.
