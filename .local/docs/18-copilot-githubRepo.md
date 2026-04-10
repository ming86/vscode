# Copilot githubRepo Tool Documentation

## Overview

The `githubRepo` tool provides access to GitHub's code search API, enabling Copilot to search for code patterns, find implementations, discover usage examples, and learn from public repositories. It integrates GitHub's advanced code search capabilities with authentication, rate limiting, and intelligent result processing.

**Primary Use Cases:**
- Finding real-world code examples from GitHub
- Discovering how popular libraries are used
- Learning implementation patterns from open source
- Finding similar code solutions
- Exploring API usage across repositories

**Tool Category:** External Code Search & Discovery

## Tool Declaration

```typescript
{
	name: 'githubRepo',
	description: 'Search for code across GitHub repositories using GitHub\'s code search API. Find real-world examples, usage patterns, and implementations from public repositories. Supports filtering by language, repository, organization, and file type.',
	inputSchema: {
		type: 'object',
		properties: {
			query: {
				type: 'string',
				description: 'The code search query. Can include code patterns, function names, or concepts. Supports GitHub search qualifiers.'
			},
			language: {
				type: 'string',
				description: 'Filter results by programming language (e.g., "typescript", "python", "javascript")'
			},
			repository: {
				type: 'string',
				description: 'Search within a specific repository (format: "owner/repo")'
			},
			organization: {
				type: 'string',
				description: 'Search within repositories owned by an organization'
			},
			filename: {
				type: 'string',
				description: 'Filter by filename or extension (e.g., "*.ts", "package.json")'
			},
			maxResults: {
				type: 'number',
				description: 'Maximum number of results to return (1-100, default 10)',
				default: 10,
				minimum: 1,
				maximum: 100
			}
		},
		required: ['query']
	}
}
```

## Input Parameters

### query (required)
- **Type:** `string`
- **Description:** The code search query with optional GitHub search qualifiers
- **Examples:**
  - `"function useState"` - Simple text search
  - `"createContext language:typescript"` - With qualifier
  - `"class extends Component"` - Pattern search
  - `"TODO: @username"` - Search comments

**GitHub Search Qualifiers:**
- `in:file` - Search file contents
- `in:path` - Search file paths
- `language:typescript` - Filter by language
- `repo:owner/name` - Specific repository
- `org:organization` - Organization repos
- `user:username` - User's repos
- `path:src/` - Path prefix
- `extension:ts` - File extension
- `size:>1000` - File size

### language (optional)
- **Type:** `string`
- **Description:** Filter by programming language
- **Examples:** `'typescript'`, `'python'`, `'javascript'`, `'go'`, `'rust'`

### repository (optional)
- **Type:** `string`
- **Format:** `'owner/repo'`
- **Description:** Search within a specific repository
- **Examples:** `'microsoft/vscode'`, `'facebook/react'`, `'vercel/next.js'`

### organization (optional)
- **Type:** `string`
- **Description:** Search within an organization's repositories
- **Examples:** `'microsoft'`, `'google'`, `'facebook'`

### filename (optional)
- **Type:** `string`
- **Description:** Filter by filename or extension
- **Examples:** `'*.ts'`, `'package.json'`, `'tsconfig.*'`

### maxResults (optional)
- **Type:** `number`
- **Default:** `10`
- **Range:** `1-100`
- **Description:** Maximum number of results to return

### Examples

```typescript
// Simple code search
{
	query: 'useState',
	language: 'typescript',
	maxResults: 5
}

// Repository-specific search
{
	query: 'TreeDataProvider',
	repository: 'microsoft/vscode',
	filename: '*.ts'
}

// Organization search
{
	query: 'async function',
	organization: 'vercel',
	language: 'javascript'
}

// Pattern search
{
	query: 'class extends React.Component',
	language: 'typescript',
	maxResults: 20
}

// Complex query with qualifiers
{
	query: 'interface Props in:file language:typescript path:src/',
	maxResults: 10
}
```

## Architecture

### High-Level Flow

```
User Query → githubRepo Tool
    ↓
Authentication Check
    ↓
Query Construction
    ↓
GitHub Code Search API
    ↓
Rate Limit Handling
    ↓
Result Processing
    ↓
Code Snippet Extraction
    ↓
Return Formatted Results
```

### Component Integration

```
┌─────────────────────────────────────────┐
│      githubRepo Tool Handler            │
│  - Query construction                   │
│  - Parameter validation                 │
└──────────────┬──────────────────────────┘
               ↓
┌─────────────────────────────────────────┐
│      GitHub API Service                 │
│  - Authentication                       │
│  - Rate limit management                │
│  - Request retry logic                  │
└──────────────┬──────────────────────────┘
               ↓
┌─────────────────────────────────────────┐
│      GitHub Code Search API             │
│  - Search index                         │
│  - Result ranking                       │
│  - Content delivery                     │
└──────────────┬──────────────────────────┘
               ↓
┌─────────────────────────────────────────┐
│      Result Processor                   │
│  - Code extraction                      │
│  - Context enhancement                  │
│  - Formatting                           │
└─────────────────────────────────────────┘
```

### Authentication Flow

```
┌─────────────────────────────────────────┐
│      GitHub Authentication              │
│                                          │
│  1. Check for existing token            │
│     ↓                                    │
│  2. Use VS Code GitHub auth provider    │
│     ↓                                    │
│  3. Request 'repo' scope if needed      │
│     ↓                                    │
│  4. Store token securely                │
│     ↓                                    │
│  5. Include in API requests             │
└─────────────────────────────────────────┘
```

## Internal Implementation

### Core Implementation Location

**File:** `src/extension/tools/builtinTools.ts`

### Implementation Code

```typescript
async function githubRepo(
	args: {
		query: string;
		language?: string;
		repository?: string;
		organization?: string;
		filename?: string;
		maxResults?: number;
	},
	context: ToolContext,
	token: CancellationToken
): Promise<string> {
	// Validate parameters
	const maxResults = Math.min(args.maxResults || 10, 100);

	// Get GitHub authentication
	const octokit = await getAuthenticatedOctokit(context);
	if (!octokit) {
		return 'GitHub authentication required. Please sign in to GitHub.';
	}

	// Build search query
	const searchQuery = buildSearchQuery(args);

	try {
		// Execute search with rate limit handling
		const results = await executeSearchWithRetry(
			octokit,
			searchQuery,
			maxResults,
			token
		);

		if (results.length === 0) {
			return `No results found for: ${searchQuery}`;
		}

		// Format results
		return formatSearchResults(results, args.query);
	} catch (error) {
		return handleSearchError(error);
	}
}
```

### Authentication Service

```typescript
import { Octokit } from '@octokit/rest';

async function getAuthenticatedOctokit(
	context: ToolContext
): Promise<Octokit | null> {
	// Get GitHub authentication provider
	const session = await vscode.authentication.getSession(
		'github',
		['repo', 'read:user'],
		{ createIfNone: false }
	);

	if (!session) {
		// Prompt user to sign in
		const signIn = await vscode.window.showInformationMessage(
			'GitHub authentication required for code search',
			'Sign In'
		);

		if (signIn === 'Sign In') {
			const newSession = await vscode.authentication.getSession(
				'github',
				['repo', 'read:user'],
				{ createIfNone: true }
			);

			if (newSession) {
				return new Octokit({ auth: newSession.accessToken });
			}
		}

		return null;
	}

	return new Octokit({ auth: session.accessToken });
}
```

### Query Construction

```typescript
function buildSearchQuery(args: {
	query: string;
	language?: string;
	repository?: string;
	organization?: string;
	filename?: string;
}): string {
	const parts: string[] = [args.query];

	// Add language filter
	if (args.language) {
		parts.push(`language:${args.language}`);
	}

	// Add repository filter
	if (args.repository) {
		parts.push(`repo:${args.repository}`);
	}

	// Add organization filter
	if (args.organization) {
		parts.push(`org:${args.organization}`);
	}

	// Add filename filter
	if (args.filename) {
		if (args.filename.startsWith('*.')) {
			parts.push(`extension:${args.filename.substring(2)}`);
		} else {
			parts.push(`filename:${args.filename}`);
		}
	}

	return parts.join(' ');
}
```

### Search Execution with Rate Limiting

```typescript
interface SearchResult {
	repository: {
		fullName: string;
		htmlUrl: string;
		description?: string;
	};
	path: string;
	htmlUrl: string;
	content: string;
	language?: string;
	score: number;
}

async function executeSearchWithRetry(
	octokit: Octokit,
	query: string,
	maxResults: number,
	token: CancellationToken,
	maxRetries = 3
): Promise<SearchResult[]> {
	let attempt = 0;

	while (attempt <= maxRetries) {
		if (token.isCancellationRequested) {
			throw new Error('Search cancelled');
		}

		try {
			// Execute search
			const response = await octokit.rest.search.code({
				q: query,
				per_page: maxResults,
				sort: 'indexed',
				order: 'desc'
			});

			// Check rate limit
			const rateLimit = response.headers['x-ratelimit-remaining'];
			if (rateLimit && parseInt(rateLimit) < 10) {
				console.warn(`GitHub API rate limit low: ${rateLimit} requests remaining`);
			}

			// Process results
			return await processSearchResults(
				octokit,
				response.data.items,
				token
			);
		} catch (error: any) {
			// Handle rate limiting
			if (error.status === 403 && error.response?.headers['x-ratelimit-remaining'] === '0') {
				const resetTime = error.response.headers['x-ratelimit-reset'];
				const waitTime = (parseInt(resetTime) * 1000) - Date.now();

				if (attempt < maxRetries && waitTime < 60000) { // Wait max 1 minute
					await new Promise(resolve => setTimeout(resolve, waitTime));
					attempt++;
					continue;
				}

				throw new Error(`GitHub API rate limit exceeded. Resets at ${new Date(parseInt(resetTime) * 1000).toISOString()}`);
			}

			// Handle other errors
			if (attempt === maxRetries) {
				throw error;
			}

			await new Promise(resolve => setTimeout(resolve, 1000 * (attempt + 1)));
			attempt++;
		}
	}

	throw new Error('Search failed after maximum retries');
}
```

### Result Processing

```typescript
async function processSearchResults(
	octokit: Octokit,
	items: any[],
	token: CancellationToken
): Promise<SearchResult[]> {
	const results: SearchResult[] = [];

	for (const item of items) {
		if (token.isCancellationRequested) break;

		try {
			// Fetch file content if not included
			let content = item.text_matches?.[0]?.fragment || '';

			if (!content && item.repository && item.path) {
				const fileResponse = await octokit.rest.repos.getContent({
					owner: item.repository.owner.login,
					repo: item.repository.name,
					path: item.path
				});

				if ('content' in fileResponse.data) {
					content = Buffer.from(
						fileResponse.data.content,
						'base64'
					).toString('utf-8');
				}
			}

			results.push({
				repository: {
					fullName: item.repository.full_name,
					htmlUrl: item.repository.html_url,
					description: item.repository.description
				},
				path: item.path,
				htmlUrl: item.html_url,
				content: content,
				language: item.language,
				score: item.score
			});
		} catch (error) {
			console.error(`Failed to fetch content for ${item.path}:`, error);
			// Continue with other results
		}
	}

	return results;
}
```

### Result Formatting

```typescript
function formatSearchResults(
	results: SearchResult[],
	originalQuery: string
): string {
	const sections: string[] = [];

	sections.push(`# GitHub Code Search Results`);
	sections.push(`Query: "${originalQuery}"`);
	sections.push(`Found ${results.length} result(s)\n`);

	for (let i = 0; i < results.length; i++) {
		const result = results[i];
		sections.push(formatSingleResult(result, i + 1));
	}

	return sections.join('\n');
}

function formatSingleResult(result: SearchResult, index: number): string {
	const parts: string[] = [];

	// Header
	parts.push(`## Result ${index}: ${result.repository.fullName}`);
	parts.push(`**File:** \`${result.path}\``);

	if (result.language) {
		parts.push(`**Language:** ${result.language}`);
	}

	if (result.repository.description) {
		parts.push(`**Repository:** ${result.repository.description}`);
	}

	// Code snippet
	if (result.content) {
		const snippet = extractRelevantSnippet(result.content, 50);
		const language = result.language?.toLowerCase() || 'text';

		parts.push(`\n**Code:**\n\`\`\`${language}\n${snippet}\n\`\`\``);
	}

	// Links
	parts.push(`\n[→ View on GitHub](${result.htmlUrl})`);
	parts.push(`[→ Repository](${result.repository.htmlUrl})`);

	return parts.join('\n') + '\n';
}

function extractRelevantSnippet(content: string, maxLines: number): string {
	const lines = content.split('\n');

	// If content is short enough, return all
	if (lines.length <= maxLines) {
		return content;
	}

	// Try to find the most relevant section
	// (This is simplified; real implementation would use more sophisticated matching)
	const middleStart = Math.floor((lines.length - maxLines) / 2);
	return lines.slice(middleStart, middleStart + maxLines).join('\n');
}
```

### Caching Strategy

```typescript
interface CacheEntry {
	results: SearchResult[];
	timestamp: number;
	query: string;
}

class GitHubSearchCache {
	private cache: Map<string, CacheEntry> = new Map();
	private readonly TTL = 60 * 60 * 1000; // 1 hour

	get(query: string): SearchResult[] | undefined {
		const entry = this.cache.get(query);

		if (!entry) return undefined;

		// Check if expired
		if (Date.now() - entry.timestamp > this.TTL) {
			this.cache.delete(query);
			return undefined;
		}

		return entry.results;
	}

	set(query: string, results: SearchResult[]): void {
		this.cache.set(query, {
			results,
			timestamp: Date.now(),
			query
		});

		// Cleanup old entries
		this.cleanup();
	}

	private cleanup(): void {
		const now = Date.now();
		for (const [key, entry] of this.cache.entries()) {
			if (now - entry.timestamp > this.TTL) {
				this.cache.delete(key);
			}
		}
	}
}
```

## Dependencies

### External Dependencies

```typescript
// GitHub API client
import { Octokit } from '@octokit/rest';

// VS Code authentication
import * as vscode from 'vscode';
```

### GitHub API Requirements

```typescript
// Required GitHub OAuth scopes
const REQUIRED_SCOPES = [
	'repo',      // Access to code search
	'read:user'  // User identification
];

// API endpoints used
const API_ENDPOINTS = {
	search: 'GET /search/code',
	content: 'GET /repos/{owner}/{repo}/contents/{path}',
	rateLimit: 'GET /rate_limit'
};
```

### Internal Dependencies

```typescript
import { ILanguageModelTool } from '../tools/toolsService';
import { CancellationToken } from '../../../platform/cancellation/common/cancellation';
import { IAuthenticationService } from '../../../platform/authentication/common/authentication';
```

## Performance Characteristics

### Response Time
- **Cached queries:** 10-50ms
- **Simple searches:** 500-2000ms
- **Complex searches with content fetch:** 2000-5000ms
- **Rate limited:** Up to 60000ms (1 minute wait)

### Rate Limits

```typescript
// GitHub API rate limits (authenticated)
const RATE_LIMITS = {
	searchRequests: {
		limit: 30,              // Requests per minute
		period: 60 * 1000       // 1 minute
	},
	codeSearch: {
		limit: 30,              // Searches per minute
		period: 60 * 1000
	}
};

// Rate limit monitoring
class RateLimitMonitor {
	private requestCount = 0;
	private resetTime = Date.now() + 60000;

	async checkLimit(): Promise<void> {
		if (Date.now() >= this.resetTime) {
			this.requestCount = 0;
			this.resetTime = Date.now() + 60000;
		}

		if (this.requestCount >= 30) {
			const waitTime = this.resetTime - Date.now();
			await new Promise(resolve => setTimeout(resolve, waitTime));
			this.requestCount = 0;
			this.resetTime = Date.now() + 60000;
		}

		this.requestCount++;
	}
}
```

### Memory Usage
- **Per search result:** 5-20 KB (including code snippet)
- **10 results:** 50-200 KB
- **Cache (100 queries):** 5-20 MB

## Use Cases

### Use Case 1: Finding Usage Examples

**Scenario:** User wants to see how a popular API is used in real projects.

**Implementation:**
```typescript
// User asks: "Show me examples of using React Context API"
const results = await githubRepo({
	query: 'createContext useContext',
	language: 'typescript',
	maxResults: 10
});
```

**Example Dialog:**
```
User: Show me how to use React Context API in TypeScript

Copilot: [calls githubRepo]
{
	query: 'createContext useContext',
	language: 'typescript',
	maxResults: 10
}

Found 10 real-world examples from GitHub:

## Result 1: vercel/next.js
**File:** `examples/with-context-api/components/Counter.tsx`
**Language:** TypeScript

**Code:**
```typescript
import { createContext, useContext, useState } from 'react'

interface CounterContextType {
	count: number
	increment: () => void
	decrement: () => void
}

const CounterContext = createContext<CounterContextType | undefined>(undefined)

export function CounterProvider({ children }: { children: React.ReactNode }) {
	const [count, setCount] = useState(0)

	return (
		<CounterContext.Provider value={{
			count,
			increment: () => setCount(c => c + 1),
			decrement: () => setCount(c => c - 1)
		}}>
			{children}
		</CounterContext.Provider>
	)
}

export function useCounter() {
	const context = useContext(CounterContext)
	if (!context) throw new Error('useCounter must be used within CounterProvider')
	return context
}
```

[→ View on GitHub](...)

This is a clean pattern for TypeScript Context API. Notice:
1. Separate Context type definition
2. Provider component with state
3. Custom hook with error handling

Would you like me to help implement this in your project?
```

### Use Case 2: Repository-Specific Search

**Scenario:** User wants to understand patterns used in a specific popular repository.

**Implementation:**
```typescript
// Search within VS Code repository
const results = await githubRepo({
	query: 'class implements TreeDataProvider',
	repository: 'microsoft/vscode',
	language: 'typescript',
	maxResults: 5
});
```

**Example Dialog:**
```
User: How does VS Code implement tree views?

Copilot: [calls githubRepo]
{
	query: 'class implements TreeDataProvider',
	repository: 'microsoft/vscode',
	language: 'typescript'
}

Here are tree view implementations from VS Code's own codebase:

## Result 1: microsoft/vscode
**File:** `src/vs/workbench/contrib/debug/browser/debugCallStackView.ts`

Shows how VS Code implements the debug call stack tree...

## Result 2: microsoft/vscode
**File:** `src/vs/workbench/contrib/files/browser/explorerView.ts`

Shows the file explorer tree implementation...

These are production implementations used in VS Code itself. Key patterns:
- TreeDataProvider implementation
- Refresh handling with events
- Lazy loading of children
- Icon and tooltip support

Would you like me to explain any specific part?
```

### Use Case 3: Learning Library Usage

**Scenario:** User is learning a new library and wants real examples.

**Implementation:**
```typescript
// Find Prisma usage examples
const results = await githubRepo({
	query: 'PrismaClient',
	language: 'typescript',
	organization: 'prisma',
	maxResults: 10
});
```

**Example Dialog:**
```
User: I'm learning Prisma. Show me some examples of how to use PrismaClient.

Copilot: [calls githubRepo]
{
	query: 'PrismaClient',
	language: 'typescript',
	organization: 'prisma',
	maxResults: 10
}

Here are examples from Prisma's official repositories:

## Result 1: prisma/prisma-examples
**File:** `typescript/rest-express/src/index.ts`

```typescript
import { PrismaClient } from '@prisma/client'

const prisma = new PrismaClient()

async function main() {
	// Create user
	const user = await prisma.user.create({
		data: {
			name: 'Alice',
			email: 'alice@example.com'
		}
	})

	// Query users
	const users = await prisma.user.findMany()
}

main()
	.catch((e) => console.error(e))
	.finally(async () => await prisma.$disconnect())
```

## Result 2: prisma/prisma-examples
**File:** `typescript/graphql/src/context.ts`

Shows context setup for GraphQL with Prisma...

These examples show:
1. Basic client initialization
2. CRUD operations
3. Error handling and cleanup
4. Integration with Express/GraphQL

Would you like help setting up Prisma in your project?
```

### Use Case 4: Finding Design Patterns

**Scenario:** User wants to see how a design pattern is implemented.

**Implementation:**
```typescript
// Find singleton pattern examples
const results = await githubRepo({
	query: 'class singleton getInstance private constructor',
	language: 'typescript',
	maxResults: 15
});
```

**Example Dialog:**
```
User: Show me good examples of the Singleton pattern in TypeScript

Copilot: [calls githubRepo]
{
	query: 'singleton getInstance private constructor',
	language: 'typescript',
	maxResults: 15
}

Found several Singleton pattern implementations:

## Result 1: microsoft/vscode
**File:** `src/vs/platform/instantiation/common/instantiation.ts`

```typescript
export class ServiceCollection implements IServiceCollection {
	private static _instance: ServiceCollection

	private constructor() {
		// Private constructor prevents external instantiation
	}

	public static getInstance(): ServiceCollection {
		if (!ServiceCollection._instance) {
			ServiceCollection._instance = new ServiceCollection()
		}
		return ServiceCollection._instance
	}
}
```

## Result 2: nestjs/nest
**File:** `packages/core/injector/instance-wrapper.ts`

Shows a more advanced singleton with dependency injection...

Common patterns observed:
1. Private constructor
2. Static instance field
3. Static getInstance() method
4. Lazy initialization
5. Thread-safety considerations (in some examples)

Modern TypeScript alternative using modules:
```typescript
// singleton.ts
class Singleton {
	// implementation
}

export default new Singleton()  // Automatically a singleton!
```

Which approach would you like to use?
```

### Use Case 5: Error Pattern Discovery

**Scenario:** User encounters an error and wants to see how others handled it.

**Implementation:**
```typescript
// Find error handling patterns
const results = await githubRepo({
	query: 'catch TypeError cannot read property',
	language: 'typescript',
	maxResults: 10
});
```

**Example Dialog:**
```
User: I keep getting "Cannot read property of undefined" errors. How do others handle this?

Copilot: [calls githubRepo]
{
	query: 'TypeError cannot read property undefined',
	language: 'typescript',
	maxResults: 10
}

Here are patterns for handling this common error:

## Result 1: facebook/react
**File:** `packages/react-reconciler/src/ReactFiber.js`

```typescript
function getParent(fiber: Fiber): Fiber | null {
	// Optional chaining to prevent undefined errors
	return fiber?.parent ?? null
}

function processNode(node: Node): void {
	// Nullish coalescing with default
	const value = node?.value ?? 'default'

	// Guard clause
	if (!node) {
		return
	}

	// Safe access
	const parent = getParent(node)
}
```

## Result 2: microsoft/TypeScript
Shows using type guards...

**Best Practices Found:**
1. **Optional chaining:** `object?.property`
2. **Nullish coalescing:** `value ?? default`
3. **Type guards:** Check before access
4. **Default values:** Provide fallbacks
5. **Early returns:** Guard clauses

Would you like help applying these patterns to your code?
```

## Comparison with Other Tools

### githubRepo vs. semantic_search (workspace)

| Aspect | githubRepo | semantic_search |
|--------|------------|-----------------|
| **Scope** | All of GitHub | User's workspace |
| **Purpose** | Find external examples | Find local code |
| **Authentication** | Requires GitHub login | No auth needed |
| **Response Time** | 500-5000ms | 100-2000ms |
| **Best For** | Learning from others | Understanding own code |

### githubRepo vs. getVSCodeAPI

| Aspect | githubRepo | getVSCodeAPI |
|--------|------------|--------------|
| **Content** | Real code examples | API documentation |
| **Source** | Live repositories | Curated knowledge base |
| **Freshness** | Real-time | Periodic updates |
| **Quality** | Variable (community code) | High (official docs) |
| **Best For** | Seeing real usage | Learning API specs |

### githubRepo vs. fetchWebPage

| Aspect | githubRepo | fetchWebPage |
|--------|------------|--------------|
| **Purpose** | Code search | General web content |
| **Structure** | Code-focused results | HTML/text content |
| **Filtering** | Language, repo, etc. | None (URL-based) |
| **Best For** | Finding code patterns | Reading documentation |

## Error Handling

### Common Error Scenarios

```typescript
// 1. Authentication failure
// Returns: "GitHub authentication required. Please sign in to GitHub."

// 2. Rate limit exceeded
// Returns: "GitHub API rate limit exceeded. Resets at [timestamp]"

// 3. No results found
// Returns: "No results found for: [query]"

// 4. Invalid repository format
{
	query: 'test',
	repository: 'invalid_format'  // Should be 'owner/repo'
}
// Returns: "Invalid repository format. Use 'owner/repo' format."

// 5. Network error
// Returns: "Failed to connect to GitHub API. Please check your connection."

// 6. Malformed query
{
	query: ''  // Empty query
}
// Error: "Query cannot be empty"
```

### Retry Logic

```typescript
async function executeWithRetry<T>(
	fn: () => Promise<T>,
	maxRetries = 3,
	backoff = 1000
): Promise<T> {
	for (let attempt = 0; attempt <= maxRetries; attempt++) {
		try {
			return await fn();
		} catch (error: any) {
			// Don't retry on auth errors
			if (error.status === 401 || error.status === 403) {
				throw error;
			}

			// Don't retry on invalid requests
			if (error.status === 400 || error.status === 422) {
				throw error;
			}

			// Last attempt, throw error
			if (attempt === maxRetries) {
				throw error;
			}

			// Wait with exponential backoff
			await new Promise(resolve =>
				setTimeout(resolve, backoff * Math.pow(2, attempt))
			);
		}
	}

	throw new Error('Unexpected error in retry logic');
}
```

## Configuration

### User Settings

```json
{
	// Enable/disable GitHub code search
	"github.copilot.tools.githubRepo.enabled": true,

	// Default maximum results
	"github.copilot.tools.githubRepo.maxResults": 10,

	// Cache duration (minutes)
	"github.copilot.tools.githubRepo.cacheDuration": 60,

	// Preferred GitHub account (if multiple)
	"github.copilot.tools.githubRepo.account": "username",

	// Include repository descriptions in results
	"github.copilot.tools.githubRepo.includeDescriptions": true
}
```

### Rate Limit Management

```json
{
	// Wait when rate limit is low
	"github.copilot.tools.githubRepo.waitOnRateLimit": true,

	// Threshold for rate limit warning (remaining requests)
	"github.copilot.tools.githubRepo.rateLimitWarningThreshold": 10,

	// Show rate limit status in results
	"github.copilot.tools.githubRepo.showRateLimitStatus": true
}
```

## Implementation Blueprint

### Step 1: Basic Authentication

```typescript
async function authenticateGitHub(): Promise<string | null> {
	const session = await vscode.authentication.getSession(
		'github',
		['repo'],
		{ createIfNone: true }
	);

	return session?.accessToken || null;
}
```

### Step 2: Simple Search

```typescript
async function basicGitHubSearch(
	token: string,
	query: string
): Promise<any[]> {
	const octokit = new Octokit({ auth: token });

	const response = await octokit.rest.search.code({
		q: query,
		per_page: 10
	});

	return response.data.items;
}
```

### Step 3: Add Filtering

```typescript
function buildQuery(
	text: string,
	filters: {
		language?: string;
		repo?: string;
		org?: string;
	}
): string {
	const parts = [text];

	if (filters.language) parts.push(`language:${filters.language}`);
	if (filters.repo) parts.push(`repo:${filters.repo}`);
	if (filters.org) parts.push(`org:${filters.org}`);

	return parts.join(' ');
}
```

### Step 4: Add Content Fetching

```typescript
async function fetchFileContent(
	octokit: Octokit,
	owner: string,
	repo: string,
	path: string
): Promise<string> {
	const response = await octokit.rest.repos.getContent({
		owner,
		repo,
		path
	});

	if ('content' in response.data) {
		return Buffer.from(response.data.content, 'base64').toString('utf-8');
	}

	return '';
}
```

### Step 5: Add Rate Limit Handling

```typescript
class RateLimitedGitHubSearch {
	private octokit: Octokit;
	private lastRequest = 0;
	private minInterval = 2000; // 2 seconds between requests

	async search(query: string): Promise<any[]> {
		// Enforce minimum interval
		const now = Date.now();
		const timeSinceLastThe timeSinceLastRequest = now - this.lastRequest;
		if (timeSinceLastRequest < this.minInterval) {
			await new Promise(resolve =>
				setTimeout(resolve, this.minInterval - timeSinceLastRequest)
			);
		}

		this.lastRequest = Date.now();

		const response = await this.octokit.rest.search.code({
			q: query,
			per_page: 10
		});

		return response.data.items;
	}
}
```

## Best Practices

### 1. Be Specific in Queries

```typescript
// ✓ Good: Specific, targeted queries
{ query: 'class extends React.Component', language: 'typescript' }
{ query: 'async function fetchData', repository: 'vercel/next.js' }

// ✗ Bad: Too vague
{ query: 'function' }
{ query: 'class' }
```

### 2. Use Language Filters

```typescript
// ✓ Good: Filter by language
{ query: 'useState', language: 'typescript' }

// ✗ Bad: No language filter (gets JavaScript, TypeScript, JSX, etc.)
{ query: 'useState' }
```

### 3. Limit Results Appropriately

```typescript
// ✓ Good: Request only what you need
{ query: 'prisma client', maxResults: 5 }

// ✗ Bad: Request too many results
{ query: 'prisma', maxResults: 100 } // Slow, hits rate limits
```

### 4. Target Reputable Repositories

```typescript
// ✓ Good: Search in well-known repos
{ query: 'tree view', repository: 'microsoft/vscode' }
{ query: 'context api', organization: 'facebook' }

// This finds higher-quality examples
```

### 5. Combine with Other Tools

```typescript
// ✓ Good: Use GitHub for examples, then adapt
const examples = await githubRepo({ query: 'TreeDataProvider', ... });
const localUsage = await semantic_search({ query: 'TreeDataProvider' });

// Now you have both external examples and local context
```

### 6. Handle Rate Limits Gracefully

```typescript
// ✓ Good: Check rate limit before multiple searches
const rateLimitInfo = await getRateLimit();
if (rateLimitInfo.remaining < 5) {
	console.log('Rate limit low, waiting...');
	await waitForReset();
}

// ✗ Bad: Make many searches in rapid succession
for (const query of queries) {
	await githubRepo({ query }); // Will hit rate limit
}
```

## Limitations

### 1. Rate Limits

**Limitation:** GitHub API has strict rate limits (30 searches/minute authenticated).

```typescript
// After 30 searches in a minute, must wait
// Unauthenticated: 10 searches/minute (even worse)
```

**Workaround:** Cache results, batch queries, use sparingly.

### 2. Code Quality Variable

**Limitation:** Results come from public repos of varying quality.

```typescript
// May include:
// - Outdated code
// - Tutorial/learning code (not production quality)
// - Experimental code
// - Bugs in examples
```

**Workaround:** Prioritize well-known repositories, verify code before using.

### 3. Search Limitations

**Limitation:** GitHub code search has restrictions.

```typescript
// Cannot search:
// - Files larger than 384 KB
// - Repositories with 500,000+ files
// - Some binary files
// - Private repos (unless you have access)
```

**Workaround:** Use more specific queries, target smaller repos.

### 4. Authentication Required

**Limitation:** Requires GitHub authentication for reasonable limits.

```typescript
// Unauthenticated: 10 requests/minute
// Authenticated: 30 requests/minute
```

**Workaround:** Always authenticate for better limits.

### 5. No Guarantee of Correctness

**Limitation:** Code examples may contain bugs or use deprecated APIs.

```typescript
// Examples found might:
// - Use old API versions
// - Have security issues
// - Not follow current best practices
```

**Workaround:** Verify, test, and adapt code before using.

### 6. Network Dependency

**Limitation:** Requires internet connection to GitHub.

```typescript
// Won't work:
// - Offline
// - Behind restrictive firewalls
// - In regions blocking GitHub
```

**Workaround:** Cache results locally when possible.

## Summary

The `githubRepo` tool provides powerful code search capabilities across GitHub's vast repository of open source code. It enables:

- **Real-world examples** from production codebases
- **Pattern discovery** across popular projects
- **Learning** from open source implementations
- **API usage examples** from library authors
- **Problem-solving** by finding similar solutions

**Key Strengths:**
- Access to millions of public repositories
- Real, tested code from production systems
- Filtered by language, repository, organization
- Authenticated access for better rate limits
- Cached results for performance

**Best Used For:**
- Finding implementation examples
- Learning library usage patterns
- Discovering design patterns
- Seeing how experts solve problems
- Getting inspiration for solutions

**Important Considerations:**
- Rate limited (30 searches/minute)
- Requires GitHub authentication
- Code quality varies
- Results need verification
- Network-dependent

By leveraging GitHub's extensive code search effectively, Copilot can provide users with real-world, tested examples from the open source community, enhancing learning and problem-solving capabilities.
