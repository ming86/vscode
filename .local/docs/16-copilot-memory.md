# Copilot Memory Tool Documentation

## Overview

The `memory` tool provides persistent, session-scoped storage for Copilot to remember information across conversations. Unlike context that resets with each message, memory persists throughout the entire session, enabling Copilot to track long-term goals, user preferences, project-specific information, and conversation state.

**Primary Use Cases:**
- Remembering user preferences and coding style
- Tracking multi-step task progress
- Storing project-specific context and conventions
- Maintaining conversation continuity
- Recording decisions and their rationale

**Tool Category:** State Management & Persistence

**Security Model:** Session-scoped, user-isolated, no cross-session or cross-user access

## Tool Declaration

```typescript
{
	name: 'memory',
	description: 'Persistent memory system for storing and retrieving information across conversation turns. Supports creating, reading, updating, and deleting memory entries. Use this to remember user preferences, project context, ongoing tasks, or any information that should persist across the session.',
	inputSchema: {
		type: 'object',
		properties: {
			command: {
				type: 'string',
				enum: ['view', 'create', 'str_replace', 'insert', 'delete', 'rename'],
				description: 'The memory operation to perform'
			},
			key: {
				type: 'string',
				description: 'Unique identifier for the memory entry (required for all commands except view)'
			},
			content: {
				type: 'string',
				description: 'The content to store (required for create, used in str_replace and insert)'
			},
			oldString: {
				type: 'string',
				description: 'String to replace in str_replace command'
			},
			newString: {
				type: 'string',
				description: 'Replacement string in str_replace command'
			},
			insertLine: {
				type: 'number',
				description: 'Line number for insert command (1-indexed)'
			},
			newKey: {
				type: 'string',
				description: 'New key name for rename command'
			}
		},
		required: ['command']
	}
}
```

## Input Parameters

### command (required)
- **Type:** `'view' | 'create' | 'str_replace' | 'insert' | 'delete' | 'rename'`
- **Description:** The memory operation to perform

#### Available Commands:

1. **`view`** - List all memory entries or view specific entry
   - **Optional params:** `key`
   - **Returns:** All entries or specific entry content

2. **`create`** - Create new memory entry
   - **Required params:** `key`, `content`
   - **Behavior:** Fails if key already exists

3. **`str_replace`** - Update memory by replacing text
   - **Required params:** `key`, `oldString`, `newString`
   - **Behavior:** Replaces first occurrence of oldString

4. **`insert`** - Insert content at specific line
   - **Required params:** `key`, `content`, `insertLine`
   - **Behavior:** Inserts at line number (1-indexed)

5. **`delete`** - Delete memory entry
   - **Required params:** `key`
   - **Behavior:** Removes entry completely

6. **`rename`** - Rename memory entry key
   - **Required params:** `key`, `newKey`
   - **Behavior:** Changes key name, preserves content

### Examples

```typescript
// View all memory entries
{ command: 'view' }

// View specific entry
{ command: 'view', key: 'user_preferences' }

// Create new entry
{
	command: 'create',
	key: 'coding_style',
	content: 'User prefers:\n- Tabs over spaces\n- Single quotes\n- Arrow functions'
}

// Update existing entry
{
	command: 'str_replace',
	key: 'coding_style',
	oldString: 'Single quotes',
	newString: 'Double quotes for user-visible strings'
}

// Insert at specific line
{
	command: 'insert',
	key: 'todo_list',
	insertLine: 3,
	content: '- [ ] Add error handling to API'
}

// Rename entry
{
	command: 'rename',
	key: 'temp_notes',
	newKey: 'project_decisions'
}

// Delete entry
{ command: 'delete', key: 'outdated_info' }
```

## Architecture

### High-Level Flow

```
User Request → Memory Tool Handler
    ↓
Command Router
    ↓
Memory Store Access
    ↓
Session-Scoped Storage
    ↓
Security Validation
    ↓
Execute Operation
    ↓
Return Result
```

### Component Integration

```
┌─────────────────────────────────────────┐
│         Memory Tool Handler             │
│  - Command routing                      │
│  - Parameter validation                 │
└──────────────┬──────────────────────────┘
               ↓
┌─────────────────────────────────────────┐
│         Memory Store Service            │
│  - CRUD operations                      │
│  - Key management                       │
│  - Content serialization                │
└──────────────┬──────────────────────────┘
               ↓
┌─────────────────────────────────────────┐
│      Session Storage Provider           │
│  - Session-scoped isolation             │
│  - In-memory storage                    │
│  - Lifecycle management                 │
└─────────────────────────────────────────┘
```

### Security Model

```
┌─────────────────────────────────────────┐
│          Session Boundary               │
│                                          │
│  ┌────────────────────────────────┐    │
│  │   User Session A               │    │
│  │   - Memory Store A             │    │
│  │   - Isolated namespace         │    │
│  └────────────────────────────────┘    │
│                                          │
│  ┌────────────────────────────────┐    │
│  │   User Session B               │    │
│  │   - Memory Store B             │    │
│  │   - Isolated namespace         │    │
│  └────────────────────────────────┘    │
│                                          │
│  No cross-session access allowed        │
└─────────────────────────────────────────┘
```

## Internal Implementation

### Core Implementation Location

**File:** `src/extension/tools/builtinTools.ts`

### Memory Store Structure

```typescript
interface MemoryEntry {
	key: string;
	content: string;
	createdAt: number;
	updatedAt: number;
	metadata?: {
		type?: 'preference' | 'context' | 'task' | 'decision';
		tags?: string[];
	};
}

class MemoryStore {
	private entries: Map<string, MemoryEntry> = new Map();
	private sessionId: string;

	constructor(sessionId: string) {
		this.sessionId = sessionId;
	}

	// Core operations
	get(key: string): MemoryEntry | undefined;
	set(key: string, content: string): void;
	delete(key: string): boolean;
	has(key: string): boolean;
	keys(): string[];
	values(): MemoryEntry[];
	clear(): void;
}
```

### Command Implementation

```typescript
async function memory(
	args: {
		command: string;
		key?: string;
		content?: string;
		oldString?: string;
		newString?: string;
		insertLine?: number;
		newKey?: string;
	},
	context: ToolContext
): Promise<string> {
	const store = getMemoryStore(context.sessionId);

	switch (args.command) {
		case 'view':
			return handleViewCommand(store, args.key);

		case 'create':
			return handleCreateCommand(store, args.key!, args.content!);

		case 'str_replace':
			return handleStrReplaceCommand(
				store,
				args.key!,
				args.oldString!,
				args.newString!
			);

		case 'insert':
			return handleInsertCommand(
				store,
				args.key!,
				args.content!,
				args.insertLine!
			);

		case 'delete':
			return handleDeleteCommand(store, args.key!);

		case 'rename':
			return handleRenameCommand(store, args.key!, args.newKey!);

		default:
			throw new Error(`Unknown command: ${args.command}`);
	}
}
```

### View Command

```typescript
function handleViewCommand(store: MemoryStore, key?: string): string {
	if (key) {
		// View specific entry
		const entry = store.get(key);
		if (!entry) {
			return `Memory entry '${key}' not found`;
		}

		return formatEntry(entry);
	}

	// View all entries
	const entries = store.values();

	if (entries.length === 0) {
		return 'Memory is empty. No entries stored yet.';
	}

	return formatAllEntries(entries);
}

function formatEntry(entry: MemoryEntry): string {
	return `# ${entry.key}\n\n${entry.content}\n\n---\nCreated: ${new Date(entry.createdAt).toISOString()}\nUpdated: ${new Date(entry.updatedAt).toISOString()}`;
}

function formatAllEntries(entries: MemoryEntry[]): string {
	const sections = entries.map(entry => {
		const preview = entry.content.length > 100
			? entry.content.substring(0, 100) + '...'
			: entry.content;

		return `## ${entry.key}\n${preview}\n`;
	});

	return `# Memory Entries (${entries.length})\n\n${sections.join('\n')}`;
}
```

### Create Command

```typescript
function handleCreateCommand(
	store: MemoryStore,
	key: string,
	content: string
): string {
	// Validate key
	if (!key || key.trim().length === 0) {
		throw new Error('Key cannot be empty');
	}

	// Check if key already exists
	if (store.has(key)) {
		return `Memory entry '${key}' already exists. Use 'str_replace' or 'insert' to update, or 'delete' first.`;
	}

	// Validate content
	if (!content || content.trim().length === 0) {
		throw new Error('Content cannot be empty');
	}

	// Create entry
	const entry: MemoryEntry = {
		key,
		content,
		createdAt: Date.now(),
		updatedAt: Date.now()
	};

	store.set(key, entry);

	return `✓ Created memory entry '${key}' (${content.length} characters)`;
}
```

### String Replace Command

```typescript
function handleStrReplaceCommand(
	store: MemoryStore,
	key: string,
	oldString: string,
	newString: string
): string {
	// Get existing entry
	const entry = store.get(key);
	if (!entry) {
		return `Memory entry '${key}' not found. Use 'create' to add new entry.`;
	}

	// Validate strings
	if (!oldString) {
		throw new Error('oldString cannot be empty');
	}

	// Check if oldString exists in content
	if (!entry.content.includes(oldString)) {
		return `String not found in memory entry '${key}':\n"${oldString}"`;
	}

	// Replace first occurrence
	const newContent = entry.content.replace(oldString, newString);

	// Update entry
	entry.content = newContent;
	entry.updatedAt = Date.now();
	store.set(key, entry);

	return `✓ Updated memory entry '${key}'\nReplaced: "${oldString}"\nWith: "${newString}"`;
}
```

### Insert Command

```typescript
function handleInsertCommand(
	store: MemoryStore,
	key: string,
	content: string,
	insertLine: number
): string {
	// Get existing entry
	const entry = store.get(key);
	if (!entry) {
		return `Memory entry '${key}' not found. Use 'create' to add new entry.`;
	}

	// Validate line number
	const lines = entry.content.split('\n');
	if (insertLine < 1 || insertLine > lines.length + 1) {
		return `Invalid line number ${insertLine}. Entry has ${lines.length} lines.`;
	}

	// Insert content at specified line
	lines.splice(insertLine - 1, 0, content);
	const newContent = lines.join('\n');

	// Update entry
	entry.content = newContent;
	entry.updatedAt = Date.now();
	store.set(key, entry);

	return `✓ Inserted content into memory entry '${key}' at line ${insertLine}`;
}
```

### Delete Command

```typescript
function handleDeleteCommand(store: MemoryStore, key: string): string {
	if (!store.has(key)) {
		return `Memory entry '${key}' not found. Nothing to delete.`;
	}

	const entry = store.get(key);
	const preview = entry!.content.substring(0, 50);

	store.delete(key);

	return `✓ Deleted memory entry '${key}'\nContent preview: "${preview}..."`;
}
```

### Rename Command

```typescript
function handleRenameCommand(
	store: MemoryStore,
	oldKey: string,
	newKey: string
): string {
	// Validate old key exists
	if (!store.has(oldKey)) {
		return `Memory entry '${oldKey}' not found. Cannot rename.`;
	}

	// Validate new key doesn't exist
	if (store.has(newKey)) {
		return `Memory entry '${newKey}' already exists. Choose a different name.`;
	}

	// Validate new key
	if (!newKey || newKey.trim().length === 0) {
		throw new Error('New key cannot be empty');
	}

	// Get entry and rename
	const entry = store.get(oldKey)!;
	store.delete(oldKey);

	entry.key = newKey;
	entry.updatedAt = Date.now();
	store.set(newKey, entry);

	return `✓ Renamed memory entry '${oldKey}' → '${newKey}'`;
}
```

### Session Management

```typescript
class SessionMemoryManager {
	private stores: Map<string, MemoryStore> = new Map();

	getStore(sessionId: string): MemoryStore {
		if (!this.stores.has(sessionId)) {
			this.stores.set(sessionId, new MemoryStore(sessionId));
		}
		return this.stores.get(sessionId)!;
	}

	clearSession(sessionId: string): void {
		this.stores.delete(sessionId);
	}

	// Cleanup old sessions
	cleanupOldSessions(maxAgeMs: number): void {
		const now = Date.now();
		for (const [sessionId, store] of this.stores.entries()) {
			const lastActivity = store.getLastActivity();
			if (now - lastActivity > maxAgeMs) {
				this.stores.delete(sessionId);
			}
		}
	}
}

// Global singleton
const sessionManager = new SessionMemoryManager();

// Periodic cleanup (1 hour)
setInterval(() => {
	sessionManager.cleanupOldSessions(60 * 60 * 1000);
}, 60 * 60 * 1000);
```

## Dependencies

### VS Code APIs

```typescript
// No direct VS Code API dependencies
// Memory is purely in-memory storage
```

### Internal Dependencies

```typescript
import { ILanguageModelTool } from '../tools/toolsService';
import { CancellationToken } from '../../../platform/cancellation/common/cancellation';
```

### Storage Considerations

The memory tool uses in-memory storage with no disk persistence:

```typescript
// ✓ Persists during session
// ✓ Isolated per session
// ✗ Does NOT persist across VS Code restarts
// ✗ Does NOT sync across devices
```

## Performance Characteristics

### Response Time
- **All operations:** 1-10ms (in-memory access)
- **View all:** 5-20ms (depends on entry count)
- **Create/Update/Delete:** <5ms

### Scalability

```typescript
// Performance guidelines
const MAX_ENTRIES_PER_SESSION = 100;
const MAX_ENTRY_SIZE = 100_000; // 100KB per entry
const MAX_TOTAL_SIZE = 5_000_000; // 5MB per session

function validateSize(store: MemoryStore, newContent: string): void {
	// Check entry count
	if (store.size >= MAX_ENTRIES_PER_SESSION) {
		throw new Error(`Memory limit reached: ${MAX_ENTRIES_PER_SESSION} entries`);
	}

	// Check entry size
	if (newContent.length > MAX_ENTRY_SIZE) {
		throw new Error(`Entry too large: ${newContent.length} chars (max: ${MAX_ENTRY_SIZE})`);
	}

	// Check total size
	const totalSize = Array.from(store.values())
		.reduce((sum, entry) => sum + entry.content.length, 0);

	if (totalSize + newContent.length > MAX_TOTAL_SIZE) {
		throw new Error(`Total memory limit reached: ${totalSize} bytes`);
	}
}
```

### Memory Usage
- **Empty store:** ~1 KB
- **Typical usage (10-20 entries):** 50-200 KB
- **Maximum (100 entries):** ~5 MB

## Use Cases

### Use Case 1: User Preferences

**Scenario:** Remember user's coding style across conversation.

**Implementation:**
```typescript
// Initial conversation
User: "I prefer using tabs and arrow functions"

Copilot: [calls memory]
{
	command: 'create',
	key: 'coding_preferences',
	content: `# User Coding Preferences
- Indentation: Tabs (not spaces)
- Functions: Arrow functions preferred
- Noted at: ${new Date().toISOString()}`
}

// Later in conversation
User: "Create a helper function"

Copilot: [calls memory to check preferences]
{ command: 'view', key: 'coding_preferences' }

[Generates code with tabs and arrow functions]
```

**Example Dialog:**
```
User: I always use semicolons and prefer const over let

Copilot: [creates memory entry]
Got it! I'll remember you prefer:
- Semicolons at end of statements
- const over let where possible

[Stores in memory: 'coding_style']

---

[Later conversation]

User: Add a counter variable

Copilot: [checks memory]
```typescript
const counter = 0;
```
Used 'const' as per your preference. Changed to 'let' if you need mutability.
```

### Use Case 2: Multi-Step Task Tracking

**Scenario:** Track progress through complex refactoring task.

**Implementation:**
```typescript
// Start of task
User: "Help me refactor the authentication system"

Copilot: [creates task tracker]
{
	command: 'create',
	key: 'auth_refactor_progress',
	content: `# Authentication Refactoring Task

## Steps:
- [ ] 1. Extract token validation logic
- [ ] 2. Create authentication middleware
- [ ] 3. Update all route handlers
- [ ] 4. Add unit tests
- [ ] 5. Update documentation

## Current Step: 1
## Decisions Made:
## Files Modified:
`
}

// After completing step 1
Copilot: [updates progress]
{
	command: 'str_replace',
	key: 'auth_refactor_progress',
	oldString: '- [ ] 1. Extract token validation logic',
	newString: '- [x] 1. Extract token validation logic'
}

{
	command: 'str_replace',
	key: 'auth_refactor_progress',
	oldString: '## Current Step: 1',
	newString: '## Current Step: 2'
}
```

**Example Dialog:**
```
User: Let's refactor the user API to use async/await

Copilot: [creates task memory]
I'll help you refactor to async/await. Let me break this down:

Step 1: Convert getUserById ✓ (completed)
Step 2: Convert createUser (current)
Step 3: Convert updateUser
Step 4: Convert deleteUser
Step 5: Update error handling
Step 6: Add tests

[Completes step 2]

✓ Converted createUser to async/await
Next: updateUser function

[Stored progress in memory: 'api_refactor']

---

[Next day, user returns]

User: Continue with the refactoring

Copilot: [checks memory]
Welcome back! You were refactoring the user API to async/await.

Progress so far:
✓ getUserById
✓ createUser
→ Next: updateUser

Shall I continue?
```

### Use Case 3: Project Context

**Scenario:** Remember project-specific conventions and architecture.

**Implementation:**
```typescript
// Learning about project
User: "In this project we use a service layer pattern with dependency injection"

Copilot: [stores project context]
{
	command: 'create',
	key: 'project_architecture',
	content: `# Project Architecture

## Patterns:
- Service Layer Pattern
- Dependency Injection

## File Structure:
(to be filled as discovered)

## Naming Conventions:
(to be filled as discovered)

## Dependencies:
(to be filled as discovered)
`
}

// Later, discovering more
User: "Services go in src/services/ and are named with Service suffix"

Copilot: [updates project context]
{
	command: 'str_replace',
	key: 'project_architecture',
	oldString: '## File Structure:\n(to be filled as discovered)',
	newString: '## File Structure:\n- Services: src/services/*Service.ts'
}
```

**Example Dialog:**
```
User: This project uses Redux Toolkit and React Query

Copilot: [stores project context]
Noted! This project uses:
- Redux Toolkit for state management
- React Query for server state

I'll keep this in mind when suggesting code.

[Stores in memory: 'project_tech_stack']

---

[Later]

User: Add a feature to fetch user profile

Copilot: [checks memory]
Since you're using React Query, I'll create a query hook:

```typescript
// useUserProfile.ts
import { useQuery } from '@tanstack/react-query';

export function useUserProfile(userId: string) {
	return useQuery({
		queryKey: ['user', userId],
		queryFn: () => fetchUserProfile(userId)
	});
}
```

This follows your project's React Query pattern.
```

### Use Case 4: Decision Recording

**Scenario:** Track why certain implementation choices were made.

**Implementation:**
```typescript
// Making architectural decision
User: "Let's use a factory pattern for the API clients"

Copilot: [records decision]
{
	command: 'create',
	key: 'design_decisions',
	content: `# Design Decisions

## API Client Pattern (${new Date().toISOString()})
- Decision: Use Factory Pattern
- Reason: Need to create different client instances with different configs
- Files: src/api/factory.ts
- Alternatives considered: Singleton, Direct instantiation
`
}

// Later, adding more decisions
Copilot: [appends new decision]
{
	command: 'insert',
	key: 'design_decisions',
	insertLine: 7,
	content: `
## Error Handling Strategy (${new Date().toISOString()})
- Decision: Use custom error classes with error codes
- Reason: Need structured errors for API responses
- Files: src/errors/
- Alternatives considered: Plain Error objects, HTTP status only
`
}
```

**Example Dialog:**
```
User: Why did we decide to use Context API instead of Redux here?

Copilot: [checks memory]
{ command: 'view', key: 'design_decisions' }

According to our previous discussion:

Decision: Use Context API for theme state
Date: 2024-11-15
Reason: Theme state is simple (just light/dark mode) and doesn't need
Redux's complexity. Context API is sufficient and reduces bundle size.

Would you like me to explain anything else about this decision?
```

### Use Case 5: Learning User's Domain

**Scenario:** Build understanding of user's business domain over time.

**Implementation:**
```typescript
// Learning domain concepts
User: "In our app, a 'Campaign' has many 'Segments' and each Segment targets different user groups"

Copilot: [stores domain knowledge]
{
	command: 'create',
	key: 'domain_model',
	content: `# Business Domain Model

## Entities:

### Campaign
- Has many Segments
- (more details TBD)

### Segment
- Belongs to Campaign
- Targets specific user groups
- (more details TBD)

## Relationships:
- Campaign 1:N Segment
`
}

// Expanding knowledge
User: "Each Segment has engagement metrics like opens, clicks, and conversions"

Copilot: [updates domain model]
{
	command: 'str_replace',
	key: 'domain_model',
	oldString: '### Segment  \n- Belongs to Campaign\n- Targets specific user groups\n- (more details TBD)',
	newString: `### Segment
- Belongs to Campaign
- Targets specific user groups
- Tracks engagement metrics:
  - Opens
  - Clicks
  - Conversions`
}
```

## Comparison with Other Tools

### Memory vs. Context (Chat History)

| Aspect | Memory Tool | Chat Context |
|--------|-------------|--------------|
| **Persistence** | Explicit, controlled | Automatic |
| **Lifetime** | Full session | Last N messages |
| **Structure** | Organized by keys | Chronological |
| **Updates** | Can modify existing | Append-only |
| **Best For** | Long-term facts | Recent conversation |

### Memory vs. read_file

| Aspect | Memory | read_file |
|--------|--------|-----------|
| **Purpose** | Store ephemeral knowledge | Read code files |
| **Persistence** | Session only | Permanent on disk |
| **Editability** | Full CRUD operations | Read-only (tool) |
| **Best For** | Runtime state | Source code |

### Memory vs. Workspace Settings

| Aspect | Memory | VS Code Settings |
|--------|--------|------------------|
| **Scope** | Session | Global/workspace |
| **Persistence** | Lost on restart | Permanent |
| **Access** | Tool only | User + extensions |
| **Best For** | Conversation state | User preferences |

## Error Handling

### Common Error Scenarios

```typescript
// 1. Key not found
{ command: 'view', key: 'nonexistent' }
// Returns: "Memory entry 'nonexistent' not found"

// 2. Duplicate key
{ command: 'create', key: 'existing', content: '...' }
// Returns: "Memory entry 'existing' already exists..."

// 3. Empty content
{ command: 'create', key: 'test', content: '' }
// Error: "Content cannot be empty"

// 4. Invalid line number
{ command: 'insert', key: 'entry', insertLine: 999, content: '...' }
// Returns: "Invalid line number 999. Entry has 10 lines."

// 5. String not found
{ command: 'str_replace', key: 'entry', oldString: 'xyz', newString: 'abc' }
// Returns: "String not found in memory entry 'entry': 'xyz'"

// 6. Size limit exceeded
{ command: 'create', key: 'huge', content: 'x'.repeat(200000) }
// Error: "Entry too large: 200000 chars (max: 100000)"
```

### Error Recovery Patterns

```typescript
// Pattern 1: Check before create
const existing = await memory({ command: 'view', key: 'preferences' });
if (existing.includes('not found')) {
	await memory({ command: 'create', key: 'preferences', content: '...' });
} else {
	await memory({ command: 'str_replace', key: 'preferences', ... });
}

// Pattern 2: Delete and recreate
await memory({ command: 'delete', key: 'temp' });
await memory({ command: 'create', key: 'temp', content: '...' });

// Pattern 3: Try-create, fallback to update
const result = await memory({ command: 'create', key: 'data', content: '...' });
if (result.includes('already exists')) {
	await memory({ command: 'str_replace', key: 'data', ... });
}
```

## Configuration

### No User Configuration

The memory tool has no user-facing configuration. It operates with fixed constraints:

```typescript
// Hard-coded limits (not configurable)
const MEMORY_CONFIG = {
	maxEntriesPerSession: 100,
	maxEntrySize: 100_000, // 100KB
	maxTotalSize: 5_000_000, // 5MB
	sessionTimeoutMs: 24 * 60 * 60 * 1000, // 24 hours
	cleanupIntervalMs: 60 * 60 * 1000 // 1 hour
};
```

### Environment Variables (Development Only)

```bash
# For testing/development
COPILOT_MEMORY_MAX_ENTRIES=200
COPILOT_MEMORY_MAX_SIZE=10485760  # 10MB
COPILOT_MEMORY_DEBUG=true
```

## Implementation Blueprint

### Step 1: Basic Storage

```typescript
class SimpleMemoryStore {
	private data: Map<string, string> = new Map();

	get(key: string): string | undefined {
		return this.data.get(key);
	}

	set(key: string, value: string): void {
		this.data.set(key, value);
	}

	delete(key: string): boolean {
		return this.data.delete(key);
	}

	has(key: string): boolean {
		return this.data.has(key);
	}
}
```

### Step 2: Add Commands

```typescript
function executeCommand(
	store: SimpleMemoryStore,
	command: string,
	args: any
): string {
	switch (command) {
		case 'view':
			return args.key
				? store.get(args.key) || 'Not found'
				: Array.from(store.data.entries())
					.map(([k, v]) => `${k}: ${v}`)
					.join('\n');

		case 'create':
			if (store.has(args.key)) {
				return 'Already exists';
			}
			store.set(args.key, args.content);
			return 'Created';

		case 'delete':
			return store.delete(args.key) ? 'Deleted' : 'Not found';

		default:
			return 'Unknown command';
	}
}
```

### Step 3: Add Session Isolation

```typescript
class SessionMemoryStore {
	private sessions: Map<string, Map<string, string>> = new Map();

	getSession(sessionId: string): Map<string, string> {
		if (!this.sessions.has(sessionId)) {
			this.sessions.set(sessionId, new Map());
		}
		return this.sessions.get(sessionId)!;
	}

	execute(sessionId: string, command: string, args: any): string {
		const store = this.getSession(sessionId);
		return executeCommand(store, command, args);
	}
}
```

### Step 4: Add Rich Metadata

```typescript
interface RichMemoryEntry {
	key: string;
	content: string;
	createdAt: Date;
	updatedAt: Date;
	accessCount: number;
	tags: string[];
}

class RichMemoryStore {
	private entries: Map<string, RichMemoryEntry> = new Map();

	create(key: string, content: string, tags: string[] = []): void {
		this.entries.set(key, {
			key,
			content,
			createdAt: new Date(),
			updatedAt: new Date(),
			accessCount: 0,
			tags
		});
	}

	get(key: string): RichMemoryEntry | undefined {
		const entry = this.entries.get(key);
		if (entry) {
			entry.accessCount++;
		}
		return entry;
	}
}
```

### Step 5: Add Advanced Operations

```typescript
class AdvancedMemoryStore extends RichMemoryStore {
	// String replace
	strReplace(key: string, oldStr: string, newStr: string): boolean {
		const entry = this.get(key);
		if (!entry) return false;

		if (!entry.content.includes(oldStr)) {
			return false;
		}

		entry.content = entry.content.replace(oldStr, newStr);
		entry.updatedAt = new Date();
		return true;
	}

	// Insert at line
	insertAt(key: string, line: number, content: string): boolean {
		const entry = this.get(key);
		if (!entry) return false;

		const lines = entry.content.split('\n');
		if (line < 1 || line > lines.length + 1) {
			return false;
		}

		lines.splice(line - 1, 0, content);
		entry.content = lines.join('\n');
		entry.updatedAt = new Date();
		return true;
	}

	// Search by tag
	findByTag(tag: string): RichMemoryEntry[] {
		return Array.from(this.entries.values())
			.filter(entry => entry.tags.includes(tag));
	}
}
```

## Best Practices

### 1. Use Descriptive Keys

```typescript
// ✓ Good: Clear, descriptive keys
{ command: 'create', key: 'user_coding_preferences', content: '...' }
{ command: 'create', key: 'project_architecture', content: '...' }
{ command: 'create', key: 'task_refactor_auth_progress', content: '...' }

// ✗ Bad: Vague or generic keys
{ command: 'create', key: 'data', content: '...' }
{ command: 'create', key: 'temp', content: '...' }
{ command: 'create', key: 'x', content: '...' }
```

### 2. Structure Content for Updates

```typescript
// ✓ Good: Structured for easy updates
{
	command: 'create',
	key: 'task_progress',
	content: `# Task: Refactor Auth

## Steps:
- [ ] Step 1: Extract validation
- [ ] Step 2: Create middleware
- [ ] Step 3: Update routes

## Current: Step 1
`
}

// ✗ Bad: Unstructured text
{
	command: 'create',
	key: 'task',
	content: 'working on auth refactoring step 1'
}
```

### 3. Keep Entries Focused

```typescript
// ✓ Good: Separate concerns
{ command: 'create', key: 'coding_style', content: '...' }
{ command: 'create', key: 'project_architecture', content: '...' }
{ command: 'create', key: 'current_task', content: '...' }

// ✗ Bad: Everything in one entry
{
	command: 'create',
	key: 'everything',
	content: 'coding style: tabs, project: uses redux, task: refactoring...'
}
```

### 4. Check Before Acting

```typescript
// ✓ Good: Check if entry exists
const existing = await memory({ command: 'view', key: 'preferences' });
if (!existing.includes('not found')) {
	// Update existing
	await memory({ command: 'str_replace', ... });
} else {
	// Create new
	await memory({ command: 'create', ... });
}

// ✗ Bad: Assume entry exists/doesn't exist
await memory({ command: 'str_replace', ... }); // May fail if doesn't exist
```

### 5. Clean Up When Done

```typescript
// ✓ Good: Remove temporary entries
await memory({ command: 'create', key: 'temp_analysis', content: '...' });
// ... use the data ...
await memory({ command: 'delete', key: 'temp_analysis' });

// ✗ Bad: Leave temporary data indefinitely
await memory({ command: 'create', key: 'temp_1', content: '...' });
// ... never deleted ...
```

### 6. Use Timestamps

```typescript
// ✓ Good: Include timestamps in content
{
	command: 'create',
	key: 'decision_api_pattern',
	content: `# Decision: Use REST API
Date: ${new Date().toISOString()}
Reason: ...`
}

// This helps track when decisions were made
```

## Limitations

### 1. No Persistence Across Restarts

**Limitation:** Memory is cleared when VS Code restarts or extension reloads.

```typescript
// Before restart: Memory has 10 entries
// After restart: Memory is empty
```

**Workaround:** For truly persistent data, use VS Code's workspace or global state.

### 2. Session-Scoped Only

**Limitation:** Cannot share memory across different chat sessions or users.

```typescript
// Session A's memory is invisible to Session B
// No cross-session communication
```

**Workaround:** None by design (security/privacy feature).

### 3. Size Limits

**Limitation:** Limited to ~5MB total per session, 100KB per entry.

```typescript
// Can't store:
// - Large datasets
// - Complete file contents
// - Extensive logs
```

**Workaround:** Store only essential information, use references to files.

### 4. No Search or Filtering

**Limitation:** Can only view all entries or specific key, no search capability.

```typescript
// Can't do: "Find all entries containing 'authentication'"
// Must: View all entries and manually search
```

**Workaround:** Use structured keys with prefixes (e.g., `task_*`, `decision_*`).

### 5. No Versioning

**Limitation:** Updates overwrite previous content, no history.

```typescript
// Can't: View previous versions of an entry
// Can't: Undo an update
```

**Workaround:** Include version info in content itself if needed.

### 6. No Relationships

**Limitation:** Entries are independent, no linking or references between them.

```typescript
// Can't: "Show me all entries related to authentication"
// Must: Use naming conventions and manual tracking
```

**Workaround:** Use consistent key naming schemes.

### 7. String Replace Limitation

**Limitation:** `str_replace` only replaces first occurrence.

```typescript
const content = 'foo bar foo';
// str_replace('foo', 'baz') → 'baz bar foo'
// Not: 'baz bar baz'
```

**Workaround:** Multiple replace calls or use insert/delete pattern.

## Summary

The `memory` tool provides essential state management for Copilot conversations. It enables:

- **Preference tracking** across conversation turns
- **Task continuity** for multi-step workflows
- **Context accumulation** about projects and domains
- **Decision recording** for architectural choices
- **Long-term learning** about user's style and needs

**Key Strengths:**
- Persistent within session
- Full CRUD operations
- Fast in-memory access
- Session isolation for security
- Flexible content structure

**Best Used For:**
- User preferences and coding style
- Multi-step task tracking
- Project-specific conventions
- Design decision recording
- Domain knowledge accumulation

**Important Considerations:**
- Session-scoped (not persistent across restarts)
- Size limited (5MB per session)
- No cross-session sharing
- No built-in search or relationships

By following best practices and understanding limitations, the memory tool becomes a powerful mechanism for maintaining conversation context and enabling sophisticated multi-turn interactions in Copilot.
