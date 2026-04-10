# GitHub Copilot Tool: createFile

## Overview

The `createFile` tool provides safe, atomic file creation with built-in overwrite protection, automatic directory creation, and encoding management. It ensures data integrity through validation and prevents accidental data loss through explicit overwrite controls.

**Key Capabilities:**

- Atomic file creation with safety checks
- Automatic parent directory creation (mkdir -p behavior)
- Overwrite protection with explicit control
- Multiple encoding support (UTF-8, ASCII, etc.)
- File permission management
- Template support for common file types

**Primary Use Cases:**

- Creating new source files
- Generating configuration files
- Creating test files
- Scaffolding project structures
- Creating documentation files

## Tool Declaration

```typescript
interface CreateFileTool {
    name: 'create_file';
    description: 'Create a new file with specified content';
    parameters: {
        filePath: string;           // Absolute path for the new file
        content: string;            // File content
        explanation?: string;       // Optional description
        overwrite?: boolean;        // Allow overwriting existing file
        encoding?: string;          // File encoding (default: utf-8)
    }
}
```

## Input Parameters

### filePath

- **Type:** `string`
- **Required:** Yes
- **Format:** Absolute file path
- **Validation:** Must be valid path, not a directory
- **Example:** `/Users/ming/project/src/services/newService.ts`

### content

- **Type:** `string`
- **Required:** Yes
- **Format:** Raw file content
- **Size Limit:** 10 MB (default, configurable)
- **Encoding:** UTF-8 by default
- **Example:**

```typescript
export class NewService {
    constructor() {}

    async performAction(): Promise<void> {
        // Implementation
    }
}
```

### explanation

- **Type:** `string`
- **Required:** No
- **Purpose:** User-facing description
- **Example:** "Create new service for handling user authentication"

### overwrite

- **Type:** `boolean`
- **Required:** No
- **Default:** `false`
- **Purpose:** Allow overwriting existing file
- **Safety:** When false, will fail if file exists

### encoding

- **Type:** `string`
- **Required:** No
- **Default:** `"utf-8"`
- **Supported Values:** `"utf-8"`, `"ascii"`, `"utf-16le"`, `"utf-16be"`, `"latin1"`
- **Example:** `"utf-8"`

## Architecture

### File Creation Flow

```
┌─────────────────────────────────────────────────────────────────┐
│                      createFile Tool Entry                       │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│                     Phase 1: Validation                          │
│  • Validate filePath format                                     │
│  • Check path is not a directory                                │
│  • Validate content size                                        │
│  • Validate encoding                                            │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│                  Phase 2: Existence Check                        │
│  • Check if file already exists                                 │
│  • If exists and overwrite=false → ERROR                        │
│  • If exists and overwrite=true → PROCEED                       │
└─────────────────────────────────────────────────────────────────┘
                              ↓
                      File exists?
                  ┌─────────┴─────────┐
                 YES                   NO
                  ↓                     ↓
        Overwrite enabled?           ┌─────────────────────────────┐
         ┌──────┴──────┐             │  Phase 3: Directory Check   │
        YES            NO             │  • Check parent directory    │
         ↓              ↓             │  • If missing → CREATE       │
┌──────────────┐  ┌─────────────┐    │  • Use mkdir -p behavior    │
│   PROCEED    │  │    ERROR    │    └─────────────────────────────┘
└──────────────┘  │  File exists│                 ↓
         ↓         └─────────────┘    ┌─────────────────────────────┐
         └───────────────┬────────────│  Phase 4: Permission Check  │
                         ↓            │  • Verify write permissions  │
┌─────────────────────────────────────────────────────────────────┐
│                  Phase 5: Encoding Preparation                   │
│  • Convert content to specified encoding                        │
│  • Validate encoding compatibility                              │
│  • Handle BOM if needed                                         │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│                  Phase 6: Atomic Write                           │
│  • Write to temporary file                                      │
│  • Set file permissions                                         │
│  • Atomic rename to target path                                 │
│  • Verify write success                                         │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│                  Phase 7: Post-Creation                          │
│  • Trigger file watchers                                        │
│  • Update workspace index                                       │
│  • Log telemetry                                                │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│                    Success Response                              │
│  • Return created file path                                     │
│  • Return file metadata                                         │
│  • Return success status                                        │
└─────────────────────────────────────────────────────────────────┘
```

### Directory Creation Flow (mkdir -p)

```
Target: /Users/ming/project/src/services/api/newService.ts

Step 1: Parse Path
──────────────────
directories = ['/Users', '/Users/ming', '/Users/ming/project',
               '/Users/ming/project/src', '/Users/ming/project/src/services',
               '/Users/ming/project/src/services/api']
file = 'newService.ts'

Step 2: Check Each Directory
─────────────────────────────
For each directory in path:
    if exists:
        verify it's a directory (not a file)
        verify write permissions
    else:
        create directory
        set permissions to 0755

Step 3: Create Missing Directories
───────────────────────────────────
/Users               → exists ✓
/Users/ming          → exists ✓
/Users/ming/project  → exists ✓
/Users/ming/project/src → exists ✓
/Users/ming/project/src/services → exists ✓
/Users/ming/project/src/services/api → MISSING → CREATE

Step 4: Verify Path Ready
──────────────────────────
All parent directories exist and are writable
Ready to create file
```

## Internal Implementation

### Core Algorithm

```typescript
async function createFile(params: CreateFileParams): Promise<CreateFileResult> {
    // Phase 1: Validation
    validateFilePath(params.filePath);
    validateContent(params.content);
    validateEncoding(params.encoding);

    // Phase 2: Existence check
    const exists = await fileExists(params.filePath);
    if (exists && !params.overwrite) {
        throw new FileExistsError(
            `File already exists: ${params.filePath}`,
            { allowOverwrite: false }
        );
    }

    // Phase 3: Directory creation
    const parentDir = path.dirname(params.filePath);
    await ensureDirectory(parentDir);

    // Phase 4: Permission check
    await checkWritePermissions(parentDir);

    // Phase 5: Encoding
    const encoding = params.encoding || 'utf-8';
    const buffer = Buffer.from(params.content, encoding);

    // Phase 6: Atomic write
    await atomicWrite(params.filePath, buffer);

    // Phase 7: Post-creation
    await notifyFileWatchers(params.filePath);
    await updateWorkspaceIndex(params.filePath);

    return {
        success: true,
        filePath: params.filePath,
        size: buffer.length,
        encoding: encoding
    };
}
```

### Atomic Write Implementation

```typescript
async function atomicWrite(filePath: string, content: Buffer): Promise<void> {
    const tempPath = `${filePath}.tmp-${Date.now()}-${Math.random()}`;

    try {
        // Step 1: Write to temporary file
        await fs.promises.writeFile(tempPath, content, { flag: 'wx' });

        // Step 2: Set permissions (0644 = rw-r--r--)
        await fs.promises.chmod(tempPath, 0o644);

        // Step 3: Atomic rename (OS-level atomic operation)
        await fs.promises.rename(tempPath, filePath);

    } catch (error) {
        // Clean up temp file if something failed
        try {
            await fs.promises.unlink(tempPath);
        } catch {
            // Ignore cleanup errors
        }
        throw error;
    }
}
```

### Directory Recursion (mkdir -p)

```typescript
async function ensureDirectory(dirPath: string): Promise<void> {
    // Normalize path
    const normalized = path.resolve(dirPath);

    // Check if already exists
    try {
        const stats = await fs.promises.stat(normalized);
        if (!stats.isDirectory()) {
            throw new Error(`Path exists but is not a directory: ${normalized}`);
        }
        return; // Directory exists, we're done
    } catch (error) {
        if (error.code !== 'ENOENT') {
            throw error; // Some other error (permission, etc.)
        }
        // Directory doesn't exist, continue
    }

    // Recursively ensure parent exists
    const parent = path.dirname(normalized);
    if (parent !== normalized) { // Prevent infinite loop at root
        await ensureDirectory(parent);
    }

    // Create this directory
    try {
        await fs.promises.mkdir(normalized, { mode: 0o755 });
    } catch (error) {
        // Race condition: directory created between stat and mkdir
        if (error.code === 'EEXIST') {
            return;
        }
        throw error;
    }
}
```

### Validation Functions

```typescript
function validateFilePath(filePath: string): void {
    if (!filePath || typeof filePath !== 'string') {
        throw new ValidationError('filePath must be a non-empty string');
    }

    if (!path.isAbsolute(filePath)) {
        throw new ValidationError('filePath must be an absolute path');
    }

    // Check for invalid characters (OS-specific)
    const invalidChars = process.platform === 'win32'
        ? /[<>:"|?*\x00-\x1F]/
        : /[\x00]/;

    if (invalidChars.test(filePath)) {
        throw new ValidationError('filePath contains invalid characters');
    }

    // Check path length (OS limits)
    const maxLength = process.platform === 'win32' ? 260 : 4096;
    if (filePath.length > maxLength) {
        throw new ValidationError(`filePath exceeds maximum length of ${maxLength}`);
    }
}

function validateContent(content: string): void {
    if (content === undefined || content === null) {
        throw new ValidationError('content cannot be null or undefined');
    }

    if (typeof content !== 'string') {
        throw new ValidationError('content must be a string');
    }

    // Check size limit (default 10 MB)
    const sizeInBytes = Buffer.byteLength(content, 'utf-8');
    const maxSize = 10 * 1024 * 1024; // 10 MB

    if (sizeInBytes > maxSize) {
        throw new ValidationError(
            `content size (${sizeInBytes} bytes) exceeds maximum (${maxSize} bytes)`
        );
    }
}

function validateEncoding(encoding?: string): void {
    if (!encoding) return;

    const supportedEncodings = [
        'utf-8', 'utf8',
        'ascii',
        'utf-16le', 'utf16le',
        'utf-16be', 'utf16be',
        'latin1'
    ];

    if (!supportedEncodings.includes(encoding.toLowerCase())) {
        throw new ValidationError(
            `Unsupported encoding: ${encoding}. ` +
            `Supported: ${supportedEncodings.join(', ')}`
        );
    }
}
```

### Overwrite Protection

```typescript
async function checkOverwrite(
    filePath: string,
    allowOverwrite: boolean
): Promise<void> {
    const exists = await fileExists(filePath);

    if (!exists) {
        return; // File doesn't exist, safe to create
    }

    if (!allowOverwrite) {
        // Get file info for error message
        const stats = await fs.promises.stat(filePath);
        throw new FileExistsError(
            `File already exists: ${filePath}`,
            {
                filePath,
                size: stats.size,
                modified: stats.mtime,
                suggestion: 'Set overwrite=true to replace existing file or choose a different path'
            }
        );
    }

    // Overwrite allowed, but let's back up first
    await createBackup(filePath);
}

async function createBackup(filePath: string): Promise<void> {
    const backupPath = `${filePath}.backup-${Date.now()}`;
    try {
        await fs.promises.copyFile(filePath, backupPath);
    } catch (error) {
        console.warn(`Could not create backup: ${error.message}`);
        // Continue anyway if backup fails
    }
}
```

## Dependencies

### Internal Services

- **IFileService**: Core file system operations
- **IWorkspaceService**: Workspace file indexing
- **IValidationService**: Input validation
- **ITelemetryService**: Usage tracking

### External Libraries

- **fs/promises**: Node.js file system (Node environment)
- **vscode.workspace.fs**: VS Code file system API (Web environment)
- **path**: Path manipulation utilities

### VS Code APIs

- `vscode.workspace.fs.writeFile`: File writing in web environment
- `vscode.workspace.onDidCreateFiles`: File creation events
- `vscode.FileSystem`: File system abstraction

## Performance Characteristics

### Time Complexity

```
Operation                Time Complexity
─────────────────────────────────────────
Path validation         O(1)
Content validation      O(n) where n = content length
Directory creation      O(d) where d = directory depth
File existence check    O(1) - OS syscall
Atomic write            O(n) where n = content length
Total                   O(n + d)
```

### Performance Benchmarks

```
Content Size    Dir Depth    Creation Time
────────────────────────────────────────────
< 1 KB         1-3          5-10 ms
10 KB          1-3          10-20 ms
100 KB         1-3          20-50 ms
1 MB           1-3          100-200 ms
10 MB          1-3          1-2 sec

< 1 KB         10           15-25 ms
< 1 KB         20           25-40 ms
```

### Memory Usage

```
Component               Memory Usage
─────────────────────────────────────
Content buffer          1x content size
Temporary buffer        1x content size
Overhead                ~10 KB
Peak total:             2x content size + 10 KB
```

## Use Cases

### Use Case 1: Create TypeScript Service

```typescript
{
    filePath: '/project/src/services/emailService.ts',
    content: `import { Injectable } from '@nestjs/common';
import { ConfigService } from '@nestjs/config';

@Injectable()
export class EmailService {
    constructor(private configService: ConfigService) {}

    async sendEmail(to: string, subject: string, body: string): Promise<void> {
        const smtpConfig = this.configService.get('smtp');
        // Implementation
    }
}
`,
    explanation: 'Create email service for sending notifications'
}
```

### Use Case 2: Create Configuration File

```typescript
{
    filePath: '/project/config/database.json',
    content: JSON.stringify({
        host: 'localhost',
        port: 5432,
        database: 'myapp',
        pool: {
            min: 2,
            max: 10
        }
    }, null, 2),
    explanation: 'Create database configuration file'
}
```

### Use Case 3: Create Test File

```typescript
{
    filePath: '/project/test/services/emailService.test.ts',
    content: `import { Test } from '@nestjs/testing';
import { EmailService } from '../../src/services/emailService';

describe('EmailService', () => {
    let service: EmailService;

    beforeEach(async () => {
        const module = await Test.createTestingModule({
            providers: [EmailService],
        }).compile();

        service = module.get<EmailService>(EmailService);
    });

    it('should be defined', () => {
        expect(service).toBeDefined();
    });

    it('should send email successfully', async () => {
        await expect(
            service.sendEmail('test@example.com', 'Test', 'Hello')
        ).resolves.not.toThrow();
    });
});
`,
    explanation: 'Create test file for email service'
}
```

### Use Case 4: Create Documentation

```typescript
{
    filePath: '/project/docs/API.md',
    content: `# API Documentation

## Endpoints

### POST /api/users
Create a new user.

**Request Body:**
\`\`\`json
{
    "name": "string",
    "email": "string"
}
\`\`\`

**Response:**
\`\`\`json
{
    "id": "string",
    "name": "string",
    "email": "string",
    "createdAt": "string"
}
\`\`\`
`,
    explanation: 'Create API documentation'
}
```

### Use Case 5: Scaffold Project Structure

```typescript
// Create multiple files for a new feature
const files = [
    {
        path: '/project/src/features/auth/auth.controller.ts',
        content: '// Controller implementation'
    },
    {
        path: '/project/src/features/auth/auth.service.ts',
        content: '// Service implementation'
    },
    {
        path: '/project/src/features/auth/auth.module.ts',
        content: '// Module definition'
    }
];

for (const file of files) {
    await createFile({
        filePath: file.path,
        content: file.content,
        explanation: `Create ${path.basename(file.path)}`
    });
}
```

## Comparison with Other Tools

### createFile vs replaceString

| Feature | createFile | replaceString |
|---------|-----------|---------------|
| **Purpose** | Create new files | Modify existing files |
| **Precondition** | File must NOT exist (unless overwrite=true) | File MUST exist |
| **Directory Handling** | Auto-creates parents | N/A |
| **Safety** | Overwrite protection | Uniqueness validation |

### createFile vs insertEdit

| Feature | createFile | insertEdit |
|---------|-----------|------------|
| **Operation** | Full file creation | Partial content addition |
| **Target** | New file | Existing file |
| **Use Case** | Scaffolding | Incremental changes |

### When to Use createFile

**✅ Use createFile when:**

- Creating a new file from scratch
- Scaffolding project structure
- Generating configuration files
- Creating test files
- Creating documentation

**❌ Don't use createFile when:**

- Modifying existing files (use replaceString)
- Adding to existing files (use insertEdit)
- File should already exist (bug in logic)

## Error Handling

### Error Scenarios

#### 1. File Already Exists

```typescript
{
    error: 'FileExistsError',
    message: 'File already exists: /path/to/file.ts',
    filePath: '/path/to/file.ts',
    fileInfo: {
        size: 1024,
        modified: '2024-01-15T10:30:00Z'
    },
    suggestion: 'Set overwrite=true to replace or choose a different path'
}
```

#### 2. Permission Denied

```typescript
{
    error: 'PermissionError',
    message: 'Cannot write to directory: permission denied',
    path: '/path/to/directory',
    permissions: 'r--r--r--',
    suggestion: 'Check directory permissions or use a different location'
}
```

#### 3. Invalid Path

```typescript
{
    error: 'ValidationError',
    message: 'filePath contains invalid characters',
    filePath: '/path/with/<invalid>.ts',
    invalidChars: ['<', '>'],
    suggestion: 'Remove invalid characters from file path'
}
```

#### 4. Content Too Large

```typescript
{
    error: 'ValidationError',
    message: 'content size exceeds maximum',
    contentSize: 15728640,
    maxSize: 10485760,
    suggestion: 'Reduce content size or increase limit in settings'
}
```

#### 5. Disk Full

```typescript
{
    error: 'DiskFullError',
    message: 'No space left on device',
    requiredSpace: 1048576,
    availableSpace: 102400,
    suggestion: 'Free up disk space and try again'
}
```

### Error Recovery

```typescript
async function createFileWithRecovery(params: CreateFileParams): Promise<void> {
    try {
        await createFile(params);
    } catch (error) {
        if (error.type === 'FileExistsError') {
            // Strategy 1: Prompt user or auto-rename
            const newPath = generateAlternativePath(params.filePath);
            console.log(`File exists, creating as: ${newPath}`);
            await createFile({ ...params, filePath: newPath });

        } else if (error.type === 'PermissionError') {
            // Strategy 2: Try alternative location
            const altPath = findWritableAlternative(params.filePath);
            if (altPath) {
                console.log(`Using alternative path: ${altPath}`);
                await createFile({ ...params, filePath: altPath });
            } else {
                throw error;
            }

        } else if (error.type === 'ValidationError' && error.field === 'contentSize') {
            // Strategy 3: Split into multiple files
            console.log('Content too large, splitting...');
            await splitAndCreateFiles(params);

        } else {
            throw error; // Unrecoverable
        }
    }
}

function generateAlternativePath(originalPath: string): string {
    const dir = path.dirname(originalPath);
    const ext = path.extname(originalPath);
    const name = path.basename(originalPath, ext);

    let counter = 1;
    let newPath = path.join(dir, `${name}_${counter}${ext}`);

    while (fs.existsSync(newPath)) {
        counter++;
        newPath = path.join(dir, `${name}_${counter}${ext}`);
    }

    return newPath;
}
```

## Configuration

### VS Code Settings

```json
{
    "copilot.tools.createFile.enabled": true,
    "copilot.tools.createFile.defaultEncoding": "utf-8",
    "copilot.tools.createFile.maxFileSize": 10485760,
    "copilot.tools.createFile.defaultPermissions": "0644",
    "copilot.tools.createFile.createBackupOnOverwrite": true,
    "copilot.tools.createFile.validateContent": true
}
```

### Configuration Options

| Setting | Type | Default | Description |
|---------|------|---------|-------------|
| `enabled` | boolean | true | Enable/disable the tool |
| `defaultEncoding` | string | "utf-8" | Default file encoding |
| `maxFileSize` | number | 10MB | Maximum file size in bytes |
| `defaultPermissions` | string | "0644" | Unix file permissions |
| `createBackupOnOverwrite` | boolean | true | Backup existing files |
| `validateContent` | boolean | true | Validate content before write |

## Implementation Blueprint

### Step 1: Tool Registration

```typescript
// src/extension/tools/createFileTool.ts
export class CreateFileTool implements Tool {
    static readonly ID = 'create_file';

    constructor(
        @IFileService private fileService: IFileService,
        @IValidationService private validator: IValidationService
    ) {}

    getDefinition(): ToolDefinition {
        return {
            name: 'create_file',
            description: 'Create a new file with content',
            parameters: {
                type: 'object',
                properties: {
                    filePath: { type: 'string' },
                    content: { type: 'string' },
                    explanation: { type: 'string' },
                    overwrite: { type: 'boolean', default: false },
                    encoding: { type: 'string', default: 'utf-8' }
                },
                required: ['filePath', 'content']
            }
        };
    }
}
```

### Step 2: File Service Extension

```typescript
// src/platform/files/fileService.ts
export class FileService implements IFileService {
    async createFile(
        path: URI,
        content: Buffer,
        options: CreateFileOptions
    ): Promise<void> {
        // Ensure parent directory exists
        await this.ensureDirectory(URI.parse(path.dirname()));

        // Atomic write
        await this.writeFileAtomic(path, content, options);
    }

    private async ensureDirectory(path: URI): Promise<void> {
        try {
            await this.stat(path);
        } catch {
            const parent = URI.parse(path.dirname());
            if (parent.toString() !== path.toString()) {
                await this.ensureDirectory(parent);
            }
            await this.createDirectory(path);
        }
    }
}
```

### Step 3: Execution Logic

```typescript
async execute(params: CreateFileParams): Promise<ToolResult> {
    // 1. Validate
    this.validator.validateFilePath(params.filePath);
    this.validator.validateContent(params.content);

    // 2. Check existence
    const uri = URI.file(params.filePath);
    const exists = await this.fileService.exists(uri);

    if (exists && !params.overwrite) {
        throw new FileExistsError(params.filePath);
    }

    // 3. Create file
    const buffer = Buffer.from(params.content, params.encoding || 'utf-8');
    await this.fileService.createFile(uri, buffer, {
        overwrite: params.overwrite || false
    });

    return {
        success: true,
        filePath: params.filePath,
        size: buffer.length
    };
}
```

## Best Practices

### 1. Always Use Absolute Paths

**❌ Bad: Relative path**

```typescript
{
    filePath: './src/newFile.ts'
}
```

**✅ Good: Absolute path**

```typescript
{
    filePath: '/Users/ming/project/src/newFile.ts'
}
```

### 2. Include Proper File Extensions

**❌ Bad: Missing extension**

```typescript
{
    filePath: '/project/src/service'
}
```

**✅ Good: Clear extension**

```typescript
{
    filePath: '/project/src/service.ts'
}
```

### 3. Validate Content Before Creating

```typescript
// Check for empty content
if (!content || content.trim().length === 0) {
    throw new Error('Content cannot be empty');
}

// Validate JSON if creating JSON file
if (filePath.endsWith('.json')) {
    try {
        JSON.parse(content);
    } catch {
        throw new Error('Invalid JSON content');
    }
}
```

### 4. Handle Encoding Appropriately

```typescript
// For text files
{
    filePath: '/project/README.md',
    content: '# My Project',
    encoding: 'utf-8'
}

// For binary-like data (use Buffer for true binary)
{
    filePath: '/project/config.bin',
    content: binaryString,
    encoding: 'latin1'
}
```

### 5. Use Overwrite Judiciously

```typescript
// Safe: Creating a new file
await createFile({
    filePath: '/project/src/newService.ts',
    content: '...',
    overwrite: false  // Default, safe
});

// Dangerous: Overwriting existing file
await createFile({
    filePath: '/project/src/existingService.ts',
    content: '...',
    overwrite: true  // Only if intentional!
});
```

## Limitations

### 1. File Size Limit

- Default maximum: 10 MB
- Large files may cause memory issues
- Consider streaming for very large files

### 2. No Template System

- Must provide complete content
- No built-in variable substitution
- No file type-specific templates

### 3. No Batch Operations

- Creates one file per call
- For multiple files, must call multiple times
- No transactional multi-file creation

### 4. Platform Differences

- Path separators (Windows vs Unix)
- File permissions (Unix only)
- Maximum path length varies by OS

### 5. No Content Validation

- Tool doesn't validate syntax
- TypeScript syntax errors won't be caught
- User responsible for valid content

## Related Tools

- **createDirectory**: For creating directories
- **replaceString**: For modifying existing files
- **insertEdit**: For adding to existing files
- **multiReplaceString**: For multiple file modifications

## Version History

- **v1.0**: Initial implementation
- **v1.1**: Added overwrite protection
- **v1.2**: Added automatic directory creation
- **v1.3**: Added atomic write support
- **v2.0**: Added backup on overwrite

## References

- File Service: `src/platform/files/`
- Tool Framework: `src/extension/tools/`
- Validation Service: `src/platform/validation/`
- VS Code File System API: `vscode.workspace.fs`
