---
name: sdlc-design
description: Use this skill when the user has an approved Feature Spec (01-feature-spec.md) and needs a visual, interactive prototype before tasks or code. Triggers include "design the UI for…", "make a prototype", "act as a designer", "mock up the screens", "design phase", "wireframe this", "let's prototype", "sdlc-design". Produces a single self-contained 02-prototype.html (Tailwind CDN, vanilla JS) that opens directly in a browser — no build step. Detects the project's existing UI framework (shadcn/ui, MUI, Ant Design, Chakra, Mantine, daisyUI, Bootstrap, etc.) and replicates its visual style so the prototype looks native to the app. For backend-only features, tells the user that design is not needed. Does NOT make architectural decisions (sdlc-adr), does NOT write production code (sdlc-impl), does NOT decompose into tasks (sdlc-pm).
---

# sdlc-design — Designer

This skill turns a **Feature Spec** into a **visual, interactive prototype** — a single HTML file that opens directly in a browser, clickable enough to validate the UX, throwaway enough to iterate on quickly.

The skill is intentionally narrow. It produces exactly one artifact: `02-prototype.html` — self-contained, no build step, runnable on double-click. No React, no TypeScript, no bundler. The prototype is a thinking tool, not production code.

If the feature has no UI surface at all, the skill **does not produce anything** — it tells the user the design phase is not needed and points to the next applicable skill.

The skill **respects existing design decisions**: the project's existing UI framework, design tokens, prior prototypes, and component primitives are the source of truth for how the prototype looks. The skill detects them and replicates their style — it does not invent a new visual language unless the project is greenfield.

---

## 1. Trigger

Activate when the user signals any of:

- An explicit design request: "design the UI", "make a prototype", "mock up the screens", "wireframe this", "act as a designer", "sdlc-design".
- A continuation in the workflow: "design phase for FEAT-003", "prototype the auth feature", "let's see what this could look like".
- Implicit: user just delivered/approved a Feature Spec and asks "what's next visually" or "show me the screens".

Do **not** activate when the user asks for technical decisions (route to `sdlc-adr`), task breakdown (`sdlc-pm`), real implementation (`sdlc-impl`), or written requirements (`sdlc-ba`). If the request is ambiguous between design and an adjacent skill, ask the user once.

---

## 2. Inputs / prerequisites

**Required:** an existing Feature Spec for the feature being designed.

- Claude Code: `docs/features/FEAT-NNN-<slug>/01-feature-spec.md`.
- Chat: a Feature Spec produced earlier in the conversation (or pasted by the user).

**Existing-design context** — actively scanned, not just optional. The prototype must align with what already exists. The scan covers two layers:

### 2.1. UI framework / component library

Detect the project's UI library by scanning, in this order:

| Framework | Detection signals |
|---|---|
| **shadcn/ui** | `components.json` at project root; `src/components/ui/*.tsx` with files named `button.tsx`, `dialog.tsx`, `input.tsx`; `class-variance-authority`, `@radix-ui/*`, `tailwind-merge` in `package.json` |
| **Material UI (MUI)** | `@mui/material`, `@mui/icons-material` in `package.json` |
| **Ant Design** | `antd` in `package.json`; possibly `@ant-design/icons` |
| **Chakra UI** | `@chakra-ui/react`, `@emotion/react` in `package.json` |
| **Mantine** | `@mantine/core`, `@mantine/hooks` in `package.json` |
| **daisyUI** | `daisyui` plugin in `tailwind.config.*`; `data-theme` attributes in code |
| **Bootstrap** | `bootstrap` in `package.json` or `<link>`/`<script>` to `bootstrap` CDN in `index.html` |
| **Fluent UI** | `@fluentui/react` or `@fluentui/react-components` |
| **Carbon** | `@carbon/react`, `@carbon/styles` |
| **Headless UI + plain Tailwind** | `@headlessui/react` with no other UI lib; raw Tailwind in `src/components/` |
| **Radix UI primitives only** | `@radix-ui/*` packages without `components.json` (no shadcn wrappers) |
| **None / custom** | None of the above. Plain CSS, plain Tailwind, or hand-rolled components in `src/components/`. |

Stop at the first match. If two are present (rare migration cases), ask the user which is canonical.

### 2.2. Tokens, primitives, prior prototypes

In addition to the framework, scan:

| Source | Where (Claude Code) | What to extract |
|---|---|---|
| Prior prototypes for **this** feature | `docs/features/FEAT-NNN-<slug>/02-prototype.html` (incl. versioned) | Layout, components, copy already validated. |
| Prototypes for **related** features | `docs/features/FEAT-*/02-prototype.html` | Reusable patterns, shared screens, persona switchers. |
| Theme / token config | `tailwind.config.{js,ts,cjs,mjs}`, `src/styles/`, `src/design-tokens.*`, CSS custom properties in `src/index.css` / `src/global.css`, `theme.ts` for MUI/Chakra/Mantine | Color palette, spacing scale, typography ramp, radii, shadows, font families. |
| Component primitives | `src/components/ui/`, `src/components/`, `packages/ui/`, Storybook config (`.storybook/`) | Existing primitives' visual shape and props. |
| Design system docs | `docs/design-system/`, `DESIGN.md`, `STYLEGUIDE.md`, `README.md` | Conventions, dos/don'ts, brand voice. |
| Brand assets | `public/`, `src/assets/` (logos, favicons, illustrations) | Logo placement, brand color, illustration style. |
| Prior reviews / critiques | `05-review-*.md`, design notes pasted by user | Constraints already imposed on the visual language. |

In **chat mode**, equivalents are: framework names mentioned, screenshots, tokens, or prototypes the user has shared earlier in the conversation, plus anything they paste inline. If nothing is shared, ask the user once which UI framework the project uses (offer a multiple-choice from the table above).

**Missing input handling:**

- No Feature Spec → ask for one (offer `sdlc-ba`, paste-inline, or thin-brief-with-warning).
- No UI framework detected → treat as greenfield: use plain Tailwind defaults, document this in the prototype header.
- Existing context **conflicts with the spec or with what the user is asking for** → name the conflict explicitly and ask before proceeding (see §8 Edge cases).

Do not silently invent the spec. Do not silently override existing tokens, primitives, or framework style.

---

## 3. Environment detection

Run this check at the start of every invocation:

1. Can you create files via the Write tool **and** does the working directory contain `docs/features/` (or `package.json` / `.git` / `src/`)?
   → **Claude Code mode**. Prototype is written to disk.
2. Otherwise:
   → **Chat mode**. Prototype is delivered inline as an HTML artifact in the conversation.

If ambiguous (e.g. tools are present but no project markers), ask the user once: "Should I write the prototype to disk or return it inline?" Remember the answer for the rest of the turn.

---

## 4. Execution

Same logical flow in both environments. Only step 5 (delivery) differs.

### Step 1 — Identify the feature, load the spec, scan existing design

1. Determine which feature this prototype is for (named explicitly, inferred from context, or asked once).
2. Read `01-feature-spec.md` end-to-end. Pay special attention to:
   - **User Stories** — each story usually maps to one screen, one modal, or one major interaction.
   - **Functional Requirements** — every UI-relevant FR must be visible somewhere in the prototype.
   - **Acceptance Criteria** — drives which states/edge cases the prototype should demonstrate.
   - **Non-Goals & Out of Scope** — do **not** design those.
   - **NFRs** with UI implications — accessibility, supported browsers, responsive expectations.
3. **Scan for UI framework** (§2.1) — read `package.json`, look for marker files (`components.json`, `tailwind.config.*`, `theme.ts`), inspect `src/components/ui/`. Stop at the first match.
4. **Scan for tokens and prior prototypes** (§2.2) — extract palette, spacing, radii, fonts; open the most recent `02-prototype.html` in any feature folder as a visual baseline.
5. Produce a short internal "design baseline" note (mental, or as the comment block in the prototype header — see Step 4):
   - Detected framework: `<name>` or `none / greenfield`.
   - Token sources: `tailwind.config.ts` (palette, spacing) / `theme.ts` / etc.
   - Baseline prototype: `FEAT-002-…/02-prototype.html` (if any).
   - Conflicts to surface: list, or "none".
6. If the spec is missing UI-critical detail, pull it once via clarification (Step 3). Don't guess.

### Step 2 — Decide whether design is needed

Check whether the feature has any UI surface at all by reading the spec:

- **No UI surface** (background jobs, internal services, pure data pipelines, infra changes, machine-to-machine APIs without an admin UI):
  → **Stop.** Tell the user: "This feature is backend-only — no design phase needed. Suggested next steps: `sdlc-adr` (if architectural decisions are open) or `sdlc-pm` (to break it into tasks)."
  → Do not produce any artifact. Do not ask clarifying questions. Exit cleanly.

- **Has UI surface** → continue.

There is no mode choice. The output is always a single `02-prototype.html`.

### Step 3 — Clarify (only if needed)

Ask **2–5 questions in a single batch** using `AskUserQuestion` only if the spec leaves real UI gaps **or** existing-design conflicts need resolving. Question candidates:

- "Which screens are must-have for this prototype?" — multi-select of inferred screens.
- "Which states should the prototype demonstrate?" — multi-select: empty / loading / error / success / partial-permission.
- "I detected `<framework>` as the UI library — replicate its style?" — options: yes / use plain Tailwind / specify another.
- "Visual style baseline?" — options: detected framework + tokens / generic neutral / specific reference (paste link). *(Only if framework detection was ambiguous.)*
- "Responsive scope?" — options: desktop only / desktop + mobile / mobile-first.

Do **not** loop. One batch, then proceed. Anything unresolved becomes an `<!-- OPEN: … -->` comment in the prototype.

### Step 4 — Build the prototype

The prototype is a single `02-prototype.html` file. It must:

- Open directly in a browser on double-click — no build step, no `npm install`, no dev server.
- Visually resemble the project's UI framework as detected in Step 1.
- Cover every UI-relevant user story plus the states selected in Step 3.

#### 4.1. File skeleton

- `<!DOCTYPE html>` + `<head>` with viewport meta and `<script src="https://cdn.tailwindcss.com"></script>`.
- If a project `tailwind.config.*` was found, mirror its `theme.extend` (colors, spacing, fontFamily, borderRadius, boxShadow) inside `<script>tailwind.config = { theme: { extend: { … } } }</script>` **before** any `<body>` content. Do not drop the project palette in favor of Tailwind defaults.
- All screens stacked in the document, separated by clear section headers (`<section id="screen-login">`). Use a sticky top nav with anchor links to jump between screens — that **is** the navigation in HTML mode. If multiple personas exist, add a persona switcher in that nav.
- Inline `<script>` for any interactivity (toggles, modals, tab switches, mocked form submits, optimistic updates). Vanilla JS only — no React, no Alpine, no build step.
- Mock data inline as JS arrays/objects when needed for lists/tables. Obviously fake names/emails — no real PII.
- Cover the states selected in Step 3 — show empty/loading/error variants side by side or behind toggle buttons rather than hiding them.
- Top-of-file HTML comment block (mandatory):
  ```html
  <!--
    FEAT-NNN-<slug> — 02-prototype.html
    Detected framework: shadcn/ui (components.json + Radix in package.json)
    Token sources: tailwind.config.ts (palette, spacing, radius)
    Baseline prototype: FEAT-002-team-billing/02-prototype.html
    Deviations from baseline: none
    Open questions: see <!-- OPEN: ... --> comments inline.
  -->
  ```

#### 4.2. Replicating the detected UI framework

The prototype must **look like the user's app**. Below is a cheat sheet for replicating each framework using only Tailwind utilities + small custom CSS. Pick the row matching the detection from Step 1.

| Framework | Palette / radius defaults | Buttons | Inputs | Cards / surfaces |
|---|---|---|---|---|
| **shadcn/ui** | Neutral palette: `zinc` or `slate` for surfaces, `primary` from CSS vars; radius `0.5rem` (`rounded-md`); subtle `border` and `border-border` | `bg-primary text-primary-foreground hover:bg-primary/90`; variants: `default`, `secondary` (`bg-secondary`), `outline` (`border bg-background`), `ghost` (`hover:bg-accent`), `destructive` (`bg-destructive`); height `h-9` `h-10` `h-11`; `inline-flex items-center justify-center rounded-md text-sm font-medium` | `flex h-10 w-full rounded-md border border-input bg-background px-3 py-2 text-sm ring-offset-background placeholder:text-muted-foreground focus-visible:outline-none focus-visible:ring-2 focus-visible:ring-ring` | `rounded-lg border bg-card text-card-foreground shadow-sm`; padding `p-6` for content; `space-y-1.5` for header |
| **Material UI** | Roboto font (`<link href="https://fonts.googleapis.com/css?family=Roboto">`); primary `#1976d2`, secondary `#9c27b0`, error `#d32f2f`; radius `4px` (`rounded`); elevation via `shadow-md`/`shadow-lg` | Uppercase text, `text-sm font-medium tracking-wider px-4 py-1.5 rounded`; variants: `contained` (filled w/ shadow), `outlined` (`border-2`), `text` (no border, no bg) | Floating-label feel: label small above input; underline variant via `border-b-2 focus:border-primary`; outlined variant via `border rounded` | Elevation cards: `bg-white rounded shadow-md`; subtle `border` rare |
| **Ant Design** | Primary `#1677ff`, success `#52c41a`, error `#ff4d4f`; radius `6px` (`rounded-md`); subtle shadow `0 2px 8px rgba(0,0,0,0.06)` | `bg-[#1677ff] text-white px-4 py-1 rounded-md hover:bg-[#4096ff]`; variants: `primary`, `default` (`border bg-white`), `dashed` (`border-dashed`), `link`, `text`; height `h-8` (small) / `h-9` (default) / `h-10` (large) | `border border-gray-300 rounded-md px-3 py-1 hover:border-[#4096ff] focus:border-[#1677ff] focus:shadow-[0_0_0_2px_rgba(22,119,255,0.1)]` | `bg-white rounded-lg border border-gray-200 shadow-sm`; header divider `border-b` |
| **Chakra UI** | Default `gray.700` text, `blue.500` primary; radius `md` (`6px`); generous spacing | `bg-blue-500 text-white rounded-md px-4 py-2 hover:bg-blue-600`; variants: `solid`, `outline` (`border-2`), `ghost`, `link` | `border border-gray-200 rounded-md px-3 py-2 focus:border-blue-500 focus:ring-1 focus:ring-blue-500` | `bg-white rounded-lg shadow-sm border border-gray-100` |
| **Mantine** | Similar to Chakra; primary blue `#228be6`; radius `4px` default; subtle shadows | `bg-[#228be6] text-white rounded px-3.5 py-1.5 hover:bg-[#1c7ed6]` | `border border-gray-300 rounded px-3 py-1.5 focus:border-[#228be6]` | `bg-white rounded shadow-sm border` |
| **daisyUI** | Theme-driven (e.g. `data-theme="light"`); semantic classes like `btn`, `card`, `input` | Replicate `btn-primary` / `btn-secondary` / `btn-ghost` look from the theme's primary color | `input input-bordered` look — `border-2 rounded-lg px-4 py-2` | `card bg-base-100 shadow-xl` look — `bg-white rounded-2xl shadow-xl` |
| **Bootstrap** | Primary `#0d6efd`, secondary `#6c757d`, danger `#dc3545`; radius `0.375rem` | `bg-[#0d6efd] text-white rounded px-3 py-1.5 hover:bg-[#0b5ed7]`; variants: `primary`, `secondary`, `outline-*`, `link` | `border border-gray-400 rounded px-3 py-1.5 focus:border-[#86b7fe] focus:ring-2 focus:ring-[#0d6efd]/25` | `bg-white rounded border border-gray-200`; optional shadow |
| **Fluent UI** | Microsoft palette: primary `#0078d4`; radius `4px`; subtle shadows | `bg-[#0078d4] text-white rounded px-4 py-1.5 hover:bg-[#106ebe]` | `border border-gray-400 rounded px-3 py-1.5` | `bg-white rounded shadow-sm` |
| **Carbon** | IBM palette: primary `#0f62fe`; radius `0` (sharp corners); flat | `bg-[#0f62fe] text-white px-4 py-2 hover:bg-[#0353e9]` (no radius) | `border-b border-gray-500 px-3 py-2 bg-transparent` | `bg-white border` (sharp corners) |
| **Headless UI + Tailwind** | No prescribed palette — read `tailwind.config.*` and use what's there | Tailwind primitives matching the project's primary color | Tailwind primitives | Tailwind primitives |
| **Radix only (no shadcn)** | Treat similarly to shadcn but without the predefined variants — read tokens from `tailwind.config.*` | Use the project's primary color from config | Same | Same |
| **None / greenfield** | Tailwind defaults: `gray` neutrals, `blue-600` primary, `rounded-md` | Standard Tailwind: `bg-blue-600 text-white rounded-md px-4 py-2 hover:bg-blue-700` | Standard: `border rounded-md px-3 py-2 focus:ring-2 focus:ring-blue-500` | `bg-white rounded-lg border shadow-sm` |

Beyond buttons / inputs / cards, replicate the framework's idioms for: modals (Material has full-screen backdrop + elevated surface; Ant has narrower modal with header divider), tabs (shadcn underline vs MUI underline-with-indicator), navigation, badges, toasts. When in doubt about a specific component's look, mimic the closest equivalent from the framework's docs.

#### 4.3. Constraints

- **English** for all UI copy in the prototype. Add a one-line note in the header if the user explicitly wants another language.
- No real backend calls. No real auth. No real PII in mock data.
- Cover **every user story** that has a UI surface. If a US is intentionally skipped, leave a visible `<!-- OPEN: … -->` comment with the reason.
- Demonstrate the **happy path end-to-end** plus at least one error/edge state.
- Accessibility floor: semantic HTML (`<button>`, `<label>`, `<nav>`, `<main>`), `alt` on images, visible focus rings, sufficient contrast. Don't claim WCAG AA — that is `design:accessibility-review`'s call.
- No production concerns: no analytics, no feature flags, no env vars. The prototype is for thinking, not shipping.
- **Any deviation from the detected framework's style must be explicit** — listed in the header comment block with a one-line rationale. Never silent.

### Step 5 — Deliver

**Claude Code mode:**
1. Ensure `docs/features/FEAT-NNN-<slug>/` exists (create if needed).
2. If `02-prototype.html` already exists, ask the user: "Replace, version (`02-prototype-v2.html`), or open the existing one for edits?"
3. Write `docs/features/FEAT-NNN-<slug>/02-prototype.html`.
4. Print the absolute path and suggest opening it in a browser (double-click or `open <path>`).

**Chat mode:**
1. Return the prototype as an HTML artifact, title `FEAT-NNN-<slug> — 02-prototype.html`.
2. Follow the global artifact rules from the environment (no `localStorage`).

In both modes:
- Print a **2–4 line summary**: detected framework, number of screens/states, user stories covered, and any deviations from the baseline.
- Print the **possible next steps** (see §6 below) as an informational list.
- Phrase the summary and next-steps line in the user's language; the artifact itself stays in English.

---

## 5. Output

| Field | Value |
|---|---|
| Filename | `02-prototype.html` |
| Path (Claude Code) | `docs/features/FEAT-NNN-<slug>/02-prototype.html` |
| Title (chat) | `FEAT-NNN-<slug> — 02-prototype.html` |
| Format | Single self-contained HTML file. Tailwind via CDN. Vanilla JS for interactivity. No build step. |
| Runs by | Double-click in a file browser, or `open <path>` in a terminal. |
| Status | Prototype (throwaway by default). |
| Language | English (UI copy + comments). |
| Mandatory header | Top-of-file HTML comment listing detected framework, token sources, baseline prototype, and explicit deviations. |
| Backend-only feature | No artifact. Skill exits with a "design not needed" message. |

---

## 6. Possible next steps

After delivering the prototype, present these as information, not a question. Phrase in the user's language. The user decides what runs next.

- Iterate on this prototype — refine specific screens, add missed states, change the visual baseline.
- `design:design-critique` — get structured feedback on hierarchy, usability, consistency.
- `design:accessibility-review` — WCAG 2.1 AA pass before engineering picks it up.
- `design:design-system` — if the prototype surfaced a missing primitive worth promoting into the system.
- `sdlc-adr` — if the prototype surfaced architectural questions (data shape, real-time vs polling, auth model).
- `sdlc-pm` — if the spec + prototype are solid enough to break into tasks (typically also wants an ADR first).

For backend-only features (skill exited early), the only relevant next steps are `sdlc-adr` and `sdlc-pm`.

Do **not** ask "ready to proceed?". Do **not** auto-invoke another skill.

---

## 7. Out of scope (do not do)

- React / TSX prototypes. The prototype is HTML only — must run on double-click. Engineering will rebuild the UI with the project's framework in `sdlc-impl`.
- Architectural decisions (state library, data fetching strategy, hosting) → `sdlc-adr`.
- Production code, real API integration, real auth → `sdlc-impl`.
- Task decomposition → `sdlc-pm`.
- Editing or rewriting the Feature Spec → `sdlc-ba`.
- API contracts / OpenAPI specs — out of scope. If needed, surface as an ADR question or implementation task.
- Producing or reorganizing a full design system → `design:design-system`.
- Pixel-perfect visuals or final brand work — the prototype is a thinking tool.
- Multi-file outputs, build configs, npm scripts. One file, one artifact.
- Auto-generating screenshots, GIFs, or marketing assets.
- Asking clarifying questions one at a time — always batch in Step 3.
- Writing prototype copy or comments in any language other than English.
- Silently overriding the detected framework's style or existing tokens. Deviation requires the explicit header note (§4.1).

---

## 8. Edge cases

- **Spec has UI implications but is silent on flow order**: infer the most plausible flow, prototype it, and leave an `<!-- OPEN: … -->` comment on the inferred sequence so the user can correct.
- **Spec covers multiple personas** (admin + end-user): build one screen-set per persona, switchable via a top-bar persona selector. Don't merge personas into one ambiguous UI.
- **Mostly backend feature with one tiny admin UI**: still produce a prototype for that UI. The "design not needed" exit only applies when there is *no* UI surface at all.
- **User insists on a prototype for a backend-only feature**: explain once that there is nothing user-facing to prototype. If they still want it, ask what UI surface they have in mind (operator dashboard? logs viewer?) before proceeding.
- **Two UI frameworks present in `package.json`** (e.g. mid-migration from MUI to shadcn): ask the user which is canonical for new work. Prototype in that one. Note the migration in the header.
- **Detected framework conflicts with the spec's NFRs** (e.g. spec demands WCAG AA but framework defaults are low-contrast): name the conflict explicitly. Either prototype with current tokens and flag the contrast issue as an open question, or override with a header note explaining why.
- **User asks to ignore the detected framework** ("ignore shadcn, design freely"): ask once whether this is a one-off exploration or a real divergence. If real divergence — record it in the header as an explicit deviation; suggest `design:design-system` to evolve the system properly.
- **No design baseline exists yet** (greenfield, first prototype in the project): use plain Tailwind defaults; note in the header — "Baseline: greenfield, conventions established here." Future prototypes will treat this one as the baseline.
- **Two prior prototypes give different baselines**: take the most recent as canonical, mention the divergence in the header, and suggest the user run `design:design-system` to consolidate.
- **Custom in-house UI library** with no recognizable framework name: read `src/components/ui/` (or equivalent), extract visual patterns from a few primitives, replicate them. Document the source files in the header.
- **Feature spec is in `Draft` status with many open questions**: produce the prototype anyway, but add an "Open questions reflected here" block in the header mapping each unresolved question to the screen/element where the assumption was made.
- **User asks for a redesign of an existing prototype**: read the existing `02-prototype.html` first. Diff against the spec, propose changes section-by-section before rewriting. Don't lose existing content silently.
- **Conflict between spec and prototype scope** (user verbally extends scope): record the deviation in the prototype's header, do **not** edit the spec from this skill — that's `sdlc-ba`'s job.

---

## 9. Template reference

There is no template file for this skill (prototype format is too variable for a useful template). Format conventions live in §4 of this `SKILL.md`.

The skill reads the real Feature Spec from `docs/features/FEAT-NNN-<slug>/01-feature-spec.md` as its prerequisite input. No template is consulted — the spec is a runtime artifact, not a template.
