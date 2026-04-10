# 02 — Package.json Manifest

## Source Location

The actual manifest is at: `/Users/ming/GitRepos/github-ming86/lmtools-read-image/package.json`

## Extension Manifest

The complete `package.json` for the extension. Adjust publisher, name, and version as needed.

```jsonc
{
    "name": "read-image-lmtool",
    "displayName": "Read Image Tool for Copilot",
    "description": "Enables Copilot to read and analyze image files from the workspace during agentic loops.",
    "version": "0.1.0",
    "publisher": "your-publisher-id",
    "license": "MIT",
    "engines": {
        "vscode": "^1.100.0"
    },
    "categories": ["AI"],
    "keywords": ["copilot", "image", "vision", "tool", "lm"],
    "activationEvents": [],
    "main": "./out/extension.js",
    "contributes": {
        "languageModelTools": [
            {
                "name": "readImage",
                "toolReferenceName": "readImage",
                "displayName": "Read Image",
                "icon": "$(file-media)",
                "userDescription": "Read an image file from the workspace and return it for visual analysis.",
                "modelDescription": "Read an image file from the workspace and return its visual content for analysis. Supports PNG, JPEG, GIF, WebP, and BMP formats. Use this tool when you need to examine the visual contents of an image file — screenshots, diagrams, charts, UI mockups, photos, or any other image. The image will be returned as visual content that you can describe and reason about.\n\nDo NOT use this tool for non-image files. Use the readFile tool for text-based files instead.\n\nMaximum file size: 5 MB.",
                "tags": ["image", "vision", "file"],
                "inputSchema": {
                    "type": "object",
                    "properties": {
                        "filePath": {
                            "type": "string",
                            "description": "The absolute path to the image file to read. Must be an image file with one of these extensions: .png, .jpg, .jpeg, .gif, .webp, .bmp"
                        }
                    },
                    "required": ["filePath"]
                }
            }
        ]
    },
    "scripts": {
        "vscode:prepublish": "npm run compile",
        "compile": "tsc -p ./",
        "watch": "tsc -watch -p ./"
    },
    "devDependencies": {
        "@types/vscode": "^1.100.0",
        "@types/node": "^20.0.0",
        "typescript": "^5.7.0"
    }
}
```

## Key Decisions in the Manifest

### Tool Name: `readImage`

The `name` field is the programmatic identifier. It must match the first argument to `vscode.lm.registerTool()` exactly. Internal Copilot tools use a `copilot_` prefix; 3rd-party extensions should use their own namespace or a plain descriptive name.

### `modelDescription` — The Most Critical Field

The `modelDescription` is the primary signal the LLM uses to decide whether to call the tool. It must:

1. **Clearly state the capability** — "Read an image file and return its visual content for analysis."
2. **List supported formats** — helps the model self-filter.
3. **Describe use cases** — "screenshots, diagrams, charts, UI mockups" gives the model retrieval cues.
4. **State anti-patterns** — "Do NOT use this for non-image files" prevents misuse.
5. **State constraints** — "Maximum file size: 5 MB" sets expectations.

Per the official VS Code tool API guide:
> "The tool description is critical because it's the primary way the language model identifies and selects the right tool."

### `inputSchema`

The schema is JSON Schema draft-07 compatible. VS Code validates tool input against it before passing to `invoke()`. A single `filePath` string parameter is sufficient — the model generates the path from context.

### `toolReferenceName`

Enables users to type `#readImage` in chat to reference the tool explicitly. Without this, the tool is only available through the model's autonomous selection.

### `icon`

`$(file-media)` is a built-in Codicon representing media files. Displayed in the chat UI when the tool is invoked.

### `tags`

Tags help categorize the tool but are not currently used for tool selection by the model. They may be used by future VS Code features for filtering.

### `when` Clause (Optional)

No `when` clause is specified — the tool is always available. If desired, you could restrict availability:

```jsonc
// Only when at least one workspace folder is open
"when": "workspaceFolderCount > 0"
```

### Minimum VS Code Version

`^1.100.0` ensures all stable APIs referenced (`lm.registerTool`, `LanguageModelDataPart.image`) are available. Adjust based on when these APIs shipped.

## `tsconfig.json`

```jsonc
{
    "compilerOptions": {
        "module": "commonjs",
        "target": "ES2022",
        "lib": ["ES2022"],
        "outDir": "out",
        "rootDir": "src",
        "sourceMap": true,
        "strict": true,
        "esModuleInterop": true,
        "skipLibCheck": true
    },
    "include": ["src/**/*"],
    "exclude": ["node_modules"]
}
```
