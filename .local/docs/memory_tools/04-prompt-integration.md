# Prompt Integration

## Overview

Memory context is injected into the LLM prompt via two `@vscode/prompt-tsx` components rendered in the agent prompt hierarchy:

| Component | Prompt Position | Render Frequency | Gate |
|-----------|----------------|-----------------|------|
| `MemoryInstructionsPrompt` | System message | Every turn | `OmitBaseAgentInstructions` (outer) + both memory feature flags (inner — returns null if both `CopilotMemoryEnabled` and `MemoryToolEnabled` are disabled) |
| `MemoryContextPrompt` | User message (in `GlobalAgentContext`) | First turn only (`isNewChat`) | Feature flags |

Both components are defined in `src/extension/tools/node/memoryContextPrompt.tsx` and consumed in `src/extension/prompts/node/agent/agentPrompt.tsx`.

## Prompt Rendering Hierarchy

```
AgentPrompt.render()
├── SystemMessage
│   ├── "You are an expert AI programming assistant..."
│   ├── <CopilotIdentityRules />
│   ├── <SafetyRules />
│   └── <MemoryInstructionsPrompt />           ← Instructions (every turn)
│
├── ...other system/user messages...
│
└── UserMessage (GlobalAgentContext)
    ├── <environment_info> ... </environment_info>
    ├── <workspace_info> ... </workspace_info>
    ├── <UserPreferences />
    ├── <MemoryContextPrompt sessionResource={...} />  ← Content (first turn only)
    └── <cacheBreakpoint />
```

## `MemoryInstructionsPrompt`

Renders static usage instructions for the memory system. The LLM sees this on every turn of the conversation.

### Render Logic

```typescript
class MemoryInstructionsPrompt extends PromptElement<BasePromptElementProps> {
    async render(state, sizing) {
        const enableCopilotMemory = this.configurationService.getExperimentBasedConfig(
            ConfigKey.CopilotMemoryEnabled, this.experimentationService);
        const enableMemoryTool = this.configurationService.getExperimentBasedConfig(
            ConfigKey.MemoryToolEnabled, this.experimentationService);

        if (!enableCopilotMemory && !enableMemoryTool) {
            return null;  // Memory fully disabled — render nothing
        }

        return <Tag name='memoryInstructions'>...</Tag>;
    }
}
```

### Rendered XML Structure

```xml
<memoryInstructions>
As you work, consult your memory files to build on previous experience...

<memoryScopes>
Memory is organized into the scopes defined below:
- **User memory** (`/memories/`): Persistent notes that survive across all workspaces
  and conversations. First 200 lines are loaded into your context automatically.
- **Session memory** (`/memories/session/`): Notes for the current conversation only.
  Session files are listed in your context but not loaded automatically.
- **Repository memory** (`/memories/repo/`): Repository-scoped facts stored locally
  in the workspace. (OR "stored via Copilot" when CAPI enabled)
</memoryScopes>

<memoryGuidelines>
Guidelines for user memory (`/memories/`):
- Keep entries short and concise — use brief bullet points or single-line facts
- Organize by topic in separate files
- Record only key insights
- Update or remove memories that turn out to be wrong or outdated
- Do not create new files unless necessary — prefer updating existing files
Guidelines for session memory (`/memories/session/`):
- Use session memory to keep plans up to date
- Do not create unnecessary session memory files
</memoryGuidelines>

<repoMemoryInstructions>  <!-- Only when CAPI enabled -->
If you come across an important fact about the codebase...use the memory tool to
store it. Use the `create` command with a path under `/memories/repo/`.
The file content should be a JSON object with: subject, fact, citations, reason, category.

<examples>
- "Use ErrKind wrapper for every public API error"
- "Prefer ExpectNoLog helper over silent nil checks in tests"
...
</examples>

<factsCriteria>
- Are likely to have actionable implications for a future task
- Are independent of changes you are making as part of your current task
- Are unlikely to change over time
- Cannot always be inferred from a limited code sample
- Contain no secrets or sensitive data
</factsCriteria>

Note: Only `create` is supported for `/memories/repo/` paths.
</repoMemoryInstructions>
</memoryInstructions>
```

### Conditional Content

| Element | Condition |
|---------|-----------|
| User memory scope description | `enableMemoryTool` |
| Session memory scope description | `enableMemoryTool` |
| Repo memory (CAPI) description | `enableCopilotMemory` |
| Repo memory (local) description | `enableMemoryTool && !enableCopilotMemory` |
| `<memoryGuidelines>` | `enableMemoryTool` |
| `<repoMemoryInstructions>` | `enableCopilotMemory` |

## `MemoryContextPrompt`

Renders the actual memory content into the user message. Only runs on the first turn of a conversation (`isNewChat` check in the parent).

### Dependencies

```typescript
class MemoryContextPrompt extends PromptElement<MemoryContextPromptProps> {
    constructor(
        props,
        @IAgentMemoryService private readonly agentMemoryService,
        @IConfigurationService private readonly configurationService,
        @IExperimentationService private readonly experimentationService,
        @IVSCodeExtensionContext private readonly extensionContext,
        @IFileSystemService private readonly fileSystemService,
        @ITelemetryService private readonly telemetryService,
    ) { }
}
```

### Props

```typescript
interface MemoryContextPromptProps extends BasePromptElementProps {
    readonly sessionResource?: string;  // vscode-chat-session://local/<id>
}
```

### Render Logic

```typescript
async render() {
    const enableCopilotMemory = /* experiment check */;
    const enableMemoryTool = /* experiment check */;

    // Fetch all data in parallel
    const userMemoryContent = enableMemoryTool
        ? await this.getUserMemoryContent() : undefined;
    const sessionMemoryFiles = enableMemoryTool
        ? await this.getSessionMemoryFiles(this.props.sessionResource) : undefined;
    const repoMemories = enableCopilotMemory
        ? await this.agentMemoryService.getRepoMemories() : undefined;
    const localRepoMemoryFiles = (enableMemoryTool && !enableCopilotMemory)
        ? await this.getLocalRepoMemoryFiles() : undefined;

    if (!enableMemoryTool && !enableCopilotMemory) return null;

    // Send telemetry
    this._sendContextReadTelemetry(...);

    // Render XML tags
    return <>
        <Tag name='userMemory'>...</Tag>
        <Tag name='sessionMemory'>...</Tag>
        <Tag name='repoMemory'>...</Tag>           {/* local repo */}
        <Tag name='repository_memories'>...</Tag>   {/* CAPI repo */}
    </>;
}
```

### Rendered XML Structure

#### User Memory (populated)
```xml
<userMemory>
The following are your persistent user memory notes. These persist across all
workspaces and conversations.

## notes.md
- Prefer tabs over spaces
- Use US English in code

## debugging.md
- Always check compilation before running tests
</userMemory>
```

#### User Memory (empty)
```xml
<userMemory>
No user preferences or notes saved yet. Use the memory tool to store persistent
notes under /memories/.
</userMemory>
```

#### Session Memory (populated)
```xml
<sessionMemory>
The following files exist in your session memory (/memories/session/). Use the
memory tool to read them if needed.

/memories/session/plan.md
/memories/session/progress.md
</sessionMemory>
```

#### Session Memory (empty)
```xml
<sessionMemory>
Session memory (/memories/session/) is empty. No session notes have been created yet.
</sessionMemory>
```

#### Local Repo Memory (populated, CAPI disabled)
```xml
<repoMemory>
The following files exist in your repository memory (/memories/repo/). These are
scoped to the current workspace. Use the memory tool to read them if needed.

/memories/repo/conventions.md
/memories/repo/build-commands.md
</repoMemory>
```

#### CAPI Repo Memory (populated, CAPI enabled)
```xml
<repository_memories>
The following are recent memories stored for this repository from previous agent
interactions...

**TypeScript Conventions**
- Fact: Use tabs for indentation, PascalCase for types
- Citations: src/tsconfig.json, CONTRIBUTING.md
- Reason: Consistent codebase style

**Build Commands**
- Fact: Run `npm run compile` for development build
- Citations: package.json

Be sure to consider these stored facts carefully...
If you come across a memory that you find useful, store it again to retain it longer.
If you find a fact incorrect or outdated, store a new fact reflecting current reality.
</repository_memories>
```

### Data Fetching Methods

#### `getUserMemoryContent()`

1. Reads `globalStorageUri/memory-tool/memories/` directory
2. Filters to non-hidden files only
3. Reads each file's content, prepending `## <filename>` headers
4. **Caps at 200 lines** (`MAX_USER_MEMORY_LINES`)
5. Returns concatenated string or `undefined` if empty

#### `getSessionMemoryFiles(sessionResource?)`

1. Requires both `storageUri` and `sessionResource`
2. Extracts session ID from the resource URI
3. Lists files in `storageUri/memory-tool/memories/<sessionId>/`
4. Returns array of virtual paths (e.g., `/memories/session/plan.md`)
5. Does NOT read file content — only lists names

#### `getLocalRepoMemoryFiles()`

1. Lists files in `storageUri/memory-tool/memories/repo/`
2. Returns array of virtual paths (e.g., `/memories/repo/conventions.md`)
3. Does NOT read file content

#### `formatMemories(memories: RepoMemoryEntry[])`

Formats CAPI repo memories into markdown:

```typescript
private formatMemories(memories: RepoMemoryEntry[]): string {
    return memories.map(m => {
        const lines = [`**${m.subject}**`, `- Fact: ${m.fact}`];
        if (m.citations) {
            const arr = normalizeCitations(m.citations) ?? [];
            if (arr.length > 0) lines.push(`- Citations: ${arr.join(', ')}`);
        }
        if (m.reason) lines.push(`- Reason: ${m.reason}`);
        return lines.join('\n');
    }).join('\n\n');
}
```

## Integration Point in `agentPrompt.tsx`

```typescript
// Import
import { MemoryContextPrompt, MemoryInstructionsPrompt }
    from '../../../tools/node/memoryContextPrompt';

// System message (every turn)
<SystemMessage>
    ...
    <MemoryInstructionsPrompt />
</SystemMessage>

// User message (first turn only, inside GlobalAgentContext)
{this.props.isNewChat && <MemoryContextPrompt sessionResource={...} />}
```
