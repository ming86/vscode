# Recommended Code Fixes (UPDATED)

This document proposes code changes to fix the skills feature on Windows and improve debuggability.

---

## 🔴 CRITICAL BUG: Case-Sensitive Folder Name Comparison on Windows

**File:** `vscode/src/vs/workbench/contrib/chat/common/promptSyntax/service/promptsServiceImpl.ts`
**Lines:** 780-787

### The Bug

The folder name validation uses a **case-sensitive** string comparison:

```typescript
const skillFolderUri = dirname(uri);
const folderName = basename(skillFolderUri);
if (sanitizedName !== folderName) {  // ❌ CASE-SENSITIVE!
    skippedNameMismatch++;
    this.logger.error(`Agent skill name "${sanitizedName}" does not match folder name "${folderName}"`);
    return;
}
```

On Windows, folder names are **case-insensitive**. A folder named `UI-Design` is the same as `ui-design`, but this code treats them as different.

### Recommended Fix

Use case-insensitive comparison on Windows:

```typescript
const skillFolderUri = dirname(uri);
const folderName = basename(skillFolderUri);
// Use case-insensitive comparison on Windows (file system is case-insensitive)
const isMatch = isWindows
    ? sanitizedName.toLowerCase() === folderName.toLowerCase()
    : sanitizedName === folderName;
if (!isMatch) {
    skippedNameMismatch++;
    this.logger.error(`Agent skill name "${sanitizedName}" does not match folder name "${folderName}"`);
    return;
}
```

**Alternative:** Use `extUriBiasedIgnorePathCase` for consistency with other path comparisons:

```typescript
import { extUriBiasedIgnorePathCase } from '../../../base/common/resources.js';

// Compare using platform-appropriate case sensitivity
const skillFolderName = basename(skillFolderUri);
if (!extUriBiasedIgnorePathCase.isEqual(
    URI.file(sanitizedName),
    URI.file(skillFolderName)
)) {
    // ... error handling
}
```

---

## Issue Summary (Updated)

The `<skills>` section fails to appear on Windows due to:
1. Mode not being Agent
2. Workspace trust affecting restricted settings
3. Silent path resolution failures
4. Lack of logging when skills are skipped

---

## Recommended Fix 1: Add Logging When Skills Are Skipped Due to Mode

**File:** `vscode/src/vs/workbench/contrib/chat/browser/widget/chatWidget.ts`

**Location:** Line 2645-2651

**Current Code:**
```typescript
private async _autoAttachInstructions({ attachedContext }: IChatRequestInputOptions): Promise<void> {
    this.logService.debug(`ChatWidget#_autoAttachInstructions: prompt files are always enabled`);
    const enabledTools = this.input.currentModeKind === ChatModeKind.Agent ? this.input.selectedToolsModel.userSelectedTools.get() : undefined;
    const enabledSubAgents = this.input.currentModeKind === ChatModeKind.Agent ? this.input.currentModeObs.get().agents?.get() : undefined;
    const computer = this.instantiationService.createInstance(ComputeAutomaticInstructions, enabledTools, enabledSubAgents);
    await computer.collect(attachedContext, CancellationToken.None);
}
```

**Recommended Change:**
```typescript
private async _autoAttachInstructions({ attachedContext }: IChatRequestInputOptions): Promise<void> {
    this.logService.debug(`ChatWidget#_autoAttachInstructions: prompt files are always enabled`);
    const isAgentMode = this.input.currentModeKind === ChatModeKind.Agent;
    const enabledTools = isAgentMode ? this.input.selectedToolsModel.userSelectedTools.get() : undefined;
    const enabledSubAgents = isAgentMode ? this.input.currentModeObs.get().agents?.get() : undefined;

    // Log when skills/tools are skipped due to mode
    if (!isAgentMode) {
        this.logService.debug(`ChatWidget#_autoAttachInstructions: Mode is ${this.input.currentModeKind}, skills and tools will not be included (requires Agent mode)`);
    }

    const computer = this.instantiationService.createInstance(ComputeAutomaticInstructions, enabledTools, enabledSubAgents);
    await computer.collect(attachedContext, CancellationToken.None);
}
```

---

## Recommended Fix 2: Add Logging When `readTool` Is Undefined

**File:** `vscode/src/vs/workbench/contrib/chat/common/promptSyntax/computeAutomaticInstructions.ts`

**Location:** Lines 240-257

**Current Code:**
```typescript
private _getTool(referenceName: string): { tool: IToolData; variable: string } | undefined {
    if (!this._enabledTools) {
        return undefined;
    }
    const tool = this._languageModelToolsService.getToolByName(referenceName);
    if (tool && this._enabledTools[tool.id]) {
        return { tool, variable: `#tool:${this._languageModelToolsService.getFullReferenceName(tool)}` };
    }
    return undefined;
}

// In _getInstructionsWithPatternsList:
const readTool = this._getTool('readFile');
```

**Recommended Change:**
```typescript
private _getTool(referenceName: string): { tool: IToolData; variable: string } | undefined {
    if (!this._enabledTools) {
        this._logService.trace(`[ComputeAutomaticInstructions] _getTool('${referenceName}'): enabledTools is undefined (not in Agent mode)`);
        return undefined;
    }
    const tool = this._languageModelToolsService.getToolByName(referenceName);
    if (tool && this._enabledTools[tool.id]) {
        return { tool, variable: `#tool:${this._languageModelToolsService.getFullReferenceName(tool)}` };
    }
    this._logService.trace(`[ComputeAutomaticInstructions] _getTool('${referenceName}'): tool not found or not enabled`);
    return undefined;
}

// In _getInstructionsWithPatternsList:
const readTool = this._getTool('readFile');
if (!readTool) {
    this._logService.debug(`[ComputeAutomaticInstructions] readFile tool not available, skills and agent instructions will not be included`);
}
```

---

## Recommended Fix 3: Log When Skills Setting Is Affected by Workspace Trust

**File:** `vscode/src/vs/workbench/contrib/chat/common/promptSyntax/service/promptsServiceImpl.ts`

**Location:** Line 755-757

**Current Code:**
```typescript
public async findAgentSkills(token: CancellationToken): Promise<IAgentSkill[] | undefined> {
    const useAgentSkills = this.configurationService.getValue(PromptsConfig.USE_AGENT_SKILLS);
    if (useAgentSkills) {
        // ... skill discovery
    }
}
```

**Recommended Change:**
```typescript
public async findAgentSkills(token: CancellationToken): Promise<IAgentSkill[] | undefined> {
    const inspectedValue = this.configurationService.inspect(PromptsConfig.USE_AGENT_SKILLS);
    const useAgentSkills = this.configurationService.getValue(PromptsConfig.USE_AGENT_SKILLS);

    // Log diagnostic information about the setting
    this.logger.trace(`[findAgentSkills] Setting inspection: ` +
        `userValue=${inspectedValue.userValue}, ` +
        `workspaceValue=${inspectedValue.workspaceValue}, ` +
        `policyValue=${inspectedValue.policyValue}, ` +
        `effectiveValue=${useAgentSkills}`);

    // Warn if workspace value exists but might be ignored due to trust
    if (inspectedValue.workspaceValue !== undefined && inspectedValue.workspaceValue !== useAgentSkills) {
        this.logger.warn(`[findAgentSkills] Workspace setting for ${PromptsConfig.USE_AGENT_SKILLS} ` +
            `(${inspectedValue.workspaceValue}) differs from effective value (${useAgentSkills}). ` +
            `This may be due to workspace trust restrictions.`);
    }

    if (useAgentSkills) {
        // ... skill discovery
    } else {
        this.logger.info(`[findAgentSkills] Skills disabled: ${PromptsConfig.USE_AGENT_SKILLS} = false`);
        return undefined;
    }
}
```

---

## Recommended Fix 4: Improve Home Directory Fallback on Windows

**File:** `vscode-copilot-chat/src/platform/env/vscode-node/nativeEnvServiceImpl.ts`

**Current Code:**
```typescript
get userHome() {
    return URI.file(os.homedir());
}
```

**Recommended Change:**
```typescript
get userHome() {
    let home = os.homedir();

    // Fallback for Windows when USERPROFILE is not set
    if (!home && process.platform === 'win32') {
        if (process.env.HOMEDRIVE && process.env.HOMEPATH) {
            home = path.join(process.env.HOMEDRIVE, process.env.HOMEPATH);
        } else if (process.env.HOME) {
            home = process.env.HOME;
        }
    }

    if (!home) {
        // Log warning if home directory cannot be determined
        console.warn('[NativeEnvService] Unable to determine user home directory. Personal skills may not be available.');
        // Return a URI that will gracefully fail
        return URI.file('');
    }

    return URI.file(home);
}
```

---

## Recommended Fix 5: Add Telemetry for Skills Skipped Scenarios

**File:** `vscode/src/vs/workbench/contrib/chat/common/promptSyntax/computeAutomaticInstructions.ts`

Add telemetry to track when skills are skipped:

```typescript
type SkillsSkippedEvent = {
    reason: 'no_enabled_tools' | 'no_read_tool' | 'setting_disabled' | 'no_skills_found';
    mode: string;
    workspaceTrusted: boolean;
};

// In _getInstructionsWithPatternsList:
if (!readTool) {
    this._telemetryService.publicLog2<SkillsSkippedEvent, SkillsSkippedClassification>('skillsSkipped', {
        reason: this._enabledTools ? 'no_read_tool' : 'no_enabled_tools',
        mode: 'unknown', // Would need to be passed in
        workspaceTrusted: true, // Would need workspace trust service
    });
}
```

---

## Recommended Fix 6: Add UI Indicator for Skills Status

**Proposal:** Add a status bar item or chat input indicator showing skills status:

- ✅ Skills: 5 loaded
- ⚠️ Skills: Disabled (not in Agent mode)
- ⚠️ Skills: Disabled (workspace untrusted)
- ❌ Skills: Disabled (setting off)

This would provide immediate visibility to users about why skills may not be working.

---

## Recommended Fix 7: Add Developer Command to Diagnose Skills

Add a command `Developer: Diagnose Copilot Skills` that outputs:

```
=== Copilot Skills Diagnostic ===

Mode: Agent ✅
Workspace Trust: Trusted ✅
chat.useAgentSkills: true ✅
  - User value: true
  - Workspace value: undefined
  - Policy value: undefined

Home Directory: C:\Users\username ✅

Skill Locations Checked:
  - C:\Users\username\.claude\skills ✅ (2 skills found)
  - C:\Users\username\.copilot\skills ❌ (folder not found)
  - D:\project\.claude\skills ✅ (1 skill found)
  - D:\project\.github\skills ❌ (folder not found)

Skills Found: 3
  - ui-design (claude-personal)
  - web-dev (claude-personal)
  - project-helper (claude-workspace)

Skills Skipped: 1
  - broken-skill: Missing 'name' attribute in SKILL.md
```

---

## Priority Order

1. **Fix 1 & 2:** Add logging for mode-based skipping (Low effort, high diagnostic value)
2. **Fix 3:** Log workspace trust impact (Low effort, high diagnostic value)
3. **Fix 7:** Developer diagnostic command (Medium effort, very high user value)
4. **Fix 4:** Home directory fallback (Low effort, improves reliability)
5. **Fix 5:** Telemetry for skipped scenarios (Medium effort, helps identify patterns)
6. **Fix 6:** UI indicator (Higher effort, best user experience)

---

## Testing Recommendations

After implementing fixes:

1. **Unit tests:**
   - Test `_getTool` with undefined `_enabledTools`
   - Test skill discovery with various setting combinations
   - Test home directory resolution fallbacks

2. **Integration tests:**
   - Verify skills appear in Agent mode, not in Ask/Edit
   - Verify restricted settings behavior in untrusted workspaces
   - Test skill discovery with workspace and personal paths

3. **Manual testing on Windows:**
   - Test with standard USERPROFILE
   - Test with OneDrive folder redirection
   - Test with network home directories
   - Test with EMU accounts
