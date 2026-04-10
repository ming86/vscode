# "Search with AI" Feature - Architecture & Implementation Research

## Overview

"Search with AI" is a VS Code feature that enables semantic/AI-powered code search alongside traditional text search. It's implemented through a provider-based architecture where extensions can register custom AI search providers via the **AITextSearchProvider API**.

---

## Architecture Layers

### 1. **Extension API Layer** (Public Interface)

**Location:** `vscode.proposed.aiTextSearchProvider.d.ts`

```typescript
export interface AITextSearchProvider {
	readonly name?: string;
	provideAITextSearchResults(
		query: string,
		options: TextSearchProviderOptions,
		progress: Progress<TextSearchResult2>,
		token: CancellationToken
	): ProviderResult<TextSearchComplete2>;
}

export namespace workspace {
	export function registerAITextSearchProvider(
		scheme: string,
		provider: AITextSearchProvider
	): Disposable;
}
```

**Key Concepts:**
- Extensions register a provider for a specific scheme (typically `'file'`)
- Provider name displayed as `"{name} Results"` in Search View
- Results can be streamed via progress callbacks
- Cancellation tokens allow interruption

---

### 2. **Core Search Service**

**Locations:**
- `src/vs/workbench/services/search/common/searchService.ts`
- `src/vs/workbench/services/search/common/search.ts`

#### Query Type Definition

```typescript
export const enum QueryType {
	File = 1,
	Text = 2,
	aiText = 3
}

export interface IAITextQueryProps<U extends UriComponents> extends ICommonQueryProps<U> {
	type: QueryType.aiText;
	contentPattern: string;
	previewOptions?: ITextSearchPreviewOptions;
	maxFileSize?: number;
	surroundingContext?: number;
	userDisabledExcludesAndIgnoreFiles?: boolean;
}

export type IAITextQuery = IAITextQueryProps<URI>;
```

#### Search Service

```typescript
export class SearchService extends Disposable implements ISearchService {
	private readonly aiTextSearchProviders = new Map<string, ISearchResultProvider>();

	registerSearchResultProvider(
		scheme: string,
		type: SearchProviderType,
		provider: ISearchResultProvider
	): IDisposable { }

	async aiTextSearch(
		query: IAITextQuery,
		token?: CancellationToken,
		onProgress?: (item: ISearchProgressItem) => void
	): Promise<ISearchComplete> { }

	async getAIName(): Promise<string | undefined> { }
}
```

**Flow:**
1. Provider registered per scheme (typically `file://`)
2. AI search queries route to appropriate provider
3. Results aggregated with progress callbacks

---

### 3. **Search Model** (ViewModel)

**Location:** `src/vs/workbench/contrib/search/browser/searchTreeModel/searchModel.ts`

```typescript
export class SearchModelImpl extends Disposable implements ISearchModel {
	aiSearch(onResult: (result: ISearchProgressItem | undefined) => void): Promise<ISearchComplete> {
		if (this.hasAIResults) {
			throw Error('AI results already exist');
		}
		if (!this._searchQuery) {
			throw Error('No search query');
		}

		const searchInstanceID = Date.now().toString();
		const tokenSource = new CancellationTokenSource();
		this.currentAICancelTokenSource = tokenSource;

		const asyncAIResults = this.searchService.aiTextSearch(
			{ 
				...this._searchQuery,
				contentPattern: this._searchQuery.contentPattern.pattern,
				type: QueryType.aiText 
			},
			tokenSource.token,
			async (p: ISearchProgressItem) => {
				onResult(p);
				this.onSearchProgress(p, searchInstanceID, false, true);
			}
		);

		return asyncAIResults;
	}

	get hasAIResults(): boolean {
		return !!(this.searchResult.getCachedSearchComplete(true)) ||
			(!!this.currentAICancelTokenSource && 
			 !this.currentAICancelTokenSource.token.isCancellationRequested);
	}

	get hasPlainResults(): boolean {
		return !!(this.searchResult.getCachedSearchComplete(false)) ||
			(!!this.currentCancelTokenSource && 
			 !this.currentCancelTokenSource.token.isCancellationRequested);
	}
}
```

**Key Features:**
- Separate cache for AI vs. plain text results
- Independent cancellation tokens
- Progress tracking per search instance
- Query type conversion to `QueryType.aiText`

---

### 4. **AI Search UI Model**

**Location:** `src/vs/workbench/contrib/search/browser/AISearch/aiSearchModel.ts`

```typescript
export class AITextSearchHeadingImpl extends TextSearchHeadingImpl<IAITextQuery> {
	public override hidden: boolean;
	
	override name(): string {
		return 'AI';
	}

	id(): string {
		return TEXT_SEARCH_HEADING_PREFIX + AI_TEXT_SEARCH_RESULT_ID;
	}

	get isAIContributed(): boolean {
		return true;
	}

	override get query(): IAITextQuery | null {
		return this._query;
	}

	override set query(query: IAITextQuery | null) {
		this.clearQuery();
		if (!query) {
			return;
		}

		this._folderMatches = (query && query.folderQueries || [])
			.map(fq => fq.folder)
			.map((resource, index) =>
				this._createBaseFolderMatch(resource, resource.toString(), index, query)
			);

		this._folderMatches.forEach(fm => this._folderMatchesMap.set(fm.resource, fm));
		this._query = query;
	}
}
```

**Hierarchy:**
```
AITextSearchHeadingImpl (results group)
  → AIFolderMatchWorkspaceRootImpl[] (per workspace folder)
    → AIFileMatch[] (per matched file)
      → TextSearchMatch2[] (actual matches)
```

---

### 5. **Search View UI**

**Location:** `src/vs/workbench/contrib/search/browser/searchView.ts`

#### "Search with AI" Button

```typescript
private appendSearchWithAIButton(messageEl: HTMLElement) {
	const searchWithAIButtonTooltip = appendKeyBindingLabel(
		nls.localize('triggerAISearch.tooltip', "Search with AI."),
		this.keybindingService.lookupKeybinding(
			Constants.SearchCommandIds.SearchWithAIActionId
		)
	);
	const searchWithAIButtonText = nls.localize(
		'searchWithAIButtonTooltip',
		"Search with AI"
	);
	const searchWithAIButton = this.messageDisposables.add(
		new SearchLinkButton(
			searchWithAIButtonText,
			() => {
				this.commandService.executeCommand(
					Constants.SearchCommandIds.SearchWithAIActionId
				);
			},
			this.hoverService,
			searchWithAIButtonTooltip
		)
	);
	dom.append(messageEl, searchWithAIButton.element);
}
```

#### Adding AI Results

```typescript
public async addAIResults() {
	const excludePatternText = this._getExcludePattern();
	const includePatternText = this._getIncludePattern();

	this.searchWidget.searchInput?.clearMessage();
	this.showEmptyStage();
	this._visibleMatches = 0;
	this.tree.setSelection([]);
	this.tree.setFocus([]);

	this.viewModel.replaceString = this.searchWidget.getReplaceValue();

	// Reuse pending aiSearch if available
	let aiSearchPromise = this._pendingSemanticSearchPromise;
	if (!aiSearchPromise) {
		this.viewModel.searchResult.setAIQueryUsingTextQuery();
		aiSearchPromise = this._pendingSemanticSearchPromise = 
			this.viewModel.aiSearch(() => {
				// Clear pending promise when first result comes in
				if (this._pendingSemanticSearchPromise === aiSearchPromise) {
					this._pendingSemanticSearchPromise = undefined;
				}
			});
	}

	aiSearchPromise.then(
		(complete) => {
			this.updateSearchResultCount(
				this.viewModel.searchResult.query?.userDisabledExcludesAndIgnoreFiles,
				this.viewModel.searchResult.query?.onlyOpenEditors,
				false
			);
			return this.onSearchComplete(
				() => { },
				excludePatternText,
				includePatternText,
				complete,
				false,
				complete.aiKeywords
			);
		},
		(e) => {
			return this.onSearchError(
				e,
				() => { },
				excludePatternText,
				includePatternText,
				undefined,
				false
			);
		}
	);
}
```

---

### 6. **Search Command Action**

**Location:** `src/vs/workbench/contrib/search/browser/searchActionsTopBar.ts`

```typescript
registerAction2(class SearchWithAIAction extends Action2 {
	constructor() {
		super({
			id: Constants.SearchCommandIds.SearchWithAIActionId,
			title: nls.localize2('SearchWithAIAction.label', "Search with AI"),
			category,
			f1: true,
			precondition: Constants.SearchContext.hasAIResultProvider,
			keybinding: {
				weight: KeybindingWeight.WorkbenchContrib,
				when: ContextKeyExpr.and(
					Constants.SearchContext.hasAIResultProvider,
					Constants.SearchContext.SearchViewFocusedKey
				),
				primary: KeyMod.CtrlCmd | KeyCode.KeyI
			}
		});
	}

	async run(accessor: ServicesAccessor, ...args: unknown[]) {
		const searchView = getSearchView(accessor.get(IViewsService));
		if (searchView) {
			searchView.requestAIResults();
		}
	}
});
```

**Precondition:** Only enabled if `hasAIResultProvider` context is true  
**Keybinding:** `Ctrl+I` / `Cmd+I` (when Search View is focused)

---

## GitHub Copilot Chat Implementation

### Provider Registration

**Location:** `vscode-copilot-chat/src/extension/conversation/vscode-node/conversationFeature.ts`

```typescript
private registerSearchProvider(): IDisposable | undefined {
	if (this._searchProviderRegistered) {
		return;
	}

	this._searchProviderRegistered = true;

	// Don't register for no auth user
	if (this.authenticationService.copilotToken?.isNoAuthUser) {
		this.logService.debug(
			'ConversationFeature: Skipping search provider registration - ' +
			'no GitHub session available'
		);
		return;
	}

	return vscode.workspace.registerAITextSearchProvider(
		'file',
		this.instantiationService.createInstance(SemanticSearchTextSearchProvider)
	);
}
```

### Semantic Search Implementation

**Location:** `vscode-copilot-chat/src/extension/workspaceSemanticSearch/node/semanticSearchTextSearchProvider.ts`

```typescript
export class SemanticSearchTextSearchProvider implements vscode.AITextSearchProvider {
	private _endpoint: IChatEndpoint | undefined = undefined;
	public readonly name: string = 'Copilot';
	public static feedBackSentKey = 'github.copilot.search.feedback.sent';
	public static latestQuery: string | undefined = undefined;
	public static feedBackTelemetry: Partial<ISearchFeedbackTelemetry> = {};

	constructor(
		@IEndpointProvider private _endpointProvider: IEndpointProvider,
		@IWorkspaceChunkSearchService private readonly workspaceChunkSearch: 
			IWorkspaceChunkSearchService,
		@ILogService private readonly _logService: ILogService,
		@ITelemetryService private readonly _telemetryService: ITelemetryService,
		@IIntentService private readonly _intentService: IIntentService,
		@IRunCommandExecutionService private readonly _commandService: 
			IRunCommandExecutionService,
		@ISearchService private readonly searchService: ISearchService,
		@IWorkspaceService private readonly workspaceService: IWorkspaceService,
		@IParserService private readonly _parserService: IParserService,
		@IRerankerService private readonly _rerankerService: IRerankerService,
	) { }

	async provideAITextSearchResults(
		query: string,
		options: vscode.TextSearchProviderOptions,
		progress: vscode.Progress<vscode.TextSearchResult2>,
		token: vscode.CancellationToken
	): Promise<vscode.TextSearchComplete2> {
		this.resetFeedbackContext();
		const sw = new StopWatch();
		// ... implementation
	}
}
```

#### Key Pipeline Steps:

1. **Workspace Chunking**: Breaks codebase into searchable chunks
2. **Semantic Search**: Finds relevant chunks using embeddings
3. **LLM Ranking**: Re-ranks results using language model
4. **Filtering**: Applies include/exclude patterns
5. **Progress Streaming**: Reports results incrementally via progress callback

#### Telemetry Tracking:

```typescript
export interface ISearchFeedbackTelemetry {
	chunkCount: number;
	rankResult: string;
	rankResultsCount: number;
	combinedResultsCount: number;
	chunkSearchDuration: number;
	llmFilteringDuration: number;
	llmBestRank?: number;
	llmWorstRank?: number;
	llmSelectedCount?: number;
	rawLlmRankingResultsCount?: number;
	parseResult?: string;
	strategy?: string;
	llmBestInRerank?: number;
	llmWorstInRerank?: number;
}
```

---

## Data Flow Diagram

```
User Action
    ↓
┌─────────────────────────────────────────────┐
│ Search View - "Search with AI" Button Click │
└────────────────┬────────────────────────────┘
                 ↓
         SearchWithAIAction
         (searchActionsTopBar.ts)
                 ↓
         SearchView.requestAIResults()
                 ↓
┌──────────────────────────────────────────────┐
│     SearchView.addAIResults()                │
│  - Sets AI query from text query             │
│  - Clears previous AI results                │
│  - Calls viewModel.aiSearch()                │
└────────────┬─────────────────────────────────┘
             ↓
┌──────────────────────────────────────────────┐
│   SearchModelImpl.aiSearch()                  │
│  - Creates cancellation token source         │
│  - Converts query to QueryType.aiText        │
│  - Calls searchService.aiTextSearch()        │
└────────────┬─────────────────────────────────┘
             ↓
┌──────────────────────────────────────────────┐
│   SearchService.aiTextSearch()               │
│  - Routes to appropriate provider            │
│  - Calls provider.doSearch()                 │
└────────────┬─────────────────────────────────┘
             ↓
┌──────────────────────────────────────────────┐
│  ISearchResultProvider.aiTextSearch()        │
│  (registered for 'file' scheme)              │
└────────────┬─────────────────────────────────┘
             ↓
┌──────────────────────────────────────────────┐
│  SemanticSearchTextSearchProvider            │
│    (GitHub Copilot Chat Extension)          │
│                                              │
│  1. provideAITextSearchResults()             │
│  2. Chunk workspace code                     │
│  3. Find relevant chunks (semantic search)   │
│  4. Rank with LLM                            │
│  5. Stream results via progress callback     │
└────────────┬─────────────────────────────────┘
             ↓
        Progress Callback
             ↓
┌──────────────────────────────────────────────┐
│  SearchView.onSearchProgress()               │
│  - Updates tree with results                 │
│  - Renders AITextSearchHeadingImpl            │
│  - Shows matches grouped by file             │
└──────────────────────────────────────────────┘
```

---

## Key Concepts

### 1. **Provider Architecture**

- **Multiple Providers**: Multiple providers can be registered for different schemes
- **One per Scheme**: Only one provider per scheme active at a time
- **Lazy Registration**: Copilot Chat registers provider when extension activates
- **Conditional**: Registration skipped if no GitHub session available

### 2. **Query Transformation**

Original text search query → AI search query

```
ITextQuery {
  contentPattern: IPatternInfo (e.g., { pattern: "foo", isRegExp: false })
  includePattern?: string
  excludePattern?: string
  folderQueries: IFolderQuery[]
}
        ↓
IAITextQuery {
  type: QueryType.aiText  // ← Type distinguishes from text search
  contentPattern: string  // ← Flattened pattern string
  includePattern?: string
  excludePattern?: string
  folderQueries: IFolderQuery[]
}
```

### 3. **Result Caching**

- Separate caches for AI (`aiTextSearchResult`) and text (`textSearchResult`) results
- Same query can have both AI and text results displayed together
- Results persist until explicitly cleared or new search initiated

### 4. **Progress Streaming**

Results reported incrementally via `Progress<TextSearchResult2>`:

```typescript
interface TextSearchResult2 {
	uri?: Uri;                    // File path
	rangeLocations: SearchRangeSetPairing[];  // Match locations
	previewText: string;          // Line content
	webviewIndex?: number;
	cellFragment?: string;
}
```

### 5. **Cancellation & Cleanup**

- Each search gets unique instance ID for tracking
- `CancellationTokenSource` allows interruption
- Token properly disposed after search completion
- Separate token source for AI vs. text searches

---

## Search Precondition

The "Search with AI" action only appears when:

```typescript
precondition: Constants.SearchContext.hasAIResultProvider
```

This context is set when:
1. Extension with AI text search provider is activated
2. Provider successfully registered for `file` scheme
3. No cancellation/error during registration

---

## Extension Points

### Extension API

```typescript
vscode.workspace.registerAITextSearchProvider(scheme, provider)
```

### Configuration

- No direct configuration for search with AI
- Availability depends on registered providers (Copilot Chat)
- Authentication required (GitHub session)

### Keybinding

- Default: `Ctrl+I` (Windows/Linux), `Cmd+I` (Mac)
- Only active when Search View has focus
- Conditional: Only if `hasAIResultProvider` context is true

---

## Interaction with Existing Search

### Coexistence

- AI results display alongside text search results
- Same search UI tree, different heading sections
- Independent toggle/visibility controls
- Separate result caching

### Query Reuse

```typescript
// AI search reuses existing text search query
this.viewModel.searchResult.setAIQueryUsingTextQuery();
aiSearchPromise = this.viewModel.aiSearch(...)
```

---

## Performance Considerations

### Copilot Chat Implementation

1. **Chunking Phase**: Pre-processes workspace into searchable chunks
2. **Semantic Search**: Fast embedding-based lookup
3. **LLM Ranking**: Filters results with language model (potentially slower)
4. **Streaming**: Reports results as they become available (not blocking)

### Telemetry

Tracks:
- Chunk count discovered
- Ranking strategy used
- Duration of each phase
- Result deduplication stats

---

## Extensibility

### Creating Custom AI Search Provider

```typescript
class MyAISearchProvider implements vscode.AITextSearchProvider {
	readonly name = "My Search";

	async provideAITextSearchResults(
		query: string,
		options: vscode.TextSearchProviderOptions,
		progress: vscode.Progress<vscode.TextSearchResult2>,
		token: vscode.CancellationToken
	): Promise<vscode.TextSearchComplete2> {
		// Your implementation
		// Report results via progress.report()
		return { limitHit: false };
	}
}

// Register in extension activation
export function activate(context: vscode.ExtensionContext) {
	vscode.workspace.registerAITextSearchProvider('file', new MyAISearchProvider());
}
```

---

## Summary

**"Search with AI"** is a provider-based architecture that:

1. **Enables** semantic/AI-powered search via extension providers
2. **Integrates** seamlessly with existing text search UI
3. **Coexists** with traditional search results
4. **Streams** results incrementally for responsiveness
5. **Supports** cancellation and progress tracking
6. **Implements** in Copilot Chat via semantic chunking + LLM ranking

The design cleanly separates concerns:
- **UI**: SearchView, AITextSearchHeadingImpl
- **Logic**: SearchModel, SearchService
- **Implementation**: Provider-specific (SemanticSearchTextSearchProvider)

This allows future extensions to provide alternative AI search strategies without modifying core search infrastructure.
