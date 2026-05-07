# ADR: <Decision Title>

**Feature:** FEAT-NNN-<slug>
**ADR ID:** ADR-FEAT-NNN-01 (use -02, -03 if multiple decisions for one feature)
**Status:** Proposed | Accepted | Deprecated | Superseded by ADR-XXX
**Date:** <YYYY-MM-DD>
**Decision makers:** <names or sdlc-adr>

---

## 1. Context

<Describe the problem and the forces at play. What is the situation that requires a decision? What constraints exist? What is the architectural question being answered?>

<Reference the Feature Spec where relevant: "Per FR-3 in the Feature Spec, the system must ..., which raises the question of ...".>

## 2. Decision Drivers

The factors that influence this decision, in order of importance:

- <Driver 1, e.g. "Must scale to 10k req/s per NFR-4">
- <Driver 2, e.g. "Team has strong TypeScript expertise, limited Rust experience">
- <Driver 3, e.g. "Must integrate with existing Postgres infrastructure">
- <Driver 4, e.g. "Time to market — 6 weeks">

## 3. Options Considered

At least three options, evaluated against the decision drivers.

### Option A: <Name>

**Description:** <How this option works in 2-4 sentences.>

**Pros:**
- <Pro 1>
- <Pro 2>

**Cons:**
- <Con 1>
- <Con 2>

**Estimated effort:** <e.g. "2 weeks for MVP">

### Option B: <Name>

**Description:** <...>

**Pros:**
- <...>

**Cons:**
- <...>

**Estimated effort:** <...>

### Option C: <Name>

**Description:** <...>

**Pros:**
- <...>

**Cons:**
- <...>

**Estimated effort:** <...>

## 4. Comparison

| Criterion | Option A | Option B | Option C |
|-----------|----------|----------|----------|
| Complexity | <Low/Med/High> | ... | ... |
| Performance | ... | ... | ... |
| Maintainability | ... | ... | ... |
| Cost | ... | ... | ... |
| Team familiarity | ... | ... | ... |
| TS/React ecosystem fit | ... | ... | ... |
| Time to implement | ... | ... | ... |

## 5. Decision

**Chosen option:** <A | B | C>

**Rationale:** <Explain why this option wins given the decision drivers. Be specific about which drivers tipped the balance and which were deprioritized.>

## 6. Tradeoffs

What we **gain** by this choice and what we **sacrifice**. Be honest — every choice has costs.

**Gained:**
- <e.g. "Simpler operational model, fewer moving parts">
- <e.g. "Reuses existing Postgres skillset">

**Sacrificed:**
- <e.g. "Lower theoretical throughput than dedicated time-series DB">
- <e.g. "More complex query patterns for time-window aggregations">

## 7. Known Limitations

What this decision deliberately does **not** solve. The boundaries of the chosen approach.

- <Limitation 1 — e.g. "This approach does not support multi-region active-active. If we need that later, see Future Optimization #2.">
- <Limitation 2 — e.g. "Read-after-write consistency is not guaranteed across replicas; eventual consistency only.">
- <Limitation 3>

## 8. Future Optimization Opportunities

Places where the implementation can be improved later, when the constraints that drove the original decision change. Not a TODO list — a roadmap of known improvement vectors.

- **<Opportunity 1>** — <e.g. "Introduce read replicas when read load exceeds 5k QPS sustained. Estimated lift: 3-5x read throughput.">
- **<Opportunity 2>** — <e.g. "Migrate hot rows to in-memory cache (Redis) when p99 latency degrades past 100ms.">
- **<Opportunity 3>** — <e.g. "Consider switching to time-series DB if event volume grows past 10M/day.">

## 9. Consequences

Implications of this decision for the codebase, team, and operations.

**For the codebase:**
- <e.g. "All persistence layer code goes through a single Repository interface — easier to swap later.">
- <e.g. "We add a new dependency on `pg` and `kysely`.">

**For the team:**
- <e.g. "No new technology to learn — existing Postgres expertise applies.">

**For operations:**
- <e.g. "One additional Postgres instance to monitor and back up.">

**For testing:**
- <e.g. "Need a Postgres test container in CI pipeline (`testcontainers-node`).">

## 10. References

- Feature Spec: `docs/features/FEAT-NNN-<slug>/01-feature-spec.md`
- Related ADRs: <ADR-XXX, ADR-YYY>
- External: <links to articles, benchmarks, RFCs>
