# Optimized Tool Descriptions for Language Models

**Document Purpose**: This file contains refined `modelDescription` and `inputSchema` descriptions optimized for LLM consumption. These descriptions prioritize clarity, decisiveness, and actionable guidance over exhaustive documentation.

**Design Principles**:

- **Concise but impactful**: No marketing fluff, no redundant phrasing
- **Decision-oriented**: Explicit "when to use" / "when NOT to use" criteria
- **Technical precision**: Surface constraints, limits, and edge cases
- **Differentiation**: Highlight unique capabilities vs. similar tools

---

## Search Tools: Supplementary Guidelines

This section provides additional context to complement the default tool descriptions in GitHub Copilot's system prompt. These guidelines clarify tool capabilities, parameter usage, and performance characteristics.

### Search Tool Categories

The workspace provides seven search tools, each optimized for different search scenarios:

1. **Semantic Search** (`#codebase`, `semantic_search`): Natural language queries against embedded code representations
2. **Symbol Lookup** (`#symbols`): Language service provider index for code identifiers
3. **Reference Tracking** (`#usages` / `list_code_usages`): LSP-based cross-reference analysis
4. **Path Matching** (`#fileSearch` / `file_search`): Glob pattern matching for file discovery
5. **Text Matching** (`#textSearch` . `grep_search`): String and regex search across file contents
6. **External Code Search** (`#githubRepo` / `github_repo`): GitHub repository code search
7. **Web Content** (`#fetch` / `fetch_webpage`): Web page content extraction

### Tool Selection Guidance

**When you have the exact symbol name:**

- `#symbols` provides indexed lookups with fuzzy matching and camelCase abbreviation support
- Returns definition locations with metadata
- Faster than semantic or text-based approaches for known identifiers

**When you need to understand code conceptually:**

- `#codebase` uses embeddings to find semantically related code
- Handles synonyms, typos, and conceptual descriptions
- Returns ranked results based on semantic similarity
- Workspace size affects behavior: small workspaces return full content, large workspaces return top matches

**When you need all references to a symbol:**

- `#usages` provides language-aware reference finding
- Distinguishes between definitions, implementations, and call sites
- Performance improves significantly when provided with `filePaths` parameter
- Recommended pattern: use `#symbols` first to get definition location, then pass to `#usages`

**When you need files matching a pattern:**

- `#fileSearch` performs glob-based path matching
- Returns paths only, no file content
- Respects `.gitignore` and workspace exclusion settings
- Useful for structure discovery before reading specific files

**When you need exact text or code patterns:**

- `#textSearch` provides fast ripgrep-based searching
- Supports both literal strings and regex patterns (specify via `isRegexp` parameter)
- Case-insensitive by default
- `includePattern` parameter significantly improves performance by limiting search scope
- Returns matches with surrounding context lines

**When you need real-world code examples:**

- `#githubRepo` searches public GitHub repositories
- Useful for learning library usage patterns or discovering implementation approaches

**When you need documentation content:**

- `#fetch` extracts content from web pages
- Returns markdown with preserved structure
- Supports semantic filtering via `query` parameter for large pages

### Parameter Usage Notes

**maxResults parameter:**

- Available in `#fileSearch`, `#textSearch`
- Default limits are typically sufficient for most use cases
- Only increase when results appear truncated
- Very large values may impact performance or cause timeouts

**includePattern parameter:**

- Available in `#textSearch`
- Accepts glob patterns to scope search
- Should be used whenever file type or location is known
- Significantly reduces search time and result noise on large workspaces

**filePaths parameter:**

- Available in `#usages`
- Optional but strongly recommended
- Should contain paths to files likely containing the symbol definition
- Can reduce search time by 50-80% compared to workspace-wide scans

**query parameter (for #fetch):**

- Performs semantic filtering on fetched content
- Useful for extracting specific sections from large documentation pages
- Returns relevance-ranked chunks when provided
- Omit to receive content in document order

**isRegexp parameter:**

- Required for `#textSearch`
- Boolean flag indicating whether `query` should be treated as regex or literal text
- Literal text search is faster when pattern matching is not needed
- Setting incorrectly may result in empty results or errors

### Tool Chaining Patterns

**Discovery → Definition → References:**

1. Use `#codebase` with conceptual query to discover relevant symbols
2. Use `#symbols` with discovered name to locate definition
3. Use `#usages` with symbol name and definition path to find all references

**Path Discovery → Content Reading:**

1. Use `#fileSearch` with glob pattern to find matching files
2. Use `#readFile` with discovered paths to read specific content

**Text Location → Detailed Reading:**

1. Use `#textSearch` to find occurrences of specific text/patterns
2. Note line numbers from results
3. Use `#readFile` with targeted line ranges for detailed context

**Documentation Research → Local Implementation:**

1. Use `#fetch` to retrieve official API documentation
2. Use `#githubRepo` to find real-world usage examples
3. Use `#codebase` or `#symbols` to locate similar patterns in local workspace

### Performance Considerations

**Search Specificity:**

- More specific tools generally execute faster than general-purpose tools
- Index-based searches (`#symbols`, `#usages`) are faster than content searches
- Providing optional hint parameters (like `filePaths`) improves performance

**Scoping Searches:**

- Using `includePattern` in `#textSearch` prevents unnecessary file scanning
- Glob patterns in `#fileSearch` should be as specific as possible
- Workspace-wide scans should be reserved for when scope is genuinely unknown

**Result Set Management:**

- Default result limits are optimized for typical use cases
- Avoid increasing `maxResults` without confirming truncation
- Large result sets consume more tokens and processing time

**Regex Optimization:**

- Use alternation (`pattern1|pattern2|pattern3`) instead of multiple searches
- Avoid overly complex patterns that may cause performance issues
- Consider whether literal text search is sufficient before using regex

### Common Patterns to Avoid

**Tool Mismatches:**

- Using `#codebase` when symbol name is already known (use `#symbols` instead)
- Using `#textSearch` for semantic searches (use `#codebase` instead)
- Using `#usages` for string searches (use `#textSearch` instead)

**Missing Context:**

- Calling `#usages` without first using `#symbols` to get `filePaths` hint
- Using `#textSearch` without `includePattern` when file types are known
- Reading entire large files when `#textSearch` could identify relevant sections

**Inefficient Batching:**

- Making multiple sequential `#textSearch` calls instead of using regex alternation
- Repeating identical searches when results could be reused

**Parameter Misuse:**

- Setting `includeIgnoredFiles=true` without specific need (searches excluded directories)
- Using empty array for optional parameters (omit parameter entirely instead)
- Providing relative paths when absolute paths are required

### Integration Guidelines

**Pre-Edit Workflow:**

- Use appropriate search tool to locate target code
- Use `#readFile` to verify current content and context
- Use `#problems` to check existing errors before modifications

**Post-Edit Validation:**

- Use `#problems` after file modifications to verify no new errors introduced
- Check specific files or entire workspace as appropriate

**Knowledge Retrieval Workflow:**

- Use `#vscodeAPI` for VS Code extension API documentation
- Use `#fetch` for general official documentation
- Use `#githubRepo` for real-world implementation examples
- Use `#codebase` to find local equivalents or related code

---

## Search Tools

### copilot_searchCodebase

**Tool Reference Name**: `codebase`

**Original modelDescription:**

```
Run a natural language search for relevant code or documentation comments from the user's current workspace. Returns relevant code snippets from the user's current workspace if it is large, or the full contents of the workspace if it is small.
```

**Optimized modelDescription:**

```
Performs semantic search across the workspace using embeddings and natural language queries to find conceptually relevant code, comments, and documentation. Returns ranked code chunks from large workspaces or full workspace content for small projects. Use when exploring unfamiliar code or finding implementations by concept rather than exact symbol names.
```

**Complementary modelDescription:**

```
Embedding-based semantic search returning code ranked by conceptual relevance. Understands intent without exact keywords—e.g., 'auth' finds authentication logic. Workspace-aware: small codebases return full content, large ones return top-10 chunks. Prefer over #textSearch for exploration; use #symbols when you know the name.
```

**Original inputSchema descriptions:**

- **query**: The query to search the codebase for. Should contain all relevant context. Should ideally be text that might appear in the codebase, such as function names, variable names, or comments.

**Optimized inputSchema descriptions:**

- **query**: Natural language query describing the code you're looking for. Include relevant context and keywords (function names, domain terms, concepts). Works best with descriptive phrases like "authentication logic" or "database connection setup" rather than single words.

**Complementary inputSchema descriptions:**

- **query**: Conceptual description of target code. Multi-word phrases work better than keywords: 'JWT token validation' > 'JWT'. Include domain terms, architectural patterns, or function purposes. Handles typos and synonyms via embeddings.

---

### copilot_searchWorkspaceSymbols

**Tool Reference Name**: `symbols`

**Original modelDescription:**

```
Search the user's workspace for code symbols using language services. Use this tool when the user is looking for a specific symbol in their workspace.
```

**Optimized modelDescription:**

```
Searches for code symbols (classes, functions, methods, variables) using language service providers and pre-built indexes. Returns symbol definitions with location metadata. Use when you know the symbol name and need to locate its declaration, NOT when searching by content or usage patterns.
```

**Complementary modelDescription:**

```
Index-based symbol search using LSP services. Sub-second response for known names. Supports fuzzy matching (typos) and camelCase abbreviations. Returns definition locations only—use #usages for references. Faster than #codebase when name is known; fails gracefully if symbol doesn't exist.
```

**Original inputSchema descriptions:**

- **symbolName**: The symbol to search for, such as a function name, class name, or variable name.

**Optimized inputSchema descriptions:**

- **symbolName**: Exact or partial symbol name to locate (e.g., "AuthService", "calculateTotal"). Supports camelCase abbreviations (e.g., "AS" matches "AuthService"). Case-insensitive with fuzzy matching.

**Complementary inputSchema descriptions:**

- **symbolName**: Symbol identifier (class/function/var name). Fuzzy-tolerant: 'Autservice' finds 'AuthService'. CamelCase shorthand: 'UC' finds 'UserController'. Partial matches ranked by relevance. Pure text, not patterns.

---

### copilot_listCodeUsages

**Tool Reference Name**: `usages`

**Original modelDescription:**

```
Request to list all usages (references, definitions, implementations etc) of a function, class, method, variable etc. Use this tool when 1. Looking for a sample implementation of an interface or class 2. Checking how a function is used throughout the codebase. 3. Including and updating all usages when changing a function, method, or constructor
```

**Optimized modelDescription:**

```
Finds all references, definitions, and implementations of a specific symbol using language service providers. Returns usage locations with surrounding code context. Use for refactoring impact analysis, finding implementation examples, or understanding how a symbol is used across the codebase. NOT for text-based searches.
```

**Complementary modelDescription:**

```
LSP-based reference finder distinguishing definitions from usages. Essential pre-refactoring: shows every call site, import, and extension. Returns code context for each match. Pair with #symbols to locate symbol first, then #usages for impact. Respects language semantics—not string matching.
```

**Original inputSchema descriptions:**

- **symbolName**: The name of the symbol, such as a function name, class name, method name, variable name, etc.
- **filePaths**: One or more file paths which likely contain the definition of the symbol. For instance the file which declares a class or function. This is optional but will speed up the invocation of this tool and improve the quality of its output.

**Optimized inputSchema descriptions:**

- **symbolName**: Name of the symbol to find usages for (e.g., "DatabaseConnection", "handleRequest"). Must be an actual code symbol, not arbitrary text.
- **filePaths**: Optional file paths likely containing the symbol's definition. Providing this significantly improves speed and accuracy by narrowing the initial search scope. Use paths from prior searches or known locations.

**Complementary inputSchema descriptions:**

- **symbolName**: Actual symbol from code, not arbitrary text. Must exist in workspace. Use output from #symbols or #codebase to get valid names.
- **filePaths**: Hint paths containing definition. Chain with #symbols: first locate, then pass path here. Cuts search time 50-80%. Array of absolute paths.

---

### copilot_findFiles

**Tool Reference Name**: `fileSearch`

**Original modelDescription:**

```
Search for files in the workspace by glob pattern. This only returns the paths of matching files. Use this tool when you know the exact filename pattern of the files you're searching for. Glob patterns match from the root of the workspace folder. Examples: - **/*.{js,ts} to match all js/ts files in the workspace. - src/** to match all files under the top-level src folder. - **/foo/**/*.js to match all js files under any foo folder in the workspace.
```

**Optimized modelDescription:**

```
Searches workspace for files matching glob patterns. Returns file paths only, no content. Use when locating files by name/extension pattern or exploring directory structure. For content-based search, use searchCodebase or findTextInFiles instead.
```

**Complementary modelDescription:**

```
Glob-based file path finder. Returns paths only—no file content. Respects .gitignore. Use for 'find all X files' or 'locate config.json'. Chain with #readFile for content. Faster than #textSearch when you know filename patterns. Recursive by default via **.
```

**Original inputSchema descriptions:**

- **query**: Search for files with names or paths matching this glob pattern.
- **maxResults**: The maximum number of results to return. Do not use this unless necessary, it can slow things down. By default, only some matches are returned. If you use this and don't see what you're looking for, you can try again with a more specific query or a larger maxResults.

**Optimized inputSchema descriptions:**

- **query**: Glob pattern to match file paths (e.g., "**/*.ts", "src/**/config.json"). Use `**` for recursive search, `*` for wildcards, `{a,b}` for alternatives. Patterns match from workspace root.
- **maxResults**: Maximum file paths to return. Omit for reasonable default. Increase only if results seem truncated. Very high values impact performance on large workspaces.

**Complementary inputSchema descriptions:**

- **query**: Glob from workspace root. `**` = any depth, `*` = wildcard, `{ts,js}` = alternates, `!` = negate. Examples: `src/**/*.test.ts`, `**/{package,tsconfig}.json`.
- **maxResults**: Omit for sensible limit (~50-100). Set when pattern too broad. No performance cost from omitting—just result truncation risk.

---

### copilot_findTextInFiles

**Tool Reference Name**: `textSearch`

**Original modelDescription:**

```
Do a fast text search in the workspace. Use this tool when you want to search with an exact string or regex. If you are not sure what words will appear in the workspace, prefer using regex patterns with alternation (|) or character classes to search for multiple potential words at once instead of making separate searches. For example, use 'function|method|procedure' to look for all of those words at once. Use includePattern to search within files matching a specific pattern, or in a specific file, using a relative path. Use this tool when you want to see an overview of a particular file, instead of using read_file many times to look for code within a file.
```

**Optimized modelDescription:**

```
Performs fast text or regex search across workspace files, returning matches with surrounding code context. Use when searching for exact strings, code patterns, or syntax elements (imports, function signatures, comments). Supports regex for flexible pattern matching. NOT for semantic understanding or symbol-based searches.
```

**Complementary modelDescription:**

```
Ripgrep-powered text/regex search. Case-insensitive, returns matches with context lines. Use for exact strings ('TODO'), patterns ('import.*from'), or multi-term searches ('error|warn|fail'). Scope via includePattern glob. Faster than #codebase for known syntax; lacks semantic understanding.
```

**Original inputSchema descriptions:**

- **query**: The pattern to search for in files in the workspace. Use regex with alternation (e.g., 'word1|word2|word3') or character classes to find multiple potential words in a single search. Be sure to set the isRegexp property properly to declare whether it's a regex or plain text pattern. Is case-insensitive.
- **isRegexp**: Whether the pattern is a regex.
- **includePattern**: Search files matching this glob pattern. Will be applied to the relative path of files within the workspace. To search recursively inside a folder, use a proper glob pattern like "src/folder/**". Do not use | in includePattern.
- **includeIgnoredFiles**: Whether to include files that would normally be ignored according to .gitignore, other ignore files and `files.exclude` and `search.exclude` settings. Warning: using this may cause the search to be slower, only set it when you want to search in ignored folders like node_modules or build outputs.
- **maxResults**: The maximum number of results to return. Do not use this unless necessary, it can slow things down. By default, only some matches are returned. If you use this and don't see what you're looking for, you can try again with a more specific query or a larger maxResults.

**Optimized inputSchema descriptions:**

- **query**: Text or regex pattern to match. For multiple terms, use regex alternation ("TODO|FIXME|HACK") in a single search. Case-insensitive by default. Optimize regex to avoid catastrophic backtracking.
- **isRegexp**: REQUIRED. Set true for regex patterns, false for literal text. Plain text searches are faster—use them when you don't need pattern matching.
- **includePattern**: Glob pattern to limit search scope (e.g., "src/**/*.ts"). Always use to improve performance when you know the file type or location. Do not use pipe (|) in glob patterns.
- **includeIgnoredFiles**: Set true ONLY when searching in node_modules, dist, or other normally excluded directories. Significantly slower. Default false respects .gitignore and exclude settings.
- **maxResults**: Omit for sensible default. Increase only if results appear truncated. High values on broad patterns can cause timeouts or memory issues.

**Complementary inputSchema descriptions:**

- **query**: Literal string or regex. Multi-word optimization: 'auth|token|jwt' > 3 searches. Avoid greedy quantifiers (.*) without bounds. Case-insensitive by engine.
- **isRegexp**: Boolean flag. false = treat query as literal (faster, escapes special chars). true = compile as regex. Mismatch causes empty results or errors.
- **includePattern**: Glob scoping filter. Always use when file type/location known. Examples: `src/**/*.{ts,tsx}`, `**/test/**`. Major performance impact on large repos.
- **includeIgnoredFiles**: false = respect .gitignore (default, faster). true = search all files including node_modules, dist, .git (slow, use rarely).
- **maxResults**: Default ~200 sufficient. Increase to 500-1000 if suspecting truncation. 10000+ risks timeout on broad patterns.

---

## File Operations

### copilot_readFile

**Tool Reference Name**: `readFile`

**Original modelDescription:**

```
Read the contents of a file. Line numbers are 1-indexed. This tool will truncate its output at 2000 lines and may be called repeatedly with offset and limit parameters to read larger files in chunks.
```

**Optimized modelDescription:**

```
Reads complete file contents with line numbers for code analysis, configuration review, and documentation access. Returns up to 2000 lines per call; use offset/limit parameters for chunked reading of larger files. Fastest way to access known file paths directly.
```

**Complementary modelDescription:**

```
Direct file reader with line-number annotations. Requires startLine/endLine—read exact range needed to conserve tokens. For 2000+ line files, call multiple times with different ranges. Fastest tool when path known; use #fileSearch or #codebase to discover paths first.
```

**Original inputSchema descriptions:**

- **filePath**: The absolute path of the file to read.
- **offset**: Optional: the 1-based line number to start reading from. Only use this if the file is too large to read at once. If not specified, the file will be read from the beginning.
- **limit**: Optional: the maximum number of lines to read. Only use this together with `offset` if the file is too large to read at once.

**Optimized inputSchema descriptions:**

- **filePath**: Absolute file path (e.g., `/workspace/src/app.ts`). Relative paths resolved to workspace root.
- **startLine**: 1-indexed starting line for reading. Required—specify exact line range needed.
- **endLine**: 1-indexed ending line (inclusive). Required—can equal startLine for single line.

**Complementary inputSchema descriptions:**

- **filePath**: Absolute path required. Get from #fileSearch, #codebase, or #symbols results. Relative paths attempt workspace root resolution but may fail.
- **startLine**: 1-based line start. startLine=1 for file beginning. Use #textSearch results to identify ranges before reading.
- **endLine**: 1-based inclusive end. endLine=startLine reads single line. Ranges >2000 lines truncate—split into multiple calls.

---

### copilot_listDirectory

**Tool Reference Name**: `listDirectory`

**Original modelDescription:**

```
List the contents of a directory. Result will have the name of the child. If the name ends in /, it's a folder, otherwise a file
```

**Optimized modelDescription:**

```
Lists immediate children of a directory, identifying files and subdirectories (marked with `/` suffix). Returns sorted entries (directories first, then alphabetical). Use for workspace navigation and structure exploration. Non-recursive—lists single level only.
```

**Complementary modelDescription:**

```
Non-recursive directory listing. Returns names only with `/` suffix for folders. Sorted: folders first alphabetically, then files. Use for step-by-step navigation ('show me src/', then 'show me src/components/'). For recursive structure, use #fileSearch with glob.
```

**Original inputSchema descriptions:**

- **path**: The absolute path to the directory to list.

**Optimized inputSchema descriptions:**

- **path**: Absolute directory path (e.g., `/workspace/src`). Relative paths resolved to workspace root. Must be a directory, not a file.

**Complementary inputSchema descriptions:**

- **path**: Absolute directory path. Fails if path is file not folder. Start with workspace root, then drill down. Trailing slashes ignored.

---

## Editing Tools

### copilot_insertEdit

**Tool Reference Name**: `insertEdit`

**Original modelDescription:**

```
Insert new code into an existing file in the workspace. Use this tool once per file that needs to be modified, even if there are multiple changes for a file. Generate the "explanation" property first. The system is very smart and can understand how to apply your edits to the files, you just need to provide minimal hints. Avoid repeating existing code, instead use comments to represent regions of unchanged code. Be as concise as possible.
```

**Optimized modelDescription:**

```
Inserts code after a semantic anchor point using AI-powered location matching. Best for adding methods, imports, or config entries. Not for replacing existing code—use replaceString instead. Requires 2-5 lines of unique context to locate insertion point. Auto-detects indentation.
```

**Complementary modelDescription:**

```
AI-guided insertion using semantic anchors. Handles indentation automatically. Use ellipsis comments (// ...existing code...) to indicate unchanged regions—concise diffs preferred. Best for additive changes; use #replaceString for modifications. Smart enough to place code in correct scope from minimal context.
```

**Original inputSchema descriptions:**

- **explanation**: A short explanation of the edit being made.
- **filePath**: An absolute path to the file to edit.
- **code**: The code change to apply to the file.

**Optimized inputSchema descriptions:**

- **filePath**: Absolute path to target file (must exist).
- **locationToInsertAfter**: Exact code snippet marking insertion point. Tool inserts AFTER this text. Include 2-5 lines of unique context for reliable matching. Whitespace-flexible matching handles indentation differences.
- **codeToInsert**: Code to insert with proper indentation. Include leading/trailing newlines as needed. Indentation will be adjusted to match surrounding code.
- **explanation**: Single-sentence description of what's being added and why (e.g., 'Add getConfig method to ConfigManager class').

**Complementary inputSchema descriptions:**

- **explanation**: Brief intent statement. Required first for context.
- **filePath**: Absolute path. File must exist—use #createFile for new files.
- **code**: Diff-style with ellipsis comments. Example: `class Foo {\n  // ...existing code...\n  newMethod() {}\n}`. AI infers placement from structure. Minimal viable diff preferred.

---

### copilot_replaceString

**Tool Reference Name**: `replaceString`

**Original modelDescription:**

```
This is a tool for making edits in an existing file in the workspace. For moving or renaming files, use run in terminal tool with the 'mv' command instead. For larger edits, split them into smaller edits and call the edit tool multiple times to ensure accuracy. Before editing, always ensure you have the context to understand the file's contents and context.
```

**Optimized modelDescription:**

```
Replaces existing code using 4-tier matching (exact → whitespace-flexible → fuzzy → semantic). Requires 3-5 lines of context around target. Handles whitespace variations automatically. For single targeted changes only—use multiReplaceString for coordinated edits or applyPatch for complex refactorings.
```

**Complementary modelDescription:**

```
Precision replacement with progressive fuzzy matching. Replaces exactly one occurrence—fails if oldString matches 0 or 2+ times. Include enough context for uniqueness. Whitespace-tolerant (tabs↔spaces OK). For atomic multi-file edits use #multiReplaceString; for 50+ changes use #applyPatch.
```

**Original inputSchema descriptions:**

- **filePath**: An absolute path to the file to edit.
- **oldString**: The exact literal text to replace, preferably unescaped. For single replacements (default), include at least 3 lines of context BEFORE and AFTER the target text, matching whitespace and indentation precisely.
- **newString**: The exact literal text to replace `old_string` with, preferably unescaped. Provide the EXACT text. Ensure the resulting code is correct and idiomatic.

**Optimized inputSchema descriptions:**

- **filePath**: Absolute path to target file (must exist).
- **oldString**: Exact literal text to replace. MUST include 3-5 lines of surrounding context to ensure unique match. Copy directly from file—do not paraphrase. Must match exactly once in file or operation fails.
- **newString**: Exact replacement text. Match indentation style of surrounding code. Can be different length than oldString. Provide complete replacement—no placeholders.

**Complementary inputSchema descriptions:**

- **filePath**: Absolute path. Use #readFile first to verify current content.
- **oldString**: Literal text with 3-5 lines context before/after target. Must be unique in file. Copy from #readFile output—don't retype. Whitespace flex: tabs/spaces normalized.
- **newString**: Complete replacement. No ellipsis or placeholders. Can be empty string to delete. Preserve language syntax and indentation style.

---

### copilot_multiReplaceString

**Tool Reference Name**: `multiReplaceString`

**Original modelDescription:**

```
This tool allows you to apply multiple replace_string_in_file operations in a single call, which is more efficient than calling replace_string_in_file multiple times. It takes an array of replacement operations and applies them sequentially.
```

**Optimized modelDescription:**

```
Applies 2-50 coordinated replacements atomically with automatic conflict detection and rollback. All operations succeed together or all fail. Detects overlapping regions and resolves dependencies. For single changes use replaceString instead. For 50+ operations or complex diffs use applyPatch.
```

**Complementary modelDescription:**

```
Batch version of #replaceString with atomic commit. All succeed or all rollback—no partial state. Processes in reverse-offset order to prevent position shifts. Detects overlaps/conflicts pre-execution. Use for refactorings touching 2-50 locations. Single change? Use #replaceString. Mega-refactor? Use #applyPatch.
```

**Original inputSchema descriptions:**

- **explanation**: A brief explanation of what the multi-replace operation will accomplish.
- **replacements**: An array of replacement operations to apply sequentially. Each requires: filePath (absolute), oldString, newString, explanation.

**Optimized inputSchema descriptions:**

- **explanation**: Overall goal of the batch operation (e.g., 'Refactor UserService to use dependency injection'). Describes why these changes are coordinated.
- **replacements**: Array of 1-50 replacement operations. Each requires: filePath (absolute), oldString (3-5 lines context, must be unique in file), newString (complete replacement), explanation (what this specific change does). Operations applied in reverse-offset order to prevent position invalidation.

**Complementary inputSchema descriptions:**

- **explanation**: High-level refactoring goal. Ties together why these N changes are a unit.
- **replacements**: Array of 2-50 operations (min 1, optimal 2-20). Each needs: filePath (absolute), oldString (unique match with context), newString (complete), explanation (specific change). Engine sorts by file offset to avoid invalidation.

---

### copilot_createFile

**Tool Reference Name**: `createFile`

**Original modelDescription:**

```
This is a tool for creating a new file in the workspace. The file will be created with the specified content. The directory will be created if it does not already exist. Never use this tool to edit a file that already exists.
```

**Optimized modelDescription:**

```
Creates new file with overwrite protection and automatic parent directory creation (mkdir -p). Default fails if file exists unless overwrite=true. Atomic write ensures data safety. Max size 10MB. Use for new files only—use replaceString to modify existing files.
```

**Complementary modelDescription:**

```
Atomic file creation with mkdir -p behavior for parent dirs. Fails if file exists (safety default). Max 10MB content. Validates on write—syntax errors only caught at compile time. For existing files use #replaceString or #insertEdit, never this.
```

**Original inputSchema descriptions:**

- **filePath**: The absolute path to the file to create.
- **content**: The content to write to the file.

**Optimized inputSchema descriptions:**

- **filePath**: Absolute path for new file. Parent directories created automatically if missing. Must include file extension.
- **content**: Complete file content as string. Max 10MB. No validation—ensure syntax is correct before creation.
- **overwrite**: Set true to replace existing file. Default false for safety—operation fails if file exists. Automatic backup created when overwriting.

**Complementary inputSchema descriptions:**

- **filePath**: Absolute path with extension. Non-existent parent dirs created automatically. Use workspace root + relative for portability.
- **content**: Full file text, any encoding. No size warning until 10MB. Pre-validate syntax yourself—tool doesn't parse content.

---

### copilot_createDirectory

**Tool Reference Name**: `createDirectory`

**Original modelDescription:**

```
Create a new directory structure in the workspace. Will recursively create all directories in the path, like mkdir -p. You do not need to use this tool before using create_file, that tool will automatically create the needed directories.
```

**Optimized modelDescription:**

```
Creates directory with recursive parent creation (mkdir -p behavior). Idempotent—succeeds if directory already exists. Sets permissions on Unix systems (default 0755). Use before file creation to ensure output paths exist. Single directory per call—use multiple calls for complex structures.
```

**Complementary modelDescription:**

```
Recursive directory creation (mkdir -p). Idempotent: no error if exists. Unix permissions default 0755 (rwxr-xr-x). Unnecessary before #createFile which auto-creates parents. Use for explicit structure creation or when setting non-default permissions.
```

**Original inputSchema descriptions:**

- **dirPath**: The absolute path to the directory to create.

**Optimized inputSchema descriptions:**

- **dirPath**: Absolute path for directory to create. All missing parent directories created automatically. Trailing slashes ignored.
- **permissions**: Unix file permissions as octal string (e.g., '0755' for rwxr-xr-x). Default '0755'. Ignored on Windows. Format must include leading zero.

**Complementary inputSchema descriptions:**

- **dirPath**: Absolute path. Trailing `/` optional. Creates full chain: `/a/b/c/d` makes all four if none exist.

---

### copilot_applyPatch

**Tool Reference Name**: `applyPatch`

**Original modelDescription:**

```
Edit text files. Do not use this tool to edit Jupyter notebooks. `apply_patch` allows you to execute a diff/patch against a text file, but the format of the diff specification is unique to this task. Use V4A diff format with *** Begin Patch and *** End Patch markers.
```

**Optimized modelDescription:**

```
Applies V4A unified diff patches using 6-pass healing (exact → whitespace → fuzzy-1/2/3 → semantic). Handles code changes since patch generation via progressive fuzzy matching. For git diffs, generated refactorings, or 100+ coordinated changes. Use replaceString for simple edits or multiReplaceString for structured operations instead.
```

**Complementary modelDescription:**

```
Unified diff engine with self-healing fuzzy matching. Tolerates code drift between patch generation and application. 6 passes from exact to semantic. Use for: git diffs, LLM-generated patches, bulk refactorings (50+). Overkill for <10 changes—use #replaceString or #multiReplaceString instead.
```

**Original inputSchema descriptions:**

- **input**: The edit patch to apply.
- **explanation**: A short description of what the tool call is aiming to achieve.

**Optimized inputSchema descriptions:**

- **input**: V4A unified diff format with file headers (--- a/path, +++ b/path) and hunks (@@ -line,count +line,count @@). Include 3-5 lines of context per hunk for reliable matching. Use git diff -U3 or similar to generate.
- **explanation**: Overall goal of patch (e.g., 'Refactor authentication system to use JWT tokens'). Optional but recommended for complex patches.

**Complementary inputSchema descriptions:**

- **input**: V4A diff: file headers `*** Update File: /path`, then hunk markers `@@anchor`, then `-`/`+` lines. Context lines unmarked. Generate via git diff or construct manually.
- **explanation**: Patch intent summary. Optional but aids debugging when hunks fail.

---

## Diagnostics Tools

### copilot_getErrors

**Tool Reference Name**: `problems`

**Original modelDescription:**

```
Get any compile or lint errors in a specific file or across all files. If the user mentions errors or problems in a file, they may be referring to these. Use the tool to see the same errors that the user is seeing. If the user asks you to analyze all errors, or does not specify a file, use this tool to gather errors for all files. Also use this tool after editing a file to validate the change.
```

**Optimized modelDescription:**

```
Retrieve compile-time and lint errors from language services. Critical for pre-validation before code modifications and post-edit verification. Omit filePaths to scan entire workspace; provide specific paths to scope diagnostics. Use when users mention "errors" or "problems," and after every file edit to validate changes.
```

**Complementary modelDescription:**

```
LSP diagnostic aggregator for compile/lint errors. Real-time view of what user sees in Problems panel. Always call post-edit for validation. Omit filePaths for workspace-wide scan; specify paths to scope. Returns severity, message, location. Essential quality gate before declaring work complete.
```

**Original inputSchema descriptions:**

- **filePaths**: The absolute paths to the files or folders to check for errors. Omit 'filePaths' when retrieving all errors.

**Optimized inputSchema descriptions:**

- **filePaths**: Absolute paths to files/folders for error checking. Omit entirely for workspace-wide scan. Supports folders to recursively check all contained files. Empty array is invalid—omit parameter instead.

**Complementary inputSchema descriptions:**

- **filePaths**: Optional array of absolute paths (files or folders). Omit completely for all-workspace scan. Empty array invalid syntax—omit param. Folder paths check recursively.

---

### copilot_getChangedFiles

**Tool Reference Name**: `changes`

**Original modelDescription:**

```
Get the list of changed files in the git working tree. Returns files that have been modified, added, deleted, or have conflicts. Can filter by change type (staged, unstaged, conflicts).
```

**Optimized modelDescription:**

```
Access git working tree status via VS Code's Git extension. Essential for pre-commit reviews, conflict resolution, and change organization. Filters by staged/unstaged/conflicts. Returns file paths with git status metadata. Fails if Git extension unavailable or no repository found.
```

**Complementary modelDescription:**

```
Git working tree status reader via VS Code Git extension. Shows what user sees in Source Control view. Filter by staged, unstaged, or conflicts. Returns paths + status flags. Use for commit prep, review workflows, conflict identification. Requires Git extension enabled.
```

**Original inputSchema descriptions:**

- **repositoryPath**: The absolute path to the git repository to look for changes in. If not provided, the active git repository will be used.
- **sourceControlState**: The kinds of git state to filter by. Allowed values are: 'staged', 'unstaged', and 'merge-conflicts'. If not provided, all states will be included.

**Optimized inputSchema descriptions:**

- **repositoryPath**: Absolute path to git repository. Omit to use active repository. Specify when workspace contains multiple repos.
- **sourceControlState**: Array of git states to filter by. Options: 'staged' (ready for commit), 'unstaged' (modified but not staged), 'merge-conflicts' (conflicted files). Omit for all states.

**Complementary inputSchema descriptions:**

- **repositoryPath**: Absolute path to .git parent dir. Omit for active repo (usual case). Multi-repo workspaces require explicit path.
- **sourceControlState**: Filter array: `['staged']`, `['unstaged']`, `['merge-conflicts']`, or combinations. Omit for all changes. Use `['staged']` pre-commit, `['merge-conflicts']` during merge.

---

## Specialized Tools

### copilot_memory

**Tool Reference Name**: `memory`

**Original modelDescription:**

```
Persistent memory system for storing and retrieving information across conversation turns. Supports creating, reading, updating, and deleting memory entries. Use this to remember user preferences, project context, ongoing tasks, or any information that should persist across the session.
```

**Optimized modelDescription:**

```
Session-scoped persistent storage (no cross-session access, cleared on restart). Store user preferences, task progress, project conventions, or decisions. Max 100 entries/session, 100KB/entry, 5MB total. Use for long-term facts vs. chat history's short-term context. Six operations: view, create, str_replace, insert, delete, rename.
```

**Complementary modelDescription:**

```
Key-value memory persisting across turns in current session. Cleared on restart—not for cross-session data. Store coding preferences, architecture decisions, naming conventions, task state. 100 entries max, 100KB each, 5MB total. Commands: view (list/read), create (new), str_replace (edit), insert (add line), delete (remove), rename (change key).
```

**Original inputSchema descriptions:**

- **command**: The memory operation to perform (view, create, str_replace, insert, delete, rename).
- **key**: Unique identifier for the memory entry (required for all commands except view).
- **content**: The content to store (required for create, used in str_replace and insert).
- **oldString**: String to replace in str_replace command.
- **newString**: Replacement string in str_replace command.

**Optimized inputSchema descriptions:**

- **command**: Operation type. view=list all or get one entry, create=new entry (fails if exists), str_replace=find-and-replace first occurrence, insert=add line at position, delete=remove entry, rename=change key name.
- **path**: Path to memory file (e.g., /memories/coding_preferences.md). Required for most commands except view-all.
- **file_text**: Content for create command. Structure with headings/sections for maintainability.
- **old_str**: Exact text to find in str_replace. Only replaces first match. Must exist or fails.
- **new_str**: Replacement text for str_replace. Can be empty to delete.
- **insert_line**: 0-indexed line position for insert. 0 = before first line.
- **insert_text**: Text to insert at specified line.
- **old_path**: Current path for rename command.
- **new_path**: New path for rename command. Must not already exist.

**Complementary inputSchema descriptions:**

- **command**: Enum. view (no path = list all, path = read one), create (new entry, error if exists), str_replace (find-replace once), insert (add line at index), delete (remove), rename (move).
- **path**: Memory path like `/memories/prefs.md`. Namespace isolated from workspace files. Required except view-all.
- **file_text**: Full content for create. Markdown recommended for structure.
- **old_str**: Exact substring for str_replace. First match only. No regex.
- **new_str**: Replacement for str_replace. Empty string = delete text.
- **insert_line**: 0-based. 0 = before line 1. insert_line=2 after 2nd line.
- **insert_text**: Text to splice in at insert_line.
- **old_path/new_path**: For rename. new_path must not exist.

---

### copilot_getVSCodeAPI

**Tool Reference Name**: `vscodeAPI`

**Original modelDescription:**

```
Get comprehensive VS Code API documentation and references for extension development. This tool provides authoritative documentation for VS Code's extensive API surface, including proposed APIs, contribution points, and best practices. Use this tool for understanding complex VS Code API interactions.
```

**Optimized modelDescription:**

```
Curated VS Code extension development knowledge base with API docs, patterns, and examples. Use for extension dev questions, API signatures, best practices, troubleshooting. Returns 5 results with descriptions, parameters, examples, related APIs. Query by API name (e.g., 'vscode.window.showQuickPick'), concept ('tree view'), or question ('how to create webview'). Do not use for general TypeScript/Node.js—scope is VS Code API only.
```

**Complementary modelDescription:**

```
VS Code extension API reference and patterns. Semantic search over official docs, proposed APIs, contribution points. Returns 5 results: signatures, examples, related APIs, best practices. Use for 'how do I X in VS Code extension' queries. Scope: VS Code API only—not TypeScript/Node.js/general programming.
```

**Original inputSchema descriptions:**

- **query**: The query to search vscode documentation for. Should contain all relevant context.

**Optimized inputSchema descriptions:**

- **query**: API name ('vscode.commands.registerCommand'), conceptual query ('language server integration'), or question ('how to create tree view'). Use 'vscode.' prefix for exact API lookups. Be specific to avoid broad results.

**Complementary inputSchema descriptions:**

- **query**: API name (e.g., `vscode.window.createTreeView`), concept ('webview panel lifecycle'), or question ('how to register command'). Prefix `vscode.` for exact API lookup. Specific terms beat vague queries.

---

### copilot_githubRepo

**Tool Reference Name**: `githubRepo`

**Original modelDescription:**

```
Searches a GitHub repository for relevant source code snippets. Only use this tool if the user is very clearly asking for code snippets from a specific GitHub repository. Do not use this tool for Github repos that the user has open in their workspace.
```

**Optimized modelDescription:**

```
GitHub code search (requires authentication, 30 searches/minute limit). Find real-world implementations, API usage examples, design patterns from public repos. Returns code snippets with metadata. Filter by language, repo, org, filename. Use for learning library usage, discovering patterns, seeing production code. Do not use for authoritative docs—prefer getVSCodeAPI or fetchWebPage. Rate limit shared across session.
```

**Complementary modelDescription:**

```
Public GitHub code search. Find real-world usage examples, patterns, implementations across repos. 30 req/min rate limit. Use for 'show me how X library is used' or 'find implementations of Y pattern'. Not for docs—use #fetch for official docs. Requires GitHub auth. Returns snippets with repo context.
```

**Original inputSchema descriptions:**

- **repo**: The name of the Github repository to search for code in. Should must be formatted as '<owner>/<repo>'.
- **query**: The query to search for repo. Should contain all relevant context.

**Optimized inputSchema descriptions:**

- **repo**: GitHub repository in 'owner/repo' format (e.g., 'microsoft/vscode'). Must be public repository.
- **query**: Natural language or code snippet to search for. Include relevant keywords and context. More specific queries yield better results.

**Complementary inputSchema descriptions:**

- **repo**: Format `owner/repo` (e.g., `facebook/react`). Public repos only. Private repos fail even if you have access.
- **query**: Code or natural language. Examples: 'useEffect cleanup', 'async error handling', 'class ExampleService'. Specific > vague.

---

### copilot_fetchWebPage

**Tool Reference Name**: `fetch`

**Original modelDescription:**

```
Fetches the main content from a web page. This tool is useful for summarizing or analyzing the content of a webpage. You should use this tool when you think the user is looking for information from a specific webpage.
```

**Optimized modelDescription:**

```
Extract main content from web pages on domain whitelist (docs sites, GitHub, Stack Overflow). Returns markdown with structure preserved. Intelligent chunking splits large pages—use 'query' parameter to retrieve only relevant sections. Max 10MB per page, 30s timeout, 10 requests/minute. Cached 1 hour. Domain restrictions enforced for security—no arbitrary sites. Use for documentation, tutorials, API refs, release notes. Cannot execute JavaScript or access auth-required pages.
```

**Complementary modelDescription:**

```
Whitelisted web scraper for docs/tutorials. Returns markdown with preserved structure. Whitelist: VS Code docs, MDN, GitHub, npm, Stack Overflow, common doc sites. Chunks large pages—use query param for semantic filtering. 10MB/page, 30s timeout, 10 req/min, 1h cache. Static content only—no JS execution, no auth pages.
```

**Original inputSchema descriptions:**

- **urls**: An array of URLs to fetch content from.
- **query**: The query to search for in the web page's content. This should be a clear and concise description of the content you want to find.

**Optimized inputSchema descriptions:**

- **urls**: Array of HTTP/HTTPS URLs to fetch. Must be on whitelist (VS Code docs, MDN, GitHub, npm, etc.). Validate domain before calling—non-whitelisted domains fail immediately.
- **query**: Semantic search within page content. Retrieves only sections matching query, sorted by relevance. Use for large docs to avoid irrelevant chunks (e.g., 'authentication' on API docs, 'installation' on README). Omit to get top N chunks by document order.

**Complementary inputSchema descriptions:**

- **urls**: Array of whitelisted URLs. Check domain first—non-whitelisted fail fast. Batch multiple pages from same domain to share rate limit efficiently.
- **query**: Semantic filter for page chunking. Example: 'installation steps' extracts only install sections from long README. Omit for full page or when page is short. Relevance-ranked results.

---

## Tool Selection Decision Matrix

### Search Operations

| Scenario | Tool |
|----------|------|
| Find code by concept/description | `searchCodebase` |
| Locate specific symbol definition | `searchWorkspaceSymbols` |
| Find all usages/references of symbol | `listCodeUsages` |
| Locate files by name pattern | `findFiles` |
| Find exact text/regex in files | `findTextInFiles` |
| Search GitHub repositories | `githubRepo` |

### File Operations

| Scenario | Tool |
|----------|------|
| Read known file path | `readFile` |
| Explore directory structure | `listDirectory` |
| Find files by pattern | `findFiles` |

### Code Editing

| Scenario | Tool |
|----------|------|
| Add new code (method/import) | `insertEdit` |
| Replace existing code (single edit) | `replaceString` |
| Multiple coordinated edits (2-50) | `multiReplaceString` |
| Complex refactoring (50+) | `applyPatch` |
| Create new file | `createFile` |
| Create directory structure | `createDirectory` |

### Validation & Diagnostics

| Scenario | Tool |
|----------|------|
| Check compilation/lint errors | `getErrors` |
| Review git changes | `getChangedFiles` |
| Validate after edits | `getErrors` |

### Knowledge Retrieval

| Scenario | Tool |
|----------|------|
| VS Code API documentation | `getVSCodeAPI` |
| External documentation | `fetchWebPage` |
| Real-world code examples | `githubRepo` |
| Session-persistent memory | `memory` |

---

## Common Anti-Patterns to Avoid

### Search Anti-Patterns

❌ Using `searchCodebase` when you know exact symbol name → Use `searchWorkspaceSymbols`
❌ Using `findTextInFiles` for semantic search → Use `searchCodebase`
❌ Using `listCodeUsages` for text search → Use `findTextInFiles`
❌ Not providing `filePaths` hint to `listCodeUsages` when known → Always provide for better performance

### Edit Anti-Patterns

❌ Using `insertEdit` to replace existing code → Use `replaceString`
❌ Using `replaceString` without 3-5 lines context → Always include sufficient context
❌ Using `createFile` to modify existing files → Use editing tools instead
❌ Multiple sequential `replaceString` calls → Use `multiReplaceString` for atomicity
❌ Using `applyPatch` for simple edits → Reserve for complex multi-file refactorings

### Parameter Anti-Patterns

❌ Setting `maxResults` to high values "just in case" → Use default, increase only if needed
❌ Setting `includeIgnoredFiles=true` by default → Only use when specifically searching ignored paths
❌ Omitting `explanation` parameters → Always provide for clarity and debugging
❌ Using relative paths when absolute required → Most tools require absolute paths

---

## Performance Optimization Guidelines

### Search Optimization

1. **Use most specific tool**: `searchWorkspaceSymbols` faster than `searchCodebase` for known symbols
2. **Provide hints**: `filePaths` in `listCodeUsages` dramatically improves speed
3. **Scope searches**: Use `includePattern` in `findTextInFiles` to limit search area
4. **Avoid broad patterns**: Specific queries return faster and more relevant results

### Edit Optimization

1. **Batch operations**: Use `multiReplaceString` instead of sequential `replaceString` calls
2. **Right-size context**: 3-5 lines sufficient; excessive context increases token usage
3. **Validate before edit**: Use `readFile` to confirm context before editing
4. **Post-edit validation**: Always call `getErrors` after edits

### Memory Management

1. **Chunk large files**: Use `offset`/`limit` in `readFile` for files >2000 lines
2. **Limit result sets**: Only increase `maxResults` when truncation confirmed
3. **Cache awareness**: `fetchWebPage` caches 1 hour—reuse results
4. **Rate limits**: `githubRepo` has 30/minute limit—batch queries when possible

---

## Version History

- **v1.0**: Initial optimized descriptions based on comprehensive tool documentation audit
