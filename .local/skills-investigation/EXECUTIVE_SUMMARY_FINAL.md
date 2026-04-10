# Skills Investigation - Executive Summary (FINAL)

**Date:** January 20, 2026
**Status:** Multiple Potential Root Causes Identified via Code Analysis

---

## Problem Statement

Users on Windows report that the `<skills>` section is missing from the system prompt. Skills were working in mid-December 2025 and stopped working after.

**Environment confirmed by user:**
- VS Code **Insiders** (post Jan 16) ✅
- Chat mode IS Agent ✅
- Workspace IS trusted ✅
- Agent mode IS enabled by organization ✅
- Skills installed in both personal (`~/.claude/skills`) and workspace (`.claude/skills`) ✅
- Skill `skill-creator` has correct `name: skill-creator` matching folder ✅
- Skill has `description:` field ✅

---

## 🔴 ROOT CAUSES IDENTIFIED (Ranked by Evidence)

### 1. `disallowConfigurationDefault: true` on Settings (HIGH CONFIDENCE)

The setting `chat.useAgentSkills` is configured with:
```typescript
{
    default: true,
    disallowConfigurationDefault: true,  // ← PROBLEM
}
```

**Impact:** The default `true` value CANNOT be applied unless product.json explicitly enables it. On some Windows builds, `chat.useAgentSkills` may evaluate to `undefined`, causing `findAgentSkills()` to return immediately without searching.

**Fix:** User should explicitly set `"chat.useAgentSkills": true` in settings.

---

### 2. Remote Workspace Context (HIGH CONFIDENCE)

If the user has **WSL, SSH, or Dev Container** connected:
- `pathService.userHome()` resolves to the **REMOTE** machine's home directory
- Skills in `C:\Users\username\.claude\skills` are invisible
- The code searches `~` on the remote, not Windows host

**Verification:** Check bottom-left corner of VS Code for "WSL", "SSH:", or "Dev Container" badge.

---

### 3. Silent Folder Non-Existence (MEDIUM CONFIDENCE)

The search service silently filters out non-existent folders:
```typescript
const exists = await Promise.all(query.folderQueries.map(q => this.fileService.exists(q.folder)));
query.folderQueries = query.folderQueries.filter((_, i) => exists[i]);
```

**If `~/.claude/skills` doesn't exist on Windows, zero results returned with NO logging.**

**Verification:**
```powershell
Test-Path "$env:USERPROFILE\.claude\skills"
```

---

## ❌ RULED OUT

| Hypothesis | Reason |
|------------|--------|
| Preview features gate | Removed in Jan 16 commit, user confirmed Insiders |
| Search can't work outside workspace | ISearchService uses ripgrep, works on any path |
| Caching issue | `findAgentSkills()` doesn't use cache |
| Skill file format wrong | User confirmed correct format |
| Case-sensitivity mismatch | User confirmed `skill-creator` matches exactly |

---

## Diagnostic Steps for Affected User

### Step 1: Check for Remote Workspace
Look at VS Code bottom-left corner. If it shows "WSL", "SSH:", or "Dev Container", that's the issue.

### Step 2: Verify Folder Exists
```powershell
Test-Path "$env:USERPROFILE\.claude\skills\skill-creator\SKILL.md"
```

### Step 3: Explicitly Enable Setting
Add to settings.json:
```json
{
    "chat.useAgentSkills": true
}
```

### Step 4: Check Output Logs
View → Output → "Window" → Search for `[findAgentSkills]`

---

## Technical Details

### Code Path for Skills

```
chatWidget.ts
  → enabledTools only if Agent mode
    → computeAutomaticInstructions.ts
      → _getTool('readFile') must return valid tool
        → promptsService.findAgentSkills()
          → GATE: chat.useAgentSkills must be true
          → fileLocator.findAgentSkills()
            → pathService.userHome() - COULD BE REMOTE!
            → searchService.fileSearch() on each location
              → GATE: folder must exist
            → Parse each SKILL.md
              → GATE: name must exist
              → GATE: name must match folder
```

### Where Logging Exists

| Location | Log Level | Message Pattern |
|----------|-----------|-----------------|
| promptsServiceImpl.ts | `error` | `[findAgentSkills] Agent skill name "X" does not match folder name "Y"` |
| promptsServiceImpl.ts | `warn` | `[findAgentSkills] Skipping duplicate agent skill name` |
| promptFilesLocator.ts | `trace` | `[PromptFilesLocator] Error searching for skills` |

### Where Logging is MISSING (Silent Failures)

- When `chat.useAgentSkills` is `undefined` or `false`
- When folder doesn't exist (filtered out silently)
- When `userHome` points to remote instead of local

---

## Recommended Code Improvements

### Add Diagnostic Logging

```typescript
// promptsServiceImpl.ts - findAgentSkills()
public async findAgentSkills(token: CancellationToken): Promise<IAgentSkill[] | undefined> {
    const useAgentSkills = this.configurationService.getValue(PromptsConfig.USE_AGENT_SKILLS);
    this.logger.info(`[findAgentSkills] useAgentSkills = ${useAgentSkills}`);

    if (!useAgentSkills) {
        this.logger.info(`[findAgentSkills] Skills disabled via setting`);
        return undefined;
    }

    // ... existing code
}
```

```typescript
// promptFilesLocator.ts - findAgentSkillsInFolder()
private async findAgentSkillsInFolder(uri: URI, token: CancellationToken): Promise<URI[]> {
    const exists = await this.fileService.exists(uri);
    if (!exists) {
        this.logService.info(`[PromptFilesLocator] Skill folder does not exist: ${uri.toString()}`);
        return [];
    }
    // ... existing code
}
```
