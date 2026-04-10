# copilot_searchCodebase Tool

## Overview

**Tool Name**: `copilot_searchCodebase` (internal: `semantic_search`)

**Purpose**: Performs intelligent semantic search across the workspace to find relevant code snippets, comments, and documentation based on natural language queries.

**Primary File**: `src/extension/tools/node/codebaseTool.tsx`

**Tool Class**: `CodebaseTool`

## Tool Declaration

```json
{
  "name": "copilot_searchCodebase",
  "toolReferenceName": "codebase",
  "displayName": "Search Codebase",
  "icon": "$(folder)",
  "modelDescription": "Run a natural language search for relevant code or documentation comments from the user's current workspace. Returns relevant code snippets from the user's current workspace if it is large, or the full contents of the workspace if it is small.",
  "inputSchema": {
    "type": "object",
    "properties": {
      "query": {
        "type": "string",
        "description": "The query to search the codebase for. Should contain all relevant context. Should ideally be text that might appear in the codebase, such as function names, variable names, or comments."
      }
    },
    "required": ["query"]
  }
}
```

## Input Parameters

```typescript
interface ICodebaseToolParams {
    query: string;  // Required: natural language search query

    // Internal parameters (not exposed to model)
    includeFileStructure?: boolean;  // Include workspace structure in results
    scopedDirectories?: string[];    // Limit search to specific directories
}
```

## Architecture

### High-Level Flow

```
User Query
    ↓
CodebaseTool.invoke()
    ↓
WorkspaceContext.prepare()
    ├─→ Query Enhancement (Meta-prompt)
    │   └─→ Resolve ambiguity, generate keywords
    ↓
WorkspaceChunkSearchService.searchFileChunks()
    ├─→ Convert query to embedding
    ├─→ Strategy Selection:
    │   ├─→ 1. FullWorkspace (small repos)
    │   ├─→ 2. CodeSearch (GitHub API + fallback)
    │   └─→ 3. Local Search (embeddings/TF-IDF)
    ↓
Filter & Re-rank Results
    ├─→ Remove ignored files
    ├─→ Compute missing embeddings
    ├─→ Sort by similarity
    └─→ Filter low-quality results
    ↓
Format & Return Results
```

## Internal Implementation

### 1. Tool Invocation Entry Point

**File**: `src/extension/tools/node/codebaseTool.tsx`

```typescript
async invoke(
    options: vscode.LanguageModelToolInvocationOptions<ICodebaseToolParams>,
    token: CancellationToken
): Promise<vscode.LanguageModelToolResult>
```

**Steps**:

1. **Mode Detection**: Check if "codebase agent" mode (multi-step search)
   - If yes → delegate to `CodebaseAgentService`
   - If no → continue with standard search

2. **Validation**: Ensure `query` parameter is provided

3. **Telemetry Setup**: Create correlation ID for tracking

4. **Context Rendering**: Call `WorkspaceContextWrapper` component

5. **Result Packaging**: Format results with metadata and references

### 2. Query Enhancement

**Component**: `WorkspaceContext` (in `src/extension/context/workspaceContext.tsx`)

**Method**: `prepare()`

The system enhances queries before searching to improve accuracy:

```typescript
async prepare(message: string, token: CancellationToken): Promise<WorkspaceChunkQuery>
```

**Query Enhancement Process**:

1. **Get Fast Endpoint**: Retrieve `copilot-fast` model for meta-prompt

2. **Build Meta-Prompt**: Create specialized prompt that asks the model to:
   - Rephrase the query, resolving pronouns ("it", "that", "this")
   - Generate up to 8 relevant keywords with variations
   - Think step-by-step about what the user is asking

3. **Smart Resolution**: Return a query object with lazy resolution:
   ```typescript
   {
       rawQuery: message,

       async resolveQuery(token: CancellationToken) {
           if (maySkipResolve()) {
               return message;  // Skip if no ambiguous words
           } else {
               return await racePromises([
                   metaPrompt.query,
                   metaPrompt.queryAndKeywords.then(x => x?.rephrasedQuery)
               ]);
           }
       },

       async resolveQueryAndKeywords(token: CancellationToken) {
           return metaPrompt.queryAndKeywords;
       }
   }
   ```

**Skip Logic**: Meta-prompt is skipped if query contains no ambiguous words like pronouns or unclear references.

**Example Enhancement**:

| Original Query | Enhanced Query | Keywords |
|---------------|----------------|----------|
| "find the auth code" | "find user authentication code" | ["auth", "authentication", "login", "session", "user", "credential", "token", "oauth"] |
| "how does this work?" | "how does the payment processing system work?" | ["payment", "process", "transaction", "checkout", "billing"] |

### 3. Search Orchestration

**Service**: `WorkspaceChunkSearchService`

**File**: `src/platform/workspace/workspaceChunkSearch/workspaceChunkSearchService.ts`

**Method**: `searchFileChunks()`

#### Step 3.1: Embedding Conversion

```typescript
const queryWithEmbeddings = this.toQueryWithEmbeddings(query, token);
```

This creates a promise that will:
1. Resolve the enhanced query string
2. Compute embedding vector for the query using `IEmbeddingsComputer`
3. Return both query text and embedding

#### Step 3.2: Strategy Selection

The system uses multiple search strategies with intelligent fallbacks:

**Strategy Priority**:

```
1. FullWorkspaceStrategy
   ↓ (if workspace too large)
2. CodeSearchStrategy (GitHub API)
   ↓ (fallback on timeout/failure)
3. Local Search Strategies:
   - EmbeddingsChunkSearch (if index ready)
   - TfIdfWithSemanticChunkSearch (if embeddings not ready)
   - TfidfChunkSearch (last resort)
```

**Strategy Details**:

##### A. FullWorkspaceStrategy

- **When**: Workspace is very small (under file limit)
- **How**: Returns entire workspace content
- **Benefit**: No need for ranking, instant results
- **Limitation**: Only viable for tiny codebases

##### B. CodeSearchStrategy (with fallback)

**Primary Strategy**:

- **API**: GitHub Code Search API
- **Timeout**: 12.5 seconds
- **File**: `src/extension/workspaceChunkSearch/codeSearchChunkSearch.ts`

**Process**:
1. Check if GitHub Code Search is available
2. Query remote indexed repository
3. Get file-level matches
4. Apply local embeddings for chunk-level ranking

**Diff Tracking**:
- Monitors local changes vs remote index
- Max diff size: 2000 files
- Max diff percentage: 70%
- Falls back to local if too much divergence

**Fallback Mechanism**:
- Timeout after 12.5s
- Launches local search in parallel
- Returns whichever completes first
- Continues background search for better results

##### C. Local Search Strategies

**C1. EmbeddingsChunkSearch**

- **When**: Local embedding index is ready
- **Timeout**: 8 seconds before falling back
- **File**: `src/platform/workspace/workspaceChunkEmbeddingsIndex/workspaceChunkEmbeddingsIndex.ts`

**Process**:
1. Get query embedding
2. Retrieve all indexed file embeddings
3. Compute similarity scores (dot product)
4. Return top N ranked chunks

**C2. TfIdfWithSemanticChunkSearch**

- **When**: Embeddings index not ready
- **Hybrid Approach**:
  1. Use TF-IDF for initial keyword-based ranking
  2. Take top results
  3. Compute embeddings on-the-fly for semantic ranking

**C3. TfidfChunkSearch**

- **When**: All else fails
- **Method**: Pure keyword-based search
- **Fast**: No embedding computation needed

#### Step 3.3: Result Filtering

After strategy execution, results are filtered:

1. **Ignore Rules**: Apply `.copilotignore` patterns
2. **File Type Filters**: Remove non-code files if configured
3. **Duplicate Removal**: Deduplicate overlapping chunks

#### Step 3.4: Re-ranking

**When**: Results come from non-embedding strategy or need refinement

**Process**:

1. **Check Remote Reranker**: Query if remote reranking service available
2. **Local Reranking** (fallback):
   - Compute embeddings for any chunks missing them
   - Calculate similarity to query embedding
   - Sort by distance/similarity score
   - Filter low-quality results (spread > 65% from top score)

**Similarity Calculation**:

```typescript
function dotProduct(a: EmbeddingVector, b: EmbeddingVector): number {
    let dotProduct = 0;
    const len = Math.min(a.length, b.length);
    for (let i = 0; i < len; i++) {
        dotProduct += a[i] * b[i];
    }
    return dotProduct;
}

function distance(queryEmbedding: Embedding, chunkEmbedding: Embedding): number {
    return dotProduct(chunkEmbedding.value, queryEmbedding.value);
}
```

Higher dot product = more similar (vectors are normalized).

### 4. Embedding System

#### Embedding Type

Current system uses:

```typescript
EmbeddingType.metis_1024_I16_Binary = 'metis-1024-I16-Binary'
```

**Specifications**:
- Dimensions: 1024
- Format: I16 (16-bit integers) Binary
- Storage: ~6% of previous model (text-embedding-3-small-512)
- Optimization: 94% storage reduction

#### Computing Embeddings

**Interface**: `IEmbeddingsComputer`

**Method**:

```typescript
computeEmbeddings(
    type: EmbeddingType,
    inputs: readonly string[],
    options?: {
        inputType?: 'document' | 'query'
    },
    telemetryInfo?: TelemetryCorrelationId,
    token?: CancellationToken,
): Promise<Embeddings>
```

**Input Types**:
- `document`: For indexing code chunks
- `query`: For search queries (may use different encoding)

**Batch Processing**: Embeddings are computed in batches for efficiency via `IChunkingEndpointClient`.

### 5. Indexing System

**Service**: `WorkspaceChunkEmbeddingsIndex`

**File**: `src/platform/workspace/workspaceChunkEmbeddingsIndex/workspaceChunkEmbeddingsIndex.ts`

#### Storage

- **Location**: Extension storage URI (persists across sessions)
- **Implementation**: `IndexStorage` class
- **Format**: Efficient binary storage for embeddings

#### Indexing Triggers

**Auto-Indexing**:
- Triggered on workspace open
- File limit: 750 files (default)
- Expanded limit: 50,000 files (with expanded capabilities)

**Manual Indexing**:
- User-initiated
- File limit: 2,500 files

**Incremental Updates**:
- Only recompute changed files
- Tracks file modification times
- Removes deleted files from index

#### Index Methods

**1. Trigger Indexing**:

```typescript
async triggerIndexingOfWorkspace(
    trigger: BuildIndexTriggerReason,  // 'auto' | 'manual'
    telemetryInfo: TelemetryCorrelationId,
    token: CancellationToken
): Promise<void>
```

**2. Search Workspace**:

```typescript
async searchWorkspace(
    query: Promise<Embedding>,
    maxResults: number,
    options: WorkspaceChunkSearchOptions,
    telemetryInfo: TelemetryCorrelationId,
    token: CancellationToken,
): Promise<FileChunkAndScore[]>
```

**Process**:
1. Load index from storage
2. Get all file embeddings
3. Compute similarity to query
4. Sort and return top N results

### 6. Chunking System

#### Constants

```typescript
MAX_CHUNKS_RESULTS = 128          // Maximum chunks to return
MAX_CHUNK_TOKEN_COUNT = 32_000    // Total token budget for all chunks
MAX_TOOL_CHUNK_TOKEN_COUNT = 20_000 // Token budget for tool calls
MAX_CHUNK_SIZE_TOKENS = 250       // Individual chunk size limit
```

#### Chunk Structure

```typescript
interface FileChunk {
    readonly file: URI;          // File URI
    readonly range: Range;       // Line range in file
    readonly text: string;       // Chunk content
    readonly isFullFile: boolean; // Whether entire file
}

interface FileChunkAndScore {
    readonly chunk: FileChunk;
    readonly distance?: EmbeddingDistance;  // Similarity score
}
```

#### Chunking Strategy

1. **File-Level**: Small files returned whole
2. **Function-Level**: Functions/methods as chunks
3. **Fixed-Size**: Large blocks split at token boundaries
4. **Context Preservation**: Include surrounding context lines

### 7. Result Formatting

**Component**: `WorkspaceChunkList`

**File**: `src/extension/context/workspaceContext.tsx`

**Output Format**:

```
# Relevant Code Chunks

## File: src/auth/login.ts (Lines 15-42)
\`\`\`typescript
export function authenticate(username: string, password: string) {
    // ... code ...
}
\`\`\`

## File: src/auth/session.ts (Lines 8-25)
\`\`\`typescript
export class SessionManager {
    // ... code ...
}
\`\`\`

[Total: 23 chunks from 12 files]
```

**References**: Each chunk is attached as a `vscode.Location` reference for IDE navigation.

## Dependencies

### Core Services

1. **IWorkspaceChunkSearchService**: Main search orchestrator
2. **IEmbeddingsComputer**: Embedding computation
3. **IChunkingEndpointClient**: Batch embedding requests
4. **IWorkspaceFileIndex**: File tracking
5. **IGithubCodeSearchService**: Remote code search
6. **IRerankerService**: Result reranking
7. **IIgnoreService**: `.copilotignore` handling
8. **IAuthenticationService**: GitHub auth
9. **IEndpointProvider**: LLM endpoint access
10. **IInstantiationService**: Dependency injection

### External APIs

- **GitHub Code Search API**: Remote semantic search
- **Copilot Embeddings API**: Vector embedding generation
- **Reranker API**: Advanced result ranking

## Performance Characteristics

### Time Budgets

| Operation | Timeout | Fallback |
|-----------|---------|----------|
| Full workspace | ~instant | N/A |
| Code search | 12.5s | Local search |
| Local embeddings | 8s | TF-IDF |
| TF-IDF fallback | Quick | None |
| Query enhancement | ~2s | Skip |

### Space Complexity

- **Embeddings**: 1024 dimensions × 2 bytes = 2KB per chunk
- **Index Size**: ~2KB × number of chunks
- **Memory**: Embeddings loaded on-demand

### Optimization Techniques

1. **Lazy Loading**: Services initialized only when needed
2. **Progressive Disclosure**: Fast results first, better results follow
3. **Racing**: Parallel strategies with timeouts
4. **Caching**: Persistent embedding storage
5. **Incremental Indexing**: Only recompute changed files
6. **Batch Processing**: Embeddings computed in batches
7. **Early Exit**: Skip expensive operations when simple search suffices

## Configuration Settings

```typescript
// Enable GitHub Code Search integration
"github.copilot.chat.codesearch.enabled": true

// Preemptively build local index
"copilotchat.workspaceChunkSearch.shouldEagerlyInitLocalIndex": false

// File limits for auto-indexing
"copilotchat.workspaceChunkSearch.automaticIndexingFileCap": 750
"copilotchat.workspaceChunkSearch.expandedAutomaticIndexingFileCap": 50000

// Manual indexing limit
"copilotchat.workspaceChunkSearch.manualIndexingFileCap": 2500
```

## Special Modes

### Codebase Agent Mode

**Trigger**: Special flag in tool invocation

**Behavior**:
- Multi-step search process
- Iterative query refinement
- More comprehensive results
- Uses `CodebaseAgentService`

**Use Case**: Complex, ambiguous queries requiring multiple search passes

## Telemetry Events

The tool emits detailed telemetry:

- **Tool invocation**: Query, parameters, mode
- **Strategy selection**: Which strategy chosen, why
- **Timing**: Each stage duration
- **Results**: Count, quality metrics
- **Failures**: Errors, fallback triggers
- **Index state**: Size, freshness, coverage

## Error Handling

### Common Error Scenarios

1. **No workspace open**: Return empty results with message
2. **Index unavailable**: Fall back to TF-IDF search
3. **API timeout**: Use local search strategies
4. **Authentication failure**: Skip remote search
5. **Large workspace**: Auto-limit to indexed files

### Graceful Degradation

The system always returns *something*:
- Full workspace → Code search → Embeddings → TF-IDF → Empty results

No single failure prevents the tool from functioning.

## Implementation Blueprint

To implement this tool in another project:

### 1. Core Requirements

- Embedding computation API (OpenAI, Anthropic, etc.)
- File system access
- Persistent storage for index
- Cancellation token support

### 2. Minimal Implementation

**Phase 1**: Basic search
- File traversal
- TF-IDF keyword search
- Simple ranking

**Phase 2**: Add embeddings
- Compute embeddings for files
- Store in persistent index
- Semantic similarity ranking

**Phase 3**: Advanced features
- Query enhancement
- Multiple strategies
- Incremental indexing
- Remote search integration

### 3. Key Algorithms

**TF-IDF Scoring**:
```typescript
function tfidfScore(query: string[], document: string[]): number {
    const tf = termFrequency(query, document);
    const idf = inverseDocumentFrequency(query, allDocuments);
    return tf * idf;
}
```

**Embedding Similarity**:
```typescript
function cosineSimilarity(a: number[], b: number[]): number {
    return dotProduct(a, b) / (magnitude(a) * magnitude(b));
}
```

**Chunking**:
```typescript
function chunkFile(content: string, maxTokens: number): Chunk[] {
    // Split on function boundaries
    // Fall back to fixed-size chunks
    // Include context lines
}
```

### 4. Storage Schema

```typescript
interface IndexEntry {
    file: string;
    chunk: {
        start: number;
        end: number;
    };
    embedding: number[];
    hash: string;  // For change detection
    timestamp: number;
}
```

### 5. API Integration Points

**Embedding API**:
```typescript
interface EmbeddingAPI {
    computeEmbeddings(
        texts: string[],
        type: 'document' | 'query'
    ): Promise<number[][]>;
}
```

**Search API** (optional):
```typescript
interface RemoteSearchAPI {
    search(query: string, repoId: string): Promise<SearchResult[]>;
}
```

## Best Practices

1. **Always provide fallbacks**: Network can fail, services can be down
2. **Use timeouts aggressively**: Better partial results than hanging
3. **Cache embeddings**: Computation is expensive
4. **Batch API calls**: Reduce round-trips
5. **Index incrementally**: Full reindex is slow
6. **Track telemetry**: Essential for optimization
7. **Respect ignore files**: Privacy and performance
8. **Normalize embeddings**: Enables dot product similarity
9. **Clean queries**: Remove noise before embedding
10. **Chunk intelligently**: Preserve semantic boundaries

## Limitations

1. **Large workspaces**: May not index everything
2. **Binary files**: Not searched
3. **Generated code**: May pollute results
4. **Query quality**: Bad queries → bad results
5. **Cold start**: First search slower (index building)
6. **API rate limits**: May throttle embedding computation
7. **Language support**: Best for common languages
8. **Context length**: Chunks may miss cross-file context

## Future Enhancements

- Multi-modal search (code + comments + docs)
- Cross-repository search
- Historical code search (git history)
- Personalized ranking (user preferences)
- Query suggestions
- Search analytics dashboard
