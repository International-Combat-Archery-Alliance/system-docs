# ICAA RFCs

RFCs capture project-level design for the ICAA platform **before** they are built, and live on
through implementation so we always know what has been decided, what is done, and what remains.

They complement `../docs/` (which describes how the platform *is*) by describing how the platform
*will be* — per feature area. Cross-cutting decisions are recorded separately as
[Architecture Decision Records](../adr/README.md).

> **Product context:** the *what and why* for these features — personas, scope, roadmap, and the
> product-level decisions — lives in **PRD-0001 ([../prd/0001-competition-platform.md](../prd/0001-competition-platform.md))**.
> If an RFC drifts from what we're building, reconcile against the PRD.
> See [../prd/README.md](../prd/README.md) for how PRDs are written.

## Status lifecycle

Each RFC has a `Status` field that moves as work progresses:

1. **Draft** — idea captured, decisions recorded, not yet accepted/built.
2. **In Progress** — accepted and actively being implemented; checkboxes get ticked as pieces land.
3. **Done** — fully implemented and deployed; the RFC stays as the record of what was built.
4. **Superseded** — a later RFC replaced this design (linked in the header).

Keep the **Implementation progress** checklist at the bottom of each RFC updated as code lands —
that is how these docs stay truthful.

## Open team decisions

League-rule and product choices the ICAA team must confirm before implementation (defaults are
recorded so work isn't blocked). **Each decision lives in its owning RFC's `## Open decisions`
section**; **[OPEN-DECISIONS.md](OPEN-DECISIONS.md)** is the index, plus a couple of cross-cutting
notes (D27/D28).

## RFC index

| # | Title | Status | Depends on |
|---|---|---|---|
| 0001 | [Teams & Rosters](0001-teams-and-rosters.md) | Draft | 0005 (player UUIDs) |
| 0002 | [Games, Schedules & Standings](0002-games-schedules-standings.md) | Draft | 0001; 0004 contribution rows in the finalize stamping transaction |
| 0003 | [Playoffs](0003-playoffs.md) | Draft | 0001, 0002 |
| 0004 | [Circuits](0004-circuits.md) | Draft | 0001, 0002 (contribution rows in the finalize stamping transaction) |
| 0005 | [Player Profiles](0005-player-profiles.md) | Draft | 0002 (history tab, after RFC-0002 ships) |

## Recommended build order

Staged, from adversarial review:

1. **0005 bio + directory first** — net-new service, zero interaction with the payment flow. Events
   tab is explicit "pending until 0002". Includes the **email-backfill pass** (hard prerequisite for
   0001's migration).
2. **0001 Stage A** — additive teams API + admin UI + `TEAM_NAME` GSI. No registration-flow changes.
3. **0001 Stage B — backfill** legacy registrations → teams (after 0005 email pass).
4. **0002** — Event.status, games CRUD, generators, standings, finalize (**lock + one bounded
   stamping transaction**; 0004's contribution rows live there — additive, no stub needed).
   Boston backfill workstream.
5. **0003 single-elimination** — bracket + placement authority.
6. **0004 circuits** — contribution rows in the stamping transaction, standings, qualification.
7. **0003 double-elimination**.
8. **0001 Stage C — the registration form swap** last, once teams data is trustworthy.

## Decisions

Design decisions that shape multiple RFCs are recorded as ADRs in
[`../adr/`](../adr/README.md):

| ADR | Decision |
|---|---|
| 0001 | Competition domain (teams, rosters, games, standings, circuits) stays co-located in `event-registration` |
| 0002 | Player references validated server-side at roster-write; name/email captured as snapshot |
| 0003 | Player results derived via per-player projection — **superseded by [ADR-0009](../adr/0009-player-history-participation-index.md)** |
| 0004 | Roster authority is admin-only for v1 |
| 0005 | Schedules & results use generate-then-edit |
| 0006 | Service-to-service HTTP with JWT machine auth (RS256 via login, JWKS-verified, spec-declared routes) |
| 0007 | User JWTs migrate from HS256/shared-secret to RS256 (login-owned key, JWKS verification) |
| 0008 | Session & admin hygiene fixes in login (logout revokes refresh; atomic rotation + reuse detection; de-admin purge) |
| 0009 | [Player history via participation index + team-history stamping](../adr/0009-player-history-participation-index.md) — supersedes 0003 |

## Platform note: service-to-service calls

The platform has no service-to-service communication today. The **first** need arrives with
[ADR-0002](../adr/0002-player-reference-validation.md) — `event-registration` verifying player UUIDs
and capturing authoritative snapshots from `player-profiles-api` at roster-write time (admin-path
only, never on the public payment flow). The approach, including **machine JWTs** (client-credentials
issued by `login`, spec-declared `icaaBearerAuth` + `m2m` scope on internal routes) and
**code-generated clients** from the callee's OpenAPI spec, is designed in
[ADR-0006](../adr/0006-internal-http-machine-auth.md). Nothing in RFCs 0001–0005 requires any other
inter-service call.