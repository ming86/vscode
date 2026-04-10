# 05 — Testing Strategy

## Test Categories

### 1. Unit Tests (Vitest / Mocha)

Unit tests for the tool logic in isolation, mocking VS Code APIs.

#### Setup

```typescript
// test/readImageTool.test.ts
import { describe, it, expect, vi, beforeEach } from 'vitest';
import * as vscode from 'vscode';

// Mock vscode module
vi.mock('vscode', () => ({
    Uri: {
        file: (p: string) => ({ fsPath: p, scheme: 'file' }),
        joinPath: (base: any, ...segments: string[]) => ({
            fsPath: require('path').join(base.fsPath, ...segments),
            scheme: 'file',
        }),
    },
    workspace: {
        fs: {
            readFile: vi.fn(),
            stat: vi.fn(),
        },
        workspaceFolders: [{ uri: { fsPath: '/workspace', scheme: 'file' } }],
    },
    FileType: { File: 1, Directory: 2 },
    LanguageModelToolResult: class {
        constructor(public content: any[]) {}
    },
    LanguageModelTextPart: class {
        constructor(public value: string) {}
    },
    LanguageModelDataPart: {
        image: (data: Uint8Array, mime: string) => ({ data, mimeType: mime }),
    },
    CancellationTokenSource: class {
        token = { isCancellationRequested: false };
    },
}));
```

#### Test Cases

```typescript
describe('ReadImageTool', () => {
    let tool: ReadImageTool;
    let token: vscode.CancellationToken;

    beforeEach(() => {
        tool = new ReadImageTool();
        token = new vscode.CancellationTokenSource().token;
    });

    describe('invoke', () => {
        it('reads a PNG file and returns an image data part', async () => {
            const imageData = new Uint8Array([0x89, 0x50, 0x4E, 0x47]); // PNG header
            vi.mocked(vscode.workspace.fs.stat).mockResolvedValue({
                type: vscode.FileType.File,
                size: imageData.length,
                ctime: 0,
                mtime: 0,
            });
            vi.mocked(vscode.workspace.fs.readFile).mockResolvedValue(imageData);

            const result = await tool.invoke(
                { input: { filePath: '/workspace/test.png' } } as any,
                token
            );

            expect(result.content).toHaveLength(1);
            expect(result.content[0]).toEqual({
                data: imageData,
                mimeType: 'image/png',
            });
        });

        it('rejects unsupported file extensions', async () => {
            const result = await tool.invoke(
                { input: { filePath: '/workspace/doc.pdf' } } as any,
                token
            );

            expect(result.content).toHaveLength(1);
            expect(result.content[0].value).toContain('Unsupported image format');
        });

        it('rejects files exceeding size limit', async () => {
            vi.mocked(vscode.workspace.fs.stat).mockResolvedValue({
                type: vscode.FileType.File,
                size: 6 * 1024 * 1024, // 6 MB
                ctime: 0,
                mtime: 0,
            });

            const result = await tool.invoke(
                { input: { filePath: '/workspace/huge.png' } } as any,
                token
            );

            expect(result.content[0].value).toContain('File too large');
        });

        it('handles file-not-found gracefully', async () => {
            vi.mocked(vscode.workspace.fs.stat).mockRejectedValue(
                new Error('ENOENT')
            );

            const result = await tool.invoke(
                { input: { filePath: '/workspace/missing.png' } } as any,
                token
            );

            expect(result.content[0].value).toContain('File not found');
        });

        it('handles directories', async () => {
            vi.mocked(vscode.workspace.fs.stat).mockResolvedValue({
                type: vscode.FileType.Directory,
                size: 0,
                ctime: 0,
                mtime: 0,
            });

            const result = await tool.invoke(
                { input: { filePath: '/workspace/images/' } } as any,
                token
            );

            expect(result.content[0].value).toContain('not a file');
        });

        it('respects cancellation before I/O', async () => {
            const cancelledToken = { isCancellationRequested: true } as vscode.CancellationToken;
            vi.mocked(vscode.workspace.fs.stat).mockResolvedValue({
                type: vscode.FileType.File,
                size: 100,
                ctime: 0,
                mtime: 0,
            });

            const result = await tool.invoke(
                { input: { filePath: '/workspace/test.png' } } as any,
                cancelledToken
            );

            expect(result.content[0].value).toContain('cancelled');
            expect(vscode.workspace.fs.readFile).not.toHaveBeenCalled();
        });

        it.each([
            ['.png', 'image/png'],
            ['.jpg', 'image/jpeg'],
            ['.jpeg', 'image/jpeg'],
            ['.gif', 'image/gif'],
            ['.webp', 'image/webp'],
            ['.bmp', 'image/bmp'],
        ])('maps %s to %s', async (ext, expectedMime) => {
            vi.mocked(vscode.workspace.fs.stat).mockResolvedValue({
                type: vscode.FileType.File,
                size: 10,
                ctime: 0,
                mtime: 0,
            });
            vi.mocked(vscode.workspace.fs.readFile).mockResolvedValue(
                new Uint8Array([1, 2, 3])
            );

            const result = await tool.invoke(
                { input: { filePath: `/test${ext}` } } as any,
                token
            );

            expect(result.content[0].mimeType).toBe(expectedMime);
        });

        it('handles case-insensitive extensions', async () => {
            vi.mocked(vscode.workspace.fs.stat).mockResolvedValue({
                type: vscode.FileType.File,
                size: 10,
                ctime: 0,
                mtime: 0,
            });
            vi.mocked(vscode.workspace.fs.readFile).mockResolvedValue(
                new Uint8Array([1])
            );

            const result = await tool.invoke(
                { input: { filePath: '/test.PNG' } } as any,
                token
            );

            expect(result.content[0].mimeType).toBe('image/png');
        });
    });

    describe('prepareInvocation', () => {
        it('returns invocation message with filename', () => {
            const result = tool.prepareInvocation(
                { input: { filePath: '/workspace/screenshots/error.png' } } as any,
                token
            );

            expect(result).toEqual({
                invocationMessage: 'Reading image: error.png',
                pastTenseMessage: 'Read image: error.png',
            });
        });
    });
});
```

### 2. Integration Tests (VS Code Extension Host)

These tests run inside the VS Code extension host and exercise the real `workspace.fs` API.

```typescript
// test/integration/readImage.test.ts
import * as vscode from 'vscode';
import * as assert from 'assert';
import * as path from 'path';
import * as fs from 'fs';

suite('ReadImageTool Integration', () => {
    const fixturesDir = path.join(__dirname, 'fixtures');

    suiteSetup(async () => {
        // Create test fixtures
        if (!fs.existsSync(fixturesDir)) {
            fs.mkdirSync(fixturesDir, { recursive: true });
        }

        // Create a minimal valid PNG (1x1 pixel, red)
        const minimalPng = Buffer.from(
            'iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAADUlEQVR42mP8/5+hHgAHggJ/PchI7wAAAABJRU5ErkJggg==',
            'base64'
        );
        fs.writeFileSync(path.join(fixturesDir, 'test.png'), minimalPng);
    });

    test('tool is registered and invocable', async () => {
        const tools = vscode.lm.tools;
        const readImageTool = tools.find(t => t.name === 'readImage');
        assert.ok(readImageTool, 'readImage tool should be registered');
    });

    test('tool has correct metadata', async () => {
        const tools = vscode.lm.tools;
        const readImageTool = tools.find(t => t.name === 'readImage');
        assert.ok(readImageTool);
        assert.strictEqual(readImageTool.name, 'readImage');
        assert.ok(readImageTool.description?.includes('image'));
        assert.ok(readImageTool.inputSchema);
    });
});
```

### 3. Manual Testing Checklist

| # | Test | Expected Result |
|---|------|-----------------|
| 1 | Install extension, open workspace with images | Tool appears in `vscode.lm.tools` |
| 2 | In Copilot Chat, type `#readImage` | Tool is suggested/referenced |
| 3 | Ask Copilot: "What does the screenshot at ./screenshot.png show?" | Model invokes readImage, describes the image |
| 4 | Ask Copilot to compare two images | Model invokes readImage twice, compares |
| 5 | Reference a non-existent image path | Model receives error text, informs user |
| 6 | Reference a text file with `.png` extension | Model receives garbled/empty image, reports it |
| 7 | Reference a 6 MB image | Model receives "File too large" error text |
| 8 | Reference a `.svg` file | Model receives "Unsupported format" error text |
| 9 | In Agent mode, give task requiring image inspection | Model autonomously invokes readImage |
| 10 | Test on SSH remote workspace | Tool works, reads image from remote |
| 11 | Test with non-English filename (e.g., `截图.png`) | Path handled correctly |
| 12 | Test with model that lacks vision | Image may be silently dropped; model should still respond |

### 4. Smoke Test Script

A quick validation script to run after building:

```bash
#!/bin/bash
# smoke-test.sh — run from extension root

echo "=== Build ==="
npm run compile || exit 1

echo "=== Package ==="
npx vsce package --no-dependencies || exit 1

echo "=== Verify package.json tool contribution ==="
node -e "
const pkg = require('./package.json');
const tools = pkg.contributes.languageModelTools;
if (!tools || !tools.find(t => t.name === 'readImage')) {
    console.error('ERROR: readImage tool not found in package.json');
    process.exit(1);
}
console.log('OK: readImage tool found in package.json');
"

echo "=== Done ==="
```
