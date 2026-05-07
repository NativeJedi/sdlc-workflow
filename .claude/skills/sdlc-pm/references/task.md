# Task T-NNN: <Task Title>

**Task ID:** T-NNN
**Feature:** FEAT-NNN-<slug>
**Status:** Todo | In Progress | Done | Blocked
**Priority:** Must-have | Should-have | Nice-to-have
**Depends on:** <T-XXX, T-YYY> (or "none")

---

## 1. Context

<What part of the feature does this task deliver? Why is it a separate task and not merged with another? Reference the relevant FR/NFR from the Feature Spec.>

Related requirements: FR-X, FR-Y, NFR-Z

## 2. Description

<Concise description of what needs to be built. 2-5 sentences. Focus on outcome, not implementation.>

## 3. Files to Touch

Files expected to be created or modified:

**Create:**
- `src/<path>/<file>.ts` — <purpose>
- `src/<path>/<file>.test.ts` — <purpose>

**Modify:**
- `src/<path>/<existing-file>.ts` — <what changes>

**Read for context (no changes expected):**
- `src/<path>/<reference-file>.ts`

## 4. Implementation Plan

Step-by-step approach. Each step should be a self-contained unit.

1. <Step 1 — e.g. "Define the data model in `src/models/foo.ts` with fields X, Y, Z.">
2. <Step 2 — e.g. "Add migration `migrations/NNN_create_foo.sql` matching the model.">
3. <Step 3 — e.g. "Implement the FooRepository class with `findById`, `create`, `update` methods.">
4. <Step 4 — e.g. "Wire up the repository to the DI container in `src/container.ts`.">
5. <Step 5 — e.g. "Add API endpoint `POST /foo` in `src/routes/foo.ts`.">

## 5. Definition of Done

The task is complete when:

- [ ] All code in "Files to Touch" is written and lints cleanly.
- [ ] Test files exist (scaffolded — actual tests filled in by `sdlc-tests`).
- [ ] No regressions in existing tests.
- [ ] Code follows existing project conventions and patterns.
- [ ] <Task-specific criterion 1, e.g. "Endpoint returns 201 with correct response shape on success">
- [ ] <Task-specific criterion 2, e.g. "Validation errors return 400 with field-level details">
- [ ] <Task-specific criterion 3>

## 6. Test Plan

Tests that `sdlc-tests` should write for this task:

**Unit tests (Vitest):**
- <What pure logic / utilities need unit tests>

**Component tests (RTL):**
- <What components need component tests, if any>

**E2E tests (Playwright):**
- <What user flows need e2e coverage, if any — usually only critical paths>

**Test data needs:**
- <e.g. "Need a fixture for valid Foo entity with all required fields">

## 7. Notes & Considerations

<Optional. Any gotchas, edge cases, performance notes, security notes, or links to relevant ADR sections.>

- <Note 1>
- <Note 2>

## 8. References

- Feature Spec: `01-feature-spec.md`
- ADR: `03-adr.md` (specifically section <X> on <topic>)
- Related tasks: <T-XXX>
