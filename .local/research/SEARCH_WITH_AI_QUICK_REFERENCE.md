# Search with AI - Quick Reference

## What Is It?

A feature in VS Code that enables AI-powered semantic code search via an extensible provider architecture. GitHub Copilot Chat implements this using workspace chunking + LLM-based ranking.

## User Experience

1. **Search for code** using the Search View (Ctrl+Shift+F)
2. **"Search with AI" button** appears (if Copilot Chat is active)
3. **Click button** or press `Ctrl+I` (with Search View focused)
4. **Results stream in** showing semantically relevant matches from Copilot

## Architecture

### Three Main Components

**Search View UI**
- Displays results in tree structure
- "AI Results" section (separate from text search results)
- "Search with AI" button triggers the action

**Search Service** (Core)
- Routes AI search queries to registered providers
- Manages cancellation tokens
- Caches results separately from text search

**AI Text Search Provider** (Extension Implementation)
- Implements `vscode.AITextSearchProvider` interface
- Copilot Chat: Uses semantic chunking + LLM ranking
- Extensible: Other extensions can provide alternative implementations

## Key Files

### VS Code Core
```
src/
  ├── vs/workbench/services/search/
  │   └── common/searchService.ts          # Search routing
  ├── vs/workbench/contrib/search/
  │   ├── browser/searchView.ts            # UI + "Search with AI" button
  │   ├── browser/searchActionsTopBar.ts   # SearchWithAIAction command
  │   └── browser/AISearch/aiSearchModel.ts  # Result tree structure
```

### Copilot Chat Extension
```
src/
  ├── extension/conversation/
  │   └── vscode-node/conversationFeature.ts  # Provider registration
  └── extension/workspaceSemanticSearch/
      └── node/semanticSearchTextSearchProvider.ts  # Implementation
```

## Extension API

```typescript
// Register a provider
vscode.workspace.registerAITextSearchProvider(
  'file',  // scheme
  provider: AITextSearchProvider  // your implementation
)

// Implement this interface
interface AITextSearchProvider {
  readonly name?: string;  // e.g., "Copilot"
  provideAITextSearchResults(
    query: string,
    options: TextSearchProviderOptions,
    progress: Progress<TextSearchResult2>,
    token: CancellationToken
  ): ProviderResult<TextSearchComplete2>
}
```

## How Copilot Chat Does It

```
User Query
    ↓
Chunk Workspace (split code into searchable pieces)
    ↓
Semantic Search (find relevant chunks using embeddings)
    ↓
LLM Ranking (re-rank results with language model)
    ↓
Stream Results (report via progress callback)
    ↓
Display in Search View
```

## Result Format

```typescript
interface TextSearchResult2 {
  uri?: Uri;                    // File URI
  rangeLocations: SearchRangeSetPairing[];  // Where in file
  previewText: string;          // Line content showing match
  webviewIndex?: number;
  cellFragment?: string;
}

interface TextSearchComplete2 {
  limitHit?: boolean;           // More results available?
}
```

## Preconditions

"Search with AI" action only appears when:
- ✅ VS Code has registered AI text search provider
- ✅ Provider is registered for `file` scheme
- ✅ Search View has focus (for keybinding)
- ✅ (For Copilot Chat): User has GitHub session

## Keybinding

| Context | Binding |
|---------|---------|
| **Windows/Linux** | `Ctrl+I` |
| **Mac** | `Cmd+I` |
| **Requirement** | Search View must have focus |

## Cancellation

Each search gets:
- Unique instance ID (for tracking)
- `CancellationTokenSource` (for interruption)
- Independent token (separate from text search)

Cancellation properly cleans up resources.

## Result Caching

- **Separate caches**: AI results ≠ text search results
- **Same query**: Can display both simultaneously
- **Independent state**: `hasAIResults` vs. `hasPlainResults`

## Telemetry (Copilot Chat)

Tracks performance metrics:
- Chunk count discovered
- Search duration (chunking vs. ranking phases)
- Ranking strategy used
- Result deduplication stats
- LLM ranking scores (best/worst)

## Extensibility

Any extension can:
1. Implement `vscode.AITextSearchProvider`
2. Register via `vscode.workspace.registerAITextSearchProvider('file', provider)`
3. Provide custom search strategy (semantic, regex-based, ML, etc.)

Only constraint: One provider per scheme.

## Design Principles

1. **Provider-based**: Pluggable architecture, not hardcoded
2. **Non-blocking**: Results stream progressively
3. **Cancellable**: User can interrupt searches
4. **Coexistent**: AI + text search results shown together
5. **Extensible**: Custom providers possible without core changes

## Context Keys (UI Visibility)

```
github.copilot.search.feedback.sent  # Feedback button visibility
hasAIResultProvider                   # "Search with AI" button visibility
```

## Common Issues & Solutions

| Issue | Cause | Solution |
|-------|-------|----------|
| No "Search with AI" button | No provider registered | Ensure Copilot Chat is enabled |
| Search blocked | Copilot Chat still processing | Wait or press ESC to cancel |
| Results not appearing | Token cancelled | Retry search |
| No authentication | Not signed in to GitHub | Login via GitHub in VS Code |

## Related Reading

- **Provider API**: `vscode.proposed.aiTextSearchProvider.d.ts`
- **Search Types**: `src/vs/workbench/services/search/common/search.ts`
- **UI Implementation**: `src/vs/workbench/contrib/search/browser/searchView.ts`
- **Copilot Implementation**: `SemanticSearchTextSearchProvider`
