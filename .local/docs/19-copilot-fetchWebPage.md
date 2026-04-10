# Copilot fetchWebPage Tool Documentation

## Overview

The `fetchWebPage` tool enables Copilot to fetch and extract content from web pages, making external documentation, articles, API references, and online resources accessible during conversations. It implements a two-layer architecture with intelligent chunking, security controls, and content optimization for LLM consumption.

**Primary Use Cases:**
- Reading external documentation from URLs
- Fetching API reference pages
- Accessing blog posts and tutorials
- Reading release notes and changelogs
- Retrieving content from documentation sites

**Tool Category:** External Content Retrieval & Processing

## Tool Declaration

```typescript
{
	name: 'fetchWebPage',
	description: 'Fetch and extract main content from a web page. Useful for reading documentation, articles, API references, or any web-based content. Returns the main text content with structure preserved (headings, lists, code blocks). Handles pagination and chunking for large documents.',
	inputSchema: {
		type: 'object',
		properties: {
			url: {
				type: 'string',
				description: 'The URL of the web page to fetch. Must be a valid HTTP or HTTPS URL.',
				format: 'uri'
			},
			query: {
				type: 'string',
				description: 'Optional search query to focus on specific content within the page. When provided, returns relevant sections matching the query.'
			},
			maxChunks: {
				type: 'number',
				description: 'Maximum number of content chunks to return for large pages (default: 3, max: 10)',
				default: 3,
				minimum: 1,
				maximum: 10
			}
		},
		required: ['url']
	}
}
```

## Input Parameters

### url (required)
- **Type:** `string` (URI format)
- **Description:** The web page URL to fetch
- **Supported schemes:** `http://`, `https://`
- **Examples:**
  - `'https://code.visualstudio.com/api/references/vscode-api'`
  - `'https://react.dev/reference/react/useState'`
  - `'https://docs.github.com/en/rest/search'`

### query (optional)
- **Type:** `string`
- **Description:** Search query to find specific content within the page
- **Behavior:** When provided, returns only relevant sections matching the query
- **Examples:**
  - `'authentication'` - Find auth-related sections
  - `'installation steps'` - Find setup instructions
  - `'error codes'` - Find error documentation

### maxChunks (optional)
- **Type:** `number`
- **Default:** `3`
- **Range:** `1-10`
- **Description:** Limit number of chunks for large documents
- **Use cases:**
  - `1` - Get just the most relevant section
  - `3` - Balanced view of the document
  - `10` - Comprehensive content extraction

### Examples

```typescript
// Simple page fetch
{
	url: 'https://docs.github.com/en/rest/search'
}

// Fetch with focused query
{
	url: 'https://code.visualstudio.com/api',
	query: 'tree view provider'
}

// Large document with chunk limit
{
	url: 'https://www.typescriptlang.org/docs/handbook/intro.html',
	maxChunks: 5
}

// API documentation fetch
{
	url: 'https://react.dev/reference/react/hooks',
	query: 'useEffect'
}
```

## Architecture

### Two-Layer Architecture

```
Layer 1: Content Extraction
┌─────────────────────────────────────────┐
│      Raw HTML Fetch                     │
│  - HTTP request                         │
│  - Security validation                  │
│  - Encoding detection                   │
└──────────────┬──────────────────────────┘
               ↓
┌─────────────────────────────────────────┐
│      Content Extraction                 │
│  - Remove scripts, styles, ads          │
│  - Extract main content area            │
│  - Preserve structure (headers, lists)  │
│  - Convert to markdown                  │
└──────────────┬──────────────────────────┘
               ↓
┌─────────────────────────────────────────┐
│      Content Cleaning                   │
│  - Remove navigation                    │
│  - Strip footers/headers                │
│  - Normalize whitespace                 │
│  - Fix broken links                     │
└─────────────────────────────────────────┘

Layer 2: Chunking & Processing
┌─────────────────────────────────────────┐
│      Content Analysis                   │
│  - Detect document structure            │
│  - Identify sections                    │
│  - Calculate chunk boundaries           │
└──────────────┬──────────────────────────┘
               ↓
┌─────────────────────────────────────────┐
│      Intelligent Chunking               │
│  - Semantic boundary detection          │
│  - Section-aware splitting              │
│  - Preserve context across chunks       │
└──────────────┬──────────────────────────┘
               ↓
┌─────────────────────────────────────────┐
│      Query Matching (if query provided) │
│  - Semantic similarity                  │
│  - Keyword matching                     │
│  - Relevance scoring                    │
└──────────────┬──────────────────────────┘
               ↓
┌─────────────────────────────────────────┐
│      Result Formatting                  │
│  - Select top chunks                    │
│  - Add metadata                         │
│  - Format for LLM                       │
└─────────────────────────────────────────┘
```

### Security Model

```
┌─────────────────────────────────────────┐
│      Security Validation                │
│                                          │
│  1. URL Whitelist Check                 │
│     - Trusted domains                   │
│     - Documentation sites               │
│     - Public APIs                       │
│                                          │
│  2. Scheme Validation                   │
│     - Only HTTPS/HTTP                   │
│     - No file://, ftp://, etc.          │
│                                          │
│  3. Content-Type Validation             │
│     - text/html, text/plain             │
│     - No executables                    │
│                                          │
│  4. Size Limits                         │
│     - Max 10MB per page                 │
│     - Timeout after 30s                 │
│                                          │
│  5. Rate Limiting                       │
│     - Max 10 requests/minute            │
│     - Per-domain limits                 │
└─────────────────────────────────────────┘
```

## Internal Implementation

### Core Implementation Location

**File:** `src/extension/tools/builtinTools.ts`

### Implementation Code

```typescript
async function fetchWebPage(
	args: {
		url: string;
		query?: string;
		maxChunks?: number;
	},
	context: ToolContext,
	token: CancellationToken
): Promise<string> {
	// Validate URL
	if (!isValidUrl(args.url)) {
		return `Invalid URL: ${args.url}. Please provide a valid HTTP or HTTPS URL.`;
	}

	// Security check
	if (!isAllowedDomain(args.url)) {
		return `URL not allowed for security reasons. Only trusted documentation sites are accessible.`;
	}

	const maxChunks = Math.min(args.maxChunks || 3, 10);

	try {
		// Fetch and extract content
		const content = await fetchAndExtract(args.url, token);

		if (!content || content.trim().length === 0) {
			return `No content could be extracted from ${args.url}`;
		}

		// Chunk content
		const chunks = chunkContent(content, {
			maxChunkSize: 2000, // ~500 tokens
			preserveStructure: true
		});

		// Filter by query if provided
		let selectedChunks = chunks;
		if (args.query) {
			selectedChunks = await filterChunksByQuery(
				chunks,
				args.query,
				maxChunks
			);
		} else {
			selectedChunks = chunks.slice(0, maxChunks);
		}

		// Format result
		return formatWebPageResult(args.url, selectedChunks, chunks.length);
	} catch (error) {
		return handleFetchError(error, args.url);
	}
}
```

### URL Validation

```typescript
function isValidUrl(url: string): boolean {
	try {
		const parsed = new URL(url);
		return parsed.protocol === 'http:' || parsed.protocol === 'https:';
	} catch {
		return false;
	}
}

function isAllowedDomain(url: string): boolean {
	const parsed = new URL(url);
	const hostname = parsed.hostname.toLowerCase();

	// Whitelist of trusted domains
	const allowedDomains = [
		// Documentation sites
		'code.visualstudio.com',
		'developer.mozilla.org',
		'docs.github.com',
		'react.dev',
		'nodejs.org',
		'typescriptlang.org',
		'python.org',

		// Package registries
		'npmjs.com',
		'pypi.org',

		// Popular tech sites
		'stackoverflow.com',
		'github.com',

		// Blogs and tutorials
		'dev.to',
		'medium.com',
		'hashnode.dev'
	];

	// Check exact match or subdomain
	return allowedDomains.some(domain =>
		hostname === domain || hostname.endsWith(`.${domain}`)
	);
}
```

### Content Fetching and Extraction

```typescript
import axios from 'axios';
import * as cheerio from 'cheerio';
import TurndownService from 'turndown';

async function fetchAndExtract(
	url: string,
	token: CancellationToken
): Promise<string> {
	// Fetch HTML
	const response = await axios.get(url, {
		timeout: 30000, // 30 second timeout
		maxContentLength: 10 * 1024 * 1024, // 10MB limit
		headers: {
			'User-Agent': 'Mozilla/5.0 (compatible; VSCodeCopilot/1.0)',
			'Accept': 'text/html,application/xhtml+xml',
			'Accept-Language': 'en-US,en;q=0.9'
		},
		cancelToken: axios.CancelToken.source().token
	});

	// Check cancellation
	if (token.isCancellationRequested) {
		throw new Error('Fetch cancelled');
	}

	// Validate content type
	const contentType = response.headers['content-type'] || '';
	if (!contentType.includes('text/html') && !contentType.includes('text/plain')) {
		throw new Error(`Unsupported content type: ${contentType}`);
	}

	// Extract main content
	const html = response.data;
	const mainContent = extractMainContent(html);

	// Convert to markdown
	const markdown = convertToMarkdown(mainContent);

	return markdown;
}

function extractMainContent(html: string): string {
	const $ = cheerio.load(html);

	// Remove unwanted elements
	$('script, style, nav, header, footer, aside, .advertisement, #comments').remove();

	// Try to find main content area
	let mainContent = $('main, article, [role="main"], .content, .documentation').first();

	// Fallback to body if no main content found
	if (mainContent.length === 0) {
		mainContent = $('body');
	}

	// Get HTML of main content
	return mainContent.html() || '';
}

function convertToMarkdown(html: string): string {
	const turndownService = new TurndownService({
		headingStyle: 'atx',
		codeBlockStyle: 'fenced',
		bulletListMarker: '-'
	});

	// Custom rules for better conversion
	turndownService.addRule('codeBlocks', {
		filter: ['pre'],
		replacement: (content, node) => {
			const codeElement = node.querySelector('code');
			const language = codeElement?.className.match(/language-(\w+)/)?.[1] || '';
			return `\n\`\`\`${language}\n${content.trim()}\n\`\`\`\n`;
		}
	});

	const markdown = turndownService.turndown(html);

	// Clean up excessive whitespace
	return markdown
		.replace(/\n{3,}/g, '\n\n')
		.trim();
}
```

### Intelligent Chunking

```typescript
interface ContentChunk {
	content: string;
	startLine: number;
	endLine: number;
	heading?: string;
	score?: number;
}

function chunkContent(
	content: string,
	options: {
		maxChunkSize: number;
		preserveStructure: boolean;
	}
): ContentChunk[] {
	const chunks: ContentChunk[] = [];
	const lines = content.split('\n');

	let currentChunk: string[] = [];
	let currentHeading: string | undefined;
	let startLine = 0;

	for (let i = 0; i < lines.length; i++) {
		const line = lines[i];

		// Detect section boundaries (headings)
		const headingMatch = line.match(/^(#{1,6})\s+(.+)$/);

		if (headingMatch && options.preserveStructure) {
			// Save previous chunk if it exists
			if (currentChunk.length > 0) {
				chunks.push({
					content: currentChunk.join('\n'),
					startLine: startLine,
					endLine: i - 1,
					heading: currentHeading
				});
			}

			// Start new chunk
			currentChunk = [line];
			currentHeading = headingMatch[2];
			startLine = i;
		} else {
			currentChunk.push(line);

			// Check if chunk is getting too large
			const chunkText = currentChunk.join('\n');
			if (chunkText.length >= options.maxChunkSize) {
				chunks.push({
					content: chunkText,
					startLine: startLine,
					endLine: i,
					heading: currentHeading
				});

				currentChunk = [];
				currentHeading = undefined;
				startLine = i + 1;
			}
		}
	}

	// Add final chunk
	if (currentChunk.length > 0) {
		chunks.push({
			content: currentChunk.join('\n'),
			startLine: startLine,
			endLine: lines.length - 1,
			heading: currentHeading
		});
	}

	return chunks;
}
```

### Query-Based Filtering

```typescript
async function filterChunksByQuery(
	chunks: ContentChunk[],
	query: string,
	maxResults: number
): Promise<ContentChunk[]> {
	// Score each chunk by relevance to query
	const scoredChunks = await Promise.all(
		chunks.map(async chunk => {
			const score = await calculateRelevanceScore(chunk.content, query);
			return { ...chunk, score };
		})
	);

	// Sort by score (descending) and take top results
	return scoredChunks
		.sort((a, b) => (b.score || 0) - (a.score || 0))
		.slice(0, maxResults);
}

async function calculateRelevanceScore(
	content: string,
	query: string
): Promise<number> {
	const queryLower = query.toLowerCase();
	const contentLower = content.toLowerCase();

	let score = 0;

	// Keyword matching
	const queryWords = queryLower.split(/\s+/);
	for (const word of queryWords) {
		const occurrences = (contentLower.match(new RegExp(word, 'g')) || []).length;
		score += occurrences * 10;
	}

	// Heading bonus
	if (content.match(/^#{1,6}/m)) {
		const headings = content.match(/^#{1,6}\s+.+$/gm) || [];
		for (const heading of headings) {
			if (heading.toLowerCase().includes(queryLower)) {
				score += 50;
			}
		}
	}

	// Exact phrase match bonus
	if (contentLower.includes(queryLower)) {
		score += 100;
	}

	// Could use embedding similarity here for semantic matching
	// const embeddingScore = await calculateSemanticSimilarity(content, query);
	// score += embeddingScore * 10;

	return score;
}
```

### Result Formatting

```typescript
function formatWebPageResult(
	url: string,
	chunks: ContentChunk[],
	totalChunks: number
): string {
	const sections: string[] = [];

	// Header
	sections.push(`# Web Page Content: ${url}`);
	sections.push(`Showing ${chunks.length} of ${totalChunks} sections\n`);

	// Format each chunk
	for (let i = 0; i < chunks.length; i++) {
		const chunk = chunks[i];

		sections.push(`## Section ${i + 1}${chunk.heading ? `: ${chunk.heading}` : ''}`);
		sections.push(chunk.content);
		sections.push(''); // Empty line between sections
	}

	// Footer with metadata
	if (chunks.length < totalChunks) {
		sections.push(`\n---`);
		sections.push(`*Note: This page has ${totalChunks} sections total. Use a more specific query or increase maxChunks to see more content.*`);
	}

	sections.push(`\n[→ View full page](${url})`);

	return sections.join('\n');
}
```

### Caching Strategy

```typescript
interface CacheEntry {
	url: string;
	content: string;
	chunks: ContentChunk[];
	timestamp: number;
	etag?: string;
}

class WebPageCache {
	private cache: Map<string, CacheEntry> = new Map();
	private readonly TTL = 60 * 60 * 1000; // 1 hour

	async get(url: string): Promise<CacheEntry | undefined> {
		const entry = this.cache.get(url);

		if (!entry) return undefined;

		// Check if expired
		if (Date.now() - entry.timestamp > this.TTL) {
			this.cache.delete(url);
			return undefined;
		}

		return entry;
	}

	set(url: string, content: string, chunks: ContentChunk[], etag?: string): void {
		this.cache.set(url, {
			url,
			content,
			chunks,
			timestamp: Date.now(),
			etag
		});
	}

	async fetchWithCache(url: string): Promise<CacheEntry> {
		const cached = await this.get(url);

		if (cached) {
			// Try conditional request with ETag
			if (cached.etag) {
				const isModified = await checkIfModified(url, cached.etag);
				if (!isModified) {
					return cached;
				}
			} else {
				// No ETag, use cached version if not expired
				return cached;
			}
		}

		// Fetch fresh content
		const response = await fetchAndExtract(url);
		const chunks = chunkContent(response, { maxChunkSize: 2000, preserveStructure: true });
		this.set(url, response, chunks);

		return { url, content: response, chunks, timestamp: Date.now() };
	}
}
```

### Rate Limiting

```typescript
class RateLimiter {
	private requests: Map<string, number[]> = new Map();
	private readonly maxRequestsPerMinute = 10;
	private readonly window = 60 * 1000; // 1 minute

	async checkLimit(domain: string): Promise<boolean> {
		const now = Date.now();
		const requests = this.requests.get(domain) || [];

		// Remove old requests outside window
		const recentRequests = requests.filter(time => now - time < this.window);

		if (recentRequests.length >= this.maxRequestsPerMinute) {
			const oldestRequest = Math.min(...recentRequests);
			const waitTime = this.window - (now - oldestRequest);

			throw new Error(
				`Rate limit exceeded for ${domain}. ` +
				`Please wait ${Math.ceil(waitTime / 1000)} seconds.`
			);
		}

		// Add current request
		recentRequests.push(now);
		this.requests.set(domain, recentRequests);

		return true;
	}
}
```

## Dependencies

### External Dependencies

```typescript
// HTTP client
import axios from 'axios';

// HTML parsing
import * as cheerio from 'cheerio';

// HTML to Markdown conversion
import TurndownService from 'turndown';

// Optional: Readability extraction
import { Readability } from '@mozilla/readability';
import { JSDOM } from 'jsdom';
```

### Internal Dependencies

```typescript
import { ILanguageModelTool } from '../tools/toolsService';
import { CancellationToken } from '../../../platform/cancellation/common/cancellation';
import { IEmbeddingService } from '../../../platform/embedding/common/embedding';
```

### VS Code APIs

```typescript
import * as vscode from 'vscode';

// Used for:
// - Settings (allowed domains list)
// - Output channel (logging)
// - Progress notifications (long fetches)
```

## Performance Characteristics

### Response Time
- **Cached pages:** 10-50ms
- **Small pages (<100KB):** 500-2000ms
- **Large pages (1-5MB):** 2000-5000ms
- **Very large pages (>5MB):** 5000-10000ms (may timeout)

### Optimization Strategies

```typescript
const OPTIMIZATION_CONFIG = {
	// Timeouts
	fetchTimeout: 30000, // 30 seconds
	chunkingTimeout: 5000, // 5 seconds

	// Size limits
	maxPageSize: 10 * 1024 * 1024, // 10MB
	maxChunkSize: 2000, // characters (~500 tokens)

	// Caching
	cacheTTL: 60 * 60 * 1000, // 1 hour
	maxCacheSize: 100, // pages

	// Concurrency
	maxConcurrentFetches: 3,

	// Rate limiting
	maxRequestsPerMinute: 10,
	maxRequestsPerDomain: 5
};
```

### Memory Usage
- **Per page (cached):** 50-500 KB (depends on content)
- **Cache (100 pages):** 5-50 MB
- **Parsing overhead:** 10-50 MB during extraction

## Use Cases

### Use Case 1: Reading Documentation

**Scenario:** User asks about a specific API from external documentation.

**Implementation:**
```typescript
// User asks: "How does the GitHub REST API search work?"
const result = await fetchWebPage({
	url: 'https://docs.github.com/en/rest/search',
	query: 'search repositories'
});
```

**Example Dialog:**
```
User: How do I search GitHub repositories using their API?

Copilot: [calls fetchWebPage]
{
	url: 'https://docs.github.com/en/rest/search',
	query: 'search repositories'
}

# Web Page Content: https://docs.github.com/en/rest/search
Showing 2 of 5 sections

## Section 1: Search repositories

You can search for repositories via the GitHub API:

```
GET /search/repositories
```

Parameters:
- `q` (required): The search query
- `sort`: Sort by stars, forks, updated
- `order`: asc or desc

Example:
```bash
curl https://api.github.com/search/repositories?q=language:typescript
```

## Section 2: Search query syntax

The search query supports qualifiers:
- `language:javascript` - Filter by language
- `stars:>1000` - Repository stars
- `user:username` - By user

Based on this documentation, here's how to search repositories:

```typescript
const response = await fetch(
	'https://api.github.com/search/repositories?q=language:typescript+stars:>1000'
);
const data = await response.json();
```

Would you like me to help implement this?
```

### Use Case 2: Tutorial Following

**Scenario:** User wants to follow a tutorial from a blog post.

**Implementation:**
```typescript
// Fetch tutorial content
const result = await fetchWebPage({
	url: 'https://dev.to/author/building-vscode-extension',
	maxChunks: 5
});
```

**Example Dialog:**
```
User: Can you help me follow this tutorial? https://dev.to/author/building-vscode-extension

Copilot: [calls fetchWebPage]
{
	url: 'https://dev.to/author/building-vscode-extension',
	maxChunks: 5
}

I've read the tutorial. It covers:

1. **Setting up the project** - Using yo code generator
2. **Creating the extension** - Basic structure
3. **Adding commands** - Command registration
4. **Testing** - Running in Extension Development Host
5. **Publishing** - To VS Code Marketplace

Let's start with step 1. Would you like me to:
1. Generate the initial project structure?
2. Explain any specific part?
3. Help you implement a feature?
```

### Use Case 3: API Reference Lookup

**Scenario:** User needs details about a specific method from API docs.

**Implementation:**
```typescript
// Fetch React documentation
const result = await fetchWebPage({
	url: 'https://react.dev/reference/react/hooks',
	query: 'useEffect cleanup'
});
```

**Example Dialog:**
```
User: How do I cleanup in useEffect?

Copilot: [calls fetchWebPage]
{
	url: 'https://react.dev/reference/react/hooks',
	query: 'useEffect cleanup'
}

According to React's official documentation:

## useEffect cleanup

To cleanup in useEffect, return a function from your effect:

```javascript
useEffect(() => {
	// Setup
	const subscription = source.subscribe();

	// Cleanup function
	return () => {
		subscription.unsubscribe();
	};
}, [source]);
```

The cleanup function runs:
1. Before the effect runs again (on re-render)
2. When the component unmounts

Common use cases:
- Canceling API requests
- Unsubscribing from subscriptions
- Clearing timers
- Removing event listeners

Would you like an example for your specific use case?
```

### Use Case 4: Release Notes Reading

**Scenario:** User wants to know what's new in a library version.

**Implementation:**
```typescript
// Fetch changelog
const result = await fetchWebPage({
	url: 'https://github.com/microsoft/vscode/blob/main/CHANGELOG.md',
	query: 'new features'
});
```

**Example Dialog:**
```
User: What's new in the latest VS Code release?

Copilot: [calls fetchWebPage]
{
	url: 'https://code.visualstudio.com/updates/v1_85',
	maxChunks: 3
}

# VS Code 1.85 Release Highlights

## New Features

**1. Inline Chat Improvements**
- New inline suggestions while typing
- Better context understanding
- Faster response times

**2. Testing UI Enhancements**
- Test Explorer redesign
- Better coverage visualization
- Improved debugging integration

**3. Terminal Updates**
- Shell integration improvements
- Better performance
- New command detection

Would you like details about any specific feature?
```

### Use Case 5: Troubleshooting Research

**Scenario:** User encountered an error and wants to read about it.

**Implementation:**
```typescript
// Fetch Stack Overflow answer
const result = await fetchWebPage({
	url: 'https://stackoverflow.com/questions/12345/error-message',
	query: 'solution'
});
```

**Example Dialog:**
```
User: I found this Stack Overflow answer about my error: [URL]

Copilot: [calls fetchWebPage]
{
	url: 'https://stackoverflow.com/questions/12345/...',
	query: 'solution'
}

I've read the Stack Overflow thread. The accepted answer suggests:

**Problem:** The error occurs because...

**Solution:**
```typescript
// Instead of:
const data = JSON.parse(undefined);

// Do this:
const data = response ? JSON.parse(response) : null;
```

**Why this works:**
- Checks if response exists before parsing
- Handles undefined/null cases
- Prevents JSON.parse error

Would you like me to apply this fix to your code?
```

## Comparison with Other Tools

### fetchWebPage vs. githubRepo

| Aspect | fetchWebPage | githubRepo |
|--------|--------------|------------|
| **Purpose** | Fetch web page content | Search GitHub code |
| **Input** | URL | Search query |
| **Output** | Page content (markdown) | Code snippets + metadata |
| **Best For** | Reading documentation | Finding code examples |

### fetchWebPage vs. getVSCodeAPI

| Aspect | fetchWebPage | getVSCodeAPI |
|--------|--------------|--------------|
| **Data Source** | External web pages | Curated knowledge base |
| **Freshness** | Real-time | Periodic updates |
| **Scope** | Any allowed domain | VS Code API only |
| **Best For** | Latest external docs | VS Code development |

### fetchWebPage vs. semantic_search

| Aspect | fetchWebPage | semantic_search |
|--------|--------------|-----------------|
| **Scope** | External web | User's workspace |
| **Purpose** | Read external content | Search local code |
| **Network** | Requires internet | Offline capable |
| **Best For** | External resources | Local code |

## Error Handling

### Common Error Scenarios

```typescript
// 1. Invalid URL
{ url: 'not-a-url' }
// Returns: "Invalid URL: not-a-url. Please provide a valid HTTP or HTTPS URL."

// 2. Blocked domain
{ url: 'https://untrusted-site.com/page' }
// Returns: "URL not allowed for security reasons..."

// 3. Network timeout
{ url: 'https://very-slow-site.com/page' }
// Returns: "Failed to fetch page: Request timeout after 30 seconds"

// 4. Page not found
{ url: 'https://docs.github.com/nonexistent' }
// Returns: "Failed to fetch page: 404 Not Found"

// 5. No content extracted
{ url: 'https://image-only-page.com' }
// Returns: "No content could be extracted from [URL]"

// 6. Rate limit exceeded
// Multiple requests to same domain
// Returns: "Rate limit exceeded for domain. Please wait X seconds."
```

### Retry Logic

```typescript
async function fetchWithRetry(
	url: string,
	maxRetries = 2
): Promise<string> {
	for (let attempt = 0; attempt <= maxRetries; attempt++) {
		try {
			return await fetchAndExtract(url);
		} catch (error: any) {
			// Don't retry on client errors
			if (error.response?.status >= 400 && error.response?.status < 500) {
				throw error;
			}

			// Don't retry on security errors
			if (error.message.includes('not allowed')) {
				throw error;
			}

			// Last attempt
			if (attempt === maxRetries) {
				throw error;
			}

			// Wait with exponential backoff
			await new Promise(resolve =>
				setTimeout(resolve, 1000 * Math.pow(2, attempt))
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
	// Enable/disable web page fetching
	"github.copilot.tools.fetchWebPage.enabled": true,

	// Allowed domains (in addition to defaults)
	"github.copilot.tools.fetchWebPage.allowedDomains": [
		"mycompany.com",
		"internal-docs.example.com"
	],

	// Cache settings
	"github.copilot.tools.fetchWebPage.cacheDuration": 60,
	"github.copilot.tools.fetchWebPage.maxCacheSize": 100,

	// Fetch settings
	"github.copilot.tools.fetchWebPage.timeout": 30000,
	"github.copilot.tools.fetchWebPage.maxPageSize": 10485760,

	// Rate limiting
	"github.copilot.tools.fetchWebPage.maxRequestsPerMinute": 10,

	// Content extraction
	"github.copilot.tools.fetchWebPage.preserveCodeBlocks": true,
	"github.copilot.tools.fetchWebPage.preserveLinks": true
}
```

### Domain Whitelist Management

```typescript
// Default allowed domains
const DEFAULT_ALLOWED_DOMAINS = [
	'code.visualstudio.com',
	'developer.mozilla.org',
	// ... more defaults
];

// Get custom domains from settings
function getAllowedDomains(): string[] {
	const config = vscode.workspace.getConfiguration('github.copilot.tools.fetchWebPage');
	const custom = config.get<string[]>('allowedDomains', []);

	return [...DEFAULT_ALLOWED_DOMAINS, ...custom];
}
```

## Implementation Blueprint

### Step 1: Basic Fetch

```typescript
import axios from 'axios';

async function basicFetch(url: string): Promise<string> {
	const response = await axios.get(url, {
		timeout: 30000
	});

	return response.data;
}
```

### Step 2: Add Content Extraction

```typescript
import * as cheerio from 'cheerio';

function extractContent(html: string): string {
	const $ = cheerio.load(html);

	// Remove unwanted elements
	$('script, style, nav, footer').remove();

	// Get main content
	const main = $('main, article, .content').first();

	return main.text();
}
```

### Step 3: Add Markdown Conversion

```typescript
import TurndownService from 'turndown';

function convertToMarkdown(html: string): string {
	const turndown = new TurndownService();
	return turndown.turndown(html);
}
```

### Step 4: Add Chunking

```typescript
function chunkByHeadings(markdown: string): string[] {
	const chunks: string[] = [];
	let current = '';

	for (const line of markdown.split('\n')) {
		if (line.match(/^#{1,6}\s/)) {
			if (current) chunks.push(current);
			current = line + '\n';
		} else {
			current += line + '\n';
		}
	}

	if (current) chunks.push(current);
	return chunks;
}
```

### Step 5: Add Query Filtering

```typescript
function filterByQuery(chunks: string[], query: string): string[] {
	const queryLower = query.toLowerCase();

	return chunks
		.filter(chunk => chunk.toLowerCase().includes(queryLower))
		.slice(0, 3);
}
```

## Best Practices

### 1. Always Validate URLs

```typescript
// ✓ Good: Validate before fetching
if (!isValidUrl(url)) {
	return 'Invalid URL';
}

// ✗ Bad: Fetch without validation
await fetchWebPage({ url: userInput });
```

### 2. Use Query Parameter for Large Pages

```typescript
// ✓ Good: Focus on relevant content
{
	url: 'https://docs.github.com/en/rest',
	query: 'authentication',
	maxChunks: 3
}

// ✗ Bad: Fetch entire large page
{
	url: 'https://docs.github.com/en/rest',
	maxChunks: 10  // Returns too much content
}
```

### 3. Respect Rate Limits

```typescript
// ✓ Good: Space out requests
await fetchWebPage({ url: url1 });
await new Promise(resolve => setTimeout(resolve, 1000));
await fetchWebPage({ url: url2 });

// ✗ Bad: Rapid successive requests
await Promise.all(urls.map(url => fetchWebPage({ url })));
```

### 4. Combine with Other Tools

```typescript
// ✓ Good: Use web content as context for local work
const docs = await fetchWebPage({ url: apiDocsUrl });
const localCode = await semantic_search({ query: 'API usage' });

// Now have both external docs and local implementation
```

### 5. Handle Errors Gracefully

```typescript
// ✓ Good: Provide fallback
const result = await fetchWebPage({ url });
if (result.includes('Failed to fetch')) {
	// Offer alternative or cached content
	return 'Unable to fetch. Here's what I remember about this...';
}
```

## Limitations

### 1. Domain Whitelist Restriction

**Limitation:** Only trusted domains are allowed for security.

```typescript
// Can fetch: https://code.visualstudio.com/api
// Cannot fetch: https://random-untrusted-site.com
```

**Workaround:** Add trusted domains to user settings.

### 2. No JavaScript Execution

**Limitation:** Cannot handle dynamically loaded content.

```typescript
// Works: Static HTML content
// Doesn't work: Content loaded by JavaScript after page load
```

**Workaround:** Use APIs directly when available instead of scraping.

### 3. Size and Timeout Limits

**Limitation:** Large pages may timeout or be truncated.

```typescript
// Max size: 10MB
// Timeout: 30 seconds
```

**Workaround:** Use query parameter to focus on specific sections.

### 4. No Authentication

**Limitation:** Cannot access pages requiring login.

```typescript
// Works: Public documentation
// Doesn't work: Pages behind authentication
```

**Workaround:** Use API tokens with dedicated API tools instead.

### 5. Extraction Quality Varies

**Limitation:** Content extraction works better on some sites than others.

```typescript
// Excellent: Well-structured documentation sites
// Poor: Complex layouts, heavy JavaScript, non-standard HTML
```

**Workaround:** Use sites with clean, semantic HTML when possible.

### 6. No Live Updates

**Limitation:** Cached content may be stale (1 hour TTL).

```typescript
// May not reflect: Very recent updates to documentation
```

**Workaround:** Clear cache or wait for TTL expiration for latest content.

## Summary

The `fetchWebPage` tool brings external web content into Copilot conversations through intelligent extraction and processing. It enables:

- **Documentation access** from trusted sites
- **Tutorial following** with structured content
- **API reference** lookups
- **Release notes** reading
- **Troubleshooting** research from Stack Overflow and blogs

**Key Strengths:**
- Two-layer architecture (extraction + chunking)
- Intelligent content extraction
- Query-based section filtering
- Security controls with domain whitelist
- Cached for performance
- Markdown formatting for LLMs

**Best Used For:**
- Reading external documentation
- Following online tutorials
- Looking up API references
- Researching solutions
- Staying updated with release notes

**Important Considerations:**
- Domain whitelist for security
- Rate limited (10 requests/minute)
- No JavaScript execution
- Size/timeout limits
- Cache may be stale (1 hour)

By leveraging this tool effectively, Copilot can seamlessly integrate external knowledge into conversations, providing users with comprehensive answers that combine local code context with authoritative external documentation.
