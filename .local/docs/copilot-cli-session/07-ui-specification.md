# UI Specification

Visual and interaction blueprint for the Copilot CLI Session webapp. A frontend developer should be able to build the entire UI from this document alone.

**Scope:** Copilot CLI session management — connect, resume, create, view sessions. No cloud agents, no GitHub remote copilot.

**Primary target:** iPhone 16 Pro Max (440×956 logical pixels, 3× retina). Desktop secondary.

**Stack:** React 19 · Vite · Tailwind CSS v4 · CodeMirror 6 · Radix Primitives · Vaul · shadcn/ui AI patterns · Shiki

---

## 1. Design System Foundation

### 1.1 Color Token Map

All values sourced from VS Code's `chatColors.ts`, the Dark Modern and Light Modern default themes, and the base color registry. CSS custom properties use the `--color-` prefix for Tailwind v4 `@theme` integration.

#### Chat-Specific Tokens

| VS Code Token | CSS Custom Property | Dark Modern | Light Modern | Purpose |
|---|---|---|---|---|
| `chat.requestBorder` | `--color-chat-request-border` | `rgba(255,255,255,0.10)` | `rgba(0,0,0,0.10)` | Message separator, card borders |
| `chat.requestBackground` | `--color-chat-request-bg` | `rgba(30,30,30,0.62)` | `rgba(255,255,255,0.62)` | Request message background (62% editor bg) |
| `chat.requestBubbleBackground` | `--color-chat-request-bubble-bg` | `rgba(38,79,120,0.30)` | `rgba(173,214,255,0.30)` | Request bubble (30% selection) |
| `chat.requestBubbleHoverBackground` | `--color-chat-request-bubble-hover` | `rgba(38,79,120,0.60)` | `rgba(173,214,255,0.60)` | Request bubble hover |
| `chat.requestCodeBorder` | `--color-chat-request-code-border` | `#004972B8` | `#0e639c40` | Code block border in requests |
| `chat.avatarBackground` | `--color-chat-avatar-bg` | `#1f1f1f` | `#f2f2f2` | Avatar circle background |
| `chat.avatarForeground` | `--color-chat-avatar-fg` | `#cccccc` | `#3b3b3b` | Avatar icon color (inherits foreground) |
| `chat.slashCommandBackground` | `--color-chat-slash-bg` | `#26477866` | `#adceff7a` | Slash command tag background |
| `chat.slashCommandForeground` | `--color-chat-slash-fg` | `#85b6ff` | `#26569e` | Slash command tag text |
| `chat.editedFileForeground` | `--color-chat-edited-file` | `#E2C08D` | `#895503` | Edited file name in file list |
| `chat.linesAddedForeground` | `--color-chat-lines-added` | `#54B054` | `#107C10` | Added lines count |
| `chat.linesRemovedForeground` | `--color-chat-lines-removed` | `#FC6A6A` | `#BC2F32` | Removed lines count |
| `chat.thinkingShimmer` | `--color-chat-thinking-shimmer` | `#ffffff` | `#000000` | Shimmer highlight peak |
| `chat.checkpointSeparator` | `--color-chat-checkpoint-sep` | `#585858` | `#a9a9a9` | Checkpoint separator line |
| `agentStatusIndicator.background` | `--color-agent-status-bg` | `rgba(255,255,255,0.05)` | `rgba(0,0,0,0.05)` | Status indicator background |

#### Inherited VS Code Tokens

| VS Code Token | CSS Custom Property | Dark Modern | Light Modern | Purpose |
|---|---|---|---|---|
| `editor.background` | `--color-editor-bg` | `#1e1e1e` | `#ffffff` | Primary surface |
| `editor.foreground` | `--color-editor-fg` | `#cccccc` | `#3b3b3b` | Primary text |
| `editor.selectionBackground` | `--color-editor-selection-bg` | `#264f78` | `#add6ff` | Selection highlight |
| `editorWidget.background` | `--color-widget-bg` | `#252526` | `#f3f3f3` | Floating widget surfaces |
| `foreground` | `--color-fg` | `#cccccc` | `#3b3b3b` | Default text |
| `descriptionForeground` | `--color-description-fg` | `rgba(204,204,204,0.7)` | `rgba(59,59,59,0.7)` | Secondary/muted text |
| `input.background` | `--color-input-bg` | `#313131` | `#ffffff` | Input field background |
| `input.border` | `--color-input-border` | `#3c3c3c` | `#cecece` | Input field border |
| `input.foreground` | `--color-input-fg` | `#cccccc` | `#3b3b3b` | Input text |
| `focusBorder` | `--color-focus-border` | `#007fd4` | `#0090f1` | Focus ring |
| `button.background` | `--color-button-bg` | `#0e639c` | `#007acc` | Primary button |
| `button.foreground` | `--color-button-fg` | `#ffffff` | `#ffffff` | Primary button text |
| `button.hoverBackground` | `--color-button-hover-bg` | `#1177bb` | `#0062a3` | Primary button hover |
| `button.secondaryBackground` | `--color-button-secondary-bg` | `#313131` | `#e8e8e8` | Secondary button |
| `button.secondaryForeground` | `--color-button-secondary-fg` | `#cccccc` | `#3b3b3b` | Secondary button text |
| `button.secondaryHoverBackground` | `--color-button-secondary-hover` | `#3c3c3c` | `#d6d6d6` | Secondary button hover |
| `textLink.foreground` | `--color-link-fg` | `#3794ff` | `#006ab1` | Link text |
| `textLink.activeForeground` | `--color-link-active-fg` | `#3794ff` | `#006ab1` | Link active/hover |
| `textPreformat.foreground` | `--color-code-fg` | `#ce9178` | `#a31515` | Inline code text |
| `textPreformat.background` | `--color-code-bg` | `rgba(10,10,10,0.4)` | `rgba(220,220,220,0.4)` | Inline code background |
| `textPreformat.border` | `--color-code-border` | `rgba(255,255,255,0.06)` | `rgba(0,0,0,0.06)` | Inline code border |
| `textBlockQuote.background` | `--color-blockquote-bg` | `rgba(127,127,127,0.1)` | `rgba(127,127,127,0.1)` | Blockquote background |
| `textBlockQuote.border` | `--color-blockquote-border` | `rgba(0,122,204,0.5)` | `rgba(0,122,204,0.5)` | Blockquote left border |
| `list.hoverBackground` | `--color-list-hover-bg` | `#2a2d2e` | `#e8e8e8` | List item hover |
| `list.activeSelectionBackground` | `--color-list-active-bg` | `#04395e` | `#0060c0` | Active list selection |
| `list.activeSelectionForeground` | `--color-list-active-fg` | `#ffffff` | `#ffffff` | Active list selection text |
| `errorForeground` | `--color-error-fg` | `#f48771` | `#e51400` | Error text |
| `icon.foreground` | `--color-icon-fg` | `#c5c5c5` | `#424242` | Icon color |
| `badge.background` | `--color-badge-bg` | `#4d4d4d` | `#c4c4c4` | Badge background |
| `badge.foreground` | `--color-badge-fg` | `#ffffff` | `#333333` | Badge text |
| `editorWarning.foreground` | `--color-warning-fg` | `#cca700` | `#bf8803` | Warning text |
| `editorInfo.foreground` | `--color-info-fg` | `#3794ff` | `#1a85ff` | Info text |

### 1.2 Typography Scale

VS Code defines chat font sizes as relative `em` units on a 13px base. The webapp uses a 14px base for mobile readability, with the same proportional scale.

| Token | Ratio | At 14px Base | Tailwind Class | Usage |
|---|---|---|---|---|
| `--vscode-chat-font-size-body-xs` | `0.846em` | ~12px | `text-xs` | Footer details, code inline |
| `--vscode-chat-font-size-body-s` | `0.923em` | ~13px | `text-sm` | Thinking text, descriptions, tool labels |
| `--vscode-chat-font-size-body-m` | `1em` | 14px | `text-base` | Body text, markdown paragraphs |
| `--vscode-chat-font-size-body-l` | `1.077em` | ~15px | `text-[15px]` | H3 headings |
| `--vscode-chat-font-size-body-xl` | `1.231em` | ~17px | `text-[17px]` | H2 headings |
| `--vscode-chat-font-size-body-xxl` | `1.538em` | ~22px | `text-[22px]` | H1 headings |

**Font family stack:**

```css
--font-sans: -apple-system, BlinkMacSystemFont, "Segoe UI", Roboto,
  Oxygen-Sans, Ubuntu, Cantarell, "Helvetica Neue", sans-serif;
--font-mono: "SF Mono", Monaco, Menlo, Consolas, "Ubuntu Mono",
  "Liberation Mono", "DejaVu Sans Mono", "Courier New", monospace;
```

**Line heights:**

| Context | Value | Tailwind |
|---|---|---|
| Body text / markdown | `1.5` | `leading-[1.5]` |
| Headings | `normal` | `leading-normal` |
| Code blocks | `1.4` | `leading-[1.4]` |
| Compact list items | `17px` | `leading-[17px]` |
| Session list detail row | `15px` | `leading-[15px]` |

**Font weights:**

| Weight | Value | Usage |
|---|---|---|
| Normal | 400 | Body text, descriptions |
| Medium | 500 | Section headers, line counts |
| Semibold | 600 | Headings (h1–h3), usernames, bold text |

### 1.3 Spacing & Layout

Extracted from `chat.css` and `agentsessionsviewer.css`:

| Token | Value | CSS Custom Property | Usage |
|---|---|---|---|
| Max content width | `950px` | `--layout-max-width` | Chat column cap |
| Message padding | `12px 16px` | — | `.interactive-item-container` |
| Compact message padding | `8px 20px` | — | `.interactive-item-compact` |
| Standard gap xs | `4px` | `--space-1` | Toolbar gaps, list item margins |
| Standard gap sm | `6px` | `--space-1.5` | Session item padding, toolbar gaps |
| Standard gap md | `8px` | `--space-2` | Header gap, button groups |
| Standard gap lg | `12px` | `--space-3` | Sidebar padding, section spacing |
| Standard gap xl | `16px` | `--space-4` | Paragraph margins, code block margins |
| Session item padding | `8px 6px` | — | `.agent-session-item` |
| Session section padding | `0 6px` | — | `.agent-session-section` |
| Approval row gap | `8px` | — | `.agent-session-approval-row` |
| Input container padding | `0 6px 6px 6px` | — | `.chat-input-container` |

**Border radius tokens:**

| Token | Value | CSS Custom Property |
|---|---|---|
| Small | `2px` | `--radius-sm` |
| Medium | `4px` | `--radius-md` |
| Large | `8px` | `--radius-lg` |
| Round | `9999px` | `--radius-full` |
| Session row | `6px` | — |

VS Code uses `var(--vscode-cornerRadius-medium)` and `var(--vscode-cornerRadius-large)` — these map to 4px and 8px respectively in default themes.

### 1.4 Icon System

**Primary library:** Lucide React (tree-shakeable, MIT licensed, ~1300 icons).

**Codicon → Lucide mapping** for the ~30 icons used in the CLI session UI:

| Context | VS Code Codicon | Lucide Equivalent | Size |
|---|---|---|---|
| Copilot avatar | `codicon-copilot` | Custom SVG (GitHub Copilot mark) | 14px |
| User avatar | `codicon-account` | `User` | 14px |
| Close/delete | `codicon-close` | `X` | 16px |
| Edit/rename | `codicon-edit` | `Pencil` | 14px |
| Terminal | `codicon-terminal` | `Terminal` | 14px |
| Folder opened | `codicon-folder-opened` | `FolderOpen` | 14px |
| Copy | `codicon-copy` | `Copy` | 14px |
| Git commit | `codicon-git-commit` | `GitCommitHorizontal` | 14px |
| Git pull request | `codicon-git-pull-request-create` | `GitPullRequestCreate` | 14px |
| Git pull request draft | `codicon-git-pull-request-draft` | `GitPullRequestDraft` | 14px |
| Git merge | `codicon-git-merge` | `GitMerge` | 14px |
| Git stash pop (apply) | `codicon-git-stash-pop` | `ArrowDownToLine` | 14px |
| Sync | `codicon-sync` | `RefreshCw` | 14px |
| Refresh | `codicon-refresh` | `RotateCcw` | 14px |
| Discard | `codicon-discard` | `Undo2` | 14px |
| Repo | `codicon-repo` | `BookMarked` | 14px |
| Chevron down | `codicon-chevron-down` | `ChevronDown` | 12px |
| Chevron right | `codicon-chevron-right` | `ChevronRight` | 12px |
| Check | `codicon-check` | `Check` | 12px |
| Error | `codicon-error` | `CircleX` | 14px |
| Warning | `codicon-warning` | `TriangleAlert` | 14px |
| Info | `codicon-info` | `Info` | 14px |
| Loading/spinner | `codicon-loading` | `Loader2` (animated) | 14px |
| Circle filled | `codicon-circle-filled` | `Circle` (filled) | 12px |
| Add | `codicon-add` | `Plus` | 14px |
| Send | `codicon-send` | `ArrowUp` | 16px |
| Stop | `codicon-debug-stop` | `Square` | 16px |
| Book | `codicon-book` | `BookOpen` | 12px |
| Pin | `codicon-pin` | `Pin` | 12px |
| Search | `codicon-search` | `Search` | 14px |
| Settings | `codicon-gear` | `Settings` | 14px |
| Menu | `codicon-menu` | `Menu` | 16px |
| More (ellipsis) | `codicon-ellipsis` | `MoreHorizontal` | 16px |

**Icon sizing:**

| Size | Pixels | Usage |
|---|---|---|
| XS | `12px` | Chevrons, status dots, thinking icons |
| SM | `14px` | Default toolbar icons, session list icons |
| MD | `16px` | Action buttons, header icons |
| LG | `20px` | Primary action buttons |

### 1.5 Theme Implementation

#### CSS Custom Property Approach

```css
/* app/styles/tokens.css */
@layer base {
  :root {
    /* Light theme (default) */
    --color-editor-bg: #ffffff;
    --color-editor-fg: #3b3b3b;
    --color-description-fg: rgba(59, 59, 59, 0.7);
    --color-input-bg: #ffffff;
    --color-input-border: #cecece;
    --color-focus-border: #0090f1;
    --color-chat-request-border: rgba(0, 0, 0, 0.10);
    --color-chat-request-bg: rgba(255, 255, 255, 0.62);
    --color-chat-avatar-bg: #f2f2f2;
    --color-chat-lines-added: #107C10;
    --color-chat-lines-removed: #BC2F32;
    --color-chat-thinking-shimmer: #000000;
    --color-chat-checkpoint-sep: #a9a9a9;
    --color-button-bg: #007acc;
    --color-button-fg: #ffffff;
    --color-button-hover-bg: #0062a3;
    --color-link-fg: #006ab1;
    --color-error-fg: #e51400;
    --color-code-fg: #a31515;
    --color-code-bg: rgba(220, 220, 220, 0.4);
    --color-list-hover-bg: #e8e8e8;
    /* ... remaining light values from Section 1.1 ... */
  }

  .dark {
    --color-editor-bg: #1e1e1e;
    --color-editor-fg: #cccccc;
    --color-description-fg: rgba(204, 204, 204, 0.7);
    --color-input-bg: #313131;
    --color-input-border: #3c3c3c;
    --color-focus-border: #007fd4;
    --color-chat-request-border: rgba(255, 255, 255, 0.10);
    --color-chat-request-bg: rgba(30, 30, 30, 0.62);
    --color-chat-avatar-bg: #1f1f1f;
    --color-chat-lines-added: #54B054;
    --color-chat-lines-removed: #FC6A6A;
    --color-chat-thinking-shimmer: #ffffff;
    --color-chat-checkpoint-sep: #585858;
    --color-button-bg: #0e639c;
    --color-button-fg: #ffffff;
    --color-button-hover-bg: #1177bb;
    --color-link-fg: #3794ff;
    --color-error-fg: #f48771;
    --color-code-fg: #ce9178;
    --color-code-bg: rgba(10, 10, 10, 0.4);
    --color-list-hover-bg: #2a2d2e;
    /* ... remaining dark values from Section 1.1 ... */
  }
}
```

#### Tailwind CSS v4 `@theme` Integration

```css
/* app/styles/theme.css */
@import "tailwindcss";

@theme {
  --color-surface: var(--color-editor-bg);
  --color-on-surface: var(--color-editor-fg);
  --color-muted: var(--color-description-fg);
  --color-primary: var(--color-button-bg);
  --color-on-primary: var(--color-button-fg);
  --color-border: var(--color-chat-request-border);
  --color-input: var(--color-input-bg);
  --color-ring: var(--color-focus-border);
  --color-destructive: var(--color-error-fg);
  --color-warning: var(--color-warning-fg);
  --color-info: var(--color-info-fg);
  --color-accent: var(--color-link-fg);
  --color-diff-added: var(--color-chat-lines-added);
  --color-diff-removed: var(--color-chat-lines-removed);

  --radius-sm: 2px;
  --radius-md: 4px;
  --radius-lg: 8px;

  --font-sans: -apple-system, BlinkMacSystemFont, "Segoe UI", Roboto,
    Oxygen-Sans, Ubuntu, Cantarell, "Helvetica Neue", sans-serif;
  --font-mono: "SF Mono", Monaco, Menlo, Consolas, "Ubuntu Mono",
    "Liberation Mono", "DejaVu Sans Mono", "Courier New", monospace;
}
```

#### Theme Toggle Mechanism

```typescript
// Persist preference in localStorage, default to system preference.
type Theme = 'light' | 'dark' | 'system';

function applyTheme(theme: Theme): void {
  const root = document.documentElement;
  const resolved = theme === 'system'
    ? (window.matchMedia('(prefers-color-scheme: dark)').matches ? 'dark' : 'light')
    : theme;
  root.classList.toggle('dark', resolved === 'dark');
  // Update meta theme-color for PWA status bar
  document.querySelector('meta[name="theme-color"]')
    ?.setAttribute('content', resolved === 'dark' ? '#1e1e1e' : '#ffffff');
}
```

---

## 2. Responsive Layout Architecture

### 2.1 Viewport Strategy

```html
<meta name="viewport"
  content="width=device-width, initial-scale=1, maximum-scale=1,
           interactive-widget=resizes-content, viewport-fit=cover" />
<meta name="theme-color" content="#1e1e1e" media="(prefers-color-scheme: dark)" />
<meta name="theme-color" content="#ffffff" media="(prefers-color-scheme: light)" />
```

Layout height uses `100dvh` (dynamic viewport height) to account for the collapsing Safari toolbar. Safe area insets are consumed at the shell level:

```css
.app-shell {
  height: 100dvh;
  padding-top: env(safe-area-inset-top);
  padding-bottom: env(safe-area-inset-bottom);
  padding-left: env(safe-area-inset-left);
  padding-right: env(safe-area-inset-right);
}
```

**Visual viewport fallback** for browsers without `dvh`:

```typescript
// Fallback for browsers without dvh support.
// Sets --vh to 1% of visual viewport height, updated on resize.
function setVH(): void {
  const vh = window.visualViewport?.height ?? window.innerHeight;
  document.documentElement.style.setProperty('--vh', `${vh * 0.01}px`);
}
window.visualViewport?.addEventListener('resize', setVH);
setVH();
```

```css
.app-shell {
  height: 100dvh;
  height: calc(var(--vh, 1vh) * 100); /* fallback */
}
```

### 2.2 Breakpoints

| Name | Range | Layout | Navigation |
|---|---|---|---|
| Mobile | `< 640px` | Single column | Bottom drawer (Vaul) |
| Tablet | `640px – 1024px` | Single column + toggle sidebar | Collapsible sidebar |
| Desktop | `> 1024px` | Sidebar + chat | Persistent sidebar |

Tailwind breakpoint config (v4 defaults are sufficient: `sm:640px`, `md:768px`, `lg:1024px`).

### 2.3 Shell Layout

#### Mobile (`< 640px`)

```
┌──────────────────────────────┐ ← safe-area-inset-top
│  ☰  Session Title     ⋯  ◑  │ ← Header (44px)
├──────────────────────────────┤
│                              │
│   Message List               │ ← flex-1, overflow-y: auto
│   (max-width: 950px,        │
│    margin: 0 auto)           │
│                              │
│   ┌────────────────────────┐ │
│   │ [User message]         │ │
│   └────────────────────────┘ │
│   ┌────────────────────────┐ │
│   │ 🤖 Response with       │ │
│   │    content parts...    │ │
│   └────────────────────────┘ │
│                              │
│              [↓]             │ ← ScrollToBottom (floating)
├──────────────────────────────┤
│  ┌──────────────────────[↑]┐ │ ← ChatInput
│  │ Ask anything...         │ │    (auto-resize textarea)
│  └─────────────────────────┘ │
│  [🔧 Mode ▾]  [+ Context]   │ ← Toolbar row
├──────────────────────────────┤ ← safe-area-inset-bottom
└──────────────────────────────┘
```

Session drawer (Vaul) slides up from bottom when `☰` is tapped:

```
┌──────────────────────────────┐
│  ════════ (drag handle) ═══  │ ← Vaul handle
│                              │
│  PINNED                      │
│  ┌──────────────────────────┐│
│  │ 📌 Session title   +3-1 ││
│  └──────────────────────────┘│
│  TODAY                       │
│  ┌──────────────────────────┐│
│  │ ● Session title   +12-4 ││
│  │ ○ Session title   2m ago ││
│  └──────────────────────────┘│
│  YESTERDAY                   │
│  ┌──────────────────────────┐│
│  │ ✓ Session title   +5-0  ││
│  └──────────────────────────┘│
│                              │
│  [+ New Session]             │ ← Fixed bottom CTA
└──────────────────────────────┘
```

#### Tablet Layout (640px–1024px)

Single-column layout with a toggleable sidebar overlay:
- Session list hidden by default, accessible via hamburger menu
- Sidebar slides in as a 320px overlay from the left
- Chat area takes full width when sidebar is hidden
- Same component rendering as desktop, narrower containers

#### Desktop (`> 1024px`)

```
┌──────────────────────────────────────────────────────────────┐
│ Header: Session Title                          ◑ Theme  ⋯   │
├──────────────┬───────────────────────────────────────────────┤
│              │                                               │
│  Session     │   Message Area (max-width: 950px, centered)  │
│  Sidebar     │                                               │
│  (280px)     │   ┌─────────────────────────────────────┐    │
│              │   │ User message                        │    │
│  ┌────────┐  │   └─────────────────────────────────────┘    │
│  │PINNED  │  │   ┌─────────────────────────────────────┐    │
│  │ Sess.. │  │   │ 🤖 Response                         │    │
│  ├────────┤  │   │    ▸ Thinking... (shimmer)          │    │
│  │TODAY   │  │   │    📄 file.ts  +12 -3               │    │
│  │ Sess.. │  │   │    markdown content...              │    │
│  │ Sess.. │  │   └─────────────────────────────────────┘    │
│  ├────────┤  │                                               │
│  │YESTER. │  │                                    [↓]       │
│  │ Sess.. │  │───────────────────────────────────────────────│
│  └────────┘  │   ┌─────────────────────────────────[↑]┐     │
│              │   │ Ask anything...                     │     │
│  [+ New]     │   └────────────────────────────────────┘     │
│              │   [🔧 Mode ▾]  [+ Context]                    │
└──────────────┴───────────────────────────────────────────────┘
```

#### Flexbox Structure

```
app-shell (flex col, h-dvh)
├── header (h-11, flex-none)
├── main (flex-1, flex row, min-h-0)
│   ├── sidebar (w-70, flex-none, hidden on mobile)
│   │   ├── search-input
│   │   ├── session-list (flex-1, overflow-y-auto)
│   │   └── new-session-button
│   └── chat-area (flex-1, flex col, min-w-0)
│       ├── message-list (flex-1, overflow-y-auto)
│       │   └── message-container (max-w-[950px], mx-auto)
│       └── input-area (flex-none)
│           ├── chat-input
│           └── toolbar
└── session-drawer (mobile only, Vaul)
```

---

## 3. Component Catalog

### Hybrid VS Code Alignment — File Naming Convention

Content-part React components mirror VS Code's source file naming 1:1, enabling visual `diff` between our React implementations and VS Code's imperative DOM code:

| Our React Component | VS Code Source File |
|---|---|
| `ChatMarkdownContentPart.tsx` | `chatMarkdownContentPart.ts` |
| `ChatCodeBlockContentPart.tsx` | `chatCodeBlockContentPart.ts` |
| `ChatTreeContentPart.tsx` | `chatTreeContentPart.ts` |
| `ChatThinkingContentPart.tsx` | `chatThinkingContent.ts` |
| `ChatCodeBlockPillWidget.tsx` | `chatCodeBlockPillWidget.ts` |
| `ChatConfirmationWidget.tsx` | `chatConfirmationWidget.ts` |

**Alignment maintenance:**

- Maintain a `VS_CODE_ALIGNMENT.md` mapping file tracking: our file → VS Code file → last synced VS Code commit SHA. This file is the single source of truth for drift detection.
- CSS token values (Section 1.1) are copied from VS Code and maintained in a token mapping file. When VS Code updates token values, the mapping file flags drift for review.
- Animation keyframes (Section 4) are copied verbatim from VS Code's CSS — pure CSS with zero dependencies, ensuring pixel-accurate behavior without abstraction overhead.

### 3.1 App Shell

#### `ChatApp`

Root layout component. Provides theme context, WebSocket connection, and global state.

```typescript
interface ChatAppProps {
  initialTheme?: 'light' | 'dark' | 'system';
  serverUrl: string;
}
```

```
┌──────────────────────────────┐
│ <ThemeProvider>               │
│   <WebSocketProvider>         │
│     <StoreProvider>           │
│       <AppShell />            │
│     </StoreProvider>          │
│   </WebSocketProvider>        │
│ </ThemeProvider>               │
└──────────────────────────────┘
```

**CSS:** `h-dvh flex flex-col bg-surface text-on-surface`

#### `Header`

Fixed-height app bar. Adapts content by breakpoint.

```typescript
interface HeaderProps {
  sessionTitle: string;
  sessionStatus: SessionStatus;
  onMenuToggle: () => void;
  onSettingsOpen: () => void;
}

type SessionStatus = 'idle' | 'running' | 'needs-input' | 'completed' | 'failed';
```

Mobile wireframe:
```
┌──────────────────────────────────────┐
│ [☰]  Session Title ···    [◑] [⋯]  │  44px
└──────────────────────────────────────┘
```

Desktop wireframe:
```
┌──────────────────────────────────────────────────┐
│ Copilot CLI   │  Session Title ···    [◑] [⋯]   │  44px
└──────────────────────────────────────────────────┘
```

**Props behavior:**
- `sessionStatus` drives a colored dot indicator: `idle` = gray, `running` = blue pulse, `needs-input` = amber pulse (matching `agent-session-needs-input-pulse`), `completed` = green, `failed` = red.
- Mobile: `☰` opens session drawer. Desktop: hamburger hidden; sidebar is persistent.

**Accessibility:** `<header role="banner">`, skip link target.

**CSS:**
```
h-11 flex items-center px-4 gap-2
border-b border-border bg-surface
```

### 3.2 Session Navigation

#### `SessionDrawer`

Mobile-only bottom sheet using Vaul. Slides up from bottom edge.

```typescript
interface SessionDrawerProps {
  open: boolean;
  onOpenChange: (open: boolean) => void;
}
```

Vaul snap points: `[0.5, 0.92]` — half-screen and near-full.

**Accessibility:** Focus trap when open. `role="dialog"`, `aria-label="Session navigation"`.

**CSS:** Vaul provides the base. Content container: `bg-surface rounded-t-xl`.

#### `SessionSidebar`

Desktop persistent sidebar. Hidden below `lg` breakpoint.

```typescript
interface SessionSidebarProps {
  className?: string;
}
```

```
┌──────────────┐
│ [🔍 Search ] │  ← Filter input
├──────────────┤
│ PINNED    (2)│  ← SessionSectionHeader
│ ┌──────────┐ │
│ │ Session  │ │  ← SessionCard
│ │ Session  │ │
│ └──────────┘ │
│ TODAY     (5)│
│ ┌──────────┐ │
│ │ Session  │ │
│ │ Session  │ │
│ │ ...      │ │
│ └──────────┘ │
├──────────────┤
│ [+ New Sess] │  ← Fixed bottom
└──────────────┘
```

**CSS:** `w-70 flex flex-col border-r border-border bg-surface`

#### `SessionList`

Virtualized scrollable list of session cards grouped by temporal section.

```typescript
interface SessionListProps {
  sessions: SessionListItem[];
  activeSessionId: string | null;
  onSessionSelect: (id: string) => void;
  filterText: string;
}

interface SessionListItem {
  id: string;
  title: string;
  status: SessionStatus;
  linesAdded: number;
  linesRemoved: number;
  lastActivity: Date;
  isPinned: boolean;
  isArchived: boolean;
  description?: string;
  badge?: string;
}
```

Temporal groups: **Pinned**, **Today**, **Yesterday**, **Last 7 Days**, **Older**, **Archived**.

**Accessibility:** `role="listbox"`, `aria-activedescendant` for keyboard selection.

#### `SessionCard`

Single session row. Mirrors VS Code's `.agent-session-item`.

```typescript
interface SessionCardProps {
  session: SessionListItem;
  isActive: boolean;
  onSelect: () => void;
  onContextMenu: (e: React.MouseEvent) => void;
}
```

```
┌──────────────────────────────────────────┐
│ [●]  Session Title         [...toolbar]  │  ← title row (17px line-height)
│      Description · +12 -3               │  ← details row (15px line-height)
└──────────────────────────────────────────┘
   ↑                              ↑
   icon-col (16px)                diff counts (tabular-nums)
   + main-col (pl-1.5)
```

**Mobile:** Touch target is the entire card (min 44px height). Toolbar shows on tap (no hover).

**CSS (from agentsessionsviewer.css):**
```
flex flex-row p-2 px-1.5 rounded-[6px]
hover:bg-list-hover-bg
```

- Title: `text-[13px] leading-[17px] truncate flex-1`
- Details row: `text-xs leading-[15px] text-muted gap-1 flex items-center`
- Diff added: `text-diff-added tabular-nums`
- Diff removed: `text-diff-removed tabular-nums`
- Separator dot: `·` (U+00B7) between description and diff counts
- Icon column: status icon, 16px, vertically centered
- Needs-input pulse: `animate-pulse` on the status icon (2s ease-in-out infinite, opacity 1→0.4→1)

#### `SessionSectionHeader`

Uppercase group header with count.

```typescript
interface SessionSectionHeaderProps {
  label: string;
  count: number;
}
```

```
PINNED                                (2)
```

**CSS (from agentsessionsviewer.css):**
```
text-[11px] font-medium uppercase text-muted
flex items-center px-1.5
```

Count: `opacity-70 mr-1 tabular-nums`

### 3.3 Message List

#### `MessageList`

Scrollable container for the conversation. Auto-scrolls to bottom on new messages unless the user has scrolled up.

```typescript
interface MessageListProps {
  messages: Message[];
  isStreaming: boolean;
}
```

**Auto-scroll logic:** Track `scrollTop + clientHeight >= scrollHeight - threshold` (threshold: 50px). If user scrolls up beyond threshold, disable auto-scroll and show `ScrollToBottom`. Resume auto-scroll when user scrolls to bottom or clicks `ScrollToBottom`.

**Accessibility:** `role="log"`, `aria-live="polite"`, `aria-label="Conversation messages"`.

**CSS:** `flex-1 overflow-y-auto overflow-x-hidden`

Inner container: `max-w-[950px] mx-auto w-full` (matches VS Code's `.interactive-session { max-width: 950px; margin: auto }`).

#### `MessageRow`

Wrapper that determines whether to render `RequestMessage` or `ResponseMessage`.

```typescript
interface MessageRowProps {
  message: Message;
  isLatest: boolean;
}

type Message = RequestMessage | ResponseMessage;

interface RequestMessage {
  type: 'request';
  id: string;
  content: string;
  timestamp: Date;
  attachments?: Attachment[];
}

interface ResponseMessage {
  type: 'response';
  id: string;
  parts: ContentPart[];
  isComplete: boolean;
  isStreaming: boolean;
  timestamp: Date;
  usage?: UsageInfo;
}
```

> **Note:** These interfaces are the canonical message types. The `Message` discriminated union in §7.1 (Zustand store) derives from these.

**CSS:** `p-3 px-4 flex flex-col` (matches `.interactive-item-container { padding: 12px 16px }`).

#### `RequestMessage`

User-authored message with avatar and optional attachments.

```
┌──────────────────────────────────────────────────┐
│ [👤] You                               2:34 PM   │  ← header
│                                                   │
│ Fix the authentication bug in the login flow      │  ← content
│                                                   │
│ [📎 auth.ts] [📎 login.test.ts]                   │  ← attachments
└──────────────────────────────────────────────────┘
```

**CSS:** Header: `flex items-center gap-2 mb-2`. Username: `text-[13px] font-semibold`. Avatar: `w-6 h-6 rounded-full bg-chat-avatar-bg flex items-center justify-center outline outline-1 outline-border`.

#### `ResponseMessage`

Assistant message containing an ordered sequence of content parts.

```
┌──────────────────────────────────────────────────┐
│ [🤖] Copilot                                     │
│                                                   │
│ ▸ Thinking... (shimmer)                          │  ← ThinkingBlock
│ ┌─ read_file ──────────────────────── ✓ ────────┐│  ← ToolInvocation
│ │  src/auth.ts                                   ││
│ └────────────────────────────────────────────────┘│
│                                                   │
│ I've identified the issue. The JWT token...      │  ← MarkdownContent
│                                                   │
│ ┌────────────────────────────────────────────────┐│  ← CodeBlock
│ │ ```typescript                          [Copy]  ││
│ │ const token = jwt.sign(payload, secret);       ││
│ │ ```                                            ││
│ └────────────────────────────────────────────────┘│
│                                                   │
│ 📄 auth.ts  +12 -3                              │  ← CodeBlockPill
│                                                   │
│ ──────── Checkpoint ─────────                    │  ← Checkpoint separator
│ ┌──── File Changes (3 files) ────────────────────┐│  ← FileChangesSummary
│ │ M  src/auth.ts         +12 -3                  ││
│ │ M  src/login.ts         +5 -1                  ││
│ │ A  src/utils/token.ts   +8 -0                  ││
│ └────────────────────────────────────────────────┘│
│                                                   │
│ [👍] [👎] [📋]                      1.2k tokens  │  ← footer
└──────────────────────────────────────────────────┘
```

Footer toolbar appears on hover (desktop) or always shows for latest response (matching VS Code behavior: `opacity: 0` → `opacity: 1` on hover/latest, 0.1s ease-in-out transition).

#### `ScrollToBottom`

Floating circular button, shown when user scrolls above auto-scroll threshold.

```typescript
interface ScrollToBottomProps {
  visible: boolean;
  onClick: () => void;
}
```

**CSS (from chat.css):**
```
absolute bottom-2 right-3 rounded-full w-7 h-7
bg-widget-bg border border-border shadow-sm
hidden  /* shown via state: block */
```

Icon: `ChevronDown` centered.

### 3.4 Content Parts

#### `MarkdownContent`

Rendered GFM markdown with syntax highlighting via Shiki.

```typescript
interface MarkdownContentProps {
  content: string;
  isStreaming?: boolean;
}
```

**Rendering pipeline:** Raw markdown → `remark-gfm` → `rehype` → Shiki for fenced code → React components.

**CSS (from chat.css):**
- Body: `leading-[1.5] text-base font-sans`
- Paragraphs: `mb-4` (margin: 0 0 16px 0)
- Headings: h1 `text-[22px]`, h2 `text-[17px]`, h3 `text-[15px]` — all `font-semibold mt-6 mb-3.5`
- Inline code: `font-mono text-xs text-code-fg bg-code-bg px-[3px] py-px rounded border border-code-border whitespace-pre-wrap`
- Links: `text-accent hover:text-link-active-fg`
- Blockquotes: `border-l-[5px] border-blockquote-border bg-blockquote-bg pl-2.5 pr-4`
- Tables: `border border-border rounded-md border-collapse:separate border-spacing-0` — cells `border border-border px-1.5 py-1`
- Lists: `ul { pl-6 }`, `ol { pl-7 }`, `li { my-1 }`
- Bold: `font-semibold`
- HR: `border-white/18` (dark) / `border-black/18` (light)
- Images: `max-w-full`

**Accessibility:** Code blocks announced with language label.

#### `ThinkingBlock`

Collapsible reasoning/thinking section with shimmer animation when active.

```typescript
interface ThinkingBlockProps {
  title: string;
  titleDetail?: string;
  isActive: boolean;
  isCollapsed: boolean;
  onToggle: () => void;
  children: React.ReactNode; // tool invocations, thinking text
  linesAdded?: number;
  linesRemoved?: number;
}
```

```
Collapsed + active (shimmer on title):
┌──────────────────────────────────────────────┐
│ ▸ Thinking...  analyzing code   +12 -3       │  ← shimmer on "Thinking..."
└──────────────────────────────────────────────┘

Expanded:
┌──────────────────────────────────────────────┐
│ ▾ Thinking...                                │
│ ┊                                            │
│ ┊ 🔍 read_file src/auth.ts            ✓     │  ← child tool invocation
│ ┊                                            │
│ ┊ The authentication logic needs...          │  ← thinking text
│ ┊                                            │
│ ┊ 🔍 search "jwt verify"              ✓     │  ← child tool invocation
│ ┊                                            │
└──────────────────────────────────────────────┘
```

The vertical line (`┊`) is a chain-of-thought connector (from `chatThinkingContent.css`):
```css
/* Implemented as a pseudo-element */
&::before {
  content: '';
  position: absolute;
  left: 10.5px;
  top: 0; bottom: 0;
  width: 1px;
  background-color: var(--color-chat-request-border);
}
```

The curved connector from header to first item uses a `border-left + border-bottom + border-bottom-left-radius` approach (absolute, left: 3px, top: 22px, height: 16px, width: 5px).

**Shimmer animation details (Section 4.1):** Applied via class when `isActive && isCollapsed`.

**Diff counts:** `+{n}` in `text-diff-added`, `-{n}` in `text-diff-removed`, shown in title row with `pl-1.5` gap.

**CSS:**
- Container: `relative text-muted`
- Thinking text: `text-sm pl-6 pr-3 py-1.5` (padding: 6px 12px 6px 24px)
- Collapse: Radix Collapsible with `grid-template-rows: 0fr → 1fr` transition

#### `ToolInvocation`

Displays a tool call with its execution state.

```typescript
interface ToolInvocationProps {
  name: string;
  args?: Record<string, unknown>;
  state: 'pending' | 'executing' | 'complete' | 'error';
  result?: string;
  isCollapsed: boolean;
  onToggle: () => void;
}
```

```
Collapsed (complete):
┌──────────────────────────────────────────────┐
│ ▸ read_file src/auth.ts                  ✓   │
└──────────────────────────────────────────────┘

Collapsed (executing):
┌──────────────────────────────────────────────┐
│ ▸ ⟳ search "jwt verify"                     │  ← shimmer on label
└──────────────────────────────────────────────┘

Expanded (complete):
┌──────────────────────────────────────────────┐
│ ▾ read_file src/auth.ts                  ✓   │
│ ┌────────────────────────────────────────────┐│
│ │ (tool result / code output)                ││
│ └────────────────────────────────────────────┘│
└──────────────────────────────────────────────┘
```

**State indicators:**
- `pending`: Pulsing border (`animate-pulse` on border-color)
- `executing`: Shimmer on label text (same gradient as ThinkingBlock)
- `complete`: Check icon in `text-diff-added`
- `error`: CircleX icon in `text-error-fg`

**Backend → Frontend State Mapping**

The backend (Doc 06) defines six granular tool-call states. The frontend collapses them into four display states:

| Backend State (Doc 06 §4) | Frontend State (§3.4) | Rationale |
|---|---|---|
| `streaming` | `executing` | Partial output arriving — show activity indicator |
| `pending-confirmation` | `pending` | Awaiting user approval before execution |
| `running` | `executing` | Tool actively executing on the backend |
| `pending-result-confirmation` | `pending` | Awaiting user approval of result |
| `completed` | `complete` | Terminal success state |
| `cancelled` | `error` | User or system cancelled — surface as failure |

**CSS:**
- Title button: `flex items-center gap-1 px-1.5 py-0.5 text-sm text-muted rounded-sm cursor-pointer`
- Hover: `text-on-surface`
- Result container: `border border-border rounded-md bg-surface text-sm mt-1 overflow-hidden`

#### `CodeBlock`

Read-only syntax-highlighted code block using Shiki.

```typescript
interface CodeBlockProps {
  code: string;
  language: string;
  fileName?: string;
}
```

```
┌──────────────────────────────────────────────────┐
│ typescript                                [Copy] │  ← toolbar (shown on hover)
│──────────────────────────────────────────────────│
│ 1  const token = jwt.sign(payload, secret, {     │
│ 2    expiresIn: '1h',                            │
│ 3  });                                           │
└──────────────────────────────────────────────────┘
```

**CSS (from codeBlockPart.css):**
- Container: `relative border border-border rounded-md overflow-hidden` (uses `--vscode-cornerRadius-medium`)
- Response code block: `border-input-border bg-[var(--color-code-block-bg)]`
- Request code block: `border-chat-request-code-border`
- Toolbar: `opacity-0 pointer-events-none` → on hover: `opacity-100 pointer-events-auto`
- Toolbar bar: `absolute -top-4 right-2.5 h-[26px] leading-[26px] bg-surface border border-border rounded-md z-10`
- Focus: `border-focus-border`

**Accessibility:** `role="region"`, `aria-label="Code block, {language}"`. Copy button: `aria-label="Copy code"`.

#### `DiffView`

File diff display using CodeMirror 6 merge extension.

```typescript
interface DiffViewProps {
  original: string;
  modified: string;
  fileName: string;
  language: string;
}
```

- **Mobile (`< 640px`):** Unified diff (single column, additions/deletions interleaved).
- **Desktop (`>= 640px`):** Split diff (side-by-side) when viewport permits, unified as fallback.

Header row with file name and line counts:

```
┌────────────────────────────────────────────────────┐
│ 📄 src/auth.ts                         +12 -3     │
│────────────────────────────────────────────────────│
│   1   │ + const token = jwt.sign(...)              │
│   2   │ - const token = oldSign(...)               │
│ ...   │                                            │
└────────────────────────────────────────────────────┘
```

**CSS:** Container: `border border-border rounded-md overflow-hidden`. Header: `flex justify-between items-center px-1 border-b border-border`.

#### `CodeBlockPill`

Compact inline file change indicator. Clickable to open diff view.

```typescript
interface CodeBlockPillProps {
  fileName: string;
  linesAdded: number;
  linesRemoved: number;
  fileIcon?: React.ReactNode;
  state: 'pending' | 'streaming' | 'complete';
  onClick: () => void;
}
```

```
 📄 auth.ts  +12 -3
```

**CSS (from chatCodeBlockPill.css):**
```
border border-border rounded px-[3px] py-px
text-sm text-muted whitespace-nowrap
cursor-pointer w-fit font-normal
leading-[1em] relative overflow-hidden
hover:bg-list-hover-bg hover:text-accent
```

- File icon: `text-[90%] align-middle`
- File label: `px-[3px] align-middle leading-[1em]`
- Added: `text-diff-added text-sm pl-1 align-middle`
- Removed: `text-diff-removed text-sm pl-1 pr-0.5 align-middle`
- Progress fill (streaming): absolute overlay, `bg-list-hover-bg`, width animated 0→100%, `transition: width 0.2s ease-out`

#### `FileChangesSummary`

Checkpoint-level summary of all file changes.

```typescript
interface FileChangesSummaryProps {
  files: FileChange[];
  isCollapsed: boolean;
  onToggle: () => void;
  totalAdded: number;
  totalRemoved: number;
}

interface FileChange {
  path: string;
  status: 'added' | 'modified' | 'deleted' | 'renamed';
  linesAdded: number;
  linesRemoved: number;
}
```

```
┌──────────────────────────────────────────────────┐
│ ▾ File Changes (3 files)           +25 -4        │
│──────────────────────────────────────────────────│
│  M  src/auth.ts              +12 -3              │
│  M  src/login.ts              +5 -1              │
│  A  src/utils/token.ts        +8 -0              │
└──────────────────────────────────────────────────┘
```

**CSS (from chatConfirmationWidget.css):** Wrapping container: `mb-2 border border-border rounded-md`. Title: `border-b border-border px-2 py-1 flex justify-between items-center gap-2.5`. File list rows: `rounded hover:bg-list-hover-bg px-0.5`. Line counts: `inline-flex gap-1 ml-1.5 text-[11px] font-medium tabular-nums`.

#### `ErrorContent`

Error display with severity icon and formatted message.

```typescript
interface ErrorContentProps {
  message: string;
  details?: string;
  severity: 'error' | 'warning' | 'info';
}
```

```
┌──────────────────────────────────────────────────┐
│ ⊘ Authentication failed                          │
│                                                   │
│ JWT secret not configured in environment.         │
│ Set AUTH_SECRET in your .env file.               │
└──────────────────────────────────────────────────┘
```

**CSS:** Icon uses severity-specific color (`text-destructive` / `text-warning` / `text-info`). Container: `p-2 px-3 flex gap-1.5`. Details: `font-mono text-xs text-muted`.

**Accessibility:** Error severity: `role="alert"`, `aria-live="assertive"`. Warning and info severities: `role="status"`, `aria-live="polite"`.

#### `ProgressIndicator`

Streaming progress — ellipsis animation or shimmer text.

```typescript
interface ProgressIndicatorProps {
  label?: string;
  isActive: boolean;
}
```

**CSS (from chat.css):** Ellipsis animation:
```css
@keyframes ellipsis {
  0%   { content: ""; }
  25%  { content: "."; }
  50%  { content: ".."; }
  75%  { content: "..."; }
  100% { content: ""; }
}

.progress-ellipsis::after {
  content: '';
  display: inline-block;
  width: 2em;
  white-space: nowrap;
  overflow: hidden;
  animation: ellipsis 1s steps(4, end) infinite;
}
```

#### `UsageDisplay`

Token usage display shown in the response footer.

```typescript
interface UsageDisplayProps {
  promptTokens: number;
  completionTokens: number;
  totalTokens: number;
}
```

**CSS:** `text-xs text-muted opacity-70 tabular-nums`

### 3.5 Approval & Interaction

#### `PermissionCard`

Tool approval card presented when a tool requires user consent.

```typescript
interface PermissionCardProps {
  toolName: string;
  description: string;
  command?: string;
  onAllow: () => void;
  onDeny: () => void;
  onAllowAlways?: () => void;
}
```

```
┌──────────────────────────────────────────────────┐
│ 🔧 Run terminal command                          │  ← title
│──────────────────────────────────────────────────│
│ $ npm run test -- --watch                        │  ← command preview
│──────────────────────────────────────────────────│
│ [Allow]  [Deny]                                  │  ← buttons
└──────────────────────────────────────────────────┘
```

**CSS (from chatConfirmationWidget.css):** Container: `mb-2 border border-border rounded-md`. Title bar: `border-b border-border px-2 py-1 flex justify-between`. Command area: `bg-chat-request-bg border-b border-border px-2.5 py-1.5`. Button row: `flex gap-1 px-2 py-1`.

**Accessibility:** `role="alertdialog"`, `aria-label="Permission request: {toolName}"`. Focus auto-set to Allow button on mount. Keyboard: Enter confirms, Escape denies.

#### `PlanApproval`

Plan mode exit confirmation — shown when transitioning from plan to execution.

```typescript
interface PlanApprovalProps {
  planSummary: string;
  onApprove: () => void;
  onReject: () => void;
  onEdit: () => void;
}
```

Same visual structure as `PermissionCard` but with three actions: Approve, Reject, Edit Plan.

#### `UserInputRequest`

Displayed when the assistant asks the user a question (ask-user pattern).

```typescript
interface UserInputRequestProps {
  question: string;
  placeholder?: string;
  onSubmit: (answer: string) => void;
  onDismiss: () => void;
}
```

```
┌──────────────────────────────────────────────────┐
│ Which database engine should I use?              │
│                                                   │
│ ┌────────────────────────────────────────────┐   │
│ │ Type your answer...                        │   │
│ └────────────────────────────────────────────┘   │
│                                        [Submit]  │
└──────────────────────────────────────────────────┘
```

**Accessibility:** `role="dialog"`, auto-focus input on mount.

#### `ConfirmationCard`

Generic confirmation dialog for destructive or significant actions.

```typescript
interface ConfirmationCardProps {
  title: string;
  message: string;
  acceptLabel: string;
  rejectLabel: string;
  onAccept: () => void;
  onReject: () => void;
  variant?: 'default' | 'destructive';
}
```

**CSS:** Same container as `PermissionCard`. Destructive variant uses `bg-destructive text-on-primary` for the accept button.

### 3.6 Input Area

#### `ChatInput`

Auto-resizing textarea with submit handling.

```typescript
interface ChatInputProps {
  value: string;
  onChange: (value: string) => void;
  onSubmit: () => void;
  onCancel: () => void;
  placeholder: string;
  isStreaming: boolean;
  disabled: boolean;
  maxHeight?: number; // default: 200px
}
```

```
┌──────────────────────────────────────────────[↑]┐
│ Ask anything...                                  │
│                                                  │
└──────────────────────────────────────────────────┘
```

**Behavior:**
- Auto-resizes from 1 line to `maxHeight` (200px), then scrolls internally.
- Enter submits. Shift+Enter inserts newline.
- Submit button toggles to Stop when `isStreaming`.
- `Cmd+Enter` always submits regardless of content.

**CSS (from chat.css):**
```
bg-input border border-input-border rounded-lg
px-1.5 pb-1.5 w-full relative
/* Focus: */ border-focus-border
```

Textarea itself: `bg-transparent outline-none resize-none w-full text-input-fg`.

**Accessibility:** `role="textbox"`, `aria-label="Message input"`, `aria-multiline="true"`.

#### `ModeSelector`

Picker for agent mode: interactive, autopilot, plan.

```typescript
interface ModeSelectorProps {
  mode: AgentMode;
  onModeChange: (mode: AgentMode) => void;
  disabled: boolean;
}

type AgentMode = 'interactive' | 'autopilot' | 'plan';
```

```
[🔧 Interactive ▾]
```

Implemented as a Radix DropdownMenu trigger button.

**CSS:** `flex items-center gap-1 h-4 px-[7px] py-[3px] text-icon-fg`. Chevron: `text-[10px] ml-1 opacity-75`. Invisible hit area padded to 44×44pt minimum per §5.1.

#### `AttachmentChips`

Displays attached context files or references.

```typescript
interface AttachmentChipsProps {
  attachments: Attachment[];
  onRemove: (id: string) => void;
}

interface Attachment {
  id: string;
  type: 'file' | 'symbol' | 'image' | 'problem';
  label: string;
  icon?: React.ReactNode;
}
```

```
[📄 auth.ts ×] [🔍 validateToken ×] [🖼️ screenshot.png ×]
```

**CSS (from chat.css):** `flex flex-row gap-1 mt-1.5 flex-wrap cursor-default`

Each chip: `flex items-center gap-1 px-2 py-0.5 rounded-md border border-border text-sm bg-code-bg`.

#### `SubmitButton`

Send/Stop toggle button positioned inside the input area.

```typescript
interface SubmitButtonProps {
  isStreaming: boolean;
  disabled: boolean;
  onSubmit: () => void;
  onCancel: () => void;
}
```

- Idle + content: `ArrowUp` icon, primary color.
- Idle + empty: `ArrowUp` icon, disabled.
- Streaming: `Square` icon (stop), red accent.

**CSS:** `w-7 h-7 rounded-full bg-primary text-on-primary flex items-center justify-center disabled:opacity-40`

#### `FollowUpSuggestions`

Suggestion pills displayed after a response completes.

```typescript
interface FollowUpSuggestionsProps {
  suggestions: string[];
  onSelect: (suggestion: string) => void;
}
```

```
[Fix the remaining tests]  [Add error handling]  [Write documentation]
```

**CSS:** `flex flex-wrap gap-2 mt-2`. Each pill: `px-3 py-1.5 text-sm border border-border rounded-full cursor-pointer hover:bg-list-hover-bg text-accent`.

### 3.7 Utility Components

#### `Avatar`

User or assistant avatar circle.

```typescript
interface AvatarProps {
  type: 'user' | 'assistant';
  size?: 'sm' | 'md'; // 18px | 24px. Default: `md` (24px).
}
```

**CSS (from chat.css):** `w-6 h-6 rounded-full bg-chat-avatar-bg flex items-center justify-center outline outline-1 outline-border`. Icon: `text-chat-avatar-fg text-sm`. Compact: `w-[18px] h-[18px]`, icon `text-xs`.

**Accessibility:** `aria-hidden="true"` — decorative.

#### `Badge`

Status and count badges.

```typescript
interface BadgeProps {
  variant: 'default' | 'success' | 'error' | 'warning';
  children: React.ReactNode;
}
```

**CSS:** `inline-flex items-center px-1.5 py-0.5 text-xs font-medium rounded-full`. Variants: default `bg-badge-bg text-badge-fg`, success `bg-green-600/20 text-diff-added`, error `bg-red-600/20 text-error-fg`.

#### `Tooltip`

Lightweight hover tooltip using Radix Tooltip primitive.

```typescript
interface TooltipProps {
  content: string;
  children: React.ReactNode;
  side?: 'top' | 'bottom' | 'left' | 'right';
  delayDuration?: number; // default: 400ms
}
```

**CSS:** `bg-widget-bg text-on-surface text-xs px-2 py-1 rounded border border-border shadow-md z-50`.

#### `CommandPalette`

`cmdk`-based command palette for keyboard-driven navigation.

```typescript
interface CommandPaletteProps {
  open: boolean;
  onOpenChange: (open: boolean) => void;
  commands: Command[];
}

interface Command {
  id: string;
  label: string;
  icon?: React.ReactNode;
  shortcut?: string;
  action: () => void;
  group?: string;
}
```

Triggered via `Cmd+K` (Mac) / `Ctrl+K` (other).

**CSS:** Centered modal overlay, `w-[min(480px,90vw)] bg-widget-bg border border-border rounded-lg shadow-xl`.

---

## 4. Animation & Transitions

### 4.1 Thinking Shimmer

Extracted directly from VS Code's `chatThinkingContent.css`. The shimmer creates a left-to-right highlight sweep using `background-clip: text` with a moving linear gradient.

```css
@keyframes chat-thinking-shimmer {
  0%   { background-position: 120% 0; }
  100% { background-position: -120% 0; }
}

.thinking-shimmer {
  background: linear-gradient(
    90deg,
    var(--color-muted)   0%,
    var(--color-muted)   30%,
    var(--color-chat-thinking-shimmer) 50%,
    var(--color-muted)   70%,
    var(--color-muted)   100%
  );
  background-size: 400% 100%;
  background-clip: text;
  -webkit-background-clip: text;
  -webkit-text-fill-color: transparent;
  animation: chat-thinking-shimmer 2s linear infinite;
  will-change: background-position;
}
```

**Technical notes:**
- `background-size: 400% 100%` provides the sweep space.
- `background-position: 120% → -120%` sweeps the highlight from right to left.
- `will-change: background-position` hints for GPU compositing.
- Only `background-position` is animated — no `transform`, no layout thrashing.
- Applied conditionally when `isActive && isCollapsed` on the ThinkingBlock title, and when a tool invocation is in `executing` state.

### 4.2 Collapsible Sections

Using Radix Collapsible with CSS `grid-template-rows` transition:

```css
[data-state="closed"] .collapsible-content {
  grid-template-rows: 0fr;
}

[data-state="open"] .collapsible-content {
  grid-template-rows: 1fr;
}

.collapsible-content {
  display: grid;
  transition: grid-template-rows 200ms ease-out;
}

.collapsible-content > div {
  overflow: hidden;
}
```

**Chevron rotation:**
```css
[data-state="closed"] .chevron-icon {
  transform: rotate(0deg);
  transition: transform 200ms ease-out;
}

[data-state="open"] .chevron-icon {
  transform: rotate(90deg);
}
```

### 4.3 Message Entry

New messages fade in with a subtle upward slide:

```css
@keyframes message-enter {
  from {
    opacity: 0;
    transform: translateY(8px);
  }
  to {
    opacity: 1;
    transform: translateY(0);
  }
}

.message-enter {
  animation: message-enter 200ms ease-out;
}
```

Applied only to the most recently appended message to avoid re-animating the entire list.

### 4.4 Streaming Text

CSS-only approach. A blinking cursor appears at the end of streaming content:

```css
@keyframes cursor-blink {
  0%, 100% { opacity: 1; }
  50%      { opacity: 0; }
}

.streaming-cursor::after {
  content: '▎';
  display: inline;
  color: var(--color-on-surface);
  animation: cursor-blink 1s step-end infinite;
  margin-left: 1px;
}
```

The `streaming-cursor` class is applied to the last rendered text node within a streaming response. Removed when `isStreaming` becomes false.

### 4.5 Tool Invocation State Transitions

```css
/* Pending: pulsing border */
.tool-invocation.pending {
  border-color: var(--color-muted);
  animation: tool-pending-pulse 2s ease-in-out infinite;
}

@keyframes tool-pending-pulse {
  0%, 100% { border-color: var(--color-muted); }
  50%      { border-color: transparent; }
}

/* Executing: shimmer on label (same as thinking shimmer) */
.tool-invocation.executing .tool-label {
  /* Apply .thinking-shimmer class */
}

/* Complete: success border transition */
.tool-invocation.complete {
  border-color: var(--color-diff-added);
  transition: border-color 150ms ease-out;
}

/* Error: error border transition */
.tool-invocation.error {
  border-color: var(--color-error-fg);
  transition: border-color 150ms ease-out;
}
```

All state transitions use `150ms` to feel snappy without being jarring. The `will-change: border-color` property is set only during active animation to avoid unnecessary compositing layers.

---

## 5. Mobile-Specific Patterns

### 5.1 Touch Targets

All interactive elements meet the 44×44pt minimum tap target (Apple HIG):

| Element | Natural Size | Touch Padding | Effective Size |
|---|---|---|---|
| Session card | ~56px height | Full width hit area | 56×full |
| Header buttons | 24px icon | 10px padding | 44×44 |
| Toolbar icons | 16px icon | 14px padding | 44×44 |
| Submit button | 28px circle | 8px padding | 44×44 |
| Suggestion pills | Auto | 12px h, 6px v | ≥44×44 |
| Copy button (code block) | 24px | 10px padding | 44×44 |

**Thumb zone analysis:** The input area and primary actions are bottom-anchored, within the natural thumb reach zone for one-handed operation. Session navigation is accessible via bottom drawer, not top-nav. Critical actions (submit, stop, approve) are all positioned in the bottom third of the viewport.

### 5.2 Gesture Interactions

| Gesture | Element | Action |
|---|---|---|
| Swipe right | Message row | Reveal action tray (copy, retry) |
| Pull down | Session drawer | Dismiss drawer (Vaul built-in) |
| Long press | Code block | Select all text + show copy tooltip |
| Swipe left | Session card | Reveal delete/archive actions |

**Swipe implementation:** Use a horizontal `touchmove` threshold (>30px) with `touchstart` origin tracking. Action tray slides in from the right edge. Released below threshold snaps back.

**Reduced motion:** All gesture animations respect `prefers-reduced-motion: reduce` — instant state changes, no sliding:
```css
@media (prefers-reduced-motion: reduce) {
  .swipe-action-tray { transition: none; }
  .tool-invocation.pending { animation: none; }
  .thinking-shimmer { animation: none; }
}
```

### 5.3 Virtual Keyboard Handling

The `interactive-widget=resizes-content` meta tag causes the visual viewport to shrink when the keyboard appears, pushing the input area above the keyboard without any JavaScript intervention.

**Fallback strategy** for browsers that do not support `interactive-widget`:

```typescript
function handleKeyboard(): void {
  const vv = window.visualViewport;
  if (!vv) return;

  vv.addEventListener('resize', () => {
    // When keyboard opens, visualViewport.height shrinks.
    // Offset the input area by the difference.
    const offset = window.innerHeight - vv.height;
    document.documentElement.style.setProperty(
      '--keyboard-inset', `${offset}px`
    );
  });
}
```

```css
.input-area {
  padding-bottom: calc(
    env(safe-area-inset-bottom) + var(--keyboard-inset, 0px)
  );
  transition: padding-bottom 100ms ease-out;
}
```

**Scroll position preservation:** When the keyboard appears, `scrollIntoView({ block: 'end', behavior: 'instant' })` is called on the last message to prevent content from being hidden behind the input area. The `instant` behavior prevents visible scroll animation.

### 5.4 PWA Considerations

**`manifest.json`:**

```json
{
  "name": "Copilot CLI Sessions",
  "short_name": "Copilot CLI",
  "display": "standalone",
  "orientation": "portrait-primary",
  "theme_color": "#1e1e1e",
  "background_color": "#1e1e1e",
  "start_url": "/",
  "icons": [
    { "src": "/icons/icon-192.png", "sizes": "192x192", "type": "image/png" },
    { "src": "/icons/icon-512.png", "sizes": "512x512", "type": "image/png" },
    { "src": "/icons/icon-maskable.png", "sizes": "512x512", "type": "image/png", "purpose": "maskable" }
  ]
}
```

> **Note:** `theme_color` in manifest.json is static. Runtime theme adaptation uses `<meta name="theme-color">` updated by the theme switching logic.

**Status bar styling:**
- Dark theme: `<meta name="theme-color" content="#1e1e1e">`
- Light theme: `<meta name="theme-color" content="#ffffff">`
- Updated dynamically on theme toggle.

**Apple-specific:**
```html
<meta name="apple-mobile-web-app-capable" content="yes" />
<meta name="apple-mobile-web-app-status-bar-style" content="black-translucent" />
<link rel="apple-touch-icon" href="/icons/apple-touch-icon.png" />
<link rel="apple-touch-startup-image" href="/icons/splash.png" />
```

---

## 6. Accessibility

### 6.1 ARIA Roles & Labels

| Element | Role | ARIA Attributes |
|---|---|---|
| Message list container | `role="log"` | `aria-label="Conversation messages"`, `aria-live="polite"` |
| Session list | `role="listbox"` | `aria-label="Sessions"`, `aria-activedescendant` |
| Session card | `role="option"` | `aria-selected`, `aria-label="{title} - {status}"` |
| Thinking block | `role="status"` | `aria-live="polite"`, `aria-label="Agent is thinking"` |
| Progress indicator | `role="status"` | `aria-live="polite"` |
| Permission card | `role="alertdialog"` | `aria-label="Permission request"`, `aria-describedby` |
| Code block | `role="region"` | `aria-label="Code block, {language}"` |
| Chat input | `role="textbox"` | `aria-label="Message input"`, `aria-multiline="true"` |
| Submit button | `role="button"` | `aria-label="Send message"` / `aria-label="Stop response"` |
| Collapsible trigger | `role="button"` | `aria-expanded`, `aria-controls` |
| Tool invocation | `role="status"` | `aria-label="{tool} - {state}"` |
| Session drawer | `role="dialog"` | `aria-label="Session navigation"`, `aria-modal="true"` |
| Command palette | `role="dialog"` | `aria-label="Command palette"`, `aria-modal="true"` |

### 6.2 Focus Management

**Focus trap in dialogs:** Permission cards, user input requests, and the command palette trap focus using Radix Dialog's built-in focus trapping. Tab cycles through interactive elements within the dialog.

**Return focus after dialog dismissal:** When a dialog closes, focus returns to the element that triggered it. Implemented via React ref capture on the triggering element.

**Skip-to-content link:** Hidden link at the top of the DOM, visible on focus:

```html
<a href="#message-list" class="sr-only focus:not-sr-only focus:absolute ...">
  Skip to messages
</a>
```

**Keyboard navigation map:**

| Context | Key | Action |
|---|---|---|
| Global | `Cmd+K` | Open command palette |
| Global | `Cmd+/` | Focus input |
| Global | `Escape` | Close drawer/dialog, blur input |
| Message list | `Tab` | Move focus to next interactive element |
| Session list | `↑` / `↓` | Navigate sessions |
| Session list | `Enter` | Open selected session |
| Input | `Enter` | Submit message |
| Input | `Shift+Enter` | Insert newline |
| Input | `Cmd+Enter` | Force submit |
| Code block | `Tab` | Skip code block (to next interactive) |
| Permission card | `Enter` | Confirm (Allow) |
| Permission card | `Escape` | Deny |

### 6.3 Screen Reader

**Message announcements:** New messages are announced via `aria-live="polite"` on the message list container. Only the newly appended message text is announced, not the entire history.

**Code block announcements:** Screen reader encounters:
> "Code block, TypeScript, 12 lines. Press Tab to skip."

**Tool invocation state changes:** State transitions are announced via `aria-live="polite"`:
- `pending` → "read_file, pending"
- `executing` → "read_file, executing"
- `complete` → "read_file, complete"
- `error` → "read_file, failed"

**Status icon descriptions:**

| Visual Icon | Screen Reader Text |
|---|---|
| Green dot | "Session completed" |
| Blue pulsing dot | "Session running" |
| Amber pulsing dot | "Session needs input" |
| Red dot | "Session failed" |
| Gray dot | "Session idle" |

---

## 7. State Management

### 7.1 Client State Shape

Complete Zustand store interface:

```typescript
// store/types.ts

interface AppState {
  // --- Theme ---
  theme: 'light' | 'dark' | 'system';
  resolvedTheme: 'light' | 'dark';
  setTheme: (theme: 'light' | 'dark' | 'system') => void;

  // --- Connection ---
  connectionStatus: 'connecting' | 'connected' | 'disconnected' | 'error';
  serverUrl: string;

  // --- Sessions ---
  sessions: Map<string, SessionState>;
  activeSessionId: string | null;
  sessionListFilter: string;

  // Session actions
  setActiveSession: (id: string) => void;
  addSession: (session: SessionState) => void;
  removeSession: (id: string) => void;
  updateSession: (id: string, patch: Partial<SessionState>) => void;

  // --- Active Chat ---
  messages: Message[];
  isStreaming: boolean;
  streamingMessageId: string | null;
  pendingApprovals: PendingApproval[];
  pendingUserInput: PendingUserInput | null;
  pendingDiff: PendingDiff | null;

  // Message actions
  appendMessage: (msg: Message) => void;
  updateMessage: (id: string, patch: Partial<Message>) => void;
  appendContentPart: (messageId: string, part: ContentPart) => void;
  updateContentPart: (messageId: string, partIndex: number, patch: Partial<ContentPart>) => void;
  setStreaming: (streaming: boolean, messageId?: string) => void;

  // --- Input ---
  inputValue: string;
  inputMode: AgentMode;
  attachments: Attachment[];
  setInputValue: (value: string) => void;
  setInputMode: (mode: AgentMode) => void;
  addAttachment: (attachment: Attachment) => void;
  removeAttachment: (id: string) => void;

  // --- UI State ---
  sidebarOpen: boolean;
  drawerOpen: boolean;
  commandPaletteOpen: boolean;
  scrolledToBottom: boolean;
  setSidebarOpen: (open: boolean) => void;
  setDrawerOpen: (open: boolean) => void;
  setCommandPaletteOpen: (open: boolean) => void;
  setScrolledToBottom: (atBottom: boolean) => void;
}

interface SessionState {
  id: string;
  title: string;
  status: SessionStatus;
  linesAdded: number;
  linesRemoved: number;
  lastActivity: Date;
  createdAt: Date;
  isPinned: boolean;
  isArchived: boolean;
  isRead: boolean;
  description?: string;
  badge?: string;
  isolationMode?: 'workspace' | 'worktree';
  hasGitRepository?: boolean;
  branchName?: string;
}

type SessionStatus = 'idle' | 'running' | 'needs-input' | 'completed' | 'failed';
type AgentMode = 'interactive' | 'autopilot' | 'plan';

type Message = {
  id: string;
  timestamp: Date;
} & (
  | { type: 'request'; content: string; attachments?: Attachment[]; }
  | { type: 'response'; parts: ContentPart[]; isComplete: boolean;
      isStreaming: boolean; usage?: UsageInfo; }
);

type ContentPart =
  | { type: 'markdownContent'; content: string; }
  | { type: 'thinking'; title: string; isActive: boolean;
      children: ContentPart[]; linesAdded?: number; linesRemoved?: number; }
  | { type: 'toolInvocation'; name: string; state: ToolState;
      args?: Record<string, unknown>; result?: string; }
  | { type: 'codeBlock'; code: string; language: string; fileName?: string; }
  | { type: 'codeBlockPill'; fileName: string; linesAdded: number;
      linesRemoved: number; state: 'pending' | 'streaming' | 'complete'; }
  | { type: 'fileChanges'; files: FileChange[]; totalAdded: number;
      totalRemoved: number; }
  | { type: 'error'; message: string; details?: string;
      severity: 'error' | 'warning' | 'info'; }
  | { type: 'progress'; label: string; isActive: boolean; }
  | { type: 'permission'; toolName: string; description: string;
      command?: string; id: string; }
  | { type: 'userInputRequest'; question: string; id: string; }
  | { type: 'followUps'; suggestions: string[]; };

type ToolState = 'pending' | 'executing' | 'complete' | 'error';

interface FileChange {
  path: string;
  status: 'added' | 'modified' | 'deleted' | 'renamed';
  linesAdded: number;
  linesRemoved: number;
}

interface Attachment {
  id: string;
  type: 'file' | 'symbol' | 'image' | 'problem';
  label: string;
  icon?: React.ReactNode;
}

interface UsageInfo {
  promptTokens: number;
  completionTokens: number;
  totalTokens: number;
}

interface PendingApproval {
  id: string;
  toolName: string;
  description: string;
  command?: string;
}

interface PendingUserInput {
  id: string;
  question: string;
  placeholder?: string;
}

interface PendingDiff {
  requestId: string;
  filePath: string;
  original: string;
  modified: string;
  description: string;
}
```

### 7.2 WebSocket Event → State Mapping

> **Note:** The event names below are the *abstracted* event vocabulary consumed by the Zustand store, not raw server message types. The WebSocket hook translates the raw protocol defined in doc-06 Section 6.1 (e.g., `response.session.list`, `event.assistant.message_delta`, `event.tool.execution_start`) into these domain events. Connection-level events (`connection.*`) are generated locally by the hook.

#### Wire Protocol → Frontend Event Mapping

The following table maps every raw wire event from Doc 06 Section 6.1 to the abstracted frontend event consumed by the Zustand store. Events marked *local* are synthesized by the WebSocket hook and have no single wire-protocol counterpart.

| Raw Wire Event (Doc 06 §6.1) | Frontend Hook Event (§7.2) | Notes |
|---|---|---|
| `response.session.list` | `session.list` | Initial session inventory |
| `response.session.created` | `session.created` | Response to `session.create` request |
| `response.session.loaded` | `session.loaded` | Historical turns populated |
| `response.session.deleted` | `session.deleted` | Confirmation of deletion |
| `response.error` | `server.error` | Protocol-level error |
| `event.assistant.message_delta` | `message.response.part.update` | Streaming markdown append |
| `event.assistant.message` | `message.response.complete` | Full turn content, ends stream |
| `event.tool.execution_start` | `message.response.part` | New toolInvocation content part added |
| `event.tool.execution_complete` | `message.response.part.update` | Tool state transitions to complete |

> **Discriminator note:** Both `event.assistant.message_delta` and `event.tool.execution_complete` map to `message.response.part.update`. Discriminate by `partIndex` and `part.type` to route updates to the correct content part renderer.
| `event.session.title_changed` | `session.title_changed` | Auto-generated or renamed title |
| `event.session.error` | `session.error` | SDK or API failure |
| `event.assistant.usage` | `usage.update` | Token counts for the response |
| `approval.permission_requested` | `approval.request` | Tool requires user permission |
| `approval.plan_mode_exit_requested` | `plan_mode.exit_requested` | Plan mode exit gate |
| `approval.user_input_requested` | `input.request` | Agent asks user a question |
| `diff.open` | `diff.open` | Passthrough — no translation |
| `diff.close` | `diff.close` | Passthrough — no translation |
| `session.name_updated` | `session.title_changed` | Manual rename via MCP tool |
| — | `message.response.start` | *Local:* emitted when first delta arrives |
| — | `message.request` | *Local:* echo of outbound user request |
| — | `session.updated` | *Local:* composite metadata patch |
| — | `session.status` | *Local:* derived from state transitions |
| — | `approval.resolved` | *Local:* emitted after `approval.respond` sent |
| — | `input.resolved` | *Local:* emitted after `approval.user_input_respond` sent |
| — | `connection.error` | *Local:* WebSocket error |
| — | `connection.reconnecting` | *Local:* auto-reconnect in progress |
| — | `connection.established` | *Local:* WebSocket connected |

| Server Event | State Mutation | Description |
|---|---|---|
| `session.list` | `sessions.clear(); data.forEach(addSession)` | Initial session list on connect |
| `session.created` | `addSession(data)` | New session created elsewhere |
| `session.updated` | `updateSession(id, data)` | Session metadata changed (status, title, diff counts) |
| `session.deleted` | `removeSession(id)` | Session removed |
| `session.status` | `updateSession(id, { status })` | Status change (running, completed, failed, needs-input) |
| `message.request` | `appendMessage({ type: 'request', ...data })` | User request echoed back |
| `message.response.start` | `appendMessage({ type: 'response', parts: [], isComplete: false, isStreaming: true }); setStreaming(true, id)` | Response stream begins |
| `message.response.part` | `appendContentPart(messageId, data.part)` | New content part (markdown, tool, etc.) |
| `message.response.part.update` | `updateContentPart(messageId, partIndex, data)` | Content part state change (tool executing → complete, markdown appended) |
| `message.response.complete` | `updateMessage(id, { isComplete: true, isStreaming: false, usage }); setStreaming(false)` | Response stream ends |
| `approval.request` | `pendingApprovals.push(data)` | Tool requires user permission |
| `approval.resolved` | `pendingApprovals.filter(a => a.id !== data.id)` | Approval granted/denied (remove from queue) |
| `input.request` | `pendingUserInput = data` | Agent asks user a question |
| `input.resolved` | `pendingUserInput = null` | User answered or dismissed |
| `session.loaded` | `messages = data.turns.map(toMessages)` | Historical messages populated on session load |
| `session.title_changed` | `updateSession(id, { title })` | Title auto-generated or manually renamed |
| `session.error` | `appendContentPart(activeMessageId, { type: 'error', ... })` | Session-level error (SDK or API failure) |
| `usage.update` | `updateMessage(id, { usage: data })` | Token usage reported for a response |
| `plan_mode.exit_requested` | `pendingApprovals.push({ ...data, toolName: 'plan_mode_exit' })` | Plan mode exit requires user approval |
| `diff.open` | `pendingDiff = data` | Server requests diff review |
| `diff.close` | `pendingDiff = null` | Diff review dismissed by server |
| `server.error` | `connectionStatus = 'error'` or show notification | Protocol-level error response |
| `connection.error` | `connectionStatus = 'error'` | WebSocket error |
| `connection.reconnecting` | `connectionStatus = 'connecting'` | Auto-reconnect in progress |
| `connection.established` | `connectionStatus = 'connected'` | Connected/reconnected |

---

## 8. Performance Architecture

React serves as the layout shell; computationally intensive hot paths are delegated to purpose-built engines that own their own DOM or run off-thread.

### Rendering Responsibility Split

| Layer | Handles | Notes |
|---|---|---|
| **React 19** | Layout composition, controls, chat message containers, settings UI | Standard React rendering; no hot-path work |
| **CodeMirror 6** | Code editing, diff views | Owns its own DOM subtree; React only mounts/unmounts the container element |
| **@tanstack/virtual** | Message list virtualization, file tree virtualization | Provides the windowing; React renders only visible items |
| **Web Workers** | Diff computation, markdown parsing (remark/rehype), syntax highlighting (Shiki worker mode), file tree indexing | Off-main-thread to prevent jank during streaming |

### Streaming Rendering Strategy

1. Buffer incoming `assistant.message_delta` events in a React `ref` (not state).
2. Flush buffered content to DOM at `requestAnimationFrame` boundary (30–60 fps).
3. Only the active (currently streaming) message triggers re-renders.
4. Completed messages are wrapped in `React.memo` and never re-render.

This decouples network event frequency from render frequency — delta events may arrive faster than frame rate without causing excess reconciliation.

### Virtualization Requirements

**Message list:**
- @tanstack/virtual with variable-height items.
- Only ~10–15 messages rendered in DOM at any time.
- Scroll-to-bottom behavior: auto-scroll during streaming, cease on manual scroll-up.
- Height measurement: `ResizeObserver` on each visible message row; cached for off-screen items.

**File trees:**
- Virtualized with @tanstack/virtual, collapsed by default.
- Expand/collapse is a local state toggle that adjusts the virtual item list without remounting the tree.

### Degradation Thresholds

| Condition | Threshold | Degraded Behavior | Detection |
|---|---|---|---|
| Code block | >50,000 lines | No syntax highlighting (plain `<pre>`) | Line count check before Shiki call |
| Diff view | >10,000 lines | Hunk-by-hunk (collapse non-visible hunks) | Diff output line count |
| Diff view | >100,000 lines | Summary only (file list + changed line counts) | Pre-diff file size check |
| Session messages | >500 messages | Paginate — load newest 50; fetch older on scroll-up | Message count from session metadata |
| `events.jsonl` | >5 MB | Server-side pagination only | File size check on session load |

**Degradation UX:**

- Show a subtle info banner when degraded mode activates: *"Large content — some features simplified for performance."*
- Never silently degrade — the user must be informed.
- Provide a "Load full content" escape hatch for code blocks (user-initiated, with a warning about potential UI stall).

### Long Session Support

Sessions can span days, weeks, or months with potentially thousands of messages. Virtualization and pagination are non-negotiable for this use case.

| Concern | Strategy |
|---|---|
| **Initial load** | Fetch session metadata + newest 50 messages only |
| **Older messages** | Loaded on demand — scroll-up triggers fetch of the next batch |
| **Session switching** | Unmount previous message list, mount new one. Do not retain the previous session's DOM. |
| **Memory pressure** | Evict off-screen message content parts (retain IDs and metadata for re-fetch) when total message count exceeds a configurable ceiling |

> For the complete constraints catalog and performance budgets, see [08-constraints-and-requirements.md](./08-constraints-and-requirements.md).

---

## 9. Visual Reference: VS Code Chat Anatomy

### 9.1 Chat Container (`.interactive-session`)

**VS Code approach:**
```css
.interactive-session {
  display: flex;
  flex-direction: column;
  max-width: 950px;
  height: 100%;
  margin: auto;
  position: relative;
}
```

Defines relative font size tokens (`--vscode-chat-font-size-body-*`) at this container level.

**Webapp replication:**
```
<div className="flex flex-col max-w-[950px] h-full mx-auto relative">
```

Font size tokens are defined on `:root` since the webapp lacks VS Code's nested font-size inheritance.

### 9.2 Message Container (`.interactive-item-container`)

**VS Code approach:**
```css
.interactive-item-container {
  padding: 12px 16px;
  display: flex;
  flex-direction: column;
  cursor: default;
  user-select: text;
  -webkit-user-select: text;
}
```

Request messages receive a semi-transparent background via `chat.requestBackground`. Response messages inherit the chat list background.

**Webapp replication:**
```
<div className="p-3 px-4 flex flex-col select-text">
```

Request messages: add `bg-chat-request-bg` class.

**Mobile adaptation:** Padding reduced to `p-3 px-3` below 640px for more horizontal content space.

### 9.3 Message Header (`.header`)

**VS Code approach:**
```css
.interactive-item-container .header {
  display: flex;
  align-items: center;
  justify-content: space-between;
  margin-bottom: 8px;
}

.header .avatar {
  width: 24px; height: 24px;
  border-radius: 50%;
  outline: 1px solid var(--vscode-chat-requestBorder);
}

.header .username {
  font-size: 13px;
  font-weight: 600;
}
```

The `.user` container wraps avatar + username with `gap: 8px`. Toolbar is `position: absolute; left: 0` for responses, hidden until hover/focus.

**Webapp replication:**
```
<div className="flex items-center justify-between mb-2">
  <div className="flex items-center gap-2">
    <Avatar type="assistant" />
    <span className="text-[13px] font-semibold">Copilot</span>
  </div>
  {/* Toolbar appears on hover/latest */}
</div>
```

**Mobile adaptation:** Toolbar always visible (no hover on touch devices). Uses `opacity-0 group-hover:opacity-100` on desktop, removed on mobile via `sm:opacity-0 sm:group-hover:opacity-100`.

### 9.4 Input Area (`.interactive-input-part`)

**VS Code approach:**
```css
.chat-input-container {
  background-color: var(--vscode-input-background);
  border: 1px solid var(--vscode-input-border, transparent);
  border-radius: var(--vscode-cornerRadius-large);
  padding: 0 6px 6px 6px;
  width: 100%;
  position: relative;
}

.chat-input-container.focused {
  border-color: var(--vscode-focusBorder);
}
```

The toolbar sits below the editor inside the input part, with `gap: 6px` and `margin-top: 4px`.

**Webapp replication:**
```
<div className="bg-input border border-input-border rounded-lg
                px-1.5 pb-1.5 w-full relative
                focus-within:border-focus-border">
  <textarea ... />
  <div className="flex items-center gap-1.5 mt-1">
    <ModeSelector />
    <AttachContextButton />
    <div className="ml-auto"><SubmitButton /></div>
  </div>
</div>
```

**Mobile adaptation:** Input is bottom-anchored. The secondary toolbar (mode selector, context) wraps to a second row if space is insufficient, using `flex-wrap`.

### 9.5 Scroll-to-Bottom Button (`.chat-scroll-down`)

**VS Code approach:**
```css
.interactive-session .chat-scroll-down {
  display: none;
  position: absolute;
  bottom: 7px; right: 12px;
  border-radius: 100%;
  width: 27px; height: 27px;
}

.interactive-session.show-scroll-down .chat-scroll-down {
  display: initial;
}
```

**Webapp replication:**
```
<button
  className={cn(
    "absolute bottom-2 right-3 rounded-full w-7 h-7",
    "bg-widget-bg border border-border shadow-sm",
    "flex items-center justify-center",
    scrolledToBottom && "hidden"
  )}
  onClick={scrollToBottom}
  aria-label="Scroll to bottom"
>
  <ChevronDown className="w-4 h-4" />
</button>
```

**Mobile adaptation:** Positioned slightly higher (bottom-4) to avoid overlapping with the input area.

### 9.6 Session List Items (`.agent-session-item`)

**VS Code approach:**
```css
.agent-session-item {
  display: flex;
  flex-direction: row;
  padding: 8px 6px;
}

.agent-session-icon-col {
  line-height: 17px;
}

.agent-session-icon {
  font-size: 12px;
  width: 16px; height: 16px;
}

.agent-session-main-col {
  padding-left: 6px;
}

.agent-session-title {
  font-size: 13px;
}

.agent-session-details-row {
  gap: 4px;
  font-size: 12px;
  line-height: 15px;
  color: var(--vscode-descriptionForeground);
}
```

Rows use `border-radius: 6px`. Diff counts use `font-variant-numeric: tabular-nums`. The needs-input pulse animates opacity 1→0.4→1 over 2s.

**Webapp replication:** Direct mapping to Tailwind classes:
```
<div className="flex flex-row p-2 px-1.5 rounded-[6px] hover:bg-list-hover-bg">
  <div className="flex items-start leading-[17px]">
    <StatusIcon status={session.status} className="w-4 h-4 text-xs" />
  </div>
  <div className="pl-1.5 flex-1 min-w-0">
    <div className="flex items-center leading-[17px] pb-1">
      <span className="text-[13px] truncate flex-1">{session.title}</span>
    </div>
    <div className="flex items-center gap-1 text-xs leading-[15px] text-muted">
      <span className="truncate">{session.description}</span>
      {hasDiff && <>
        <span>·</span>
        <span className="text-diff-added tabular-nums">+{session.linesAdded}</span>
        <span className="text-diff-removed tabular-nums">-{session.linesRemoved}</span>
      </>}
    </div>
  </div>
</div>
```

**Mobile adaptation:** Touch target expanded to full row height (min 48px). Long-press triggers context menu rather than hover-revealed toolbar.

### 9.7 Thinking Content (`.chat-thinking-box`)

**VS Code approach:** A nested tree structure with:
1. A collapsible header button with title, optional shimmer, and diff counts.
2. A vertical "chain of thought" line connecting child items via `::before` pseudo-elements.
3. Child items (tool invocations, thinking text) indented with connector dots.
4. Curved connector from header to first item using `border-left + border-bottom + border-radius`.

The chain-of-thought line uses `mask-image: linear-gradient(...)` to create gaps between items, with special handling for first/last/only children.

**Webapp replication:** The connector line is replicated using `absolute left-[10.5px] top-0 bottom-0 w-px bg-border` with Tailwind's `before:` prefix for the pseudo-element. The mask-image approach is kept as custom CSS since Tailwind does not have a built-in utility for this.

```css
/* Custom CSS for chain-of-thought connector */
.thinking-item::before {
  content: '';
  position: absolute;
  left: 10.5px;
  top: 0;
  bottom: 0;
  width: 1px;
  background-color: var(--color-border);
  mask-image: linear-gradient(
    to bottom, #000 0 5px, transparent 5px 25px, #000 25px 100%
  );
}

.thinking-item:first-child::before {
  mask-image: linear-gradient(
    to bottom, transparent 0 25px, #000 25px 100%
  );
}

.thinking-item:last-child::before {
  mask-image: linear-gradient(
    to bottom, #000 0 5px, transparent 5px 100%
  );
}

.thinking-item:only-child::before {
  background: none;
  mask-image: none;
}
```

**Mobile adaptation:** Connector line is slightly simplified — mask-image remains but left offset adjusts for narrower padding. Thinking items use smaller horizontal padding (`px-3` instead of `px-4`).

### 9.8 Code Block Pills (`.chat-codeblock-pill-widget`)

**VS Code approach:** Inline pill with file icon, name, and ±line counts. Border with `1px solid var(--vscode-chat-requestBorder)`, `border-radius: 4px`, `padding: 1px 3px`. On hover, background changes to `list.hoverBackground` and text to `textLink.foreground`. A progress fill overlay animates width during streaming.

**Webapp replication:**
```
<button
  className="border border-border rounded px-[3px] py-px
             text-sm text-muted whitespace-nowrap
             cursor-pointer w-fit font-normal
             leading-[1em] relative overflow-hidden
             hover:bg-list-hover-bg hover:text-accent"
  onClick={openDiff}
>
  <FileIcon className="text-[90%] align-middle" />
  <span className="px-[3px] align-middle leading-[1em]">{fileName}</span>
  {linesAdded > 0 && <span className="text-diff-added text-sm pl-1">+{linesAdded}</span>}
  {linesRemoved > 0 && <span className="text-diff-removed text-sm pl-1 pr-0.5">-{linesRemoved}</span>}
  {state === 'streaming' && (
    <div className="absolute inset-0 bg-list-hover-bg transition-[width] duration-200 ease-out"
         style={{ width: `${progress}%` }} />
  )}
</button>
```

**Mobile adaptation:** No change required — pills are naturally touch-friendly at their rendered size, and no hover state is needed (tap activates).

### 9.9 Confirmation Widget (`.chat-confirmation-widget2`)

**VS Code approach:** A bordered card with:
1. Title bar: `border-bottom`, `padding: 4px 8px`, flex layout with title and actions.
2. Message area: `bg-chat-request-bg`, `border-bottom`, `padding: 6px 9px`.
3. Button bar: `padding: 4px 8px`, flex with `gap: 4px`.

The newer `chat-confirmation-widget2` uses full-width border with rounded corners (`--vscode-cornerRadius-medium`).

**Webapp replication:**
```
<div className="mb-2 border border-border rounded-md">
  <div className="border-b border-border px-2 py-1 flex justify-between items-center gap-2.5">
    {/* Title with icon */}
  </div>
  <div className="bg-chat-request-bg border-b border-border px-2.5 py-1.5">
    {/* Command or content preview */}
  </div>
  <div className="flex gap-1 px-2 py-1">
    {/* Action buttons */}
  </div>
</div>
```

**Mobile adaptation:** Buttons expand to full-width stacked layout below 360px viewport width. Touch targets ensured at 44px height.

---

*End of specification.*
