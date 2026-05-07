---
name: sdlc-docs
description: Use this skill when a feature has been implemented (or is at any phase where the diagram surface needs to catch up) and the project's architecture / flow diagrams need to be created or updated. Triggers include "update the diagrams", "update C4", "draw the sequence", "diagram for FEAT-NNN", "sequence diagram for the login flow", "refresh c4/context.mmd", "sdlc-docs", "after sdlc-tests", "draw the architecture", "update flows for the auth feature". Reads all available feature artifacts (01-feature-spec.md, 02-prototype.*, 03-adr*.md, 04-tasks/T-NNN-{slug}.md, 05-review-T-NNN.md, 06-security-audit.md, the diff or implemented code) plus the existing project-level diagrams under docs/c4/ and existing per-flow diagrams under docs/flows/. Produces or updates Mermaid only — docs/c4/context.mmd, docs/c4/container.mmd, and one docs/flows/{flow-name}/sequence.mmd per affected flow. Flow names are kebab-case and match user-facing terminology (user-login, password-reset, checkout). Updates are surgical — adds new actors / containers / external systems / messages, removes ones that the current state no longer supports, leaves untouched parts of the diagram alone, and reports a per-diagram change summary. Identifies "affected flows" from the spec's user stories, the ADR's chosen integrations, the prototype's screens, and the diff's new endpoints — not from imagination. Does NOT write prose docs (no README updates, no CHANGELOG, no API reference, no inline doc comments), does NOT draw component-level C4 (Level 3), ER diagrams, deployment diagrams, or activity diagrams (out of scope). Does NOT write the spec (sdlc-ba), prototype (sdlc-design), ADR (sdlc-adr), task breakdown (sdlc-pm), code (sdlc-impl), review (sdlc-review), security audit (sdlc-security), or tests (sdlc-tests). Outputs Mermaid files only — no separate report artifact.
---

# sdlc-docs — Technical Writer (diagrams only)

This skill takes the **current state of a feature** — its spec, ADR(s), prototype, tasks, code, review, audit, tests — and produces or updates **Mermaid diagrams** that describe the system at two levels:

- **Project-level architecture** — `docs/c4/context.mmd` and `docs/c4/container.mmd`, shared across the whole project.
- **Per-flow sequence diagrams** — `docs/flows/<flow-name>/sequence.mmd`, one folder per user-facing flow.

It is the final phase of the SDLC workflow.

The skill is intentionally narrow. It writes **diagrams only**. No README updates. No CHANGELOG. No API reference. No inline doc comments. No prose paragraphs explaining what the system does. Prose documentation, component-level C4 (Level 3), ER diagrams, and deployment diagrams are deliberately out of scope. The output of this skill is exactly three categories of `.mmd` files and a short chat summary.

The skill **respects existing decisions**. The C4 diagrams are project-wide artifacts — many features share them. A new feature does not get to overwrite the project's container picture; it gets to *extend* it. The merge protocol in §5 is the heart of this skill: existing nodes / relationships are kept by default, new ones are added, removed ones are deleted only when the current state of the codebase / ADRs no longer supports them, and conflicts are surfaced to the user instead of silently resolved.

The skill **does not invent flows**. Affected flows are identified from concrete signals — user stories in the Feature Spec, integrations chosen in the ADR, screens in the prototype, endpoints / handlers / events in the diff. Flows that are not anchored to one of these signals do not get a diagram. "Looks like there should be a flow for X" is not a signal.

In Claude Code, the skill writes `.mmd` files to disk under `docs/c4/` and `docs/flows/<flow-name>/`. In chat, the skill returns each diagram as an inline Mermaid artifact. There is **no subagent** for this skill — the work is always done inline.

---

## 1. Trigger

Activate when the user signals any of:

- An explicit diagram request: "update the diagrams", "draw the architecture", "update C4", "refresh c4/context.mmd", "draw the sequence", "sequence diagram for the login flow", "diagram for FEAT-NNN", "sdlc-docs".
- A continuation in the workflow: "now update the docs for what was just shipped", "after sdlc-tests, refresh the flows", "we're done with FEAT-003 — update the diagrams".
- A scoped diagram request: "only update the container diagram", "regenerate just the password-reset sequence", "redo `docs/flows/checkout/`".
- A from-scratch project request: "the project has no C4 yet — generate the initial diagrams", "bootstrap `docs/c4/`".

Do **not** activate when the user asks for product requirements (route to `sdlc-ba`), UI prototypes (`sdlc-design`), architectural decisions (`sdlc-adr`), task breakdown (`sdlc-pm`), code (`sdlc-impl`), correctness/design review (`sdlc-review`), security audit (`sdlc-security`), or tests (`sdlc-tests`).

If the user asks for diagrams the skill does not produce — **component-level C4 (Level 3)**, **ER diagrams**, **deployment diagrams**, **state diagrams**, **activity diagrams**, **mind maps** — the skill says so explicitly (out of scope for this skill) and offers the three diagram types it does produce as alternatives where they fit.

If the user asks for **prose documentation** (a README, a CHANGELOG, an API reference, JSDoc comments, an architecture write-up) — the skill declines and routes the user to a general writing pass or to `engineering:documentation` if available. `sdlc-docs` does not write prose.

If the request is ambiguous between "draw a new diagram" and "update an existing one" — default to the merge mode (§5) and surface what was found in the diagram baseline.

---

## 2. Inputs / prerequisites

The skill is built around one **required signal** — *something concrete to diagram against* — and a layered set of context that is read but not required.

### 2.1. Required: a concrete anchor

Exactly one of these must be available. The skill never draws a diagram from imagination.

| Anchor kind | What it looks like | What it drives |
|---|---|---|
| **Feature folder** | `docs/features/FEAT-NNN-<slug>/` with at least the spec | Identifies affected flows from user stories; identifies new containers / external systems from the ADR; identifies new messages from the diff (when present). |
| **Diff** | `git diff <base>..<head>` (Claude Code) or pasted unified diff (chat) | Identifies new endpoints, handlers, queue jobs, external HTTP calls, schema changes — each is a signal that maps to a node or a message in one of the diagrams. |
| **Module / file path** | An explicit path or glob (`src/auth/`, `src/checkout/**`) | Treats the listed code as the surface to be reflected in the diagrams (used when there is no feature folder, e.g. legacy code being diagrammed for the first time). |
| **From-scratch project bootstrap** | The user explicitly asks for initial C4 from the codebase | Reads `package.json`, top-level folders, and any infrastructure config to seed the first version of `context.mmd` and `container.mmd`. No flows are auto-generated in this mode — flows require a feature anchor or an explicit user-named flow. |

The feature folder is the strongest anchor (it provides spec + ADR + tasks + diff). Diff and module scopes are progressively weaker. From-scratch bootstrap is the weakest and the only mode in which the skill is tolerated to *infer* containers from the codebase shape — and even then, it presents what it inferred to the user as a plan before writing (§5.2).

### 2.2. Existing-context — read, scanned, never silently overridden

The skill scans the surrounding context before drawing. The scan is broader than for any other phase because every prior artifact in the feature folder is potentially diagram-relevant.

| Source | Where (Claude Code) | What to extract |
|---|---|---|
| Feature Spec | `docs/features/FEAT-NNN-<slug>/01-feature-spec.md` | **User stories** — each one is a flow candidate. **Stakeholders** — drive `Person(...)` nodes in C4 Context. **NFRs** — security/observability/scaling NFRs hint at external systems (SIEM, analytics, CDN, queue) that may need to appear in C4. |
| Prototype | `docs/features/FEAT-NNN-<slug>/02-prototype.html` or `02-prototype.tsx` | **Screens / routes** — entry points for sequence flows. **Visible interactions** — clicks / submits that map to messages in a sequence diagram. |
| ADR(s) | `docs/features/FEAT-NNN-<slug>/03-adr*.md` and sibling features' ADRs | **Chosen options** for tech / integrations — every chosen external service (auth provider, email service, payment gateway, queue, cache, observability sink) maps to a `System_Ext(...)` in C4 Context and a `Container(...)` / `System_Ext(...)` in C4 Container. **Rejected options are explicitly NOT drawn.** **Known limitations** are noted in the chat summary if they imply a missing diagram element. |
| Tasks | `docs/features/FEAT-NNN-<slug>/04-tasks/T-NNN-<slug>.md` | **Files-to-Touch** — gives the implementation surface; new files in `src/api/` / `src/workers/` / `src/jobs/` hint at new containers or new endpoints. **Test Plan** — sometimes names flows explicitly. |
| Diff (when present) | `git diff <base>..<head>` | The most concrete signal: new HTTP routes, new queue producers/consumers, new external HTTP clients, new env vars (often = new external system), new DB tables (do **not** generate ER diagrams — but a brand-new container may be implied if the schema is owned by a new service). |
| Review / Audit | `05-review-T-*.md`, `06-security-audit.md` | Read for **explicit notes about architecture**, e.g. a review that says "the worker now talks directly to S3 — the container diagram should reflect that". The skill does not invent diagram changes from a review's mention of "consider extracting X" — only from confirmed/done changes. |
| Existing project-level diagrams | `docs/c4/context.mmd`, `docs/c4/container.mmd` | The current state of the project diagrams. Drives the **merge** in §5 — what to keep, what to add, what to remove. |
| Existing flow diagrams | `docs/flows/*/sequence.mmd` | The current state of per-flow diagrams. Drives per-flow merge (§5.4): kept-as-is, updated-in-place, deleted (only when the flow no longer exists), or added (a new flow). |
| Codebase shape | `package.json`, `src/`, `prisma/schema.prisma`, infra config (`docker-compose.yml`, `Dockerfile`, `.github/workflows/`, `terraform/`, `k8s/`) | Confirms which containers actually exist (web app, API, worker, scheduled jobs, DB, cache, queue). Used to validate that what the diagrams claim matches what is actually there. |
| Templates | `references/c4-context.mmd`, `references/c4-container.mmd`, `references/sequence.mmd` (next to this `SKILL.md`) | The starting shape when a diagram does not yet exist. Templates are *reference shapes*, not literal output — placeholder names and relationships are replaced with the real ones from the project. |

From the scan, build a short internal "diagram baseline" before writing the first line of Mermaid. The baseline is the header of the chat summary (§7) so the user can see what the skill saw.

The baseline contains:

- Anchor kind: `feature | diff | module | bootstrap`.
- Anchor identity: `FEAT-NNN | diff <base>..<head> | <path> | bootstrap`.
- Affected flows detected: `<list of kebab-case names>` (or `none` for bootstrap mode).
- Existing project diagrams: `context.mmd: present|absent`, `container.mmd: present|absent`.
- Existing flow diagrams: `<list of <flow-name>/sequence.mmd>` (or `none`).
- New containers / external systems implied by the anchor: `<short list>` (e.g. "Stripe added per ADR §4.2", "BullMQ worker added per T-005").
- Removals implied by the anchor: `<short list or 'none'>` (e.g. "drop `Mailgun` external — replaced by SendGrid in 03-adr-02-email.md").
- Conflicts to surface: `<list or 'none'>` (e.g. "container.mmd shows MongoDB; ADR-01 mandates Postgres — surface to user").

### 2.3. Missing-input handling

Each missing input has a single ask. The skill never silently invents the input.

- **No anchor at all** (no feature folder, no diff, no module path, and the user did not ask for bootstrap) → ask once:
  > I need something concrete to diagram against. Pick one:
  > - **Feature folder** — name a `FEAT-NNN-<slug>` and I'll read its spec, ADR(s), tasks, and any review/audit/tests.
  > - **Diff** — let me diff the working tree against `main`, or paste a unified diff.
  > - **Module / path** — give me a path or glob (e.g. `src/auth/`).
  > - **Bootstrap** — say "bootstrap C4 from the codebase" if the project has no diagrams yet.
- **Feature folder named but does not exist** → ask once:
  > I don't see `docs/features/FEAT-NNN-<slug>/`. Options: (a) point me at the right slug, (b) drop the feature framing — name a diff or a module instead, (c) run `sdlc-ba` first to create the feature folder.
- **Feature folder exists but is empty** (no spec, no ADR, no tasks) → ask once:
  > The feature folder exists but contains no spec or ADR. Without those, I have no anchor for which flows to draw or which containers to update. Options: (a) run `sdlc-ba` and `sdlc-adr` first, (b) name a diff or a module instead, (c) bootstrap from the codebase only and skip per-flow diagrams.
- **No `docs/c4/` directory yet, on a project that has been around for a while** → don't fail. Note in the baseline that project-level diagrams will be created from scratch and surface this as a deliberate choice in the chat summary so the user can review the inferred container set carefully.
- **`references/*.mmd` missing inside the skill folder** → don't fail. Use built-in template shapes (the structure shown in §6) and note the absence. The templates ship with the skill and should be present; if they aren't, the skill works but flags the anomaly.
- **Diff is empty** (working tree is clean against the named base) and the anchor is `diff` → stop and report:
  > The diff is empty against `<base>`. There is nothing to reflect in the diagrams from the diff. Pick a different anchor (feature / module / bootstrap) or change the base.
- **Diff contains only test files / docs / config** → fall back to feature-folder mode if a feature is in scope; otherwise stop and report. Diagrams describe runtime behavior; pure test/doc/config changes do not move the diagram.
- **Conflicting evidence between artifacts** (e.g. ADR says Postgres, container.mmd shows MongoDB, codebase has Prisma schema for Postgres) → do **not** silently pick a winner. Surface the conflict in the chat summary with the three sources and ask the user which is canonical. The skill writes nothing until the conflict is resolved.

The skill does not silently widen the scope. "Update the diagrams" with no feature in flight does not become "redraw the entire project" without an explicit ask.

---

## 3. Environment detection

Run this check at the start of every invocation:

1. Can you read files via the Read/Glob/Grep tools, **and** does the working directory contain `docs/` (or `package.json` / `.git` / `src/`), **and** can you write files?
   → **Claude Code mode.** Diagrams are written to disk under `docs/c4/` and `docs/flows/<flow-name>/`. The skill creates missing parent directories as needed.
2. Files are readable but not writable, or only conversation tools are available:
   → **Chat mode.** Each diagram is returned as an inline Mermaid artifact, named with the path it would have on disk (`docs/c4/context.mmd`, `docs/flows/user-login/sequence.mmd`), so the user can drop it into the right place themselves.

If the mode is ambiguous (tools present, project markers absent), ask the user once: "Should I write the diagrams to disk under `docs/`, or return them inline?". Remember the answer for the rest of the turn.

There is **no Bash dependency** — this skill does not run anything. It does not validate Mermaid syntax by piping through a renderer; the syntax rules are encoded in §6 and applied by inspection.

---

## 4. What goes into which diagram

This is the substantive part of the skill. Every node, every relationship, every message must answer: "*which diagram does this belong to, and why this one?*". Misplacing things across the three diagrams is the most common failure mode.

### 4.1. C4 Context (`docs/c4/context.mmd`) — Level 1

**Audience.** A non-technical reader (PM, exec, new-hire on day 1) who needs a one-glance picture of "what is this system, who uses it, what does it talk to".

**What goes in:**

- **Persons** — the human actors who interact with the system. Typical set: end user, admin, support, internal operator. One `Person(...)` per distinct role.
- **The system itself** — exactly one `System(...)` representing the product as a whole. The system is a single box at this level — it is not decomposed into containers.
- **External systems** — every third-party service the product interacts with: auth provider, email service, payment gateway, analytics, CDN, search, observability sink, AI/ML provider. One `System_Ext(...)` per distinct service.
- **Relationships** — `Rel(...)` between persons and the system, and between the system and each external. Each relationship has a one-clause label ("Authenticates users via", "Sends emails via") and a transport ("OAuth 2.0 / OIDC", "REST API", "HTTPS").

**What does NOT go in:**

- Internal containers (web app, API, worker, DB) — those live in C4 Container.
- Specific endpoints, queues, messages — those live in sequence diagrams.
- Components inside containers (controllers, services) — that's Level 3 and out of scope.
- Deployment topology (regions, k8s clusters, load balancers) — out of scope.

**One Context per project.** New features add `Person(...)` / `System_Ext(...)` / `Rel(...)` to the existing Context, they do not produce per-feature Contexts. If a feature does not introduce a new actor or a new external system, the Context is left untouched.

### 4.2. C4 Container (`docs/c4/container.mmd`) — Level 2

**Audience.** A developer onboarding to the system or another team that integrates with it. Answers "what are the runtime processes / data stores, and how do they talk to each other".

**What goes in:**

- **Persons** (same as Context — copied across both diagrams; this is the C4 convention).
- **External systems** (same as Context — also repeated).
- **System boundary** — `System_Boundary(system, "...") { ... }` wrapping the project's containers.
- **Containers** — independently deployable / runnable units inside the boundary:
  - Web Application (`Container(spa, ..., "React, TypeScript, Vite", ...)`)
  - API (`Container(api, ..., "Node.js, Express/Fastify, TypeScript", ...)`)
  - Background Worker (`Container(worker, ..., "Node.js, BullMQ", ...)`) — only if there actually is one.
  - Scheduled Jobs / Cron (`Container(cron, ..., ...)`) — only if there actually is one.
  - Database (`ContainerDb(db, ..., "PostgreSQL", ...)`)
  - Cache / session store (`ContainerDb(cache, ..., "Redis", ...)`)
  - Object storage (`ContainerDb(blob, ..., "S3", ...)`) — when actually used.
  - Search index (`ContainerDb(search, ..., "Elasticsearch", ...)`) — when actually used.
- **Relationships** — between persons, containers, and externals. Labels include the protocol ("JSON/HTTPS", "SQL/TCP", "Redis protocol", "S3 SDK", "Kafka", "Stripe SDK").

**What does NOT go in:**

- Components inside a container (controllers, repositories, hooks, slices) — Level 3, out of scope.
- Endpoint paths, message payload shapes, retry counts — those live in sequence diagrams.
- Speculative containers ("we *should* have a worker") — only what actually runs.
- Deployment artifacts (k8s pod, ALB, ASG) — out of scope.

**One Container per project.** Same merge discipline as Context — new feature adds, leaves the rest alone.

### 4.3. Sequence diagrams (`docs/flows/<flow-name>/sequence.mmd`) — per flow

**Audience.** A developer about to read or change the code for a specific user-facing flow. Answers "what happens, in order, when the user does X".

**Scope per file:** **one user-facing flow.** A flow is the sequence of interactions kicked off by a single user intent — login, password reset, checkout, file upload, submit invoice, run report. Flows are **kebab-case** and named with **user-facing terminology**, not internal endpoint names: `user-login` not `post-auth-login`, `password-reset` not `verify-token-and-rotate`, `checkout` not `place-order-with-payment-intent`.

**What goes in (per flow):**

- The **actor** (`actor User`) — the human or external system initiating the flow.
- The **participants** that the flow actually talks to. Use container-level granularity by default (`SPA`, `API`, `Worker`, `DB`, `Cache`, `Auth`, `Email`, `Stripe`) — the same names as in C4 Container. Drop into module-level granularity (`SessionService`, `RateLimiter`) **only** when the flow's value depends on it (e.g. when the rate limiter's behavior is the reason the flow exists).
- **Messages** — synchronous calls (`->>`), responses (`-->>`), self-calls (`->>` to self). Each message has a one-line label. Long payloads use `<br/>` to break across lines.
- **Notes** — `Note over A,B: ...` for short clarifications and for separating alternative paths.
- **Alternative paths** — typically two to three per flow: one happy path, one or two error paths (invalid input, rate limit, downstream failure). Use `Note over` headers ("Alternative: Invalid credentials") to separate them — Mermaid's `alt`/`else` is also valid but the `Note over` style is the project's default per the template.

**What does NOT go in:**

- Multiple flows in one file. Each flow is its own folder + `sequence.mmd`.
- Internal helper calls inside a single container that don't cross a boundary (e.g. a controller calling its own service). The point of a sequence diagram is interactions across containers / externals; intra-container calls belong to code, not a diagram.
- Speculative flows. If the spec mentions "we'll add 2FA later", no `2fa-enrollment` flow is drawn until 2FA actually exists in the code.
- Component-level steps (`getUserById`, `validateZodSchema`) unless that step is the load-bearing element of the flow.

**One folder per flow.** `docs/flows/user-login/sequence.mmd` is one file in a folder of the same name. The folder convention (rather than a flat `docs/flows/user-login.mmd`) leaves room for the user to add diagrams or notes alongside the sequence later — the skill does not write those, but it respects the convention.

### 4.4. The "which diagram does this belong to" decision tree

When a new piece of information lands (a new external system, a new endpoint, a new actor), walk this tree before writing anything:

```
Is it a new human role or a new third-party service?
├── Yes, new human role        → Person in BOTH context.mmd AND container.mmd.
├── Yes, new third-party       → System_Ext in BOTH context.mmd AND container.mmd.
│                                 If the project also calls it from a specific flow, add the corresponding
│                                 messages in that flow's sequence.mmd.
└── No, it is internal.

Is it a new runtime process / data store inside the project?
├── Yes (worker, scheduled job, new DB, new cache, new search index)
│                              → Container in container.mmd ONLY.
│                                 (Does NOT change context.mmd — the Context stays at one System box.)
│                                 If a flow now talks to it, update / create that flow's sequence.mmd.
└── No, it is a code-shape change.

Is it a new endpoint / message / external call within an existing flow?
├── Yes                        → Update the relevant flow's sequence.mmd (add message(s)).
│                                 Container.mmd is touched ONLY if the call introduces a new
│                                 inter-container relationship that did not exist before
│                                 (e.g. API now talks to a Worker for the first time).
└── No, it is purely internal to one container (refactor, naming).
                                → No diagram change. The skill does NOT draw component-level
                                  refactors (Level 3 is out of scope).
```

The tree is mechanical and is applied to **every** delta surfaced from §2. If a delta does not lead to a diagram change, the skill says so explicitly in the chat summary — silence is ambiguous.

---

## 5. Execution

The same logical flow runs in both environments. Steps 1–4 are identical; Step 5 (delivery channel) differs by environment.

### Step 1 — Identify anchor, load context, scan existing diagrams

1. Determine **anchor** (per §2.1):
   - Named explicitly ("draw the diagrams for FEAT-003", "update the flows for the auth feature", "redo `docs/flows/checkout/`") → use that.
   - Inferred from a recent `sdlc-impl` / `sdlc-tests` invocation in the conversation → use the same feature, but state the inference in the diagram baseline so the user can override.
   - Otherwise ask once (§2.3).
2. For a feature anchor — load **all** artifacts in the feature folder: `01-feature-spec.md`, `02-prototype.*`, `03-adr*.md`, `04-tasks/T-*.md`, `05-review-T-*.md` (any present), `06-security-audit.md` (if present). Read each end-to-end. The skill does not skim.
3. For any anchor — compute the **diff** when one is meaningfully available (`git diff <base>..<head>`) and use it as the deltas-to-reflect signal.
4. Read **existing project-level diagrams** at `docs/c4/context.mmd` and `docs/c4/container.mmd`. Parse them into an internal node / edge set. If they don't exist, mark them as bootstrap targets.
5. Read **existing per-flow diagrams** at `docs/flows/*/sequence.mmd`. Parse each into an internal participant / message set, keyed by flow folder name.
6. Read **codebase shape** to validate claims: `package.json` (deps imply integrations — `@stripe/stripe-js` → Stripe in C4; `bullmq` → Worker container; `ioredis` → Redis cache; `pg`/`prisma` → Postgres DB), top-level folders (`src/api/`, `src/workers/`, `src/jobs/`, `e2e/`), infra config when present.
7. Build the **diagram baseline** (§2.2): anchor identity, affected flows detected, existing diagrams found, additions implied, removals implied, conflicts detected.

### Step 2 — Identify affected flows and propose the change set

Affected flows come from concrete signals, in this priority order:

1. **User stories in the Feature Spec.** Each user story that crosses a system boundary is a flow candidate. "As a user, I can sign in with email + password" → `user-login`. "As a user, I can reset my password via email" → `password-reset`.
2. **Prototype screens / interactions.** A submit button in the prototype that posts to an API → a flow whose entry point is that submit. Multi-step prototypes (a stepper, a wizard) often map to one flow per terminal action.
3. **ADR's chosen integrations.** Adding Stripe → typically a `checkout` or `payment-collection` flow. Adding S3 → typically a `file-upload` or `report-export` flow.
4. **Diff signals.** New HTTP routes, new queue producers/consumers, new external HTTP clients in the diff each map to messages — and the message belongs to a flow whose name is derived from the route's user-facing intent (`POST /password/reset/request` → `password-reset`).

Use this to compute the **change plan**, structured as three lists:

- **C4 Context** — additions / removals: `+ Person(support, ...)`, `+ System_Ext(stripe, ...)`, `- System_Ext(mailgun, ...)`. Or `no changes`.
- **C4 Container** — additions / removals: `+ Container(worker, ...)`, `+ Rel(api, stripe, "Charges via", "REST API")`, `- ContainerDb(mongo, ...)`. Or `no changes`.
- **Flows** — for each affected flow: `create | update | delete | leave-untouched`, with one-line rationale per item. New flows have their proposed kebab-case name.

Show the change plan in the chat **before writing**. The user can object / adjust:

- "Don't add Stripe yet — the ADR is still pending."
- "The flow name should be `account-deletion`, not `user-account-removal`."
- "Skip the password-reset flow update — already done manually."

This is the only gating step in the skill. There is no per-diagram confirmation after this — once the change plan is accepted, all diagrams are written in one pass.

For **bootstrap mode** (§2.1), the change plan is the entire proposed initial state of `context.mmd` and `container.mmd`. The user reviews the inferred container list especially carefully; bootstrap is the only mode in which the skill is doing meaningful inference rather than reflecting concrete signals.

### Step 3 — Merge protocol (the heart of the skill)

For each diagram that needs to change, apply this protocol. **Surgical merge** is the default. **Full rewrite** happens only when the user explicitly asks ("redo the container diagram from scratch") or in bootstrap mode (where there is nothing to merge against).

#### 3.1. C4 Context / C4 Container merge

1. **Parse** the existing `.mmd` file into a set of nodes and a set of relationships. Preserve every comment (`%% ...`) and every layout hint (`UpdateLayoutConfig(...)`) verbatim.
2. **Apply additions.** New nodes go after the last node of the same kind (new `Person` after the existing persons; new `System_Ext` after the existing externals; new `Container` inside the boundary, after the last container). New relationships go in the corresponding section.
3. **Apply removals.** Only remove a node when **all** of these hold:
   - The current state of the codebase / ADRs no longer supports it (e.g. the dependency was removed from `package.json`, the integration was explicitly deprecated in an ADR).
   - No other relationship in the diagram currently points to it.
   - The user's change plan in Step 2 confirmed the removal.
   If any of these does not hold, the node stays and the skill notes the suspected staleness in the chat summary instead of removing.
4. **Preserve order.** Existing node order is preserved. Comments above existing nodes stay above them.
5. **Preserve labels.** If a node's existing label is more accurate than what the skill would write fresh ("Auth0 (legacy) — being migrated to Cognito" — keep the parenthetical), the existing label wins. The skill does not "clean up" labels without an explicit ask.
6. **Validate consistency.** After the merge:
   - Every relationship references a defined node.
   - Every container is inside a `System_Boundary`.
   - The `title` line stays (or is updated to reflect a renamed system, only with user confirmation).

#### 3.2. Per-flow sequence merge

1. **Parse** the existing `sequence.mmd` into participants and messages (in order). Preserve all `Note over` blocks and section comments.
2. **Apply additions.** New messages go in the spot in the sequence implied by the flow's actual order. The skill does not append blindly to the end; it reads the surrounding messages to find the right insertion point.
3. **Apply changes to existing messages.** A message label that is now stale (e.g. endpoint renamed `POST /auth/login` → `POST /auth/sign-in`) is updated in place.
4. **Apply removals.** A message is removed only when the corresponding code path is gone from the codebase / diff and the change plan in Step 2 confirmed the removal.
5. **Apply alternative-path additions.** New error paths (e.g. a new "rate limit" branch) go after the happy path, separated by a `Note over A,B: Alternative: <name>` header.
6. **Participant set.** Add new participants only when actually used by a new message; remove only when no message refers to them after the merge.
7. **Title line.** The `title` updates only if the flow was explicitly renamed (with user confirmation in Step 2).

#### 3.3. Surfacing conflicts

If the merge would silently overwrite an existing decision — e.g. an existing message says "API → Auth0" and the current ADR says Cognito — the skill **stops on this diagram only**, surfaces the conflict in the chat with:

- The existing message / node and where it lives (file + line).
- The new value implied by the anchor and where the anchor states it (ADR section, diff line).
- Two options: "keep existing" / "apply the change".

The skill does not auto-pick. Other diagrams continue if they have no conflicts.

### Step 4 — Write the diagrams

Write diagrams **one file at a time**, in this order:

1. **C4 Context** (`docs/c4/context.mmd`) — even though Container often has more changes, Context is shorter and easier to validate first.
2. **C4 Container** (`docs/c4/container.mmd`).
3. **Sequence diagrams**, one per affected flow, in alphabetical order of flow name.

For each file:

- In Claude Code: write directly to disk, creating the parent directory if needed (`mkdir -p docs/c4/`, `mkdir -p docs/flows/<flow-name>/`).
- In chat: emit one inline Mermaid artifact per file. Name the artifact with the path it would have on disk.
- After each file: capture the change summary line ("`docs/c4/container.mmd`: +1 container, +2 relationships, 0 removals") for the chat summary.

The skill does **not** write any other file. No README. No changelog. No sidecar notes file. Only `.mmd`.

### Step 5 — Deliver and stop

Deliver the chat summary (§7). The skill then **stops**. There is no interactive follow-up phase (unlike `sdlc-review` and `sdlc-security`, which stay in the conversation for discussion). Diagrams are deliverables, not discussion topics — if the user wants to revise, they invoke `sdlc-docs` again with a narrowed scope.

---

## 6. Mermaid syntax rules and stylistic constraints

These rules are encoded in the templates (`references/c4-context.mmd`, `references/c4-container.mmd`, `references/sequence.mmd` next to this `SKILL.md`) and enforced by inspection. They exist to keep diagrams consistent across features and to keep them renderable by Mermaid Live Editor / GitHub / VS Code Mermaid plugin — the three rendering surfaces the project supports.

### 6.1. C4 (context and container)

- The **first non-comment line** is `C4Context` or `C4Container`.
- The **second line** is the `title` line: `title System Context Diagram for <System Name>` or `title Container Diagram for <System Name>`. The system name is the canonical project name (read from the existing diagram's title, from `package.json` `name`, or from the user when ambiguous).
- **Section comments** (`%% External actors`, `%% External systems`, `%% System boundary`, `%% Relationships`) are preserved across merges and used as logical separators.
- Identifiers (`user`, `admin`, `api`, `db`, `cache`, `stripe`) are short, lowercase, and stable. Once an identifier exists in a diagram, the skill does **not** rename it across merges — code review tools, embedded references, and external links may depend on the identifier.
- `Person(...)` and `System_Ext(...)` are the only allowed actor / external shapes at Context level.
- `Container(...)` and `ContainerDb(...)` are the only allowed inside-boundary shapes at Container level.
- `Rel(source, target, "label", "transport")` — both label and transport are mandatory; transport is the protocol / SDK ("JSON/HTTPS", "SQL/TCP", "Redis protocol", "S3 SDK", "Kafka", "Stripe SDK").
- The optional `UpdateLayoutConfig($c4ShapeInRow="3", $c4BoundaryInRow="1")` line is preserved if it exists; not added if it doesn't.

### 6.2. Sequence

- The **first line** is `sequenceDiagram`.
- The **second line** is the `title` line, in the form `title <Flow Name> — <one-clause description>`. Example: `title User Login Flow — email + password`. The flow name in the title is the human-readable form of the kebab-case folder name.
- **Actor** for the human initiator: `actor User`. Use the same noun across all sequence diagrams for the same kind of actor (always `User`, `Admin`, `Support`, etc.).
- **Participants** are aliased to short uppercase / mixed-case identifiers, with a `<br/>` second line for the technology: `participant SPA as Web App<br/>(React)`. The identifier (`SPA`, `API`, `DB`) matches the C4 Container identifier whenever possible.
- **Messages** use `->>` (sync), `-->>` (response), `-)` (async / fire-and-forget). Self-messages use `->>` to self.
- **Labels** are short; long payloads break across lines with `<br/>`. Example: `API->>DB: SELECT user<br/>WHERE email = ?`.
- **Section breaks for alternative paths** use `Note over A,B: Alternative: <name>` (the project default per the template). Mermaid's `alt`/`else`/`end` blocks are also valid Mermaid but are not used by this skill — the `Note over` style is consistent and reads better in plain-text diffs.
- **No `loop`, `par`, `critical`, `break` blocks** unless the flow's value genuinely depends on them (a real retry loop, a real parallel call). Adding them speculatively is noise.

### 6.3. Common rules across all three diagram types

- **Encoding.** UTF-8, LF line endings, final newline.
- **No trailing whitespace.** No tab characters (use 4-space indent — matches the templates).
- **Identifier stability.** Once an identifier exists in any diagram, the skill keeps it across merges. Renaming an identifier requires user confirmation in Step 2 (it breaks references in tools that link to the diagram source).
- **Comment preservation.** Every `%% ...` comment in an existing diagram is preserved verbatim across merges, in the same position relative to its anchor node.
- **No styling directives** (`classDef`, `style`, theming) unless they already exist in the file. The diagrams render against the rendering surface's default theme; the skill does not introduce per-file theming.

### 6.4. Things the skill never writes

- **Component-level C4 (Level 3).** No `Component(...)` shapes. Out of scope.
- **Code-level C4 (Level 4).** Out of scope.
- **ER diagrams.** `erDiagram` is forbidden. Schema is documented in the codebase (Prisma schema, migrations) — not in `docs/`.
- **Deployment diagrams.** `Deployment_Node(...)` shapes are forbidden. Out of scope.
- **State, activity, mind-map, gantt, journey diagrams.** All out of scope.
- **Snapshots of the prototype.** No `flowchart` of UI screens — the prototype itself (`02-prototype.html`) is the artifact for that.

If the user explicitly asks for one of these, the skill says so directly: "That diagram type is out of scope for this skill. The three I produce are C4 Context, C4 Container, and per-flow sequence — would one of those cover what you need?"

---

## 7. Output

The skill's deliverable is **the `.mmd` files themselves** plus a **chat summary**. There is no separate report artifact (per design).

### 7.1. Files

In Claude Code:

- `docs/c4/context.mmd` — created or updated in place.
- `docs/c4/container.mmd` — created or updated in place.
- `docs/flows/<flow-name>/sequence.mmd` — one file per affected flow, created or updated in place. New flow folders are created as needed (`mkdir -p`).

In chat:

- Each diagram is returned as a separate inline Mermaid artifact, named with the colocated path it would have on disk (`docs/c4/context.mmd`, `docs/flows/user-login/sequence.mmd`).

The skill does not write any non-`.mmd` files.

### 7.2. Chat summary structure

The summary follows this structure, in this order:

```
## sdlc-docs — <anchor identity>

### Diagram baseline
<Step 1 output: anchor, affected flows detected, existing diagrams found, additions/removals implied, conflicts detected.>

### Change plan
<The §5.2 three-list plan: Context changes, Container changes, Flow changes — in the form the user reviewed.>

### Files written / updated / deleted
<Per file: path, action (created | updated | unchanged | deleted), short delta summary
 ("+1 container, +2 relationships, 0 removals" or "no changes — diagram already reflects the current state").>

### Conflicts surfaced
<Per conflict: existing value, anchor-implied value, decision the user made or is asked to make.
 If none, omit this section.>

### Possible next steps
<§8 list — informational, not a question.>
```

The summary is **terse**. It does not narrate process. It does not paste full diagram contents into the chat (the file is the artifact; the chat shows the delta). If a user wants to see the rendered result, they open the `.mmd` file in their renderer of choice.

---

## 8. Possible next steps

After producing the diagrams, the skill does **not** ask "ready to proceed?". It lists possible next steps as **information**, and the user picks the next move.

- `sdlc-docs` re-run — when more changes land later (a follow-up feature, a refactor across containers), invoking `sdlc-docs` again on the new anchor triggers another surgical merge. The diagrams stay current as the project evolves.
- `sdlc-ba` / `sdlc-adr` — if the diagram update surfaced a conflict that needs a product or architectural decision (e.g. the system now talks to two payment gateways for unclear reasons), the appropriate upstream skill resolves it; `sdlc-docs` reflects the result on the next pass.
- `sdlc-review` / `sdlc-security` — if the diagram review (visual inspection) surfaces a concerning shape (a worker that bypasses the API to write directly to the DB; an external system that has no authentication boundary in the diagram), the relevant audit skill formalizes the concern.
- **Visual rendering** — the user opens `docs/c4/context.mmd` / `docs/c4/container.mmd` / `docs/flows/<flow-name>/sequence.mmd` in Mermaid Live Editor (https://mermaid.live), GitHub's built-in `.mmd` rendering, or a VS Code Mermaid plugin. The skill does not render images itself.
- **Cycle close** — if this `sdlc-docs` pass was the last step of a feature cycle (`sdlc-ba` → `sdlc-design` → `sdlc-adr` → `sdlc-pm` → `sdlc-impl` → `sdlc-review` → `sdlc-security` → `sdlc-tests` → `sdlc-docs`), the feature is complete from the workflow's perspective. The artifacts under `docs/features/FEAT-NNN-<slug>/` plus the updated project-level diagrams are the durable record.

Possible next steps are listed; they are not proposed in order or prioritized. The user decides.
