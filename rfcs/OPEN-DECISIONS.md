# Open team decisions

League-rule and product choices raised during design review that the ICAA team must confirm. A
**recommended default** is recorded for each so implementation isn't blocked — the schema is built to
accommodate the alternatives where it matters. Decisions here are called out from
[RFC-0001](0001-teams-and-rosters.md), [RFC-0002](0002-games-schedules-standings.md),
[RFC-0003](0003-playoffs.md), [RFC-0004](0004-circuits.md), [RFC-0005](0005-player-profiles.md),
and the RFC index.

**Resolved at design review (2026-09-01):** the items previously marked "decide before code" are now
decided — D8 (keep the existing time-based registration close), D2/D3/D4 (forfeits/withdrawals are
recorded by admins per game; no automatic rule machinery), D11 (no `qualifyingOnly` mode; every
playoff bracket completes via recorded results/forfeits), and D19/D20 (qualification is an
admin-managed field — the computed list is a suggestion). The remaining items below are safe to ship
with their recorded defaults.

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
| D2 | **Forfeit/withdrawal score** — how is a forfeited or unplayed game scored? **Resolved:** no fixed rule is designed for — forfeits are niche, so when one happens the admin records the game status and score directly (forfeit remains a first-class game status). | Admin records per game (status + score) | [x] (2026-09-01) |
| D3 | **Mid-event withdrawal** — what happens to a withdrawn team's remaining games? **Resolved:** no automatic default-win rule; the admin handles the withdrawal as it happens (record forfeits / adjust). | Admin handles as it happens | [x] (2026-09-01) |
| D4 | **Double forfeit** — both teams absent. **Resolved:** both record a loss, pf/pa 0 — admin sets the game status/scores. | Both lose, 0–0 | [x] (2026-09-01) |
| D5 | **DNS / never-played teams** — excluded from placement and from circuit points. Confirmed? (DNS = `status: WITHDRAWN|DNS` at finalize-check; finalize warns if a participant has zero games.) | Exclude DNS from placement + points | [ ] |
| D6 | **Bye representation** — round robin: **virtual** (no game row; team rests; GP = N−1); swiss: a **`BYE`-status game row** with no pf/pa and no win credited — never an unresolved `SCHEDULED` game. Confirm, or should a bye credit a win? | Split as stated | [ ] |
| D7 | **Swiss tiny fields** — ≤ 4 teams → force ROUND_ROBIN. Confirm N and the down-pair relaxation. | Force round robin for N ≤ 4 | [ ] |
| D8 | **Late registration** — when does event sign-up close? **Resolved:** keep the existing time-based `registrationCloseTime` — no change to the live paid flow (moving an event to IN_PROGRESS does not auto-close). Late teams still enter via the admin late-team flow. | Keep time-based close (no change) | [x] (2026-09-01) |

## Playoffs / placement

| # | Decision | Recommended default | Status |
|---|---|---|---|
| D9 | **Placement authority** — official `placement` = bracket-adjusted final order (champ 1st, runner-up 2nd, losing semifinalists 3rd/4th by qualifying rank, etc.), with qualifying order shown as "Qualifying Rank". Confirm. | Bracket-adjusted placement | [ ] |
| D10 | **Points for playoff depth** — the circuit points table keys on `placement`, so a semifinalist beats a round-1 loser. Confirm the default table (1:100, 2:80, 3:65, 4:55, 5:45, 6:35, 7:25, 8:15) or provide a league table. | Default table ($5) | [ ] |
| D11 | **Stuck brackets** — what if a playoff bracket can't be resolved? **Resolved:** assume every bracket finishes — playoffs are just recorded games. Admins resolve disputes via game status/score entry; there is **no `qualifyingOnly` mode** in v1. Accepted risk: a genuinely unresolved game blocks finalize. | No qualifyingOnly; resolve via results/forfeits | [x] (2026-09-01) |

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
| D19 | **Qualification shortfall** — fewer eligible teams than slots. **Resolved:** the championship field is admin-managed — the computed list is a suggestion (seeded by defaults) and admins review/set the final field. | Admin-managed (computed list is a suggestion) | [x] (2026-09-01) |
| D20 | **Qualified-team declines** — a qualified team declines. **Resolved:** admin-managed — the suggestion rolls to the next eligible team; admins confirm the replacement. | Admin-managed (next-eligible suggestion) | [x] (2026-09-01) |
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