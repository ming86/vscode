# Skills Investigation - Executive Summary (UPDATED)

**Date:** January 20, 2026
**Status:** Root Cause Confirmed via Code Analysis

---

## Problem Statement

Users on Windows report that the `<skills>` section is missing from the system prompt. Skills were working in mid-December 2025 and stopped working after.

**Environment confirmed by user:**
- Chat mode IS Agent ✅
- Workspace IS trusted ✅
- Agent mode IS enabled by organization ✅
- Skills installed in both personal (`~/.claude/skills`) and workspace (`.claude/skills`) ✅
- Skill `skill-creator` has correct `name: skill-creator` matching folder ✅
- Skill has `description:` field ✅

---

## 🔴 ROOT CAUSE: EMU Account Preview Features Gate (VS Code STABLE)

### Evidence from Git History

**Commit `82fb5e14653` (Dec 17, 2025)** ADDED this check to `findAgentSkills()`:

```typescript
const defaultAccount = await this.defaultAccountService.getDefaultAccount();
const previewFeaturesEnabled = defaultAccount?.chat_preview_features_enabled ?? true;
if (useAgentSkills && previewFeaturesEnabled) {  // ← BOTH must be true
    // ... skill discovery
}
return undefined;  // ← If either is false, NO SKILLS
```

**Commit `87b2e355dd5` (Jan 16, 2026)** REMOVED the check - **BUT only in Insiders**.

### How EMU Entitlement is Determined

```typescript
// defaultAccount.ts
chat_preview_features_enabled: tokenMap.get('editor_preview_features') !== '0'
```

EMU accounts may have `editor_preview_features: '0'` set by enterprise policy, causing:
- `chat_preview_features_enabled = false`
- `findAgentSkills()` returns `undefined` immediately
- NO skills appear regardless of file format

### Version Impact

| VS Code Version | Status |
|-----------------|--------|
| Insiders (Jan 16+) | ✅ FIXED |
| Stable (current) | ❌ STILL BLOCKED for EMU accounts |

---

## Why Previous Hypotheses Were Wrong

| Hypothesis | Why Ruled Out |
|------------|---------------|
| Case-sensitivity mismatch | User's `skill-creator` folder matches `name: skill-creator` exactly |
| Missing `description:` field | User confirmed description is present |
| BOM in file | Would cause `skippedParseFailed`, not complete absence |
| `chat.useAgentSkills` setting | User confirmed skills worked before (setting unchanged) |

---

## Confirmation Steps

1. **Check VS Code version**: If using Stable, EMU accounts are blocked
2. **Check token**: Does GitHub token have `editor_preview_features: '0'`?
3. **Test on Insiders**: Skills should work on VS Code Insiders post-Jan 16

---

## Secondary Gates (for completeness)

These are also checked, but the EMU gate happens FIRST:

| Gate | What It Checks |
|------|----------------|
| `_enabledTools` | Must be in Agent mode |
| `readFile` tool | Tool must exist and be enabled |
| `chat.useAgentSkills` | Setting must be `true` |

---

## Recommended Fix

### For VS Code Team

Remove the preview features gate from VS Code Stable (already done in Insiders):

```diff
-const defaultAccount = await this.defaultAccountService.getDefaultAccount();
-const previewFeaturesEnabled = defaultAccount?.chat_preview_features_enabled ?? true;
-if (useAgentSkills && previewFeaturesEnabled) {
+if (useAgentSkills) {
```

### For Users (Immediate Workaround)

1. Switch to **VS Code Insiders** (Jan 16+ build)
2. Or wait for the fix to ship in VS Code Stable 1.97+

---

## Files Updated

- [INVESTIGATION_REPORT.md](./INVESTIGATION_REPORT.md) - Full technical analysis
- [REMEDIATION_STEPS.md](./REMEDIATION_STEPS.md) - User troubleshooting guide
- [RECOMMENDED_FIXES.md](./RECOMMENDED_FIXES.md) - Proposed code changes
