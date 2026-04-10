````markdown
# runSubagent Tool Documentation

## Overview

**Tool Name:** `runSubagent`

**Purpose:** Launches a new agent to handle complex, multi-step tasks autonomously within an isolated subagent context. Enables efficient organization of tasks and context window management by delegating work to a separate agent invocation.

**Implementation Location:** `vscode/src/vs/workbench/contrib/chat/common/tools/runSubagentTool.ts`

**Tool Class:** `RunSubagentTool`

## Tool Declaration

```json
{
 "type": "function",
 "function": {
  "name": "runSubagent",
  "description": "Launch a new agent to handle complex, multi-step tasks autonomously. This tool is good at researching complex questions, searching for code, and executing multi-step tasks. When you are searching for a keyword or file and are not confident that you will find the right match in the first few tries, use this agent to perform the search for you.\n\n- Agents do not run async or in the background, you will wait for the agent's result.\n- When the agent is done, it will return a single message back to you. The result returned by the agent is not visible to the user. To show the user the result, you should send a text message back to the user with a concise summary of the result.\n- Each agent invocation is stateless. You will not be able to send additional messages to the agent, nor will the agent be able to communicate with you outside of its final report. Therefore, your prompt should contain a highly detailed task description for the agent to perform autonomously and you should specify exactly what information the agent should return back to you in its final and only message to you.\n- The agent's outputs should generally be trusted\n- Clearly tell the agent whether you expect it to write code or just to do research (search, file reads, web fetches, etc.), since it is not aware of the user's intent",
  "parameters": {
   "type": "object",
   "properties": {
    "prompt": {
     "type": "string",
     "description": "A detailed description of the task for the agent to perform"
    },
    "description": {
     "type": "string",
     "description": "A short (3-5 word) description of the task"
    },
    "subagentType": {
     "type": "string",
     "description": "Optional ID of a specific agent to invoke. If not provided, uses the current agent."
    }
   },
   "required": ["prompt", "description"]
  }
 }
}
```

## Input Parameters

### TypeScript Interface

```typescript
interface IRunSubagentToolInputParams {
 prompt: string;           // Required: Detailed task description for the agent
 description: string;      // Required: Short (3-5 word) description of the task
 subagentType?: string;    // Optional: ID of a specific agent to invoke
}
```

### Parameter Details

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `prompt` | string | Yes | A detailed description of the task for the agent to perform. Should include all context needed for autonomous execution |
| `description` | string | Yes | A short (3-5 word) description of the task. Used for progress display and invocation messages |
| `subagentType` | string | No | Optional ID of a specific agent (mode) to invoke. If not provided, uses the default agent. Only available when `chat.tools.subagent.customAgents` setting is enabled |

### Parameter Examples

```typescript
// Basic research task
{
 "prompt": "Search the codebase for all implementations of the IAuthService interface. For each implementation, document the file path, class name, and list of public methods. Return a markdown table summarizing your findings.",
 "description": "Find auth implementations"
}

// Code modification task with context
{
 "prompt": "In the file src/utils/validation.ts, add a new validation function called 'validateEmail' that checks if a string is a valid email address using a regex pattern. The function should return a boolean. Follow the existing code style in the file.",
 "description": "Add email validator"
}

// Delegating to a specific agent
{
 "prompt": "Create a comprehensive design document for implementing a caching layer for the database queries. Include architecture diagrams, API design, and performance considerations.",
 "description": "Design caching system",
 "subagentType": "Plan"
}
```

## Architecture

### Flow Diagram

```
┌─────────────────────────────────────────────────────────────┐
│                      runSubagent                            │
│                   (RunSubagentTool)                         │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
                    ┌─────────────────┐
                    │ Input Validation│
                    │ - prompt        │
                    │ - description   │
                    │ - subagentType? │
                    └─────────────────┘
                              │
                              ▼
                ┌─────────────────────────────┐
                │ Resolve Mode Configuration  │
                │ - Find mode by name         │
                │ - Get mode-specific model   │
                │ - Get mode-specific tools   │
                │ - Get mode instructions     │
                └─────────────────────────────┘
                              │
                              ▼
                ┌─────────────────────────────┐
                │ Disable Recursive Tools     │
                │ - Disable runSubagent       │
                │ - Disable manageTodoList    │
                └─────────────────────────────┘
                              │
                              ▼
                ┌─────────────────────────────┐
                │ Build Agent Request         │
                │ - Session info              │
                │ - Prompt as message         │
                │ - Selected model            │
                │ - Available tools           │
                │ - Mode instructions         │
                └─────────────────────────────┘
                              │
                              ▼
                ┌─────────────────────────────┐
                │ Invoke Default Agent        │
                │ - IChatAgentService         │
                │ - Progress callback         │
                │ - Collect markdown output   │
                └─────────────────────────────┘
                              │
                              ▼
                ┌─────────────────────────────┐
                │ Process Progress Events     │
                │ - Forward tool invocations  │
                │ - Forward edits             │
                │ - Collect markdown content  │
                └─────────────────────────────┘
                              │
                              ▼
                    ┌─────────────────┐
                    │ Return Final    │
                    │ Markdown Result │
                    └─────────────────┘
```

### Component Interaction

```
┌──────────────────┐
│  Parent Agent    │
│   (Claude/GPT)   │
└────────┬─────────┘
         │ Request: prompt, description, subagentType?
         ▼
┌──────────────────────────────────┐
│       RunSubagentTool            │
│  - getToolData()                 │
│  - prepareToolInvocation()       │
│  - invoke()                      │
└────────┬─────────────────────────┘
         │
         ├─────────────────┬────────────────┬────────────────┐
         │                 │                │                │
         ▼                 ▼                ▼                ▼
┌──────────────────┐ ┌──────────────┐ ┌──────────────┐ ┌──────────────┐
│ IChatModeService │ │IChatService  │ │ILanguageModel│ │ILMToolsService│
│ - findModeByName │ │- getSession  │ │Service       │ │ - toToolRefs  │
└──────────────────┘ └──────────────┘ │- lookupModel │ └──────────────┘
                                      └──────────────┘
         │
         ▼
┌──────────────────────────────────┐
│       IChatAgentService          │
│  - getDefaultAgent()             │
│  - invokeAgent()                 │
└──────────────────────────────────┘
         │
         ▼
┌──────────────────────────────────┐
│       Subagent Execution         │
│  - Full agent capabilities       │
│  - Tool access (except self)     │
│  - Markdown response collection  │
└──────────────────────────────────┘
```

## Internal Implementation

### Core Implementation

```typescript
export class RunSubagentTool extends Disposable implements IToolImpl {

 constructor(
  @IChatAgentService private readonly chatAgentService: IChatAgentService,
  @IChatService private readonly chatService: IChatService,
  @IChatModeService private readonly chatModeService: IChatModeService,
  @ILanguageModelToolsService private readonly languageModelToolsService: ILanguageModelToolsService,
  @ILanguageModelsService private readonly languageModelsService: ILanguageModelsService,
  @ILogService private readonly logService: ILogService,
  @ILanguageModelToolsService private readonly toolsService: ILanguageModelToolsService,
  @IConfigurationService private readonly configurationService: IConfigurationService,
 ) {
  super();
 }

 async invoke(
  invocation: IToolInvocation,
  _countTokens: CountTokensCallback,
  _progress: ToolProgress,
  token: CancellationToken
 ): Promise<IToolResult> {
  const args = invocation.parameters as IRunSubagentToolInputParams;

  // Validate invocation context
  if (!invocation.context) {
   throw new Error('toolInvocationToken is required for this tool');
  }

  // Get the chat model and request for writing progress
  const model = this.chatService.getSession(
   invocation.context.sessionResource
  ) as ChatModel | undefined;

  if (!model) {
   throw new Error('Chat model not found for session');
  }

  const request = model.getRequests().at(-1)!;

  try {
   // Get the default agent
   const defaultAgent = this.chatAgentService.getDefaultAgent(
    ChatAgentLocation.Chat,
    ChatModeKind.Agent
   );

   if (!defaultAgent) {
    return createToolSimpleTextResult('Error: No default agent available');
   }

   // Resolve mode-specific configuration if subagentType is provided
   let modeModelId = invocation.modelId;
   let modeTools = invocation.userSelectedTools;
   let modeInstructions: IChatRequestModeInstructions | undefined;

   if (args.subagentType) {
    const mode = this.chatModeService.findModeByName(args.subagentType);
    if (mode) {
     // Use mode-specific model if available
     modeModelId = this.resolveModeModel(mode) ?? modeModelId;

     // Use mode-specific tools if available
     modeTools = this.resolveModeTools(mode) ?? modeTools;

     // Get mode instructions
     modeInstructions = this.resolveModeInstructions(mode);
    } else {
     this.logService.warn(
      `RunSubagentTool: Agent '${args.subagentType}' not found, using current configuration`
     );
    }
   }

   // Track markdown output
   const markdownParts: string[] = [];
   let inEdit = false;

   const progressCallback = (parts: IChatProgress[]) => {
    for (const part of parts) {
     if (part.kind === 'prepareToolInvocation' ||
         part.kind === 'textEdit' ||
         part.kind === 'notebookEdit' ||
         part.kind === 'codeblockUri') {

      if (part.kind === 'codeblockUri' && !inEdit) {
       inEdit = true;
       model.acceptResponseProgress(request, {
        kind: 'markdownContent',
        content: new MarkdownString('```\n'),
        fromSubagent: true
       });
      }
      model.acceptResponseProgress(request, part);

      if (part.kind === 'prepareToolInvocation') {
       markdownParts.length = 0; // Reset on new tool invocation
      }
     } else if (part.kind === 'markdownContent') {
      if (inEdit) {
       model.acceptResponseProgress(request, {
        kind: 'markdownContent',
        content: new MarkdownString('\n```\n\n'),
        fromSubagent: true
       });
       inEdit = false;
      }
      markdownParts.push(part.content.value);
     }
    }
   };

   // Disable recursive tools
   if (modeTools) {
    modeTools[RunSubagentToolId] = false;
    modeTools[ManageTodoListToolToolId] = false;
   }

   // Build the agent request
   const agentRequest: IChatAgentRequest = {
    sessionId: invocation.context.sessionId,
    sessionResource: invocation.context.sessionResource,
    requestId: invocation.callId ?? `subagent-${Date.now()}`,
    agentId: defaultAgent.id,
    message: args.prompt,
    variables: { variables: [] },
    location: ChatAgentLocation.Chat,
    isSubagent: true,
    userSelectedModelId: modeModelId,
    userSelectedTools: modeTools,
    modeInstructions,
   };

   // Invoke the agent
   const result = await this.chatAgentService.invokeAgent(
    defaultAgent.id,
    agentRequest,
    progressCallback,
    [],
    token
   );

   // Check for errors
   if (result.errorDetails) {
    return createToolSimpleTextResult(
     `Agent error: ${result.errorDetails.message}`
    );
   }

   return createToolSimpleTextResult(
    markdownParts.join('') || 'Agent completed with no output'
   );

  } catch (error) {
   const errorMessage = `Error invoking subagent: ${
    error instanceof Error ? error.message : 'Unknown error'
   }`;
   this.logService.error(errorMessage, error);
   return createToolSimpleTextResult(errorMessage);
  }
 }

 async prepareToolInvocation(
  context: IToolInvocationPreparationContext,
  _token: CancellationToken
 ): Promise<IPreparedToolInvocation | undefined> {
  const args = context.parameters as IRunSubagentToolInputParams;
  return {
   invocationMessage: args.description,
  };
 }
}
```

### Tool Data Definition

```typescript
getToolData(): IToolData {
 const runSubagentToolData: IToolData = {
  id: RunSubagentToolId,
  toolReferenceName: VSCodeToolReference.runSubagent,
  canBeReferencedInPrompt: true,
  icon: ThemeIcon.fromId(Codicon.organization.id),
  displayName: localize('tool.runSubagent.displayName', 'Run Subagent'),
  userDescription: localize(
   'tool.runSubagent.userDescription',
   'Runs a task within an isolated subagent context. Enables efficient organization of tasks and context window management.'
  ),
  modelDescription: BaseModelDescription,
  source: ToolDataSource.Internal,
  inputSchema: {
   type: 'object',
   properties: {
    prompt: {
     type: 'string',
     description: 'A detailed description of the task for the agent to perform'
    },
    description: {
     type: 'string',
     description: 'A short (3-5 word) description of the task'
    }
   },
   required: ['prompt', 'description']
  }
 };

 // Add subagentType parameter if custom agents are enabled
 if (this.configurationService.getValue(ChatConfiguration.SubagentToolCustomAgents)) {
  runSubagentToolData.inputSchema!.properties!['subagentType'] = {
   type: 'string',
   description: 'Optional ID of a specific agent to invoke. If not provided, uses the current agent.'
  };
  runSubagentToolData.modelDescription +=
   `\n- If the user asks for a certain agent by name, you MUST provide that EXACT subagentType (case-sensitive) to invoke that specific agent.`;
 }

 return runSubagentToolData;
}
```

## Dependencies

### Service Dependencies

```typescript
// Core VS Code services required
@IChatAgentService          // Agent management and invocation
@IChatService               // Chat session management
@IChatModeService           // Mode/agent type resolution
@ILanguageModelToolsService // Tool configuration and references
@ILanguageModelsService     // Language model resolution
@ILogService                // Logging infrastructure
@IConfigurationService      // Settings management
```

### Key Interfaces

```typescript
// Tool invocation context
interface IToolInvocation {
 context?: {
  sessionId: string;
  sessionResource: URI;
 };
 modelId?: string;
 userSelectedTools?: Record<string, boolean>;
 callId?: string;
 parameters: unknown;
}

// Agent request structure
interface IChatAgentRequest {
 sessionId: string;
 sessionResource: URI;
 requestId: string;
 agentId: string;
 message: string;
 variables: { variables: [] };
 location: ChatAgentLocation;
 isSubagent: boolean;
 userSelectedModelId?: string;
 userSelectedTools?: Record<string, boolean>;
 modeInstructions?: IChatRequestModeInstructions;
}

// Progress callback types
interface IChatProgress {
 kind: 'prepareToolInvocation' | 'textEdit' | 'notebookEdit' |
       'codeblockUri' | 'markdownContent';
 // ... additional properties based on kind
}
```

## Performance Characteristics

### Execution Time

| Operation | Complexity | Notes |
|-----------|------------|-------|
| Mode resolution | O(1) | Direct lookup by name |
| Agent invocation | Variable | Depends on task complexity |
| Progress forwarding | O(n) | n = number of progress events |
| Result collection | O(m) | m = markdown parts count |

### Memory Usage

The subagent operates in a separate context but shares the session:

```typescript
// Memory considerations
const memoryFactors = {
 sessionReference: 'minimal',     // URI and ID only
 progressBuffer: 'accumulating',  // Markdown parts collected
 toolResults: 'returned',         // Final result returned to parent
};
```

### Recursion Prevention

The tool explicitly disables itself in subagent contexts:

```typescript
if (modeTools) {
 modeTools[RunSubagentToolId] = false;        // Prevent infinite recursion
 modeTools[ManageTodoListToolToolId] = false; // Subagent manages own tasks
}
```

## Use Cases

### 1. Complex Research Tasks

**Scenario:** Delegate a multi-file search to find all implementations of a pattern.

```typescript
{
 "prompt": "Search the entire codebase for all usages of the deprecated `oldApiCall` function. For each usage, document:\n1. File path\n2. Line number\n3. The function/method containing the usage\n4. Suggested replacement using `newApiCall`\n\nReturn your findings as a markdown table.",
 "description": "Find deprecated API usages"
}
```

### 2. Code Modification with Context

**Scenario:** Modify multiple related files with awareness of dependencies.

```typescript
{
 "prompt": "Refactor the UserService class to implement the new IUserService interface. This involves:\n1. Reading the interface definition from src/interfaces/IUserService.ts\n2. Updating the class in src/services/UserService.ts\n3. Updating any constructor injection sites\n4. Adding missing method implementations\n\nMake the actual code changes and report what you modified.",
 "description": "Implement IUserService"
}
```

### 3. Delegating to Specialized Agents

**Scenario:** Use a planning agent for architecture design.

```typescript
{
 "prompt": "Design a comprehensive caching strategy for our API endpoints. Consider:\n- Cache invalidation patterns\n- TTL strategies for different data types\n- Redis vs in-memory caching trade-offs\n- Implementation approach with minimal code changes\n\nProvide a detailed technical design document.",
 "description": "Design caching strategy",
 "subagentType": "Plan"
}
```

### 4. Parallel Investigation

**Scenario:** Research multiple topics simultaneously (called sequentially but managed independently).

```typescript
// First subagent
{
 "prompt": "Investigate the authentication flow in this codebase. Document the entry points, middleware, and token validation logic. Return a flowchart in mermaid syntax.",
 "description": "Map auth flow"
}

// Second subagent
{
 "prompt": "Investigate the database access patterns. Document the ORM usage, query patterns, and connection pooling. Return a summary with code examples.",
 "description": "Map database patterns"
}
```

### 5. Context Window Management

**Scenario:** Offload a large search to preserve parent context.

```typescript
{
 "prompt": "Read and summarize all test files in the src/__tests__ directory. For each test file, provide:\n- File name\n- Number of test cases\n- Main functionality being tested\n- Any mocked dependencies\n\nProvide a concise summary table, not the full file contents.",
 "description": "Summarize test coverage"
}
```

## Comparison with Other Tools

### vs Direct Agent Invocation

| Aspect | runSubagent | Direct Agent |
|--------|-------------|--------------|
| **Context** | Isolated, stateless | Shared, continuous |
| **Token Budget** | Separate | Shared |
| **Tool Access** | Configurable | Full |
| **Communication** | Single response | Interactive |
| **Use Case** | Delegated tasks | Main conversation |

### vs Parallel Tool Calls

| Aspect | runSubagent | Multiple Tools |
|--------|-------------|----------------|
| **Execution** | Sequential agent | Parallel operations |
| **Intelligence** | Full agent reasoning | Single tool logic |
| **Flexibility** | Multi-step capable | Fixed operations |
| **Overhead** | Higher (full agent) | Lower (direct call) |

### When to Use runSubagent

**Use runSubagent when:**
- Task requires multiple steps or tool invocations
- Context window needs management (offload large searches)
- Task benefits from autonomous reasoning
- Parent agent needs to delegate and continue

**Don't use runSubagent when:**
- Single, simple operation suffices
- Immediate result needed (subagent adds latency)
- Task requires parent context access
- Interactive refinement expected

## Error Handling

### Common Error Scenarios

**1. Missing Invocation Context**

```typescript
if (!invocation.context) {
 throw new Error('toolInvocationToken is required for this tool');
}

// Occurs when: Tool invoked outside proper chat context
// Resolution: Ensure tool is invoked within active chat session
```

**2. No Default Agent Available**

```typescript
const defaultAgent = this.chatAgentService.getDefaultAgent(
 ChatAgentLocation.Chat,
 ChatModeKind.Agent
);

if (!defaultAgent) {
 return createToolSimpleTextResult('Error: No default agent available');
}

// Occurs when: No agent registered for Chat location with Agent mode
// Resolution: Check agent extension is loaded
```

**3. Session Not Found**

```typescript
const model = this.chatService.getSession(
 invocation.context.sessionResource
) as ChatModel | undefined;

if (!model) {
 throw new Error('Chat model not found for session');
}

// Occurs when: Session expired or invalid resource
// Resolution: Verify session is active
```

**4. Unknown Agent Type**

```typescript
const mode = this.chatModeService.findModeByName(args.subagentType);
if (!mode) {
 this.logService.warn(
  `RunSubagentTool: Agent '${args.subagentType}' not found, using current configuration`
 );
}

// Occurs when: subagentType doesn't match any registered mode
// Resolution: Falls back to default agent gracefully
```

**5. Agent Execution Error**

```typescript
if (result.errorDetails) {
 return createToolSimpleTextResult(
  `Agent error: ${result.errorDetails.message}`
 );
}

// Occurs when: Subagent encounters runtime error
// Resolution: Error message returned to parent for handling
```

### Error Recovery

```typescript
try {
 const result = await this.chatAgentService.invokeAgent(
  defaultAgent.id,
  agentRequest,
  progressCallback,
  [],
  token
 );

 if (result.errorDetails) {
  return createToolSimpleTextResult(
   `Agent error: ${result.errorDetails.message}`
  );
 }

 return createToolSimpleTextResult(
  markdownParts.join('') || 'Agent completed with no output'
 );

} catch (error) {
 const errorMessage = `Error invoking subagent: ${
  error instanceof Error ? error.message : 'Unknown error'
 }`;
 this.logService.error(errorMessage, error);
 return createToolSimpleTextResult(errorMessage);
}
```

## Configuration

### Settings

```json
{
 // Enable custom agent selection via subagentType parameter
 "chat.tools.subagent.customAgents": true
}
```

### Programmatic Access

```typescript
// Check if custom agents are enabled
const customAgentsEnabled = this.configurationService.getValue(
 ChatConfiguration.SubagentToolCustomAgents
);

// Dynamically modify tool schema based on configuration
if (customAgentsEnabled) {
 toolData.inputSchema.properties['subagentType'] = {
  type: 'string',
  description: 'Optional ID of a specific agent to invoke.'
 };
}
```

## Implementation Blueprint

### Step-by-Step Implementation Guide

**Phase 1: Tool Registration**

```typescript
// 1. Define tool ID and constants
export const RunSubagentToolId = 'runSubagent';

// 2. Create tool class
export class RunSubagentTool extends Disposable implements IToolImpl {
 constructor(
  @IChatAgentService private readonly chatAgentService: IChatAgentService,
  @IChatService private readonly chatService: IChatService,
  // ... other dependencies
 ) {
  super();
 }

 getToolData(): IToolData {
  // Return tool metadata and schema
 }

 async invoke(...): Promise<IToolResult> {
  // Implementation
 }
}

// 3. Register tool with tools service
toolsService.registerTool(new RunSubagentTool(...));
```

**Phase 2: Mode Resolution**

```typescript
private resolveModeConfiguration(
 subagentType: string | undefined,
 defaults: { modelId?: string; tools?: Record<string, boolean> }
): ModeConfiguration {
 if (!subagentType) {
  return defaults;
 }

 const mode = this.chatModeService.findModeByName(subagentType);
 if (!mode) {
  return defaults;
 }

 return {
  modelId: this.resolveModelFromMode(mode) ?? defaults.modelId,
  tools: this.resolveToolsFromMode(mode) ?? defaults.tools,
  instructions: this.resolveInstructionsFromMode(mode),
 };
}
```

**Phase 3: Progress Handling**

```typescript
private createProgressCallback(
 model: ChatModel,
 request: IChatRequest
): (parts: IChatProgress[]) => void {
 const markdownParts: string[] = [];
 let inEdit = false;

 return (parts: IChatProgress[]) => {
  for (const part of parts) {
   switch (part.kind) {
    case 'prepareToolInvocation':
    case 'textEdit':
    case 'notebookEdit':
    case 'codeblockUri':
     this.forwardProgressPart(model, request, part, inEdit);
     if (part.kind === 'prepareToolInvocation') {
      markdownParts.length = 0;
     }
     break;
    case 'markdownContent':
     markdownParts.push(part.content.value);
     break;
   }
  }
 };
}
```

**Phase 4: Agent Invocation**

```typescript
private async invokeSubagent(
 agent: IChatAgent,
 request: IChatAgentRequest,
 progressCallback: ProgressCallback,
 token: CancellationToken
): Promise<IToolResult> {
 const result = await this.chatAgentService.invokeAgent(
  agent.id,
  request,
  progressCallback,
  [],
  token
 );

 if (result.errorDetails) {
  return createToolSimpleTextResult(`Agent error: ${result.errorDetails.message}`);
 }

 return createToolSimpleTextResult(
  this.collectMarkdownOutput() || 'Agent completed with no output'
 );
}
```

## Best Practices

### 1. Provide Detailed Prompts

```typescript
// ❌ Vague prompt
{
 "prompt": "Fix the bug",
 "description": "Fix bug"
}

// ✅ Detailed prompt with context
{
 "prompt": "In src/utils/parser.ts, there's a bug in the parseDate function where it fails to handle ISO 8601 dates with timezone offsets. The function is on line 45. Fix the regex pattern to properly capture the timezone offset. Test cases are in src/utils/parser.test.ts.",
 "description": "Fix date parser bug"
}
```

### 2. Specify Expected Output

```typescript
// ❌ No output specification
{
 "prompt": "Find all uses of the config object",
 "description": "Find config usages"
}

// ✅ Clear output expectation
{
 "prompt": "Find all uses of the config object in the src/ directory. Return a markdown table with columns: File Path, Line Number, Usage Context (the line of code). Limit to first 20 results.",
 "description": "Find config usages"
}
```

### 3. Indicate Action Type

```typescript
// ✅ Research only
{
 "prompt": "Research how authentication is implemented in this codebase. DO NOT make any code changes. Return a summary document.",
 "description": "Research auth flow"
}

// ✅ Code changes expected
{
 "prompt": "Implement the missing error handling in src/api/handler.ts. Make the actual code changes and report what you modified.",
 "description": "Add error handling"
}
```

### 4. Use Appropriate Agent Types

```typescript
// ✅ Use planning agent for design
{
 "prompt": "Design the architecture for...",
 "description": "Design architecture",
 "subagentType": "Plan"
}

// ✅ Use code agent for implementation
{
 "prompt": "Implement the feature...",
 "description": "Implement feature"
 // No subagentType - uses default code agent
}
```

### 5. Keep Descriptions Concise

```typescript
// ❌ Too verbose
{
 "description": "Search through the entire codebase to find all implementations of the interface"
}

// ✅ Concise (3-5 words)
{
 "description": "Find interface implementations"
}
```

## Limitations

### 1. Stateless Execution

**Limitation:** Each subagent invocation is completely stateless.

**Impact:**
- Cannot continue previous subagent work
- No memory between invocations
- Must include all context in prompt

**Mitigation:**
```typescript
// Include all necessary context in the prompt
{
 "prompt": "Previously, we found issues in files A, B, C (listed above). Now, for each file, implement the fix following pattern X...",
 "description": "Apply fixes"
}
```

### 2. No Interactive Refinement

**Limitation:** Cannot send follow-up messages to subagent.

**Impact:**
- Must get prompt right the first time
- Cannot ask clarifying questions
- Output is final

**Mitigation:**
- Use detailed, unambiguous prompts
- Specify output format explicitly
- Include examples if helpful

### 3. Tool Restrictions

**Limitation:** Subagent cannot use runSubagent or manageTodoList tools.

**Rationale:**
- Prevents infinite recursion
- Simplifies task tracking
- Maintains bounded execution

### 4. Synchronous Execution

**Limitation:** Parent waits for subagent completion.

**Impact:**
- Blocks parent progress
- Long tasks delay response
- No parallel subagent execution

### 5. Result Visibility

**Limitation:** Subagent results are not visible to user unless parent shares them.

**Behavior:**
- Only markdown output returned to parent
- Parent must summarize/forward results
- Tool invocations visible in UI

### 6. Context Boundaries

**Limitation:** Subagent has separate context window.

**Trade-offs:**
- Pro: Offloads parent context
- Con: Cannot access parent's accumulated knowledge
- Con: Must repeat shared context

### 7. Model/Tool Configuration

**Limitation:** Mode-specific configuration requires `chat.tools.subagent.customAgents` setting.

**Default behavior:** Uses parent's model and tools without this setting.

---

**Related Documentation:**
- [Tool Architecture Overview](./00-architecture-overview.md)
- [copilot_searchCodebase](./01-copilot-searchCodebase.md)
- [copilot_findFiles](./04-copilot-findFiles.md)

**Last Updated:** 2025-01-22

````
