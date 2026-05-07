---
name: sdlc-tests
description: Use this skill when implementation has produced code (or when an existing module / diff / feature scope) needs tests. Triggers include "write tests", "tests for FEAT-NNN", "tests for T-NNN", "tests for the auth helpers", "tests for this module", "tests for this diff", "add tests", "test this", "sdlc-tests", "after sdlc-impl". Reads the task file (when present), the diff (when present), the module surface, the Feature Spec ACs, the ADR(s) for technical constraints, sibling test files for codebase conventions, vitest/playwright config, and any MSW handlers already in the repo. Writes Vitest unit tests, React Testing Library component tests, and (only after explicit user confirmation in chat for the proposed happy path) Playwright e2e for critical flows. Mocking — MSW for HTTP, vi.mock for module-level mocks. Snapshot tests forbidden everywhere, even for pure functions. Coverage philosophy is acceptance-criteria-driven plus smart edge cases — one test per AC as baseline when a task file is present, plus targeted edge cases. Test files are colocated next to the code (foo.test.ts beside foo.ts). RTL is minimized via an explicit decision tree — components that are pure passthrough get no test, components whose only logic is one hook get a hook-level renderHook test instead of a component test, and only components with conditional rendering / forms / user interaction / derived state get a real RTL test. In Claude Code, delegates to the sdlc-test-writer subagent (Read + Write on test files only + Bash on test commands only — never npm install, never package.json edits, never prod-code edits) and runs the test suite — green = success summary; red = follow §6 failure protocol (stop on any doubt about whether the failure is a test bug or a code bug, never silently modify production code, never skip / weaken / delete failing tests to make them green). Re-test on the same scope = smart merge (delete tests for removed/renamed surface, keep tests still anchored, add tests for new surface, output a diff report). Does NOT write the spec (sdlc-ba), prototype (sdlc-design), ADR (sdlc-adr), task breakdown (sdlc-pm), production code (sdlc-impl), code review (sdlc-review), security audit (sdlc-security), or diagrams (sdlc-docs). Outputs test files only — no separate report artifact.
---

# sdlc-tests — Test Writer

This skill takes a **scope to test** — a task file, a module, a diff, or a named feature surface — and produces **executable tests** that anchor the behavior of that scope to something concrete (an Acceptance Criterion in a task, a documented contract in a module's exports, the intent of a diff).

The skill is intentionally narrow. It writes **tests only** — no production code, no documentation, no review, no audit. The output is test files at colocated paths next to the code under test, plus a short chat summary. There is no separate report artifact (per design). Tests are the artifact.

The skill **respects existing decisions**: a test that contradicts an ADR's chosen option, a Feature Spec NFR, or a sibling-test convention without acknowledging the contradiction is itself a test bug. Every test is anchored — to a task's Acceptance Criterion, a module export's documented contract, an edge case implied by a NFR, a defense pattern already in the codebase, or the explicit intent of a diff. Tests that "feel useful" but are not anchored to anything concrete do not get written.

In Claude Code, the skill delegates to the **`sdlc-test-writer` subagent** when available. The subagent runs with **Read + Write on test files only + Bash on a fixed test-command allowlist** (`npm test`, `npm run test*`, `npx vitest`, `npx playwright test`) — by design it cannot edit production code, edit `package.json`, install packages, run git, or run any non-test command. If the subagent file is missing, the skill performs the work inline and notes the fallback. In chat, the subagent is unavailable; the skill always works inline and **cannot run** the suite (it writes the tests as artifacts and notes that running is the user's responsibility).

The skill **does not modify production code under any circumstance**. Not to make a test pass. Not to "fix a small thing while we're here". Not even when the failure is obviously a one-character typo in production. If a test fails and the failure is on the production side, the skill **stops and reports** — the next move is the user's, typically by re-invoking `sdlc-impl`. This is a structural rule, not a stylistic preference, and it shows up explicitly in the failure protocol (§6) and in the subagent's tool allowlist (§7).

The skill **never writes snapshot tests**. Not `toMatchSnapshot`. Not `toMatchInlineSnapshot`. Not even for pure functions. Pure-function output gets explicit assertions (`expect(format(...)).toBe('expected literal')`) so the test reads as a contract, not as a tripwire. This is non-negotiable and is enforced both in the skill body (§4) and in the subagent prompt (§7).

The skill **minimizes RTL component tests** by routing component-level work through an explicit decision tree (§4). Components that are pure passthrough (`<Layout>`, `<Icon>`, `<Heading>`) get no test. Components whose only logic is a single hook get a `renderHook` test of that hook, not an RTL test of the shell. Only components with conditional rendering, forms, interaction, or derived state get a real RTL test. The decision tree is applied **before** writing any RTL test — and it is enforced by the failure protocol (§6.4): an RTL test that is later judged to be on a bucket-1 or bucket-2 component is itself a test bug.

---

## 1. Trigger

Activate when the user signals any of:

- An explicit test request: "write tests", "add tests", "test this", "tests for this", "tests for the auth helpers", "tests for FEAT-003", "tests for T-007", "sdlc-tests".
- A continuation in the workflow: "now write tests for what `sdlc-impl` produced", "write tests for the diff", "tests for the changes I just made".
- A scoped test request: "tests for `src/auth/session.ts`", "tests only for the validators module", "test the new endpoints".
- A re-test request: "rerun tests on T-007", "update tests after the refactor", "regenerate tests for the login feature".

Do **not** activate when the user asks for product requirements (route to `sdlc-ba`), UI prototypes (`sdlc-design`), architectural decisions (`sdlc-adr`), task breakdown (`sdlc-pm`), code writing (`sdlc-impl`), correctness/design review (`sdlc-review`), security audit (`sdlc-security`), or diagrams (`sdlc-docs`).

If the request is ambiguous between "write tests" and "review tests", default to writing — and note the assumption. If the user says "review my tests for issues", defer to `sdlc-review` instead.

If the user asks for tests with **no scope** (no task, no module, no diff, no feature folder), the skill does **not** assume the most recent feature. It asks once (§2 — missing-input handling) — every test must be anchored to a concrete surface, and silent assumption defeats the anchoring.

---

## 2. Inputs / prerequisites

The skill is built around one **required input** — a **scope to test** — and one set of context that is read but not required.

### 2.1. Required: scope

Exactly one of the following must be available:

| Scope kind | What it looks like | What it anchors |
|---|---|---|
| **Task** | `docs/features/FEAT-NNN-<slug>/04-tasks/T-NNN-<slug>.md` | Tests are anchored to the task's Acceptance Criteria. One test per AC as baseline (§4.2). |
| **Module / file path** | An explicit path or glob (`src/auth/session.ts`, `src/validators/*.ts`) | Tests are anchored to the module's public surface — exported functions, classes, hooks, components — and their documented contracts (JSDoc, type signatures, names). |
| **Diff** | `git diff <base>..<head>` (Claude Code) or pasted unified diff (chat) | Tests are anchored to the changed lines — what behavior the diff intends to add, modify, or remove. |
| **Feature scope (no diff)** | A named `FEAT-NNN-<slug>` on legacy / pre-workflow code | Tests are anchored to the union of `Files-to-Touch` from `04-tasks/*.md` and any user-provided file list. The Feature Spec ACs are used when available; otherwise the surface anchor (per the Module rule) applies. |

The task scope is the strongest anchor (explicit ACs, explicit DoD, explicit Test Plan). The module / diff / feature scopes are progressively weaker — they require the skill to derive the anchor from the code itself.

### 2.2. Existing-context — read, scanned, never silently overridden

The skill scans the surrounding context before writing tests:

| Source | Where (Claude Code) | What to extract |
|---|---|---|
| Task file (when scope is task) | `docs/features/FEAT-NNN-<slug>/04-tasks/T-NNN-<slug>.md` | Acceptance Criteria (1:1 to tests, §4.2), Files to Touch, Test Plan section, Definition of Done, Notes. |
| Diff (when scope is diff or task with diff) | `git diff <base>..<head>` | Exact added / removed / modified lines. Tests should exercise the new branches and any modified code paths. |
| Module surface (when scope is module/feature) | Source file(s) | Exported functions, classes, hooks, components; JSDoc/TSDoc; type signatures; constants used as contracts. |
| Feature Spec | `docs/features/FEAT-NNN-<slug>/01-feature-spec.md` | NFRs (a11y, perf, security implications) — these inform edge cases, not direct tests, but a NFR like "the form must handle slow networks" → a debounce-waitFor edge case. |
| ADR(s) | `docs/features/FEAT-NNN-<slug>/03-adr*.md` | Chosen technical options (validation lib, ORM, error model). Tests use the chosen options' idioms. Known limitations from the ADR are *expected* untested gaps — note them, don't fabricate tests against rejected options. |
| Sibling test files | `**/*.test.ts(x)`, `**/*.spec.ts(x)`, `e2e/**/*.spec.ts(x)` | Naming conventions, factory style (separate `factories.ts` vs inline), mocking patterns (existing MSW handlers vs `vi.mock`), test-utility helpers, custom render functions. The skill **uses** what's there rather than inventing parallel idioms. |
| Test runner config | `vitest.config.ts`, `playwright.config.ts`, `vite.config.ts` (when test config is co-located), `jest.config.*` if present | Detected runner; existing setup files (`setup.ts`, `setupTests.ts`); jsdom vs node env per file pattern; path aliases (`@/*`); coverage exclusions. The skill writes against **the runner that is configured**, not a runner it would prefer. |
| MSW handlers | `src/mocks/handlers.ts`, `src/test/handlers/*.ts`, or whatever the repo uses | Existing API mocks. The skill extends these rather than creating a parallel mock layer. |
| Codebase shape | `package.json`, `tsconfig.json`, `src/`, validation lib, error helpers, auth utilities | Same context every other skill scans (consistency with §3.6). |
| Prior tests for this scope | The colocated test file(s) already on disk | Re-test mode (§5.6 — smart merge): keep what's still anchored, delete what's no longer anchored, add what's new. |

From the scan, build a short internal "test baseline" before writing the first test:

- Scope kind: `task | module | diff | feature`.
- Scope identity: `T-NNN | path/glob | diff range | FEAT-NNN`.
- Anchors detected: `<N ACs | M public exports | K changed lines | etc>`.
- Test runner: `vitest` (always — Vitest is the SDLC default).
- DOM env in use for jsdom files: `jsdom | happy-dom`.
- Component framework: `react` (always — React is the project default).
- Mocking primitives detected: `MSW handlers at <path> | vi.mock idioms in <files> | none`.
- Test file convention: `colocated (*.test.ts beside *.ts)` (always — confirmed by the user; the skill does NOT switch to `__tests__/` even if a stray legacy file exists).
- Test-utility helpers detected: `<list of renderWithProviders, setupUser, factories, etc.>`.
- Existing tests for this scope: `<N files | none>` → drives re-test smart merge (§5.6).

### 2.3. Missing-input handling

Each missing input has a single ask. The skill never silently invents the input.

- **No scope at all** (no task, no module, no diff, no feature) → ask once:
  > I need a scope to test. Pick one:
  > - **Task** — name the task file (e.g. `T-007` in `FEAT-003-user-auth`).
  > - **Module / file** — give me a path (e.g. `src/auth/session.ts`) or a glob (e.g. `src/validators/*.ts`).
  > - **Diff** — let me diff the working tree against `main`, or paste a unified diff.
  > - **Feature** — name a `FEAT-NNN-<slug>` and I'll cover the union of its Files-to-Touch.
- **No task file at the named path** but the user said "task" → ask once:
  > I don't see a task file at `04-tasks/T-007-<slug>.md`. Options: (a) point me at the right path, (b) drop the task framing — name a module / diff / feature instead, (c) run `sdlc-pm` first to create the task.
- **No Acceptance Criteria in the task file** but task scope is selected → note in the test baseline that AC-driven mapping is degraded and ask once:
  > The task file exists but has no Acceptance Criteria section. Options: (a) paste the ACs in chat, (b) re-run `sdlc-pm` to fix the task, (c) proceed with module-surface anchoring instead of AC anchoring (one test per public export plus edge cases) — flagged in the test plan.
- **No Feature Spec / ADR** when scope is task or feature → don't ask. Just note in the test baseline that NFR-driven edge cases and ADR-idiom alignment cannot be checked. Tests still get written from the task / module / diff anchor.
- **No test runner config** (`vitest.config.ts` absent, `package.json` has no `test` script) → ask once:
  > I don't see a test runner configured. The project default is Vitest. Options: (a) confirm Vitest and I'll write tests against the default config (you'll need to add the config + scripts yourself — I won't edit `package.json`), (b) point me at the existing runner if there is one, (c) abort.
  > Note that even with Vitest confirmed, I cannot install packages or edit `package.json` — that's on you.
- **No `playwright.config.ts`** but the proposed test plan includes e2e → don't fail. Drop e2e from the plan and note it in the chat summary:
  > Playwright config not detected — dropping the proposed e2e for `<flow>`. To enable e2e on this scope, add `playwright.config.ts` (and Playwright in `devDependencies`) and re-run.
- **Diff is empty** (the working tree is clean against the named base) → stop and report:
  > The diff is empty against `<base>`. There is nothing to test from the diff. Pick a different scope (task / module / feature) or change the base.
- **Diff contains only test files** → switch to module scope, narrow to the modules the test files target, and proceed. Note the switch in the chat summary.
- **Diff contains only docs / config / non-source** → stop and report. There's nothing to test.

The skill does not silently widen the scope. "Test the auth feature" with an empty `04-tasks/` does not become "test everything in `src/auth/`" without an explicit ask.

---

## 3. Environment detection

Run this check at the start of every invocation:

1. Can you read files via the Read/Glob/Grep tools, **and** does the working directory contain `docs/features/` (or `package.json` / `.git` / `src/`), **and** can you write files, **and** can you run shell commands, **and** can a subagent be invoked?
   → **Claude Code mode.** Delegate to the `sdlc-test-writer` subagent if it exists; otherwise inline. Test files are written to disk at colocated paths next to the code under test. The suite is run after writing.
2. Files are readable / writable but no project markers, **or** no subagent infrastructure, **or** no Bash:
   → **Claude Code, no-subagent fallback** (when subagent missing) or **Claude Code, no-bash mode** (when Bash is missing). Tests are written to disk at colocated paths. If Bash is unavailable, the suite is **not** run; the chat summary notes this and instructs the user to run the suite manually.
3. No tool execution available, only conversation:
   → **Chat mode.** The skill writes tests **as inline artifacts** (one artifact per test file), names them with the colocated path they would have on disk (`src/auth/session.test.ts`), and explicitly notes that running is the user's responsibility.

If the mode is ambiguous (tools present, project markers present, subagent presence unclear), proceed in Claude Code mode and probe for the subagent in §5.2. If the probe fails, fall back without re-asking the user.

---

## 4. Test types, layering, and the RTL decision tree

This is the substantive part of the skill. It says **what kind of tests to write**, **at which layer**, and — critically — **when not to write a test**.

### 4.1. Three layers, one runner default

The project uses three test types. Each has a clear when-to-use:

- **Vitest unit tests** — pure logic, utilities, services, validators, formatters, hooks, server-side handlers' business logic in isolation. Fast (sub-second), no DOM needed (or `jsdom` for hooks that touch `window`).
- **React Testing Library (RTL) component tests** — components with real behavior (conditional rendering, forms, user interaction, derived state). Render the component, drive it via `userEvent`, assert against accessible queries (`getByRole` first). RTL tests run on Vitest with `jsdom` / `happy-dom`.
- **Playwright e2e tests** — critical user flows end-to-end, exercising the whole stack against a running app. Only for happy paths of business-critical journeys (auth, signup, password reset, checkout, payment, the headline feature of the product). Not for component permutations.

There is no separate "integration test" tier. Integration-style coverage is achieved by RTL tests with MSW handlers — RTL renders the real component tree, MSW intercepts the HTTP call, the component sees a realistic response. That's the integration story.

### 4.2. Coverage philosophy — AC-driven plus smart edge cases

Coverage is **not** measured by line / branch percentage. The skill does not run `c8` or `v8` coverage and does not chase a number. Instead:

1. **AC mapping (when task scope).** Every Acceptance Criterion in the task file gets at least one test. The test traces back to the AC explicitly — a leading comment like `// AC#3: returns 401 when the session token is expired`, or the AC paraphrased in the `it(...)` string.
2. **Surface mapping (when module / feature scope without ACs).** Every public export gets at least one test for its happy path and one test per documented edge case (from JSDoc, type unions, named constants used as the contract).
3. **Diff mapping (when diff scope without ACs).** Every changed branch in the diff — every new conditional, every modified return path, every new error case — gets a test. Pure cosmetic refactors get no new test (but if a sibling test exists and the refactor changed its meaning, the existing test is updated).
4. **Edge cases on top.** For each anchor (AC / export / diff branch), the skill adds targeted edge cases the anchor implies but does not state explicitly: empty / null / undefined input, off-by-one, boundary values, time / timezone, locale, race condition where applicable, the realistic failure mode of the dependency (network error, DB unique-violation, validation rejection).
5. **Explicit non-coverage.** Things the skill deliberately does not test: trivial getters, framework-provided behavior (React's render, Vitest's own assertions), code that is itself a test helper, code that is documented in the ADR as a known untested limitation.

The chat summary at the end (§8) shows the AC ↔ test mapping table so the user can see anchoring at a glance.

### 4.3. Style rules — applied to every test

These rules are non-negotiable. They are enforced both in this skill body and in the subagent prompt (§7).

- **AAA structure.** Every `it(...)` block has visually obvious Arrange / Act / Assert sections, separated by blank lines. Long Arrange goes into a helper or a factory. Multi-Act tests are a smell — split them.
- **Explicit assertions only.** No snapshot tests anywhere. No `toMatchSnapshot`, no `toMatchInlineSnapshot`. Pure-function output gets `expect(result).toBe('exact literal')` or `expect(result).toEqual({ ... })`.
- **Accessible queries first (RTL).** `getByRole` → `getByLabelText` → `getByPlaceholderText` → `getByText` → `getByDisplayValue` → `getByTestId` (last resort, only when the DOM offers no semantic anchor and refactoring it is out of scope). `getByTestId` in a new test is a smell that needs justification in the test comment.
- **`userEvent` over `fireEvent` (RTL).** `userEvent.setup()` per test, drive interactions through it. `fireEvent` is reserved for events `userEvent` does not model (specific synthetic events).
- **MSW for HTTP.** Tests that exercise components / hooks / services that make HTTP calls go through MSW handlers, not `vi.mock('axios')` or `global.fetch = vi.fn()`. New handlers are added to the existing MSW handler module when one exists, otherwise to a colocated `*.test.ts` `setupServer(...)`.
- **`vi.mock` for module mocks.** Modules without a network surface (utilities, third-party SDKs, environment helpers) are mocked with `vi.mock('./module', ...)`. The mocked surface mirrors the real one — same export names, same call signatures.
- **No timers without `vi.useFakeTimers()`.** Any test that depends on `setTimeout` / `setInterval` / `Date.now()` uses fake timers, advances them explicitly, and restores them in `afterEach`.
- **No tests sleep on real time.** No `await new Promise(r => setTimeout(r, 100))` to "wait for the thing". Use `await waitFor(() => expect(...))`.
- **Factories over inline literals (when factories exist).** If the codebase already has a `factories.ts` for a domain entity, use it. If it doesn't, inline literals are fine — the skill does not introduce a factories layer on its own without an explicit user ask.
- **One test, one behavior.** Each `it(...)` asserts one behavior. "And also" in a test name is a smell.
- **AC traceability comment.** When the scope is a task, every test starts with a one-line comment naming the AC (`// AC#2: rejects passwords shorter than 12 chars`) so re-test smart merge can re-anchor mechanically.

### 4.4. The RTL decision tree (the most important part of this section)

Before writing any RTL component test, the skill walks this tree. The result is one of four outcomes — and three of them are "do not write an RTL test".

```
Component under consideration has:
├─ Bucket 1 — Pure passthrough / composition only
│   - <Layout>{children}</Layout>
│   - <Card><Title /><Body /></Card>
│   - <Heading level={2}>{children}</Heading>
│   - <Icon name="check" size="sm" />
│   - <Spacer />
│   - <Button {...props} /> that just spreads to the underlying button
│   →  NO TEST. The component has nothing to break that types/lint don't catch.
│       If a regression here matters, it is a visual concern → covered by Storybook /
│       visual regression / a11y audit, not Vitest+RTL.
│
├─ Bucket 2 — One hook is the whole logic, JSX is a thin shell
│   - const Badge = ({ id }) => { const u = useUser(id); return <span>{u?.name ?? '…'}</span>; }
│   - const ThemeIndicator = () => { const t = useTheme(); return <span>{t}</span>; }
│   →  TEST THE HOOK directly with renderHook(() => useUser(id), { wrapper }).
│       Cover the hook's states (loading / success / error) and edge inputs.
│       The component itself gets NO RTL test — its rendering is trivially derived
│       from the hook's return value, and any bug there is a hook bug or a typo
│       caught by TypeScript.
│
├─ Bucket 3 — Real component logic
│   - <LoginForm> with validation, submit, error display, disabled-submit-while-pending.
│   - <UserTable> with filter, sort, pagination, empty state.
│   - <Modal> with open/close, focus trap, ESC-to-close, click-outside.
│   - <Stepper> with step state, next/prev, terminal step behavior.
│   - <RichTextEditor> with selection, formatting toggles.
│   - Anything with conditional rendering driven by user interaction or local state
│     beyond a single hook return.
│   →  RTL TEST. Render the component, drive it with userEvent, assert with
│       getByRole-first. Use MSW for any HTTP call the component triggers.
│
└─ Bucket 4 — Composed page / feature with multiple hooks + side effects
    - <CheckoutPage> orchestrating cart hook + address hook + payment hook + redirect.
    - <DashboardLayout> with auth guard + data hooks + feature-flagged sections.
    - <SearchExperience> with debounced query + URL sync + result list + filters.
    →  RTL TEST AS INTEGRATION. Render with the real provider tree (use the project's
       `renderWithProviders` helper if it exists, otherwise reproduce it in the test
       file). Use MSW for all HTTP. The test is shaped like a small flow:
       "user lands on the page → types → sees results → applies a filter → sees
       fewer results". One per critical journey through the page. Edge cases stay
       in lower-layer tests (the hooks) so the page-level test stays readable.
```

**Bucket lookup is mandatory before writing any RTL test.** The test plan (§5.3) shows the bucket and the rationale for every component-level entry. If a component lands in bucket 1 or bucket 2 and an existing test is found for it, the test is flagged for **deletion** in the smart-merge diff (§5.6) — these tests are not just unnecessary, they are net-negative (they slow CI, fail on cosmetic refactors, train the team to chase tests rather than read them).

**Bucket 2 example, in full** — to make this concrete:

```tsx
// src/users/UserBadge.tsx
export const UserBadge = ({ id }: { id: string }) => {
  const { data: user, isLoading } = useUser(id);
  if (isLoading) return <span>Loading…</span>;
  return <span>{user?.name ?? 'Unknown'}</span>;
};
```

**Wrong** (an RTL test on `<UserBadge>`): renders the component, mocks the hook with `vi.mock('./useUser')`, asserts `screen.getByText('Alice')`. This is bucket-2 — the hook is the logic, the JSX is shell.

**Right** (a `renderHook` test on `useUser`):

```ts
// src/users/useUser.test.ts
import { renderHook, waitFor } from '@testing-library/react';
import { useUser } from './useUser';
import { withProviders } from '../test/withProviders';

describe('useUser', () => {
  it('returns the user for a known id', async () => {
    const { result } = renderHook(() => useUser('u-1'), { wrapper: withProviders });
    await waitFor(() => expect(result.current.isLoading).toBe(false));
    expect(result.current.data?.name).toBe('Alice');
  });

  it('returns undefined data and an error for an unknown id', async () => {
    // MSW handler for /users/u-404 returns 404 in the test setup.
    const { result } = renderHook(() => useUser('u-404'), { wrapper: withProviders });
    await waitFor(() => expect(result.current.isLoading).toBe(false));
    expect(result.current.data).toBeUndefined();
    expect(result.current.error).toBeDefined();
  });
});
```

The component needs no test of its own. Its rendering is "loading → 'Loading…'", "success → name", "no data → 'Unknown'" — all three derive trivially from the hook's three states, which the hook test already covers. A typo in the JSX (`'Loading…'` vs `'Loading...'`) is caught by TypeScript / lint / visual review, not by RTL.

### 4.5. Mocking — when MSW, when vi.mock, when neither

- **MSW** for any test that exercises code which makes HTTP calls (`fetch`, `axios`, the project's API client). Handlers go in the existing module (when present) or in a colocated test setup. MSW intercepts at the network layer — the code under test does not know it is mocked.
- **vi.mock** for module-level mocks of things without a network surface: utility modules, third-party SDKs (`@stripe/stripe-js`), environment helpers (`./env`), feature-flag clients, telemetry. The mocked module exports the same names as the real one.
- **Neither** for pure functions and pure logic — they get tested against real inputs. No mocking, no setup, no teardown.

The skill never:

- Mocks `console.log` / `console.error` to silence test noise. If a test expects an error to be logged, it asserts on the spy. If logs are too noisy, that is a setup concern, not a per-test mock.
- Mocks `Date` directly; uses `vi.setSystemTime(...)` with `vi.useFakeTimers()`.
- Mocks the module under test. The module under test is the thing being tested — mocking it defeats the test.
- Globally replaces `fetch` — that is MSW's job.
- Mocks React (`vi.mock('react', ...)`) — full stop.

### 4.6. Playwright — only with explicit confirmation

E2E tests are gated by an explicit user confirmation step in the chat (§5.4). The skill never writes Playwright tests "because the task mentioned e2e" without that confirmation. The reason is cost: e2e tests are slow, brittle in CI, and have a maintenance cost orders of magnitude higher than unit / RTL — writing them speculatively is a tax on the user.

When the skill detects a candidate for e2e:

- The proposed flow is a **happy path** of a critical user journey (auth, signup, password reset, checkout, payment, the headline feature).
- It crosses **multiple pages or screens**.
- It exercises a **real DB write** or a **security boundary** that unit tests cannot fully cover.

…the skill includes the candidate in the test plan with a `(proposed e2e)` marker and asks for explicit confirmation in chat (§5.4). One confirmation per flow. If the user declines, the skill drops the e2e from the plan and proceeds with unit + RTL only.

E2E tests, when written, follow:

- One file per flow, named after the flow (`e2e/login.spec.ts`, `e2e/password-reset.spec.ts`).
- One `test(...)` per scenario; no nested `describe` for permutations.
- Selectors prefer `getByRole` / `getByLabel` / `getByText`; `data-testid` only when the DOM offers no anchor (same hierarchy as RTL).
- Network: real, against the dev server / staging the project's `playwright.config.ts` already targets. The skill does not introduce an MSW-like layer for Playwright — that defeats the purpose of e2e.
- Setup / teardown: the skill uses Playwright's `beforeEach` for per-test state and assumes the project provides a way to seed users / data (most projects do via a script or fixture). If no seeding mechanism exists, the skill notes this as a limitation in the chat summary and does not invent one.

---

## 5. Execution

The same logical flow runs in all environments. Steps 1–3 are identical; Step 4 (subagent vs inline) branches; Step 5 (delivery channel) differs by environment; Step 6 (run the suite) is Claude Code only.

### Step 1 — Identify scope, load context, scan the codebase

1. Determine **scope** (per §2.1):
   - Named explicitly ("tests for T-007", "tests for `src/auth/session.ts`", "tests for the diff against main") → use that.
   - Inferred from a recent `sdlc-impl` invocation in the conversation → use the same task / module the impl touched, but state the inference in the test baseline so the user can override.
   - Otherwise ask once (§2.3).
2. Load the **task file** (when scope is task) and parse the Acceptance Criteria, Files-to-Touch, Test Plan, DoD. Note any AC the impl explicitly deferred.
3. Compute the **diff** (when relevant): `git diff <base>..<head>` against the configured base; default base is the merge-base with `main`. Print the chosen base in the chat summary.
4. Read the Feature Spec NFR section (when present) and pull edge-case implications for each NFR (a slow-network NFR → a `waitFor` test with a delayed MSW handler; an a11y NFR → an RTL test that queries by role and asserts focus management).
5. Read the ADR(s) and note the chosen options (validation lib, error model, ORM). Tests use these options' idioms.
6. Scan sibling test files for **conventions**: factories, helpers, custom render, MSW handler module path, AAA style. Adopt what's there.
7. Detect **runner config**: Vitest config, Playwright config, setup files, path aliases.
8. Detect **existing tests for this scope** for the re-test smart merge (§5.6).

The output of Step 1 is the internal "test baseline" (§2.2) — a short header that sits at the top of the chat summary so the user can see what the skill saw before writing.

### Step 2 — Decide subagent vs inline

Decision matrix:

| Signal | Outcome |
|---|---|
| Claude Code mode + `sdlc-test-writer` subagent is available    | **Delegate to subagent.** |
| Claude Code mode + subagent file missing | **Inline work.** Note in the chat summary: `Test writer: inline (subagent not found)`. |
| Chat mode | **Inline work.** Subagents are unavailable. Tests are returned as inline artifacts. |
| User explicitly asks for inline ("just write them yourself") | **Inline work.** Honor the override. |

Probe whether the `sdlc-test-writer` subagent is available. Claude Code resolves agents from `.claude/agents/` (project-local) or `~/.claude/agents/` (user-level) — either is fine; the skill does not care which. If the agent cannot be resolved, fall back to inline. Do not invoke a non-existent agent.

### Step 3 — Build the test plan (gating step)

Before writing any test, the skill builds and shows a **test plan** as a markdown table. The plan is the contract between the skill and the user; the user can object before the skill spends tokens on full test bodies.

The plan columns:

| Column | What it holds |
|---|---|
| `Anchor` | The AC ID (`AC#3`), the export name (`createSession()`), or the diff hunk (`session.ts:42-58`). |
| `Scope unit under test` | The function / hook / component / endpoint being tested. |
| `Layer & rationale` | One of `Vitest unit` / `Vitest hook` / `RTL (bucket 3)` / `RTL (bucket 4 — integration)` / `Playwright e2e (proposed)` — plus a one-line rationale (e.g. "form has validation + submit + error states → bucket 3"). For components, the bucket number from §4.4 is mandatory. For components that landed in bucket 1 / bucket 2, they appear in the plan with `→ no test` or `→ hook test on <hookName> instead` — making the omission explicit. |
| `File path` | The colocated test file path (`src/auth/session.test.ts`, `src/users/useUser.test.ts`, `e2e/login.spec.ts`). |
| `Cases` | A short bullet list of the test cases the file will contain. Each case is one `it(...)` call. |

Below the table, the plan also lists:

- **Components routed away from RTL** — explicit list of components that landed in bucket 1 (no test) or bucket 2 (hook test instead), with a one-line bucket rationale. This is what makes "RTL minimization" visible to the user instead of invisible.
- **Edge cases the skill will add** — a short bulleted list of edge-case categories per anchor (boundary, null/undefined, timezone, network failure, etc.).
- **Existing tests on this scope** — count and names (re-test smart merge preview, §5.6).
- **Proposed e2e** (when relevant) — the candidate flow(s) with happy-path description and rationale, awaiting confirmation in Step 4.

### Step 4 — Confirm e2e (only when proposed)

If the test plan contains any `Playwright e2e (proposed)` entries, the skill stops and asks the user one question per flow:

> Proposed e2e for **`<flow-name>`**:
>
> Happy path:
> 1. User opens `/login`.
> 2. Enters valid credentials.
> 3. Submits.
> 4. Lands on `/dashboard`.
> 5. Sees their name in the header.
>
> Rationale: this is a critical security boundary (auth) and unit + RTL cannot exercise the real cookie/session round-trip.
>
> Confirm e2e for this flow? (yes / no / change scope)

If the user confirms — keep in the plan. If the user declines — drop, note dropped in the chat summary. If the user adjusts ("only test the happy path up to step 3") — narrow the test and continue.

The skill does not ask about e2e for non-proposed flows. It does not re-ask on subsequent runs unless the plan changes.

### Step 5 — Write the tests

Write tests **one file at a time**, in this order:

1. **Vitest unit tests** (pure logic, services, validators) — fastest, simplest, lowest risk of cascading rework.
2. **Vitest hook tests** (`renderHook` for bucket-2-displaced hooks and standalone hooks).
3. **RTL component tests** (bucket 3, then bucket 4).
4. **Playwright e2e tests** (only the confirmed flows from Step 4).

Within each file:

- Start with imports — keep them sorted (third-party first, project absolute imports, project relative imports).
- Add a top-level `describe(<unit name>, () => { ... })` per export under test. Nested `describe` is reserved for grouping cases by behavior dimension (e.g. `describe('when token is expired', ...)`), not for "permutations of props".
- Each `it(...)` has the AC traceability comment (when scope is task), a clear sentence (`it('returns 401 when the session token is expired', ...)`).
- AAA spacing in the body. No multi-Act tests.
- For RTL: render with the project's `renderWithProviders` if it exists; otherwise reproduce only the providers the component actually needs.
- For MSW: extend the existing handler module when present (`src/mocks/handlers.ts`), or set up `setupServer(...)` colocated. Existing handlers are reused; do not redefine the same endpoint twice.
- For Playwright: one file per flow, one `test(...)` per scenario, no nested `describe`.

Do not write more than one test file before reading what's already in `package.json` to confirm test scripts. The skill does not change scripts, only reads them.

### Step 6 — Run the suite (Claude Code only)

After all files are written, the skill runs the suite. Prefer the most specific command available, in this order:

1. The project's documented test script: `npm test` if `package.json` has it, or `npm run test`, or whichever script is named for the runner.
2. Targeted run on the new / modified files: `npx vitest run <file1> <file2> ...` for the unit/RTL tests, `npx playwright test e2e/<flow>.spec.ts` for the new e2e.
3. After the targeted run is green, a full-suite run: `npm test` (or equivalent) to confirm no collateral damage.

Capture the exit code and the output. Report a one-line summary in chat:

- **Green** (all green, both targeted and full-suite): "All `<N>` new tests passed; full suite green (`<M>` tests total). Coverage: AC-driven, see test plan above."
- **Red** (any failure): hand off to §6 (failure protocol).

The skill does not silently retry the suite. It does not run `npx vitest -u` to update snapshots (snapshots are forbidden anyway). It does not pass `--bail=false` or similar to swallow failures. The first failure is the failure to investigate.

### Step 6.5 — In chat mode (no Bash)

If running in chat (or Claude Code without Bash):

- Tests are returned as inline artifacts, one per file, named with their colocated path on disk (`src/auth/session.test.ts`).
- The skill does not run the suite. It states: "I cannot run the suite from here. Run `npm test` (or your project's test command) and tell me the result if anything fails — I can help debug."
- The chat summary lists which tests would have been run.

### Step 6 (continued) — Deliver

The chat summary contains:

- The test baseline (Step 1 output).
- The test plan table.
- The list of files written / modified / deleted.
- The smart-merge diff (Step §5.6) when re-testing a scope that already had tests.
- The suite run result (Claude Code only) — green summary or stop+report (§6).
- The "Possible next steps" list (§9).

### Step §5.6 — Re-test mode (smart merge)

When the skill is invoked on a scope that already has tests (`src/auth/session.test.ts` already exists for the requested module, or `04-tasks/T-007.md` already has tests in the relevant feature folder), the skill performs **smart merge** rather than rewriting from scratch or appending blindly.

The procedure:

1. **Parse existing tests.** For each `it(...)` in each existing file, extract: (a) the AC traceability comment if any, (b) the unit-under-test (`describe` block), (c) the test name, (d) the assertions.
2. **Re-anchor.** For each existing test, ask: is its anchor still in the current scope?
   - Task scope: is the AC still in the task file at the same wording (or close)? If yes — keep. If the AC was reworded — keep but update the comment. If the AC was deleted — flag for deletion.
   - Module scope: is the export still exported with the same signature? If yes — keep. If renamed — rename in the test (one-character or one-symbol rename, no logic change). If deleted — flag for deletion.
   - Diff scope: is the changed branch still present? If yes and the change is now consistent — keep. If the branch was reverted — flag for deletion.
3. **Re-bucket components.** For each existing RTL test, re-walk the §4.4 decision tree against the current component code. If the component has degenerated to bucket 1 / bucket 2 — flag the RTL test for deletion (replaced by hook test or no test).
4. **Identify gaps.** ACs / exports / diff branches with no current test → add a new test.
5. **Output the diff report** in the chat summary:
   ```
   Re-test diff for src/auth/session.test.ts:
   - kept       ×4   (AC#1, AC#2, AC#4, AC#6)
   - updated    ×1   (AC#3 — reworded; comment + name updated, body unchanged)
   - deleted    ×1   (former AC#5 — AC removed from task file)
   - added      ×2   (new AC#7, new AC#8)
   ```
6. **Apply the diff** to the test files.

Smart merge does not change tests in ways that hide regressions. If a test was passing on the previous code shape and now fails because the underlying behavior changed, that is a §6 case — the skill stops and reports, it does not "update the assertion to match the new reality" without an explicit ask.

---

## 6. Failure protocol — "test bug or code bug?"

When the suite is red, the skill does **not** silently retry. It does **not** modify production code. It does **not** weaken / skip / delete the failing test to make the suite green. It applies the protocol below — and the protocol is **stop-on-doubt** by design (per the user-confirmed decision): the skill only auto-fixes evident test bugs, and stops on anything else.

### 6.1. Three buckets

Every failing test lands in exactly one of three buckets. The bucket determines the action.

#### Bucket A — Evident test bug (auto-fix once, then re-run)

A test bug is **evident** only when **all** of these hold:

- The fix is in the test file or in test-only setup (MSW handlers, factories, helper).
- The fix does not change what is being asserted, only how it is set up or expressed.
- The fix is unambiguous: a typo in `expected`, a missing `await`, a missing MSW handler that the test forgot to register, an `userEvent.setup()` that was forgotten, an `act()` warning the test missed, a wrong selector that no semantic anchor in the rendered DOM matches, a path-alias import that resolved against the wrong tsconfig path, a fake-timer cleanup that was forgotten in `afterEach`.
- The production code visibly satisfies the AC / contract / diff intent that the test is anchored to. The test is wrong, the code is right.

Action: **fix the test, re-run once.** Maximum **one** retry per failing test. If the retry is still red, escalate to Bucket C (ambiguity) — do not loop.

#### Bucket B — Evident code bug (stop, report, do not modify code)

A code bug is **evident** only when **all** of these hold:

- The test is anchored to a concrete contract (AC, JSDoc-documented signature, diff intent that is unambiguous from the diff itself).
- The assertion expresses the contract correctly (a careful read of the test against the contract finds no test-side error).
- The production code does something other than what the contract says.
- The fix would be in production code — outside the skill's allowed scope.

Action: **stop and report.** The skill writes a structured failure block in the chat summary (§6.3) and **does not write any further tests on this scope** until the user resolves it (typically by re-invoking `sdlc-impl`). Tests already written for other anchors stay; only the affected file / test is left red on disk so the user can see it.

#### Bucket C — Ambiguity (stop, report both hypotheses)

If neither bucket fits — the test passes on a careful re-read, the code passes on a careful re-read, but the runner says red — the failure is **ambiguous**. This is the default when the skill is not 100% sure.

Action: **stop and report**. The chat summary includes two hypotheses ("might be a test bug because…", "might be a code bug because…"), each with the evidence the skill found. The user decides; the skill does not pick a side.

Examples that land in Bucket C:

- The AC is worded ambiguously and could justify either the current code behavior or the test's expected.
- A race condition the skill is not certain whether to model with `waitFor` or with fake timers.
- A localized failure where the same code path passes in one test and fails in another that looks structurally similar.
- A failure that disappears on a second run with no code change (flaky — the skill does not retry to "make it green"; it reports the flakiness as a finding).

### 6.2. Hard prohibitions during failure handling

These are absolute. They apply to the skill running inline, to the subagent, and to the chat-mode skill.

- **Never edit production code.** Not a typo. Not a `console.log`. Not a one-line refactor that would "make this test happy". The subagent's tool allowlist makes this structural; the skill respects it inline as well.
- **Never delete a failing test** to clear the suite. `it.skip` is also forbidden.
- **Never weaken an assertion** ("change `toBe` to `toMatch`", "loosen the regex", "drop a property from `toEqual`") to clear a failure.
- **Never silently ignore a `console.error`** the test surfaces (act warnings, deprecation warnings that hint at a bug). If the test expects no errors, an error is part of the failure.
- **Never run more than one retry** per Bucket-A fix. After the retry, the failure is either gone (continue) or escalated to Bucket C (stop).
- **Never modify `package.json`** or test-runner config to clear a failure. If the failure points at a missing dep, that is a Bucket B / C report — the user installs.

### 6.3. The stop-and-report block

When the skill stops on Bucket B or Bucket C, the chat summary contains a block of this exact shape:

```
STOP — failing test, classification: <Code bug | Ambiguity>

Test:        src/auth/session.test.ts > "AC#3: returns 401 when the token is expired"
Anchor:      AC#3 in 04-tasks/T-007-session-handler.md
Expected:    response.status === 401 with body { error: 'token_expired' }
Actual:      response.status === 500 with body { error: 'internal' }
Code under:  src/auth/session.ts:42-58 (function verifyToken)

Hypothesis 1 (code bug):
  verifyToken catches the JwtExpiredError but does not re-throw a 401-mapped error;
  the generic 500 handler then catches it. The contract in AC#3 says 401.
  → Action: re-invoke sdlc-impl on T-007 with this test as the failing anchor.

Hypothesis 2 (test bug, only mentioned if not fully ruled out):
  <empty — ruled out: the AC wording is unambiguous, the test asserts it directly>
```

The block is mechanical and predictable. The skill does not editorialize; it presents what it found and what it would do next.

### 6.4. RTL-on-wrong-bucket — a special test bug

A common failure mode worth calling out: an existing RTL test on a component that, on re-evaluation, lands in §4.4 bucket 1 or bucket 2. Symptoms:

- The test mocks the only hook the component uses (`vi.mock('./useUser')`) and asserts the rendered name.
- The test breaks on a cosmetic refactor (whitespace, wrapper change) that did not change behavior.
- The test name reads like "renders the user name" — pure shape, no behavior.

Classification: **test bug** (Bucket A, but the auto-fix is not "patch the assertion" — it is **delete the RTL test and write the hook test instead**). The smart-merge diff records this as `replaced by useXxx hook test`.

The skill applies this only when the bucket misclassification is unambiguous from re-reading §4.4 against the current component code. If there is any doubt, escalate to Bucket C and let the user decide.

---

## 7. Subagent delegation (Claude Code)

In Claude Code, the skill delegates to the **`sdlc-test-writer` subagent** when it exists. The subagent runs with a **strictly limited tool set** by design — it cannot edit production code, install packages, edit `package.json`, or run any non-test command.

### 7.1. Tool allowlist

The subagent's frontmatter declares:

```
tools: Read, Glob, Grep, Write, Bash
```

But the **Write** and **Bash** tools are scoped by the subagent's prompt to a fixed allowlist:

- **Write** is permitted **only** on these path patterns:
  - `**/*.test.ts`, `**/*.test.tsx`, `**/*.spec.ts`, `**/*.spec.tsx`
  - `e2e/**/*.spec.ts`, `e2e/**/*.spec.tsx`, `e2e/**/*.ts` (when the project uses `e2e/` for Playwright)
  - `**/factories.ts`, `**/factories.tsx`, `src/test/**/*.ts`, `src/mocks/**/*.ts` (when extending existing handler / factory layers — never creating a parallel one without an explicit ask up the chain)
  - `__mocks__/**` (when the project already uses this convention)

  Write on **any other path** is forbidden — including production source files, `package.json`, `tsconfig.json`, `vitest.config.ts`, `playwright.config.ts`, and any documentation.

- **Bash** is permitted **only** for these commands:
  - `npm test`, `npm run test`, `npm run test:*` (where `*` matches the project's existing scripts).
  - `npx vitest`, `npx vitest run`, `npx vitest run <file...>`, `npx vitest --reporter=...`.
  - `npx playwright test`, `npx playwright test <file...>`.
  - `pnpm test`, `pnpm vitest`, `pnpm playwright` and `yarn test`, `yarn vitest`, `yarn playwright` mirrors when the project uses pnpm / yarn.

  Bash on **any other command** is forbidden — including `npm install`, `npm i`, `npm ci`, `git`, `cat`, `ls`, `find`, `grep` (use the Grep tool), `sed`, `awk`, `node` (for ad-hoc scripts), `tsc`, `eslint`. `npm install` of any kind is hard-forbidden — the user owns dep management.

The allowlist is enforced by the subagent at the prompt level (it refuses to issue Write / Bash calls outside the allowlist) **and** by the skill at the dispatch level (it does not pass instructions that would require disallowed operations). If the subagent encounters a situation that would require a disallowed operation (a missing dep, a runner-config bug), it stops and reports up — it does not "find a workaround".

### 7.2. Inputs the skill passes to the subagent

When delegating, the calling skill briefs the subagent with:

- **The scope** — task file body (when scope is task), file paths (when scope is module), diff (when scope is diff or task-with-diff), feature surface union (when scope is feature).
- **The test baseline** (Step 1 output) — runner, MSW handler path, factory style, test convention, existing tests for this scope.
- **The test plan** (Step 3) — the table the user has seen and (if e2e was proposed) confirmed.
- **The §4 style rules** (AAA, MSW vs vi.mock, no snapshots, RTL bucket decision tree) — pasted into the brief or referenced by file path.
- **The §6 failure protocol** — pasted or referenced. The subagent must apply it on red runs.

If any of those are missing from the brief, the subagent says so explicitly in its output and refuses to proceed in fabricated mode. Don't fabricate ACs. Don't fabricate a runner that is not configured.

### 7.3. Output the subagent returns

The subagent returns a structured handoff:

- **Files written** (paths + line counts).
- **Files modified** (paths + brief diff summary, e.g. `+3 it() blocks, -1 deleted, 1 renamed`).
- **Files deleted** (paths + reason — re-test smart merge or RTL-bucket-misclassification).
- **Suite run result** — green summary, or the §6.3 stop-and-report block.
- **Any conflicts surfaced** — a test that the AC implies but cannot be written without violating the §4 rules (e.g. an AC that demands snapshot-style verification of a complex DOM tree). The subagent flags these for the user, does not write the test.

The calling skill assembles this into the chat summary the user sees.

### 7.4. Fallback when subagent is missing

If the `sdlc-test-writer` subagent cannot be resolved at either location (`.claude/agents/` project-local or `~/.claude/agents/` user-level), the skill performs the work inline using the same rules. The chat summary notes `Test writer: inline (subagent not found)`. There is no functional difference for the user — only the structural read-only / allowlist guarantees are weaker (the inline-running skill could in principle edit production code; it doesn't, by rule).

---

## 8. Output

The skill's deliverable is **the test files themselves** plus a **chat summary**. There is no separate report artifact (per design).

### 8.1. Files

In Claude Code:

- Test files are written to disk at colocated paths:
  - `src/auth/session.ts` → `src/auth/session.test.ts`
  - `src/users/useUser.ts` → `src/users/useUser.test.ts`
  - `src/checkout/CheckoutPage.tsx` → `src/checkout/CheckoutPage.test.tsx`
- E2E files: `e2e/<flow-name>.spec.ts` (matches the project's `playwright.config.ts` `testDir`).
- New MSW handlers extend the existing handler module when present; otherwise added to a colocated `setupServer(...)` in the test file.
- New factories: only when an existing factory module exists and the skill is extending it. The skill does not introduce a factory module on its own without an explicit user ask.

In chat:

- Each test file is returned as a separate inline artifact, named with the colocated path it would have on disk (`src/auth/session.test.ts`).

### 8.2. Chat summary structure

The summary follows this structure, in this order:

```
## sdlc-tests — <scope identity>

### Test baseline
<Step 1 output: scope kind, runner, conventions detected, existing tests count.>

### Test plan
<The §5.3 table: Anchor / Scope unit / Layer & rationale / File path / Cases.>

### Components routed away from RTL
<List of bucket-1 / bucket-2 components and why no RTL test was written.>

### Files written / modified / deleted
<Per file: path, action, line count or diff summary.>

### Re-test diff (only when re-testing)
<Per file: kept / updated / deleted / added counts and AC references.>

### Suite run result (Claude Code only)
<Either: "All N tests green; full suite green (M total)."
 Or:     The §6.3 stop-and-report block.>

### Possible next steps
<§9 list — informational, not a question.>
```

The summary is **terse**. It does not narrate process. It does not editorialize. It surfaces what was done, what was not done, and what a reasonable next step is.

### 8.3. AC ↔ test mapping table

When scope is task, the summary additionally renders an AC ↔ test mapping table:

| AC | Test file | `it(...)` |
|---|---|---|
| AC#1: user can log in with valid credentials | `src/auth/login.test.ts` | "logs in and returns a session token for valid credentials" |
| AC#2: rejects empty password | `src/auth/login.test.ts` | "rejects empty password with 400 and field-level error" |
| AC#3: returns 401 when the session token is expired | `src/auth/session.test.ts` | "returns 401 with token_expired body when the token is past its exp" |

ACs without a corresponding row are explicitly listed below the table with a reason — usually "(deferred to T-008)" or "(non-testable: NFR observability concern)".

---

## 9. Possible next steps

After producing the artifact, the skill does **not** ask "ready to proceed?". It lists possible next steps as **information**, and the user picks the next move.

- `sdlc-review` — if not yet done on the same task, the new tests (and the §6 stop-and-report when present) are valuable input for the reviewer.
- `sdlc-security` — the new tests can highlight defense gaps (a test for "rejects empty password" is also a security signal); the auditor can reuse the test plan as a partial coverage map.
- `sdlc-impl` — when the skill stopped on a Bucket B (code bug), re-invoking `sdlc-impl` on the failing test as a concrete failing anchor is the canonical path forward.
- `sdlc-docs` — if the new tests describe a new flow at the integration / e2e level, the flow may warrant a sequence diagram update.
- `sdlc-tests` re-run — once the user has resolved the stop-and-report case (typically by another `sdlc-impl` run), invoking `sdlc-tests` again on the same scope triggers smart merge (§5.6), keeping the work that was already correct.

Possible next steps are listed; they are not proposed in order or prioritized. The user decides.
