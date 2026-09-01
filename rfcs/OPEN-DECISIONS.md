# Open team decisions

League-rule and product choices raised during design review that the ICAA team must confirm. A
**recommended default** is recorded for each so implementation isn't blocked — the schema is built to
accommodate the alternatives where it matters. Decisions here are called out from
[RFC-0001](0001-teams-and-rosters.md), [RFC-0002](0002-games-schedules-standings.md),
[RFC-0003](0003-playoffs.md), [RFC-0004](0004-circuits.md), [RFC-0005](0005-player-profiles.md),
and the RFC index.

**Decide BEFORE code starts:** D8 (registration-close behavior — changes the live paid flow),
D11 (`qualifyingOnly` escape hatch availability), D2/D3/D4 (forfeit/withdrawal rule stored semantics),
and D19/D20 (qualification shortfall/declines — external people-and-money consequences). The rest are
safe to ship with their defaults.

## How to use this

- Each row is an open question. Mark **`[ ]`** → **`[x]`** and record the decision date once the
  team agrees.
- Where the default is already implemented as described, flipping a decision may require a small
  config/migration; where noted, it changes behavior only (no schema churn).

---

## Tournament-day rules (block RFC-0002 finalize ± generators)

| # | Decision | Recommended default | Status |
|---|---|---|---|
| D1 | **Draws in games** — may a regulation game end tied? Default: v1 disallows ties (resolved by a tiebreak, e.g. sudden-death) so every completed game has a winner; the `draws` field stays reserved. If ties are allowed, they count as 0.5W/0.5L and the sort key becomes wins → draws → net → pf. | No ties in v1; resolve | [ ] |
| D2 | **Forfeit default score** — the fixed score for a forfeited/unplayed game (used in the stored `eligibilityRule`). Must be a legal regulation score, and it must not be a league-arbitrary number that inflates net. | 5–0 (must equal a real attainable regulation score) | [ ] · decide before code |
| D3 | **Mid-event withdrawal** — fixed rule chosen per event at generation time (stored on EVENT): played games stand; every remaining game vs the withdrawn team = default win (D2) for the opponent. Rejects the "admin decides per round" alternative that makes seeds schedule-luck. | Fixed default-win rule, chosen per event | [ ] · decide before code |
| D4 | **Double forfeit** — both teams absent: both record a loss, pf/pa 0. | Both lose, 0–0 | [ ] · decide before code |
| D5 | **DNS / never-played teams** — excluded from placement and from circuit points. Confirmed? (DNS = `status: WITHDRAWN|DNS` at finalize-check; finalize warns if a participant has zero games.) | Exclude DNS from placement + points | [ ] |
| D6 | **Bye representation** — round robin: **virtual** (no game row; team rests; GP = N−1); swiss: a **`BYE`-status game row** with no pf/pa and no win credited — never an unresolved `SCHEDULED` game. Confirm, or should a bye credit a win? | Split as stated | [ ] |
| D7 | **Swiss tiny fields** — ≤ 4 teams → force ROUND_ROBIN. Confirm N and the down-pair relaxation. | Force round robin for N ≤ 4 | [ ] |
| D8 | **Late registration** — registration closes when the event is set `IN_PROGRESS`. Late teams only via admin `late-team` (regenerates unplayed games, preserves results). Confirm the operationally simple version — **this changes the live paid flow** (today time-based `registrationCloseTime`). | Close at IN_PROGRESS | [ ] · decide before code |

## Playoffs / placement

| # | Decision | Recommended default | Status |
|---|---|---|---|
| D9 | **Placement authority** — official `placement` = bracket-adjusted final order (champ 1st, runner-up 2nd, losing semifinalists 3rd/4th by qualifying rank, etc.), with qualifying order shown as "Qualifying Rank". Confirm. | Bracket-adjusted placement | [ ] |
| D10 | **Points for playoff depth** — the circuit points table keys on `placement`, so a semifinalist beats a round-1 loser. Confirm the default table (1:100, 2:80, 3:65, 4:55, 5:45, 6:35, 7:25, 8:15) or provide a league table. | Default table ($5) | [ ] |
| D11 | **`qualifyingOnly` finalize** — when a bracket can't resolve (disputed forfeit), finalize with no champion + PENDING bracket, or refuse. **The escape hatch must ship IN finalize's UI/API** — otherwise a stuck bracket makes the event unfinalizeable forever. | Refuse by default; qualifyingOnly = explicit opt-in (must be reachable) | [ ] · decide before code |

## Circuits

| # | Decision | Recommended default | Status |
|---|---|---|---|
| D12 | **`fieldSizeEffect`** — with the flat top-8 table, winning a 4-team scrimmage pays the same as winning a 12-team event (farming risk). Default: `SCALE` (points scaled by field size) or a min-field rule (e.g. < K teams award no points). | SCALE or min-field ≥ 5 | [ ] |
| D13 | **`eligibilityMode`** — what happens when a team's season snapshots fall below `minRosterSize`: HOLD (points kept, ineligible for qualification), FORFEIT (ineligible-event contributions excluded at read time), UNRANKED (hidden from standings at read time). | HOLD | [ ] |
| D14 | **Stability bar** — which snapshots count toward "roster stays stable": min snapshot size over member events? last N events? retro or prospective from the moment of the drop? | Min snapshot size across member events; prospective | [ ] |
| D15 | **Roster lock window** — freeze roster changes N hours before each event (`lockHoursBeforeEvent`). Enforce in v1 or document as future? | Future (v1 documents only) | [ ] |
| D16 | **Tied placements** — teams sharing a placement integer both get the higher placement's points? | Both get higher placement | [ ] |
| D17 | **Drop-lowest** — per-team across-events; applied at read time. Confirm on/off and `dropLowest` count. | 0 (off) in v1; read-time when on | [ ] |
| D18 | **Mid-season join** — team joining after some events: points from join date (default) vs retroactive. | From join date | [ ] |
| D19 | **Qualification shortfall** — eligible teams < `numQualifiers`: run short? fill with next eligible? cancel main event? | Run short | [ ] · decide before code |
| D20 | **Qualified-team declines** — alternates mechanism. | NEXT_ELIGIBLE | [ ] · decide before code |
| D21 | **Main event as a member event** — is the championship also scored for points? (If yes, its finalize must not re-rank the qualification list used to fill it.) | No (main event is the target, not a scorer) | [ ] |

## Teams / players / product

| # | Decision | Recommended default | Status |
|---|---|---|---|
| D22 | **Player on multiple teams** — allowed in v1 (projection key already supports it via `RESULT#<eventId>#<teamId>`). | Allowed; review when self-service lands | [ ] |
| D23 | **Team name uniqueness** — enforced via name reservation + 409. Confirm (vs best-effort + merge). | Enforce via reservation | [ ] |
| D24 | **Post-launch team merges** — two teams turn out to be the same org: ship a merge operation or explicitly defer (keep survivor, re-point future events)? | Defer mechanics; future | [ ] |
| D25 | **Public player pages** — public read immediately, or admin-only until data is trustworthy (per RFC-0005)? | Admin-only initially, public after backfill verified | [ ] |
| D26 | **Free agents** — individual (ByIndividual) registrations coexist but earn no circuit points in v1. Confirm. | Unscored in v1 | [ ] |

## Cross-cutting

| # | Decision | Recommended default | Status |
|---|---|---|---|
| D27 | **ADR-0006 status** — machine-token S2S design (RS256, login-issued, JWKS-verified, exact `m2m:` scope): accepted as written. | Accepted | [x] (2026-08-31) |
| D28 | **README/docs** — `system-docs/docs/architecture.md` update ("frontend is the only consumer" is amended once ADR-0006 lands). | Update on first S2S implementation | [ ] |