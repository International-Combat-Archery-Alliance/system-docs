# RFC-0003: Playoffs

> Status: **Draft** · Date: 2026-08-31 (revised after adversarial review)
> Depends on: RFC-0001 (teams), RFC-0002 (games/standings/status)
> Decisions: ADR-0005 (generate-then-edit), RFC-0002 §6 (placement authority)
> Open rule choices: §Open decisions (below)

## 1. Background

With qualifying standings available per event (RFC-0002), events can have **playoffs** that decide a
champion. The playoff bracket is its own feature/generator: it runs off the results of the
tournament's **qualifying round** (round robin or swiss). This RFC covers single- and
double-elimination bracket generation, progression, and champion determination, plus the **placement
authority** the rest of the system consumes.

## 2. Goals / Non-goals (v1)

### Goals
- Generate **single-elimination** and **double-elimination** brackets from final qualifying standings
  (seeds), materialized as `GAME` items in `phase: PLAYOFF`.
- Bracket structure with links (`prevAId`/`prevBId`/`nextGameId`/`nextGameSlot`) so winners advance
  and the bracket renders as a tree.
- First-round **byes** for top seeds (as `BYE`-resolved games — they never block finalize).
- Champion determined by finishing the bracket; **`placement` = bracket-adjusted final order** (the
  single placement authority defined in RFC-0002 §6), with qualifying order shown separately as
  "Qualifying Rank".
- Admin can edit scores / override participants in bracket games.

### Non-goals (defer)
- Best-of-N series, consolation/third-place games (points for non-1st/2nd come from the default
  placement table — §6), multi-day scheduling optimization.

## 3. Data model

Playoff games reuse the `Game` item from RFC-0002 with `phase: PLAYOFF`:

```
PK = "GAME#<eventId>"   SK = "GAME#PLAYOFF#<round%02d>#<seq%02d>"
sideA/sideB {teamId | EMPTY until filled | BYE}, scores, status (…| BYE),
prevAId/prevBId (feeders), nextGameId/nextGameSlot ("A"|"B"),
status: SCHEDULED|COMPLETED|FORFEIT|DOUBLE_FORFEIT|CANCELLED|BYE, Version
```

Bracket **byes** are `BYE`-status games with one side `BYE`; they resolve automatically (the seeded
side advances and is written into the next game's slot). Like RFC-0002 byes, they never appear as
unresolved `SCHEDULED` games.

## 4. Generators

`POST /events/v1/{eventId}/playoffs/generate` (admin) — body `{type: SINGLE_ELIMINATION |
DOUBLE_ELIMINATION, numTeams?}`. Input: the event's qualifying standings. Refuses if any PLAYOFF
game already has a result (`COMPLETED`/`FORFEIT`/…). Both generators are **pure functions from
(seeds, count) → game graph**, pinned to a **canonical bracket layout** (rank order → slot fold) and
covered by fixture tests for N = 3…16 (including non-power-of-2 fields).

### Single elimination
- Field `N` (≤ qualified teams). Rounds = `ceil(log2 N)`; games = N−1 (real games; byes are
  `BYE`-resolved, not played).
- **Canonical seeding** (pinned, not "1vN, 2vN−1" loosely): the standard rank-order→slot fold so
  that e.g. N=6 gives round-1 games 3v6 and 4v5 (top seeds 1,2 get byes and meet the correct
  winners), matching the conventional single-elim bracket. Fixture tests assert the exact tree.
- Links: round `r` feeds `nextGameId` round `r+1` slot A/B by bracket position.

### Double elimination
- Winners + Losers tracks; losing a W game drops into L; losing an L game eliminates; L-bracket
  pairing follows the standard round-counts pattern; **reset final** (`resetFinal: true` default —
  the W-winner losing the first final forces a second final). Graph fully link-encoded — the
  renderer walks `prev/next/slot` rather than assuming static layout. **Higher complexity than
  single-elimination** — phased after single-elim lands.

## 5. Progression

- Result entry reuses `PUT /events/v1/{eventId}/games/{gameId}` with the same EVENT `ConditionCheck
  (status != FINALIZED)`. When a playoff game completes, the **next game's slot is filled from the
  winner** in the same transaction. Admins can override a filled slot (forfeit ruling).
- **Qualifying games become READ-ONLY once any PLAYOFF game exists** — enforced in RFC-0002's
  `PUT /games` validation (a bracket's seeds come from qualifying; editing them mid-bracket silently
  corrupts seeds). Score corrections after brackets exist go through **unfinalize → edit → finalize**
  (RFC-0002 §6) — recompute only re-runs the fan-out of an already-`FINALIZED` event.
- `GET /events/v1/{eventId}/bracket` (public) returns the playoff game graph for tree rendering;
  `GET .../games` returns the flat schedule (QUALIFYING before PLAYOFF).
- A slot that cannot be filled (team dropped out) → admin resolves as `FORFEIT` (no auto-advance
  beyond `BYE`).

## 6. Champion, placement, and finalize

- When the final resolves, the winner is the event **champion** (`championTeamId`).
- **`placement` (the one authority):** 1st = champion, 2nd = runner-up, 3rd/4th = losing
  semifinalists, 5th–8th = losing quarterfinalists, etc., with ties within a round **resolved by
  qualifying rank** and teams that never won a playoff game placed after all playoff winners,
  ordered by qualifying rank. Qualifying order remains a separate, displayed "Qualifying Rank" and is
  never the stored placement.
- **Points for playoff depth** come from the circuit's placement table (`points.placement[]`,
  RFC-0004) keyed on this `placement` — a semifinalist is rewarded above a round-1 loser by default
  (table config; see RFC-0004 §Open decisions).
- **Finalize precondition (RFC-0002 §6):** no unresolved PLAYOFF games — a bracket stuck on a
  disputed/no-show game is resolved by recording a result or `FORFEIT`. There is **no
  `qualifyingOnly` mode** (decision D11): every bracket is expected to finish — playoff games are
  just games we record. A genuinely unresolved game blocks finalize (accepted risk).

## 7. UI (icaa.world)

- Event page **Playoffs** card (bracket tree) rendered from `GET .../bracket`.
- Admin: generator form (type + field size), bracket score entry, and slot-override controls.

## 8. Edge cases

- Regenerating after a result exists (blocked); byes interacting with layout (fixture-tested);
- double-elim reset-final rule; forfeits advancing teams; team dropout mid-bracket (FORFEIT slot,
  no auto-advance); bracket generated from a qualifying ranking that then changes (blocked by the
  read-only rule in §5); finalize with an unresolved bracket (must be resolved — no qualifyingOnly mode).

## 9. Checklist

- [ ] Bracket graph model + links in `Game`
- [ ] Single-elimination generator w/ canonical layout + byes + fixtures N=3..16
- [ ] Winner-advance transaction (fill next slot)
- [ ] Qualifying-games-read-only enforcement (RFC-0002 validation)
- [ ] `GET .../bracket` + UI bracket card
- [ ] Double-elimination generator (W/L tracks + reset final) + tests
- [ ] Champion + bracket-adjusted placement in finalize; circuit points keyed on placement
- [ ] Finalize refuses while any playoff game is unresolved (no qualifyingOnly mode)
- [ ] Admin override controls

## Open decisions

League-rule/placement choices specific to playoffs. Keep global `D#`s; the
[index](../OPEN-DECISIONS.md) maps every decision to its owning RFC.

| # | Decision | Recommended default | Status |
|---|---|---|---|
| D9 | **Placement authority** — official `placement` = bracket-adjusted final order (champ 1st, runner-up 2nd, losing semifinalists 3rd/4th by qualifying rank, etc.), with qualifying order shown as "Qualifying Rank". Confirm. | Bracket-adjusted placement | [ ] |
| D11 | **Stuck brackets** — what if a playoff bracket can't be resolved? **Resolved:** assume every bracket finishes — playoffs are just recorded games. Admins resolve disputes via game status/score entry; there is **no `qualifyingOnly` mode** in v1. Accepted risk: a genuinely unresolved game blocks finalize. | No qualifyingOnly; resolve via results/forfeits | [x] (2026-09-01) |