# Feature Spec: <Feature Name>

**Feature ID:** FEAT-NNN-<slug>
**Status:** Draft | In Review | Approved | Implemented
**Author:** <name or sdlc-ba>
**Created:** <YYYY-MM-DD>
**Last updated:** <YYYY-MM-DD>

---

## 1. Problem Statement

<Describe the problem this feature solves. What pain does the user/business experience today? Why does this matter now? Keep it 2-4 sentences. Focus on the problem, not the solution.>

## 2. Users & Stakeholders

**Primary users:**
- <User type 1> — <what they need from this feature>
- <User type 2> — <what they need from this feature>

**Secondary users / stakeholders:**
- <Stakeholder> — <their interest>

## 3. Goals

What success looks like for this feature:

- <Goal 1 — concrete and measurable where possible>
- <Goal 2>
- <Goal 3>

## 4. Non-Goals

What this feature explicitly does **not** do. Crucial for scope control:

- <Non-goal 1 — and why it's out of scope>
- <Non-goal 2>

## 5. User Stories

Format: As a `<role>`, I want to `<action>` so that `<outcome>`.

- **US-1:** As a `<role>`, I want to `<action>` so that `<outcome>`.
- **US-2:** ...
- **US-3:** ...

## 6. Functional Requirements (FR)

What the system must **do**. Each requirement is testable and traceable.

| ID | Requirement | Related US |
|----|-------------|------------|
| FR-1 | The system shall ... | US-1 |
| FR-2 | The system shall ... | US-1, US-2 |
| FR-3 | The system shall ... | US-3 |

## 7. Non-Functional Requirements (NFR)

How the system must **behave**. Quality attributes.

| ID | Category | Requirement |
|----|----------|-------------|
| NFR-1 | Performance | <e.g. p95 response time under 200ms for endpoint X> |
| NFR-2 | Security | <e.g. all PII encrypted at rest> |
| NFR-3 | Accessibility | <e.g. WCAG 2.1 AA compliance for all new UI> |
| NFR-4 | Scalability | <e.g. handle 1000 concurrent users without degradation> |
| NFR-5 | Observability | <e.g. all errors logged with correlation IDs> |
| NFR-6 | Availability | <e.g. 99.9% uptime for the new endpoints> |
| NFR-7 | Compatibility | <e.g. support last 2 major versions of Chrome/Safari/Firefox> |

*Remove rows that don't apply. Add rows for any other quality attribute relevant to the feature.*

## 8. Product Tradeoffs

Product-level decisions about scope, time, and value. **Technical tradeoffs (e.g. database choice, framework choice) belong in the ADR, not here.**

- **<Tradeoff 1>:** <e.g. "Ship MVP without admin panel — admins will use SQL directly until v2"> → rationale: <why>
- **<Tradeoff 2>:** <e.g. "No mobile-optimized layout in v1"> → rationale: <why>
- **<Tradeoff 3>:** <e.g. "English-only at launch, i18n in v2"> → rationale: <why>

## 9. Acceptance Criteria

The feature is considered done when:

- [ ] AC-1: <criterion>
- [ ] AC-2: <criterion>
- [ ] AC-3: <criterion>
- [ ] All FRs above are satisfied and verifiable.
- [ ] All NFRs above are met and measured.

## 10. Out of Scope (deferred to future versions)

Things considered but explicitly pushed to later releases:

- <Item 1> — target: v2
- <Item 2> — target: backlog

## 11. Open Questions

Questions that need answers before or during implementation:

- [ ] <Q1>
- [ ] <Q2>

## 12. References

- Related features: <FEAT-XXX>
- Related ADRs: <to be created>
- External docs: <links>
