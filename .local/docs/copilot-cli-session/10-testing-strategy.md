# 10 — Testing Strategy

> **Status:** Draft
> Last updated: 2025-07-19
> Depends on: [05-implementation-guide.md](./05-implementation-guide.md), [08-constraints-and-requirements.md](./08-constraints-and-requirements.md)
> Related: [09-deployment.md](./09-deployment.md), [04-data-models.md](./04-data-models.md)

---

## Table of Contents

1. [Testing Philosophy](#1-testing-philosophy)
2. [Test Tooling & Configuration](#2-test-tooling--configuration)
3. [Test File Structure](#3-test-file-structure)
4. [Unit Tests — Stores & Utilities](#4-unit-tests--stores--utilities)
5. [Component Tests](#5-component-tests)
6. [Integration Tests](#6-integration-tests)
7. [E2E Tests (Puppeteer/CDP)](#7-e2e-tests-puppeteercdp)
8. [Performance Tests](#8-performance-tests)
9. [Accessibility Tests](#9-accessibility-tests)
10. [Visual Regression Tests](#10-visual-regression-tests)
11. [Constraint Verification Matrix](#11-constraint-verification-matrix)
12. [Mock & Fixture Catalog](#12-mock--fixture-catalog)
13. [CI Pipeline & Coverage Gates](#13-ci-pipeline--coverage-gates)
14. [TDD Implementation Workflow](#14-tdd-implementation-workflow)
15. [AI-Agent-Driven Development Testing](#15-ai-agent-driven-development-testing)

---

## 1. Testing Philosophy

### Core Principles

1. **Constraint-Driven** — Every constraint from [08-constraints-and-requirements.md](./08-constraints-and-requirements.md) maps to at least one test. The [Constraint Verification Matrix](#11-constraint-verification-matrix) is the source of truth.
2. **Testing Trophy** — Favor integration over unit tests: Static ~10%, Unit ~25%, Integration ~40%, E2E ~25%.
3. **Behavior Over Implementation** — Assert user-visible behavior, not internals.
4. **Deterministic** — Fake timers for time-dependent tests, MSW for network, mocked rAF for animations.
5. **Dual Paradigm** — Automated tests (CI) for regression; AI-agent testing (§15) for exploratory dev-time validation.

### Coverage Targets

| Category | Branch | Line | Rationale |
|----------|--------|------|-----------|
| Zustand stores | 95% | 95% | Single source of truth for all state |
| SDK client layer | 95% | 95% | Entry point for all SDK interactions |
| Security utilities | 100% | 100% | Zero tolerance for gaps |
| Data mappers / Zod schemas | 100% | 100% | Contract boundaries |
| React components | 80% | 85% | Behavior-driven via RTL |
| Web Workers | 90% | 90% | Isolated, easily testable |
| Integration tests | — | — | 100% of critical user journeys |
| E2E tests | — | — | All happy paths + top 5 error scenarios |

### What We Do NOT Test

- VS Code extension host internals (SCP-04, SCP-05)
- Third-party library internals (CodeMirror, Shiki, Zustand)
- Offline mode (SCP-06)

### Risk-Based Test Priority

Tests are prioritized by impact × probability. High-priority areas get the most test investment:

| Priority | Area | Risk | Min Coverage |
|----------|------|------|-------------|
| P0 | Security (SEC-*) | Data leak, unauthorized access | 100% branch |
| P0 | SDK client (SDK-*) | Broken sessions, data loss | 95% branch |
| P1 | WebSocket / streaming | Missed messages, stale UI | 90% branch |
| P1 | Store state management | Inconsistent UI, lost state | 95% branch |
| P2 | Component rendering | Visual bugs, a11y gaps | 80% branch |
| P2 | Workers (diff/md/tree) | Degraded perf, fallback needed | 90% branch |
| P3 | Theming / visual polish | Cosmetic issues | 70% branch |

---

## 2. Test Tooling & Configuration

| Layer | Tool | Purpose |
|-------|------|---------|
| Runner | **Vitest** | Vite-native, ESM-first |
| DOM | **jsdom** | Browser simulation (broader ARIA support than happy-dom) |
| Components | **@testing-library/react** + **user-event** | Behavior-driven rendering + interaction |
| E2E | **Puppeteer** | CDP-based browser automation (NOT Playwright) |
| Accessibility | **axe-core** / **vitest-axe** | WCAG 2.1 AA |
| Visual | **pixelmatch** + **pngjs** | Pixel-level screenshot diff |
| API mock | **MSW** | Network-level interception |
| Benchmarks | **vitest.bench** | Statistical micro-benchmarks |

Vitest config pattern (key settings only):

```typescript
// vitest.config.ts
export default defineConfig({
  test: {
    globals: true,
    environment: 'jsdom',
    setupFiles: ['./src/test/setup.ts'],
    coverage: {
      provider: 'v8',
      thresholds: { branches: 80, functions: 80, lines: 85, statements: 85 },
    },
  },
});
```

### Test Setup File

```typescript
// src/test/setup.ts
import '@testing-library/jest-dom/vitest';
import 'vitest-axe/extend-expect';
import { cleanup } from '@testing-library/react';
import { afterEach, vi } from 'vitest';

afterEach(cleanup);

// Mock requestAnimationFrame for streaming store tests
vi.stubGlobal('requestAnimationFrame', (cb: FrameRequestCallback) => setTimeout(cb, 0));
vi.stubGlobal('cancelAnimationFrame', (id: number) => clearTimeout(id));
```

---

## 3. Test File Structure

```
src/test/
├── setup.ts                 # RTL cleanup, MSW, axe matchers
├── helpers/                 # render.tsx, factories.ts, ws-mock.ts, worker-mock.ts
├── mocks/                   # copilot-client.ts, mcp-server.ts, handlers.ts (MSW)
├── fixtures/                # sessions/, events/, screenshots/
├── integration/             # Store ↔ Component ↔ WS specs (*.spec.ts)
├── accessibility/           # axe-core audits
├── perf/                    # vitest.bench benchmarks (*.bench.ts)
└── e2e/                     # Puppeteer tests (*.e2e.ts)

src/stores/__tests__/        # Zustand store unit tests
src/components/__tests__/    # Component unit tests
src/workers/__tests__/       # Web Worker unit tests
src/security/__tests__/      # Security utility unit tests
```

### Naming Conventions

| Type | Pattern | Example |
|------|---------|---------|
| Unit test | `*.test.ts` | `useStreamingStore.test.ts` |
| Component test | `*.test.tsx` | `SessionList.test.tsx` |
| Integration test | `*.spec.ts` / `*.spec.tsx` | `streaming-pipeline.spec.ts` |
| E2E test | `*.e2e.ts` | `session-lifecycle.e2e.ts` |
| Benchmark | `*.bench.ts` | `streaming.bench.ts` |

---

## 4. Unit Tests — Stores & Utilities

### 4.1 Store Test Coverage Map

Zustand stores are tested via their vanilla API (`getState()` / `subscribe()`). No React rendering needed.

| Store | State / Actions to Test | Constraints |
|-------|------------------------|-------------|
| **useStreamingStore** | `appendChunk` buffers before flush; `flushSession` merges chunks; rAF-batched flush triggers single subscriber notification; per-session buffer isolation; degradation flag when content > threshold | TCH-03, TCH-20, TCH-22 |
| **useChatStore** | Add user/assistant messages in order; inline tool invocations; clear on session switch; **no write/persist methods** (read-only events.jsonl) | SDK-03, SDK-11, TCH-17 |
| **useSessionStore** | Load sessions from disk directory structure; single active session mutex; pagination of session list; create + switch sessions | SDK-06, SDK-10, OPS-08, TCH-08 |
| **useConnectionStore** | Connect/disconnect state transitions; exponential backoff (1→2→4→8→max 30s); backoff reset on success; auth token in connection | TCH-07, ARC-03, OPS-05, SEC-09 |
| **useThemeStore** | Toggle between exactly 2 themes (Dark Modern, Light Modern); CSS custom properties update; persistence | SCP-03 |

Store test pattern (one example for reference):

```typescript
describe('useStreamingStore', () => {
  beforeEach(() => { useStreamingStore.getState().reset(); vi.useFakeTimers(); });
  afterEach(() => vi.useRealTimers());

  it('should batch chunks into single render on rAF (TCH-03, TCH-22)', () => {
    const spy = vi.fn();
    useStreamingStore.subscribe(spy);
    const s = useStreamingStore.getState();
    for (let i = 0; i < 50; i++) s.appendChunk('s1', { text: `c${i} ` });
    const before = spy.mock.calls.length;
    vi.runAllTimers();
    expect(spy.mock.calls.length).toBe(before + 1);
  });
});
```

### 4.2 SDK Client Tests

| What to Test | Constraints |
|-------------|-------------|
| `connect()` starts MCP server, blocks until handshake | SDK-01, SDK-02, SDK-08 |
| `disconnect()` releases lock file + unix socket | SDK-07, OPS-02, OPS-03 |
| `getCapabilities()` returns negotiated capabilities | SDK-05 |
| Idle timeout disconnects after 30s | SDK-04 |
| Stale lock detection + cleanup | OPS-04 |
| GitHub token used for auth | SDK-12 |
| Session directory structure validated | SDK-10 |
| Agent mode communicated via capabilities | SDK-09 |

### 4.3 Security Utility Tests

| What to Test | Constraints |
|-------------|-------------|
| Nonce generation: unique, cryptographically random | SEC-02, OPS-09 |
| Nonce validation: accept valid, reject empty/tampered | SEC-02 |
| Host header check: reject non-localhost | SEC-03 |
| CSP header: restrictive policy generated correctly | SEC-06 |
| JWT validation: accept valid, reject expired/malformed | SEC-08 |
| Unix socket permissions: 0600 | SEC-04 |

### 4.4 Worker & Data Mapper Tests

| What to Test | Constraints |
|-------------|-------------|
| DiffWorker: compute diff off main thread | TCH-04 |
| DiffWorker: graceful error on malformed input | TCH-04 |
| MarkdownWorker: parse markdown off main thread | TCH-05 |
| MarkdownWorker: handle large (100KB+) documents | TCH-05 |
| TreeWorker: build file tree off main thread | TCH-06 |
| TreeWorker: handle empty + deeply nested paths | TCH-06 |
| `mapSessionFromDisk`: valid JSON → model, malformed → Zod error | — |
| `mapEventsToMessages`: events.jsonl → message array | SDK-03 |
| `mapEventsToMessages`: handle truncated/corrupt lines gracefully | SDK-03 |

### 4.5 Zod Schema Validation Tests

All request/response boundaries must validate with Zod schemas. 100% coverage required.

| Schema | Valid Case | Invalid Cases |
|--------|-----------|--------------|
| `SessionSchema` | Full session object | Missing id, invalid status, extra fields |
| `EventSchema` | Each event type | Unknown type, missing timestamp |
| `CapabilitiesSchema` | All capability flags | Missing required fields |
| `HealthResponseSchema` | `{ status: "ok" }` | Missing status |

---

## 5. Component Tests

### Component Coverage Map

All 15 components tested via React Testing Library (render → assert → interact).

| Component | What to Assert | Constraints |
|-----------|---------------|-------------|
| **ChatMarkdownContentPart** | Headings render at correct level; code blocks have Shiki classes; links have correct `href`; streaming content updates incrementally | TCH-05, TCH-15 |
| **ChatCodeBlockContentPart** | Shiki highlighting for read-only; CodeMirror 6 for editable; copy button works; renders without Shiki fallback | TCH-01, TCH-14, TCH-15 |
| **ThinkingBlock** | Collapsible via `aria-expanded`; respects `prefers-reduced-motion`; shows streaming indicator | TCH-12 |
| **ToolInvocation** | Status transitions: running → completed → error; displays tool name + params; `role="status"`, `aria-live="polite"` | — |
| **PermissionCard** | `role="alertdialog"` with `aria-label`; focus trap; Allow/Reject buttons ≥ 44×44px touch target | SEC-10, TCH-11 |
| **ApprovalDialog** | Focus on primary action; Escape dismisses; Tab cycles within dialog | SEC-10 |
| **ChatInputBox** | Enter submits; Shift+Enter adds newline; empty input blocked; accessible label | — |
| **SessionList** | Renders cards from data; `role="listbox"` + arrow key nav; virtualizes >50 items (DOM count < item count); pagination | TCH-02, TCH-08, ARC-04 |
| **SessionCard** | Displays title + status + file count; click calls onSelect; time grouping (Today/Yesterday/Older) | ARC-04 |
| **MessageList** | `role="log"` + `aria-live="polite"`; virtualized rendering; auto-scrolls on new message | TCH-02 |
| **SessionDrawer** | Vaul bottom sheet on mobile viewport; sidebar on desktop; `role="dialog"` + focus trap | TCH-13 |
| **App** | Theme toggle renders both themes; lazy-loads heavy components (dynamic imports); mounts without errors | SCP-03, TCH-21 |
| **CommandPalette** | Ctrl+K opens; fuzzy search filters; Escape closes; `role="dialog"` | — |
| **StatusBar** | Shows connection status; updates on reconnect; displays session info | OPS-05 |
| **ErrorBoundary** | Catches render errors; shows fallback UI; `role="alert"` + `aria-live="assertive"` | — |

### Component Test Guidelines

- Use `screen.getByRole()` over `getByTestId()` whenever possible — ensures a11y compliance by default
- Use `userEvent` (not `fireEvent`) for all user interactions — more realistic event sequences
- Test loading/error/empty states for every component, not just the happy path
- Test with both themes by wrapping with `ThemeProvider` + each theme value

---

## 6. Integration Tests

Integration tests verify store → component → WebSocket pipelines work end-to-end in jsdom.

### Integration Coverage Map

| Scenario | What to Verify | Constraints |
|----------|---------------|-------------|
| **Store → UI rendering** | Mutate `useChatStore` → `MessageList` renders updated messages | SDK-11 |
| **Streaming pipeline** | `useStreamingStore.appendChunk()` → content appears in `MessageList` after rAF flush | TCH-03, TCH-22 |
| **Tool invocations inline** | Add tool invocation to chat store → `ToolInvocation` component renders with correct status | — |
| **WebSocket connect** | Mock WS `open` → `useConnectionStore` transitions to `connected` → UI shows connected | ARC-03 |
| **WebSocket message dispatch** | Mock WS `message` with `content_delta` → streaming store buffers → UI renders content | SDK-11, ARC-03 |
| **WebSocket reconnection** | Mock WS `close(1006)` → store transitions to `reconnecting` → UI shows indicator | TCH-07, OPS-05 |
| **SDK → Store → UI** | Mock `CopilotClient.listSessions()` → `useSessionStore` loads → `SessionList` renders | SDK-01, SDK-05 |
| **Theme switching** | Toggle `useThemeStore` → CSS custom properties change on rendered components | SCP-03 |
| **Session navigation** | Click `SessionCard` → `useChatStore` loads messages → `MessageList` renders them | ARC-04 |

Integration test pattern (one example):

```typescript
it('should update UI as streaming chunks arrive (TCH-03)', async () => {
  useChatStore.getState().addAssistantMessage('s1', '', { streaming: true });
  render(<MessageList sessionId="s1" />);
  useStreamingStore.getState().appendChunk('s1', { text: 'Hello world!' });
  await waitFor(() => expect(screen.getByText(/Hello world!/)).toBeInTheDocument());
});
```

### Integration Test Guidelines

- Each test covers a **cross-boundary flow** (store ↔ component, client ↔ store, WS ↔ store)
- Use real Zustand stores (not mocks) — the integration IS the store + component together
- Mock only the outermost boundary (WebSocket, HTTP, filesystem)
- `waitFor` + `findBy*` for async state updates — never `sleep()`

---

## 7. E2E Tests (Puppeteer/CDP)

All E2E tests use Puppeteer with headless Chrome. Setup helper provides `launchApp()` / `teardownApp()` / `waitForWebSocket()`.

### E2E Test Infrastructure

| Utility | Purpose |
|---------|---------|
| `launchApp()` | Build → start Hono server → return Puppeteer `page` + `browser` |
| `teardownApp()` | Close browser, stop server, cleanup temp files |
| `waitForWebSocket()` | Poll `page.evaluate()` until WS connected |
| `setMobileViewport(page)` | Set 375×667, touch emulation, mobile UA |

### E2E Scenario Coverage Map

| Scenario | Steps | Constraints |
|----------|-------|-------------|
| **Session list load** | Navigate → wait for `[data-testid="session-list"]` → assert cards render | SDK-10 |
| **Create session** | Click new-session button → assert chat input appears | OPS-01 |
| **Send message + receive stream** | Type in input → press Enter → wait for `[data-testid="assistant-message"]` → assert content | ARC-03, SDK-11 |
| **Tool invocation display** | Send message triggering tool → wait for `[data-testid="tool-invocation"]` → assert tool name | — |
| **Concurrent tabs** | Open second tab → assert no errors → both tabs functional | OPS-10 |
| **Mobile responsive** | Set viewport 375×667 → verify Vaul drawer renders, touch targets ≥44px, `100dvh`, viewport meta | ARC-08, TCH-09, TCH-10, TCH-11, TCH-13 |
| **WebSocket resilience** | Force-close WS via `page.evaluate` → verify reconnect indicator → verify backoff timing → verify state sync | TCH-07, OPS-05 |
| **Security: nonce required** | Request without nonce → assert 401 response | SEC-02 |
| **Security: headers** | Check CSP header present and restrictive; CORS blocks non-localhost; Host header validated | SEC-03, SEC-05, SEC-06 |
| **Security: localhost only** | Assert server bound to 127.0.0.1 | SEC-01 |
| **Security: no client secrets** | Search client bundle for API keys / tokens → assert none found | SEC-11 |
| **Security: tunnel auth** | If tunnel active, assert auth required | SEC-12 |
| **PWA** | Assert `manifest.json` served and valid; service worker registered | TCH-18 |
| **Health check** | `GET /api/health` → assert 200 | OPS-06 |
| **Single deployable** | `npm start` launches complete app | ARC-05, OPS-01 |
| **cloudflared optional** | App functions without cloudflared installed | OPS-07 |
| **Session exclusivity** | Open same session in two contexts → assert mutex enforced | SDK-06, OPS-08 |
| **Async file ops** | Assert no synchronous file I/O in client | OPS-11 |

---

## 8. Performance Tests

### Benchmark Thresholds

| Metric | Threshold | Test Type | Constraints |
|--------|-----------|-----------|-------------|
| Initial page load (LCP) | < 2.0s | E2E trace | TCH-20 |
| Time to interactive (TTI) | < 3.0s | E2E trace | TCH-20 |
| First streaming chunk render | < 100ms | vitest.bench | TCH-03, TCH-22 |
| 1000-message scroll | > 30fps | E2E trace | TCH-02, TCH-20 |
| Diff computation (10K lines) | < 100ms | vitest.bench | TCH-04 |
| Markdown parse (100KB) | < 200ms | vitest.bench | TCH-05 |
| File tree build (1K files) | < 50ms | vitest.bench | TCH-06 |
| Memory (1000 messages) | < 100MB | E2E heap snapshot | TCH-20 |
| WebSocket reconnect | < 5s | E2E | TCH-07 |
| Bundle size (gzipped) | < 500KB | CI build output | TCH-20 |
| Streaming: 1000 chunk append+flush | < 50ms | vitest.bench | TCH-03 |

### Performance Test Guidelines

- **Benchmarks** (`vitest.bench`): run in isolation, 100+ iterations, report p50/p95/p99
- **E2E traces**: use Puppeteer `page.tracing.start()/stop()` for LCP/TTI/CLS
- **Memory**: `page.metrics()` for JS heap size after load
- **Regression detection**: CI compares benchmark results against `main` branch baseline; fail on >10% regression

---

## 9. Accessibility Tests

### Accessibility Coverage Map

| Test | What to Verify | Constraints |
|------|---------------|-------------|
| **axe-core audit: chat view** | Zero WCAG 2.1 AA violations | TCH-11 |
| **axe-core audit: session list** | Zero violations | TCH-11 |
| **axe-core audit: both themes** | Dark + Light pass independently | SCP-03, TCH-11 |
| **Keyboard: session list** | Arrow keys navigate, Enter selects | — |
| **Keyboard: command palette** | Ctrl+K opens, Escape closes, Tab cycles | — |
| **Keyboard: dialogs** | Escape dismisses, focus trapped | SEC-10 |
| **Touch targets** | All interactive elements ≥ 44×44px | TCH-11 |
| **Reduced motion** | Animations disabled when `prefers-reduced-motion: reduce` | TCH-12 |

### Required ARIA Attributes

| Component | Required | Notes |
|-----------|----------|-------|
| MessageList | `role="log"`, `aria-live="polite"` | Announces new messages |
| SessionList | `role="listbox"`, `aria-activedescendant` | Keyboard selection |
| SessionDrawer | `role="dialog"`, focus trap | Modal on mobile |
| PermissionCard | `role="alertdialog"`, `aria-label` | Security-critical |
| ThinkingBlock | `aria-expanded` | Collapsible |
| ToolInvocation | `role="status"`, `aria-live="polite"` | Status transitions |
| Code blocks | `role="region"`, `aria-label` (language) | Navigable landmark |
| Error alerts | `role="alert"`, `aria-live="assertive"` | Immediate |

---

## 10. Visual Regression Tests

Uses **pixelmatch** + **pngjs**. Baselines in `src/test/fixtures/screenshots/`. Update with `UPDATE_BASELINES=true npm run test:visual`.

| Scenario | Viewport | Theme | Threshold |
|----------|----------|-------|-----------|
| Chat view with messages | 1280×800 | Dark | 0.1% |
| Chat view with messages | 1280×800 | Light | 0.1% |
| Session list | 1280×800 | Dark | 0.1% |
| Mobile chat view | 375×667 | Dark | 0.5% |
| Mobile session drawer | 375×667 | Dark | 0.5% |
| Code block with highlighting | Component | Both | 0.1% |
| Approval dialog | Component | Both | 0.1% |

Visual tests validate ARC-09 (VS Code alignment) and SCP-03 (two themes).

### Visual Regression Workflow

1. **Baseline capture**: Run `UPDATE_BASELINES=true npm run test:visual` → screenshots saved to `src/test/fixtures/screenshots/`
2. **CI comparison**: Each PR run captures screenshots → pixelmatch compares to baseline → fail if diff > threshold
3. **Baseline update**: When intentional UI changes land, re-capture baselines in the same PR
4. **Review**: Screenshots diffs attached to PR as artifacts for human review

---

## 11. Constraint Verification Matrix

This is the **core deliverable** of the testing strategy. Every constraint has at least one test mapped.

### SDK Constraints (SDK-01 — SDK-12)

| ID | Constraint | Test Type(s) | Verified In |
|----|-----------|-------------|-------------|
| SDK-01 | CopilotClient single entry point | Unit | SDK client tests |
| SDK-02 | MCP server spawning on connect | Unit | SDK client tests |
| SDK-03 | events.jsonl read-only | Unit | useChatStore (no write methods) |
| SDK-04 | 30s idle timeout disconnect | Unit + E2E | SDK client + session lifecycle E2E |
| SDK-05 | Capabilities negotiation | Unit + Integration | SDK client + SDK→UI integration |
| SDK-06 | Per-session mutex | Unit + E2E | useSessionStore + session exclusivity E2E |
| SDK-07 | Lock file create/release | Unit | SDK client tests |
| SDK-08 | MCP blocking until handshake | Unit | SDK client tests |
| SDK-09 | Agent modes from capabilities | Integration | SDK→UI integration |
| SDK-10 | Session directory structure | Unit + E2E | useSessionStore + session lifecycle E2E |
| SDK-11 | EventEmitter streaming | Integration | WebSocket flow integration |
| SDK-12 | GitHub token authentication | Unit | SDK client tests |

### Architecture Constraints (ARC-01 — ARC-10)

| ID | Constraint | Test Type(s) | Verified In |
|----|-----------|-------------|-------------|
| ARC-01 | Single process | E2E | Session lifecycle E2E |
| ARC-02 | React as application shell | Component | App component tests |
| ARC-03 | WebSocket primary transport | Integration + E2E | WS flow integration + chat E2E |
| ARC-04 | REST for queries, WS for real-time | Integration | Store→UI + session navigation |
| ARC-05 | Single deployable artifact | E2E | `npm start` E2E |
| ARC-06 | MCP unix socket transport | Unit | MCP server unit tests |
| ARC-08 | Mobile-first responsive | E2E | Mobile responsive E2E |
| ARC-09 | VS Code visual alignment | Visual | Visual regression tests |
| ARC-10 | Lightweight provider interface | Unit | SDK client tests |

### Scope Constraints (SCP-01 — SCP-06)

| ID | Constraint | Test Type(s) | Verified In |
|----|-----------|-------------|-------------|
| SCP-01 | CLI sessions only | Unit | useSessionStore |
| SCP-02 | Local filesystem only | Unit | SDK client tests |
| SCP-03 | Exactly two themes | Component + Visual | App + useThemeStore + visual regression |
| SCP-04 | No LSP client | Static | Code review / CI grep |
| SCP-05 | No extension host | Static | Code review / CI grep |
| SCP-06 | No offline mode | Unit | SDK client (no service worker) |

### Security Constraints (SEC-01 — SEC-12)

| ID | Constraint | Test Type(s) | Verified In |
|----|-----------|-------------|-------------|
| SEC-01 | Localhost-only binding | E2E | Security E2E |
| SEC-02 | Nonce authentication | Unit + E2E | Security utils + security E2E |
| SEC-03 | Host header validation | Unit + E2E | Security utils + security E2E |
| SEC-04 | Unix socket permissions 0600 | Unit | MCP server tests |
| SEC-05 | CORS restricts origins | E2E | Security E2E |
| SEC-06 | Content Security Policy | Unit + E2E | CSP generator + security E2E |
| SEC-08 | JWT token validation | Unit | Security utils |
| SEC-09 | WebSocket auth token | Unit | useConnectionStore |
| SEC-10 | Sandboxed tool approval | Component | PermissionCard + ApprovalDialog |
| SEC-11 | No secrets in client bundle | Static + E2E | CI scan + security E2E |
| SEC-12 | Tunnel authentication | E2E | Security E2E |

### Technical Constraints (TCH-01 — TCH-22)

| ID | Constraint | Test Type(s) | Verified In |
|----|-----------|-------------|-------------|
| TCH-01 | CodeMirror renders without Shiki | Component | ChatCodeBlockContentPart |
| TCH-02 | Virtualized lists | Component + Perf | SessionList + MessageList + benchmarks |
| TCH-03 | Batched streaming renders | Unit + Perf | useStreamingStore + streaming bench |
| TCH-04 | Diff computation in worker | Unit + Perf | DiffWorker + diff bench |
| TCH-05 | Markdown parsing in worker | Unit + Component | MarkdownWorker + ChatMarkdownContentPart |
| TCH-06 | File tree in worker | Unit + Perf | TreeWorker + tree bench |
| TCH-07 | WS reconnection backoff (max 30s) | Unit + E2E | useConnectionStore + WS resilience E2E |
| TCH-08 | Session list pagination | Unit + Component | useSessionStore + SessionList |
| TCH-09 | Viewport meta tag | E2E | Mobile responsive E2E |
| TCH-10 | 100dvh main container | E2E | Mobile responsive E2E |
| TCH-11 | Touch targets ≥ 44px | E2E + A11y | Mobile E2E + axe audit |
| TCH-12 | Reduced motion respect | Component + A11y | ThinkingBlock + a11y tests |
| TCH-13 | Vaul drawer on mobile | Component + E2E | SessionDrawer + mobile E2E |
| TCH-14 | CodeMirror 6 for editable blocks | Component | ChatCodeBlockContentPart |
| TCH-15 | Shiki syntax highlighting | Component | ChatMarkdownContentPart |
| TCH-16 | Zustand for all state | Unit | All store tests |
| TCH-17 | No events.jsonl writes | Unit | useChatStore (no write methods) |
| TCH-18 | PWA manifest | E2E | PWA E2E |
| TCH-19 | Tailwind v4 CSS | Static | Build output analysis |
| TCH-20 | Degradation thresholds | Unit + Perf | useStreamingStore + all benchmarks |
| TCH-21 | Lazy loading heavy components | Component | App (dynamic imports) |
| TCH-22 | rAF flush for streaming | Unit | useStreamingStore |

### Operational Constraints (OPS-01 — OPS-12)

| ID | Constraint | Test Type(s) | Verified In |
|----|-----------|-------------|-------------|
| OPS-01 | Single `npm start` | E2E | Session lifecycle E2E |
| OPS-02 | Lock file cleanup on shutdown | Unit | SDK client tests |
| OPS-03 | Unix socket cleanup on shutdown | Unit | MCP server tests |
| OPS-04 | Stale lock detection | Unit | SDK client tests |
| OPS-05 | Graceful reconnect + backoff reset | Unit + E2E | useConnectionStore + WS resilience E2E |
| OPS-06 | Health check endpoint | E2E | `GET /api/health` E2E |
| OPS-07 | cloudflared optional | E2E | App without cloudflared E2E |
| OPS-08 | Session exclusivity | Unit + E2E | useSessionStore + session exclusivity E2E |
| OPS-09 | Ephemeral unique nonces | Unit | Nonce generation tests |
| OPS-10 | Concurrent tab support | E2E | Concurrent tabs E2E |
| OPS-11 | Async file operations | Unit + E2E | SDK client + async I/O E2E |
| OPS-12 | No auto-update mechanism | Static | Code review / CI grep |

### Coverage Summary

All **72 constraints** are mapped:
- **12 SDK** — 12 mapped (Unit + Integration + E2E)
- **10 ARC** — 9 mapped (ARC-07 N/A)
- **6 SCP** — 6 mapped (Unit + Static + Visual)
- **12 SEC** — 11 mapped (SEC-07 N/A; Unit + E2E + Static)
- **22 TCH** — 22 mapped (Unit + Component + Perf + E2E)
- **12 OPS** — 12 mapped (Unit + E2E + Static)

### Constraint-to-Test Traceability Rules

1. Every test file header comment lists which constraint IDs it verifies
2. Test names include constraint IDs in parentheses: `it('should batch renders (TCH-03, TCH-22)')`
3. CI coverage report maps files to constraint categories
4. Any new constraint MUST have a test before the PR merges
5. Removing a constraint's test requires explicit approval in the PR description

### Constraints Not Tested (with justification)

| ID | Constraint | Why Not Tested |
|----|-----------|----------------|
| ARC-07 | Reserved / future | Not yet defined |
| SEC-07 | Reserved / future | Not yet defined |

---

## 12. Mock & Fixture Catalog

### Mock Objects

| Mock | Location | Key Interface |
|------|----------|--------------|
| `MockCopilotClient` | `src/test/mocks/copilot-client.ts` | `connect`, `disconnect`, `sendMessage`, `listSessions`, `getCapabilities`, `on`, `off` |
| `MockMCPServer` | `src/test/mocks/mcp-server.ts` | `start`, `stop`, `handleRequest` |
| `createMockWebSocket()` | `src/test/helpers/ws-mock.ts` | `send`, `close`, `simulateMessage`, `simulateOpen`, `simulateClose`, `simulateError`, `getSentMessages` |
| `createMockWorker()` | `src/test/helpers/worker-mock.ts` | `postMessage`, `terminate`, `onmessage` |
| MSW handlers | `src/test/mocks/handlers.ts` | `server.use()` for per-test overrides |

### Fixture Files

| Fixture | Contents | Used By |
|---------|----------|---------|
| `sessions/empty-session.json` | Session with no messages | Creation tests |
| `sessions/active-session.json` | 5 messages, active status | Chat display tests |
| `sessions/multi-turn-session.json` | 50 messages + tool calls | Integration + perf |
| `events/streaming-chunks.json` | 100 content_delta events | Streaming + perf |
| `events/tool-invocations.json` | terminal, file_edit, web_search | ToolInvocation tests |

### Factory Functions (`src/test/helpers/factories.ts`)

| Factory | Returns | Default | Overrides |
|---------|---------|---------|-----------|
| `createSession(overrides?)` | `ISession` | Active, empty, auto ID | Any field |
| `createSessionSummary(overrides?)` | `ISessionSummary` | 5 messages, active | messageCount, status |
| `createStreamingChunk(overrides?)` | Content delta | Sequential IDs | text, type |
| `createToolCall(overrides?)` | Tool call object | Function type, JSON args | name, args |
| `createMessage(overrides?)` | Chat message | User role, auto timestamp | role, content |

### MSW Handler Patterns

| Endpoint | Default Behavior | Override Pattern |
|----------|-----------------|-----------------|
| `GET /api/sessions` | Return 3 sessions | `server.use(rest.get('/api/sessions', ...))` |
| `GET /api/sessions/:id` | Return active session | Override for error/empty states |
| `GET /api/health` | Return `{ status: "ok" }` | Override for unhealthy state |
| WebSocket `/ws` | Auto-connect, echo | `createMockWebSocket()` for fine control |

---

## 13. CI Pipeline & Coverage Gates

### Pipeline Stages

| Stage | Runs | Depends On |
|-------|------|-----------|
| **lint** | `npm run lint && npm run typecheck` | — |
| **unit-integration** | `vitest run --shard=N/4 --coverage` (×4 parallel) | lint |
| **e2e** | Build → start server → `vitest run --config vitest.e2e.config.ts` | lint |
| **performance** | `vitest bench` | unit-integration |
| **visual-regression** | Build → start server → visual regression tests | lint |

### Coverage Gates (enforced in CI)

| Metric | Minimum | Enforcement |
|--------|---------|-------------|
| Branch coverage | 80% | `vitest.config.ts` thresholds |
| Function coverage | 80% | `vitest.config.ts` thresholds |
| Line coverage | 85% | `vitest.config.ts` thresholds |
| Statement coverage | 85% | `vitest.config.ts` thresholds |
| Security tests | 100% pass | CI fails on any security test failure |
| E2E tests | 100% pass | CI fails on any E2E test failure |
| Performance benchmarks | No >10% regression | Comparison against main baseline |

### CI Failure Handling

| Failure Type | Action |
|-------------|--------|
| Lint / typecheck | Fix immediately — blocks all downstream stages |
| Unit test | Fix or mark `it.skip` with linked issue |
| Coverage below threshold | Add missing tests — no threshold lowering without team approval |
| E2E flake | Retry once (built-in); if still fails, investigate and fix |
| Visual diff | Review screenshot artifacts → update baseline or fix regression |
| Performance regression | Profile with `vitest bench --reporter=verbose` → optimize or accept with justification |

### npm Scripts

| Script | Command |
|--------|---------|
| `test` | `vitest run` |
| `test:watch` | `vitest watch` |
| `test:coverage` | `vitest run --coverage` |
| `test:e2e` | `vitest run --config vitest.e2e.config.ts` |
| `test:perf` | `vitest bench` |
| `test:a11y` | `vitest run src/test/accessibility/` |
| `test:visual` | `vitest run --config vitest.e2e.config.ts src/test/e2e/visual-regression.e2e.ts` |
| `test:all` | `npm run test && npm run test:e2e && npm run test:perf` |

---

## 14. TDD Implementation Workflow

Every phase follows red → green → refactor → verify.

| Phase | What to Build | Tests First | Tests After | Constraints Verified |
|-------|--------------|-------------|-------------|---------------------|
| 1. Scaffolding | Vite + React + Tailwind v4 | App renders, theme toggle | Build output | SCP-03, TCH-19 |
| 2. Data Layer | Zustand stores, Zod schemas | All store tests (§4), schema validation | Integration (§6) | TCH-16, SDK-03, SDK-10 |
| 3. SDK Integration | CopilotClient, MCP, WS | Client + wsStore tests (§4) | SDK→UI integration (§6) | SDK-01 to SDK-12, ARC-06 |
| 4. Core Components | MessageList, SessionList, etc. | Component tests (§5) | Visual baselines (§10) | ARC-02, ARC-04, TCH-02 |
| 5. Interactive Features | ToolInvocation, Approval, etc. | Component tests (§5) | A11y tests (§9) | SEC-10, TCH-11, TCH-12 |
| 6. Web Workers | Diff, markdown, tree workers | Worker tests + benchmarks (§4, §8) | Component integration | TCH-04, TCH-05, TCH-06 |
| 7. Security | Nonce, CSP, host validation | Security tests (§4) | E2E security (§7) | SEC-01 to SEC-12 |
| 8. Integration | Store → Component → WS wiring | Integration tests (§6) | Cross-component flows | ARC-03, SDK-11 |
| 9. E2E & Polish | Full workflows, mobile, perf | E2E (§7) + benchmarks (§8) | Visual regression (§10) | OPS-*, ARC-08 |

### TDD Checklist (per feature)

1. [ ] Write failing test that describes expected behavior
2. [ ] Implement minimum code to make test pass
3. [ ] Refactor — extract helpers, improve names
4. [ ] Run full suite — no regressions
5. [ ] Update constraint matrix if new constraints covered
6. [ ] Check coverage report — no new uncovered branches

---


## 15. AI-Agent-Driven Development Testing

> **2025-2026 Emerging Practice.** This section covers a complementary testing paradigm where an AI coding agent (e.g., Copilot CLI, Claude, Cursor) uses **Chrome DevTools MCP** to interactively validate the running webapp during development. This is NOT a replacement for automated tests — it's a development-time workflow that catches issues before they become test failures.

### 15.1 What Is AI-Agent-Driven Testing?

Traditional automated tests are **scripted** — a developer writes test code, CI runs it. AI-agent-driven testing is **interactive** — an AI agent connects to a real browser via Chrome DevTools Protocol (CDP), navigates the app, inspects DOM/CSS/network/console, takes screenshots, and provides real-time feedback.

**Key difference:** The AI agent *adapts* to what it sees. If a button renders in the wrong color, it notices. If a console error appears, it investigates. If a layout breaks at a specific viewport, it reports it. No pre-scripted assertions needed.

```
┌──────────────────┐    Chrome DevTools MCP     ┌──────────────┐
│  AI Coding Agent │ ◀────────────────────────▶ │  Chrome/Edge  │
│  (Copilot CLI)   │    take_snapshot()          │  (live app)   │
│                  │    take_screenshot()        │               │
│                  │    evaluate_script()        │               │
│                  │    click() / fill()         │               │
│                  │    list_console_messages()  │               │
│                  │    list_network_requests()  │               │
│                  │    performance_start_trace()│               │
│                  │    lighthouse_audit()       │               │
└──────────────────┘                            └──────────────┘
```

### 15.2 When to Use Each Approach

| Dimension | Automated Tests (Vitest/Puppeteer) | AI-Agent Testing (Chrome MCP) |
|-----------|-----------------------------------|-------------------------------|
| **When** | Every PR, CI pipeline | During development, before committing |
| **Who runs it** | CI server, developer via `npm test` | AI agent interactively |
| **Deterministic?** | Yes — same input → same output | No — agent adapts to what it sees |
| **Speed** | Fast (seconds to minutes) | Slower (agent thinks + acts) |
| **Coverage** | Pre-defined assertions | Exploratory, adaptive |
| **Best for** | Regression prevention | Discovery, visual validation, debugging |
| **Catches** | Known failure modes | Unknown/unexpected issues |
| **Cost** | CPU time | LLM tokens |

**Recommended workflow:**
1. Implement feature with AI agent assistance
2. AI agent validates via Chrome MCP (this section)
3. Write automated tests based on what was validated
4. CI runs automated tests on every PR

### 15.3 Chrome DevTools MCP Tool Reference

The Chrome DevTools MCP server exposes ~30 tools. These are the ones most relevant to development testing:

#### Inspection Tools

| Tool | Use Case | Example Prompt |
|------|----------|----------------|
| `take_snapshot` | Get accessibility tree (DOM structure with UIDs) | "Take a snapshot of the session list and verify ARIA roles" |
| `take_screenshot` | Visual validation, regression comparison | "Screenshot the chat view at 375px width" |
| `list_console_messages` | Catch runtime errors, React warnings | "Check for any console errors after loading a session" |
| `list_network_requests` | Verify API calls, WebSocket connections | "List all WebSocket frames sent during session load" |
| `get_network_request` | Inspect request/response bodies | "Show me the WebSocket upgrade request headers" |
| `evaluate_script` | Run assertions in the browser context | "Check if `useSessionStore.getState().sessions.length > 0`" |

#### Interaction Tools

| Tool | Use Case | Example Prompt |
|------|----------|----------------|
| `click` | Simulate user clicks | "Click the first session card in the list" |
| `fill` | Type into inputs | "Type 'fix the login bug' into the chat input" |
| `press_key` | Keyboard shortcuts, form submission | "Press Enter to send the message" |
| `hover` | Tooltip validation, hover states | "Hover over the tool invocation card" |
| `navigate_page` | Page navigation, reload testing | "Navigate to the app URL" |

#### Performance & Accessibility Tools

| Tool | Use Case | Example Prompt |
|------|----------|----------------|
| `lighthouse_audit` | Accessibility, SEO, best practices scores | "Run a Lighthouse accessibility audit" |
| `performance_start_trace` | Core Web Vitals, rendering performance | "Record a performance trace during message streaming" |
| `emulate` | Device simulation, dark mode | "Emulate iPhone 14 viewport (390×844, 3x, mobile, touch)" |
| `resize_page` | Responsive breakpoint testing | "Resize to 375px width and verify the session drawer" |

### 15.4 Development Testing Workflows

#### Workflow 1: Component Visual Validation

When implementing a new component (e.g., `PermissionCard`), use the AI agent to validate it matches the spec in [07-ui-specification.md](./07-ui-specification.md):

```
Developer prompt to AI agent:
"I just implemented PermissionCard. The app is running at http://localhost:3000.
Please:
1. Navigate to a session with a pending permission request
2. Take a screenshot of the permission card
3. Verify:
   - The card has role='alertdialog' and aria-label contains the tool name
   - Allow/Reject buttons meet 44×44pt touch target minimum (TCH-11)
   - Focus is on the Allow button by default
   - The card renders correctly at 375px mobile width
4. Test keyboard interaction: Tab between buttons, Enter to approve, Escape to reject
5. Check for any console errors or warnings"
```

The agent would then use:
- `navigate_page` → go to the app
- `take_snapshot` → verify ARIA attributes
- `take_screenshot` → visual check
- `resize_page` → test responsive behavior
- `press_key` → test keyboard navigation
- `list_console_messages` → check for errors

#### Workflow 2: Streaming Performance Validation

After implementing the streaming buffer (TCH-03, TCH-22), validate performance:

```
Developer prompt to AI agent:
"I implemented the streaming token buffer with rAF flush.
Please:
1. Open the app and start a new session
2. Send a message that will generate a long response
3. Start a performance trace during the response
4. Verify:
   - No per-token React re-renders (check React DevTools or rendering counts)
   - DOM updates only happen at rAF boundaries
   - Main thread stays responsive (no long tasks >50ms during streaming)
5. Take a screenshot of the streaming in progress
6. Report the trace results"
```

#### Workflow 3: WebSocket Reconnection Testing

Validate exponential backoff (TCH-07):

```
Developer prompt to AI agent:
"Test WebSocket reconnection behavior:
1. Open the app and connect to a session
2. Use evaluate_script to close the WebSocket: window.__ws.close()
3. Monitor console logs for reconnection attempts
4. Verify the backoff pattern: 1s → 2s → 4s → 8s → max 30s
5. Verify the UI shows a 'Reconnecting...' indicator
6. After reconnection, verify the session state is synced (OPS-05)"
```

#### Workflow 4: Cross-Browser/Device Validation

Test responsive design across breakpoints:

```
Developer prompt to AI agent:
"Test the session drawer across device sizes:
1. At 375px (mobile): Verify Vaul bottom sheet renders (TCH-13)
2. At 768px (tablet): Verify sidebar layout
3. At 1280px (desktop): Verify full panel layout
4. For each:
   - Take a screenshot
   - Verify touch targets ≥ 44×44pt (TCH-11)
   - Check viewport uses 100dvh (TCH-10)
   - Verify interactive-widget=resizes-content meta tag (TCH-09)"
```

#### Workflow 5: Accessibility Audit

Run comprehensive accessibility checks:

```
Developer prompt to AI agent:
"Run a full accessibility audit:
1. Navigate to the app
2. Run a Lighthouse accessibility audit
3. Take a snapshot and verify these ARIA requirements:
   - Message list: role='log', aria-live='polite'
   - Session list: role='listbox', aria-activedescendant for selection
   - Session drawer: role='dialog', focus trap active
   - Code blocks: role='region', aria-label includes language
   - Error alerts: role='alert', aria-live='assertive'
4. Test keyboard-only navigation through the entire app
5. Check prefers-reduced-motion is respected (TCH-12)
6. Report any violations"
```

#### Workflow 6: Security Validation

Verify security constraints interactively:

```
Developer prompt to AI agent:
"Verify security controls:
1. Try accessing the app without the nonce parameter → should get 401 (SEC-02)
2. Check response headers for CSP (SEC-08), CORS (SEC-05)
3. Inspect the Host header validation (SEC-03)
4. Verify no secrets are visible in client-side JavaScript (SEC-12):
   - Take a snapshot of the page source
   - Search for API keys, tokens, or credentials
5. Check WebSocket upgrade request includes auth (SEC-10)"
```

### 15.5 AI-Agent Test Prompts Library

Pre-built prompts the development team can use during implementation. Each maps to one or more constraints:

#### Session Management

```markdown
# Prompt: Session List Validation
"Navigate to {APP_URL}. Take a snapshot of the session list.
Verify: sessions are grouped by time (Today/Yesterday/Older),
each card shows title + status + file count,
clicking a card opens the session chat view.
Constraints: ARC-04 (REST for list), TCH-02 (virtualized if >50 items)"
```

```markdown
# Prompt: Session Ownership Check
"Open session {SESSION_ID} in the app. Run evaluate_script to check
if a lock file exists at ~/.copilot/ide/*.lock. Verify the lock file
contains the correct PID and socket path. Try opening the same session
in a second tab and verify exclusive ownership (OPS-08)."
```

#### Chat & Streaming

```markdown
# Prompt: Message Rendering Quality
"Open a session with existing chat history. Take screenshots of:
1. A user message with code blocks
2. An assistant response with markdown (headers, lists, code fences)
3. A thinking block (should be collapsible)
4. A tool invocation card (should show name, params, status)
Verify Shiki syntax highlighting is active (TCH-15),
CodeMirror is used for diffs (TCH-14), not Monaco."
```

```markdown
# Prompt: Streaming Fidelity
"Send a message and monitor the response stream. Verify:
1. Tokens appear progressively (not all at once)
2. Markdown renders incrementally (not re-parsed on each token)
3. No visual glitching during code block accumulation
4. The stop button is visible during streaming
5. Clicking stop halts the response"
```

#### Theme & PWA

```markdown
# Prompt: Theme Validation
"Test both themes:
1. Switch to Dark Modern → screenshot the full app
2. Switch to Light Modern → screenshot the full app
3. Verify CSS custom properties match VS Code's chatColors.ts tokens
4. Verify only these two themes exist (SCP-03)"
```

```markdown
# Prompt: PWA Installation
"Check PWA readiness:
1. Run Lighthouse audit → verify PWA score
2. Check for manifest.json (TCH-18)
3. Verify service worker registration
4. Test add-to-home-screen prompt on mobile viewport"
```

### 15.6 Integrating AI-Agent Testing into the Development Workflow

#### Development Cycle with AI-Agent Testing

```
┌─────────────────────────────────────────────────────────────────┐
│ Feature Development Cycle (with AI-Agent Testing)              │
│                                                                 │
│  1. Write failing automated tests (TDD - §14)                  │
│  2. Implement the feature                                       │
│  3. Run automated tests → fix until green                       │
│  4. ★ AI-Agent Validation (this section):                       │
│     a. Agent connects to running app via Chrome DevTools MCP    │
│     b. Agent runs relevant prompts from §15.5                   │
│     c. Agent reports visual issues, a11y violations, perf       │
│     d. Developer fixes issues agent found                       │
│  5. Write additional automated tests for issues found in (4)    │
│  6. Commit                                                      │
└─────────────────────────────────────────────────────────────────┘
```

#### Setting Up Chrome DevTools MCP for the Project

1. **Install the MCP server** (project-level or global):

```bash
npm install -g @anthropic-ai/chrome-devtools-mcp
# or configure in your AI agent's MCP settings
```

2. **Configure the AI agent** (e.g., in `.copilot/config.json` or similar):

```json
{
  "mcpServers": {
    "chrome-devtools": {
      "command": "npx",
      "args": ["@anthropic-ai/chrome-devtools-mcp"]
    }
  }
}
```

3. **Start the webapp** in development mode:

```bash
npm run dev  # Starts Vite dev server at http://localhost:5173
```

4. **Tell the agent** to connect and begin testing:

```
"Open http://localhost:5173?nonce={DEV_NONCE} in Chrome
and run the session list validation prompt"
```

#### When to Use AI-Agent Testing vs. Automated Tests

| Scenario | Use Automated Tests | Use AI-Agent Testing |
|----------|--------------------|--------------------|
| Regression on known behavior | ✅ | — |
| Visual polish / design review | — | ✅ |
| New component first-time validation | — | ✅ |
| Responsive layout across breakpoints | ✅ (Puppeteer snapshots) | ✅ (interactive exploration) |
| Accessibility audit | ✅ (axe-core in CI) | ✅ (Lighthouse + manual check) |
| Performance profiling | ✅ (benchmark thresholds) | ✅ (trace analysis) |
| Exploratory / ad-hoc testing | — | ✅ |
| CI/CD gate | ✅ | — |
| Debugging a specific user report | — | ✅ |

### 15.7 Recording AI-Agent Findings as Automated Tests

When the AI agent discovers an issue, convert it to an automated test to prevent regression:

```typescript
// Example: AI agent found that the session drawer doesn't trap focus
// Convert to automated test:

describe('SessionDrawer', () => {
  it('should trap focus when open [discovered by AI-agent testing]', async () => {
    const { getByRole } = render(<SessionDrawer isOpen={true} />);

    const dialog = getByRole('dialog');
    expect(dialog).toBeInTheDocument();

    // Tab should cycle within the drawer
    await userEvent.tab();
    expect(document.activeElement).toBeWithin(dialog);

    // Shift+Tab should also stay within
    await userEvent.tab({ shift: true });
    expect(document.activeElement).toBeWithin(dialog);
  });
});
```

**Convention:** Tests that originate from AI-agent findings include `[discovered by AI-agent testing]` in the test name for traceability.

---

> **End of document.**  
> This testing strategy ensures comprehensive verification of all 72 constraints defined in [08-constraints-and-requirements.md](./08-constraints-and-requirements.md). Every constraint maps to at least one automated test, and the TDD workflow ensures tests drive implementation from the start. Section 15 extends the strategy with AI-agent-driven development testing using Chrome DevTools MCP — the emerging 2025-2026 practice where AI agents interactively validate the running app during development, complementing traditional automated tests.
