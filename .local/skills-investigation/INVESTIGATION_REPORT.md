# Skills Missing from System Prompt - Investigation Report (UPDATED)

**Date:** January 20, 2026
**Issue:** `<skills>` section missing from system prompt on Windows for some users
**Environment:** EMU (Enterprise Managed Users) GitHub account, Microsoft account for VS Code sync
**Timeline:** Skills working in mid-December 2025, stopped working late December 2025 / early January 2026

---

## 🔴 EXECUTIVE SUMMARY (Updated Jan 20, 2026)

### Most Likely Root Causes (Ranked by Probability)

| # | Cause | Probability | Applies If |
|---|-------|-------------|------------|
| 1 | **VS Code version between Dec 17 - Jan 16** | HIGH | `chat_preview_features_enabled` check still active for EMU accounts |
| 2 | **BOM (Byte Order Mark) in skill file** | MEDIUM-HIGH | File saved with UTF-8 BOM (Windows Notepad default) |
| 3 | **Trailing whitespace in `name:` attribute** | MEDIUM | `name: skill-creator ` (with space) ≠ folder `skill-creator` |
| 4 | **`chat.useAgentSkills` setting is `false`** | MEDIUM | Migrated from old `chat.useClaudeSkills: false` |
| 5 | **`readFile` tool disabled** | MEDIUM | User disabled tool in settings or Tools menu |
| 6 | **Chat mode not Agent** | LOW | User confirmed mode IS Agent |

### Immediate Diagnostic Steps

1. **Check VS Code version**: Must be newer than Jan 16, 2026 (commit `87b2e355dd5`)
2. **Check for BOM**: Open SKILL.md in hex editor - should NOT start with `EF BB BF`
3. **Check for trailing spaces**: `name: skill-creator` must not have trailing whitespace
4. **Check setting**: `chat.useAgentSkills` must be `true`
5. **Check logs**: View → Output → "Window" → Search for `[findAgentSkills]`

### Telemetry Skip Counters to Check

If skills are being discovered but rejected, these counters will be non-zero:
- `skippedMissingName` - YAML frontmatter missing `name:`
- `skippedMissingDescription` - Missing `description:`
- `skippedNameMismatch` - Name doesn't match folder name
- `skippedParseFailed` - YAML parse error (could be BOM or encoding issue)

---

## 🆕 vscode-copilot-chat Extension Investigation (January 20, 2026)

### customInstructionsService.ts Analysis

The skills handling in vscode-copilot-chat is implemented in:
**[src/platform/customInstructions/common/customInstructionsService.ts](src/platform/customInstructions/common/customInstructionsService.ts)**

#### Recent Commits (Since December 2025)

```
73a6bb7e Adopt latest provider pattern for org/enterprise custom agents (#2737)
cb544ba1 Custom chat render for agent skills (#2756)
5ce4f46f Updates for Agent Skills alignment (#2634)
b3a174eb User skill can not be loaded on Windows (#2563)   ← WINDOWS FIX
ea0e8d8c allow reading personal skills, refining other isExternalInstructionsFile checks (#2392)
```

#### Key Finding: Windows Fix Applied (b3a174eb)

On **December 12, 2025**, a Windows-specific fix was applied to use `extUriBiasedIgnorePathCase` for path comparisons:

```typescript
// BEFORE (case-sensitive on Windows):
import { isEqualOrParent, joinPath } from '...resources';
const skillFolderUri = joinPath(this.envService.userHome, SKILL_FOLDER);
return isEqualOrParent(uri, skillFolderUri);

// AFTER (case-insensitive for file:// scheme on Windows):
import { extUriBiasedIgnorePathCase } from '...resources';
const skillFolderUri = extUriBiasedIgnorePathCase.joinPath(this.envService.userHome, folder);
return extUriBiasedIgnorePathCase.isEqualOrParent(uri, skillFolderUri);
```

#### Current Skill Folder Locations (as of cb544ba1)

```typescript
const WORKSPACE_SKILL_FOLDERS = ['.github/skills', '.claude/skills'];
const PERSONAL_SKILL_FOLDERS = ['.copilot/skills', '.claude/skills'];
const USE_AGENT_SKILLS_SETTING = 'chat.useAgentSkills';
```

#### How Skills Are Matched (`_matchInstructionLocationsFromSkills` Observable)

```typescript
this._matchInstructionLocationsFromSkills = observableFromEvent(
    (handleChange) => {
        // Listens for:
        // 1. Configuration changes to 'chat.useAgentSkills'
        // 2. Workspace folder changes
    },
    () => {
        if (this.configurationService.getNonExtensionConfig<boolean>(USE_AGENT_SKILLS_SETTING)) {
            const personalSkillFolderUris = PERSONAL_SKILL_FOLDERS.map(folder =>
                extUriBiasedIgnorePathCase.joinPath(this.envService.userHome, folder));
            const workspaceSkillFolderUris = this.workspaceService.getWorkspaceFolders().flatMap(workspaceFolder =>
                WORKSPACE_SKILL_FOLDERS.map(folder =>
                    extUriBiasedIgnorePathCase.joinPath(workspaceFolder, folder)));

            const topLevelSkillsFolderUris = [...personalSkillFolderUris, ...workspaceSkillFolderUris];

            return ((uri: URI) => {
                for (const topLevelSkillFolderUri of topLevelSkillsFolderUris) {
                    if (extUriBiasedIgnorePathCase.isEqualOrParent(uri, topLevelSkillFolderUri)) {
                        const relativePath = extUriBiasedIgnorePathCase.relativePath(topLevelSkillFolderUri, uri);
                        if (relativePath) {
                            const skillName = relativePath.split('/')[0];  // First path segment
                            const skillFolderUri = extUriBiasedIgnorePathCase.joinPath(topLevelSkillFolderUri, skillName);
                            return { skillName, skillFolderUri };
                        }
                    }
                }
                return undefined;  // ← RETURNS UNDEFINED IF NO MATCH
            });
        }
        return (() => undefined);  // ← RETURNS UNDEFINED IF SETTING DISABLED
    }
);
```

#### userHome Resolution

Personal skill paths use `os.homedir()` via `INativeEnvService.userHome`:

```typescript
// src/platform/env/vscode-node/nativeEnvServiceImpl.ts
import * as os from 'os';
export class NativeEnvServiceImpl extends EnvServiceImpl implements INativeEnvService {
    get userHome() {
        return URI.file(os.homedir());  // On Windows: C:\Users\<username>
    }
}
```

#### Potential Issues on Windows

1. **Setting Check:** Skills require `chat.useAgentSkills` to be `true`. If the setting is `false` (or migrated from `false`), the observable returns `() => undefined`, and NO skills are discovered.

2. **Path Case Sensitivity:** Although `extUriBiasedIgnorePathCase` is used, it's only case-insensitive for `file://` scheme URIs. The logic in [resources.ts](src/util/vs/base/common/resources.ts#L342):
   ```typescript
   export const extUriBiasedIgnorePathCase = new ExtUri(uri => {
       return uri.scheme === Schemas.file ? !isLinux : true;  // Case-insensitive on Windows/Mac
   });
   ```

3. **Workspace Service:** If `getWorkspaceFolders()` returns an empty array (no workspace open), only personal skill folders are checked.

4. **Skill Name Extraction:** Uses forward slash `/` to split path segments:
   ```typescript
   const skillName = relativePath.split('/')[0];
   ```
   This should work as `extUriBiasedIgnorePathCase.relativePath()` normalizes paths.

### Integration with VS Code (vscode repo)

The actual skill discovery and validation happens in VS Code core:

**[promptFilesLocator.ts](vscode/src/vs/workbench/contrib/chat/common/promptSyntax/utils/promptFilesLocator.ts):**
- `findAgentSkills()` searches for `SKILL.md` files
- Uses `pathService.userHome()` for tilde expansion
- Searches `~/.copilot/skills`, `~/.claude/skills`, `.github/skills`, `.claude/skills`

**[promptsServiceImpl.ts](vscode/src/vs/workbench/contrib/chat/common/promptSyntax/service/promptsServiceImpl.ts):**
- `findAgentSkills()` method validates discovered skills
- **Validates name matches folder name (CASE-SENSITIVE!)** ← THIS IS THE PROBLEM
- Sends skills to `computeAutomaticInstructions.ts`

**Recent VS Code Fix (January 16, 2026):** `87b2e355dd5`
- Removed `chat_preview_features_enabled` requirement
- Skills no longer require preview features flag

---

## 🔴 CRITICAL UPDATE: New Root Cause Identified

**User confirmed:** Mode IS Agent, workspace IS trusted, Agent mode IS enabled by organization.

After researching git history, the investigation identified **NEW validation requirements** added between December 17, 2025 and January 15, 2026 as the root cause.

### Primary Root Cause: Folder Name Validation (Case-Sensitive)

**Commit:** `6cb58a3eca5` (January 15, 2026) - "Unify agent skills internal architecture"

A new validation was added requiring the **folder name to exactly match the `name:` attribute** in SKILL.md. The comparison is **CASE-SENSITIVE**, which breaks existing skills on Windows:

```typescript
// promptsServiceImpl.ts:780-787
const skillFolderUri = dirname(uri);
const folderName = basename(skillFolderUri);
if (sanitizedName !== folderName) {  // CASE-SENSITIVE COMPARISON!
    skippedNameMismatch++;
    this.logger.error(`Agent skill name "${sanitizedName}" does not match folder name "${folderName}"`);
    return;  // Skill silently skipped
}
```

### Secondary Root Cause: Required Fields Added

**Commit:** `b76549b5cda` (December 17, 2025) - "Add validation for name and description fields"

Both `name:` and `description:` became **required** fields. Skills missing either are silently skipped.

---

## Timeline of Breaking Changes

| Date | Commit | Change | Impact |
|------|--------|--------|--------|
| **Dec 17** | `b76549b5cda` | Added `name:` and `description:` as **required** fields | Skills without these fields silently skipped |
| **Jan 15** | `6cb58a3eca5` | Added folder name ↔ skill name **exact match** | **BREAKING: Case-sensitive on Windows** |

---

## Windows-Specific Path Handling Changes (Dec 2025 - Jan 2026)

### Critical Commits Summary

| Date | Repo | Commit | Description | Windows Impact |
|------|------|--------|-------------|----------------|
| **2025-12-12** | `vscode-copilot-chat` | `b3a174eb` | **User skill can not be loaded on Windows** | **FIX:** Changed to `extUriBiasedIgnorePathCase` for case-insensitive path comparison |
| **2025-12-17** | `vscode` | `82fb5e14653` | Agent Skills cleanup | Added `chat_preview_features_enabled` check |
| **2025-12-17** | `vscode` | `b76549b5cda` | **Add validation for name/description** | **BREAKING:** Required fields added |
| **2025-12-18** | `vscode` | `89b8c4e9fa4` | Updates for Agent Skills alignment | Changed folder paths from strings to objects with metadata |
| **2026-01-14** | `vscode` | `6cb58a3eca5` | **Unify agent skills internal architecture** | **BREAKING:** Folder name must match skill name (case-sensitive!) |
| **2026-01-15** | `vscode` | `8a2adc0d0ac` | **Fix missing user prompt files** | **FIX:** Added search in VS Code user data prompts folder |

---

### Key Finding #1: Windows Case-Insensitivity Fix (b3a174eb)

**Commit:** `b3a174ebc6a393a0ba96be52184b96a5b9ff6115`
**Date:** 2025-12-12
**Repo:** vscode-copilot-chat
**Title:** User skill can not be loaded on Windows (#2563)

**Change:**
```diff
-import { isEqualOrParent, joinPath, dirname as uriDirname } from '...resources';
+import { extUriBiasedIgnorePathCase } from '...resources';

-const folderUri = uriDirname(joinPath(extension.extensionUri, contribution.path));
+const folderUri = extUriBiasedIgnorePathCase.dirname(Uri.joinPath(...));

-if (isEqualOrParent(uri, location)) {
+if (extUriBiasedIgnorePathCase.isEqualOrParent(uri, location)) {

-const skillFolderUri = joinPath(this.envService.userHome, SKILL_FOLDER);
+const skillFolderUri = extUriBiasedIgnorePathCase.joinPath(this.envService.userHome, SKILL_FOLDER);
```

**Why This Matters:**
- Windows uses case-insensitive paths (`C:\Users\...` = `c:\users\...`)
- The regular `joinPath` and `isEqualOrParent` functions are case-sensitive
- `extUriBiasedIgnorePathCase` handles Windows case-insensitivity correctly
- Without this fix, skill paths like `C:\Users\Bob\.claude\skills` would NOT match `c:\users\bob\.claude\skills`

---

### Key Finding #2: VS Code Core Uses Case-Sensitive Path Functions ⚠️

**Location:** [promptFilesLocator.ts](../../../vscode/src/vs/workbench/contrib/chat/common/promptSyntax/utils/promptFilesLocator.ts)

**Current code uses case-SENSITIVE functions:**
```typescript
import { basename, dirname, isEqualOrParent, joinPath } from '...resources.js';

// Line 300: Tilde path expansion
const uri = joinPath(userHome, configuredLocation.substring(2));

// Line 323: Relative path resolution
const absolutePath = joinPath(workspaceFolder.uri, configuredLocation);
```

**Problem:**
- The `vscode` repo's `promptFilesLocator.ts` does NOT use `extUriBiasedIgnorePathCase`
- The `vscode-copilot-chat` repo's `customInstructionsService.ts` DOES use it
- This creates an inconsistency where skills may be found via one path but not recognized via another

---

### Key Finding #3: Architecture Change in Commit 6cb58a3eca5

**Date:** 2026-01-14
**Title:** Unify agent skills internal architecture (#286860)

**Major Changes:**

1. **Tilde Path Handling Introduced:**
```typescript
// config.ts - new function
export function isTildePath(path: string): boolean {
    return path.startsWith('~/') || path.startsWith('~\\');
}
```

2. **Default Skill Paths Changed:**
```diff
-// OLD: Separate workspace and home folders
-export const DEFAULT_AGENT_SKILLS_WORKSPACE_FOLDERS = [
-    { path: '.github/skills', type: 'github-workspace' },
-    { path: '.claude/skills', type: 'claude-workspace' }
-];
-export const DEFAULT_AGENT_SKILLS_USER_HOME_FOLDERS = [
-    { path: '.copilot/skills', type: 'copilot-personal' },
-    { path: '.claude/skills', type: 'claude-personal' }
-];

+// NEW: Unified with tilde paths
+export const DEFAULT_SKILL_SOURCE_FOLDERS: readonly IPromptSourceFolder[] = [
+    { path: '.github/skills', source: PromptFileSource.GitHubWorkspace, storage: PromptsStorage.local },
+    { path: '.claude/skills', source: PromptFileSource.ClaudeWorkspace, storage: PromptsStorage.local },
+    { path: '~/.copilot/skills', source: PromptFileSource.CopilotPersonal, storage: PromptsStorage.user },
+    { path: '~/.claude/skills', source: PromptFileSource.ClaudePersonal, storage: PromptsStorage.user },
+];
```

3. **Skill Path Validation Regex:**
```typescript
// Validates skill paths - note Windows handling
export const VALID_SKILL_PATH_PATTERN = '^(?![A-Za-z]:[\\\\/])(?![\\\\/])(?!~(?![\\\\/]))(?!.*[*?\\[\\]{}]).*\\S.*$';
```

This regex:
- Rejects Windows absolute paths like `C:\...`
- Rejects Unix absolute paths like `/...`
- Requires tilde paths to have `/` or `\` after `~`
- Uses `~/` format in defaults (Unix-style)

---

### Potential Windows Issues

1. **Tilde Path Uses Forward Slash:** Default paths use `~/` but Windows users might configure `~\`
   - The `isTildePath()` function handles both
   - But default paths only use `~/`

2. **Path Comparison Case Sensitivity:** VS Code core uses case-sensitive path functions
   - vscode-copilot-chat fixed this in commit `b3a174eb`
   - VS Code core (`promptFilesLocator.ts`) may still have issues

3. **User Home Resolution:** Uses `IPathService.userHome()` which relies on:
   - `os.homedir()` in Node.js
   - `USERPROFILE` environment variable on Windows

---

## Git History Analysis - VS Code Repository (Dec 1, 2025 - Jan 20, 2026)

### All Commits to promptSyntax (Skills Discovery)

| Date | Commit | Description | Impact |
|------|--------|-------------|--------|
| **2025-12-17** | `82fb5e14653` | **Agent Skills cleanup** | Added `chat_preview_features_enabled` check, renamed `findClaudeSkills` → `findAgentSkills` |
| **2025-12-17** | `b76549b5cda` | Add validation for name and description fields | Skills without name/description now REJECTED |
| **2025-12-18** | `89b8c4e9fa4` | **Updates for Agent Skills alignment** | Changed folder paths from strings to objects with metadata |
| 2025-12-31 | `99d8a2706e6` | Reorganize workbench/contrib/chat | Major refactor, file moves |
| 2026-01-09 | `88d46fe33ac` | Add proposed API for prompt files providers | New extension API |
| **2026-01-14** | `6cb58a3eca5` | **Unify agent skills internal architecture** | **BREAKING: Added folder name must match skill name validation** |
| 2026-01-14 | `97b6da89878` | Enable agent skills by default | Changed default from `false` → `true` |
| 2026-01-15 | `a357e63b0b7` | Prompt files provider API V2 | API improvements |
| 2026-01-15 | `12bce8da5a9` | Agent skills management UX and file support | UI changes |
| 2026-01-15 | `03f79ac30bd` | Agent skills extension file contribution API | Extension contributions |
| **2026-01-16** | `87b2e355dd5` | **Remove preview feature requirement for Agent Skills** | Removed `chat_preview_features_enabled` gate |
| 2026-01-17 | `c13eb90a29f` | Load instructions on demand and don't add images | Performance |
| 2026-01-19 | `0fb4f779292` | Allow agents to define which subagents can be used | New feature |

### Git History Analysis - vscode-copilot-chat Repository

The following commits modified skills-related code in the `vscode-copilot-chat` repository during the timeframe when skills stopped working:

| Date | Commit | Description | Files Changed |
|------|--------|-------------|---------------|
| 2025-12-04 | `ea0e8d8c` | Allow reading personal skills, refining isExternalInstructionsFile checks (#2392) | customInstructionsService.ts |
| **2025-12-12** | **`b3a174eb`** | **User skill can not be loaded on Windows (#2563)** | customInstructionsService.ts |
| **2025-12-18** | **`5ce4f46f`** | **Updates for Agent Skills alignment (#2634)** - **BREAKING CHANGES** | customInstructionsService.ts, readFileTool.tsx |
| 2026-01-14 | `cb544ba1` | Custom chat render for agent skills (#2756) | customInstructionsService.ts |
| 2026-01-15 | `73a6bb7e` | Adopt latest provider pattern for org/enterprise custom agents (#2737) | customInstructionsService.ts |

### Critical Change #1: Preview Features Gate Added Then Removed ⚠️⚠️⚠️

**Commit `82fb5e14653` (Dec 17, 2025) ADDED the preview features check:**
```typescript
// promptsServiceImpl.ts - findAgentSkills()
public async findAgentSkills(token: CancellationToken): Promise<IAgentSkill[] | undefined> {
    const useAgentSkills = this.configurationService.getValue(PromptsConfig.USE_AGENT_SKILLS);
    const defaultAccount = await this.defaultAccountService.getDefaultAccount();
    const previewFeaturesEnabled = defaultAccount?.chat_preview_features_enabled ?? true;
    if (useAgentSkills && previewFeaturesEnabled) {  // ← BOTH conditions must be true
        // ... skills processing
    }
    return undefined;  // ← Returns undefined if either is false!
}
```

**Impact:** Between Dec 17, 2025 and Jan 16, 2026, skills required `chat_preview_features_enabled: true` from the user's GitHub account. This was likely causing skills to not work for some users.

**Commit `87b2e355dd5` (Jan 16, 2026) REMOVED the preview features check:**
```diff
-const defaultAccount = await this.defaultAccountService.getDefaultAccount();
-const previewFeaturesEnabled = defaultAccount?.chat_preview_features_enabled ?? true;
-if (useAgentSkills && previewFeaturesEnabled) {
+if (useAgentSkills) {
```

**This may explain why skills stopped working around late December for some users and may have started working again after Jan 16.**

### Critical Change #2: Setting Renamed (Dec 17-18, 2025) ⚠️

**Commit `82fb5e14653` renamed the setting:**
- **Old:** `chat.useClaudeSkills`
- **New:** `chat.useAgentSkills`

**Migration exists but may cause issues:**
```typescript
// vscode/chat.contribution.ts:903-908
{
    key: 'chat.useClaudeSkills',
    migrateFn: (value, _accessor) => ([
        ['chat.useClaudeSkills', { value: undefined }],
        ['chat.useAgentSkills', { value }]  // ← Old value migrates, including FALSE
    ])
}
```

**Impact:** If a user had `chat.useClaudeSkills: false` set (or VS Code stored a default of `false`), the migration carries that `false` value to the new setting, disabling skills.

### Critical Change #3: Folder Paths Changed (Dec 17-18, 2025) ⚠️

**Commit `82fb5e14653` changed where skills are discovered:**

**Before (single folder):**
```typescript
const SKILL_FOLDER = '.claude/skills';
```

**After (multiple folders with metadata):**
```typescript
export const DEFAULT_SKILL_SOURCE_FOLDERS = [
    { path: '.github/skills', source: 'github-workspace', storage: 'local' },
    { path: '.claude/skills', source: 'claude-workspace', storage: 'local' },
    { path: '~/.copilot/skills', source: 'copilot-personal', storage: 'user' },  // NEW!
    { path: '~/.claude/skills', source: 'claude-personal', storage: 'user' },
];
```

**Impact:** The code now looks in additional folders. Users may need to check the new locations.

### Critical Change #4: Default Changed from `false` to `true` (Jan 14, 2026)

**Commit `97b6da89878` changed the default value:**
```diff
[PromptsConfig.USE_AGENT_SKILLS]: {
    type: 'boolean',
-   default: false,
+   default: true,
}
```

**Impact:** After Jan 14, 2026, skills are enabled by default. However, users who had the old default persisted may still have `false`.

### Windows-Specific Fix (Dec 12, 2025)

**Commit `b3a174eb` in vscode-copilot-chat fixed case-sensitivity issues:**
```diff
-import { isEqualOrParent, joinPath, dirname as uriDirname } from '...resources';
+import { extUriBiasedIgnorePathCase } from '...resources';

-const folderUri = uriDirname(joinPath(extension.extensionUri, contribution.path));
+const folderUri = extUriBiasedIgnorePathCase.dirname(Uri.joinPath(extension.extensionUri, contribution.path));

-if (isEqualOrParent(uri, location)) {
+if (extUriBiasedIgnorePathCase.isEqualOrParent(uri, location)) {
```

This fix addressed Windows path case-insensitivity issues. However, skills reportedly worked in mid-December, so this fix was applied before the regression window.

### API Change (Jan 15, 2026)

**Commit `73a6bb7e` in vscode-copilot-chat made `isExternalInstructionsFile` async:**
```diff
-isExternalInstructionsFile(uri: URI): boolean;
+isExternalInstructionsFile(uri: URI): Promise<boolean>;
```

**Impact:** Any code calling this synchronously would break.

---

## 🔴 MOST LIKELY ROOT CAUSES (Based on Git History)

### Timeline of Breaking Changes

```
Dec 17, 2025: chat_preview_features_enabled check ADDED to findAgentSkills()
              ↓
              Users with preview features disabled → skills stop working
              ↓
Jan 16, 2026: chat_preview_features_enabled check REMOVED
              ↓
              Skills should work again for those users
```

**If the user's skills stopped working late December 2025 and the user has NOT updated VS Code since Jan 16, 2026:**
→ The preview features gate is still active and blocking skills.

### Sequence of Events

1. **Dec 17**: Setting renamed + preview features gate added + folder paths changed
2. **Dec 18**: Additional folder path changes + skill type metadata
3. **Jan 14**: Folder name must match skill name validation added + default changed to `true`
4. **Jan 16**: Preview features gate REMOVED

**If the user is on a VS Code version between Dec 17 and Jan 16:**
- `chat_preview_features_enabled` from their GitHub account must be `true`
- This is a server-side flag controlled by GitHub

**If the user is on VS Code version after Jan 14:**
- Skill folder name must EXACTLY match the `name:` attribute in SKILL.md

---

## Root Causes (Ordered by Likelihood)

### 1. 🔴 HIGH: Chat Mode Not Set to Agent

**Impact:** Skills completely missing from prompt
**Likelihood:** Very High

Skills are **only** included in the system prompt when the chat is in Agent mode. The code path:

```
chatWidget.ts:2647 → enabledTools only set if ChatModeKind.Agent
  → computeAutomaticInstructions.ts:240 → _getTool() returns undefined
    → computeAutomaticInstructions.ts:253 → readTool is undefined
      → computeAutomaticInstructions.ts:257 → if (readTool) block SKIPPED
        → Skills section NEVER added
```

**Key Code:**
```typescript
// chatWidget.ts:2647
const enabledTools = this.input.currentModeKind === ChatModeKind.Agent
    ? this.input.selectedToolsModel.userSelectedTools.get()
    : undefined;
```

**Why it differs between devices:**
- Mode is **workspace-scoped** and **NOT synced** between devices
- The `chat.defaultMode` experiment assigns different treatments to different devices
- Once a workspace has a persisted mode (Ask/Edit), it stays that way
- Organization policy can disable Agent mode

**Verification:**
- Check what mode is active in the chat input (look for Ask/Edit/Agent indicator)
- Click mode picker to see if Agent is available

---

### 2. 🟠 MEDIUM-HIGH: Workspace Trust Affecting Restricted Settings

**Impact:** Skills silently disabled
**Likelihood:** Medium-High

Both skills-related settings have `restricted: true`:

```typescript
// chat.contribution.ts
[PromptsConfig.USE_AGENT_SKILLS]: {
    default: true,
    restricted: true,  // ← CRITICAL
},
[PromptsConfig.SKILLS_LOCATION_KEY]: {
    restricted: true,  // ← CRITICAL
}
```

**Behavior:**
- **Trusted workspace:** Workspace settings for restricted settings ARE applied
- **Untrusted workspace:** Workspace settings for restricted settings ARE IGNORED (silently fall back to user/global defaults)

**Scenario causing issue:**
1. User has `chat.useAgentSkills: true` in workspace settings only
2. Workspace is untrusted
3. Setting is silently ignored → skills disabled

**Key Code:**
```typescript
// configuration.ts:699
reparseWorkspaceSettings(): ConfigurationModel {
    this._workspaceConfiguration.reparseWorkspaceSettings({
        scopes: WORKSPACE_SCOPES,
        skipRestricted: this.isUntrusted()  // ← Restricted settings skipped if untrusted
    });
}
```

**Verification:**
- Check if workspace is trusted (look for shield icon in status bar)
- Check where `chat.useAgentSkills` is configured (user vs workspace settings)

---

### 3. 🟠 MEDIUM: Explicit Settings Value

**Impact:** Skills disabled
**Likelihood:** Medium

User explicitly set `chat.useAgentSkills: false` somewhere:

**Possible sources:**
1. User Settings (`settings.json`)
2. Workspace Settings (`.vscode/settings.json`)
3. Migrated from old `chat.useClaudeSkills: false` setting

**Migration code:**
```typescript
// chat.contribution.ts:903-908
{
    key: 'chat.useClaudeSkills',
    migrateFn: (value, _accessor) => ([
        ['chat.useClaudeSkills', { value: undefined }],
        ['chat.useAgentSkills', { value }]  // ← Old false value migrates
    ])
}
```

**Verification:**
- Search for `useAgentSkills` and `useClaudeSkills` in all settings files
- Check Settings UI for "Use Agent Skills" setting

---

### 4. 🟠 MEDIUM: Home Directory Resolution on Windows

**Impact:** Personal skills not found
**Likelihood:** Medium

Personal skill paths (`~/.claude/skills`, `~/.copilot/skills`) depend on `os.homedir()`:

```typescript
// nativeEnvServiceImpl.ts:14-16
get userHome() {
    return URI.file(os.homedir());  // Relies on USERPROFILE env var
}
```

**Windows path resolution:**
1. Node.js `os.homedir()` uses `USERPROFILE` environment variable
2. Falls back to `HOMEDRIVE` + `HOMEPATH`
3. If both are missing, returns empty string

**Potential issues:**
- `USERPROFILE` not set or points to wrong location
- Network home directories causing issues
- OneDrive folder redirection affecting paths

**Key Code for path expansion:**
```typescript
// promptFilesLocator.ts:297-306
if (isTildePath(configuredLocation)) {
    if (userHome) {
        const uri = joinPath(userHome, configuredLocation.substring(2));
        // ... process path
    }
    continue;  // SILENT SKIP if userHome is undefined
}
```

**Verification:**
- Open terminal and run: `echo %USERPROFILE%`
- Verify path exists: `dir C:\Users\<username>\.claude\skills`

---

### 5. 🟡 LOWER: Organization/EMU Policy Disabling Agent Mode

**Impact:** Agent mode unavailable → skills missing
**Likelihood:** Lower (but affects EMU users)

Organization can disable Agent mode via policy:

```typescript
// chat.contribution.ts:553-568
[ChatConfiguration.AgentEnabled]: {
    type: 'boolean',
    default: true,
    policy: {
        name: 'ChatAgentMode',
        category: PolicyCategory.InteractiveSession,
        value: (account) => account.policyData?.chat_agent_enabled === false ? false : undefined,
    }
}
```

**When Agent mode is disabled by policy:**
- Agent mode shows a lock icon in mode picker
- Agent mode cannot be selected
- Skills block will NOT be included

**EMU Token-based Policy:**
```typescript
// chatModes.ts:226-229
private isAgentModeDisabledByPolicy(): boolean {
    return this.configurationService.inspect<boolean>(ChatConfiguration.AgentEnabled).policyValue === false;
}
```

**Verification:**
- Check if Agent mode shows a lock icon
- Check `chat.agent.enabled` setting and its effective value

---

### 6. 🟡 LOWER: `readFile` Tool Disabled

**Impact:** Skills section skipped
**Likelihood:** Lower

Skills section requires the `readFile` tool to be available:

```typescript
// computeAutomaticInstructions.ts:253-257
const readTool = this._getTool('readFile');
if (readTool) {
    // ... skills section added here
}
```

**Verification:**
- Check if `readFile` tool appears in tools list when in Agent mode

---

## Code Locations Reference

| Component | File | Key Lines |
|-----------|------|-----------|
| Mode check | `vscode/src/vs/workbench/contrib/chat/browser/widget/chatWidget.ts` | 2647-2651 |
| Tool gating | `vscode/src/vs/workbench/contrib/chat/common/promptSyntax/computeAutomaticInstructions.ts` | 240-249, 253-257 |
| Skills injection | `vscode/src/vs/workbench/contrib/chat/common/promptSyntax/computeAutomaticInstructions.ts` | 307-320 |
| Skills discovery | `vscode/src/vs/workbench/contrib/chat/common/promptSyntax/service/promptsServiceImpl.ts` | 755-814 |
| Settings definition | `vscode/src/vs/workbench/contrib/chat/browser/chat.contribution.ts` | 735-770 |
| Workspace trust | `vscode/src/vs/workbench/services/configuration/browser/configuration.ts` | 693-700 |
| Agent policy | `vscode/src/vs/workbench/contrib/chat/common/chatModes.ts` | 226-229 |
| Home directory | `vscode-copilot-chat/src/platform/env/vscode-node/nativeEnvServiceImpl.ts` | 14-16 |
| Skill folder paths | `vscode-copilot-chat/src/platform/customInstructions/common/customInstructionsService.ts` | 83-84, 188-202 |

---

## Default Skill Locations

```typescript
// promptFileLocations.ts:97-102
export const DEFAULT_SKILL_SOURCE_FOLDERS = [
    { path: '.github/skills', source: 'github-workspace', storage: 'local' },
    { path: '.claude/skills', source: 'claude-workspace', storage: 'local' },
    { path: '~/.copilot/skills', source: 'copilot-personal', storage: 'user' },
    { path: '~/.claude/skills', source: 'claude-personal', storage: 'user' },
];
```

---

## Telemetry

Skills discovery emits the `agentSkillsFound` telemetry event with:
- `totalSkillsFound`: Number of skills discovered
- `claudePersonal`, `claudeWorkspace`, `copilotPersonal`, `githubWorkspace`: Per-source counts
- `skippedMissingName`, `skippedNameMismatch`, `skippedParseFailed`: Skip reasons

**Note:** This telemetry only fires when `findAgentSkills()` is called, which requires Agent mode.

---

## Diagnostic Commands

Users can check:
1. **Output Channel:** View → Output → "GitHub Copilot Chat"
2. **Developer Tools:** Help → Toggle Developer Tools → Console
3. **Settings:** Check `chat.useAgentSkills` effective value
4. **Workspace Trust:** Look for shield icon in status bar

---

## NEW FINDING: Skill Validation Requirements Added (Dec 2025 - Jan 2026)

**CRITICAL:** Multiple validation requirements were added in December 2025 and January 2026 that can cause skills to be silently skipped.

### Timeline of Changes

| Date | Commit | Change |
|------|--------|--------|
| Dec 17, 2025 | `b76549b5cda` | Added name and description validation |
| Dec 18, 2025 | `89b8c4e9fa4` | Added telemetry for skipped skills, duplicate name detection |
| Jan 15, 2026 | `6cb58a3eca5` | Added **folder name matching requirement** |
| Jan 15, 2026 | `03f79ac30bd` | Consolidated validation into `SkillNameMismatchError` class |

### Current Validation Requirements

Skills are validated in `promptsServiceImpl.ts` and **silently skipped** if any check fails:

#### 1. **Required `name` Attribute**
```yaml
---
name: "skill-name"  # ← REQUIRED
---
```
**Error Type:** `SkillMissingNameError`

#### 2. **Required `description` Attribute**
```yaml
---
name: "skill-name"
description: "What this skill does"  # ← REQUIRED
---
```
**Error Type:** `SkillMissingDescriptionError`

#### 3. **⚠️ NEW: Folder Name Must Match Skill Name**
```
~/.claude/skills/my-skill/SKILL.md
                ^^^^^^^^
                └── Folder name must EXACTLY match the `name:` attribute

---
name: "my-skill"  # ← Must match parent folder name
description: "..."
---
```
**Error Type:** `SkillNameMismatchError`

This was added in commit `6cb58a3eca5` (Jan 15, 2026) per "agentskills.io specification".

### Validation Code

```typescript
// promptsServiceImpl.ts:715-719
// Validate that the sanitized name matches the parent folder name (per agentskills.io specification)
const skillFolderUri = dirname(uri);
const folderName = basename(skillFolderUri);
if (sanitizedName !== folderName) {
    this.logger.error(`[validateAndSanitizeSkillFile] Agent skill name "${sanitizedName}" does not match folder name "${folderName}": ${uri}`);
    throw new SkillNameMismatchError(uri, sanitizedName, folderName);
}
```

### Skip Counters in Telemetry

```typescript
let skippedMissingName = 0;        // No name attribute
let skippedMissingDescription = 0; // No description attribute
let skippedDuplicateName = 0;      // Duplicate skill name
let skippedParseFailed = 0;        // YAML parse error
let skippedNameMismatch = 0;       // Folder name != skill name
```

### Why This Breaks Existing Skills

1. **Old skills without `description:`** — Now skipped
2. **Folder name != skill name** — Very likely for user-created skills:
   ```
   ~/.claude/skills/my-awesome-skill/SKILL.md
   ---
   name: "My Awesome Skill"  # MISMATCH! Expected "my-awesome-skill"
   ---
   ```
3. **Case sensitivity on Windows may or may not help** — The comparison is exact string match

### Recommended Fix for Users

Ensure SKILL.md follows:
```
~/.claude/skills/<exact-skill-name>/SKILL.md
                                    ↓
---
name: "<exact-skill-name>"   # Must match folder name exactly
description: "A description" # Required
---
# Skill content...
```

---

## Settings Schema Changes (Dec 2025 - Jan 2026)

### Summary of Schema Changes

| Date | Commit | Setting | Change Type | Impact |
|------|--------|---------|-------------|--------|
| Dec 17, 2025 | `82fb5e14653` | `chat.useAgentSkills` | Settings cleanup | Removed `experimental` tag |
| Dec 17, 2025 | `8a279596e95` | `chat.useAgentSkills` | Description update | Changed description text |
| Dec 18, 2025 | `89b8c4e9fa4` | Internal | Metadata change | Folder paths became objects with metadata |
| Jan 14, 2026 | `6cb58a3eca5` | `chat.agentSkillsLocations` | **NEW SETTING** | Added skill locations configuration with path validation |
| Jan 14, 2026 | `97b6da89878` | `chat.useAgentSkills` | **Default changed** | `false` → `true` |

### 1. New Setting: `chat.agentSkillsLocations` (Jan 14, 2026)

**Commit:** `6cb58a3eca5` - "Unify agent skills internal architecture"

```typescript
[PromptsConfig.SKILLS_LOCATION_KEY]: {
    type: 'object',
    title: "Agent Skills Locations",
    markdownDescription: "Specify where agent skills are located...",
    default: {
        '.github/skills': true,
        '.claude/skills': true,
        '~/.copilot/skills': true,
        '~/.claude/skills': true,
    },
    additionalProperties: { type: 'boolean' },
    propertyNames: {
        pattern: VALID_SKILL_PATH_PATTERN,  // ← NEW validation pattern
        patternErrorMessage: "Skill location paths must either be relative paths or start with '~' for user home directory.",
    },
    restricted: true,  // ← Requires workspace trust
    tags: ['prompts', 'reusable prompts', 'prompt snippets', 'instructions'],
}
```

### 2. New Path Validation Pattern: `VALID_SKILL_PATH_PATTERN`

**Location:** `promptFilesLocator.ts:667`

```typescript
export const VALID_SKILL_PATH_PATTERN = '^(?![A-Za-z]:[\\\\/])(?![\\\\/])(?!~(?![\\\\/]))(?!.*[*?\\[\\]{}]).*\\S.*$';
```

**What this regex validates:**
- ❌ Not a Windows absolute path (e.g., `C:\Users\...`)
- ❌ Not a Unix absolute path (e.g., `/home/user/...`)
- ✅ Tilde paths must be followed by `/` or `\` (e.g., `~/skills` or `~\skills`)
- ❌ No glob pattern characters: `* ? [ ] { }`
- ✅ Must have at least one non-whitespace character

**Test Results:**

| Path | Valid? |
|------|--------|
| `.github/skills` | ✅ |
| `.claude/skills` | ✅ |
| `~/.copilot/skills` | ✅ |
| `~/.claude/skills` | ✅ |
| `./my-skills` | ✅ |
| `../shared-skills` | ✅ |
| `~/skills` | ✅ |
| `~\skills` | ✅ |
| `my-skills` | ✅ |
| `C:\Users\test\skills` | ❌ (Windows absolute) |
| `/home/user/skills` | ❌ (Unix absolute) |
| `~invalid` | ❌ (Tilde not followed by `/` or `\`) |

### 3. `restricted: true` Property on Skills Settings

**Both skill-related settings have `restricted: true`:**

```typescript
[PromptsConfig.USE_AGENT_SKILLS]: {
    default: true,
    restricted: true,         // ← CRITICAL
    disallowConfigurationDefault: true,
},
[PromptsConfig.SKILLS_LOCATION_KEY]: {
    restricted: true,         // ← CRITICAL
},
```

**Behavior of `restricted` settings:**
- **Trusted workspace:** Settings from workspace `.vscode/settings.json` ARE applied
- **Untrusted workspace:** Workspace settings ARE IGNORED, falls back to user/global defaults

**⚠️ Impact:**
If a user configured skill paths in workspace settings and the workspace is untrusted, those paths will be silently ignored.

### 4. `disallowConfigurationDefault: true`

```typescript
[PromptsConfig.USE_AGENT_SKILLS]: {
    default: true,
    disallowConfigurationDefault: true,  // ← Prevents extensions from changing default
}
```

This prevents extensions from overriding the default value via `configurationDefaults` in their package.json.

### 5. Skill Path Validation Logic

**File:** `promptFilesLocator.ts:349-365`

```typescript
private toAbsoluteLocationsForSkills(
    configuredLocations: readonly IPromptSourceFolder[],
    userHome: URI | undefined
): readonly IResolvedPromptSourceFolder[] {
    // Filter and validate skill paths before resolving
    const validLocations = configuredLocations.filter(sourceFolder => {
        const configuredLocation = sourceFolder.path;
        if (!isValidSkillPath(configuredLocation)) {
            this.logService.warn(`Skipping invalid skill path (glob patterns and absolute paths not supported): ${configuredLocation}`);
            return false;  // ← SILENTLY SKIPPED (only logged as warning)
        }
        return true;
    });

    // Use the standard resolution logic for valid paths
    return this.toAbsoluteLocations(validLocations, userHome);
}
```

**Key behaviors:**
1. Invalid paths are **silently skipped** (only logged as warning)
2. Absolute paths (both Windows and Unix) are rejected
3. Glob patterns are rejected
4. User home resolution requires `userHome` parameter

### 6. Tilde Path Expansion

**File:** `config.ts:257-267`

```typescript
export function isTildePath(path: string): boolean {
    return path.startsWith('~/') || path.startsWith('~\\');
}
```

**And in `promptFilesLocator.ts:297-306`:**

```typescript
if (isTildePath(configuredLocation)) {
    // If userHome is not provided, we cannot resolve tilde paths so we skip this entry
    if (userHome) {
        const uri = joinPath(userHome, configuredLocation.substring(2));
        // ... add to results
    }
    continue;  // ← SILENTLY SKIPS if userHome is undefined
}
```

**⚠️ Windows-specific concern:**
- Both `~/` and `~\` are supported for tilde paths
- If `userHome` cannot be resolved, tilde paths are silently skipped
- `userHome` comes from `os.homedir()` which depends on `USERPROFILE` environment variable

### 7. Default Skill Locations Changed

**Before (single constant):**
```typescript
const SKILL_FOLDER = '.claude/skills';
```

**After (multiple locations with metadata):**
```typescript
export const DEFAULT_SKILL_SOURCE_FOLDERS: readonly IPromptSourceFolder[] = [
    { path: '.github/skills', source: PromptFileSource.GitHubWorkspace, storage: PromptsStorage.local },
    { path: '.claude/skills', source: PromptFileSource.ClaudeWorkspace, storage: PromptsStorage.local },
    { path: '~/.copilot/skills', source: PromptFileSource.CopilotPersonal, storage: PromptsStorage.user },
    { path: '~/.claude/skills', source: PromptFileSource.ClaudePersonal, storage: PromptsStorage.user },
];
```

**Impact:**
- More locations are searched by default
- Locations now have explicit `storage` type (`local` vs `user`)
- `user` storage paths require `userHome` resolution

---

## Potential Settings-Related Causes

### 1. 🔴 Invalid Path in `chat.agentSkillsLocations`

If a user manually configured a path that doesn't pass the `VALID_SKILL_PATH_PATTERN` validation:
- Path is silently skipped with only a warning log
- Skills in that location won't be discovered

**Example:**
```json
{
    "chat.agentSkillsLocations": {
        "C:\\Users\\me\\skills": true  // ← Invalid! Absolute path rejected
    }
}
```

### 2. 🔴 Workspace Trust + Restricted Settings

Skills settings have `restricted: true`. In an untrusted workspace:
- `chat.useAgentSkills` workspace setting is ignored
- `chat.agentSkillsLocations` workspace setting is ignored
- Falls back to user settings or defaults

### 3. 🟠 Migrated `chat.useClaudeSkills: false`

The migration from `chat.useClaudeSkills` to `chat.useAgentSkills` preserves the value:
```typescript
migrateFn: (value, _accessor) => ([
    ['chat.useClaudeSkills', { value: undefined }],
    ['chat.useAgentSkills', { value }]  // ← false migrates as false
])
```

If a user had the old setting set to `false`, skills remain disabled after migration.

### 4. 🟠 `userHome` Resolution Failure on Windows

If `os.homedir()` returns an unexpected value on Windows:
- `~/.claude/skills` and `~/.copilot/skills` won't resolve
- Personal skills won't be discovered
- Only workspace skills (`.github/skills`, `.claude/skills`) would work

---

## Skill Locations Configuration - Detailed Analysis

### 1. Setting Definition: `chat.agentSkillsLocations`

**File:** `vscode/src/vs/workbench/contrib/chat/browser/chat.contribution.ts:744-767`

```typescript
[PromptsConfig.SKILLS_LOCATION_KEY]: {
    type: 'object',
    title: "Agent Skills Locations",
    markdownDescription: "Specify where agent skills are located...",
    default: {
        ...DEFAULT_SKILL_SOURCE_FOLDERS.map((folder) => ({ [folder.path]: true })).reduce(...)
    },
    additionalProperties: { type: 'boolean' },
    propertyNames: {
        pattern: VALID_SKILL_PATH_PATTERN,  // ← PATH VALIDATION REGEX
        patternErrorMessage: "Skill location paths must either be relative paths or start with '~' for user home directory."
    },
    restricted: true,  // ← REQUIRES WORKSPACE TRUST
}
```

### 2. Default Skill Locations

**File:** `vscode/src/vs/workbench/contrib/chat/common/promptSyntax/config/promptFileLocations.ts:106-112`

```typescript
export const DEFAULT_SKILL_SOURCE_FOLDERS: readonly IPromptSourceFolder[] = [
    { path: '.github/skills',    source: PromptFileSource.GitHubWorkspace,  storage: PromptsStorage.local },
    { path: '.claude/skills',    source: PromptFileSource.ClaudeWorkspace,  storage: PromptsStorage.local },
    { path: '~/.copilot/skills', source: PromptFileSource.CopilotPersonal,  storage: PromptsStorage.user },
    { path: '~/.claude/skills',  source: PromptFileSource.ClaudePersonal,   storage: PromptsStorage.user },
];
```

**✅ `.claude/skills` IS in the default locations** - both workspace and personal (`~/.claude/skills`).

### 3. Path Validation Regex: `VALID_SKILL_PATH_PATTERN`

**File:** `vscode/src/vs/workbench/contrib/chat/common/promptSyntax/utils/promptFilesLocator.ts:665-676`

```typescript
/**
 * The regex validates:
 * - Not a Windows absolute path (e.g., C:\)
 * - Not starting with / (Unix absolute path)
 * - If starts with ~, must be followed by / or \
 * - No glob pattern characters: * ? [ ] { }
 * - At least one non-whitespace character
 */
export const VALID_SKILL_PATH_PATTERN = '^(?![A-Za-z]:[\\\\/])(?![\\\\/])(?!~(?![\\\\/]))(?!.*[*?\\[\\]{}]).*\\S.*$';

export function isValidSkillPath(path: string): boolean {
    return new RegExp(VALID_SKILL_PATH_PATTERN).test(path);
}
```

**Rejected Paths:**
| Path Type | Example | Reason |
|-----------|---------|--------|
| Windows absolute | `C:\Users\me\skills` | Starts with drive letter |
| Unix absolute | `/home/user/skills` | Starts with `/` |
| Invalid tilde | `~skills`, `~.config` | Tilde not followed by `/` or `\` |
| Glob patterns | `skills/*`, `**/*.md` | Contains `*`, `?`, `[`, `]`, `{`, `}` |
| Empty/whitespace | `""`, `"   "` | No non-whitespace characters |

**Accepted Paths:**
| Path Type | Example |
|-----------|---------|
| Relative | `my-skills`, `./my-skills`, `folder/subfolder` |
| Parent relative | `../shared-skills`, `../../common/skills` |
| Tilde (user home) | `~/skills`, `~/.claude/skills`, `~\.claude\skills` |

### 4. Path Resolution Flow

**File:** `vscode/src/vs/workbench/contrib/chat/common/promptSyntax/utils/promptFilesLocator.ts:349-363`

```typescript
private toAbsoluteLocationsForSkills(configuredLocations, userHome) {
    // Step 1: Filter invalid paths
    const validLocations = configuredLocations.filter(sourceFolder => {
        if (!isValidSkillPath(sourceFolder.path)) {
            this.logService.warn(`Skipping invalid skill path: ${sourceFolder.path}`);
            return false;  // ← SILENT SKIP WITH LOG WARNING
        }
        return true;
    });

    // Step 2: Resolve valid paths
    return this.toAbsoluteLocations(validLocations, userHome);
}
```

### 5. Tilde Expansion for `userHome`

**File:** `vscode/src/vs/workbench/contrib/chat/common/promptSyntax/utils/promptFilesLocator.ts:297-306`

```typescript
if (isTildePath(configuredLocation)) {
    if (userHome) {  // ← REQUIRES userHome TO BE DEFINED
        const uri = joinPath(userHome, configuredLocation.substring(2));
        // ... add to results
    }
    continue;  // ← SILENTLY SKIPS if userHome is undefined
}
```

### 6. `userHome` Resolution Chain

**Windows (Electron):**
```
NativePathService constructor
  ↓
environmentService.userHome  (INativeWorkbenchEnvironmentService)
  ↓
AbstractEnvironmentService.userHome  (common/environmentService.ts:50)
  ↓
URI.file(this.paths.homeDir)
  ↓
configuration.homeDir  (from main process)
  ↓
os.homedir()  (Node.js)
  ↓
USERPROFILE environment variable (Windows)
```

**Fallback chain for `os.homedir()` on Windows:**
1. `%USERPROFILE%` environment variable
2. `%HOMEDRIVE%` + `%HOMEPATH%`
3. Empty string if both are missing

### 7. Potential Windows-Specific Issues

| Issue | Symptom | Verification |
|-------|---------|--------------|
| `USERPROFILE` not set | Personal skills not found | `echo %USERPROFILE%` in terminal |
| Wrong `USERPROFILE` value | Personal skills in wrong location | Check value matches actual home |
| OneDrive folder redirection | Skills folder moved unexpectedly | Check if `%USERPROFILE%\.claude` exists |
| Network home directory | Slow or failed skill loading | Check if home is on network share |
| Special characters in path | URI parsing issues | Check for spaces, non-ASCII in username |

### 8. How Settings Can Override Defaults

The `chat.agentSkillsLocations` setting is an **object map** where:
- Keys are paths
- Values are `true` (enabled) or `false` (disabled)

**To override and exclude `.claude/skills`:**
```json
{
    "chat.agentSkillsLocations": {
        ".github/skills": true,
        ".claude/skills": false,  // ← EXPLICITLY DISABLED
        "~/.copilot/skills": true,
        "~/.claude/skills": true
    }
}
```

**To use ONLY custom paths:**
```json
{
    "chat.agentSkillsLocations": {
        "my-custom-skills": true  // ← All defaults omitted = defaults still apply
    }
}
```

**Note:** If defaults are not explicitly set to `false`, they are STILL ENABLED. The setting merges with defaults.

---

## Skill Discovery Mechanism Deep Dive

### Search Mechanism Analysis

**This section addresses the specific question: Could the glob pattern `*/${SKILL_FILENAME}` cause issues on Windows?**

#### Answer: NO - The glob pattern handling is Windows-compatible

The investigation confirms the skill discovery mechanism correctly handles Windows paths:

### 1. Call Chain for Skill Discovery

```
findAgentSkills() [promptsServiceImpl.ts#L686]
    ↓
fileLocator.findAgentSkills(token) [promptFilesLocator.ts#L516-527]
    ↓
findAgentSkillsInFolder(uri, token) [promptFilesLocator.ts#L505-512]
    ↓
searchFilesInLocation(uri, `*/SKILL.md`, token) [promptFilesLocator.ts#L391-410]
    ↓
this.searchService.fileSearch(searchOptions, token) [ISearchService]
```

### 2. The `searchFilesInLocation` Implementation

**File:** [promptFilesLocator.ts#L391-410](vscode/src/vs/workbench/contrib/chat/common/promptSyntax/utils/promptFilesLocator.ts#L391-L410)

```typescript
private async searchFilesInLocation(folder: URI, filePattern: string | undefined, token: CancellationToken): Promise<URI[]> {
    const disregardIgnoreFiles = this.configService.getValue<boolean>('explorer.excludeGitIgnore');
    const workspaceRoot = this.workspaceService.getWorkspaceFolder(folder);

    const searchOptions: IFileQuery = {
        folderQueries: [{ folder, disregardIgnoreFiles }],
        type: QueryType.File,
        shouldGlobMatchFilePattern: true,  // ← CRITICAL: enables glob matching mode
        excludePattern: workspaceRoot ? getExcludePattern(workspaceRoot.uri) : undefined,
        ignoreGlobCase: true,              // ← Case-insensitive glob matching
        sortByScore: true,
        filePattern                         // ← Pattern: "*/SKILL.md"
    };

    const searchResult = await this.searchService.fileSearch(searchOptions, token);
    return searchResult.results.map(r => r.resource);
}
```

### 3. How `shouldGlobMatchFilePattern: true` Affects Matching

When `shouldGlobMatchFilePattern` is true, the search engine uses `glob.match()` instead of fuzzy matching:

**File:** [search.ts#L650-655](vscode/src/vs/workbench/services/search/common/search.ts#L650-L655)

```typescript
export function isFilePatternMatch(candidate: IRawFileMatch, filePatternToUse: string, fuzzy = true): boolean {
    const pathToMatch = candidate.searchPath ? candidate.searchPath : candidate.relativePath;
    return fuzzy ?
        fuzzyContains(pathToMatch, filePatternToUse) :
        glob.match(filePatternToUse, pathToMatch);  // ← When shouldGlobMatchFilePattern=true
}
```

### 4. Glob Pattern Regex Supports Both Path Separators

**File:** [glob.ts#L47-48](vscode/src/vs/base/common/glob.ts#L47-L48)

```typescript
const PATH_REGEX = '[/\\\\]';        // Matches BOTH / and \
const NO_PATH_REGEX = '[^/\\\\]';    // Matches anything EXCEPT / and \
```

The regex generated for pattern `*/SKILL.md` handles both separators:
- `skill-creator/SKILL.md` ✓ matches
- `skill-creator\SKILL.md` ✓ matches

### 5. Ripgrep (rg) Handles Platform-Native Paths

**File:** [ripgrepFileSearch.ts#L20-28](vscode/src/vs/workbench/services/search/node/ripgrepFileSearch.ts#L20-L28)

```typescript
export function spawnRipgrepCmd(config, folderQuery, includePattern, excludePattern, numThreads) {
    const rgArgs = getRgArgs(config, folderQuery, includePattern, excludePattern, numThreads);
    const cwd = folderQuery.folder.fsPath;  // ← Uses platform-native path
    return {
        cmd: cp.spawn(rgDiskPath, rgArgs.args, { cwd }),
        // ...
    };
}
```

Ripgrep:
1. Receives the glob pattern via `-g */SKILL.md`
2. Searches using platform-native paths
3. Returns results with platform-native separators
4. VS Code's glob matcher then handles both `/` and `\` in matching

### 6. Path Normalization in ripgrepFileSearch.ts

**File:** [ripgrepFileSearch.ts#L141-147](vscode/src/vs/workbench/services/search/node/ripgrepFileSearch.ts#L141-L147)

```typescript
// glob.ts requires forward slashes, but a UNC path still must start with \\
if (key.startsWith('\\\\')) {
    key = '\\\\' + key.substr(2).replace(/\\/g, '/');
} else {
    key = key.replace(/\\/g, '/');  // ← Normalizes backslashes to forward slashes for glob
}
```

### 7. Conclusion: Glob Pattern is NOT the Issue

The skill discovery search mechanism:
1. ✅ Uses ripgrep which handles Windows paths correctly
2. ✅ Normalizes paths for glob matching
3. ✅ `glob.ts` regex supports both `/` and `\` separators
4. ✅ `ignoreGlobCase: true` handles case-insensitive matching

**The root cause must be elsewhere** - most likely:
- Configuration not enabled
- Skill validation failures (name/description missing, folder name mismatch)
- Workspace trust blocking restricted settings
- `userHome` resolution issues for personal skill paths

---

## Related Files

- [REMEDIATION_STEPS.md](./REMEDIATION_STEPS.md) - User-facing troubleshooting guide
- [RECOMMENDED_FIXES.md](./RECOMMENDED_FIXES.md) - Proposed code changes
