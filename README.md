# SDLC Workflow

A set of nine Claude skills (plus three subagents) that walk a feature through the full software lifecycle — from a raw idea to documented, tested, audited code.

Each phase is its own skill. You run them one at a time and review the output before moving on. Skills don't auto-chain.

## How it works

```
idea → sdlc-ba → sdlc-design → sdlc-adr → sdlc-pm → sdlc-impl → sdlc-review → sdlc-security → sdlc-tests → sdlc-docs
```

Every skill produces a concrete artifact stored under your project's `docs/features/FEAT-NNN-<slug>/` folder. The presence of that artifact is the only "state" — there is no separate tracker file.

You can skip phases or run a single skill on its own (for example, `sdlc-review` on legacy code without the upstream artifacts). Skills check what's already there and fall back gracefully when prerequisites are missing.

Project-level diagrams (C4 context, C4 container) live in `docs/c4/`. Per-flow sequence diagrams live in `docs/flows/<flow-name>/`. Code goes wherever your project keeps it.

## Skills

| # | Skill | What it does | Output |
|---|---|---|---|
| 1 | `sdlc-ba` | Turns a raw idea into a structured Feature Spec — problem, users, FRs, NFRs, acceptance criteria. | `01-feature-spec.md` |
| 2 | `sdlc-design` | Builds a self-contained HTML prototype that opens in a browser. Detects your UI framework and matches its style. Skips for backend-only features. | `02-prototype.html` |
| 3 | `sdlc-adr` | Researches at least three options per architectural decision, picks one, and records tradeoffs, limitations, and future optimizations. | `03-adr.md` |
| 4 | `sdlc-pm` | Decomposes the feature into independently implementable tasks — sized for one impl run each, with priorities and dependencies. | `04-tasks/T-NNN-<slug>.md` |
| 5 | `sdlc-impl` | Implements one task at a time. Splits multi-layer work into UI / Frontend / Backend sub-stages with a soft gate between them. Scaffolds empty test files (does not fill them). | code + test scaffolds |
| 6 | `sdlc-review` | Reviews a diff against its task, ADR, spec, and codebase conventions. Findings grouped Blocking / Significant / Nit with file:line citations. Stays in chat for follow-up discussion. | `05-review-T-NNN.md` |
| 7 | `sdlc-security` | OWASP-style audit of a feature diff. Findings classified Critical / High / Medium / Low / Info, mapped to OWASP/CWE, with mitigation directions. Stays in chat for discussion. | `06-security-audit.md` |
| 8 | `sdlc-tests` | Writes Vitest unit, RTL component, and Playwright e2e tests anchored to acceptance criteria. Runs the suite. Never edits production code or weakens assertions to make red tests green. | colocated `*.test.ts(x)` + `e2e/*.spec.ts` |
| 9 | `sdlc-docs` | Updates Mermaid diagrams (C4 context, C4 container, per-flow sequences) to reflect what was actually shipped. Surgical merge — adds new nodes, leaves the rest alone. | `.mmd` files under `docs/c4/` and `docs/flows/` |

Owner skills (`sdlc-ba`, `sdlc-adr`, `sdlc-pm`, `sdlc-docs`) ship their templates inside `references/` — the skill is fully self-contained.

## Subagents

Three skills delegate to a Claude Code subagent for a fresh-context, tool-restricted pass. If a subagent isn't available the skill works inline.

| Subagent | Used by | Tools | Why |
|---|---|---|---|
| `sdlc-code-reviewer` | `sdlc-review` | Read / Glob / Grep | Read-only enforcement — the reviewer cannot edit the diff it's reviewing. |
| `sdlc-security-auditor` | `sdlc-security` | Read / Glob / Grep | Read-only enforcement, plus a specialized OWASP-tuned prompt. |
| `sdlc-test-writer` | `sdlc-tests` | Read / Glob / Grep / Write (test files only) / Bash (test commands only) | Structurally cannot edit production code, install packages, or run ad-hoc bash. |

## Installation

Skills and subagents live in your global Claude Code home so every project can use them:

```
~/.claude/skills/sdlc-*
~/.claude/agents/sdlc-*
```

To install (or re-sync) from this repo:

```bash
mkdir -p ~/.claude/skills ~/.claude/agents
cp -R .claude/skills/sdlc-* ~/.claude/skills/
cp -R .claude/agents/sdlc-* ~/.claude/agents/
```

## Project conventions

- **Feature folder** — `docs/features/FEAT-NNN-<slug>/` (zero-padded number, kebab-case slug).
- **Task file** — `T-NNN-<slug>.md` inside `04-tasks/`.
- **Flow folder** — `docs/flows/<kebab-case-flow-name>/sequence.mmd`.
- **Artifact prefixes** (`01-`, `02-`, …) are fixed per phase across all features.
- **Artifact language** — English. Conversation language — yours.

## Defaults

When no project conventions exist, skills assume:

- **Language:** TypeScript
- **Runtime:** Node.js
- **Frontend:** React
- **Tests:** Vitest, React Testing Library, Playwright
- **Diagrams:** Mermaid (C4 + sequence)

Override these in the conversation or in your project's `package.json` / config files — the skills will detect and follow what you actually use.
