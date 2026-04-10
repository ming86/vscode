# Language Model Tools Documentation Index

## Overview

This directory contains comprehensive technical documentation for all Language Model Tools in the GitHub Copilot Chat extension. Each tool is documented with architectural details, implementation specifics, usage examples, and blueprints for reimplementation.

## Documentation Structure

All tool documentation follows a consistent structure:

1. **Overview** - Purpose, capabilities, and primary file location
2. **Tool Declaration** - JSON schema from package.json
3. **Input Parameters** - TypeScript interfaces with detailed descriptions
4. **Architecture** - Flow diagrams and system design
5. **Internal Implementation** - Code snippets and algorithms
6. **Dependencies** - Required services and external integrations
7. **Performance Characteristics** - Time/space complexity and benchmarks
8. **Use Cases** - Real-world examples with sample inputs/outputs
9. **Comparison with Other Tools** - When to use each tool
10. **Error Handling** - Common scenarios and recovery strategies
11. **Configuration** - Settings and options
12. **Implementation Blueprint** - Step-by-step guide for reimplementation
13. **Best Practices** - Recommendations and patterns
14. **Limitations** - Known constraints and workarounds

## Tool Categories

### System Architecture

#### [00-architecture-overview.md](./00-architecture-overview.md)

Comprehensive overview of the languageModelTools system architecture including:

- Tool registration and contribution system
- Dependency injection infrastructure
- Name mapping (internal vs external names)
- Invocation flow and lifecycle
- Type definitions and interfaces
- Tool categories and organization

**Start here** to understand how tools are registered, invoked, and integrated into the extension.

---

### Search & Discovery Tools

#### [01-copilot-searchCodebase.md](./01-copilot-searchCodebase.md) - Semantic Code Search

**Tool**: `copilot_searchCodebase`

**Purpose**: Semantic search across workspace using embeddings-based indexing

**Key Features**:

- Multi-strategy search (full workspace, code search, local embeddings, TF-IDF)
- Query enhancement with meta-prompts
- GitHub Code Search integration
- Incremental local indexing
- 8s timeout with fallbacks

**When to Use**: Finding code related to concepts, not exact symbol names

---

#### [02-copilot-searchWorkspaceSymbols.md](./02-copilot-searchWorkspaceSymbols.md) - Symbol Search

**Tool**: `copilot_searchWorkspaceSymbols`

**Purpose**: Fast symbol search using language service providers

**Key Features**:

- Language service integration (TypeScript, Python, etc.)
- CamelCase and fuzzy matching
- Pre-indexed symbol database
- Sub-100ms response time

**When to Use**: Finding specific classes, functions, variables by name

---

#### [03-copilot-listCodeUsages.md](./03-copilot-listCodeUsages.md) - Find References

**Tool**: `copilot_listCodeUsages`

**Purpose**: Find all usages, definitions, and implementations of symbols

**Key Features**:

- Parallel queries for definitions, references, implementations
- Smart symbol location with file path hints
- Cross-file reference tracking
- Categorization by usage type

**When to Use**: Understanding how a symbol is used throughout the codebase

---

#### [04-copilot-findFiles.md](./04-copilot-findFiles.md) - File Pattern Search

**Tool**: `copilot_findFiles`

**Purpose**: Glob pattern-based file discovery

**Key Features**:

- Full glob syntax support (`**/*.ts`, `{a,b}`, `[abc]`)
- 20s timeout protection
- Respects .copilotignore
- Pattern normalization

**When to Use**: Finding files by name patterns or extensions

---

#### [05-copilot-findTextInFiles.md](./05-copilot-findTextInFiles.md) - Text Search

**Tool**: `copilot_findTextInFiles`

**Purpose**: Full-text and regex search across workspace

**Key Features**:

- Regex and literal search modes
- Automatic fallback between modes
- includePattern filtering
- Context preview with matches

**When to Use**: Finding specific text, error messages, or patterns in code

---

### File Operations

#### [06-copilot-readFile.md](./06-copilot-readFile.md) - Read File Contents

**Tool**: `copilot_readFile`

**Purpose**: Read file contents with line range support

**Key Features**:

- Pagination with offset/limit
- 2000-line hard limit per read
- Encoding auto-detection
- Notebook format support

**When to Use**: Reading specific sections of files or paginating large files

---

#### [07-copilot-listDirectory.md](./07-copilot-listDirectory.md) - List Directory

**Tool**: `copilot_listDirectory`

**Purpose**: List directory contents

**Key Features**:

- Single-level directory listing
- File vs directory differentiation
- Workspace boundary enforcement
- Empty directory detection

**When to Use**: Exploring project structure or verifying file existence

---

### File Editing Tools

#### [08-copilot-insertEdit.md](./08-copilot-insertEdit.md) - Insert Code Edits

**Tool**: `copilot_insertEdit`

**Purpose**: AI-guided code insertion with minimal hints

**Key Features**:

- Smart code mapping with "...existing code..." hints
- Context-based positioning
- Automatic indentation adjustment
- Pre/post diagnostics comparison

**When to Use**: Adding new code sections with AI guidance

---

#### [09-copilot-replaceString.md](./09-copilot-replaceString.md) - Replace Text

**Tool**: `copilot_replaceString`

**Purpose**: Exact string replacement with context matching

**Key Features**:

- 4-strategy matching (exact, whitespace-flexible, fuzzy, similarity)
- Uniqueness validation
- Self-healing with LLM correction
- Indentation transformation

**When to Use**: Precise edits when you know exact text to replace

---

#### [10-copilot-multiReplaceString.md](./10-copilot-multiReplaceString.md) - Multiple Replacements

**Tool**: `copilot_multiReplaceString`

**Purpose**: Apply multiple replacements in one operation

**Key Features**:

- Conflict detection
- Partial success handling
- Edit merging per file
- No transaction/rollback

**When to Use**: Making multiple related edits across files

---

#### [13-copilot-applyPatch.md](./13-copilot-applyPatch.md) - Apply Diff Patches

**Tool**: `copilot_applyPatch`

**Purpose**: Apply custom V4A diff format patches

**Key Features**:

- V4A unified diff format
- 6-pass progressive matching
- Context-based chunk positioning
- Self-healing with LLM

**When to Use**: Complex multi-file changes with contextual positioning

---

### File Creation Tools

#### [11-copilot-createFile.md](./11-copilot-createFile.md) - Create New File

**Tool**: `copilot_createFile`

**Purpose**: Create new files with content

**Key Features**:

- Automatic directory creation
- Overwrite protection
- Encoding detection
- Notebook support

**When to Use**: Creating new files from scratch

---

#### [12-copilot-createDirectory.md](./12-copilot-createDirectory.md) - Create Directory

**Tool**: `copilot_createDirectory`

**Purpose**: Create directories recursively

**Key Features**:

- mkdir -p behavior
- Idempotent operation
- Cross-platform compatibility

**When to Use**: Setting up directory structures

---

### Diagnostics & Git Tools

#### [14-copilot-getErrors.md](./14-copilot-getErrors.md) - Get Compile Errors

**Tool**: `copilot_getErrors`

**Purpose**: Retrieve compiler and linter diagnostics

**Key Features**:

- Language service integration
- Severity filtering (Error, Warning)
- File/directory filtering
- Notebook cell error handling

**When to Use**: Analyzing compilation errors before/after edits

---

#### [15-copilot-getChangedFiles.md](./15-copilot-getChangedFiles.md) - Get Git Changes

**Tool**: `copilot_getChangedFiles`

**Purpose**: Get git diffs of current changes

**Key Features**:

- Staged/unstaged/conflict filtering
- Unified diff generation
- Multi-repository support
- .copilotignore integration

**When to Use**: Reviewing changes before commit or understanding modifications

---

### Specialized Tools

#### [16-copilot-memory.md](./16-copilot-memory.md) - Persistent Memory

**Tool**: `copilot_memory`

**Purpose**: Store and retrieve persistent information across conversations

**Key Features**:

- 6 commands: view, create, str_replace, insert, delete, rename
- File-based storage in workspace
- Security with path validation
- Session isolation

**When to Use**: Remembering user preferences, project context, or task state

---

#### [17-copilot-getVSCodeAPI.md](./17-copilot-getVSCodeAPI.md) - VS Code API Docs

**Tool**: `copilot_getVSCodeAPI`

**Purpose**: Query VS Code API documentation

**Key Features**:

- Semantic search over API docs
- Proposed vs stable API differentiation
- Code examples and patterns
- Extension development guidance

**When to Use**: Developing VS Code extensions or using VS Code APIs

---

#### [18-copilot-githubRepo.md](./18-copilot-githubRepo.md) - GitHub Repo Search

**Tool**: `copilot_githubRepo`

**Purpose**: Search public GitHub repositories

**Key Features**:

- GitHub Code Search API integration
- Authentication with GitHub token
- Repository index management
- Rate limiting (30 searches/min)

**When to Use**: Finding code examples from specific GitHub repositories

---

#### [19-copilot-fetchWebPage.md](./19-copilot-fetchWebPage.md) - Fetch Web Content

**Tool**: `copilot_fetchWebPage`

**Purpose**: Fetch and extract web page content

**Key Features**:

- Two-layer architecture (extraction + chunking)
- Reader mode content extraction
- Query-based filtering
- Domain approval security

**When to Use**: Getting documentation from websites or API references

---

### Agent Tools

#### [20-runSubagent.md](./20-runSubagent.md) - Run Subagent

**Tool**: `runSubagent`

**Purpose**: Launch a new agent to handle complex, multi-step tasks autonomously

**Key Features**:

- Isolated subagent context for task delegation
- Stateless execution with single response
- Custom agent/mode selection support
- Context window management
- Recursive tool prevention (subagent cannot spawn subagents)

**When to Use**: Delegating complex research, multi-file searches, or autonomous multi-step tasks

---

## Tool Selection Guide

### When to Use Which Search Tool?

| Need | Tool | Reason |
|------|------|--------|
| Find code by concept | `searchCodebase` | Semantic understanding |
| Find specific symbol | `searchWorkspaceSymbols` | Fast, name-based |
| Find symbol usages | `listCodeUsages` | References tracking |
| Find files by pattern | `findFiles` | Glob matching |
| Find specific text | `findTextInFiles` | Text/regex search |
| Explore directory | `listDirectory` | File listing |

### When to Use Which Edit Tool?

| Need | Tool | Reason |
|------|------|--------|
| Add new code section | `insertEdit` | AI-guided insertion |
| Replace known text | `replaceString` | Exact matching |
| Multiple changes | `multiReplaceString` | Batch operations |
| Complex diff | `applyPatch` | Contextual patching |
| Create new file | `createFile` | File creation |
| Create directories | `createDirectory` | Directory setup |

### When to Use Which Diagnostic Tool?

| Need | Tool | Reason |
|------|------|--------|
| Check compilation errors | `getErrors` | Language diagnostics |
| Review git changes | `getChangedFiles` | Diff viewing |
| Remember context | `memory` | Persistent storage |

### When to Use Which External Tool?

| Need | Tool | Reason |
|------|------|--------|
| VS Code API help | `getVSCodeAPI` | Extension development |
| GitHub code search | `githubRepo` | Public code examples |
| Fetch documentation | `fetchWebPage` | Web content |

### When to Use Agent Tools?

| Need | Tool | Reason |
|------|------|--------|
| Complex multi-step task | `runSubagent` | Autonomous execution |
| Research delegation | `runSubagent` | Context isolation |
| Large codebase search | `runSubagent` | Context window management |

## Implementation Patterns

### Common Patterns Across Tools

1. **Service-Oriented Architecture**
   - Dependency injection via `@IServiceName` decorators
   - Services abstract VS Code APIs

2. **Prompt-TSX Rendering**
   - Output formatted via `renderPromptElementJSON`
   - Custom `PromptElement` components

3. **Security**
   - Workspace boundary enforcement
   - `.copilotignore` integration
   - Path validation

4. **Error Handling**
   - Cancellation token support
   - Graceful degradation
   - Telemetry tracking

5. **Performance**
   - Lazy initialization
   - Parallel operations where possible
   - Timeout protection
   - Result caching

### Tool Lifecycle

1. **provideInput** - Auto-detect parameters from context (optional)
2. **resolveInput** - Transform/enrich parameters (optional)
3. **prepareInvocation** - Validation and confirmation (optional)
4. **invoke** - Main execution
5. **Result** - Return `LanguageModelToolResult`

## Development Guide

### Adding a New Tool

1. Create tool class implementing `ICopilotTool<TParams>`
2. Define parameter interface
3. Register with `ToolRegistry.registerTool(ToolClass)`
4. Add declaration to `package.json` `contributes.languageModelTools`
5. Map internal name to external name in `names.ts`
6. Implement `invoke()` method
7. Add tests

### Testing Tools

- Unit tests: Mock service dependencies
- Integration tests: Test with real VS Code APIs
- Simulation tests: End-to-end scenarios

### Best Practices

1. **Keep tools focused** - Single responsibility
2. **Use services** - Don't call VS Code APIs directly
3. **Handle cancellation** - Check `token.isCancellationRequested`
4. **Provide feedback** - Use `prepareInvocation` for confirmations
5. **Log telemetry** - Track success/failure rates
6. **Document well** - Add comprehensive tool descriptions

## File Organization

```
.local/docs/
├── 00-architecture-overview.md       # System architecture
├── 01-copilot-searchCodebase.md      # Semantic search
├── 02-copilot-searchWorkspaceSymbols.md  # Symbol search
├── 03-copilot-listCodeUsages.md      # Find references
├── 04-copilot-findFiles.md           # File pattern search
├── 05-copilot-findTextInFiles.md     # Text search
├── 06-copilot-readFile.md            # Read files
├── 07-copilot-listDirectory.md       # List directories
├── 08-copilot-insertEdit.md          # Insert edits
├── 09-copilot-replaceString.md       # Replace text
├── 10-copilot-multiReplaceString.md  # Multiple replacements
├── 11-copilot-createFile.md          # Create files
├── 12-copilot-createDirectory.md     # Create directories
├── 13-copilot-applyPatch.md          # Apply patches
├── 14-copilot-getErrors.md           # Get diagnostics
├── 15-copilot-getChangedFiles.md     # Get git changes
├── 16-copilot-memory.md              # Persistent memory
├── 17-copilot-getVSCodeAPI.md        # VS Code API docs
├── 18-copilot-githubRepo.md          # GitHub search
├── 19-copilot-fetchWebPage.md        # Fetch web content
├── 20-runSubagent.md                 # Run subagent
└── README.md                          # This file
```

## Additional Resources

### Source Code References

- **Tool Registry**: `src/extension/tools/toolsRegistry.ts`
- **Tool Service**: `src/extension/tools/toolsService.ts`
- **Tool Contribution**: `src/extension/tools/tools.ts`
- **Tool Names**: `src/extension/tools/names.ts`
- **Individual Tools**: `src/extension/tools/node/` and `src/extension/tools/vscode-node/`

### Key Interfaces

- `ICopilotTool<T>` - Base tool interface
- `IToolsService` - Tool management service
- `LanguageModelToolResult` - Tool result type
- `ToolName` - Internal tool name enum
- `ContributedToolName` - External tool name enum

### VS Code APIs

- `vscode.lm.registerTool()` - Register tools
- `vscode.lm.invokeTool()` - Invoke tools
- `vscode.LanguageModelTool` - Tool interface
- `vscode.LanguageModelToolResult` - Result type

## Contributing

When adding new tool documentation:

1. Follow the standard structure outlined above
2. Include comprehensive code examples
3. Provide practical use cases
4. Document error scenarios
5. Add implementation blueprints
6. Update this index with the new tool

## Version

Documentation Version: 1.0.0
Last Updated: November 2025
Extension Version: GitHub Copilot Chat (latest)

---

For questions or feedback on this documentation, please refer to the GitHub Copilot Chat extension repository.
