# ICAA PRDs

Product Requirements Documents (PRDs) capture the **what and why** — the product spec — for a
coherent piece of work, at a level any stakeholder (league board, organizers, engineers) can read
and sign off on. They are the **north star**: the reference we check RFCs against when we revise or
implement them.

RFCs describe *how* a feature will be built; ADRs record *why* behind cross-cutting technical
decisions. PRDs are the layer above: *what we are building, for whom, in what order, and how we'll
know it worked.* This README is the convention guide for writing one.

## Where a PRD sits in the docs

| Doc | Owns | Audience |
|---|---|---|
| **PRD** (`prd/NNNN-*.md`) | *What & why* — outcomes, personas, scope, priorities/sequencing, product decisions, success criteria | Everyone |
| RFC (`rfcs/NNNN-*.md`) | *How* — design, data model, APIs, migration | Engineers |
| ADR (`adr/NNNN-*.md`) | *Why* behind a technical approach | Engineers |
| [OPEN-DECISIONS](../rfcs/OPEN-DECISIONS.md) | Live league-rule/product decision log | Team |
| [docs/](../docs) | How the system *is* today (current state) | Engineers |

The rule of thumb: **if the PRD and an RFC disagree on *what* we're building, the PRD wins** — fix
the RFC.

## House rules (how every PRD should look)

1. **Product-level, in user terms.** Write for people, not systems. No data models, interfaces,
   migration steps, service/repo names, implementation phases, or engineering acceptance tests — those
   belong in the RFC. Describe outcomes ("a team's history spans events") not mechanisms.
2. **Reference, don't restate.** Link to the RFC/ADR for design detail. A PRD that re-specifies the
   design becomes a third source of truth that drifts.
3. **One PRD per coherent product program** (e.g. PRD-0001 = Competition Platform, spanning several
   RFCs). Add a new PRD for a new program; number globally.
4. **Be opinionated on what ships and what waits.** Include an explicit non-goals / out-of-scope
   list and a product-framed roadmap, not just a list of features.
5. **Own the decisions, not the decision log.** Summarize the decisions that block or shape the
   work in user terms and link to `OPEN-DECISIONS.md` for the full log.
6. **Define product-level "done."** Success criteria are measurable outcomes (a person can do X,
   Y is no longer manual), not "the endpoint exists."
7. **Keep it tight.** The PRD is the entry point; a reader should finish it before deciding to dive
   into RFCs. If a section is growing into a design doc, move it to an RFC and link.

## Required structure (template)

Every PRD should contain these sections, in order:

```
# PRD-NNNN: <Title>

> Status: Draft | In Review | Approved | Implementing | Superseded
> Date: YYYY-MM-DD · Owner: <name>
> Scope: <product areas affected>

## 1. What we're building
## 2. Who it's for              (personas + today's pain → what they get)
## 3. Goals & non-goals         (non-goals = deferred *within* this program)
## 4. Feature scope             (per feature: product summary + link to RFC/ADR)
## 5. How we ship it            (product roadmap / sequencing, incl. hard rules)
## 6. Product decisions to confirm   (user-terms summary + link to OPEN-DECISIONS)
## 7. How we'll know it's working    (success criteria)
## 8. Future program candidates (revisit triggers; non-goals from §3 stay in §3)
## 9. References
```

Optional sections as the program needs them: metrics/KPIs, dependencies on other programs.

**Required if the program touches auth or payments:** a **"Risks & rollout notes"** section (after
References) calling out anything that could disrupt the live system — cutover/maintenance windows,
data-completeness risk in backfills, and any place the money path could break. These are exactly the
things a non-engineer approving the spec must hear.

## Lifecycle

- **Draft** — being written / not yet agreed.
- **In Review** — out for stakeholder sign-off.
- **Approved** — signed off; RFCs may be implemented against it.
- **Implementing** — work is in flight against this spec.
- **Superseded** — a later PRD replaced this one (linked in the header).

Keep the PRD updated as the product changes — it is the living reference, not a point-in-time
artifact. When scope shifts materially, record it here (and reconcile the RFCs).

## Index

| # | Title | Status | Date |
|---|---|---|---|
| 0001 | [Competition Platform](0001-competition-platform.md) | Draft | 2026-09-01 |
