# Skills Missing - User Remediation Steps (UPDATED Jan 20, 2026)

This guide helps users troubleshoot when the `<skills>` section is missing from the GitHub Copilot Chat system prompt.

---

## 🔴 EXECUTIVE SUMMARY

**Most likely causes for Windows users (in order of probability):**

1. **VS Code version too old** - Must be newer than Jan 16, 2026 (for EMU accounts)
2. **BOM (Byte Order Mark) in skill file** - Windows Notepad adds this by default
3. **Trailing whitespace in `name:` field** - `name: skill-creator ` ≠ folder `skill-creator`
4. **`chat.useAgentSkills` setting is `false`** - Check settings
5. **`readFile` tool disabled** - Required for skills to appear

---

## 🔵 ARCHITECTURE: How Skills Flow from VS Code Core to Extension

### Data Flow Diagram

```
┌────────────────────────────────────────────────────────────────────────────────┐
│                              VS Code Core                                       │
├────────────────────────────────────────────────────────────────────────────────┤
│                                                                                 │
│  1. promptsServiceImpl.ts::findAgentSkills()                                   │
│     ├── Checks: chat.useAgentSkills setting                                    │
│     ├── Calls: fileLocator.findAgentSkills() for workspace/personal skills     │
│     ├── Calls: getExtensionPromptFiles(skill) for extension-contributed skills │
│     └── Validates: name matches folder, description exists                      │
│                                                                                 │
│  2. promptFilesLocator.ts::findAgentSkills()                                   │
│     ├── Searches: .github/skills/, .claude/skills/ (workspace)                 │
│     ├── Searches: ~/.copilot/skills/, ~/.claude/skills/ (personal)             │
│     └── Pattern: */SKILL.md                                                     │
│                                                                                 │
│  3. computeAutomaticInstructions.ts::collect()                                 │
│     ├── Calls: findAgentSkills()                                               │
│     ├── Renders: <skills>...</skills> XML block                                │
│     └── Returns: IPromptTextVariableEntry with id="vscode.prompt.instructions.text" │
│                                                                                 │
│  4. chatWidget.ts (submits request)                                            │
│     └── Passes: ChatRequestVariableSet containing instruction entries          │
│                                                                                 │
└────────────────────────────────────────────────────────────────────────────────┘
                                        │
                                        ▼
┌────────────────────────────────────────────────────────────────────────────────┐
│                           Extension Host API                                    │
├────────────────────────────────────────────────────────────────────────────────┤
│                                                                                 │
│  ChatRequest.references[] contains:                                            │
│  - vscode.prompt.instructions.text → skills list as XML string                 │
│  - vscode.prompt.instructions.root__<URI> → instruction files                  │
│  - vscode.prompt.instructions__<URI> → referenced instruction files            │
│                                                                                 │
└────────────────────────────────────────────────────────────────────────────────┘
                                        │
                                        ▼
┌────────────────────────────────────────────────────────────────────────────────┐
│                         Copilot Chat Extension                                  │
├────────────────────────────────────────────────────────────────────────────────┤
│                                                                                 │
│  1. chatParticipantRequestHandler.ts                                           │
│     └── Creates: ChatVariablesCollection(request.references)                   │
│                                                                                 │
│  2. chatVariablesCollection.ts::isPromptInstruction()                          │
│     └── Checks: variable.reference.id.startsWith('vscode.prompt.instructions') │
│                                                                                 │
│  3. customInstructions.tsx::render()                                           │
│     ├── Iterates: chatVariables with isPromptInstruction()                     │
│     ├── For string values: Includes directly in prompt                         │
│     └── For URI values: Reads file content                                     │
│                                                                                 │
│  4. Extension ALSO has its own skill path logic in:                            │
│     └── customInstructionsService.ts                                           │
│         ├── Defines: PERSONAL_SKILL_FOLDERS = ['.copilot/skills', '.claude/skills'] │
│         ├── Defines: WORKSPACE_SKILL_FOLDERS = ['.github/skills', '.claude/skills'] │
│         └── Checks: USE_AGENT_SKILLS_SETTING = 'chat.useAgentSkills'           │
│                                                                                 │
└────────────────────────────────────────────────────────────────────────────────┘
```

### Key Insight: Skills are NOT Discovered by the Extension

The extension does NOT independently discover skills. Instead:

1. **VS Code core** discovers skills via `promptsServiceImpl.findAgentSkills()`
2. **VS Code core** renders them into `<skills>` XML via `computeAutomaticInstructions`
3. **VS Code core** passes them as `vscode.prompt.instructions.text` in `request.references`
4. **The extension** receives them via the VS Code extension API as `ChatPromptReference`
5. **The extension** includes them in the prompt via `customInstructions.tsx`

### Critical Files (VS Code Core)

| File | Purpose |
|------|---------|
| [promptsServiceImpl.ts](../../../vscode/src/vs/workbench/contrib/chat/common/promptSyntax/service/promptsServiceImpl.ts#L755) | `findAgentSkills()` - discovers and validates skills |
| [promptFilesLocator.ts](../../../vscode/src/vs/workbench/contrib/chat/common/promptSyntax/utils/promptFilesLocator.ts#L519) | `findAgentSkills()` - file system discovery |
| [computeAutomaticInstructions.ts](../../../vscode/src/vs/workbench/contrib/chat/common/promptSyntax/computeAutomaticInstructions.ts#L309) | Renders `<skills>` XML block |
| [chatVariableEntries.ts](../../../vscode/src/vs/workbench/contrib/chat/common/attachments/chatVariableEntries.ts#L463) | Defines `vscode.prompt.instructions.text` entry |

### Critical Files (Extension)

| File | Purpose |
|------|---------|
| [customInstructions.tsx](src/extension/prompts/node/panel/customInstructions.tsx) | Renders instructions including skills into prompt |
| [chatVariablesCollection.ts](src/extension/prompt/common/chatVariablesCollection.ts#L109) | `isPromptInstruction()` filter |
| [customInstructionsService.ts](src/platform/customInstructions/common/customInstructionsService.ts#L85) | Extension-side skill path definitions |

### What Could Block Skills on Extension Side

1. **`isPromptInstruction()` check fails** - If the reference ID doesn't start with `vscode.prompt.instructions`
2. **`chatVariables` is empty or filtered** - If request.references doesn't contain the skill entries
3. **`CustomInstructions` component not rendered** - If `includeCodeGenerationInstructions` is `false`
4. **Extension's `chat.useAgentSkills` check** - The extension also checks this setting independently

### Debugging: Where to Add Logging

To debug skill flow, add logging at these points:

1. **VS Code Core** - `promptsServiceImpl.ts::findAgentSkills()` line 756
2. **VS Code Core** - `computeAutomaticInstructions.ts::_getInstructionsWithPatternsList()` line 309
3. **Extension** - `customInstructions.tsx::render()` - log `chatVariables` contents

---

## Quick Diagnostic Steps

### Step 1: Check for BOM in Skill File (WINDOWS CRITICAL)

Windows Notepad and some editors add a UTF-8 BOM (Byte Order Mark) to files, which breaks YAML parsing.

**Check with PowerShell:**
```powershell
$file = "$env:USERPROFILE\.claude\skills\<your-skill-folder>\SKILL.md"
$bytes = [System.IO.File]::ReadAllBytes($file)
if ($bytes[0] -eq 0xEF -and $bytes[1] -eq 0xBB -and $bytes[2] -eq 0xBF) {
    Write-Host "❌ BOM DETECTED - This will break skill loading!" -ForegroundColor Red
} else {
    Write-Host "✅ No BOM - Good!" -ForegroundColor Green
}
```

**Fix BOM issue:**
```powershell
# Remove BOM from skill file
$file = "$env:USERPROFILE\.claude\skills\<your-skill-folder>\SKILL.md"
$content = Get-Content $file -Raw
$utf8NoBOM = New-Object System.Text.UTF8Encoding $false
[System.IO.File]::WriteAllText($file, $content, $utf8NoBOM)
Write-Host "BOM removed successfully!"
```

### Step 2: Check for Trailing Whitespace in `name:` Field

```powershell
$file = "$env:USERPROFILE\.claude\skills\<your-skill-folder>\SKILL.md"
$content = Get-Content $file -Raw
if ($content -match 'name:\s*([^\r\n]+)[\r\n]') {
    $name = $Matches[1]
    if ($name -ne $name.Trim()) {
        Write-Host "❌ TRAILING WHITESPACE in name: '$name'" -ForegroundColor Red
    } else {
        Write-Host "✅ No trailing whitespace in name" -ForegroundColor Green
    }
}
```

### Step 3: Check `chat.useAgentSkills` Setting

```
1. Ctrl+Shift+P → "Preferences: Open Settings (JSON)"
2. Search for "useAgentSkills"
3. If you see "chat.useAgentSkills": false, change it to true
4. Also check for old "chat.useClaudeSkills": false
```

### Step 4: Check VS Code Output for Errors

```
1. View → Output (Ctrl+Shift+U)
2. Select "Window" from the dropdown
3. Search for "[findAgentSkills]"
4. Look for error messages like:
   - "does not match folder name"
   - "missing name attribute"
   - "Failed to validate"
```

---

## 🔴 MOST LIKELY CAUSE: Folder Name Validation

Since January 15, 2026, VS Code requires that the **folder name exactly matches the `name:` attribute** in your SKILL.md file. This is a **case-sensitive** comparison.

### Example of the Problem

Your skill folder structure:
```
C:\Users\YourName\.claude\skills\
    └── My-Awesome-Skill\          <-- Folder name (note capital letters)
        └── SKILL.md
```

Your SKILL.md file:
```markdown
---
name: my-awesome-skill             <-- This MUST match folder name exactly!
description: Does awesome things
---
```

**The Problem:** `My-Awesome-Skill` ≠ `my-awesome-skill` (case mismatch!)

### The Fix

**Option A: Rename the folder to match the `name:` attribute:**
```cmd
cd C:\Users\YourName\.claude\skills
ren "My-Awesome-Skill" "my-awesome-skill"
```

**Option B: Change the `name:` attribute to match the folder:**
```markdown
---
name: My-Awesome-Skill             <-- Now matches folder name
description: Does awesome things
---
```

---

## Quick Checklist (Updated)

- [ ] **Folder name matches `name:` attribute exactly** (case-sensitive!)
- [ ] SKILL.md has a `name:` attribute in the YAML header
- [ ] SKILL.md has a `description:` attribute in the YAML header (now required)
- [ ] You are in **Agent mode** (not Ask or Edit mode)
- [ ] Workspace is trusted

---

## Step 1: Verify Skill File Format (CRITICAL)

**Since December 17, 2025**, skills require BOTH `name:` AND `description:` fields.
**Since January 15, 2026**, the folder name MUST exactly match the `name:` field (case-sensitive).

### Required SKILL.md Format

```
<skill-folder>/           <-- Folder name
    └── SKILL.md
```

The SKILL.md file MUST have:
```markdown
---
name: <skill-folder>      <-- MUST exactly match parent folder name (case-sensitive!)
description: <text>       <-- REQUIRED since Dec 17, 2025
---

# Your skill content here
```

### Examples

✅ **CORRECT:**
```
~/.claude/skills/ui-design/SKILL.md
---
name: ui-design           <-- Matches folder "ui-design"
description: UI design assistance
---
```

❌ **WRONG (case mismatch):**
```
~/.claude/skills/UI-Design/SKILL.md
---
name: ui-design           <-- Does NOT match folder "UI-Design"
description: UI design assistance
---
```

❌ **WRONG (missing description):**
```
~/.claude/skills/ui-design/SKILL.md
---
name: ui-design
---                       <-- Missing description!
```

### Validation Command (Windows)

Run this PowerShell script to check your skills:
```powershell
$skillsPath = "$env:USERPROFILE\.claude\skills"
if (Test-Path $skillsPath) {
    Get-ChildItem $skillsPath -Directory | ForEach-Object {
        $folder = $_.Name
        $skillFile = Join-Path $_.FullName "SKILL.md"
        if (Test-Path $skillFile) {
            $content = Get-Content $skillFile -Raw
            if ($content -match '(?ms)---\s*name:\s*([^\r\n]+)') {
                $name = $Matches[1].Trim().Trim('"').Trim("'")
                if ($name -ceq $folder) {
                    Write-Host "✅ $folder - OK" -ForegroundColor Green
                } else {
                    Write-Host "❌ $folder - Name mismatch! Folder='$folder' but name='$name'" -ForegroundColor Red
                }
            } else {
                Write-Host "❌ $folder - Missing 'name:' field" -ForegroundColor Red
            }
        } else {
            Write-Host "❌ $folder - Missing SKILL.md" -ForegroundColor Red
        }
    }
} else {
    Write-Host "Skills folder not found: $skillsPath"
}
```

---

## Step 2: Check VS Code Logs for Skill Errors

The VS Code logs will show exactly why skills are being skipped.

### How to Check:
1. **View → Output** (or **Ctrl+Shift+U**)
2. Select **"GitHub Copilot Chat"** from the dropdown
3. Search for `[findAgentSkills]` messages

### What to Look For:
```
[findAgentSkills] Agent skill name "ui-design" does not match folder name "UI-Design"
[findAgentSkills] Failed to validate Agent skill file: ... missing name attribute
[findAgentSkills] Failed to validate Agent skill file: ... missing description
```

---

## Step 3: Verify Skills Settings

### Check `chat.useAgentSkills`:

1. Open Settings: **Ctrl+,** (Windows/Linux) or **Cmd+,** (Mac)
2. Search for `useAgentSkills`
3. Ensure "Chat: Use Agent Skills" is **checked/enabled**

---

## Step 4: Check Skill Folder Locations

### Personal Skills (User Home)

```cmd
dir "%USERPROFILE%\.claude\skills"
dir "%USERPROFILE%\.copilot\skills"
```

### Workspace Skills

```
.github/skills/
.claude/skills/
```

---

## Step 5: Verify Chat Mode is Agent

Skills only appear in **Agent mode**.

1. Look at the chat input area
2. It should show "Agent" (not "Ask" or "Edit")
3. If not, click the mode picker and select "Agent"
```

---

## Step 5: Verify Environment Variables (Windows-Specific)

Skills rely on the `USERPROFILE` environment variable for personal skill paths.

### Check USERPROFILE:

1. Open Command Prompt
2. Run:
   ```cmd
   echo %USERPROFILE%
   ```
3. Should output something like: `C:\Users\YourUsername`

### If USERPROFILE is Wrong or Empty:

1. Open System Properties: **Win+R** → `sysdm.cpl` → Environment Variables
2. Check/Set `USERPROFILE` to your user folder
3. Restart VS Code after changes

### Alternative Check:

In VS Code terminal (PowerShell):
```powershell
[System.Environment]::GetFolderPath('UserProfile')
```

---

## Step 6: Check Output Logs

### View Copilot Chat Logs:

1. **View → Output** (or **Ctrl+Shift+U**)
2. Select "GitHub Copilot Chat" from the dropdown
3. Look for:
   - Errors related to skill loading
   - Messages about `findAgentSkills`
   - Path resolution issues

### Enable Verbose Logging:

1. Open Settings
2. Search for `github.copilot.chat.logLevel`
3. Set to `debug` or `trace`
4. Restart VS Code
5. Reproduce the issue
6. Check logs again

---

## Step 7: Test with a Fresh Workspace

1. Create a new empty folder
2. Create a test skill:
   ```
   mkdir -p .claude/skills/test-skill
   ```
3. Create `SKILL.md`:
   ```markdown
   ---
   name: test-skill
   description: Test skill for debugging
   ---

   # Test Skill
   This is a test skill.
   ```
4. Open VS Code in this folder
5. Trust the workspace
6. Open Copilot Chat in Agent mode
7. Ask Copilot to list available skills

---

## Common Issues and Solutions

| Issue | Solution |
|-------|----------|
| Skills work on Mac but not Windows | Check `USERPROFILE` env var; ensure paths use backslashes correctly |
| Skills work in one workspace but not another | Ensure workspace is trusted; check workspace-specific settings |
| Skills disappeared after update | Check if `chat.useClaudeSkills` was migrated to `chat.useAgentSkills: false` |
| Agent mode shows lock icon | Contact IT admin; enterprise policy may disable Agent mode |
| Skills folder exists but not detected | Verify `SKILL.md` file format; name must match folder name |

---

## Still Having Issues?

If you've followed all steps and skills are still missing:

1. **Collect diagnostics:**
   - VS Code version (`Help → About`)
   - Copilot Chat extension version
   - Output channel logs
   - Settings JSON (redact sensitive data)
   - `echo %USERPROFILE%` output

2. **Report the issue:**
   - File an issue with the collected information
   - Include which troubleshooting steps you tried

---

## Technical Details

For developers investigating the issue, see:
- [INVESTIGATION_REPORT.md](./INVESTIGATION_REPORT.md) - Root cause analysis
- [RECOMMENDED_FIXES.md](./RECOMMENDED_FIXES.md) - Proposed code improvements

---

## Settings Sync Analysis (Jan 20, 2026)

### Configuration Scope Analysis

The skill-related settings are defined in [chat.contribution.ts](../../vscode/src/vs/workbench/contrib/chat/browser/chat.contribution.ts#L735-L770) with the following characteristics:

| Setting | Scope | Sync Behavior | Notes |
|---------|-------|---------------|-------|
| `chat.useAgentSkills` | **WINDOW** (default) | **SYNCED** | No explicit scope set, inherits WINDOW |
| `chat.agentSkillsLocations` | **WINDOW** (default) | **SYNCED** | No explicit scope set, inherits WINDOW |

**Important:** Neither setting has `ignoreSync: true` or `disallowSyncIgnore: true`, meaning both settings **ARE synced** across machines when settings sync is enabled.

### Storage State Analysis

The `PromptsService` stores **disabled prompt files** in storage at [promptsServiceImpl.ts#L653-679](../../vscode/src/vs/workbench/contrib/chat/common/promptSyntax/service/promptsServiceImpl.ts#L653):

```typescript
private readonly disabledPromptsStorageKeyPrefix = 'chat.disabledPromptFiles.';
// Uses StorageScope.PROFILE + StorageTarget.USER
this.storageService.store(
    this.disabledPromptsStorageKeyPrefix + type,
    JSON.stringify(disabled),
    StorageScope.PROFILE,
    StorageTarget.USER  // ← USER target means this IS synced!
);
```

**Critical Finding:** The disabled prompts list uses `StorageTarget.USER`, which **IS synced** via global state sync (see [globalStateSync.ts#L341](../../vscode/src/vs/platform/userDataSync/common/globalStateSync.ts#L341)).

### Workspace Trust Interaction

All skill-related settings have `restricted: true`:
- `chat.useAgentSkills` - **restricted**
- `chat.agentSkillsLocations` - **restricted**

**Restricted settings** require workspace trust. If the workspace is not trusted, these values are **NOT read from workspace settings** — only user settings are used.

**Workspace trust is NOT synced.** It is stored per-workspace/machine.

### Potential Sync-Related Issues

| Issue | Impact | Severity |
|-------|--------|----------|
| **Synced `false` for disabled skills** | If user disabled skills on one machine, `StorageTarget.USER` syncs that to all machines | HIGH |
| **Stale disabled URIs** | Disabled skill URIs reference machine-specific paths (e.g., `~/.claude/skills`); URI may not resolve on other machines | MEDIUM |
| **Workspace trust difference** | Working device trusts workspace, non-working device doesn't → settings are ignored | HIGH |
| **Different VS Code versions** | Skill feature may not exist in older versions; synced settings could be ignored | MEDIUM |
| **Profile mismatch** | StorageScope.PROFILE syncs within same profile; different profiles = different state | MEDIUM |

### Likely Root Cause: Workspace Trust

The `restricted: true` flag combined with workspace trust NOT being synced is the most likely culprit:

1. **Device A (works):** Workspace is trusted → `chat.useAgentSkills=true` is read from workspace/user settings
2. **Device B (broken):** Workspace is NOT trusted → `chat.useAgentSkills` value is ignored, defaults used

To verify: Check "Manage Workspace Trust" on the non-working Windows device.

### Investigation Commands for User

**Check workspace trust (in VS Code):**
- `Ctrl+Shift+P` → "Manage Workspace Trust"

**Check if skills are explicitly disabled in storage (Developer Tools Console):**
```javascript
const storageService = window.workbench.getService('IStorageService');
console.log(storageService.get('chat.disabledPromptFiles.skill', 0, '[]'));
```

**Check actual setting values (may differ from synced):**
```javascript
const configService = window.workbench.getService('IConfigurationService');
console.log('useAgentSkills:', configService.getValue('chat.useAgentSkills'));
console.log('locations:', configService.getValue('chat.agentSkillsLocations'));
```

### Recommended Next Steps

1. **Verify Workspace Trust** on non-working Windows device
2. **Check disabled skills storage** using DevTools command above
3. **Compare VS Code versions** between working/non-working machines
4. **Check if same profile** is used on both machines (Settings Sync uses profiles)

---

## 🔴 CRITICAL FINDING: `chat_preview_features_enabled` Entitlement Check (Jan 20, 2026)

### Root Cause for EMU Accounts

Between **December 17, 2025** and **January 16, 2026**, VS Code gated agent skills behind the `chat_preview_features_enabled` entitlement. **EMU (Enterprise Managed Users) accounts may have this entitlement set to `false` by enterprise policy.**

### Timeline of Changes

| Commit | Date | Description |
|--------|------|-------------|
| `82fb5e14653` | **2025-12-17** | "Agent Skills cleanup (#283934)" — **INTRODUCED the entitlement check** |
| `87b2e355dd5` | **2026-01-16** | "Remove preview feature requirement for Agent Skills (#288477)" — **REMOVED the check** |

### EXACT Code That Broke Skills (Commit 82fb5e14653)

```diff
+import { IDefaultAccountService } from '../../../../../../platform/defaultAccount/common/defaultAccount.js';

 constructor(
     // ... other services
+    @IDefaultAccountService private readonly defaultAccountService: IDefaultAccountService
 ) { ... }

-public async findClaudeSkills(token: CancellationToken): Promise<IClaudeSkill[] | undefined> {
-    const useClaudeSkills = this.configurationService.getValue(PromptsConfig.USE_CLAUDE_SKILLS);
-    if (useClaudeSkills) {
+public async findAgentSkills(token: CancellationToken): Promise<IAgentSkill[] | undefined> {
+    const useAgentSkills = this.configurationService.getValue(PromptsConfig.USE_AGENT_SKILLS);
+    const defaultAccount = await this.defaultAccountService.getDefaultAccount();
+    const previewFeaturesEnabled = defaultAccount?.chat_preview_features_enabled ?? true;
+    if (useAgentSkills && previewFeaturesEnabled) {
```

**The breaking change:** Added `&& previewFeaturesEnabled` condition, gating skills behind the entitlement.

### EXACT Code That Fixed It (Commit 87b2e355dd5)

```diff
-import { IDefaultAccountService } from '../../../../../../platform/defaultAccount/common/defaultAccount.js';

 constructor(
     // ... other services
-    @IDefaultAccountService private readonly defaultAccountService: IDefaultAccountService,
 ) { ... }

 public async findAgentSkills(token: CancellationToken): Promise<IAgentSkill[] | undefined> {
     const useAgentSkills = this.configurationService.getValue(PromptsConfig.USE_AGENT_SKILLS);
-    const defaultAccount = await this.defaultAccountService.getDefaultAccount();
-    const previewFeaturesEnabled = defaultAccount?.chat_preview_features_enabled ?? true;
-    if (useAgentSkills && previewFeaturesEnabled) {
+    if (useAgentSkills) {
```

### How `chat_preview_features_enabled` Is Determined

From [defaultAccount.ts#L461](../../vscode/src/vs/workbench/services/accounts/common/defaultAccount.ts#L461):

```typescript
const tokenMap = this.extractFromToken(chatData.token);
return {
    // Editor preview features are disabled if the flag is present and set to 0
    chat_preview_features_enabled: tokenMap.get('editor_preview_features') !== '0',
    // ...
};
```

**Mechanism:**
1. VS Code fetches token entitlements from GitHub's API
2. The token contains an `editor_preview_features` claim
3. If this claim equals `'0'`, `chat_preview_features_enabled` becomes `false`
4. **EMU accounts may have this set to `'0'` by enterprise policy**

### Version Impact Matrix

| User's VS Code Version | Status | Notes |
|------------------------|--------|-------|
| VS Code Insiders **before Dec 17, 2025** | ✅ WORKS | No entitlement check |
| VS Code Insiders **Dec 17, 2025 to Jan 15, 2026** | ❌ BROKEN for EMU | Entitlement check added |
| VS Code Insiders **after Jan 16, 2026** | ✅ WORKS | Entitlement check removed |
| VS Code Stable **1.96.x** (current) | ❌ LIKELY BROKEN for EMU | Fix not yet in stable |
| VS Code Stable **1.97+** (future) | ✅ SHOULD WORK | When fix ships |

### Current State (Jan 20, 2026)

- The fix (`87b2e355dd5`) is **only in `main` branch** (Insiders)
- **Not yet tagged for stable release**
- **Users on VS Code Stable are still affected**
- Users on **VS Code Insiders after Jan 16** should be fixed

### How to Verify This Is the Cause

**In VS Code Developer Tools Console:**
```javascript
// Check the entitlement value
const accountService = window.workbench.getService('IDefaultAccountService');
const account = await accountService.getDefaultAccount();
console.log('preview features enabled:', account?.chat_preview_features_enabled);
console.log('full policy data:', account?.policyData);
```

If `chat_preview_features_enabled` is `false`, this is the root cause.

### Immediate Workaround

**There is NO workaround** — the check was in VS Code core, not the Copilot extension.

**Options:**
1. **Use VS Code Insiders** (builds after Jan 16, 2026)
2. **Wait for VS Code Stable 1.97+** (when fix ships)
3. **Contact enterprise admin** to enable `editor_preview_features` entitlement (if possible)

### Correlation with User's Timeline

| Event | Date | Notes |
|-------|------|-------|
| User reports skills worked | **Mid-December 2025** | Before breaking commit |
| Breaking commit merged | **December 17, 2025** | `82fb5e14653` |
| User reports skills stopped working | **Late December 2025** | After breaking commit shipped |
| Fix merged | **January 16, 2026** | `87b2e355dd5` |
| Fix in stable | **TBD** | Awaiting release |

**This timeline matches perfectly.** Skills worked before Dec 17, broke after Dec 17, and will be fixed when the Jan 16 commit ships to stable.
