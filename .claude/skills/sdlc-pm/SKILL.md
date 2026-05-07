---
name: sdlc-pm
description: Use this skill when the user has an approved Feature Spec (01-feature-spec.md) and the relevant ADR(s) (03-adr.md or 03-adr-NN-{slug}.md) and needs the feature decomposed into independently implementable tasks. Triggers include "break this into tasks", "act as a PM", "plan the work", "decompose the feature", "create task files for FEAT-NNN", "let's split this into tickets", "sdlc-pm". Produces task files in 04-tasks/T-NNN-{slug}.md inside docs/features/FEAT-NNN-{slug}/ in Claude Code, or as inline markdown artifacts in chat. Each task is sized to fit one sdlc-impl run, references FR/NFR/AC from the spec, respects ADR constraints, and declares Priority, Depends on. Delivers in two stages — first a plan (task list with priorities and dependencies) confirmed via AskUserQuestion, then the actual files. Does NOT estimate effort, does NOT identify risks, does NOT split by UI/Frontend/Backend layers (sdlc-impl decides that), does NOT write code (sdlc-impl), does NOT write the spec (sdlc-ba) or ADR (sdlc-adr).
---

# sdlc-pm — Project Manager

This skill turns a **Feature Spec** plus **ADR(s)** into a set of **task files** — concrete, scoped units of work that `sdlc-impl` can pick up one at a time.

The skill is intentionally narrow: it produces `04-tasks/T-NNN-<slug>.md` files (one per task), then stops. It does **not** auto-chain into implementation. It does **not** estimate effort. It does **not** maintain a risk register. Those rituals are deliberately out of scope for this skill.

The skill **respects existing decisions**: the Feature Spec defines *what*, the ADR defines *how* at the architectural level, and the existing codebase defines the conventions any task must follow. The skill aligns with all three or surfaces conflicts explicitly — never silently overrides.

Delivery is two-stage: a **plan checkpoint** first (the list of proposed tasks with priorities and dependencies, presented via `AskUserQuestion`), then the actual file writes. This avoids large rewrites when the user wants the breakdown reshaped.

---

## 1. Trigger

Activate when the user signals any of:

- An explicit decomposition request: "break this into tasks", "decompose the feature", "act as a PM", "plan the work", "split into tickets", "task breakdown", "sdlc-pm".
- A continuation in the workflow: "tasks for FEAT-003", "let's go to PM on the auth feature", "now do the planning pass".
- A scope-shaping question: "how should we slice this work?", "what's the order to build this?".

Do **not** activate when the user asks for product requirements (route to `sdlc-ba`), architectural decisions (`sdlc-adr`), UI prototypes (`sdlc-design`), code (`sdlc-impl`), review (`sdlc-review`), or test writing (`sdlc-tests`). If the request is ambiguous between PM and an adjacent skill, ask the user once.

The skill assumes the user has already decided that decomposition is worthwhile. It does not push back on whether the feature "deserves" task files. Once invoked, it scopes the breakdown (Step 2).

---

## 2. Inputs / prerequisites

**Required:** an existing Feature Spec **and** at least one ADR for the feature being planned.

- Claude Code: `docs/features/FEAT-NNN-<slug>/01-feature-spec.md` and `docs/features/FEAT-NNN-<slug>/03-adr.md` (or `03-adr-NN-<slug>.md` files).
- Chat: a Feature Spec and ADR(s) produced earlier in the conversation (or pasted by the user).

**Existing-context** — actively scanned, not just optional. Tasks must be consistent with what is already decided. The scan covers:

| Source | Where (Claude Code) | What to extract |
|---|---|---|
| Feature Spec | `docs/features/FEAT-NNN-<slug>/01-feature-spec.md` | FR / NFR / Acceptance Criteria / Out of Scope / Open Questions. Each task must reference the FR/NFR/AC it satisfies. |
| ADR(s) for this feature | `docs/features/FEAT-NNN-<slug>/03-adr*.md` | Chosen tech, named libraries, integration shape, data model decisions, rejected options. Tasks must align with the chosen options, not the rejected ones. |
| ADRs in **sibling** features | `docs/features/FEAT-*/03-adr*.md` | Project-wide tech choices already made (DB, queue, auth, deployment). Tasks reuse them rather than reintroducing alternatives. |
| Prototype | `docs/features/FEAT-NNN-<slug>/02-prototype.html` | Concrete screens / states / interactions. Useful for sizing UI tasks and naming screen-scoped subtasks. Optional input. |
| Existing tasks for **this** feature | `docs/features/FEAT-NNN-<slug>/04-tasks/T-*.md` | Already-written tasks that the new pass must extend, supersede, or coexist with. |
| Project-level architecture | `docs/c4/context.mmd`, `docs/c4/container.mmd` | Containers, external systems, boundaries — drives task naming for cross-container work. |
| Existing codebase | `package.json`, `src/`, `prisma/schema.prisma`, `migrations/`, `.github/workflows/` | Actual file layout and conventions. The "Files to Touch" section of each task must use real, plausible paths. |

**Missing input handling:**

- No Feature Spec → ask once: "I need an `01-feature-spec.md` to work from. Run `sdlc-ba` first, paste the spec inline, or proceed from a thin brief (with quality warning)?"
- No ADR for a feature with non-trivial technical surface → ask once: "I don't see `03-adr.md`. Run `sdlc-adr` first, paste decisions inline, or proceed without ADR (tasks may need rework once decisions are made)?" If the feature is purely product-shaped (small CRUD, no architectural choice), proceed without ADR and note this in the plan checkpoint.
- Existing tasks already cover the same feature → ask: "I see N existing tasks under `04-tasks/`. Append new tasks, replace the set, or revise specific ones?"
- **Spec has unresolved Open Questions that block a specific candidate task** → do **not** invent answers. Either drop that task from the plan and surface the gap (preferred), or push the user back to `sdlc-ba` to resolve. Non-blocking Open Questions become explicit assumptions in the affected task's §7 Notes.

Do not silently invent the spec. Do not silently override the ADR. Do not silently overwrite existing tasks.

---

## 3. Environment detection

Run this check at the start of every invocation:

1. Can you create files via the Write tool **and** does the working directory contain `docs/features/` (or `package.json` / `.git` / `src/`)?
   → **Claude Code mode**. Task files written to disk under `04-tasks/`.
2. Otherwise:
   → **Chat mode**. Task files delivered inline as separate markdown artifacts in the conversation.

If ambiguous (tools present but no project markers), ask the user once: "Should I write the task files to disk or return them inline?" Remember the answer for the rest of the turn.

---

## 4. Execution

Same logical flow in both environments. Only Step 6 (delivery) differs.

### Step 1 — Identify the feature, load the spec + ADR, scan existing context

1. Determine which feature this PM pass is for (named explicitly, inferred from the most recently discussed feature, or asked once).
2. Read `01-feature-spec.md` end-to-end. Pay special attention to:
   - **Functional Requirements** — every must-have FR ends up covered by at least one task.
   - **Non-Functional Requirements** — drive cross-cutting tasks (observability, rate limiting, accessibility pass).
   - **Acceptance Criteria** — each AC must be traceable to a Definition-of-Done line in some task.
   - **Open Questions** — surface those that block decomposition; carry the rest as assumptions.
   - **Out of Scope / Non-Goals** — do **not** create tasks for those.
3. Read every ADR file for this feature end-to-end. Extract:
   - Chosen options (tech, libraries, integration shape).
   - Consequences §9 — these often imply tasks (e.g. "introduce Prisma" → migration setup task; "switch to BullMQ" → worker scaffolding task).
   - Known limitations §7 — tasks must respect them.
4. Scan existing context per §2 — sibling ADRs, prototype, existing tasks, codebase shape. Build a short mental "context baseline":
   - Already-decided tech stack: `<list>` (e.g. "Postgres + Prisma; Express; Zod; Vitest").
   - Existing file layout signals: `<paths>` (e.g. "`src/modules/<domain>/`", "`prisma/schema.prisma`").
   - Existing tasks for this feature: `<list or 'none'>`.
   - Conflicts to surface: `<list or 'none'>`.

### Step 2 — Decompose into candidate tasks

Produce a list of candidate tasks. Slicing heuristics, in priority order:

1. **One task = one `sdlc-impl` run.** A task should be implementable end-to-end in a single focused pass without needing intermediate user decisions. If a candidate spans more than ~3 unrelated FRs or touches more than ~10–15 files, split it.
2. **Slice by feature seam, not by layer.** Do **not** pre-split into UI / Frontend / Backend tasks — `sdlc-impl` decides whether to stage sub-layers when it picks up the task. Slice along *capability* boundaries: data model, API surface, client integration, screen-or-flow, background job, third-party adapter, config/feature flag, migration.
3. **Every must-have FR must be reachable.** For each must-have FR, point to the task(s) that deliver it. Gaps are bugs in the breakdown.
4. **Cross-cutting NFRs become their own task only when non-trivial.** Logging hooks → fold into the relevant feature task. Rate limiting middleware that didn't exist before → standalone task. Accessibility pass on net-new UI → standalone task only if the prototype shows real complexity.
5. **First-mover infra tasks come first.** If the ADR introduces a new dependency (Prisma, BullMQ, Redis, OpenAPI generator), the *setup* task for it is task one; downstream tasks depend on it.
6. **Avoid micro-tasks.** A task that only edits a single config line is not worth a file. Fold it into the nearest feature task.

For each candidate, draft (mentally or as scratch notes):

- Provisional title — imperative, scoped (e.g. "Add `users` table and Prisma model", not "Database stuff").
- Slug — kebab-case, ≤ 5 words (e.g. `users-prisma-model`, `auth-api-endpoints`, `login-screen`, `password-reset-email`).
- One-line description — outcome, not how.
- Priority — `Must-have` / `Should-have` / `Nice-to-have`. Trace to the spec's §6 FR priorities and §8 product tradeoffs. Default `Must-have` only if the FR is must-have *and* the task is required for the AC.
- Depends on — explicit list of other candidate IDs that must finish first. Includes both *technical* prerequisites (the model exists before the API) and *functional* prerequisites (login exists before "remember me").
- Related FR / NFR / AC IDs from the spec.
- Files to Touch (rough) — real plausible paths, drawn from §1 codebase scan.

Number the tasks `T-001`, `T-002`, … zero-padded to 3 digits. Numbering is **scoped to the feature**, not global. If existing tasks live under `04-tasks/`, continue from the highest existing ID.

Compute the topological "waves" (Wave 1: tasks with no deps; Wave 2: tasks that only depend on Wave 1; …). Used in the summary to communicate parallelization opportunities; **not** persisted as a separate artifact.

### Step 3 — Clarify (only if needed)

Skip this step if the spec + ADR + scan provided enough signal. Otherwise, ask **2–5 questions in a single batch** using `AskUserQuestion`. Question candidates:

- "Top priority signal — which dimension shapes Must-have vs Nice-to-have most?" — options: AC traceability / spec §6 priority / time-to-MVP / risk reduction / user-stated preference.
- "Are there must-reuse modules I should slot tasks into?" — options: `<detected paths>` / no constraint / different layout (free text).
- "Any tasks I should *not* generate?" — multi-select drawn from candidates that look optional or cross-cutting.
- "How granular do you want the breakdown?" — options: coarse (≤ 5 tasks) / standard (~6–10) / fine (>10).
- "If an Open Question from the spec blocks a candidate task, prefer to:" — options: drop the task and surface the gap / write the task with assumptions in §7 Notes / push back to `sdlc-ba`.

Do **not** loop. One batch per skill invocation. Anything still unclear becomes either an explicit assumption in the affected task's §7 Notes or a deferred candidate listed in the plan checkpoint.

### Step 4 — Plan checkpoint (gated)

Before writing any file, present the proposed plan to the user via `AskUserQuestion`. The plan is a compact table — one row per candidate task — with:

- ID (`T-NNN`).
- Title.
- Priority (`M` / `S` / `N` shorthand for must / should / nice).
- Depends on (IDs only, comma-separated, or `—`).
- One-line outcome.

Below the table, name the **waves** (parallelizable groups) so the user sees what can be built in parallel.

Then present at most 4 questions to confirm or adjust:

1. "Approve this plan as-is?" — options: yes (Recommended) / change priorities / change scope (drop/merge/add) / change order (deps).
2. (Only if existing tasks present) "How to handle existing tasks under `04-tasks/`?" — options: append / replace all / revise specific ones.
3. (Only if multi-decision splits are possible) "Single set or split by sub-area?" — options: one set under `04-tasks/` (Recommended) / group by sub-area in nested folders.
4. "Write all task files now, or one at a time with confirmation?" — options: write all now (Recommended) / one-by-one.

If the user picks "change …", iterate the plan **once** in the same turn — re-show the adjusted table — then proceed. Do not loop indefinitely; if the user keeps reshaping, surface that and ask whether they'd rather revise the spec or ADR first.

### Step 5 — Write the task files

For each approved task, produce a file strictly following the template at `references/task.md` (next to this `SKILL.md`). Section-by-section guidance:

| Section | Rule |
|---|---|
| Header | `Task ID: T-NNN`. `Feature: FEAT-NNN-<slug>`. `Status: Todo`. `Priority` from the plan. `Depends on` from the plan (or `none`). |
| 1. Context | 2–4 sentences. Which slice of the feature this task delivers and why it is its own task. List `Related requirements: FR-X, FR-Y, NFR-Z, AC-N` — every task must trace to at least one FR/NFR/AC. |
| 2. Description | 2–5 sentences. Outcome, not implementation. Avoid restating the FR verbatim. |
| 3. Files to Touch | Real plausible paths only (drawn from the §1 codebase scan). Split into Create / Modify / Read for context. Include test scaffold paths under Create — `sdlc-impl` will scaffold empty test files; `sdlc-tests` fills them. |
| 4. Implementation Plan | 3–8 self-contained steps. Each step names the artifact being added/changed. No code blocks longer than ~5 lines — this is a plan, not the implementation. |
| 5. Definition of Done | Required floor checkboxes (lint clean, test scaffolds exist, no regressions, follows existing conventions) **plus** task-specific criteria — one per AC the task helps satisfy. Phrase as observable outcomes ("endpoint returns 201 with `{id, createdAt}`"), not implementation checks. |
| 6. Test Plan | Layered: unit (Vitest) for pure logic, component (RTL) for components, e2e (Playwright) only for critical flows. List **what to test**, not how. Include test data needs if the task introduces new fixtures. |
| 7. Notes & Considerations | Optional. Use for: assumptions inherited from non-blocking Open Questions; ADR-section references that constrain this task; gotchas about existing code; perf/security notes that are not big enough for a separate task. Honest > exhaustive. |
| 8. References | `Feature Spec: 01-feature-spec.md`. `ADR: 03-adr.md` with the specific section(s) that constrain this task (e.g. "§5 Decision on persistence"). Related task IDs (predecessor / sibling / successor). |

Hard constraints on task files:

- **English** for all task content.
- File name: `T-NNN-<kebab-case-slug>.md`. Slug ≤ 5 words.
- The plan's `Depends on` must be **acyclic**. If a cycle appears during drafting, that's a sign the slicing is wrong — re-slice.
- No effort estimates. No risk register. No RACI. No story points.
- Do not invent FR/NFR/AC IDs. If a task doesn't trace to any, the slicing is wrong — either the task is out of scope or the spec is missing something. Surface it instead of fabricating.
- Do not pre-split tasks into UI / Frontend / Backend sub-tasks. `sdlc-impl` makes that call.
- Tasks that touch the public API contract should reference the ADR's API decision section explicitly.

### Step 6 — Deliver

**Claude Code mode:**

1. Ensure `docs/features/FEAT-NNN-<slug>/04-tasks/` exists. Create it if missing.
2. For each task, write `04-tasks/T-NNN-<slug>.md`.
3. If a target file already exists and the user picked "append" or "revise specific" in Step 4, write only the diff or the new files — never silently overwrite.
4. Print the absolute path of each file written, plus the wave grouping.

**Chat mode:**

1. Return each task as a separate markdown artifact, titled `FEAT-NNN-<slug> — T-NNN-<slug>.md`.
2. Order artifacts by task ID. The wave grouping appears in the summary message, not duplicated as an artifact.

In both modes:

- Print a **3–6 line summary**: number of tasks, distribution by priority (M / S / N), wave count, the must-have-blocked-by-something paths (if any), and a one-line note about anything intentionally not covered (e.g. "FR-7 deferred — Open Question OQ-2 unresolved").
- Print the **possible next steps** (see §6 below) as an informational list.
- Phrase the summary and next-steps line in the user's language; the artifacts themselves stay in English.

---

## 5. Output

| Field | Value |
|---|---|
| Filename pattern | `T-NNN-<kebab-case-slug>.md` (zero-padded to 3 digits) |
| Path (Claude Code) | `docs/features/FEAT-NNN-<slug>/04-tasks/<filename>` |
| Title (chat) | `FEAT-NNN-<slug> — T-NNN-<slug>.md` |
| Format | Markdown matching `references/task.md` |
| Status field | `Todo` (the user / `sdlc-impl` flips it during execution) |
| Priority field | `Must-have` / `Should-have` / `Nice-to-have` |
| Depends on | Comma-separated `T-NNN` IDs scoped to the feature, or `none` |
| Numbering | Per-feature, continues from highest existing `T-NNN` |
| Language | English |

---

## 6. Possible next steps

After delivering the task files, present these as information, not a question. Phrase in the user's language. The user decides what runs next.

- `sdlc-impl` — pick a Wave-1 task and implement it. Typically the first must-have without dependencies.
- Iterate on the breakdown — adjust priorities, drop tasks, merge tasks, re-slice a too-big task.
- `sdlc-ba` — if decomposition exposed a real product gap that should be back-ported into the spec.
- `sdlc-adr` — if decomposition surfaced a technical decision that wasn't captured (e.g. a needed background-job mechanism).
- `sdlc-design` — if a task needs a UI surface that the existing prototype doesn't cover.
- Run `sdlc-pm` again — after the first few tasks ship, to refine the plan with what was learned.

Do **not** ask "ready to proceed?". Do **not** auto-invoke `sdlc-impl`.

---

## 7. Out of scope (do not do)

- Product requirements / user stories → `sdlc-ba`.
- Architectural decisions, library choice, integration shape → `sdlc-adr`.
- UI prototypes / screen layouts → `sdlc-design`.
- Code, real implementation → `sdlc-impl`.
- Code review / security audit → `sdlc-review` / `sdlc-security`.
- Test writing → `sdlc-tests`.
- Documentation diagrams → `sdlc-docs`.
- Effort estimation, story points, t-shirt sizes — deliberately out for this skill.
- Risk register, RACI charts, project schedules — deliberately out for this skill.
- Pre-splitting tasks by UI / Frontend / Backend layers — that is `sdlc-impl`'s call.
- Cross-feature epic planning — workflow is per-feature.
- Auto-promoting `Status: Todo` to `In Progress` / `Done` — `sdlc-impl` and humans handle status.
- Writing tasks in any language other than English.
- Inventing FR/NFR/AC IDs that don't exist in the spec.
- Silently overwriting existing tasks — must ask the append/replace/revise question.
- Generating tasks for items the spec lists in §10 Out of Scope or §4 Non-Goals.
- Writing files before the plan checkpoint (Step 4) is approved.

---

## 8. Edge cases

- **No Feature Spec exists**: ask once. Offer `sdlc-ba`, paste-inline, or thin-brief-with-quality-warning. Do not fabricate the spec.
- **No ADR exists for a feature with real architectural surface**: ask once. Offer `sdlc-adr`, paste-inline, or proceed-without-ADR-with-warning. If the feature is genuinely small (CRUD on existing models, copy changes, config tweaks), proceed without ADR and note it in the plan summary.
- **Spec has unresolved Open Questions that block a specific candidate task**: drop that task from the plan, surface the gap explicitly in the plan summary ("FR-7 not covered — depends on OQ-2"), and recommend `sdlc-ba` to resolve. Do not invent an answer.
- **Spec has unresolved Open Questions that do *not* block decomposition**: carry them as explicit assumptions in the affected task's §7 Notes. Phrase as `Assumption (OQ-N): …`.
- **ADR rejects an option that the user is now asking the tasks to use**: do not silently override the ADR. Ask once: "ADR §5 picked X over Y; task plan should align with X. Proceed, or revisit the ADR (`sdlc-adr`)?"
- **Existing tasks already cover the same feature**: ask whether to append, replace, or revise. If revise specific — read the existing file end-to-end first, then propose changes section-by-section. Don't rewrite from scratch unless asked.
- **Cyclic dependency in the candidate plan**: re-slice. A cycle is always a sign that two tasks should be one task or that one of them should be split differently.
- **Single must-have task that is too big** (touches >15 files, >3 unrelated FRs): split during Step 2. Do not rationalize a fat task as "a complex task" — `sdlc-impl` cannot run it in one pass.
- **Many micro-tasks** (each touching 1–2 lines): merge them. Aim for 4–12 tasks per feature unless the feature is genuinely large.
- **Feature is purely product-shaped, no real ADR-worthy decisions**: proceed without ADR. The plan summary should note "No ADR — feature is fully constrained by the spec and existing conventions."
- **User picks "fine granularity" but the feature is small**: warn once that the slicing will produce micro-tasks; proceed if confirmed.
- **User wants tasks for a feature where the prototype hasn't been built yet but UI is non-trivial**: warn that UI tasks may need rework once `sdlc-design` runs; either suggest `sdlc-design` first, or generate UI tasks as `Should-have` placeholders pending the prototype.
- **Conflict between spec wording and ADR consequence** (spec says "use email", ADR §9 implies SMS): name the conflict explicitly in the plan checkpoint. Do not silently pick a side. Push back to `sdlc-ba` or `sdlc-adr` to resolve.
- **Task plan must reference a sibling feature's task** (cross-feature dependency): use the fully-qualified ID `FEAT-NNN-<slug>:T-NNN` in `Depends on`. Note in §7 Notes that the dependency is cross-feature.
- **User invokes `sdlc-pm` mid-implementation**: respect the existing tasks. New tasks are appended with continued numbering. Do not retroactively rewrite the priorities of in-progress tasks unless asked.
- **Plan checkpoint produces no must-have tasks**: that is a smell — the spec's must-have FRs aren't being delivered by anything. Re-check FR coverage before proceeding.

---

## 9. Template reference

The canonical template lives at `references/task.md` next to this `SKILL.md`. The skill reads it before writing. The template ships with the skill — no project-side `docs/templates/` is required.

When writing multiple tasks in one invocation, each one independently follows the full template — no shortened "secondary task" format.
