# VS Code Search Service Investigation Report

## Executive Summary

This report documents how the VS Code search service works for finding skill files, with particular attention to Windows-specific issues and potential causes for skills not being discovered.

## Search Service Architecture

### `searchFilesInLocation` Implementation

**Location:** [promptFilesLocator.ts](../vscode/src/vs/workbench/contrib/chat/common/promptSyntax/utils/promptFilesLocator.ts#L392-L420)

```typescript
private async searchFilesInLocation(folder: URI, filePattern: string | undefined, token: CancellationToken): Promise<URI[]> {
    const disregardIgnoreFiles = this.configService.getValue<boolean>('explorer.excludeGitIgnore');
    const workspaceRoot = this.workspaceService.getWorkspaceFolder(folder);
    const getExcludePattern = (folder: URI) => getExcludes(this.configService.getValue<ISearchConfiguration>({ resource: folder })) || {};

    const searchOptions: IFileQuery = {
        folderQueries: [{ folder, disregardIgnoreFiles }],
        type: QueryType.File,
        shouldGlobMatchFilePattern: true,  // ← Key: Uses glob matching, not fuzzy
        excludePattern: workspaceRoot ? getExcludePattern(workspaceRoot.uri) : undefined,
        ignoreGlobCase: true,              // ← Case-insensitive glob matching
        sortByScore: true,
        filePattern
    };

    const searchResult = await this.searchService.fileSearch(searchOptions, token);
    return searchResult.results.map(r => r.resource);
}
```

**Key Parameters:**
- `shouldGlobMatchFilePattern: true` - Uses glob matching instead of fuzzy matching
- `ignoreGlobCase: true` - Case-insensitive glob matching on file patterns
- `filePattern: `*/SKILL.md`` - The pattern used to find skill files

### Skill Search Flow

**Location:** [promptFilesLocator.ts#L507](../vscode/src/vs/workbench/contrib/chat/common/promptSyntax/utils/promptFilesLocator.ts#L507)

```typescript
private async findAgentSkillsInFolder(uri: URI, token: CancellationToken): Promise<URI[]> {
    return await this.searchFilesInLocation(uri, `*/${SKILL_FILENAME}`, token);
}
```

The pattern `*/SKILL.md` is designed to find files named `SKILL.md` that are exactly one directory level deep inside the skills folder.

## File Search Flow

1. **Service Entry Point:** `ISearchService.fileSearch()` in [searchService.ts](../vscode/src/vs/workbench/services/search/common/searchService.ts#L131)

2. **Provider Activation:** The search service activates extensions registered for file search via `onSearch:{scheme}` event

3. **Provider Selection:** Providers are selected based on the URI scheme (e.g., `file`, `vscode-remote`)

4. **Search Execution:** For the `file` scheme, this typically goes to:
   - **Desktop:** ripgrep-based search via [fileSearch.ts](../vscode/src/vs/workbench/services/search/node/fileSearch.ts)
   - **Web:** Extension-based file search provider

5. **Result Processing:** Results flow through [fileSearchManager.ts](../vscode/src/vs/workbench/services/search/common/fileSearchManager.ts)

## Glob Pattern Processing

### Pattern Anchoring

**Location:** [ripgrepSearchUtils.ts#L11-L13](../vscode/src/vs/workbench/services/search/node/ripgrepSearchUtils.ts#L11-L13)

```typescript
export function anchorGlob(glob: string): string {
    return glob.startsWith('**') || glob.startsWith('/') ? glob : `/${glob}`;
}
```

Patterns NOT starting with `**` or `/` get anchored with `/` prefix.
- Input: `*/SKILL.md`
- Output: `/*/SKILL.md`

### Path Normalization

**Location:** [ripgrepFileSearch.ts#L92-L102](../vscode/src/vs/workbench/services/search/node/ripgrepFileSearch.ts#L92-L102)

```typescript
// glob.ts requires forward slashes, but a UNC path still must start with \\
// #38165 and #38151
if (key.startsWith('\\\\')) {
    key = '\\\\' + key.substr(2).replace(/\\/g, '/');
} else {
    key = key.replace(/\\/g, '/');
}
```

**Critical:** Backslashes are converted to forward slashes for glob matching, except for UNC paths.

### Glob Regex Pattern

**Location:** [glob.ts#L27-L29](../vscode/src/vs/base/common/glob.ts#L27-L29)

```typescript
const PATH_REGEX = '[/\\\\]';        // any slash or backslash
const NO_PATH_REGEX = '[^/\\\\]';    // any non-slash and non-backslash
```

The glob matcher explicitly handles **both** forward and backward slashes.

## Windows-Specific Considerations

### 1. Path Separator Handling

VS Code's glob implementation handles both `/` and `\` path separators:

```typescript
// From glob.ts
const PATH_REGEX = '[/\\\\]';  // Matches both / and \
```

**This is NOT a likely cause of Windows issues** - the glob engine is designed to work cross-platform.

### 2. Case Sensitivity

The `ignoreGlobCase: true` parameter ensures case-insensitive matching:

```typescript
const searchOptions: IFileQuery = {
    ignoreGlobCase: true,  // Case-insensitive on Windows
    // ...
};
```

**This setting should allow `SKILL.MD`, `skill.md`, `Skill.Md` to all match.**

### 3. Drive Letter Handling

**Location:** [ripgrepFileSearch.ts#L152-L157](../vscode/src/vs/workbench/services/search/node/ripgrepFileSearch.ts#L152-L157)

```typescript
export function fixDriveC(path: string): string {
    const root = extpath.getRoot(path);
    return root.toLowerCase() === 'c:/' ?
        path.replace(/^c:[/\\]/i, '/') :
        path;
}
```

**Note:** Drive `C:` paths are normalized to `/` for ripgrep compatibility. Other drive letters are NOT normalized, which could cause issues if skills are on D:, E:, etc.

### 4. UNC Path Support

**Location:** [ripgrepFileSearch.ts#L93-L97](../vscode/src/vs/workbench/services/search/node/ripgrepFileSearch.ts#L93-L97)

```typescript
if (key.startsWith('\\\\')) {
    key = '\\\\' + key.substr(2).replace(/\\/g, '/');
}
```

UNC paths (`\\server\share`) are preserved with initial `\\` but internal backslashes are normalized.

### 5. Exclude Patterns from Configuration

**Location:** [search.ts#L425-L438](../vscode/src/vs/workbench/services/search/common/search.ts#L425-L438)

```typescript
export function getExcludes(configuration: ISearchConfiguration, includeSearchExcludes = true): glob.IExpression | undefined {
    const fileExcludes = configuration?.files?.exclude;
    const searchExcludes = includeSearchExcludes && configuration?.search?.exclude;
    // ...merges both exclude patterns
}
```

**Potential Issue:** User's `files.exclude` or `search.exclude` settings might inadvertently exclude skill folders.

## Async/Timing Considerations

### 1. Extension Activation Dependency

**Location:** [searchService.ts#L147-L152](../vscode/src/vs/workbench/services/search/common/searchService.ts#L147-L152)

```typescript
const providerActivations: Promise<unknown>[] = [Promise.resolve(null)];
schemesInQuery.forEach(scheme =>
    providerActivations.push(this.extensionService.activateByEvent(`onSearch:${scheme}`))
);
providerActivations.push(this.extensionService.activateByEvent('onSearch:file'));

await Promise.all(providerActivations);
await this.extensionService.whenInstalledExtensionsRegistered();
```

**Potential Issue:** If the search provider extension hasn't fully activated yet, searches may fail silently or return empty results.

### 2. Folder Existence Check

**Location:** [searchService.ts#L160](../vscode/src/vs/workbench/services/search/common/searchService.ts#L160)

```typescript
const exists = await Promise.all(query.folderQueries.map(query =>
    this.fileService.exists(query.folder)
));
query.folderQueries = query.folderQueries.filter((_, i) => exists[i]);
```

**Potential Issue:** If the skills folder doesn't exist yet, it gets filtered out before search begins.

### 3. No Search Provider Available

**Location:** [searchService.ts#L197-L204](../vscode/src/vs/workbench/services/search/common/searchService.ts#L197-L204)

```typescript
if (!provider) {
    if (someSchemeHasProvider) {
        this.logService.warn(`No search provider registered for scheme: ${scheme}...`);
        return;  // ← Silent failure
    } else {
        provider = await this.waitForProvider(query.type, scheme);
    }
}
```

**Potential Issue:** If no provider is available for the file scheme, the search may hang indefinitely or fail silently.

### 4. Cancellation Token Handling

The search service properly handles cancellation tokens throughout the flow, but if a token is cancelled early (e.g., due to rapid UI interactions), results may not be returned.

## Potential Causes for Skills Not Found on Windows

### Most Likely Causes

1. **Skills folder doesn't exist yet**
   - The `exists` check filters out non-existent folders
   - New installations may not have the `~/.copilot/skills` or `~/.claude/skills` folders created

2. **User exclude patterns**
   - Check `files.exclude` and `search.exclude` in VS Code settings
   - Common patterns like `**/.*` could exclude `.claude` or `.copilot` folders

3. **Ripgrep not finding files**
   - If ripgrep process fails silently, no results are returned
   - Check if ripgrep is installed and working correctly

4. **Extension activation timing**
   - If skills are searched for before the file search provider is ready
   - Race condition between UI initialization and extension activation

### Less Likely but Possible

5. **Non-C: drive paths**
   - Only `C:` drive gets path normalization
   - Skills on `D:` or other drives might have path issues

6. **Network/UNC paths**
   - If user's home directory is on a network share
   - UNC path handling has edge cases

7. **Symlink handling**
   - The `followSymlinks` option affects traversal
   - Skills in symlinked directories might be missed

## Diagnostic Steps

### 1. Verify Folder Existence

```powershell
# Windows PowerShell
Test-Path "$env:USERPROFILE\.copilot\skills"
Test-Path "$env:USERPROFILE\.claude\skills"
```

### 2. Check VS Code Exclude Settings

Open Settings and search for:
- `files.exclude`
- `search.exclude`
- `explorer.excludeGitIgnore`

### 3. Test Search Service Directly

```typescript
// In VS Code Developer Tools Console
const searchService = await vscode.commands.executeCommand('vscode.executeWorkspaceSymbolProvider', '*/SKILL.md');
console.log(searchService);
```

### 4. Check Ripgrep Output

Enable search logging:
```json
{
    "search.useRipgrep": true,
    "search.searchOnType": false
}
```

Then check Developer Tools → Console for search-related messages.

### 5. Verify Skills Configuration

Check if skills feature is enabled:
```json
{
    "chat.useAgentSkills": true
}
```

## Recommendations

1. **Add logging to skill discovery** - Log when skills folder is searched and what results are found

2. **Create folders if missing** - Automatically create `~/.copilot/skills` on first use

3. **Surface search errors** - Don't silently fail when search provider is unavailable

4. **Validate exclude patterns** - Warn if exclude patterns might affect skill discovery

5. **Add retry logic** - Retry skill search if initial attempt returns empty during startup

---

*Investigation completed: January 20, 2026*
