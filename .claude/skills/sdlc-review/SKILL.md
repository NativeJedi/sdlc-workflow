---
name: sdlc-review
description: Use this skill when implementation has produced a diff that needs a structured code review against its task file (T-NNN-{slug}.md) and surrounding context (Feature Spec, ADR, codebase conventions). Triggers include "review T-NNN", "review this diff", "review the implementation", "act as code reviewer", "code review for FEAT-NNN", "before I merge", "is this code safe?", "sdlc-review". Reads the diff plus the corresponding task file, ADR(s), Feature Spec, and existing codebase conventions, then produces 05-review-{task-id}.md grouped into Blocking / Significant / Nits with file:line citations. In Claude Code, delegates to the sdlc-code-reviewer subagent (read-only — Read/Glob/Grep) when available; falls back to inline review if the subagent is missing. In chat, performs the review inline. After producing the artifact, the skill stays in the conversation for interactive dialogue — explaining specific findings, suggesting concrete fixes, debating tradeoffs. Never edits production code (review is read-only). Never edits the task's Status field. Never silently overrides ADR or spec — surfaces conflicts as findings. Does NOT write the spec (sdlc-ba), ADR (sdlc-adr), task breakdown (sdlc-pm), code (sdlc-impl), security audit (sdlc-security), tests (sdlc-tests), or diagrams (sdlc-docs).
---

# sdlc-review — Code Reviewer

This skill takes a **diff produced by `sdlc-impl`** (or any equivalent change set) and produces a **structured review** that calls out blocking issues, significant concerns, and nits — each anchored to specific file:line locations and grounded in the task file, the ADR(s), the Feature Spec, and the existing codebase conventions.

The skill is intentionally narrow. It reviews **one task's diff at a time** by default (the artifact is named `05-review-<task-id>.md`). It does not refactor the code itself. It does not modify the diff. It does not run the tests, write new tests, or "improve" the implementation behind the user's back. The output is a written review plus an open conversation in which the user and the skill discuss findings together.

The skill **respects existing decisions**: a finding that contradicts the spec, the ADR, or the codebase conventions without acknowledging the contradiction is itself a review bug. Every finding is anchored in something concrete — a line of code, a section of the task file, an option chosen in the ADR, a pattern already used elsewhere in the repo.

In Claude Code, the skill delegates to the **`sdlc-code-reviewer` subagent** when available. The subagent runs with **Read-only** tools (Read / Glob / Grep) — by design it cannot modify the diff, which is exactly the property a fresh-perspective reviewer should have. If the subagent file is missing, the skill performs the review inline and notes the fallback. In chat, the subagent is unavailable; the skill always reviews inline.

After delivering the artifact, the skill **stays available** for an interactive dialogue — the artifact is the structured starting point; the dialogue is where the value is. The user can ask "why is finding B-2 blocking?", "show me a fix for S-4", "I disagree with N-1 — change my mind", and the skill engages substantively from the same context it just used to produce the review.

---

## 1. Trigger

Activate when the user signals any of:

- An explicit review request: "review T-001", "review this diff", "review the implementation", "act as code reviewer", "code review for FEAT-003", "is this code safe?", "before I merge", "sdlc-review".
- A continuation in the workflow: "now review what `sdlc-impl` just produced", "review pass on the auth feature", "go to review on T-002".
- A scoped review request mapped to a known artifact: "review the users API change against T-002".

Do **not** activate when the user asks for product requirements (route to `sdlc-ba`), architectural decisions (`sdlc-adr`), task breakdown (`sdlc-pm`), UI prototypes (`sdlc-design`), code writing (`sdlc-impl`), security audit (`sdlc-security`), test writing (`sdlc-tests`), or diagrams (`sdlc-docs`). If the request is ambiguous between review and security audit ("is this safe?"), default to `sdlc-review` for general correctness/design and route to `sdlc-security` only when the surface area is auth / input handling / secrets / external attack surface — or run them in sequence if the user wants both.

If the user asks to review *something* with no diff and no task file in sight, the skill checks for a recent diff (Claude Code: `git diff` against the working tree or a named base; chat: the most recent code artifacts in the conversation). If none, ask once: "What should I review — paste the diff, point me to the task ID, or scope it to specific files?"

---

## 2. Inputs / prerequisites

**Required:** a **diff** to review **and** the **task file** the diff implements **and** read access to the codebase the diff touches.

- Claude Code: the diff comes from `git diff` (against `HEAD`, an explicit base, or a named branch — ask once if ambiguous), the task file is `docs/features/FEAT-NNN-<slug>/04-tasks/T-NNN-<slug>.md`, the ADR(s) are `docs/features/FEAT-NNN-<slug>/03-adr*.md`, the spec is `01-feature-spec.md`, and the working tree is reachable via Read/Glob/Grep.
- Chat: the diff is pasted (unified diff or per-file blocks), the task content is pasted (or produced earlier in the conversation), the ADR(s) and Feature Spec are pasted or referenced from earlier turns, and the surrounding code context is whatever the user shared.

**Existing-context** — actively scanned, not just optional. The review must be consistent with what is already decided and how the code is already written. The scan covers:

| Source | Where (Claude Code) | What to extract |
|---|---|---|
| Diff under review | `git diff <base>..<head>` (or working tree) | Exact lines added/removed/modified. Every finding cites file:line from this set. |
| Task file | `docs/features/FEAT-NNN-<slug>/04-tasks/T-NNN-<slug>.md` | Files to Touch (did the diff stay inside?), Implementation Plan (was it followed?), Definition of Done (is each box satisfiable from the diff?), Test Plan (do scaffolds match?), Related requirements (FR/NFR/AC the diff must satisfy). |
| ADR(s) for this feature | `docs/features/FEAT-NNN-<slug>/03-adr*.md` | Chosen tech / libraries / patterns — the diff must align with chosen, never with rejected. Known limitations — the diff should not silently re-open them. Tradeoffs — referenced when a finding is "you're paying the rejected cost". |
| Feature Spec | `docs/features/FEAT-NNN-<slug>/01-feature-spec.md` | FR / NFR / Acceptance Criteria the task traces to. The diff must satisfy the AC the task is responsible for; missing AC is a Blocking finding. |
| Sibling ADRs | `docs/features/FEAT-*/03-adr*.md` | Project-wide tech choices (DB, queue, auth, deployment) the diff must reuse rather than reintroduce alternatives. |
| Project-level architecture | `docs/c4/context.mmd`, `docs/c4/container.mmd` | Container boundaries — a diff that crosses a boundary the C4 doesn't show is at minimum a Significant finding (and likely a `sdlc-docs` follow-up). |
| Prior reviews on this feature | `docs/features/FEAT-NNN-<slug>/05-review-*.md` | Findings already raised — don't repeat them; do flag if the new diff regresses on a previously-fixed issue. |
| Codebase shape | `package.json`, `tsconfig.json`, `src/`, `prisma/schema.prisma`, `migrations/`, `.eslintrc*`, `biome.json`, `vitest.config.*`, `tailwind.config.*` | Tooling versions, module layout, lint/format/test commands, framework versions — informs whether the diff "fits". |
| Touched modules in context | files referenced by the diff but not changed | Naming, types, error handling, validation, test style — "is this how this codebase does it?" |

From the scan, build a short internal "review baseline" before opening the first finding:

- Diff scope: `<N files, +X / -Y lines, sub-stages touched (UI / Frontend / Backend)>`.
- Task ID + title: `<T-NNN — title>`.
- Files-to-Touch contract: `<inside | outside | partial>` (any file in the diff not in Files to Touch is at least a Significant finding unless trivially adjacent and surfaced honestly).
- Detected lint/format/test commands: `<list>`.
- Validation lib: `<name>`.
- Error-handling pattern: `<shape>`.
- UI primitives: `<list>`.
- Conflicts surfaced upstream by `sdlc-impl`: `<list or 'none'>` (re-check whether they remain unresolved in the diff).

**Missing input handling:**

- No diff visible → ask once: "I need the diff to review. Should I diff the working tree against `main`, against `HEAD~1`, or against an explicit base? In chat, paste it inline."
- No task file → ask once: "I don't see `T-NNN-<slug>.md`. Run `sdlc-pm` first, paste the task inline, or proceed in 'unscoped review' mode (with a quality warning that I cannot validate Definition of Done or Files-to-Touch contract)?"
- No ADR for a feature with non-trivial technical surface → ask once: "I don't see `03-adr.md`. Paste the relevant decisions, run `sdlc-adr`, or proceed without ADR (architectural alignment cannot be checked — review will be local)?"
- No Feature Spec → ask once: "I don't see `01-feature-spec.md`. Paste the spec, run `sdlc-ba`, or proceed without spec (FR/NFR/AC cannot be checked — review will skip the requirements column)?"
- Diff is empty (no changes) → stop and report. Don't fabricate findings.
- Diff contains only test files → narrow the review to test quality, scaffolding completeness, and Test Plan coverage; route to `sdlc-tests` if the user wanted production-code review.
- Diff contains only the task file or markdown docs → tell the user this isn't a code review surface; route to the appropriate skill (`sdlc-pm` for task edits, `sdlc-ba` for spec, `sdlc-adr` for ADR).

Do not silently invent the diff. Do not silently fabricate the task file. Do not silently override the ADR.

---

## 3. Environment detection

Run this check at the start of every invocation:

1. Can you read files via the Read/Glob/Grep tools, **and** does the working directory contain `docs/features/` (or `package.json` / `.git` / `src/`), **and** can a subagent be invoked?
   → **Claude Code mode**. Delegate to the `sdlc-code-reviewer` subagent if it exists; otherwise inline. Artifact is written as `docs/features/FEAT-NNN-<slug>/05-review-<task-id>.md`.
2. Files are readable but no project markers (or no subagent infrastructure):
   → **Claude Code, no-subagent fallback**. Review inline using read tools; write the artifact to disk if the feature folder exists, otherwise return inline and note it.
3. No tool execution available, only conversation:
   → **Chat mode**. Review inline against pasted context. Artifact returned as inline markdown titled `05-review-<task-id>.md`.

If ambiguous (tools present, project markers present, subagent presence unclear), proceed in Claude Code mode and probe for the subagent in §5. If the probe fails, fall back without re-asking the user.

---

## 4. Execution

The same logical flow runs in both environments. Only Step 5 (delivery channel) differs and Step 4.5 (subagent vs inline) branches.

### Step 1 — Identify the diff and task, load context, scan the codebase

1. Determine **what to review**:
   - Named explicitly ("review T-002") → use that task ID.
   - Inferred from a recent `sdlc-impl` invocation in the conversation → use the task that pass implemented.
   - Inferred from the diff's touched files matched against `Files to Touch` of existing tasks → pick the task with the highest match score.
   - Otherwise ask once.
2. Determine the **diff base**:
   - Explicit instruction wins.
   - Otherwise default to the diff between the current working tree and the most recent commit on the feature's working branch — but always print the chosen base in the artifact's metadata so the user can override.
3. Read the task file end-to-end. Internalize Files to Touch, Implementation Plan, Definition of Done, Test Plan, Notes, Related requirements.
4. Read the ADR(s) end-to-end. Note chosen options, rejected options, known limitations, tradeoffs.
5. Read the Feature Spec sections referenced by the task's `Related requirements` — the FR / NFR / AC the diff must satisfy.
6. Scan the codebase per §2 to learn the project's conventions. The review measures the diff against these conventions, not against generic "best practices".
7. Pull the diff itself: file list, hunks, line numbers. The reviewer must be able to cite `path/to/file.ts:42-58` for every finding that points at code.

### Step 2 — Decide subagent vs inline

Decision matrix:

| Signal | Outcome |
|---|---|
| Claude Code mode + `sdlc-code-reviewer` subagent is available    | **Delegate to subagent.** |
| Claude Code mode + subagent file missing | **Inline review.** Note in the artifact metadata: `Reviewer: inline (subagent not found)`. |
| Chat mode | **Inline review.** Subagents are unavailable. |
| User explicitly asks for inline ("just review it yourself") | **Inline review.** Honor the override. |

Probe whether the `sdlc-code-reviewer` subagent is available. Claude Code resolves agents from `.claude/agents/` (project-local) or `~/.claude/agents/` (user-level) — either is fine; the skill does not care which. If the agent cannot be resolved, fall back to inline review. Do not invoke a non-existent agent.

### Step 3 — Build the finding list (the actual review)

Whether running inline or via the subagent, the same review structure is produced. The reviewer reads the diff hunk-by-hunk and the surrounding context, and opens findings in three severity buckets:

- **Blocking (B)** — must be fixed before merge. Bugs, broken contracts, security holes (intentionally light-touch here — `sdlc-security` does the deep pass), violations of the ADR's chosen options, missing Definition-of-Done items that are not trivially addable, broken Files-to-Touch contract (a file outside the contract was modified without acknowledgment), regressions on previously-fixed issues from prior reviews, code that fails the spec's Acceptance Criteria.
- **Significant (S)** — should be fixed. Design issues, missing error handling, anti-patterns, parallel idioms (a second validation library introduced when one already exists), poor separation of concerns within a sub-stage, missing edge-case handling that AC implies, unjustified deviation from existing module layout, performance concerns the task should reasonably handle, accessibility gaps in UI changes that the spec's NFR calls out, missing test scaffolds that the Test Plan specified.
- **Nit (N)** — optional polish. Naming, formatting, micro-refactors, doc-comment phrasing, import order if not auto-fixed, redundant local types, minor readability passes. Nits are advisory; the reviewer never blocks merge on a nit.

Severity rubric — when a finding could plausibly land in two buckets, place it according to *what would happen if it shipped*:

- "Production breakage / data loss / security regression / contract break" → Blocking.
- "Future maintenance pain / inconsistency / reviewer-noise compound interest" → Significant.
- "I'd write it slightly differently" → Nit.

Each finding has a fixed structure (filled below in §4.4). The skill aims for **specific, citable, actionable** — never "this could be cleaner" without saying *what* and *how*.

### Step 4 — Write the artifact

The artifact is a single markdown file (or inline artifact in chat) named `05-review-<task-id>.md`. Its skeleton:

~~~markdown
# Code Review — T-NNN: <Task Title>

**Feature:** FEAT-NNN-<slug>
**Task:** T-NNN — <title>
**Reviewer:** sdlc-review (subagent: sdlc-code-reviewer | inline)
**Diff base:** <e.g. main..HEAD, or HEAD~1..HEAD, or working tree>
**Diff scope:** <N files, +X / -Y lines>
**Date:** <YYYY-MM-DD>

---

## Summary

<2–5 sentences. Lead with the verdict (e.g. "Approve with nits", "Request changes — 2 blocking", "Block — contract break with ADR §5.2"). Mention the most important finding by ID. State whether DoD is satisfiable from this diff.>

**Verdict:** Approve | Approve with nits | Request changes | Block

**Counts:** Blocking: N · Significant: N · Nits: N

---

## Definition-of-Done check

For each DoD checkbox in T-NNN-<slug>.md, state whether the diff satisfies it.

- [x] <DoD item 1> — satisfied: <evidence, file:line>
- [ ] <DoD item 2> — not satisfied: <reason, points to finding B-2>
- [~] <DoD item 3> — partial: <what's missing>

---

## Files-to-Touch contract check

| Path | In contract? | In diff? | Notes |
|---|---|---|---|
| `src/modules/users/users.service.ts` | Yes | Yes | OK |
| `src/modules/users/users.repository.ts` | Yes | Yes | OK |
| `src/server.ts` | Yes (Modify) | Yes | OK |
| `src/utils/crypto.ts` | No | Yes | Out-of-contract (see S-1) |

---

## Findings

### Blocking

#### B-1 — <one-line headline>
**Where:** `path/to/file.ts:42-58`
**What:** <2–4 sentences describing the issue precisely.>
**Why blocking:** <The specific contract / AC / ADR clause this violates. Quote where useful.>
**Suggested fix:** <Concrete direction, ideally a small code sketch.>

#### B-2 — …

### Significant

#### S-1 — …

### Nits

#### N-1 — …

---

## Alignment with the ADR

Bullet-by-bullet check that the diff aligns with the ADR's chosen options. Each bullet is either ✅ (aligned), ⚠️ (partial / drift, points to a Significant finding), or ❌ (violates, points to a Blocking finding).

- ✅ Persistence layer uses Prisma per ADR §5.1.
- ⚠️ Validation uses Zod (chosen) but introduces a parallel `assert*` helper (S-3).
- ❌ Auth bypasses the central middleware from sibling FEAT-002 ADR (B-1).

---

## Test scaffold coverage

Cross-reference the diff's test scaffolds against the task's Test Plan. Note missing scaffolds (Significant), wrong shape (Significant), or already-filled tests when only scaffolds were expected (raise as a question, not a finding).

---

## Open questions for the user

Things the reviewer cannot decide without input — typically a tradeoff the user owns. List them; do **not** turn them into findings.

- <Q1>
- <Q2>

---

## Out of scope for this review

Briefly note what was deliberately not covered:

- Deep security audit → `sdlc-security`.
- Test logic correctness → `sdlc-tests` (only scaffolds are checked here).
- Diagram updates → `sdlc-docs`.
~~~

Sections that have nothing to report should still appear with an explicit "None." rather than be omitted — this keeps reviews comparable across tasks.

### Step 4.4 — Finding card schema

Every finding card uses the same fields, in this order, no exceptions:

| Field | Required | Purpose |
|---|---|---|
| ID (`B-1`, `S-3`, `N-2`) | Yes | Stable handle for dialogue ("can you explain B-1?"). |
| Headline (one line) | Yes | The finding in 8–14 words. |
| Where | Yes (or "N/A — diff-wide") | `path:line` or `path:start-end` citation. Multi-file findings list each. |
| What | Yes | 2–4 sentences. Concrete description of the issue. |
| Why | Yes | The contract / AC / ADR / convention this violates, with a brief quote where helpful. |
| Suggested fix | Yes | Direction + small code sketch. Sketches stay short (≤ 10 lines). For nits, can be a one-liner. |
| Severity rationale | Only when ambiguous | One sentence on why this finding is in the bucket it's in (used to defuse "why is this blocking?" before it's asked). |

If a finding genuinely cannot point at a line (e.g. "the diff is missing a migration file entirely"), `Where` is `N/A — diff-wide` and the `What` field describes what should have existed.

### Step 4.5 — Run the review

**Subagent path (Claude Code, subagent present):**

1. Compose a brief for the subagent containing: the diff, the task file body, the ADR(s) body, the relevant Feature Spec sections, the codebase baseline summary built in Step 1, and a pointer to this skill's review structure.
2. Invoke the `sdlc-code-reviewer` subagent with **Read / Glob / Grep only**. The subagent is read-only by design — it cannot edit the diff.
3. The subagent returns a finding list following §4.4 schema. The skill assembles the findings into the artifact skeleton from §4.
4. The skill performs a quick sanity pass on the subagent's output:
   - Every finding has a citation or `N/A — diff-wide`.
   - No finding edits code (suggested fixes are *suggestions*, not commits).
   - Severity buckets follow the §3 rubric.
   - Blocking findings have a Why field that names the violated contract.
   - If anything is missing, the skill fills it from its own context rather than re-invoking the subagent.

**Inline path:**

1. The skill walks the diff hunk-by-hunk against the loaded context.
2. Findings are written directly into the artifact as they're discovered, in §4.4 schema.
3. After the first pass, the skill re-reads the artifact and: (a) deduplicates findings that point at the same root cause from two angles, (b) re-checks severity assignments against §3, (c) ensures every Blocking finding has its Why anchored in a quoted contract.

Both paths produce the same artifact shape. The user should not be able to tell which path was used except from the metadata line.

### Step 5 — Deliver

**Claude Code mode:**

1. Write the artifact to `docs/features/FEAT-NNN-<slug>/05-review-<task-id>.md`. If a prior review for the same task exists (`05-review-T-NNN.md`), don't overwrite — append a `## Re-review (<date>)` section to the existing file with the new findings. The original review stays as historical context.
2. Print a compact summary in the conversation: verdict + counts + the headlines of all Blocking findings. Don't dump the full artifact in chat — the user opens the file.
3. Print the **possible next steps** (see §7) as an informational list.

**Chat mode:**

1. Return the artifact as a single inline markdown artifact titled `05-review-<task-id>.md`.
2. After the artifact, print the same compact summary (verdict + counts + Blocking headlines).
3. Print the possible next steps.

In both modes:

- Phrase the conversational summary in the user's language; the artifact itself stays in English.
- The task file's `Status` field is **never** edited. Even if the verdict is "Approve", flipping `Status: Done` is a human decision after they decide to merge.
- Do not call `AskUserQuestion` after delivering the artifact. The dialogue mode in §6 is conversational, not gated.

### Step 6 — Verification (lightweight, Claude Code only)

Unlike `sdlc-impl`, this skill does not run lint/typecheck/tests — those are `sdlc-impl`'s and `sdlc-tests`' responsibilities, and a review is read-only by design. The skill performs only artifact-level checks before delivery:

1. The artifact path is correct (`docs/features/FEAT-NNN-<slug>/05-review-<task-id>.md`) and the feature folder exists.
2. Every finding has the §4.4 fields present.
3. Every Blocking finding cites a contract (task DoD / ADR / spec / sibling-ADR / convention) in its Why field.
4. The Files-to-Touch table covers every file the diff touches.
5. The DoD check covers every checkbox from the task file.
6. The ADR alignment section has at least one bullet per ADR section that the task referenced.

If any check fails, the skill repairs the artifact before delivering. It does not surface artifact-shape problems to the user.

In chat mode, only checks 2–6 apply (path is N/A).

---

## 5. Subagent delegation

This skill is one of three (`sdlc-review`, `sdlc-security`, `sdlc-tests`) that delegate to a Claude Code subagent.

**Subagent name:** `sdlc-code-reviewer`.
**Subagent location:** `sdlc-code-reviewer.md` in `.claude/agents/` (project-local) or `~/.claude/agents/` (user-level). Claude Code convention — **not** inside this skill package.
**Tools granted:** Read / Glob / Grep — **read-only**. The subagent cannot edit the diff. This is a feature, not a limitation: the reviewer's job is to surface issues, not silently fix them.

**When to invoke:** Only in Claude Code mode, only after probing for the subagent file (§3 step 1, §4 Step 2). A successful invocation passes:

- The diff (file list + hunks).
- The task file body.
- The ADR(s) body.
- The relevant Feature Spec sections (FR / NFR / AC).
- The codebase baseline summary.
- The §4.4 finding-card schema and the §3 severity rubric.

**When to fall back to inline:** If the `sdlc-code-reviewer` subagent cannot be resolved at either location, or the invocation errors out, or the user explicitly requests inline review. The fallback is **silent** to the user except for the artifact metadata line `Reviewer: inline (<reason>)`. The skill does not re-ask permission to fall back.

**Why a subagent at all:**

- **Fresh perspective.** The subagent's context starts clean — it doesn't carry the implementation decisions from `sdlc-impl` and is less likely to rationalize them.
- **Enforced read-only.** The tool allowlist makes "the reviewer accidentally edits the code" structurally impossible.
- **Specialized prompt.** The subagent's system prompt is tuned for review (see the `sdlc-code-reviewer` agent file), which keeps this skill's `SKILL.md` from carrying every prompt detail.

In chat, none of this applies — there are no subagents — and the skill performs the review inline using the same finding schema and severity rubric.

---

## 6. Interactive dialogue mode

The artifact is the structured starting point. The dialogue is where the value is.

After delivery, the skill **stays in the conversation** without a special invocation. The user can:

- Ask why a finding is in its severity bucket: *"Why is B-1 blocking and not significant?"* → The skill cites the Why field's contract and walks through the §3 rubric for the call.
- Ask for a concrete fix: *"Show me what S-3 should look like."* → The skill expands the Suggested-fix sketch into a more complete diff against the relevant file. The skill **does not** apply the fix — it shows the sketch and lets the user run `sdlc-impl` again if they want it written.
- Debate severity: *"I think B-2 should be a nit because we don't ship until QA passes."* → The skill engages substantively. If the user's argument moves the call (e.g. the contract is softer than the skill assumed), the skill updates the artifact and notes the change. If not, the skill explains why it disagrees and leaves the artifact intact.
- Walk through a hunk together: *"Explain what's happening in users.service.ts:120-145 and why you flagged it."* → The skill narrates the hunk and ties the narration back to the finding.
- Request scope changes: *"Re-review only the backend sub-stage."* → The skill produces a narrowed re-review section appended to the artifact.

What the dialogue is **not**:

- It is not a license to apply fixes. The skill never edits production code; the reviewer is read-only by design (and structurally so when the subagent runs).
- It is not a substitute for `sdlc-security`. Deep security questions ("walk me through every CSRF vector") are routed to `sdlc-security`.
- It is not a re-implementation channel. Big shape changes route back to `sdlc-impl` (and possibly to `sdlc-pm` if the task itself was wrong).
- It is not unbounded. If the dialogue surfaces a real disagreement that the reviewer cannot resolve (e.g. the user wants to override an ADR), the skill names the disagreement explicitly and points to the right escalation path.

The skill carries the same context for the dialogue that produced the artifact. It does not re-scan the codebase on every dialogue turn — but it **does** re-open files when a question requires precise line-level recall.

---

## 7. Possible next steps

After the artifact is delivered (and at sensible pauses in the dialogue), present these as information, not a question. Phrase in the user's language. The user decides what runs next.

- `sdlc-impl` — apply the suggested fixes for Blocking and Significant findings. The skill does not apply them itself.
- `sdlc-tests` — fill the test scaffolds the implementation produced; particularly relevant if the review surfaced missing scaffolds or wrong-shaped scaffolds.
- `sdlc-security` — run a deeper security audit. Recommended whenever the diff touches auth / input handling / secrets / external surface area, even if the review didn't flag anything as Blocking.
- `sdlc-docs` — update C4 / sequence diagrams if the diff introduced a new container, actor, or flow.
- `sdlc-review` again — re-review after fixes. The re-review appends to the existing artifact rather than overwriting.
- `sdlc-adr` — only when the dialogue surfaced a real architectural choice the existing ADR didn't cover. Rare; usually a hard escalation path.

Do **not** ask "ready to proceed?". Do **not** auto-invoke another skill. Do **not** edit the task's `Status` field — even on "Approve" verdict.

---

## 8. Out of scope (do not do)

- Editing production code → `sdlc-impl`. The skill is read-only by design.
- Editing the task file, ADR, or spec → respective owning skills. The reviewer surfaces conflicts; it does not resolve them by rewriting their sources of truth.
- Editing `Status: Todo` / `In Progress` / `Done` — never. Status is human territory.
- Running lint / typecheck / tests — those belong to `sdlc-impl` (verification) and `sdlc-tests` (test pass). The review is artifact-and-conversation only.
- Writing new tests or filling existing scaffolds → `sdlc-tests`. The skill only checks that scaffolds match the Test Plan.
- Deep security audit → `sdlc-security`. The skill flags obvious security smells in passing but does not run an OWASP pass.
- Diagram updates → `sdlc-docs`. The skill notes when a diagram update is implied; it does not produce one.
- Approving the change in any external system (PR review, ticket transition). The artifact's `Verdict:` line is informational, not a state change.
- Auto-promoting to the next task. One review per invocation; the user re-invokes for the next task.
- Re-deriving the diff from scratch when given an explicit one. Trust the user's chosen base unless it's missing.
- Inventing findings when none exist. A clean review with `Verdict: Approve` and zero findings is a valid output.
- Rewriting code in the artifact's Suggested-fix field beyond a small sketch (≤ 10 lines). Long sketches route to `sdlc-impl`.
- Repeating findings already raised in a prior review on the same feature without explicitly noting "this is a re-flag of [B-1 from previous review]".

---

## 9. Edge cases

- **Diff is huge** (> 800 changed lines or > 25 files). Don't attempt a single monolithic review. Either: (a) ask the user once to scope ("review only the backend sub-stage", "review only the new modules"), or (b) chunk the review by sub-stage (UI / Frontend / Backend) and produce one §4 skeleton per chunk in the same artifact, with a unified Summary at the top.
- **Diff spans multiple tasks.** Ask once: "This diff touches T-001 and T-002. Should I produce two artifacts (`05-review-T-001.md`, `05-review-T-002.md`) or a combined feature-level review (`05-review-FEAT-NNN.md`)?" Default recommendation depends on coupling: tightly coupled diffs → combined; independent → split.
- **No task file (legacy / external code review).** Run in "unscoped review" mode: no Files-to-Touch table, no DoD check, no Test Plan cross-reference. The review focuses purely on code quality, ADR alignment (if an ADR is reachable), and codebase consistency. The artifact is named `05-review-<short-handle>.md` with the handle confirmed once with the user. State the unscoped mode in the metadata line.
- **No spec, no ADR.** Same as above — drop the relevant sections rather than fabricating contracts to violate.
- **Diff is empty.** Stop and report. No findings, no artifact.
- **Diff contains only generated files / lockfiles.** Note this and ask once whether to proceed with a real-code review (and where the source files are) or skip.
- **Review on a re-implementation** (T-NNN was already implemented and reviewed; user re-implemented after blocking findings). Read the prior `05-review-T-NNN.md`. Each new finding either: (a) is brand new — fine, raise normally; (b) is a regression on a previously-fixed issue — raise as Blocking with explicit reference to the prior finding ID; (c) was previously raised and remains unfixed — raise with the prior ID and elevated phrasing. Append the new pass as `## Re-review (<date>)` rather than overwriting.
- **Reviewer disagrees with the ADR.** The reviewer doesn't get to override the ADR. If the diff aligns with the ADR but the reviewer thinks the ADR is wrong, that observation goes in `Open questions for the user` — not as a finding. Route to `sdlc-adr` to revise.
- **The diff fixes a previously-Blocking finding incorrectly.** Raise a new Blocking finding pointing at the prior finding ID and explaining why the fix doesn't actually fix it.
- **`Files to Touch` includes a file the diff didn't change.** Not always a finding — the task may have been over-scoped. Flag as Significant only when the un-touched file was load-bearing for the DoD or AC; otherwise note in the Files-to-Touch table without raising a finding.
- **Diff modifies a test file from a prior task** (cross-task test churn). Significant by default. Cross-task changes need explicit acknowledgment in the diff or task notes.
- **Subagent invocation fails mid-review** (timeout, error). Fall back to inline silently; note in metadata `Reviewer: inline (subagent error: <one-line reason>)`. Do not retry the subagent — re-trying tends to compound context bloat for negligible gain on review quality.
- **User pastes a partial diff in chat.** Flag the boundaries explicitly: "I can see file A and file B's hunks; file C is referenced but not visible — findings for file C are deferred." Don't invent contents.
- **User asks for an inline-only review with a verdict, no artifact.** Honor: produce the §4 content as an inline reply, skip writing the artifact file. Note that re-runs and dialogue still work, just without a persistent artifact.
- **Dialogue turn requests a code change.** Respond with a sketch or a routing message — never apply. Suggested phrasing: "I can sketch the fix here; to actually apply it, run `sdlc-impl` against this artifact."
- **Dialogue turn re-litigates a finding the user already accepted.** Engage briefly, but if the discussion goes in circles, summarize the disagreement once and move on — the artifact is not updated unless new evidence is offered.
- **Two consecutive reviews on the same task with no implementation in between** (user just wants a second pass). Run normally; the second pass appends to the same artifact. The second pass should explicitly note "no diff change since last review" if true and limit findings to those genuinely missed in the first pass.
- **Prior review's Blocking findings remain in the diff after `sdlc-impl` claims to have addressed them.** This is the most common re-review trap. Be explicit: "Prior B-1 was supposed to be fixed by changes at <file:line>; the change is present but does not resolve the contract violation because <reason>." Re-raise as Blocking with the prior ID.
- **Prototype-derived UI sub-stage.** Compare the implemented UI against `02-prototype.html` (or `.tsx`). Drift between prototype and implementation is Significant unless the spec explicitly allowed deviation. Trivial drift (color tokens, spacing units that match the design system) is a Nit.
- **Diff introduces a new dependency not named in the ADR.** Significant by default; Blocking if the ADR explicitly considered and rejected that dependency. Suggest re-running `sdlc-adr` rather than asking the reviewer to bless it.
- **Migrations / schema changes.** Always check migration reversibility, default values for new non-null columns, index coverage. Schema concerns are at minimum Significant; broken migrations are Blocking.
- **Logging / telemetry changes.** Check for accidental sensitive-data logging (PII / secrets / tokens) — Blocking if found, even though this is technically `sdlc-security` territory; the reviewer's pattern-matching threshold here is "obvious" not "exhaustive".
- **Code language is not English.** Code (identifiers, comments, log strings) must be English. Non-English code is at minimum Significant; for prominent identifiers in shared modules, it's Blocking if the codebase otherwise enforces English.
- **The conversation language is not English.** The artifact stays in English; the conversation summary and dialogue stay in the user's language. Don't mix.

---

## 10. Template reference

There is no separate template file for this skill — the review structure is described in the SKILL.md itself. The artifact skeleton in §4 is the template; the finding card schema in §4.4 is the per-finding template.

The skill reads the real task file from `docs/features/FEAT-NNN-<slug>/04-tasks/T-NNN-<slug>.md` (DoD, Files to Touch, Test Plan) as its prerequisite input — measured against the diff. No template is consulted; the task is a runtime artifact, not a template.
