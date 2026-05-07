---
name: sdlc-adr
description: Use this skill when the user has an approved Feature Spec (01-feature-spec.md) and needs structured Architecture Decision Records for the non-trivial technical decisions the feature requires (storage, framework, integration approach, real-time vs polling, auth model, etc.). Triggers include "write an ADR", "act as an architect", "make architectural decisions for…", "we need to decide between X and Y", "architect this feature", "design decisions for FEAT-NNN", "sdlc-adr". Produces 03-adr.md inside docs/features/FEAT-NNN-{slug}/ in Claude Code, or as an inline markdown artifact in chat. For each decision, evaluates at least three options against named drivers, recommends one, and documents tradeoffs, known limitations, and future optimization opportunities. Does NOT write the spec (sdlc-ba), does NOT design UI (sdlc-design), does NOT decompose into tasks (sdlc-pm), does NOT write code (sdlc-impl).
---

# sdlc-adr — Architect

This skill turns a **Feature Spec** into one or more **Architecture Decision Records** — structured documents that capture *what* technical choice was made, *why*, what was sacrificed, and what is deliberately out of scope.

The skill is intentionally narrow: it produces `03-adr.md` (or a small set of `03-adr-*.md` files for multi-decision features), then stops. It does not auto-chain into task breakdown or implementation. It does not estimate effort beyond the rough "Estimated effort" field per option in the template.

The skill **respects existing decisions**: prior ADRs in this feature, ADRs in sibling features, and the existing codebase shape are the source of truth. The skill aligns with them or surfaces the conflict explicitly — never silently overrides.

---

## 1. Trigger

Activate when the user signals any of:

- An explicit ADR request: "write an ADR", "act as an architect", "make architectural decisions for…", "draft an ADR for…", "sdlc-adr".
- A decision question: "should we use Postgres or DynamoDB for…", "polling vs websockets for…", "JWT vs sessions for…" — anything that asks the skill to weigh options and recommend one.
- A continuation in the workflow: "ADR for FEAT-003", "let's do the architecture for the auth feature", "architectural pass on the spec we just wrote".

Do **not** activate when the user asks for product requirements (route to `sdlc-ba`), UI prototypes (`sdlc-design`), task breakdown (`sdlc-pm`), or code (`sdlc-impl`). If the request is ambiguous between ADR and an adjacent skill, ask the user once.

The skill does **not** try to decide on its own whether a feature "needs" an ADR. The user invokes the skill when they think a decision is worth recording. Once invoked, the skill scopes which decisions to capture (Step 2).

---

## 2. Inputs / prerequisites

**Required:** an existing Feature Spec for the feature being architected.

- Claude Code: `docs/features/FEAT-NNN-<slug>/01-feature-spec.md`.
- Chat: a Feature Spec produced earlier in the conversation (or pasted by the user).

**Existing-context** — actively scanned, not just optional. ADRs must be consistent with what is already decided. The scan covers:

| Source | Where (Claude Code) | What to extract |
|---|---|---|
| Prior ADRs for **this** feature | `docs/features/FEAT-NNN-<slug>/03-adr.md`, `03-adr-*.md` | Existing decisions that the new ADR must extend, supersede, or align with. |
| ADRs in **sibling** features | `docs/features/FEAT-*/03-adr*.md` | Project-wide tech choices already made (DB, queue, auth, deployment). |
| Project-level architecture | `docs/c4/context.mmd`, `docs/c4/container.mmd` | Containers, external systems, boundaries already in place. |
| Existing codebase | `package.json`, `src/`, infra files (`Dockerfile`, `docker-compose.*`, `terraform/`, `.github/workflows/`) | Actual stack in use: DB drivers, queue clients, auth libs, deployment target. |
| Spec-side product tradeoffs | `01-feature-spec.md` §8 (Product Tradeoffs) | Constraints already accepted on the product side. Do not re-litigate them here. |
| Prototype | `02-prototype.html` | Hints at data shape, real-time needs, screen-driven complexity. Optional input. |

**Missing input handling:**

- No Feature Spec → ask once: "I need an `01-feature-spec.md` to work from. Run `sdlc-ba` first, paste the spec inline, or proceed from a thin brief (with quality warning)?"
- No prior ADRs / no codebase signals → treat as greenfield; note this in the ADR's Context section.
- Existing context **conflicts with what the spec asks for** → name the conflict explicitly and ask before proceeding (see §8 Edge cases).

Do not silently invent the spec. Do not silently override prior ADRs.

---

## 3. Environment detection

Run this check at the start of every invocation:

1. Can you create files via the Write tool **and** does the working directory contain `docs/features/` (or `package.json` / `.git` / `src/`)?
   → **Claude Code mode**. ADR(s) written to disk.
2. Otherwise:
   → **Chat mode**. ADR(s) delivered inline as markdown artifact(s) in the conversation.

If ambiguous (tools present but no project markers), ask the user once: "Should I write the ADR to disk or return it inline?" Remember the answer for the rest of the turn.

---

## 4. Execution

Same logical flow in both environments. Only Step 6 (delivery) differs.

### Step 1 — Identify the feature, load the spec, scan existing context

1. Determine which feature this ADR is for (named explicitly, inferred from the most recently discussed feature, or asked once).
2. Read `01-feature-spec.md` end-to-end. Pay special attention to:
   - **Functional Requirements** with technical implications (storage, integration, real-time).
   - **Non-Functional Requirements** — these usually drive ADR drivers (scale, latency, security, availability).
   - **Acceptance Criteria** — sometimes implies a specific shape of decision (e.g. "audit log must be tamper-evident" → storage decision).
   - **Open Questions** — many ADR-shaped questions live here.
3. Scan existing context per §2 — prior ADRs (this feature + siblings), C4 diagrams, `package.json`, infra. Build a short mental "context baseline":
   - Already-decided tech: `<list>` (e.g. "Postgres + Prisma; Express; Redis for cache").
   - Prior ADRs touching this feature: `<list or 'none'>`.
   - Conflicts to surface: `<list or 'none'>`.
4. If the spec is in `Draft` with many open questions touching technical concerns, decide whether to (a) proceed and document assumptions in the ADR Context, or (b) push back and suggest the user resolve them in `sdlc-ba` first. Default: proceed unless a blocker.

### Step 2 — Scope the decisions

Identify the discrete architectural decisions this feature requires. Each candidate decision must satisfy **all** of:

- It has **at least 3 plausible options**. If only one obvious answer exists, it is not an ADR-worthy decision — note it as an implementation detail and skip.
- The choice has **lasting consequences** (changing it later is non-trivial). Reversible micro-choices belong in code, not in an ADR.
- It is **not already decided** in a prior ADR or by clear codebase convention. If it is, reference the existing decision and skip.

Common decision shapes for this workflow's TS/Node/React stack: persistence layer (which DB, which ORM), data access pattern (repository vs query builder vs raw), state management (server state vs client state, library choice), API style (REST vs GraphQL vs RPC), real-time transport (polling vs SSE vs WebSocket), auth model (session vs JWT vs OIDC delegate), background jobs (in-process vs queue + worker), file storage (DB blob vs object storage vs CDN), validation strategy (Zod vs Yup vs hand-rolled), error handling boundary, multitenancy isolation, deployment topology, observability stack.

After listing candidate decisions, present them to the user via `AskUserQuestion`:

- "I see N decisions worth recording for this feature: `<short labels>`. Which do you want as ADRs?" — multi-select.
- Include a "describe another" option for decisions the skill missed.
- If the user picks more than one decision, ask the file-strategy question (see §5):
  - **Single file** — one `03-adr.md` with N decision sections. Recommended when decisions are tightly coupled.
  - **Multiple files** — `03-adr-01-<slug>.md`, `03-adr-02-<slug>.md`, … Recommended when decisions are independent and one might be revised later in isolation.
  - Default: single file. Only split if the user picks split or the decisions are obviously independent (e.g. "DB choice" + "real-time transport" — orthogonal).

### Step 3 — Clarify (only if needed)

For each in-scope decision, check whether the spec gives enough signal to weigh options. If not, ask **3–6 questions in a single batch** using `AskUserQuestion`. Question candidates:

- Decision drivers: "Top driver for the persistence choice?" — options: latency / scale / team familiarity / cost / time-to-market / operational simplicity.
- Hard constraints: "Any hard constraints I should respect?" — multi-select: must reuse Postgres / must run on AWS / must not introduce new infra / must support multi-region / no constraint.
- Scale targets: "Expected order of magnitude?" — small (≤1k req/day) / medium (≤1k req/s) / large (≤10k req/s) / massive (>10k req/s).
- Research depth: "Use WebSearch for current benchmarks/docs, or model knowledge only?" — model-only (faster) / WebSearch (slower, fresher).
- Stack flexibility: "Allowed to introduce a new dependency?" — yes / prefer reuse of existing / no new infra at all.

Do **not** loop. One batch per skill invocation. Anything still unclear becomes an explicit assumption in the ADR's Context section, not silent inference.

### Step 4 — Research options (hybrid)

For each in-scope decision, produce **at least 3 options** evaluated against the drivers from Step 3.

**Default: model knowledge.** Generate options from the model's knowledge of the TS/Node/React ecosystem. Fast, no external calls.

**Escalate to WebSearch when** any of:

- The user explicitly requested "research", "current benchmarks", "latest docs", or chose WebSearch in Step 3.
- The decision involves a fast-moving area where stale knowledge is risky (new vector DBs, edge runtimes, recent framework versions, rate-limit behavior of third-party APIs).
- Two or more options look equivalent on model knowledge alone, and a benchmark/doc lookup would break the tie.
- The user named a specific product whose pricing or feature set may have changed.

When using WebSearch:

- One focused query per decision. Don't sprawl. Cite sources in the ADR's References section.
- Prefer official docs, recent (≤2 years) benchmarks, and primary sources. Skip listicles and SEO blogs.
- If WebSearch fails or is unavailable, fall back to model knowledge and note the limitation.

For each option, capture: short description (2–4 sentences), pros, cons, rough effort estimate. Be honest about cons — every option has them.

### Step 5 — Write the ADR(s)

Produce ADR content strictly following the template at `references/adr.md` (next to this `SKILL.md`). Section-by-section guidance:

| Section | Rule |
|---|---|
| Header | `Status: Proposed`. `Date: YYYY-MM-DD`. `Decision makers: sdlc-adr` (unless user named a person). `ADR ID: ADR-FEAT-NNN-01` (zero-padded, numbering scoped to the feature). |
| 1. Context | 3–6 sentences. State the problem and the forces. Reference specific spec sections (`Per FR-3`, `Per NFR-2`). Note any assumptions inherited from Step 3 ambiguity. |
| 2. Decision Drivers | 3–6 drivers, ordered by importance. Each grounded in the spec or in stated user constraints — no generic "must be scalable". |
| 3. Options Considered | At least 3. Each with description, pros, cons, estimated effort. Don't pad with strawmen — every option must be genuinely plausible. |
| 4. Comparison | Fill only the rows that **discriminate** between options. Drop rows where every option scores the same — they don't help the decision. |
| 5. Decision | One option named. Rationale explicitly tied to which drivers tipped the balance. Name the deprioritized drivers honestly. |
| 6. Tradeoffs | Symmetric: gained vs sacrificed. Technical tradeoffs only — product tradeoffs belong to the spec. |
| 7. Known Limitations | What this decision deliberately does **not** solve. The boundaries of the chosen approach. Honest > exhaustive. |
| 8. Future Optimization Opportunities | Roadmap of known improvement vectors with rough triggers ("when X exceeds Y, do Z"). Not a TODO list. |
| 9. Consequences | Concrete implications for codebase / team / operations / testing. Specific package names, infra components, test infra changes. |
| 10. References | Feature Spec path, related ADRs (sibling features), external sources from Step 4 WebSearch. |

When writing **multiple** ADRs in a single invocation:

- Each ADR is self-contained and follows the full template.
- Cross-reference related decisions in §10 (`See also: ADR-FEAT-NNN-02`).
- Number them in the order presented in Step 2.
- If two decisions depend on each other, write the upstream one first.

Hard constraints on the artifact:

- **English** for the ADR content.
- Status starts as `Proposed`. The user (not the skill) flips it to `Accepted` after review.
- No code blocks longer than ~10 lines. ADRs describe decisions, not implementations.
- No vendor advocacy — every option's cons must be real.
- Don't invent benchmarks. Either cite a real source or label as "model estimate".

### Step 6 — Deliver

**Claude Code mode:**

1. Ensure `docs/features/FEAT-NNN-<slug>/` exists.
2. Decide filename(s) per Step 2 file-strategy:
   - Single decision → `03-adr.md`.
   - Multiple decisions, single file → `03-adr.md` with N decision sections under one header.
   - Multiple decisions, multiple files → `03-adr-01-<slug>.md`, `03-adr-02-<slug>.md`, …
3. If a target file already exists, ask: "Replace, append a new decision section, or write as `03-adr-NN-<slug>.md` instead?"
4. Write the file(s).
5. Print the absolute path(s) to the user.

**Chat mode:**

1. Return each ADR as a separate markdown artifact, titled `FEAT-NNN-<slug> — 03-adr.md` or `FEAT-NNN-<slug> — 03-adr-NN-<slug>.md`.
2. The artifact contains the full ADR; no truncation.

In both modes:

- Print a **2–5 line summary** of what was produced (number of decisions, chosen options, top tradeoff per decision).
- Print the **possible next steps** (see §6 below) as an informational list.
- Phrase the summary and next-steps line in the user's language; the artifact itself stays in English.

---

## 5. Output

| Field | Value |
|---|---|
| Filename (single decision) | `03-adr.md` |
| Filename (multi, single file) | `03-adr.md` |
| Filename (multi, split) | `03-adr-01-<slug>.md`, `03-adr-02-<slug>.md`, … |
| Path (Claude Code) | `docs/features/FEAT-NNN-<slug>/<filename>` |
| Title (chat) | `FEAT-NNN-<slug> — <filename>` |
| Format | Markdown matching `references/adr.md` |
| Status field | `Proposed` (user promotes to `Accepted` outside the skill) |
| ADR ID | `ADR-FEAT-NNN-NN` (zero-padded, scoped to the feature) |
| Language | English |

---

## 6. Possible next steps

After delivering the ADR(s), present these as information, not a question. Phrase in the user's language. The user decides what runs next.

- Review and promote `Status` from `Proposed` to `Accepted` — outside the skill.
- Iterate on this ADR — refine drivers, add an option, revisit the recommendation.
- `sdlc-pm` — if spec + ADR are solid, break the feature into tasks.
- `sdlc-design` — if the prototype hasn't been built yet and the ADR surfaced UI-shaped questions.
- `sdlc-ba` — if the ADR exposed a real product gap that should be back-ported into the spec.
- Run `sdlc-adr` again — if more decisions surface during implementation that warrant a record.

Do **not** ask "ready to proceed?". Do **not** auto-invoke another skill.

---

## 7. Out of scope (do not do)

- Product requirements / scope decisions → `sdlc-ba`.
- UI mockups, screen layouts → `sdlc-design`.
- Task decomposition, sequencing, priorities → `sdlc-pm`.
- Code, file scaffolding, package installs → `sdlc-impl`.
- Code review or security audit → `sdlc-review` / `sdlc-security`.
- Effort estimation beyond the per-option "Estimated effort" line in the template.
- Risk register, RACI charts, project plans.
- Cross-feature ADR registries or global decision logs (workflow is per-feature).
- Promoting status from `Proposed` to `Accepted` autonomously — that is a human decision.
- Writing the ADR in any language other than English.
- Generating options that are obviously non-viable just to hit the "≥3 options" rule. Real options only.
- Silently overriding prior ADRs — must be marked `Supersedes ADR-XXX` and the prior ADR's status flipped to `Superseded by ADR-YYY` (user confirms before flipping).

---

## 8. Edge cases

- **No Feature Spec exists**: ask once. Offer `sdlc-ba`, paste-inline, or thin-brief-with-quality-warning. Don't fabricate the spec.
- **Spec is in `Draft` with many open questions**: proceed if the open questions are not blockers for the in-scope decisions. Document each affecting assumption in the ADR's Context. If a key decision driver is fully unknown — push back and suggest resolving in `sdlc-ba`.
- **Only one obvious option exists** for a candidate decision: it is not ADR-worthy. Skip it. Mention in the summary line that you skipped trivial decisions.
- **User asks to record a decision with only 2 plausible options**: still write it, but note in §3 that the third option was "do nothing / status quo" and evaluate it honestly.
- **Prior ADR already covers the same decision in a sibling feature** (e.g. DB choice already made project-wide): do **not** re-decide. Reference the existing ADR in §10 and write only the *delta* that is feature-specific (or skip entirely).
- **New ADR contradicts an existing one**: do not silently override. Either:
  1. Align with the existing decision and document why the feature still works (preferred), or
  2. Mark new ADR as `Supersedes ADR-XXX` with a clear rationale section. Confirm with the user before flipping the prior ADR's status.
- **Codebase already uses tech that contradicts the spec's NFRs** (e.g. spec demands sub-100ms p99, but project is on a hosted DB known to be slow): name the conflict in §1 Context and either (a) make the ADR about how to *work around* it within the existing stack, or (b) make the ADR explicitly about replacing it. Don't pretend the conflict doesn't exist.
- **User asks for an ADR on a topic the spec doesn't actually require**: ask once. If they confirm, proceed but record in §1 that the decision is preemptive ("not currently driven by an FR/NFR but anticipated for…").
- **WebSearch is needed but unavailable**: fall back to model knowledge; note "WebSearch unavailable; options based on model knowledge as of cutoff" in §10 References.
- **Multiple in-scope decisions are deeply intertwined** (e.g. choosing the DB locks in the ORM): write them as one combined ADR with sub-sections, not two separate ADRs. State the coupling explicitly.
- **User wants a single ADR file but the decisions are obviously independent**: respect the user's choice but note in §1 that the decisions are independent and could be split later.
- **User invokes the skill on a feature with no spec and refuses to write one**: produce a thin ADR labeled `Status: Draft` with a prominent assumption block; warn that quality is constrained without spec grounding.
- **Spec mentions a specific tech the user dislikes** (e.g. spec says "use Redis", user wants alternatives): the spec wins on product intent, but tech choice is the ADR's job — evaluate Redis as Option A alongside ≥2 alternatives, and let the comparison decide. If Redis loses, that becomes the recommendation.

---

## 9. Template reference

The canonical template lives at `references/adr.md` next to this `SKILL.md`. The skill reads it before writing. The template ships with the skill — no project-side `docs/templates/` is required.

When writing multiple ADRs in one invocation, each one independently follows the full template — no shortened "secondary ADR" format.
