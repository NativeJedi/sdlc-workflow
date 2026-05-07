---
name: sdlc-ba
description: Use this skill when the user has a raw product idea, feature request, or vague problem statement and needs a structured Feature Spec (problem, users, user stories, FR, NFR, product tradeoffs, acceptance criteria). Triggers include "write a feature spec", "do business analysis", "act as a BA", "draft requirements for…", "I have an idea for…", "let's spec out…", "BA on this feature", "sdlc-ba". Produces 01-feature-spec.md inside docs/features/FEAT-NNN-{slug}/ in Claude Code, or as an inline markdown artifact in chat. Does NOT make technical decisions (those belong to sdlc-adr), does NOT decompose into tasks (sdlc-pm), does NOT design UI (sdlc-design).
---

# sdlc-ba — Business Analyst

This skill turns a raw product idea into a **Feature Spec** — a structured document that captures the *what* and *why*, never the *how*.

The skill is intentionally narrow: it produces one artifact (`01-feature-spec.md`), then stops. It does not auto-chain into design, ADR, or task breakdown.

---

## 1. Trigger

Activate when the user signals any of:

- A raw idea: "I want to build…", "users keep asking for…", "we need a way to…".
- An explicit BA request: "do BA on this", "write a feature spec", "act as business analyst", "draft requirements", "sdlc-ba".
- A continuation: "let's spec out the auth feature", "start the spec for FEAT-003".

Do **not** activate when the user asks for technical decisions (route to `sdlc-adr`), task breakdown (`sdlc-pm`), code (`sdlc-impl`), or UI prototypes (`sdlc-design`). If the request is ambiguous between BA and an adjacent skill, ask the user once.

---

## 2. Inputs / prerequisites

**Required:** a raw idea or problem statement from the user. That's the only hard requirement.

**Optional context** to use if available:
- Existing feature folder (when continuing or revising a spec).
- Related features (other folders under `docs/features/`).
- Domain notes the user pastes inline.

**Missing input handling:** if no idea is given (e.g. user just types "run BA"), ask once: "What feature or problem should we spec out?"

---

## 3. Environment detection

Run this check at the start of every invocation:

1. Can you create files via the Write tool **and** does the working directory contain `docs/features/` (or `package.json` / `.git` / `src/`)?
   → **Claude Code mode**. Artifacts are written to disk.
2. Otherwise:
   → **Chat mode**. Artifact is delivered inline as a markdown artifact in the conversation.

If ambiguous (e.g. tools are present but no project markers), ask the user once: "Should I write the spec to disk or return it inline?" Remember the answer for the rest of the turn.

---

## 4. Execution

Same logical flow in both environments. Only step 5 (delivery) differs.

### Step 1 — Identify the feature

Determine **which feature** this spec belongs to. Three cases:

**Case A — User names a feature explicitly** ("spec for FEAT-003", "the auth feature"):
- Claude Code: locate the folder under `docs/features/`. If it exists and already contains `01-feature-spec.md`, ask the user: "A spec already exists for FEAT-003. Update it, replace it, or create a new feature?"
- Chat: use the named feature as the artifact title; if the spec was already produced earlier in the conversation, ask the same question.

**Case B — User describes a new idea**:
- Derive a kebab-case slug from the idea (3–5 words, lowercase, hyphenated). Examples: `user-auth`, `password-reset`, `csv-export`, `team-billing`.
- Claude Code: scan `docs/features/` for the highest existing `FEAT-NNN-…` folder. The new ID is `FEAT-<NNN+1>` (zero-padded to 3 digits, starting from `FEAT-001`).
- **Always confirm with the user before creating the folder**: "I'll create `FEAT-007-team-billing/` under `docs/features/`. OK, or want a different slug/number?"
- Chat: present the proposed `FEAT-NNN-slug` in the artifact header and let the user override.

**Case C — User is vague about scope** ("let's plan something for billing"):
- Treat as Case B but ask the slug/scope question as part of the clarification batch in Step 3.

### Step 2 — Assess input quality

Read what the user provided. Check whether each of these is **clear enough to write a useful spec**:

1. **Problem** — what pain exists today, who feels it.
2. **Primary user** — who will use the feature.
3. **Success signal** — how we'll know it worked (qualitative is fine).
4. **MVP scope** — what's in v1 vs deferred.
5. **Hard constraints** — anything non-negotiable (regulatory, integrations, deadlines).

If two or more of these are unclear → go to Step 3 (clarify). If at most one is unclear → proceed to Step 4 and mark the gap as an Open Question.

### Step 3 — Clarify (only if input is too vague)

Ask **3–7 questions in a single batch** using `AskUserQuestion`. Pick only the questions that target real gaps from Step 2 — never ask about things the user already covered.

Question-design rules:
- Where the answer space is small and known, offer multiple-choice options.
- Where the answer is open-ended (e.g. "describe the primary user"), still use `AskUserQuestion` with options like "Describe in free text" so the user can type freely.
- Keep questions short. One concept per question.
- Phrase them in the user's language (Ukrainian if the user writes in Ukrainian, English otherwise).

Examples of well-targeted questions:
- "Who is the primary user of this feature?" — options: end-customer / internal admin / API consumer / multiple.
- "What's the core problem in one sentence?" — open-ended.
- "What's must-have for v1?" — multi-select of capabilities you've inferred from the idea.
- "Any hard constraints?" — multi-select of likely candidates (deadline, regulation, must integrate with X) + free text.
- "How will we know v1 worked?" — open-ended.

Do **not** loop. One batch, then move on. Anything still unclear becomes an entry in §11 Open Questions of the spec.

### Step 4 — Write the spec

Produce `01-feature-spec.md` strictly following the template at `references/feature-spec.md` (next to this `SKILL.md`). Section-by-section guidance:

| Section | Rule |
|---|---|
| Header (Feature ID, Status, Author, dates) | `Status: Draft`. `Author: sdlc-ba`. Dates in `YYYY-MM-DD`. |
| 1. Problem Statement | 2–4 sentences. Problem only — never the solution. |
| 2. Users & Stakeholders | Concrete roles with their needs. Avoid "users" as a single bucket. |
| 3. Goals | Measurable where possible ("reduce checkout drop-off"), qualitative is acceptable when no metric exists yet. |
| 4. Non-Goals | Explicit exclusions. This is scope control, not a placeholder — leave the section out only if there are genuinely no exclusions. |
| 5. User Stories | `As a <role>, I want to <action> so that <outcome>`. Number them `US-1`, `US-2`, … |
| 6. Functional Requirements | Each row testable. Reference user stories in the `Related US` column. Use shall-language. |
| 7. Non-Functional Requirements | Cover only the categories that **actually apply** — drop empty rows. Common omissions are fine; padding is not. |
| 8. Product Tradeoffs | Scope/time/value decisions only. **Never** technical tradeoffs (DB choice, framework choice → ADR). |
| 9. Acceptance Criteria | Distinct from FRs. Phrased as user-observable outcomes, not implementation checks. |
| 10. Out of Scope | Items considered and explicitly deferred to v2/backlog. |
| 11. Open Questions | Whatever remained unresolved after Step 3. Honest > exhaustive. |
| 12. References | Related FEAT-IDs, ADRs (likely "to be created"), external links. |

Hard constraints on the artifact:
- **English** for the spec content.
- No code. No DB schemas. No tech-stack mentions. No UI mockups. Those belong to other phases.
- No effort estimates, no risk register — out of scope for this skill.
- Don't invent details. If the user said "fast", don't write "p95 < 200ms" — either ask in Step 3 or leave it as an Open Question.

### Step 5 — Deliver

**Claude Code mode:**
1. Ensure `docs/features/FEAT-NNN-<slug>/` exists (create if needed). Confirm path with user before first write (see Step 1, Case B).
2. Write `docs/features/FEAT-NNN-<slug>/01-feature-spec.md`.
3. Print the absolute path to the user.

**Chat mode:**
1. Return the spec as a markdown artifact titled `FEAT-NNN-<slug> — 01-feature-spec.md`.
2. The artifact contains the full spec; no truncation.

In both modes:
- Print a **2–4 line summary** of what was produced (number of FRs, NFRs, open questions, key tradeoffs).
- Print the **possible next steps** (see §6 below) as an informational list.
- Phrase the summary and next-steps line in the user's language; the artifact itself stays in English.

---

## 5. Output

| Field | Value |
|---|---|
| Filename | `01-feature-spec.md` |
| Path (Claude Code) | `docs/features/FEAT-NNN-<slug>/01-feature-spec.md` |
| Title (chat) | `FEAT-NNN-<slug> — 01-feature-spec.md` |
| Format | Markdown matching `references/feature-spec.md` |
| Status field | `Draft` |
| Language | English |

---

## 6. Possible next steps

After delivering the spec, present these as information, not a question. Phrase in the user's language. The user decides what runs next.

- `sdlc-design` — if the feature has UI and needs a prototype before tasks.
- `sdlc-adr` — if there are non-trivial technical decisions to make (storage, framework, integration approach).
- `sdlc-pm` — if the spec is solid enough to break into tasks (typically also needs an ADR first).
- Iterate on this spec — if the user wants to refine FRs, add user stories, or revisit tradeoffs.

Do **not** ask "ready to proceed?". Do **not** auto-invoke another skill.

---

## 7. Out of scope (do not do)

- Technical decisions (DB choice, framework, deployment topology) → `sdlc-adr`.
- Task decomposition → `sdlc-pm`.
- UI mockups, screen layouts → `sdlc-design`.
- Code → `sdlc-impl`.
- Effort estimates, risk register, RACI charts.
- Global decision logs across features (workflow is per-feature).
- Auto-numbering without confirming the proposed `FEAT-NNN-slug` first.
- Asking clarifying questions one at a time — always batch in Step 3.
- Writing the spec in any language other than English.

---

## 8. Edge cases

- **Empty repo, Claude Code mode**: if `docs/features/` doesn't exist, create it, then create `FEAT-001-<slug>/`. Confirm the path before writing.
- **User is editing an existing spec**: read the existing `01-feature-spec.md` first, then propose changes section-by-section. Don't rewrite from scratch unless asked.
- **User pastes a long product brief**: skip Step 3 unless real gaps remain; treat the brief as input directly.
- **User asks BA on someone else's code**: explain that BA produces a forward-looking spec, not reverse-engineering documentation. Suggest `sdlc-docs` for diagrams of existing systems.
- **Conflict between user input and architecture defaults** (e.g. user wants Python): record it in §12 References as a deviation note. Don't rewrite the architecture from inside a single spec.

---

## 9. Template reference

The canonical template lives at `references/feature-spec.md` next to this `SKILL.md`. The skill reads it before writing. The template ships with the skill — no project-side `docs/templates/` is required.
