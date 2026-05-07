---
name: sdlc-impl
description: Use this skill when the user has an approved task file (T-NNN-{slug}.md) under docs/features/FEAT-NNN-{slug}/04-tasks/ and wants the code implemented (TS/Node/React). Triggers include "implement T-NNN", "code this task", "act as engineer", "pick up T-001", "build it", "let's write the code", "sdlc-impl". Reads the task file plus the ADR(s) and Feature Spec, scans existing codebase conventions, decomposes multi-layer work into UI / Frontend / Backend sub-stages with a soft gate between them (announce the transition, brief pause for objections, then continue), implements one sub-stage at a time, runs lint + typecheck + smoke run of existing unit tests on touched modules after each sub-stage, scaffolds empty test files matched to the Test Plan but does NOT fill tests (that is sdlc-tests). Never edits the task's Status field. On conflicts with the spec or ADR — stops and escalates, never silently overrides. Does NOT write the spec (sdlc-ba), ADR (sdlc-adr), task breakdown (sdlc-pm), reviews (sdlc-review), security audits (sdlc-security), tests (sdlc-tests), or diagrams (sdlc-docs).
---

# sdlc-impl — Implementer

This skill turns an **approved task file** into **real code** — TypeScript / Node / React, written into the existing repository (Claude Code) or returned as inline artifacts (chat).

The skill is intentionally narrow. It implements **one task at a time**, scoped exactly to the contents of `T-NNN-<slug>.md`. It does not invent new features, does not rewrite the spec, does not retroactively re-architect the system. It does not write tests beyond empty scaffolds — that is `sdlc-tests`. It does not produce a review of its own output — that is `sdlc-review`.

The skill **respects existing decisions**: the Feature Spec defines *what*, the ADR defines *how* at the architectural level, the task file defines the slice, and the existing codebase defines the conventions every line of new code must follow. The skill aligns with all four. When the four disagree, the skill **stops and escalates** — never silently picks a side.

Multi-layer tasks are split internally into **three sub-stages — UI / Frontend / Backend** — with a **soft gate** between them: at the boundary, the skill prints a 2–4 line summary of what just shipped and what's next, gives the user a brief moment to object, and continues if there is no objection. Single-layer tasks are implemented straight through.

The skill never touches the task's `Status` field. Status is human / PM territory.

---

## 1. Trigger

Activate when the user signals any of:

- An explicit implementation request: "implement T-001", "code this task", "act as engineer", "pick up the next task", "build it", "let's write the code", "sdlc-impl".
- A continuation in the workflow: "go to impl on FEAT-003", "start implementing", "wave 1 of the plan", "T-002 next".
- A scoped code request that maps cleanly to an existing task file: "write the users API endpoint per T-002".

Do **not** activate when the user asks for product requirements (route to `sdlc-ba`), architectural decisions (`sdlc-adr`), task breakdown (`sdlc-pm`), UI prototypes (`sdlc-design`), code review (`sdlc-review`), security audit (`sdlc-security`), test writing (`sdlc-tests`), or diagrams (`sdlc-docs`). If the request is ambiguous between impl and an adjacent skill, ask the user once.

If the user asks to implement *something* with no task file in sight, the skill first checks whether a task file exists; if not, it points to `sdlc-pm` rather than fabricating a plan inline.

---

## 2. Inputs / prerequisites

**Required:** an existing task file **and** the ADR(s) it depends on **and** access to the codebase it touches.

- Claude Code: `docs/features/FEAT-NNN-<slug>/04-tasks/T-NNN-<slug>.md`, `docs/features/FEAT-NNN-<slug>/03-adr*.md`, and the working tree under `src/` (or equivalent).
- Chat: the task content pasted (or produced earlier in the conversation) plus the ADR(s) and any code snippets the user shares as context.

**Existing-context** — actively scanned, not just optional. The implementation must be consistent with what is already decided and how the code is already written. The scan covers:

| Source | Where (Claude Code) | What to extract |
|---|---|---|
| Task file | `docs/features/FEAT-NNN-<slug>/04-tasks/T-NNN-<slug>.md` | Files to Touch, Implementation Plan, Definition of Done, Test Plan, Notes (assumptions, gotchas), Depends-on (must already be Done — verify). |
| ADR(s) for this feature | `docs/features/FEAT-NNN-<slug>/03-adr*.md` | Chosen tech, libraries, integration shape, rejected options, known limitations. Implementation aligns with chosen, never reaches for rejected. |
| Feature Spec | `docs/features/FEAT-NNN-<slug>/01-feature-spec.md` | FR/NFR/AC the task traces to. The implementation must satisfy the AC the task is responsible for. |
| Sibling ADRs | `docs/features/FEAT-*/03-adr*.md` | Project-wide tech choices (DB, queue, auth, deployment) that the new code must reuse. |
| Project-level architecture | `docs/c4/context.mmd`, `docs/c4/container.mmd` | Container boundaries, external systems — informs where new code physically lives. |
| Prior implementations of this feature | files referenced by `Depends on` task IDs (already merged) | Naming, module layout, error handling, type definitions, validation library, test style. |
| Codebase shape | `package.json`, `tsconfig.json`, `src/`, `prisma/schema.prisma`, `migrations/`, `.eslintrc*`, `biome.json`, `vitest.config.*`, `tailwind.config.*` | Tooling versions, module layout, lint/format/test commands, framework versions. |
| Prototype | `docs/features/FEAT-NNN-<slug>/02-prototype.html` (or `.tsx`) | Concrete screens, copy, interaction shapes — the UI sub-stage rebuilds them with the project's real framework. |

From the scan, build a short internal "implementation baseline":

- Detected lint/format/test commands: `<list>` (e.g. `npm run lint`, `npm run typecheck`, `npm run test`).
- Module layout signal: `<paths>` (e.g. `src/modules/<domain>/{routes,services,repositories,validators}.ts`).
- Validation lib: `<name>` (e.g. Zod / Yup / Joi).
- Error handling pattern: `<shape>` (e.g. central error middleware throwing `AppError`).
- UI framework / component primitives: `<list>` (e.g. shadcn/ui at `src/components/ui/`).
- Conflicts to surface: `<list or 'none'>`.

**Missing input handling:**

- No task file → ask once: "I need a `T-NNN-<slug>.md` to implement. Run `sdlc-pm` first, paste the task inline, or describe the change as a thin brief (with a quality warning)?"
- No ADR for a feature with non-trivial technical surface → ask once: "I don't see `03-adr.md`. Run `sdlc-adr` first, paste decisions inline, or proceed without ADR (implementation may need rework)?"
- Task `Depends on` references unmet (the predecessor task isn't merged into the working tree) → stop and surface: "T-NNN depends on T-MMM, which I don't see implemented yet. Implement T-MMM first, mark it done in your tracker, or proceed with explicit deviation note?" Default to stopping.
- Codebase scan returns nothing useful (greenfield repo) → note this in the start summary and use the SDLC defaults (TypeScript / Node / React / Vitest). Treat the first task as establishing conventions.

Do not silently invent the task. Do not silently override the ADR. Do not silently fabricate predecessor task output.

---

## 3. Environment detection

Run this check at the start of every invocation:

1. Can you read **and** write files via the Read/Write/Edit tools, **and** does the working directory contain `docs/features/` (or `package.json` / `.git` / `src/`)?
   → **Claude Code mode**. Code is written into real files; lint/typecheck/smoke run executes via bash.
2. Otherwise:
   → **Chat mode**. Code is delivered as inline artifacts. No tool execution. The skill explicitly notes that lint/typecheck/tests cannot run.

If ambiguous (tools present but no project markers), ask the user once: "Should I write files into the repository or return code inline?" Remember the answer for the rest of the turn.

---

## 4. Execution

Same logical flow in both environments. Only Step 5 (delivery) and Step 6 (verification) differ.

### Step 1 — Identify the task, load context, scan the codebase

1. Determine which task to implement (named explicitly, inferred from "wave 1 first must-have", or asked once).
2. Read the task file end-to-end. Internalize:
   - **Files to Touch** — the contract for which files this task may add/modify. The skill stays inside this list unless it discovers a needed adjacent change (then it surfaces the deviation in Step 4 summary).
   - **Implementation Plan** — the steps the skill follows. Treat as a working plan, not gospel — adapt if the codebase scan reveals a better fit, but call out the deviation explicitly.
   - **Definition of Done** — the bar. Every checkbox must be satisfiable by the code at the end. Floor checkboxes (lint clean, scaffolds exist, no regressions) plus task-specific ones.
   - **Test Plan** — the spec for `sdlc-tests`. Determines what test scaffolds (empty files with `describe`/`it.todo` placeholders) the skill creates.
   - **Notes & Considerations** — explicit assumptions and ADR section pointers.
3. Read the ADR(s) end-to-end. Pay close attention to chosen options (§5) and Consequences (§9) — these constrain the implementation.
4. Read the Feature Spec sections referenced by the task's `Related requirements`.
5. Run the codebase scan per §2. If something the task references doesn't exist where expected (e.g. `src/modules/users/` referenced but the project uses `src/users/`), trust the codebase, note the discrepancy in Step 4 summary.
6. Verify `Depends on`. For each predecessor task, confirm its outputs are present (the files it should have created exist). If any are missing → see "missing input handling" in §2.

### Step 2 — Decide single-layer vs multi-layer

Inspect the task's Files to Touch and Implementation Plan and bucket the work into the three sub-stages:

| Sub-stage | What lives here |
|---|---|
| **UI** | Pure presentational React components: props in, JSX out. Tailwind for styling, local `useState` only for UI-only state, no API calls, no global state, no routing. |
| **Frontend** | Hooks, state management (Zustand / Redux / TanStack Query / etc.), routing, API clients, forms, client-side validation, data fetching, error boundaries. |
| **Backend** | API endpoints, services, repositories, DB models, migrations, server-side validation, authentication, background jobs. |

- **Single-layer task** (touches only one bucket, ≤ ~6 files): implement straight through, skip Step 3, jump to Step 4 (the implementation pass).
- **Multi-layer task** (touches two or more buckets): proceed to Step 3 to choose the order.

Borderline cases (e.g. one tiny shared util used by both backend and frontend): treat as single-layer rooted in the heavier bucket; keep the shared util tightly scoped.

### Step 3 — Choose sub-stage order (multi-layer only)

Ask the user via `AskUserQuestion` — exactly **one question, 2–4 options**:

- "This task spans UI + Frontend + Backend (or a subset). In which order?"
  - **Backend → Frontend → UI** *(API-first — recommended when contracts are crisp from the ADR; UI rebuilds last from the prototype)*.
  - **UI → Frontend → Backend** *(UI-first — recommended when the prototype is the strongest spec; backend wires up to a known shape)*.
  - **Backend → UI → Frontend** *(parallel-friendly when backend and UI can be built independently against the contract; the wiring layer comes last).*

If the task touches only two of three buckets, present only the two relevant orderings.

Mark the natural default `(Recommended)` per the ADR's strongest signal — if the ADR commits to a concrete API contract, recommend Backend-first; if the prototype is detailed and the API is open-ended, recommend UI-first.

The user's choice **freezes the order** for this invocation. No re-asking mid-implementation.

### Step 4 — Implement, sub-stage by sub-stage (soft gate between)

For each sub-stage in the chosen order, run the inner pass below. After each sub-stage finishes, the **soft gate** triggers — see §4.3 at the bottom of this step.

#### 4.1. Inner pass per sub-stage

1. **Re-scan** for the files the sub-stage touches. Open every file you intend to modify *before* writing to confirm current contents — never edit blind.
2. **Plan the edits** mentally (or as a 3–5 line outline in chat). Group changes by file. Keep cross-file consistency: if a type is added in one file, every consumer in this sub-stage uses it.
3. **Write code** that follows the codebase baseline from §2:
   - **Naming** — match the project's casing, suffix conventions, and module boundaries. If the project uses `*.service.ts` / `*.repository.ts` suffixes, the new code uses them too.
   - **Types** — explicit return types on exported functions; no implicit `any`. Reuse existing domain types (don't redeclare a `User` if `src/types/user.ts` exists).
   - **Validation** — use the project's existing library (Zod / Yup / Joi). Don't introduce a parallel one.
   - **Error handling** — match the existing pattern (custom `AppError` classes, central middleware, `Result` type, etc.). Don't introduce a parallel mechanism.
   - **Imports** — match aliasing (`@/` vs relative). Match style (named vs default). Run a mental import-sort.
   - **Comments** — terse and only where the *why* isn't obvious. No JSDoc unless the project uses JSDoc.
   - **Tailwind / UI primitives** — UI sub-stage reuses `src/components/ui/` (or equivalent). Don't reinvent buttons/inputs; don't drop the project palette.
4. **Scaffold test files** for every production file the sub-stage adds, per the task's Test Plan:
   - Create a sibling test file (e.g. `users.service.ts` → `users.service.test.ts`) using the project's existing test path convention (sibling vs `__tests__/` vs `tests/`).
   - The scaffold contains: imports, a top-level `describe(<unit>, …)` with one `it.todo(<phrasing taken from Test Plan>)` per Test Plan bullet that targets this unit. Empty body. **No real assertions.**
   - For UI sub-stage components: scaffold a `*.test.tsx` with `it.todo` lines mapping to the Test Plan's component-test bullets.
   - For e2e bullets in the Test Plan (Playwright): scaffold an empty spec under the project's e2e folder *only if* the e2e folder already exists. If not, leave a single `<!-- TODO sdlc-tests: e2e folder bootstrap -->` line in a placeholder file at the project's expected e2e path and call it out in the soft-gate summary.
   - **Never write actual test logic.** That is `sdlc-tests`. The skill's job is to make the file exist with the right shape so `sdlc-tests` knows what to fill.
5. **Apply the changes**:
   - Claude Code: use `Edit` for in-file changes, `Write` for new files. One file per tool call. Read before editing if the file wasn't read earlier in this invocation.
   - Chat: append the file to the staged artifact list. One artifact per file. Do not merge unrelated files into one artifact.

#### 4.2. Conflict during the inner pass — STOP and ESCALATE

If, while implementing, the skill discovers a real conflict between (a) the task file, (b) the ADR, (c) the Feature Spec, or (d) the existing codebase that cannot be resolved without choosing a side, **stop**. Do not silently pick. Do not add a deviation note and continue.

Surface the conflict clearly:

1. Name the conflict in one sentence: which file, which decision, which side disagrees with which.
2. Quote the relevant lines from each source (max 3 lines per source).
3. Offer 3–4 options via `AskUserQuestion`:
   - **Align with the existing decision** — modify the task interpretation to fit ADR/spec/codebase.
   - **Supersede the ADR** — invoke `sdlc-adr` to record a new decision, pause `sdlc-impl` until the ADR is updated.
   - **Revise the spec / task** — invoke `sdlc-ba` (spec) or `sdlc-pm` (task) and pause.
   - **Proceed with explicit deviation** — only when the conflict is local and harmless; the deviation goes in the post-implementation summary (Step 7), never silently in the code.

Until the user picks, the skill does not write more code in the affected area. (It may still finish unrelated parts of the current sub-stage if they are independent.)

#### 4.3. Soft gate at sub-stage boundary

When a sub-stage finishes (inner pass + verification per Step 6 done), print a compact transition summary:

```
✅ Backend done — 5 files written, 3 modified, lint clean, typecheck clean, smoke tests green.
   Created: src/modules/users/{users.service.ts, users.repository.ts, users.routes.ts, users.schema.ts}
            src/modules/users/users.service.test.ts (scaffold, it.todo only)
   Modified: src/server.ts (route mount), prisma/schema.prisma (User model)

➡️ Next: Frontend sub-stage — API client + form hooks + form component scaffold.
   Continuing automatically. Reply within ~10s if you want to pause, change scope, or stop.
```

Then **proceed**. Do not block on a question. Do not call `AskUserQuestion`. The user has the conversation channel — if they reply with a stop / change instruction before the next sub-stage starts producing files, the skill yields. If they don't reply, the next sub-stage runs.

(This is the "soft gate" by design. The hard variant — stop and explicitly ask — is only triggered by the conflict path in §4.2.)

### Step 5 — Deliver

**Claude Code mode:**

1. All edits land in the working tree. The skill does **not** stage, commit, push, or branch (git integration out of scope).
2. Print the **diff summary** as a list grouped by sub-stage: created / modified / deleted.
3. Print a `git diff --stat`-style line per sub-stage if the project is a git repo; if `git` isn't available, skip silently.
4. Note any test scaffolds created (path + the `it.todo` count) so `sdlc-tests` can plan its pass.

**Chat mode:**

1. Return one artifact per file. Title each with the relative path: `src/modules/users/users.service.ts`, etc.
2. After all artifacts are emitted, print the same diff-summary list.
3. Note explicitly: "I cannot run lint / typecheck / tests in chat — please run `<commands>` locally and re-paste any failures." List the commands inferred from `package.json` if it was shared, otherwise the SDLC defaults.

In both modes:

- The task's `Status` field is **never** edited (per design — the source of truth is artifact presence, not status flips).
- Print the **possible next steps** (see §7 below) as an informational list.
- Phrase the summary and next-steps in the user's language; the code itself stays in English.

### Step 6 — Verification (Claude Code only)

After each sub-stage's inner pass, before printing the soft-gate transition (§4.3), the skill runs verification on the touched modules:

1. **Lint** — detect from `package.json` (`scripts.lint`, `eslint`, `biome`). Run scoped to the files touched in this sub-stage where possible; otherwise project-wide. If a config doesn't exist, skip and note "no lint config detected."
2. **Typecheck** — run `tsc --noEmit` (or the project's `scripts.typecheck`). This is project-wide by nature; a clean output is required.
3. **Smoke run of existing unit tests** — run `vitest run --changed` if available, otherwise filter by the touched module path (`vitest run src/modules/users`). The skill **does not** run the full suite (slow), and **does not** write new tests (that is `sdlc-tests`).

If any of the three fails:

- **Lint failures** that are auto-fixable → run the project's autofix (`eslint --fix` / `biome check --write`) and re-run lint. If still failing, stop and report.
- **Typecheck failures** → fix forward (the skill caused them; the skill must repair them). If the failure is in code outside this sub-stage's Files to Touch, treat as a §4.2 conflict (the task scope is wrong).
- **Test failures** in existing tests on touched modules → analyze:
  - If the failure is because the public contract changed and the existing test was tied to the old contract — the skill **stops and reports**, since contract changes belong in the ADR / task discussion (consistency rule).
  - If the failure is unrelated flakiness → re-run once. If still failing, stop and report.
  - The skill **does not** modify existing tests to make them pass. That is a `sdlc-tests` concern with explicit user input.

In chat mode, Step 6 is replaced by a single line: "Run `<lint command>`, `<typecheck command>`, `<test command>` locally and report any failures."

---

## 5. Output

| Field | Value |
|---|---|
| Files created/modified | Inside the task's Files to Touch list, plus test scaffolds. |
| Test scaffolds | Sibling test file per production file, `describe` + `it.todo` only, no logic. |
| Status field | **Never edited.** Status remains whatever was in the task file when the skill started. |
| Lint / typecheck | Run after each sub-stage in Claude Code. Must be clean before the soft gate proceeds. |
| Smoke tests | Run on touched modules only (`--changed` or path-filtered). |
| Diff summary | Printed at the end, grouped by sub-stage. |
| Soft-gate output | 4–8 line transition summary between sub-stages, with auto-continue. |
| Hard-stop trigger | Conflict with task / ADR / spec / codebase (§4.2). |
| Code language | English (identifiers, comments, log strings). |
| Conversation language | User's language. |

---

## 6. Soft gate — what it is and isn't

The soft gate is **not** an `AskUserQuestion`. It is a printed transition message followed by a deliberate continuation. Mechanics:

- The skill emits the transition summary (§4.3 example) at the end of each sub-stage in a multi-layer task.
- The skill then begins the next sub-stage's inner pass (§4.1).
- If the user types a message before the next sub-stage emits its first file change, the skill yields immediately and treats the message as the next instruction (pause, change scope, stop, switch order, etc.). The skill does **not** finish the current write before yielding.
- If the user does not interject, the skill continues uninterrupted.

This is consistent with the gated-checkpoints model (the user *can* intervene), but the gate is structured to **not block by default**, since the sub-stage order was already approved in Step 3.

The only event that forces a hard stop is the conflict path (§4.2). Everything else is soft.

---

## 7. Possible next steps

After the implementation is delivered (or after a hard stop), present these as information, not a question. Phrase in the user's language. The user decides what runs next.

- `sdlc-tests` — fill the test scaffolds the skill produced. Typically the immediate next step.
- `sdlc-review` — review the diff for blocking issues, anti-patterns, missed edge cases. Can run before or after `sdlc-tests`.
- `sdlc-security` — when the task touches auth / input handling / secrets / external surface area.
- `sdlc-docs` — update C4 / sequence diagrams if the implementation introduced a new container, actor, or flow.
- `sdlc-impl` again — pick up the next task in the wave (typically the next must-have without unmet dependencies).
- `sdlc-adr` — if the implementation surfaced a real architectural choice that the existing ADR didn't cover (rare; usually a §4.2 hard stop).

Do **not** ask "ready to proceed?". Do **not** auto-invoke another skill. Do **not** edit `Status: Done` — that is a human / PM decision after review.

---

## 8. Out of scope (do not do)

- Writing the spec → `sdlc-ba`. The skill does not invent missing requirements.
- Architectural decisions → `sdlc-adr`. The skill does not pick libraries or change persistence.
- Task decomposition → `sdlc-pm`. The skill implements one task; it does not split or merge tasks.
- UI prototypes → `sdlc-design`. The skill rebuilds the prototype with the project's real framework, but does not produce a separate `02-prototype.html`.
- Code review / security audit → `sdlc-review` / `sdlc-security`. The skill does not produce a review of its own output.
- Test logic → `sdlc-tests`. The skill writes only `describe` + `it.todo` scaffolds.
- Diagrams → `sdlc-docs`.
- Editing `Status: Todo` / `In Progress` / `Done` — never. Status is human territory.
- Editing the task file's body — never. The task file is read-only context. If the task is wrong, route to `sdlc-pm`.
- Editing the ADR or spec — never. If they are wrong for this task, hard-stop per §4.2.
- Git commits, branches, PRs.
- Auto-promoting to the next task. Each invocation is one task.
- Running the full test suite. Smoke run on touched modules only. Full-suite execution belongs to CI and `sdlc-tests`.
- Adding new dependencies that aren't named in the ADR. If the implementation seems to need one, hard-stop per §4.2.
- Silent deviations from the task's Files to Touch. Adjacent file changes are surfaced explicitly in the diff summary.
- Writing code in any language other than English (identifiers, comments, log strings).

---

## 9. Edge cases

- **Task is too big** (Files to Touch lists >15 files or spans many unrelated FRs): stop and tell the user — "This task looks oversized for one `sdlc-impl` run. Suggest re-slicing via `sdlc-pm` before implementing." Do not start the implementation. Tasks should be sized for one impl run; if they aren't, that's a PM bug, not an impl one.
- **Task is too small** (touches 1–2 lines, no new files): implement directly, skip sub-stage decomposition, skip the soft gate. Print a one-line summary instead.
- **Ambiguous sub-stage bucketing** (a file feels half-frontend, half-UI, like a smart form component): bucket by *responsibility center of gravity*. If it owns API state → Frontend. If it is purely presentational once props arrive → UI. Mention the call in the soft-gate summary.
- **Predecessor task isn't merged** (`Depends on: T-001` but `T-001`'s files don't exist): hard-stop. Implement T-001 first, or have the user explicitly waive the dependency with a deviation note.
- **Existing tests fail on a touched module before the skill makes any change** (broken main): surface immediately. Don't proceed and risk masking the breakage. Suggest the user fix the broken main first, or explicitly ask the skill to proceed regardless.
- **Lint config conflicts with the ADR's stylistic decision**: surface as §4.2 conflict. The fix is either updating the lint config (separate task) or revising the ADR.
- **Scaffold path differs from the project's test path convention**: trust the project. If `__tests__/` is the convention, scaffold there even if the Test Plan implies sibling files.
- **Test framework mismatch** (project uses Jest, ADR/architecture says Vitest): hard-stop per §4.2 — do not introduce a parallel test runner. Either update the ADR or accept the project's existing tooling.
- **UI sub-stage but no `02-prototype.html` exists**: ask the user once — "No prototype found. Build the UI from the spec's user stories alone, or run `sdlc-design` first?" Default recommendation: `sdlc-design` first if the UI surface is non-trivial.
- **Backend sub-stage but ADR is silent on persistence / API shape**: hard-stop. Don't fabricate a data model. Route to `sdlc-adr`.
- **Project is a monorepo** (apps/, packages/): respect package boundaries from `package.json` workspaces. The task's Files to Touch should already reference the right package; if not, surface as a §4.2 conflict.
- **Two valid module locations** (`src/modules/users/` and `src/users/` both exist): pick the one the task references; if the task is silent, pick the one with more recent activity (mtime / git log) and note the choice.
- **User invokes `sdlc-impl` without a feature folder or task** ("just write me an Express endpoint that does X"): outside the workflow. Either point to `sdlc-pm` to formalize the task, or ask the user once whether to proceed in "freestyle" mode (no task file, no scaffolds, no Status protection) — only proceed in freestyle if explicitly confirmed.
- **Soft-gate is "interrupted" mid-write** (user types a stop instruction while a file is being written): finish the *current file's atomic write* (don't leave a half-written file) and then yield. Print the partial diff summary up to that point.
- **Verification fails after the last sub-stage**: don't print "Done." Print the failure clearly, list the commands the user can run to reproduce, and wait. Don't auto-revert — that is the user's call.
- **Multi-task batch request** ("implement T-001, T-002, T-003 in a row"): one task per invocation. After T-001 finishes, the skill prints next-steps and stops; the user re-invokes for T-002. This matches the architecture's gated philosophy and avoids context bloat.
- **Task's Definition of Done includes a checkbox the skill cannot verify** (e.g. "manual QA pass"): mark the box `[ ]` (unchecked) in the diff summary's notes — but again, the skill **does not edit the task file itself** to flip checkboxes. It only reports.
- **Chat mode + the user pastes a partial codebase**: implement against what was shared, explicitly flag every file referenced but not visible. Don't invent the contents of unseen files.
- **Scaffolded test file already exists from a prior run** (resumed implementation): read it first; preserve any `it` lines the user added; only add new `it.todo` lines for Test Plan items not yet covered. Never overwrite.

---

## 10. Template reference

There is no template file for this skill (output is code, format too variable for a single template). All formatting conventions are inherited from the existing codebase via the §2 scan.

The skill reads the real task file from `docs/features/FEAT-NNN-<slug>/04-tasks/T-NNN-<slug>.md` as its prerequisite input. No template is consulted — the task is a runtime artifact, not a template.
