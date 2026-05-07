# SDLC Workflow — Architecture & Context

> This document describes the architecture of a modular SDLC workflow built as a set of Claude Skills. It is meant to be shared as context when building, editing, or extending any individual skill in the workflow. Read this fully before touching any specific skill.

---

## 1. Goal

Provide a developer with an end-to-end SDLC automation, where each phase of development (business analysis, design, architecture decisions, planning, implementation, code review, security audit, tests, documentation) is executed by a dedicated Claude Skill.

The workflow is **gated**: each skill produces an artifact, the user reviews it, and only then moves on to the next skill. No automatic chaining.

The workflow is **modular**: the user picks which skills to run. There is no requirement to go through all nine. A skill can be used on its own (e.g. only `sdlc-review` on external code) or as part of a longer chain.

---

## 2. High-level architecture

Nine independent skills, one per SDLC phase:

| # | Skill | Role | Main artifact |
|---|---|---|---|
| 1 | `sdlc-ba` | Business Analyst | `01-feature-spec.md` |
| 2 | `sdlc-design` | Designer | `02-prototype.html` or `02-prototype.tsx` |
| 3 | `sdlc-adr` | Architect | `03-adr.md` |
| 4 | `sdlc-pm` | Project Manager | `04-tasks/T-NNN-*.md` |
| 5 | `sdlc-impl` | Implementer | Code (TS/Node/React) |
| 6 | `sdlc-review` | Code Reviewer | `05-review-<task-id>.md` + interactive dialogue |
| 7 | `sdlc-security` | Security Auditor | `06-security-audit.md` + interactive dialogue |
| 8 | `sdlc-tests` | Test Writer | Test files (no separate report) |
| 9 | `sdlc-docs` | Technical Writer | Mermaid diagrams (C4 + Sequence) |

Each skill is **fully autonomous**. No shared "conventions" skill. Each skill's `SKILL.md` internally contains the conventions it needs (file naming, feature folder structure, tech stack defaults). Duplication is accepted as a tradeoff for independence.

Three of the skills (`sdlc-review`, `sdlc-security`, `sdlc-tests`) optionally delegate to a Claude Code **subagent** when the environment supports it. In the chat (claude.ai), those same skills do the work inline themselves.

---

## 3. Core design principles

### 3.1. Environment-adaptive
Each skill detects its environment and adapts output:

- **Claude Code** (filesystem access, real repository): artifacts are written as real files under `docs/features/FEAT-NNN-<slug>/`, except for project-level diagrams which go to `docs/c4/` and `docs/flows/`.
- **claude.ai chat** (no persistent filesystem): artifacts are returned as inline artifacts in the conversation (markdown, HTML, or Mermaid).

The skill's instruction is the same in both cases. Only the output destination differs.

### 3.2. No state file
There is no `state.md` or any external state tracking. The single source of truth is **the presence of artifacts themselves**:

- In Claude Code: files in `docs/features/FEAT-NNN-<slug>/`, ordered by numeric prefix (`01-`, `02-`, `03-`, …).
- In chat: artifacts visible in the conversation history.

If a skill needs to know what was produced before, it reads the existing artifacts. If they are missing, it asks the user.

### 3.3. Explicit prerequisites with fallback
Each skill declares what it needs as input. When the input is missing:

1. First try to locate it automatically (scan the feature folder in Claude Code; scan the conversation in chat).
2. If not found, inform the user and offer options:
   - Run the prerequisite skill first.
   - Provide the missing information manually in the chat.
   - Proceed without it (with explicit quality warning).

No skill silently fails. No skill silently assumes.

### 3.4. Gated checkpoints
After producing its artifact, each skill:

1. Delivers the artifact (file or inline).
2. Prints a short summary of what was produced.
3. Lists possible next steps as **information, not a question**. Example:
   > Possible next steps: `sdlc-adr` (if an architectural decision is needed), `sdlc-pm` (if ready to break down into tasks).

The skill does **not** ask "ready to proceed?". It does **not** auto-chain. The user decides what to run next.

### 3.5. No epic/decision-log hierarchy (KISS)
The workflow operates at one level: a **feature**. No epics above. No global decision log across features. Each feature is self-contained. This keeps the mental model simple. Can be added later if needed.

### 3.6. Consistency with existing decisions
Every skill reads what's already there before producing output. The workflow values **consistency over novelty**: a new artifact that contradicts a prior one without acknowledging it is a bug, not a feature.

Each skill, before generating, must scan and respect:

- **Prior artifacts in the same feature folder** (`docs/features/FEAT-NNN-<slug>/`) — earlier spec, prototype, ADR, tasks, reviews. Later phases align with earlier ones; if alignment is impossible, the skill surfaces the conflict to the user instead of silently overriding.
- **Project-level artifacts** — `docs/c4/`, `docs/flows/`, ADRs in sibling features. Architecture, flows, and prior decisions are shared context.
- **Existing codebase conventions** — file structure, naming, type definitions, error handling, test style. Code-producing skills (`sdlc-impl`, `sdlc-tests`) follow what's already there rather than introducing parallel idioms.
- **Existing design system & UI conventions** — components, design tokens, Tailwind/theme config, color/typography choices, layout primitives. UI-producing skills (`sdlc-design`, `sdlc-impl` UI sub-stage) reuse these instead of inventing new ones.

When a skill detects a real conflict between the user's current request and an existing decision, the protocol is:

1. Name the conflict explicitly (which file, which decision).
2. Offer options: align with the existing decision, supersede it (and document why), or escalate to the appropriate skill (`sdlc-ba` for spec changes, `sdlc-adr` for technical decisions).
3. Do not proceed silently in either direction.

This principle is a sibling of §3.2: existing artifacts are the source of truth not only for *progress* but also for *conventions*.

---

## 4. Repository layout (Claude Code)

When running in Claude Code against a real repository, artifacts follow this structure:

```
docs/
├── c4/                                 ← project-level architecture diagrams
│   ├── context.mmd                     ← updated by sdlc-docs
│   └── container.mmd                   ← updated by sdlc-docs
│
├── flows/                              ← per-flow sequence diagrams
│   ├── login/
│   │   └── sequence.mmd                ← created/updated by sdlc-docs
│   ├── checkout/
│   │   └── sequence.mmd
│   └── ...
│
└── features/
    └── FEAT-001-user-auth/             ← per-feature artifacts
        ├── 01-feature-spec.md          ← sdlc-ba
        ├── 02-prototype.html           ← sdlc-design (or .tsx)
        ├── 03-adr.md                   ← sdlc-adr
        ├── 04-tasks/                   ← sdlc-pm
        │   ├── T-001-db-schema.md
        │   ├── T-002-api-endpoints.md
        │   └── T-003-ui-components.md
        ├── 05-review-T-001.md          ← sdlc-review (one per task)
        ├── 05-review-T-002.md
        └── 06-security-audit.md        ← sdlc-security
```

**Naming conventions:**
- Feature folder: `FEAT-<zero-padded-number>-<kebab-case-slug>`.
- Task file: `T-<zero-padded-number>-<kebab-case-slug>.md`.
- Flow folder: `<kebab-case-flow-name>` (matches the user-facing flow name).
- Artifact numeric prefixes (`01-`, `02-`, …) are **fixed** per phase across all features.

In chat, the same naming scheme applies to artifact titles, even though files are not written to disk.

---

## 5. Tech stack defaults

Each skill assumes this stack unless the user overrides:

- **Language**: TypeScript
- **Runtime**: Node.js
- **Frontend**: React
- **Testing**: Vitest (unit), React Testing Library (component), Playwright (e2e)
- **Diagrams**: Mermaid (C4 via `C4Context`/`C4Container` syntax; sequence via `sequenceDiagram`)
- **Prototype format**: chosen by `sdlc-design` based on feature context (HTML+Tailwind for throwaway mocks, React+mocks for UIs that will be reused in `sdlc-impl`)

Artifact language: **English** for all artifacts.

Code language: English (default).

---

## 6. Subagents (Claude Code only)

Three skills delegate to subagents when running in Claude Code:

| Skill | Subagent | Why a subagent | Tools granted |
|---|---|---|---|
| `sdlc-review` | `code-reviewer` | Fresh perspective, isolated context, enforced read-only | Read only |
| `sdlc-security` | `security-auditor` | Specialized OWASP-focused prompt, isolated context, read-only | Read only |
| `sdlc-tests` | `test-writer` | Can dig into code for edge cases without polluting main context; can run tests | Read + Write (tests only) + Bash (test commands) |

Subagents live in `.claude/agents/` of the user's repository (Claude Code convention), **not** inside the skill packages. The skill's `SKILL.md` documents that the subagent is optional — if missing, the skill does the work inline.

In the chat, subagents are unavailable. The skill always does the work inline.

---

## 7. Per-skill contract

Every skill follows the same internal structure. When building a skill, its `SKILL.md` must cover these sections:

1. **Trigger** — when this skill activates (keywords, intent).
2. **Inputs / prerequisites** — what must exist before this skill runs. What to do if missing.
3. **Environment detection** — how to tell whether it is running in Claude Code (filesystem) or chat (inline).
4. **Execution** — step-by-step instructions for the actual work, differentiated by environment where needed.
5. **Subagent delegation** (only for review/security/tests) — when to invoke the subagent, what to pass, what to do if the subagent is not available.
6. **Output** — exact artifact format, filename convention, where it goes.
7. **Possible next steps** — informational list, not a question.
8. **Template** (if any) — in a separate file inside the skill folder.

### 7.1. Placeholder convention in `SKILL.md`

The YAML frontmatter `description` field is parsed strictly and **must not contain XML-like tokens** (anything matching `<…>`). Use this convention for placeholders:

- **Frontmatter `description`**: curly braces — `FEAT-NNN-{slug}`, `T-NNN-{slug}`, `{filename}`.
- **Markdown body**: angle brackets — `FEAT-NNN-<slug>`, `T-NNN-<slug>`, `<filename>`.

The body convention matches how the rest of this architecture document and the templates write placeholders, so paths in skill prose stay readable. The frontmatter exception exists purely because the loader treats `<…>` as XML and rejects the skill.

When creating or editing a skill, scan the frontmatter once for `<` characters and rewrite any placeholder to `{…}` form.

---

## 8. Per-skill specifications

### 8.1. `sdlc-ba` — Business Analyst

**Input:** raw idea from the user.
**Output artifact:** `01-feature-spec.md` — a **Feature Spec** document.

**Content of the Feature Spec:**
- Problem statement
- Users / stakeholders
- Goals and non-goals (explicit exclusions)
- User stories
- **Functional Requirements (FR)** — what the system must do.
- **Non-Functional Requirements (NFR)** — performance, security, accessibility, scalability, observability, etc.
- **Product tradeoffs** — scope vs time, must-have vs nice-to-have, MVP cuts. *Technical tradeoffs belong to ADR, not here.*
- Acceptance criteria
- Open questions

**Behavior:** if the input is too vague, the skill asks 3–7 clarifying questions before writing.

**Template:** `docs/templates/feature-spec.md`.

---

### 8.2. `sdlc-design` — Designer

**Input:** `01-feature-spec.md`.
**Output artifact:** `02-prototype.html` or `02-prototype.tsx`.

**Behavior:** chooses prototype format based on feature context:
- Plain HTML + Tailwind CDN — for quick throwaway mocks and rough exploration.
- React components with mocked data — for UIs that will be reused in `sdlc-impl`.

The prototype is visually complete and interactive, covering the main screens and key interactions from the spec.

For purely backend features, this step is skipped or produces an OpenAPI contract instead.

**Template:** none (prototype format is too variable for a useful template).

---

### 8.3. `sdlc-adr` — Architect

**Input:** `01-feature-spec.md`.
**Output artifact:** `03-adr.md`.

**Behavior:** identifies the key architectural decisions required by the feature. For each decision, researches at least three options, compares them across relevant axes (complexity, performance, maintainability, cost, team familiarity, TS/React ecosystem fit), states a recommendation with rationale.

**Content of the ADR (extended MADR style):**
- Context — what problem the decision solves.
- Options considered — at least three, each with pros/cons.
- Decision — what was chosen and why.
- **Tradeoffs** — what is gained, what is sacrificed by the chosen option (technical tradeoffs live here).
- **Known limitations** — what the decision does **not** cover or solve.
- **Future optimization opportunities** — places where the implementation can be improved later.
- Consequences — implications for the codebase and team.

**Template:** `docs/templates/adr.md`.

---

### 8.4. `sdlc-pm` — Project Manager

**Input:** `01-feature-spec.md` + `03-adr.md`.
**Output artifacts:** task files in `04-tasks/T-NNN-*.md`, one file per task.

**Behavior:** decomposes the feature into independently implementable tasks. For each task, defines:
- Title and description
- Context (what part of the feature this task delivers)
- Files to touch
- Step-by-step implementation plan
- Definition of done
- Test plan
- **Priority** — must-have / should-have / nice-to-have.
- **Dependencies** — explicit references to other task IDs that must be completed first.

Tasks are sized to be completed in one `sdlc-impl` run.

The skill does **not** estimate effort and does **not** identify risks (deliberately out of scope to avoid empty rituals).

**Template:** `docs/templates/task.md`.

---

### 8.5. `sdlc-impl` — Implementer

**Input:** a specific task file from `04-tasks/`, plus `03-adr.md` for architectural constraints, plus the existing codebase.

**Output:** code changes (TS/Node/React).

**Behavior:**

The skill first assesses the size of the task. If the task spans multiple layers (UI + frontend logic + backend), the skill:

1. Lays out the work in 3 sub-stages:
   - **UI** — pure presentational React components: props in, JSX out, Tailwind for styling, local `useState` for UI-only state, no API calls, no global state.
   - **Frontend** — everything between UI and backend: hooks, state management, routing, API clients, forms, client-side validation, data fetching.
   - **Backend** — API endpoints, services, DB models, migrations, server-side validation, authentication.
2. Asks the user in which order to implement the sub-stages. Common orderings:
   - Backend → Frontend → UI (API-first, mock UI later).
   - UI → Backend → Frontend (UI-first with mocks, then wire up).
   - Backend → UI → Frontend (parallel-friendly when team is split).
3. Implements one sub-stage at a time, gated between sub-stages.

For small tasks that fit a single layer, the skill implements directly without asking.

In Claude Code: modifies real files. In chat: returns code as artifacts.

The skill scaffolds test files (empty/skeleton) but does not fill them in — that is `sdlc-tests`' job.

**Template:** none.

---

### 8.6. `sdlc-review` — Code Reviewer

**Input:** the diff produced by `sdlc-impl` and the corresponding task file.

**Output artifact:** `05-review-<task-id>.md`.

**Behavior:**

Performs a structured review covering:
- **Blocking issues** — must fix before merge (bugs, broken contracts, security holes).
- **Significant concerns** — should fix (design issues, missing error handling, anti-patterns).
- **Nits** — optional polish (naming, formatting, micro-refactors).

Each finding includes specific file:line references where applicable.

In Claude Code, delegates to the `code-reviewer` subagent. In chat, performs the review inline.

After producing the artifact, the skill **stays in the conversation** and is ready to discuss the review interactively: explain reasoning behind specific findings, suggest concrete fixes, debate tradeoffs, walk through the code together. The artifact is the structured starting point; the dialogue is where the value is.

**Template:** none (review structure is described in the SKILL.md itself).

---

### 8.7. `sdlc-security` — Security Auditor

**Input:** the full feature diff (or the full feature scope if running on existing code).

**Output artifact:** `06-security-audit.md`.

**Behavior:**

Performs a security audit focused on the TS/Node/React stack:
- Authentication and authorization
- Input validation (server-side and client-side)
- XSS, CSRF
- Injection (SQL, NoSQL, command)
- Secrets handling and storage
- Dependency vulnerabilities
- CORS configuration
- Rate limiting and abuse protection
- Sensitive data exposure (logs, error messages, API responses)

Findings are categorized by severity: **Critical / High / Medium / Low / Info**.

In Claude Code, delegates to the `security-auditor` subagent. In chat, performs the audit inline.

After producing the artifact, the skill stays available for interactive discussion of findings, recommended mitigations, and tradeoffs (same pattern as `sdlc-review`).

**Template:** none.

---

### 8.8. `sdlc-tests` — Test Writer

**Input:** task file + diff (or relevant code if running on existing code).

**Output:** test files only — no separate report artifact.

**Behavior:**

Writes tests appropriate to the layer:
- **Vitest** — for pure logic and utilities.
- **React Testing Library** — for component tests.
- **Playwright** — for critical user-flow e2e tests (only when warranted).

In Claude Code, after writing tests, the skill **runs the full test suite** and verifies it is green. If tests fail:

- The skill analyzes whether the failure is **a problem in the test itself** (wrong expectation, bad mock, flaky setup) or **a real bug in the code under test**.
- If it is a problem in the test → the skill fixes the test and re-runs.
- If it is a real bug in the code → the skill **stops** and reports to the user with details: which test, what failed, what the actual vs expected behavior is, and why it looks like a code bug rather than a test bug.

The skill does not silently modify production code to make tests pass.

In chat, the skill writes tests as artifacts but cannot run them — it notes this explicitly.

In Claude Code, delegates to the `test-writer` subagent (which has Read + Write for tests + Bash for running them).

**Template:** none.

---

### 8.9. `sdlc-docs` — Technical Writer

**Input:** all feature artifacts.

**Output artifacts (Mermaid):**

| Diagram | Path | Scope |
|---|---|---|
| C4 Context | `docs/c4/context.mmd` | Project-level. Update existing or create. |
| C4 Container | `docs/c4/container.mmd` | Project-level. Update existing or create. |
| Sequence | `docs/flows/<flow-name>/sequence.mmd` | Per flow. One folder per flow. |

**Behavior:**

1. Reads project-level diagrams from `docs/c4/`. If they exist — updates them to reflect the new feature (new actors, new containers, new connections). If they don't exist — creates them from scratch using the current state of the codebase.
2. Identifies key flows introduced or modified by the feature. For each, creates or updates `docs/flows/<flow-name>/sequence.mmd`. Flow names are kebab-case and match user-facing terminology (e.g. `user-login`, `password-reset`).
3. Does **not** write prose documentation (no README updates, no CHANGELOG, no API docs). Only diagrams.

**Templates:** `docs/templates/c4-context.mmd`, `docs/templates/c4-container.mmd`, `docs/templates/sequence.mmd`.

---

## 9. Environment detection logic

Every skill performs this check at start:

```
If the assistant has access to bash_tool / file creation tools AND the working directory looks like a real project (has package.json, .git, src/, or similar):
  → Claude Code mode. Write artifacts to the appropriate path under docs/.
Else:
  → Chat mode. Deliver artifacts inline.
```

If the mode is ambiguous, the skill asks the user once and remembers the answer for the rest of the turn.

---

## 10. Feature identification

A feature is identified by its folder name (`FEAT-001-user-auth`) or, in chat, by a slug the user provides or the skill infers from the conversation.

When a skill starts:

1. If the user explicitly names a feature (`"run BA on FEAT-003"` or `"continuing with the auth feature"`) — use that.
2. Else, scan `docs/features/` (Claude Code) or the conversation (chat) for the most recently discussed feature.
3. If still unclear — ask the user.

For new features (`sdlc-ba` on a fresh idea), the skill proposes a slug based on the idea and asks the user to confirm or override.

---

## 11. What is deliberately out of scope

These are things that could be added later but are **not** part of the initial workflow:

- Global ADR/decision log across features.
- Epic/feature/task three-level hierarchy.
- Git integration (auto-commits after each phase).
- Issue tracker integration (Jira/Linear/GitHub Issues export).
- Hooks for automated validation or formatting.
- Auto-chaining between phases.
- State tracking beyond artifact presence.
- A shared "conventions" skill.
- Refactoring or bug-fix tracks (workflow is feature-oriented for now).
- Effort estimation and risk identification in `sdlc-pm`.
- Test reports as standalone artifacts.
- Prose documentation (README, CHANGELOG, API docs) in `sdlc-docs`.
- Component-level C4 diagrams, ER diagrams, deployment diagrams in `sdlc-docs`.

---

## 12. How to build one skill in isolation

When opening a new chat to build skill N:

1. Share this document as context.
2. Share the relevant template file(s) for the skill being built.
3. State: "I want to build `sdlc-<n>`. Here is the architecture and the template. Let's design and write the `SKILL.md`."
4. Discuss skill-specific questions (trigger wording, edge cases, prompt details) — this document intentionally stays high-level.
5. Produce the skill folder: `SKILL.md` + (if applicable) the template inside, + (for review/security/tests) the matching subagent file.
6. Validate with a small dry-run example.

Each skill should be buildable in one focused chat without needing any other skill to exist first. The only shared assumption is this architecture document.

---

## 13. Glossary

- **Skill** — a Claude Skill package (folder with `SKILL.md`), portable across Claude Code and claude.ai.
- **Subagent** — a separate Claude invocation in Claude Code with its own system prompt, context, and tool allowlist. Unavailable in chat.
- **Hook** — a deterministic script running on Claude Code events. Not used in this workflow.
- **Feature** — a unit of work tracked end-to-end by the workflow. Has its own folder under `docs/features/`.
- **Task** — a sub-unit of a feature, scoped to one `sdlc-impl` run. Lives in `04-tasks/` within the feature folder.
- **Flow** — a user-facing scenario (login, checkout, password reset) for which a sequence diagram is maintained under `docs/flows/<flow-name>/`.
- **Artifact** — any output of a skill (spec, prototype, ADR, task file, code, review, audit, test, diagram).
- **Gated** — the workflow property that each phase requires explicit user action to proceed to the next.
- **Feature Spec** — the artifact produced by `sdlc-ba`. Contains FR, NFR, product tradeoffs, acceptance criteria.
- **ADR** — Architecture Decision Record. Produced by `sdlc-adr`. Contains technical decisions, tradeoffs, limitations, optimization opportunities.
