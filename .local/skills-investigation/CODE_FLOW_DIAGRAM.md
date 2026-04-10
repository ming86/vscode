# Skills Investigation - Code Flow Diagram

## Skills Loading Code Path

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           USER SENDS MESSAGE                                 │
└─────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│  chatWidget.ts:2290                                                          │
│  await this._autoAttachInstructions(requestInputs)                          │
└─────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│  chatWidget.ts:2647                                                          │
│  ┌─────────────────────────────────────────────────────────────────────┐    │
│  │  const enabledTools = this.input.currentModeKind === ChatModeKind.Agent   │
│  │      ? this.input.selectedToolsModel.userSelectedTools.get()              │
│  │      : undefined;                                                          │
│  └─────────────────────────────────────────────────────────────────────┘    │
│                                                                              │
│  ⚠️  GATE 1: If mode is NOT Agent, enabledTools = undefined                 │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
                                      │
                         ┌────────────┴────────────┐
                         │                         │
              Mode = Agent                 Mode = Ask/Edit
                         │                         │
                         ▼                         ▼
              enabledTools = {...}       enabledTools = undefined
                         │                         │
                         └────────────┬────────────┘
                                      │
                                      ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│  chatWidget.ts:2650                                                          │
│  const computer = this.instantiationService.createInstance(                  │
│      ComputeAutomaticInstructions, enabledTools, enabledSubAgents           │
│  );                                                                          │
└─────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│  computeAutomaticInstructions.ts:51                                          │
│  constructor(                                                                │
│      private readonly _enabledTools: UserSelectedTools | undefined,         │
│      ...                                                                     │
│  )                                                                           │
└─────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│  computeAutomaticInstructions.ts:240-249  (_getTool method)                 │
│  ┌─────────────────────────────────────────────────────────────────────┐    │
│  │  private _getTool(referenceName: string) {                               │
│  │      if (!this._enabledTools) {                                          │
│  │          return undefined;  // ⚠️  GATE 2: No tools if not Agent mode    │
│  │      }                                                                    │
│  │      const tool = this._languageModelToolsService.getToolByName(...);    │
│  │      if (tool && this._enabledTools[tool.id]) {                          │
│  │          return { tool, variable };                                       │
│  │      }                                                                    │
│  │      return undefined;                                                    │
│  │  }                                                                        │
│  └─────────────────────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│  computeAutomaticInstructions.ts:253                                         │
│  const readTool = this._getTool('readFile');                                │
│                                                                              │
│  ⚠️  GATE 3: If enabledTools undefined, readTool = undefined                │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
                                      │
                         ┌────────────┴────────────┐
                         │                         │
              readTool defined           readTool = undefined
                         │                         │
                         ▼                         ▼
              Continue to skills         SKIP ENTIRE BLOCK
                         │                    (Skills NOT added)
                         │                         │
                         ▼                         │
┌────────────────────────────────────────┐         │
│  computeAutomaticInstructions.ts:257    │         │
│  if (readTool) {                        │         │
│      // Skills block here               │         │
│  }                                      │         │
│                                         │         │
│  ⚠️  GATE 4: Skills block gated        │         │
│                                         │         │
└────────────────────────────────────────┘         │
                         │                         │
                         ▼                         │
┌────────────────────────────────────────┐         │
│  promptsServiceImpl.ts:756              │         │
│  const useAgentSkills = this           │         │
│      .configurationService             │         │
│      .getValue(USE_AGENT_SKILLS);      │         │
│                                         │         │
│  ⚠️  GATE 5: Setting must be true      │         │
│  (Affected by workspace trust for      │         │
│   restricted settings)                  │         │
│                                         │         │
└────────────────────────────────────────┘         │
                         │                         │
          ┌──────────────┴──────────────┐          │
          │                             │          │
    Setting = true              Setting = false   │
          │                             │          │
          ▼                             ▼          │
   Continue search             Return undefined    │
          │                             │          │
          ▼                             │          │
┌────────────────────────────────────────┐         │
│  Skill Discovery                        │         │
│                                         │         │
│  Search locations:                      │         │
│  - ~/.copilot/skills                    │         │
│  - ~/.claude/skills                     │         │
│  - .github/skills (workspace)           │         │
│  - .claude/skills (workspace)           │         │
│                                         │         │
│  ⚠️  GATE 6: Path resolution           │         │
│  - Tilde expansion requires userHome   │         │
│  - On Windows: os.homedir() uses       │         │
│    USERPROFILE env var                  │         │
│                                         │         │
└────────────────────────────────────────┘         │
                         │                         │
          ┌──────────────┴──────────────┐          │
          │                             │          │
    Skills found > 0          Skills found = 0     │
          │                             │          │
          ▼                             ▼          │
┌────────────────────────────────┐   No skills      │
│  ADD <skills> SECTION          │   added          │
│                                 │      │          │
│  <skills>                       │      │          │
│    <skill>                      │      │          │
│      <name>...</name>           │      │          │
│      <description>...</desc>    │      │          │
│      <file>...</file>           │      │          │
│    </skill>                     │      │          │
│  </skills>                      │      │          │
│                                 │      │          │
└────────────────────────────────┘      │          │
                         │              │          │
                         └──────────────┴──────────┘
                                      │
                                      ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                        SYSTEM PROMPT GENERATED                               │
│                                                                              │
│  If any gate failed:                                                        │
│  - <skills> section will be MISSING                                         │
│  - NO error or warning is logged (silent failure)                           │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

## Gate Summary

| Gate | Location | Condition | Failure Result |
|------|----------|-----------|----------------|
| 1 | chatWidget.ts:2647 | `currentModeKind === ChatModeKind.Agent` | `enabledTools = undefined` |
| 2 | computeAutomaticInstructions.ts:241 | `this._enabledTools` is truthy | `_getTool()` returns `undefined` |
| 3 | computeAutomaticInstructions.ts:253 | `readTool` is defined | `readTool = undefined` |
| 4 | computeAutomaticInstructions.ts:257 | `if (readTool)` | Entire block skipped |
| 5 | promptsServiceImpl.ts:756 | `useAgentSkills === true` | Returns `undefined` early |
| 6 | promptFilesLocator.ts:297 | `userHome` is defined | Tilde paths silently skipped |

## Key Insight

**All gates must pass for skills to appear in the system prompt.**

The most common failure is **Gate 1** - the user is not in Agent mode. This single condition causes Gates 2, 3, and 4 to fail automatically, resulting in the entire skills section being skipped silently.
