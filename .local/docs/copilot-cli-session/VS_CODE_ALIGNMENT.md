# VS Code Alignment & Maintenance Workflow

> Last updated: 2026-04-21
> Constraint: **ARC-10** (Hybrid Alignment) — see [08-constraints-and-requirements.md](./08-constraints-and-requirements.md)

---

## Table of Contents

1. [Purpose](#1-purpose)
2. [Source Locations](#2-source-locations)
3. [Component Mapping](#3-component-mapping)
4. [Monitoring Workflow](#4-monitoring-workflow)
5. [Update Procedure](#5-update-procedure)
6. [SDK Update Procedure](#6-sdk-update-procedure)
7. [Webapp Codebase Update Workflow](#7-webapp-codebase-update-workflow)
8. [Documentation Sync Checklist](#8-documentation-sync-checklist)
9. [Automated Drift Detection](#9-automated-drift-detection)
10. [Version Compatibility Matrix](#10-version-compatibility-matrix)
11. [Decision Log](#11-decision-log)

---

## 1. Purpose

This document serves two roles:

1. **Component mapping** — maps webapp components to VS Code source counterparts for hybrid alignment (ARC-10)
2. **Maintenance workflow** — defines how to detect, evaluate, and propagate upstream changes from VS Code and the Copilot SDK into the webapp and its documentation

Without a systematic workflow, the docs and webapp will drift from VS Code's implementation, causing bugs, broken references, and misleading guidance.

---

## 2. Source Locations

### VS Code (Copilot Extension)

```
Repository: github/vscode (or your fork)
Branch:     main
Root path:  extensions/copilot/src/extension/chatSessions/

Key directories:
  copilotcli/                    # CLI session integration
  copilotcli/node/               # Node-side services (SessionService, LocalSessionManager)
  copilotcli/browser/            # Browser-side stubs
  common/                        # Shared interfaces, enums, data models

Key files to watch:
  copilotcli/node/copilotcliSessionService.ts   # Core service (LocalSessionManager instantiation)
  common/agentSessionModel.ts                   # ISession, ISessionState, SessionStatus
  common/agentSessionsModel.ts                  # AgentSessionsModel (grouping, sorting)
  common/sessionDatabase.ts                     # SQLite schema (file edits)
  common/sessionPersistence.ts                  # Lock file, working directory resolution
  common/agentSessionMetadata.ts                # IAgentSessionMetadata
```

### Copilot SDK (Public)

```
Repository: github/copilot-sdk
Branch:     main
Root path:  nodejs/src/

Key files to watch:
  index.ts                   # Public exports
  client.ts                  # CopilotClient class
  session.ts                 # CopilotSession class
  types.ts                   # All type definitions
  generated/rpc.ts           # RPC method definitions (auto-generated)
  package.json               # Version, dependencies
```

### Copilot CLI

```
Package: @github/copilot (npm)
Changelog: https://github.com/github/copilot-sdk/blob/main/references/cli-CHANGELOG.md
```

---

## 3. Component Mapping

Maps each webapp component to its VS Code source counterpart. Alignment levels:

- **Pixel** — visually identical (CSS, layout, spacing)
- **Behavioral** — same interaction model (events, state transitions, gestures)
- **Structural** — same data model/schema (types, interfaces, enums)
- **Conceptual** — same idea, different implementation (adapted for mobile/web)

| Webapp Component | VS Code Source | Alignment | Doc Reference |
|---|---|---|---|
| Session list (sidebar) | `agentSessionsModel.ts` → `AgentSessionsModel` | Structural | Doc 04 §2 |
| Session grouping (Today/Yesterday/Week/Older) | `agentSessionsModel.ts` → date-based grouping | Behavioral | Doc 01 §2.3 |
| Chat message rendering | `chatWidget.ts` → `ChatWidget` | Conceptual | Doc 07 §4 |
| Code block rendering | CodeMirror 6 (webapp) vs Monaco (VS Code) | Conceptual | Doc 06 §4.2 |
| File diff display | CodeMirror merge view vs Monaco diff editor | Conceptual | Doc 07 §4.11 |
| Session status indicators | `SessionStatus` enum + bitset encoding | Structural | Doc 02 §1 |
| Tool approval dialog | `chatConfirmationWidget.ts` | Behavioral | Doc 07 §4.8 |
| Markdown rendering | `marked` + `DOMPurify` (same libraries) | Pixel | Doc 06 §4.1 |
| Agent mode selector | Session mode RPC (`session.mode.set`) | Behavioral | Doc 06 §3.2 |
| Session persistence | `events.jsonl` + `session-state/` directory | Structural | Doc 02 §3 |
| File edit tracking | `session.db` SQLite schema | Structural | Doc 02 §2 |
| State protocol | Agent Host State Protocol (JSON-RPC) | Structural | Doc 03 §4 |
| *More rows added during implementation* | | | |

---

## 4. Monitoring Workflow

### 4.1 Frequency

| Check | Frequency | Trigger |
|-------|-----------|---------|
| VS Code upstream commits | Weekly | Manual or CI cron |
| SDK npm releases | On publish | npm audit / GitHub watch |
| CLI npm releases | On publish | npm audit / GitHub watch |
| Documentation review | Monthly | Calendar reminder |

### 4.2 Git-Based Change Detection

Monitor the VS Code repo for changes to the Copilot extension:

```bash
#!/usr/bin/env bash
# check-upstream.sh — Run weekly to detect VS Code Copilot changes

VSCODE_REPO="/path/to/vscode"
WATCH_PATHS=(
  "extensions/copilot/src/extension/chatSessions/"
  "extensions/copilot/package.json"
)

cd "$VSCODE_REPO" || exit 1
git fetch origin main --quiet

# Show commits touching watched paths since last check
LAST_CHECK=$(cat .last-upstream-check 2>/dev/null || echo "1 week ago")

for watch_path in "${WATCH_PATHS[@]}"; do
  echo "=== Changes in: $watch_path ==="
  git --no-pager log --oneline --since="$LAST_CHECK" origin/main -- "$watch_path"
done

date -Iseconds > .last-upstream-check
```

### 4.3 npm Release Monitoring

```bash
#!/usr/bin/env bash
# check-sdk-releases.sh — Check for new SDK/CLI releases

EXPECTED_SDK="0.2.2"
EXPECTED_CLI="1.0.24"

LATEST_SDK=$(npm view @github/copilot-sdk version 2>/dev/null)
LATEST_CLI=$(npm view @github/copilot version 2>/dev/null)

if [ "$LATEST_SDK" != "$EXPECTED_SDK" ]; then
  echo "⚠️  SDK update: $EXPECTED_SDK → $LATEST_SDK"
  echo "   Review: https://github.com/github/copilot-sdk/blob/main/CHANGELOG.md"
fi

if [ "$LATEST_CLI" != "$EXPECTED_CLI" ]; then
  echo "⚠️  CLI update: $EXPECTED_CLI → $LATEST_CLI"
  echo "   Review: https://github.com/github/copilot-sdk/blob/main/references/cli-CHANGELOG.md"
fi
```

### 4.4 GitHub Watch Notifications

Set up GitHub watch on these repos:
- `github/vscode` — Releases only (or custom: `extensions/copilot/` path filter via Actions)
- `github/copilot-sdk` — Releases + Issues
- `@github/copilot-sdk` on npm — use `npm-check-updates` or Dependabot

---

## 5. Update Procedure

When an upstream change is detected, follow this decision tree:

```
Upstream change detected
  │
  ├── Does it affect a mapped component? (§3 table)
  │     ├── NO → Log in Decision Log (§10), no action needed
  │     └── YES ↓
  │
  ├── Is it a breaking change?
  │     ├── YES → PRIORITY: Update webapp code + docs immediately
  │     │         1. Read the upstream diff
  │     │         2. Update the webapp implementation
  │     │         3. Update affected docs (see §7 checklist)
  │     │         4. Update version matrix (§9)
  │     │         5. Run full test suite
  │     │         6. Log in Decision Log (§10)
  │     │
  │     └── NO → NORMAL: Schedule for next maintenance window
  │               1. Evaluate impact scope
  │               2. Update docs if terminology/types changed
  │               3. Update webapp if behavior changed
  │               4. Log in Decision Log (§10)
  │
  └── Is it a new feature?
        ├── In scope for webapp? → Plan implementation, update docs
        └── Out of scope? → Log in Decision Log, add to SCP constraints if needed
```

### 5.1 Step-by-Step: Evaluating a VS Code Commit

```bash
# 1. Read the commit
cd /path/to/vscode
git --no-pager show <commit-sha> --stat
git --no-pager show <commit-sha> -- extensions/copilot/src/extension/chatSessions/

# 2. Identify affected interfaces/types
# Look for changes to: interface definitions, enum values, method signatures,
# constructor parameters, event names, RPC endpoints

# 3. Cross-reference with docs
# Search for the changed symbol in our docs:
grep -rn "ChangedSymbolName" /path/to/docs/copilot-cli-session/

# 4. Assess impact
# - Type rename? → Update all doc references
# - New field? → Add to data model doc (02), implementation guide (05)
# - Removed field? → Mark as deprecated or remove from docs
# - Behavioral change? → Update protocol doc (03), implementation guide (05)

# 5. If webapp code exists, update it too
```

### 5.2 Step-by-Step: Evaluating an SDK Release

```bash
# 1. Read the changelog
# Check: https://github.com/github/copilot-sdk/blob/main/CHANGELOG.md

# 2. Check for breaking changes
cd /path/to/copilot-sdk
git --no-pager diff v${OLD_VERSION}..v${NEW_VERSION} -- nodejs/src/types.ts
git --no-pager diff v${OLD_VERSION}..v${NEW_VERSION} -- nodejs/src/client.ts
git --no-pager diff v${OLD_VERSION}..v${NEW_VERSION} -- nodejs/src/session.ts

# 3. Cross-reference with webapp docs (06-09)
# SDK symbols are used in docs 06-09 only (docs 01-05 use VS Code internals)
grep -rn "CopilotClient\|CopilotSession\|defineTool\|SessionConfig" \
  /path/to/docs/copilot-cli-session/0[6-9]*.md

# 4. Update docs and code as needed
```

---

## 6. SDK Update Procedure

The `@github/copilot-sdk` is in **public preview** (v0.2.x). Breaking changes are expected. This section covers how to safely upgrade.

### 6.1 Pre-Upgrade Checklist

- [ ] Read the full changelog for all versions between current and target
- [ ] Check protocol version compatibility (`sdk-protocol-version.json`)
- [ ] Check minimum CLI version requirement
- [ ] Search for deprecated APIs in the changelog
- [ ] Back up current `package-lock.json`

### 6.2 Upgrade Steps

```bash
# 1. Install new version
npm install @github/copilot-sdk@latest

# 2. Check for TypeScript compilation errors
npx tsc --noEmit

# 3. Run test suite
npm test

# 4. If errors, consult the changelog for migration guidance
# Common changes (based on v0.2.x history):
#   - Type renames (e.g., Session → CopilotSession)
#   - Module reorganization (e.g., copilot.types removed)
#   - Callback signature changes (e.g., onElicitationRequest)
#   - Package rename (e.g., rpc → generated/rpc)

# 5. Update docs
# - 06-webapp-extraction-guide.md: SDK integration section (§3)
# - 08-constraints-and-requirements.md: SDK version in constraints
# - This file: Version matrix (§9)
```

### 6.3 Post-Upgrade Verification

After upgrading, verify these critical paths:

1. **Client lifecycle:** `new CopilotClient()` → `start()` → `createSession()` → `send()` → `disconnect()` → `stop()`
2. **Session resume:** `resumeSession(id)` loads previous conversation
3. **Event streaming:** `session.on()` receives delta events with correct property names
4. **Tool definitions:** `defineTool()` with Zod schemas registers successfully
5. **RPC methods:** `session.rpc.mode.set()`, `session.rpc.skills.list()` respond correctly
6. **Permission handling:** `onPermissionRequest` callback fires for tool approvals

---

## 7. Webapp Codebase Update Workflow

When an upstream change (VS Code or SDK) requires webapp code changes, follow this structured workflow.

### 7.1 Impact Assessment

Before changing code, classify the upstream change:

| Change Type | Webapp Impact | Example |
|-------------|---------------|---------|
| **SDK type rename** | Update imports, type annotations | `Session` → `CopilotSession` |
| **SDK method signature change** | Update call sites, possibly refactor | New required parameter in `send()` |
| **SDK method removed** | Replace with new equivalent | `dispose()` → `disconnect()` |
| **New SDK event type** | Add WS relay + React handler + UI component | New `assistant.plan_delta` event |
| **Event field rename** | Update parsers, WS message handlers, React state | `data.content` → `data.deltaContent` |
| **New VS Code UI pattern** | Evaluate for adoption, implement if in scope | New inline diff viewer |
| **VS Code rendering change** | Update CSS/components for alignment | Spacing/color change in message bubbles |
| **Protocol change** | Update server WS protocol + client handlers | New handshake field |
| **New tool type** | Add permission UI + result renderer | New `apply_patch` tool |

### 7.2 Webapp Update Steps

```
1. CREATE A BRANCH
   git checkout -b update/<component>-<version>

2. UPDATE DEPENDENCIES
   npm install @github/copilot-sdk@<new-version>
   npx tsc --noEmit   # catch type errors immediately

3. FIX TYPE ERRORS (server-side first)
   ├── src/server/sdk-provider.ts     # SDK client wrapper
   ├── src/server/session-manager.ts  # Session lifecycle
   ├── src/server/ws-relay.ts         # Event → WebSocket bridge
   └── src/server/routes.ts           # REST API handlers

4. FIX TYPE ERRORS (client-side)
   ├── src/client/stores/session.ts   # Zustand state
   ├── src/client/hooks/useSession.ts # React hooks
   ├── src/client/components/         # UI components
   └── src/client/types/events.ts     # Shared event types

5. UPDATE SHARED TYPES
   └── src/shared/types.ts            # Types shared between server & client

6. RUN TESTS
   npm test                    # Unit tests
   npm run test:e2e            # End-to-end tests (if available)
   npm run typecheck           # Full type check

7. MANUAL VERIFICATION
   ├── Start a new session → send message → verify streaming
   ├── Resume an existing session → verify history loads
   ├── Test tool approval flow → verify permission dialog
   ├── Test on mobile viewport (440×956) → verify responsive layout
   └── Test agent mode switching → verify mode RPC

8. UPDATE DOCUMENTATION
   └── Follow §8 Documentation Sync Checklist

9. UPDATE THIS FILE
   ├── Version Matrix (§10) — bump versions
   └── Decision Log (§11) — record the change

10. MERGE
    git add -A && git commit -m "chore: update to copilot-sdk vX.Y.Z"
```

### 7.3 Common SDK Migration Patterns

Recurring patterns from SDK v0.2.x history. Use as a quick-reference when upgrading:

**Pattern: Type rename**
```typescript
// Before
import { Session } from "@github/copilot-sdk";
const s: Session = await client.createSession(config);

// After
import { CopilotSession } from "@github/copilot-sdk";
const s: CopilotSession = await client.createSession(config);

// Quick fix: find-and-replace with word boundaries
// rg -l '\bSession\b' src/ | xargs sed -i 's/\bSession\b/CopilotSession/g'
```

**Pattern: Method rename/removal**
```typescript
// Before
await session.dispose();

// After
await session.disconnect(); // preserves data
// OR
await session.abort();      // cancels in-flight work, then disconnects
```

**Pattern: Event field rename**
```typescript
// Before (server relay)
session.on((event) => {
  if (event.type === "assistant.message_delta") {
    ws.send(JSON.stringify({ delta: event.data.content }));
  }
});

// After
session.on((event) => {
  if (event.type === "assistant.message_delta") {
    ws.send(JSON.stringify({ delta: event.data.deltaContent }));
  }
});
```

**Pattern: Callback signature change**
```typescript
// Before — separate method call
session.respondToPermission(requestId, { approved: true });

// After — deferred promise in config
const session = await client.createSession({
  onPermissionRequest: async (request) => {
    const approved = await showPermissionDialog(request);
    return { approved };
  },
});
```

### 7.4 Regression Testing After Update

Run these checks after every SDK/CLI update:

```bash
# 1. Type safety
npx tsc --noEmit --strict

# 2. Unit tests
npm test -- --coverage

# 3. SDK integration smoke test
node -e "
  const { CopilotClient } = require('@github/copilot-sdk');
  const c = new CopilotClient();
  c.start().then(() => {
    console.log('✅ Client starts');
    return c.stop();
  }).then(() => {
    console.log('✅ Client stops');
  }).catch(e => {
    console.error('❌ SDK smoke test failed:', e.message);
    process.exit(1);
  });
"

# 4. Event streaming verification
# Start a session, send a prompt, verify delta events arrive via WebSocket

# 5. Mobile viewport check
# Open http://localhost:3000 in Chrome DevTools with iPhone 16 Pro Max preset
```

### 7.5 Rollback Procedure

If an SDK update causes issues that can't be quickly resolved:

```bash
# 1. Revert to known-good version
npm install @github/copilot-sdk@<previous-version>

# 2. Verify rollback works
npx tsc --noEmit && npm test

# 3. Log the issue
# Add to Decision Log (§11) with details of what broke and why

# 4. Create a tracking issue for the upgrade
```

---

## 8. Documentation Sync Checklist

When an upstream change requires doc updates, use this checklist to ensure all affected docs are updated consistently.

### 7.1 Type/Interface Change

- [ ] `02-data-model.md` — Update interface definition
- [ ] `03-protocol.md` — Update if it affects wire protocol
- [ ] `05-implementation-guide.md` — Update pseudocode examples
- [ ] `06-webapp-extraction-guide.md` — Update SDK integration code
- [ ] `07-ui-specification.md` — Update if it affects UI state/rendering
- [ ] `08-constraints-and-requirements.md` — Update if constraint affected
- [ ] This file — Update Component Mapping table (§3)

### 7.2 New Event Type

- [ ] `03-protocol.md` — Add to event catalog
- [ ] `05-implementation-guide.md` — Add handling example
- [ ] `06-webapp-extraction-guide.md` — Add WebSocket relay
- [ ] `07-ui-specification.md` — Add UI component if visual

### 7.3 SDK API Change

- [ ] `06-webapp-extraction-guide.md` — Update all SDK code samples
- [ ] `08-constraints-and-requirements.md` — Update SDK version constraint
- [ ] This file — Update Version Matrix (§9)

### 7.4 UI/UX Change

- [ ] `04-ui-integration.md` — Update command/menu definitions
- [ ] `07-ui-specification.md` — Update component specs, design tokens
- [ ] This file — Update Component Mapping alignment level

### 7.5 Architecture Change

- [ ] `01-architecture.md` — Update layer diagram
- [ ] `03-protocol.md` — Update communication channels
- [ ] `05-implementation-guide.md` — Update affected steps
- [ ] `06-webapp-extraction-guide.md` — Update if webapp architecture affected

---

## 9. Automated Drift Detection

### 9.1 Symbol Extraction Script

Extract all VS Code symbols referenced in docs and verify they still exist:

```bash
#!/usr/bin/env bash
# verify-symbols.sh — Check that documented symbols exist in source

DOCS_DIR="/path/to/docs/copilot-cli-session"
VSCODE_SRC="/path/to/vscode/extensions/copilot/src"
SDK_SRC="/path/to/copilot-sdk/nodejs/src"

# Extract class/interface names from docs
SYMBOLS=$(grep -rohP '`(I[A-Z]\w+|[A-Z][a-z]\w+(?:Model|Service|Provider|Widget|Control))`' \
  "$DOCS_DIR"/*.md | sort -u | tr -d '`')

echo "Checking ${#SYMBOLS[@]} symbols..."

for sym in $SYMBOLS; do
  if ! grep -rq "$sym" "$VSCODE_SRC" "$SDK_SRC" 2>/dev/null; then
    echo "❌ NOT FOUND: $sym"
  fi
done

echo "Done."
```

### 9.2 CI Integration (GitHub Actions)

```yaml
# .github/workflows/doc-drift.yml
name: Documentation Drift Check
on:
  schedule:
    - cron: '0 9 * * 1'  # Weekly on Monday
  workflow_dispatch:

jobs:
  check-drift:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Check SDK version
        run: |
          EXPECTED="0.2.2"
          LATEST=$(npm view @github/copilot-sdk version)
          if [ "$LATEST" != "$EXPECTED" ]; then
            echo "::warning::SDK updated: $EXPECTED → $LATEST"
          fi
      - name: Check CLI version
        run: |
          EXPECTED="1.0.24"
          LATEST=$(npm view @github/copilot version)
          if [ "$LATEST" != "$EXPECTED" ]; then
            echo "::warning::CLI updated: $EXPECTED → $LATEST"
          fi
```

---

## 10. Version Compatibility Matrix

Current known-good versions. Update this table when upgrading any component.

| Component | Version | Min Compatible | Last Verified | Notes |
|-----------|---------|----------------|---------------|-------|
| `@github/copilot-sdk` | 0.2.2 | 0.2.0 | 2026-04-21 | Public preview; breaking changes expected |
| `@github/copilot` (CLI) | 1.0.24 | 1.0.21 | 2026-04-21 | Transitive dep of copilot-sdk |
| SDK protocol | v3 | v2 | 2026-04-21 | Declared in `sdk-protocol-version.json` |
| Node.js | 24.x LTS | 22.x | 2026-04-21 | SDK requires Node 18+; webapp requires Node 24+ (see Doc 06 §10) |
| VS Code | 1.100+ | — | 2026-04-21 | Source reference only (not a runtime dep) |
| Hono | 4.12.12 | 4.0.0 | 2026-04-21 | Pinned version at last audit; doc 06 uses semver range `^4.12.0` |
| React | 19.2.5 | 19.0.0 | 2026-04-21 | |
| TypeScript | 6.0.2 | 5.5.0 | 2026-04-21 | |

---

## 11. Decision Log

Record all upstream changes evaluated and the decision made.

| Date | Upstream Change | Impact | Decision | Docs Updated |
|------|----------------|--------|----------|--------------|
| 2026-04-21 | Initial documentation created | — | Baseline | All |
| *Add rows as changes are evaluated* | | | | |

---

## Quick Reference: What Changed? Where to Update?

```
VS Code type renamed?
  → docs 01, 02, 03, 04, 05 + this file §3

SDK method renamed/removed?
  → docs 06, 08 + webapp code (§7) + this file §10

New CLI version with new events?
  → docs 03, 05, 06 + webapp code (§7) + this file §10

UI component added/changed in VS Code?
  → docs 04, 07 + webapp code (§7) + this file §3

New constraint needed?
  → doc 08 + this file §3

Webapp code needs updating?
  → Follow §7 workflow → update docs per §8 checklist → update §10 matrix
```
