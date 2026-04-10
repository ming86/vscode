# GitHub Copilot Tool: createDirectory

## Overview

The `createDirectory` tool provides recursive directory creation with `mkdir -p` semantics, ensuring all parent directories are created as needed. It includes safety checks, permission management, and idempotent behavior.

**Key Capabilities:**

- Recursive parent directory creation (mkdir -p behavior)
- Idempotent operation (no error if directory exists)
- Atomic directory creation
- Permission management (Unix systems)
- Validation and safety checks
- Cross-platform compatibility

**Primary Use Cases:**

- Creating nested directory structures
- Scaffolding project layouts
- Organizing file hierarchies
- Preparing output directories
- Setting up test fixtures

## Tool Declaration

```typescript
interface CreateDirectoryTool {
    name: 'create_directory';
    description: 'Create a directory and all necessary parent directories';
    parameters: {
        dirPath: string;            // Absolute path for the directory
        explanation?: string;       // Optional description
        permissions?: string;       // Unix permissions (e.g., "0755")
    }
}
```

## Input Parameters

### dirPath

- **Type:** `string`
- **Required:** Yes
- **Format:** Absolute directory path
- **Behavior:** Creates all missing parent directories
- **Example:** `/Users/ming/project/src/services/api`

### explanation

- **Type:** `string`
- **Required:** No
- **Purpose:** User-facing description
- **Example:** "Create directory for API service modules"

### permissions

- **Type:** `string`
- **Required:** No
- **Default:** `"0755"` (rwxr-xr-x)
- **Format:** Octal string (e.g., "0755", "0700")
- **Platform:** Unix/Linux/macOS only (ignored on Windows)
- **Example:** `"0755"` (owner: rwx, group: r-x, others: r-x)

## Architecture

### Directory Creation Flow

```
┌─────────────────────────────────────────────────────────────────┐
│                  createDirectory Tool Entry                      │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│                     Phase 1: Validation                          │
│  • Validate dirPath format                                      │
│  • Check path is absolute                                       │
│  • Validate permissions format                                  │
│  • Check for invalid characters                                 │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│                  Phase 2: Path Normalization                     │
│  • Resolve to absolute path                                     │
│  • Normalize separators                                         │
│  • Remove trailing slashes                                      │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│                  Phase 3: Existence Check                        │
│  • Check if path exists                                         │
│  • If exists, verify it's a directory                           │
│  • If directory exists, return success (idempotent)             │
└─────────────────────────────────────────────────────────────────┘
                              ↓
                      Path exists?
                  ┌─────────┴─────────┐
                 YES                   NO
                  ↓                     ↓
            Is directory?         ┌─────────────────────────────┐
         ┌──────┴──────┐          │  Phase 4: Parse Path        │
        YES            NO          │  • Split into segments       │
         ↓              ↓          │  • Identify all parents      │
┌──────────────┐  ┌─────────────┐ └─────────────────────────────┘
│   SUCCESS    │  │    ERROR    │              ↓
│ (idempotent) │  │  Not a dir  │ ┌─────────────────────────────┐
└──────────────┘  └─────────────┘ │  Phase 5: Recursive Create  │
                                   │  For each directory:         │
                                   │    1. Check if exists        │
                                   │    2. If not, create         │
                                   │    3. Set permissions        │
                                   └─────────────────────────────┘
                                                ↓
                                   ┌─────────────────────────────┐
                                   │  Phase 6: Verify Creation   │
                                   │  • Check directory exists    │
                                   │  • Verify permissions        │
                                   │  • Check write access        │
                                   └─────────────────────────────┘
                                                ↓
                                   ┌─────────────────────────────┐
                                   │  Phase 7: Post-Creation     │
                                   │  • Trigger file watchers     │
                                   │  • Update workspace index    │
                                   │  • Log telemetry             │
                                   └─────────────────────────────┘
                                                ↓
┌─────────────────────────────────────────────────────────────────┐
│                    Success Response                              │
│  • Return created directory path                                │
│  • Return list of created directories                           │
│  • Return success status                                        │
└─────────────────────────────────────────────────────────────────┘
```

### Recursive Creation Algorithm

```
Target: /Users/ming/project/src/services/api

Step 1: Normalize Path
───────────────────────
Input:  /Users/ming/project/src/services/api/
Output: /Users/ming/project/src/services/api

Step 2: Split into Segments
────────────────────────────
[
    '/',
    '/Users',
    '/Users/ming',
    '/Users/ming/project',
    '/Users/ming/project/src',
    '/Users/ming/project/src/services',
    '/Users/ming/project/src/services/api'
]

Step 3: Check Each Segment
───────────────────────────
/                                           → exists ✓ (skip)
/Users                                      → exists ✓ (skip)
/Users/ming                                 → exists ✓ (skip)
/Users/ming/project                         → exists ✓ (skip)
/Users/ming/project/src                     → exists ✓ (skip)
/Users/ming/project/src/services            → MISSING → CREATE
/Users/ming/project/src/services/api        → MISSING → CREATE

Step 4: Create Missing Directories
───────────────────────────────────
mkdir(/Users/ming/project/src/services, 0755)  → success
mkdir(/Users/ming/project/src/services/api, 0755)  → success

Step 5: Verify
──────────────
Check /Users/ming/project/src/services/api exists → ✓
Check permissions → ✓
Check writable → ✓
```

## Internal Implementation

### Core Algorithm

```typescript
async function createDirectory(params: CreateDirectoryParams): Promise<CreateDirectoryResult> {
    // Phase 1: Validation
    validateDirPath(params.dirPath);
    validatePermissions(params.permissions);

    // Phase 2: Normalization
    const normalizedPath = path.resolve(params.dirPath);

    // Phase 3: Existence check
    try {
        const stats = await fs.promises.stat(normalizedPath);
        if (stats.isDirectory()) {
            // Directory already exists - idempotent success
            return {
                success: true,
                dirPath: normalizedPath,
                created: false,
                alreadyExists: true
            };
        } else {
            throw new NotADirectoryError(`Path exists but is not a directory: ${normalizedPath}`);
        }
    } catch (error) {
        if (error.code !== 'ENOENT') {
            throw error; // Some other error
        }
        // Directory doesn't exist, continue to creation
    }

    // Phase 4: Recursive creation
    const createdDirs = await createDirectoryRecursive(
        normalizedPath,
        params.permissions || '0755'
    );

    // Phase 5: Verify
    await verifyDirectory(normalizedPath);

    // Phase 6: Post-creation
    await notifyFileWatchers(normalizedPath);

    return {
        success: true,
        dirPath: normalizedPath,
        created: true,
        createdDirectories: createdDirs
    };
}
```

### Recursive Creation Implementation

```typescript
async function createDirectoryRecursive(
    dirPath: string,
    permissions: string
): Promise<string[]> {
    const createdDirs: string[] = [];
    const normalizedPath = path.resolve(dirPath);

    // Base case: check if exists
    try {
        const stats = await fs.promises.stat(normalizedPath);
        if (stats.isDirectory()) {
            return []; // Already exists, nothing to create
        } else {
            throw new NotADirectoryError(`Path exists but is not a directory: ${normalizedPath}`);
        }
    } catch (error) {
        if (error.code !== 'ENOENT') {
            throw error;
        }
        // Directory doesn't exist, continue
    }

    // Recursive case: ensure parent exists
    const parent = path.dirname(normalizedPath);
    if (parent !== normalizedPath) { // Prevent infinite loop at root
        const parentCreated = await createDirectoryRecursive(parent, permissions);
        createdDirs.push(...parentCreated);
    }

    // Create this directory
    try {
        const mode = parseInt(permissions, 8);
        await fs.promises.mkdir(normalizedPath, { mode });
        createdDirs.push(normalizedPath);
    } catch (error) {
        // Race condition: another process created it between stat and mkdir
        if (error.code === 'EEXIST') {
            // Verify it's actually a directory
            const stats = await fs.promises.stat(normalizedPath);
            if (!stats.isDirectory()) {
                throw new NotADirectoryError(`Path exists but is not a directory: ${normalizedPath}`);
            }
            // It's a directory, that's fine
        } else {
            throw error;
        }
    }

    return createdDirs;
}
```

### Alternative: Node.js Built-in (Modern)

```typescript
async function createDirectoryModern(
    dirPath: string,
    permissions: string
): Promise<void> {
    // Node.js 10.12.0+ has recursive option built-in
    const mode = parseInt(permissions, 8);

    await fs.promises.mkdir(dirPath, {
        recursive: true,
        mode: mode
    });
}
```

### Validation Functions

```typescript
function validateDirPath(dirPath: string): void {
    if (!dirPath || typeof dirPath !== 'string') {
        throw new ValidationError('dirPath must be a non-empty string');
    }

    if (!path.isAbsolute(dirPath)) {
        throw new ValidationError('dirPath must be an absolute path');
    }

    // Check for invalid characters
    const invalidChars = process.platform === 'win32'
        ? /[<>:"|?*\x00-\x1F]/
        : /[\x00]/;

    if (invalidChars.test(dirPath)) {
        throw new ValidationError('dirPath contains invalid characters');
    }

    // Check path length
    const maxLength = process.platform === 'win32' ? 260 : 4096;
    if (dirPath.length > maxLength) {
        throw new ValidationError(`dirPath exceeds maximum length of ${maxLength}`);
    }
}

function validatePermissions(permissions?: string): void {
    if (!permissions) return;

    // Must be octal string: "0755", "0700", etc.
    if (!/^0[0-7]{3}$/.test(permissions)) {
        throw new ValidationError(
            `Invalid permissions format: ${permissions}. ` +
            `Expected octal format like "0755"`
        );
    }

    const mode = parseInt(permissions, 8);

    // Permissions must be in valid range (0o000 to 0o777)
    if (mode < 0 || mode > 0o777) {
        throw new ValidationError(
            `Permissions out of range: ${permissions}. ` +
            `Must be between 0000 and 0777`
        );
    }
}
```

### Verification

```typescript
async function verifyDirectory(dirPath: string): Promise<void> {
    // Check existence
    let stats: fs.Stats;
    try {
        stats = await fs.promises.stat(dirPath);
    } catch (error) {
        throw new VerificationError(`Directory was not created: ${dirPath}`);
    }

    // Check it's a directory
    if (!stats.isDirectory()) {
        throw new VerificationError(`Path exists but is not a directory: ${dirPath}`);
    }

    // Check write permissions (try to create a temp file)
    const testFile = path.join(dirPath, `.write-test-${Date.now()}`);
    try {
        await fs.promises.writeFile(testFile, '', { flag: 'wx' });
        await fs.promises.unlink(testFile);
    } catch (error) {
        throw new VerificationError(`Directory is not writable: ${dirPath}`);
    }
}
```

### Cross-Platform Handling

```typescript
function normalizePath(dirPath: string): string {
    // Normalize separators
    let normalized = path.normalize(dirPath);

    // On Windows, handle UNC paths
    if (process.platform === 'win32' && normalized.startsWith('\\\\')) {
        // Keep UNC prefix
        return normalized;
    }

    // Remove trailing slash (except root)
    if (normalized.length > 1 && normalized.endsWith(path.sep)) {
        normalized = normalized.slice(0, -1);
    }

    return normalized;
}

function platformSpecificPermissions(permissions: string): number {
    if (process.platform === 'win32') {
        // Windows doesn't use Unix permissions
        // Just use default
        return 0o755;
    }

    return parseInt(permissions, 8);
}
```

## Dependencies

### Internal Services

- **IFileService**: Core file system operations
- **IWorkspaceService**: Workspace indexing
- **IValidationService**: Input validation
- **ITelemetryService**: Usage tracking

### External Libraries

- **fs/promises**: Node.js file system
- **path**: Path manipulation
- **vscode.workspace.fs**: VS Code file system API (web)

### VS Code APIs

- `vscode.workspace.fs.createDirectory`: Directory creation (web)
- `vscode.workspace.onDidCreateFiles`: File system events

## Performance Characteristics

### Time Complexity

```
Let d = depth of directory structure

Operation                Time Complexity
────────────────────────────────────────
Path validation         O(1)
Path normalization      O(n) where n = path length
Existence checks        O(d) - one per directory level
Directory creation      O(d) - one mkdir per level
Verification            O(1)
Total                   O(d)
```

### Performance Benchmarks

```
Directory Depth    Creation Time    Notes
─────────────────────────────────────────────────────
1                 5-10 ms          Single directory
3                 10-15 ms         Typical project structure
5                 15-25 ms         Nested feature structure
10                25-40 ms         Deep nesting
20                40-70 ms         Unusually deep

Already exists    < 1 ms           Idempotent check
```

### Memory Usage

```
Component               Memory Usage
─────────────────────────────────────
Path buffer             ~1 KB per directory
Created dirs array      ~100 bytes per directory
Overhead                ~5 KB
Total (10 levels):      ~15 KB
```

## Use Cases

### Use Case 1: Create Feature Directory Structure

```typescript
// Create nested directory for a new feature
{
    dirPath: '/project/src/features/authentication',
    explanation: 'Create directory structure for authentication feature'
}

// Then create subdirectories
await createDirectory({
    dirPath: '/project/src/features/authentication/controllers'
});
await createDirectory({
    dirPath: '/project/src/features/authentication/services'
});
await createDirectory({
    dirPath: '/project/src/features/authentication/models'
});
```

### Use Case 2: Create Output Directory

```typescript
// Ensure output directory exists before writing files
{
    dirPath: '/project/dist/bundles',
    explanation: 'Create output directory for bundled assets'
}
```

### Use Case 3: Scaffold Project Structure

```typescript
// Create complete project structure
const directories = [
    '/project/src',
    '/project/src/components',
    '/project/src/services',
    '/project/src/utils',
    '/project/src/types',
    '/project/test',
    '/project/test/unit',
    '/project/test/integration',
    '/project/docs',
    '/project/dist'
];

for (const dir of directories) {
    await createDirectory({
        dirPath: dir,
        explanation: `Create ${path.basename(dir)} directory`
    });
}
```

### Use Case 4: Create Test Fixtures Directory

```typescript
// Create directory for test fixtures with restricted permissions
{
    dirPath: '/project/test/fixtures/temp',
    permissions: '0700',  // Only owner can access
    explanation: 'Create temporary directory for test fixtures'
}
```

### Use Case 5: Create Logs Directory

```typescript
// Create logs directory with appropriate permissions
{
    dirPath: '/var/log/myapp',
    permissions: '0755',
    explanation: 'Create application logs directory'
}
```

## Comparison with Other Tools

### createDirectory vs createFile

| Feature | createDirectory | createFile |
|---------|----------------|-----------|
| **Purpose** | Create directories | Create files |
| **Recursive** | Yes (mkdir -p) | Creates parent dirs |
| **Idempotent** | Yes (no error if exists) | No (error if exists) |
| **Permissions** | Configurable | Fixed |
| **Content** | N/A | Required |

### When to Use createDirectory

**✅ Use createDirectory when:**

- Creating directory structures
- Preparing locations for file creation
- Scaffolding project layouts
- Ensuring output directories exist
- Organizing file hierarchies

**❌ Don't use createDirectory when:**

- Creating files (use createFile)
- File's parent will be auto-created by createFile
- Single-level directory (though it still works)

## Error Handling

### Error Scenarios

#### 1. Path Exists But Is Not A Directory

```typescript
{
    error: 'NotADirectoryError',
    message: 'Path exists but is not a directory',
    path: '/path/to/file.txt',
    type: 'file',
    suggestion: 'Choose a different path or remove the existing file'
}
```

#### 2. Permission Denied

```typescript
{
    error: 'PermissionError',
    message: 'Cannot create directory: permission denied',
    path: '/restricted/directory',
    parentPermissions: 'r-xr-xr-x',
    suggestion: 'Check parent directory permissions or use a different location'
}
```

#### 3. Invalid Path

```typescript
{
    error: 'ValidationError',
    message: 'dirPath contains invalid characters',
    path: '/path/with/<invalid>',
    invalidChars: ['<', '>'],
    suggestion: 'Remove invalid characters from path'
}
```

#### 4. Path Too Long

```typescript
{
    error: 'ValidationError',
    message: 'dirPath exceeds maximum length',
    pathLength: 300,
    maxLength: 260,
    platform: 'win32',
    suggestion: 'Use a shorter path or enable long paths on Windows'
}
```

#### 5. Invalid Permissions

```typescript
{
    error: 'ValidationError',
    message: 'Invalid permissions format',
    permissions: '755',  // Missing leading 0
    expectedFormat: '0755',
    suggestion: 'Use octal format with leading 0, e.g., "0755"'
}
```

### Error Recovery

```typescript
async function createDirectoryWithRecovery(
    params: CreateDirectoryParams
): Promise<void> {
    try {
        await createDirectory(params);
    } catch (error) {
        if (error.type === 'NotADirectoryError') {
            // Strategy 1: Use alternative path
            console.log('Path exists as file, using alternative...');
            const altPath = `${params.dirPath}_dir`;
            await createDirectory({ ...params, dirPath: altPath });

        } else if (error.type === 'PermissionError') {
            // Strategy 2: Try with sudo or different location
            console.log('Permission denied, trying alternative location...');
            const altPath = findWritableAlternative(params.dirPath);
            if (altPath) {
                await createDirectory({ ...params, dirPath: altPath });
            } else {
                throw error;
            }

        } else if (error.type === 'ValidationError' && error.field === 'pathLength') {
            // Strategy 3: Shorten path
            console.log('Path too long, using shorter version...');
            const shortenedPath = shortenPath(params.dirPath);
            await createDirectory({ ...params, dirPath: shortenedPath });

        } else {
            throw error; // Unrecoverable
        }
    }
}

function findWritableAlternative(originalPath: string): string | null {
    // Try user's home directory
    const home = os.homedir();
    const basename = path.basename(originalPath);
    const altPath = path.join(home, '.myapp', basename);

    try {
        fs.accessSync(path.dirname(altPath), fs.constants.W_OK);
        return altPath;
    } catch {
        return null;
    }
}
```

## Configuration

### VS Code Settings

```json
{
    "copilot.tools.createDirectory.enabled": true,
    "copilot.tools.createDirectory.defaultPermissions": "0755",
    "copilot.tools.createDirectory.verifyAfterCreation": true,
    "copilot.tools.createDirectory.idempotent": true
}
```

### Configuration Options

| Setting | Type | Default | Description |
|---------|------|---------|-------------|
| `enabled` | boolean | true | Enable/disable the tool |
| `defaultPermissions` | string | "0755" | Default Unix permissions |
| `verifyAfterCreation` | boolean | true | Verify directory after creation |
| `idempotent` | boolean | true | No error if directory exists |

## Implementation Blueprint

### Step 1: Tool Registration

```typescript
// src/extension/tools/createDirectoryTool.ts
export class CreateDirectoryTool implements Tool {
    static readonly ID = 'create_directory';

    constructor(
        @IFileService private fileService: IFileService,
        @IValidationService private validator: IValidationService
    ) {}

    getDefinition(): ToolDefinition {
        return {
            name: 'create_directory',
            description: 'Create a directory with mkdir -p behavior',
            parameters: {
                type: 'object',
                properties: {
                    dirPath: { type: 'string' },
                    explanation: { type: 'string' },
                    permissions: { type: 'string', default: '0755' }
                },
                required: ['dirPath']
            }
        };
    }
}
```

### Step 2: File Service Extension

```typescript
// src/platform/files/fileService.ts
export class FileService implements IFileService {
    async createDirectory(
        path: URI,
        options: CreateDirectoryOptions
    ): Promise<void> {
        // Use recursive option if available
        await this.mkdir(path, { recursive: true, mode: options.mode });
    }

    private async mkdir(
        path: URI,
        options: { recursive: boolean; mode?: number }
    ): Promise<void> {
        if (this.isNodeEnvironment) {
            const mode = options.mode || 0o755;
            await fs.promises.mkdir(path.fsPath, {
                recursive: options.recursive,
                mode
            });
        } else {
            // Web environment
            await vscode.workspace.fs.createDirectory(path);
        }
    }
}
```

### Step 3: Execution Logic

```typescript
async execute(params: CreateDirectoryParams): Promise<ToolResult> {
    // 1. Validate
    this.validator.validateDirPath(params.dirPath);
    this.validator.validatePermissions(params.permissions);

    // 2. Normalize
    const normalizedPath = path.resolve(params.dirPath);
    const uri = URI.file(normalizedPath);

    // 3. Check existence (idempotent)
    try {
        const stat = await this.fileService.stat(uri);
        if (stat.type === FileType.Directory) {
            return {
                success: true,
                dirPath: normalizedPath,
                alreadyExists: true
            };
        } else {
            throw new NotADirectoryError(normalizedPath);
        }
    } catch (error) {
        if (error.code !== 'ENOENT') {
            throw error;
        }
    }

    // 4. Create recursively
    const mode = parseInt(params.permissions || '0755', 8);
    await this.fileService.createDirectory(uri, { mode });

    return {
        success: true,
        dirPath: normalizedPath,
        created: true
    };
}
```

## Best Practices

### 1. Always Use Absolute Paths

**❌ Bad: Relative path**

```typescript
{
    dirPath: './src/services'
}
```

**✅ Good: Absolute path**

```typescript
{
    dirPath: '/Users/ming/project/src/services'
}
```

### 2. Use Appropriate Permissions

```typescript
// Public directories (755)
{
    dirPath: '/var/www/html/public',
    permissions: '0755'
}

// Private directories (700)
{
    dirPath: '/home/user/.secrets',
    permissions: '0700'
}

// Shared directories (775)
{
    dirPath: '/var/www/shared',
    permissions: '0775'
}
```

### 3. Leverage Idempotency

```typescript
// Safe to call multiple times
await createDirectory({ dirPath: '/project/dist' });
await createDirectory({ dirPath: '/project/dist' });  // No error
```

### 4. Create Before File Operations

```typescript
// Ensure directory exists before creating files
await createDirectory({ dirPath: '/project/output' });
await createFile({
    filePath: '/project/output/result.json',
    content: '{}'
});
```

### 5. Batch Directory Creation

```typescript
// Create multiple directories efficiently
const directories = [
    '/project/src/components',
    '/project/src/services',
    '/project/src/utils'
];

await Promise.all(
    directories.map(dir =>
        createDirectory({ dirPath: dir })
    )
);
```

## Limitations

### 1. No Bulk Operations

- Creates one directory per call
- For multiple directories, must call multiple times
- No transactional multi-directory creation

### 2. Platform Differences

- Permissions only on Unix-like systems
- Windows ignores permission parameter
- Path length limits vary by OS

### 3. No Template System

- No built-in directory templates
- Must manually create structure
- No batch scaffold operations

### 4. No Ownership Control

- Cannot set owner/group
- Uses current process user
- Would need elevated privileges for different owner

### 5. Race Conditions

- Multiple processes creating same directory
- Generally safe due to idempotency
- But check-then-create is not atomic

## Related Tools

- **createFile**: For creating files
- **replaceString**: For modifying files
- **insertEdit**: For adding to files
- **multiReplaceString**: For multiple modifications

## Version History

- **v1.0**: Initial implementation
- **v1.1**: Added idempotent behavior
- **v1.2**: Added permission support
- **v1.3**: Added verification step
- **v2.0**: Complete rewrite with better error handling

## References

- File Service: `src/platform/files/`
- Tool Framework: `src/extension/tools/`
- Validation Service: `src/platform/validation/`
- Node.js fs: `https://nodejs.org/api/fs.html`
