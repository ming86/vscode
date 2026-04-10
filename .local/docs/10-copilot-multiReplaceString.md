# GitHub Copilot Tool: multiReplaceString

## Overview

The `multiReplaceString` tool enables atomic execution of multiple string replacement operations across one or more files in a single transaction. It provides sophisticated conflict detection, partial success handling, and rollback capabilities to ensure data integrity when making coordinated changes.

**Key Capabilities:**

- Multiple simultaneous replacements in single or multiple files
- Atomic transaction semantics (all-or-nothing)
- Advanced conflict detection between replacements
- Partial success handling with detailed reporting
- Automatic rollback on failure
- Order-independent execution with dependency resolution

**Primary Use Cases:**

- Coordinated refactoring across related code sections
- Batch updates to configuration files
- Renaming with multiple occurrences
- Multi-step bug fixes requiring consistent changes

## Tool Declaration

```typescript
interface MultiReplaceStringTool {
    name: 'multi_replace_string_in_file';
    description: 'Apply multiple string replacements atomically across files';
    parameters: {
        replacements: Array<{
            filePath: string;
            oldString: string;
            newString: string;
            explanation: string;
        }>;
        explanation: string;
    }
}
```

## Input Parameters

### replacements

- **Type:** `Array<ReplacementOperation>`
- **Required:** Yes
- **Min Items:** 1
- **Max Items:** 50 (configurable)
- **Purpose:** List of replacement operations to execute

**ReplacementOperation Structure:**

```typescript
interface ReplacementOperation {
    filePath: string;       // Absolute path to target file
    oldString: string;      // Text to find (with 3-5 lines context)
    newString: string;      // Replacement text
    explanation: string;    // Description of this specific change
}
```

### explanation

- **Type:** `string`
- **Required:** Yes
- **Purpose:** Overall description of the batch operation
- **Example:** "Refactor UserService to use dependency injection"

### Example Input

```typescript
{
    replacements: [
        {
            filePath: '/project/src/services/userService.ts',
            oldString: `export class UserService {
    private db: Database;

    constructor() {
        this.db = new Database();
    }`,
            newString: `export class UserService {
    constructor(private db: Database) {}`,
            explanation: 'Convert to constructor injection'
        },
        {
            filePath: '/project/src/services/userService.ts',
            oldString: `async getUser(id: string): Promise<User> {
        return this.db.users.findById(id);
    }`,
            newString: `async getUser(id: string): Promise<User> {
        return await this.db.users.findById(id);
    }`,
            explanation: 'Add explicit await'
        },
        {
            filePath: '/project/src/app.ts',
            oldString: `const userService = new UserService();`,
            newString: `const userService = new UserService(database);`,
            explanation: 'Pass database instance'
        }
    ],
    explanation: 'Refactor UserService to use dependency injection'
}
```

## Architecture

### Transaction Flow

```
┌─────────────────────────────────────────────────────────────────┐
│                  multiReplaceString Entry                        │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│                    Phase 1: Validation                           │
│  • Validate all parameters                                      │
│  • Check file existence                                         │
│  • Verify read/write permissions                                │
│  • Validate replacement count limits                            │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│                   Phase 2: Backup Creation                       │
│  • Create in-memory backup of all target files                  │
│  • Store original content for rollback                          │
│  • Maintain file metadata (permissions, timestamps)             │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│              Phase 3: Conflict Detection                         │
│  • Group replacements by file                                   │
│  • Detect overlapping regions                                   │
│  • Check for ordering dependencies                              │
│  • Build execution graph                                        │
└─────────────────────────────────────────────────────────────────┘
                              ↓
                  Conflicts detected?
               ┌─────────┴─────────┐
              YES                   NO
               ↓                     ↓
┌──────────────────────────┐  ┌──────────────────────────────┐
│   Conflict Resolution    │  │  Phase 4: Match Resolution   │
│  • Analyze conflicts     │  │  • Find all oldString         │
│  • Determine safe order  │  │    matches                    │
│  • Resolve dependencies  │  │  • Use 4-strategy system      │
│  • Update exec graph     │  │  • Verify uniqueness          │
└──────────────────────────┘  │  • Map to file offsets        │
               ↓              └──────────────────────────────┘
               └─────────┬─────────┘
                         ↓
┌─────────────────────────────────────────────────────────────────┐
│            Phase 5: Ordered Execution Planning                   │
│  • Sort by file and offset (reverse order)                      │
│  • Ensure later offsets processed first                         │
│  • Prevent offset invalidation                                  │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│              Phase 6: Apply Replacements                         │
│  For each replacement (in reverse offset order):                │
│    1. Extract matched region                                    │
│    2. Splice in newString                                       │
│    3. Update file content buffer                                │
│    4. Track success/failure                                     │
└─────────────────────────────────────────────────────────────────┘
                              ↓
                     All successful?
                  ┌─────────┴─────────┐
                 YES                   NO
                  ↓                     ↓
┌──────────────────────────┐  ┌──────────────────────────────┐
│  Phase 7: Commit         │  │  Phase 7: Rollback           │
│  • Write all modified    │  │  • Restore from backups      │
│    files atomically      │  │  • Log failure details       │
│  • Clear backups         │  │  • Return partial results    │
│  • Log success           │  │  • Throw detailed error      │
└──────────────────────────┘  └──────────────────────────────┘
                  ↓                     ↓
┌──────────────────────────────────────────────────────────────────┐
│                    Return Result Summary                         │
│  • Total operations attempted                                    │
│  • Successful replacements                                       │
│  • Failed replacements with reasons                              │
│  • Files modified                                                │
│  • Rollback status                                               │
└──────────────────────────────────────────────────────────────────┘
```

### Conflict Detection Algorithm

```
┌─────────────────────────────────────────────────────────────────┐
│               Conflict Detection Process                         │
└─────────────────────────────────────────────────────────────────┘

Step 1: Group by File
─────────────────────
file1.ts: [replacement1, replacement2, replacement3]
file2.ts: [replacement4]
file3.ts: [replacement5, replacement6]

Step 2: Find Match Locations
─────────────────────────────
For each replacement:
    location = findMatch(fileContent, oldString)
    store: { replacement, startOffset, endOffset }

Step 3: Check for Overlaps (per file)
──────────────────────────────────────
For each pair of replacements in same file:
    if overlap(range1, range2):
        → CONFLICT: overlapping regions
    if range1.end > range2.start && range2.end > range1.start:
        → CONFLICT: intersecting changes

Step 4: Check for Dependencies
───────────────────────────────
For replacements in same file:
    if replacement2 depends on result of replacement1:
        → DEPENDENCY: must execute in order

    Detection:
    - Does oldString2 appear in newString1?
    - Does newString1 contain context from oldString2?

Step 5: Build Execution Graph
──────────────────────────────
nodes = all replacements
edges = conflicts and dependencies

if has_cycle(graph):
    → ERROR: circular dependencies

topological_sort(graph) → execution_order
```

## Internal Implementation

### Core Algorithm

```typescript
async function multiReplaceString(params: MultiReplaceParams): Promise<Result> {
    // Phase 1: Validation
    validateParams(params);

    // Phase 2: Backup
    const backups = await createBackups(params.replacements);

    try {
        // Phase 3: Conflict Detection
        const conflicts = await detectConflicts(params.replacements);

        if (conflicts.length > 0) {
            const resolved = await resolveConflicts(conflicts);
            if (!resolved) {
                throw new ConflictError('Unresolvable conflicts detected', conflicts);
            }
        }

        // Phase 4: Match Resolution
        const matches = await resolveAllMatches(params.replacements);

        // Phase 5: Order Planning
        const orderedOps = planExecution(matches);

        // Phase 6: Execute
        const results = await executeReplacements(orderedOps);

        // Phase 7: Commit
        await commitChanges(results);

        return {
            success: true,
            operations: results,
            filesModified: getUniqueFiles(results)
        };

    } catch (error) {
        // Phase 7: Rollback
        await rollback(backups);
        throw error;
    }
}
```

### Conflict Detection Implementation

```typescript
interface Conflict {
    type: 'overlap' | 'dependency' | 'ordering';
    replacements: [ReplacementOp, ReplacementOp];
    details: string;
    resolvable: boolean;
}

function detectConflicts(replacements: ReplacementOp[]): Conflict[] {
    const conflicts: Conflict[] = [];
    const byFile = groupByFile(replacements);

    for (const [filePath, ops] of byFile.entries()) {
        // Find match locations for all operations in this file
        const matches = ops.map(op => ({
            op,
            range: findMatchRange(filePath, op.oldString)
        }));

        // Check each pair for conflicts
        for (let i = 0; i < matches.length; i++) {
            for (let j = i + 1; j < matches.length; j++) {
                const match1 = matches[i];
                const match2 = matches[j];

                // Overlap detection
                if (rangesOverlap(match1.range, match2.range)) {
                    conflicts.push({
                        type: 'overlap',
                        replacements: [match1.op, match2.op],
                        details: `Replacements overlap at offsets ${match1.range.start}-${match2.range.end}`,
                        resolvable: false
                    });
                }

                // Dependency detection
                const dep = checkDependency(match1.op, match2.op);
                if (dep) {
                    conflicts.push({
                        type: 'dependency',
                        replacements: [match1.op, match2.op],
                        details: dep.reason,
                        resolvable: true
                    });
                }
            }
        }
    }

    return conflicts;
}

function rangesOverlap(range1: Range, range2: Range): boolean {
    return !(range1.end <= range2.start || range2.end <= range1.start);
}

function checkDependency(op1: ReplacementOp, op2: ReplacementOp): Dependency | null {
    // Does op2 depend on op1's result?
    if (op2.oldString.includes(op1.newString)) {
        return {
            dependent: op2,
            dependsOn: op1,
            reason: 'op2 references result of op1'
        };
    }

    // Does op1 depend on op2's result?
    if (op1.oldString.includes(op2.newString)) {
        return {
            dependent: op1,
            dependsOn: op2,
            reason: 'op1 references result of op2'
        };
    }

    return null;
}
```

### Execution Planning

```typescript
function planExecution(matches: MatchedOperation[]): OrderedOperation[] {
    // Group by file
    const byFile = new Map<string, MatchedOperation[]>();
    for (const match of matches) {
        const ops = byFile.get(match.filePath) || [];
        ops.push(match);
        byFile.set(match.filePath, ops);
    }

    // Sort operations within each file
    for (const [file, ops] of byFile.entries()) {
        // CRITICAL: Sort by offset in REVERSE order
        // This prevents earlier replacements from invalidating later offsets
        ops.sort((a, b) => b.range.start - a.range.start);
    }

    // Flatten back to single array
    const ordered: OrderedOperation[] = [];
    for (const ops of byFile.values()) {
        ordered.push(...ops.map((op, index) => ({
            ...op,
            executionOrder: index
        })));
    }

    return ordered;
}
```

### Atomic Execution

```typescript
async function executeReplacements(
    operations: OrderedOperation[]
): Promise<OperationResult[]> {
    const results: OperationResult[] = [];
    const fileContents = new Map<string, string>();

    // Load all file contents once
    for (const op of operations) {
        if (!fileContents.has(op.filePath)) {
            const content = await readFile(op.filePath);
            fileContents.set(op.filePath, content);
        }
    }

    // Execute operations (already in correct order)
    for (const op of operations) {
        try {
            let content = fileContents.get(op.filePath)!;

            // Perform replacement
            const before = content.substring(0, op.range.start);
            const after = content.substring(op.range.end);
            const newContent = before + op.newString + after;

            // Update in-memory content
            fileContents.set(op.filePath, newContent);

            results.push({
                operation: op,
                success: true,
                oldLength: op.oldString.length,
                newLength: op.newString.length
            });

        } catch (error) {
            results.push({
                operation: op,
                success: false,
                error: error.message
            });

            // Fail fast - stop on first error
            throw new ExecutionError('Replacement failed', results);
        }
    }

    // Write all modified files atomically
    for (const [filePath, content] of fileContents.entries()) {
        await writeFile(filePath, content);
    }

    return results;
}
```

### Rollback System

```typescript
interface Backup {
    filePath: string;
    content: string;
    metadata: FileMetadata;
}

async function createBackups(replacements: ReplacementOp[]): Promise<Backup[]> {
    const uniqueFiles = new Set(replacements.map(r => r.filePath));
    const backups: Backup[] = [];

    for (const filePath of uniqueFiles) {
        const content = await readFile(filePath);
        const metadata = await getFileMetadata(filePath);

        backups.push({ filePath, content, metadata });
    }

    return backups;
}

async function rollback(backups: Backup[]): Promise<void> {
    for (const backup of backups) {
        try {
            await writeFile(backup.filePath, backup.content);
            await restoreMetadata(backup.filePath, backup.metadata);
        } catch (error) {
            console.error(`Failed to rollback ${backup.filePath}:`, error);
            // Continue rolling back other files
        }
    }
}
```

## Dependencies

### Internal Services

- **IFileService**: File operations with atomic writes
- **ITextMatchService**: Multi-strategy string matching
- **ITransactionService**: Transaction management and rollback
- **ITelemetryService**: Operation tracking and analytics

### External Libraries

- **string-similarity**: For fuzzy matching
- **graph-algorithms**: For dependency resolution
- **atomic-file-write**: For atomic file operations

### VS Code APIs

- `vscode.workspace.fs`: File system operations
- `vscode.window.showWarningMessage`: User notifications on conflicts

## Performance Characteristics

### Time Complexity

```
Let:
- n = number of replacements
- m = average file size
- f = number of unique files

Operations:
- Validation: O(n)
- Backup: O(f × m)
- Conflict Detection: O(n² × m)  [pairwise comparison]
- Match Resolution: O(n × m)      [per-file matching]
- Execution: O(n × m)             [ordered replacements]
- Commit: O(f × m)                [write files]

Total: O(n² × m + f × m)
Dominated by conflict detection for large n
```

### Performance Benchmarks

```
Replacements    Files    File Size    Conflict Det.    Total Time
────────────────────────────────────────────────────────────────────
5               1        10 KB        10-20 ms         50-100 ms
10              3        50 KB        50-100 ms        200-300 ms
25              5        100 KB       200-400 ms       800-1200 ms
50              10       100 KB       800-1500 ms      2-4 sec
```

### Memory Usage

```
Component               Memory Usage
─────────────────────────────────────
Original file backups   f × m
Modified file buffers   f × m
Match metadata          n × 1 KB
Conflict graph          O(n²) nodes
Peak total:             2(f × m) + O(n²)
```

## Use Cases

### Use Case 1: Coordinated Refactoring

```typescript
// Refactor class to use dependency injection
{
    replacements: [
        {
            filePath: '/project/src/services/api.ts',
            oldString: `export class ApiService {
    private http: HttpClient;

    constructor() {
        this.http = new HttpClient();
    }`,
            newString: `export class ApiService {
    constructor(private http: HttpClient) {}`,
            explanation: 'Convert to constructor injection'
        },
        {
            filePath: '/project/src/services/api.ts',
            oldString: `private handleError(error: any): void {
        console.error('API Error:', error);
    }`,
            newString: `private handleError(error: any): void {
        this.logger.error('API Error:', error);
    }`,
            explanation: 'Use injected logger'
        },
        {
            filePath: '/project/src/app.ts',
            oldString: `const api = new ApiService();`,
            newString: `const api = new ApiService(httpClient);`,
            explanation: 'Pass dependencies'
        }
    ],
    explanation: 'Refactor ApiService to use dependency injection'
}
```

### Use Case 2: Multi-File Rename

```typescript
// Rename interface across files
{
    replacements: [
        {
            filePath: '/project/src/types/user.ts',
            oldString: `export interface UserData {`,
            newString: `export interface User {`,
            explanation: 'Rename interface'
        },
        {
            filePath: '/project/src/services/userService.ts',
            oldString: `async getUser(id: string): Promise<UserData> {`,
            newString: `async getUser(id: string): Promise<User> {`,
            explanation: 'Update return type'
        },
        {
            filePath: '/project/src/services/userService.ts',
            oldString: `import { UserData } from '../types/user';`,
            newString: `import { User } from '../types/user';`,
            explanation: 'Update import'
        }
    ],
    explanation: 'Rename UserData interface to User'
}
```

### Use Case 3: Configuration Batch Update

```typescript
// Update multiple configuration values
{
    replacements: [
        {
            filePath: '/project/config/app.json',
            oldString: `"timeout": 5000,`,
            newString: `"timeout": 10000,`,
            explanation: 'Increase timeout'
        },
        {
            filePath: '/project/config/app.json',
            oldString: `"retries": 3,`,
            newString: `"retries": 5,`,
            explanation: 'Increase retry count'
        },
        {
            filePath: '/project/config/app.json',
            oldString: `"logLevel": "info"`,
            newString: `"logLevel": "debug"`,
            explanation: 'Enable debug logging'
        }
    ],
    explanation: 'Update API configuration for production'
}
```

### Use Case 4: Bug Fix with Multiple Locations

```typescript
// Fix the same bug pattern in multiple places
{
    replacements: [
        {
            filePath: '/project/src/utils/math.ts',
            oldString: `return items.reduce((sum, item) => sum + item.price, 0);`,
            newString: `return items.reduce((sum, item) => sum + (item.price * item.quantity), 0);`,
            explanation: 'Fix calculation in math util'
        },
        {
            filePath: '/project/src/services/cart.ts',
            oldString: `const total = cart.items.reduce((sum, item) => sum + item.price, 0);`,
            newString: `const total = cart.items.reduce((sum, item) => sum + (item.price * item.quantity), 0);`,
            explanation: 'Fix calculation in cart service'
        }
    ],
    explanation: 'Fix total calculation to include quantity'
}
```

## Comparison with Other Tools

### multiReplaceString vs replaceString

| Feature | multiReplaceString | replaceString |
|---------|-------------------|---------------|
| **Operations** | Multiple simultaneous | Single operation |
| **Atomicity** | All-or-nothing | N/A |
| **Conflict Detection** | Yes, automatic | N/A |
| **Rollback** | Automatic on failure | N/A |
| **Performance** | O(n²) for conflicts | O(n) |
| **Complexity** | High | Low |
| **Best For** | Coordinated changes | Single targeted change |

### multiReplaceString vs applyPatch

| Feature | multiReplaceString | applyPatch |
|---------|-------------------|------------|
| **Format** | Simple operations | Unified diff |
| **Context** | User-provided | Built-in |
| **Flexibility** | High (any changes) | Medium (diff format) |
| **Healing** | No (exact match) | Yes (6-pass system) |
| **Best For** | Programmatic batches | Generated patches |

### When to Use multiReplaceString

**✅ Use multiReplaceString when:**

- Making multiple related changes that must succeed together
- Refactoring requires coordinated updates
- Need automatic conflict detection
- Want atomic transaction semantics
- Making 2-50 changes (sweet spot)

**❌ Don't use multiReplaceString when:**

- Making a single change (use replaceString)
- Changes are independent (separate replaceString calls)
- Need advanced healing (use applyPatch)
- > 50 operations (performance degradation)

## Error Handling

### Error Scenarios

#### 1. Overlapping Replacements

```typescript
{
    error: 'ConflictError',
    type: 'overlap',
    message: 'Replacements have overlapping regions',
    conflicts: [
        {
            replacement1: { filePath: '...', oldString: '...', line: 42 },
            replacement2: { filePath: '...', oldString: '...', line: 44 },
            overlap: { start: 42, end: 46 },
            resolution: 'Combine into single replacement or provide non-overlapping context'
        }
    ]
}
```

#### 2. Circular Dependencies

```typescript
{
    error: 'DependencyError',
    type: 'circular',
    message: 'Circular dependency detected between replacements',
    cycle: [
        { operation: 1, dependsOn: 2 },
        { operation: 2, dependsOn: 3 },
        { operation: 3, dependsOn: 1 }
    ],
    resolution: 'Break cycle by reordering or combining operations'
}
```

#### 3. Partial Failure

```typescript
{
    error: 'PartialFailureError',
    message: 'Some replacements failed, all changes rolled back',
    results: [
        { operation: 1, success: true },
        { operation: 2, success: true },
        { operation: 3, success: false, error: 'Match not found' },
        { operation: 4, success: false, error: 'Not executed (rolled back)' }
    ],
    rollbackStatus: 'complete'
}
```

#### 4. Ambiguous Matches

```typescript
{
    error: 'AmbiguousMatchError',
    message: 'Multiple matches found for one or more operations',
    ambiguousOps: [
        {
            operation: 2,
            filePath: '/path/to/file.ts',
            matches: [
                { line: 42, context: '...' },
                { line: 156, context: '...' }
            ],
            resolution: 'Provide more unique context'
        }
    ]
}
```

### Error Recovery Strategies

```typescript
async function multiReplaceWithRecovery(
    params: MultiReplaceParams
): Promise<Result> {
    try {
        return await multiReplaceString(params);

    } catch (error) {
        if (error.type === 'ConflictError' && error.resolvable) {
            // Strategy 1: Auto-resolve dependencies
            console.log('Resolving conflicts automatically...');
            const resolved = autoResolveConflicts(error.conflicts);
            return await multiReplaceString({ ...params, replacements: resolved });

        } else if (error.type === 'AmbiguousMatchError') {
            // Strategy 2: Add more context
            console.log('Adding more context to ambiguous operations...');
            const clarified = await clarifyAmbiguous(error.ambiguousOps);
            return await multiReplaceString({ ...params, replacements: clarified });

        } else if (error.type === 'PartialFailureError') {
            // Strategy 3: Retry failed operations
            console.log('Retrying failed operations...');
            const failedOps = error.results.filter(r => !r.success);
            const retry = adjustFailedOps(failedOps);
            return await multiReplaceString({
                replacements: retry,
                explanation: 'Retry failed operations'
            });

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
    "copilot.tools.multiReplace.enabled": true,
    "copilot.tools.multiReplace.maxOperations": 50,
    "copilot.tools.multiReplace.conflictDetection": "strict",
    "copilot.tools.multiReplace.autoResolveConflicts": true,
    "copilot.tools.multiReplace.failureMode": "rollback",
    "copilot.tools.multiReplace.backupEnabled": true
}
```

### Configuration Options

| Setting | Type | Default | Description |
|---------|------|---------|-------------|
| `enabled` | boolean | true | Enable/disable the tool |
| `maxOperations` | number | 50 | Maximum operations per call |
| `conflictDetection` | string | "strict" | "strict", "relaxed", or "off" |
| `autoResolveConflicts` | boolean | true | Attempt automatic conflict resolution |
| `failureMode` | string | "rollback" | "rollback" or "partial" |
| `backupEnabled` | boolean | true | Create backups before changes |

## Implementation Blueprint

### Step 1: Tool Registration

```typescript
// src/extension/tools/multiReplaceStringTool.ts
export class MultiReplaceStringTool implements Tool {
    static readonly ID = 'multi_replace_string_in_file';

    constructor(
        @IFileService private fileService: IFileService,
        @ITransactionService private transactions: ITransactionService,
        @IConflictDetector private conflictDetector: IConflictDetector
    ) {}

    getDefinition(): ToolDefinition {
        return {
            name: 'multi_replace_string_in_file',
            description: 'Apply multiple replacements atomically',
            parameters: {
                type: 'object',
                properties: {
                    replacements: {
                        type: 'array',
                        items: {
                            type: 'object',
                            properties: {
                                filePath: { type: 'string' },
                                oldString: { type: 'string' },
                                newString: { type: 'string' },
                                explanation: { type: 'string' }
                            },
                            required: ['filePath', 'oldString', 'newString', 'explanation']
                        },
                        minItems: 1
                    },
                    explanation: { type: 'string' }
                },
                required: ['replacements', 'explanation']
            }
        };
    }
}
```

### Step 2: Transaction Service

```typescript
// src/platform/transaction/transactionService.ts
export class TransactionService implements ITransactionService {
    async executeTransaction<T>(
        operation: () => Promise<T>,
        rollback: () => Promise<void>
    ): Promise<T> {
        try {
            const result = await operation();
            return result;
        } catch (error) {
            await rollback();
            throw error;
        }
    }
}
```

### Step 3: Conflict Detector

```typescript
// src/platform/conflict/conflictDetector.ts
export class ConflictDetector implements IConflictDetector {
    detectConflicts(
        operations: ReplacementOp[],
        fileContents: Map<string, string>
    ): Conflict[] {
        const conflicts: Conflict[] = [];
        const byFile = groupByFile(operations);

        for (const [file, ops] of byFile.entries()) {
            const content = fileContents.get(file)!;
            const matches = this.resolveMatches(ops, content);
            const detected = this.checkOverlaps(matches);
            conflicts.push(...detected);
        }

        return conflicts;
    }
}
```

## Best Practices

### 1. Group Related Changes

**✅ Good: Logically grouped**

```typescript
{
    replacements: [
        // All changes related to dependency injection
        { /* constructor change */ },
        { /* usage change 1 */ },
        { /* usage change 2 */ }
    ],
    explanation: 'Convert to dependency injection'
}
```

**❌ Bad: Unrelated changes**

```typescript
{
    replacements: [
        { /* dependency injection */ },
        { /* unrelated bug fix */ },
        { /* style change */ }
    ]
}
```

### 2. Order by Dependency

```typescript
{
    replacements: [
        // First: Define the interface
        { filePath: 'types.ts', oldString: '...', newString: '...' },
        // Then: Update imports
        { filePath: 'service.ts', oldString: 'import ...', newString: '...' },
        // Finally: Use in code
        { filePath: 'service.ts', oldString: 'function ...', newString: '...' }
    ]
}
```

### 3. Provide Unique Context

```typescript
// Each oldString should be unique within its file
{
    replacements: [
        {
            oldString: `export class UserService {
    constructor(private db: Database) {}  // Unique context

    async getUser(id: string): Promise<User> {`
        },
        {
            oldString: `async getUser(id: string): Promise<User> {
        return await this.db.users.findById(id);  // Different context
    }`
        }
    ]
}
```

### 4. Limit Batch Size

```typescript
// Keep under 50 operations for best performance
if (replacements.length > 50) {
    // Split into multiple batches
    const batches = chunk(replacements, 25);
    for (const batch of batches) {
        await multiReplaceString({ replacements: batch, explanation: '...' });
    }
}
```

### 5. Handle Rollback Gracefully

```typescript
try {
    await multiReplaceString(params);
} catch (error) {
    if (error.type === 'PartialFailureError') {
        console.log('Changes rolled back successfully');
        console.log('Failed operations:', error.results.filter(r => !r.success));
        // Analyze and retry if appropriate
    }
}
```

## Limitations

### 1. Operation Count Limit

- Default maximum: 50 operations
- Performance degrades with O(n²) conflict detection
- Memory usage increases linearly

### 2. No Cross-Operation Context

- Each operation matches independently
- Cannot use result of one operation in another's search
- Dependencies must be explicitly ordered

### 3. Atomic Semantics Cost

- All files held in memory during transaction
- Memory usage: 2x total file size
- Large files (> 1 MB each) can cause issues

### 4. Conflict Resolution Limitations

- Overlapping changes cannot be auto-resolved
- Circular dependencies will fail
- Manual intervention required for complex conflicts

### 5. No Partial Commit

- Either all operations succeed or all rolled back
- No "commit successful operations" mode
- Use separate calls if partial success acceptable

## Related Tools

- **replaceString**: For single replacements
- **insertEdit**: For adding new code
- **applyPatch**: For complex refactorings
- **createFile**: For creating new files

## Version History

- **v1.0**: Initial implementation with basic conflict detection
- **v1.1**: Added automatic dependency resolution
- **v1.2**: Improved rollback reliability
- **v1.3**: Added partial success reporting
- **v2.0**: Complete rewrite with transaction semantics

## References

- Transaction Service: `src/platform/transaction/`
- Conflict Detector: `src/platform/conflict/`
- Tool Framework: `src/extension/tools/`
- File Service: `src/platform/files/`
