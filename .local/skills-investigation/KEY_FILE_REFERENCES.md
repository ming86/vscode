# Skills Investigation - Key File References

Quick reference for all files involved in the skills feature.

---

## VS Code Core (vscode repo)

### Skills Settings Definition
- **File:** `src/vs/workbench/contrib/chat/browser/chat.contribution.ts`
- **Lines:** 735-770
- **Key:** `chat.useAgentSkills`, `chat.agentSkillsLocations` definitions with `restricted: true`

### Mode Check & Tools Assignment
- **File:** `src/vs/workbench/contrib/chat/browser/widget/chatWidget.ts`
- **Lines:** 2645-2651
- **Key:** `enabledTools` only set if `ChatModeKind.Agent`

### System Prompt Generation
- **File:** `src/vs/workbench/contrib/chat/common/promptSyntax/computeAutomaticInstructions.ts`
- **Lines:** 240-249 (`_getTool`), 253-257 (`readTool` gate), 307-320 (skills block)
- **Key:** Skills section injection logic

### Skills Discovery Service
- **File:** `src/vs/workbench/contrib/chat/common/promptSyntax/service/promptsServiceImpl.ts`
- **Lines:** 755-814
- **Key:** `findAgentSkills()` implementation

### Skills File Locator
- **File:** `src/vs/workbench/contrib/chat/common/promptSyntax/utils/promptFilesLocator.ts`
- **Lines:** 297-306 (tilde expansion), 505-517 (folder search), 632-651 (path validation)
- **Key:** Skill folder discovery and path resolution

### Default Skill Locations
- **File:** `src/vs/workbench/contrib/chat/common/promptSyntax/config/promptFileLocations.ts`
- **Lines:** 97-102
- **Key:** `DEFAULT_SKILL_SOURCE_FOLDERS` array

### Workspace Trust & Restricted Settings
- **File:** `src/vs/workbench/services/configuration/browser/configuration.ts`
- **Lines:** 693-700
- **Key:** `skipRestricted` based on workspace trust

### Chat Mode Service
- **File:** `src/vs/workbench/contrib/chat/common/chatModes.ts`
- **Lines:** 226-229
- **Key:** `isAgentModeDisabledByPolicy()`

### Selected Tools Model
- **File:** `src/vs/workbench/contrib/chat/browser/widget/input/chatSelectedTools.ts`
- **Lines:** 159-168
- **Key:** `userSelectedTools` observable derivation

---

## Copilot Chat Extension (vscode-copilot-chat repo)

### Environment Service (Home Directory)
- **File:** `src/platform/env/vscode-node/nativeEnvServiceImpl.ts`
- **Lines:** 14-16
- **Key:** `userHome` property using `os.homedir()`

### Custom Instructions Service
- **File:** `src/platform/customInstructions/common/customInstructionsService.ts`
- **Lines:** 83-84 (skill folder constants), 188-202 (skill URI construction)
- **Key:** Skill folder path handling

### Copilot Token (EMU Detection)
- **File:** `src/platform/authentication/common/copilotToken.ts`
- **Lines:** 186-188
- **Key:** `isEditorPreviewFeaturesEnabled()`, enterprise checks

### Context Keys
- **File:** `src/extension/contextKeys/vscode-node/contextKeys.contribution.ts`
- **Lines:** 171-180
- **Key:** `github.copilot.previewFeaturesDisabled` context key

---

## Configuration Keys

| Key | Type | Default | File |
|-----|------|---------|------|
| `chat.useAgentSkills` | boolean | `true` | chat.contribution.ts |
| `chat.agentSkillsLocations` | object | DEFAULT_SKILL_SOURCE_FOLDERS | chat.contribution.ts |
| `chat.agent.enabled` | boolean | `true` | chat.contribution.ts |

---

## Telemetry Events

| Event | File | Purpose |
|-------|------|---------|
| `agentSkillsFound` | promptsServiceImpl.ts:852-900 | Tracks skills discovery results |

---

## Output Channels

| Channel | Purpose |
|---------|---------|
| "GitHub Copilot Chat" | Main extension logging |

---

## Useful Commands

| Command ID | Purpose |
|------------|---------|
| `github.copilot.debug.showOutputChannel` | Show Copilot Chat output |
| `github.copilot.chat.debug.exportPromptArchive` | Export prompt data |
