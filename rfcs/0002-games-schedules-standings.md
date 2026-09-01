# RFC-0002: Games, Schedules & Standings

> Status: **Draft** · Date: 2026-08-31 (revised after adversarial review)
> Depends on: RFC-0001 (teams & event participation); RFC-0004 points hook (interface stubbed into
> the finalize fan-out, §11)
> Decisions: ADR-0003 (projection/race fixes), ADR-0005 (generate-then-edit)
> Open rule choices: `OPEN-DECISIONS.md`

## 1. Background

Today an event's games, schedule, and standings exist only as **hardcoded JSX** for two Boston events
in `icaa.world/src/components/EventPageTemplate.tsx`; every other event shows "coming soon". The
`Event` aggregate has **no status field**. This RFC makes games, schedules, and standings
first-class, API-backed data.

## 2. Goals / Non-goals (v1)

### Goals
- `Game` entity per event: phase (qualifying/playoff), round, sides, scores, status.
- **Generate-then-edit** — round robin and swiss generators (pure, deterministic, unit-tested).
- Standings **derived deterministically from games** at read time; no stored standings row.
- Event lifecycle: `OPENED → IN_PROGRESS → FINALIZED`; finalize locks the event, then fans out
  player projections + circuit points through a single idempotent recompute path (ADR-0003).
- Replace the hardcoded event-page data with API-backed cards.

### Non-goals (defer)
- Live score-entry UIs with timers; per-game player lineups; rescheduling tools beyond score edits.

## 3. Data model (additions to the `event-registration` table)

```
Game:                PK = "GAME#<eventId>"          SK = "GAME#<phase>#<round%02d>#<seq%02d>"
                     id, eventId, phase (QUALIFYING|PLAYOFF), round, seq (display #),
                     status (SCHEDULED|COMPLETED|FORFEIT|DOUBLE_FORFEIT|CANCELLED|BYE),
                     sideA {teamId|null, isBye: bool}, sideB {teamId|null, isBye: bool},
                     scoreA?, scoreB?, forfeitSide? ("A"|"B"|null),
                     nextGameId?, nextGameSlot? ("A"|"B"), prevAId?, prevBId? (playoffs),
                     notes?, startTime?, Version
```

- **SK ordering:** round/seq are **zero-padded** (`%02d`) so lexical order == schedule order within a
  phase. `PLAYOFF` sorts before `QUALIFYING` lexically ('P' < 'Q'); the handler must order QUALIFYING
  before PLAYOFF explicitly (generator tests assert SK ordering).
- **Rule:** only EVENT masters and TEAM masters carry `GSI1PK`/`GSI1SK`; **GAME, participation,
  membership, history, RESULT#, and CIRCUIT items carry NO GSI attributes** (prevents poisoning the
  existing GSI1 events-list query that `begins_with(GSI1SK, "EVENT#")`).

### Event changes
- `status: OPENED | IN_PROGRESS | FINALIZED`, **optional in the API spec** (defaults server-side to
  `OPENED`; the live SPA admin form won't break when the schema first ships).
- `championTeamId?` (set when a playoff bracket resolves).
- **Registration close stays time-based:** the existing `registrationCloseTime` field (already
  `required` in the spec and enforced in the registration handlers) remains the **sole** trigger —
  moving an event to `IN_PROGRESS` does **not** auto-close it (decision D8, `OPEN-DECISIONS.md`).
  Late teams enter via the admin `late-team` flow (§7). There is exactly one close concept, not a
  second flag to drift.
- **No forfeit/withdrawal rule machinery** in v1 (decisions D2/D3/D4, `OPEN-DECISIONS.md`): when a
  team forfeits or withdraws, the admin records the game status/score directly (§5); no
  `eligibilityRule` is designed or stored.

## 4. Schedule generation (generate-then-edit)

`POST /events/v1/{eventId}/games/generate` — body `{format, opts}`. Admin only. Registration must be
closed (past `registrationCloseTime`) and the event moved to `IN_PROGRESS` to generate. **Input is
the event's `CONFIRMED` (paid) participation roster only** — ever-`REGISTERED`/unpaid rows are
excluded, so schedules, standings, finalize, and circuit points can never include a team that didn't
pay (RFC-0001 §6 lifecycle). Refuses if games exist unless `replace: true` and no game is
`COMPLETED` **or `FORFEIT`/`DOUBLE_FORFEIT`** (a forfeit is a result — never destroyed by replace).

**Round robin** (`format: ROUND_ROBIN`):
- Circle method. Games = **N(N−1)/2** real games; **rounds = N−1 (even N) or N (odd N)**; games per
  round = `floor(N/2)`; odd fields leave one team **resting** per round.
- **Byes are virtual:** the generator emits only real games; a resting team simply has no game that
  round and records `GP = N−1`. No `SCHEDULED`-with-one-side games exist, so the finalize
  no-pending-games precondition can never be blocked by a bye.
- `opts.mirror: true` → double round robin (rounds double).
- Side A/B alternates per round (green/yellow balance). Generator unit tests assert every pair meets
  exactly once, the SK order, and side balance.

**Swiss** (`format: SWISS`):
- Generated **one round at a time** because pairings depend on prior results. Pairing = pure
  function: sort by score group → net → PF → name, pair top-down within groups, **never re-match**
  already-played pairs; when within-group pairing is impossible, **down-pair** to the next score
  group (explicit relaxation rule). Odd field → the lowest-score group's team with the fewest byes
  gets a **virtual bye** (recorded on the game as `status: BYE`, no pf/pa, treated as resolved —
  see §5). Guard: tiny fields (advisable: use ROUND_ROBIN for ≤ 4 teams).
- BYE allocation: lowest score group, never given twice if avoidable; a bye does **not** count as a
  win in v1 (no free points) — an open rule, default recorded in `OPEN-DECISIONS.md`.

Both generators are pure + deterministic (fixtures for N=3…16, odd/even, mirror, and swiss edge
cases) and live in a `schedules/` (or `games/`) domain package.

## 5. Games, standings & forfeit/withdrawal effects

`PUT /events/v1/{eventId}/games/{gameId}` (admin) sets `status` + scores. The transaction carries a
**`ConditionCheck` on the EVENT item (`status != FINALIZED`)** so post-finalize edits are structurally
impossible (not just handler-policy). Validation:

- `COMPLETED` requires two real sides and scores.
- `FORFEIT` requires `forfeitSide`; `DOUBLE_FORFEIT` = neither side shows (both record a loss, pf/pa
  0) — representable.
- `BYE` needs no scores, counts as resolved, and touches nothing in standings.
- After finalize, edits require **unfinalize → edit → finalize** (§6); `recompute` only re-runs the
fan-out of an event that is still `FINALIZED`.

**Standings** (`GET /events/v1/{eventId}/standings`, public, derived on read): per team `gp, wins,
losses, pf, pa, net`. Sort: **wins ↓ → net ↓ → pf ↓ → name**. Only `COMPLETED`/`FORFEIT` QUALIFYING
games count. `BYE` and `CANCELLED` never count.

A team must have ≥ 1 completed/forfeit game **and `CONFIRMED` participation** to be **placed** at
finalize; teams with none are `DNS`/unranked (excludes no-shows and unpaid entries from placements
and from circuit points — RFC-0004).

### Effects of forfeits & withdrawals (admin-recorded, not rule-driven)

- **Forfeits/withdrawals are recorded per game by the admin** (`status: FORFEIT`/`DOUBLE_FORFEIT` +
  scores) — there is **no fixed default-win rule** designed for v1 (decisions D2/D3/D4,
  `OPEN-DECISIONS.md`). A mid-event withdrawal is handled as it happens; `DOUBLE_FORFEIT` (both teams
  lose, pf/pa 0) remains first-class.
- **Draws:** v1 **disallows draws** — tied regulation games are resolved (sudden-death / tiebreak) so
  every completed game has a winner. The `draws` schema field is reserved for a future rule; the
  sort key needs no draw arithmetic. (Recorded in `OPEN-DECISIONS.md` in case the org wants tied
  games.)

## 6. Finalize (lock + fan-out), recompute, and unfinalize

`POST /events/v1/{eventId}/finalize` (admin), idempotent.

**Preconditions:** all QUALIFYING games resolved (`COMPLETED`/`FORFEIT`/`DOUBLE_FORFEIT`/`CANCELLED`/
`BYE`) **and no unresolved PLAYOFF games** — a bracket dispute is resolved by recording a result or
forfeit (there is **no `qualifyingOnly` mode**, decision D11; a genuinely unresolved game blocks
finalize — accepted risk, RFC-0003). Warn + require explicit confirm when a participant has zero
games but is not marked `WITHDRAWN`/DNS.

**Flow (per ADR-0003):**
1. **Atomic transaction:** EVENT → `status: FINALIZED`, `projectionsStatus: PENDING` (+Version lock).
   That is the whole lock; no team-result rows are snapshot-stored from a pre-lock read.
2. **Post-transaction idempotent fan-out:** player `RESULT#<eventId>#<teamId>` projections (from the
   standings/snapshot **frozen at this instant**) and circuit contribution rows (RFC-0004) written in
   chunked batches (≤ 25/request, `UnprocessedItems` retry loop).
3. On completion: EVENT `projectionsCompleteAt = now`. **Delivered alarm (same PR as finalize, not a
   someday):** CloudWatch custom metric `icaaFinalizePending` emitted when PENDING set; alarm
   `FINALIZED_PENDING > 0 for 15 min` → SNS; runbook: "verify games committed → `POST recompute` →
   confirm `projectionsCompleteAt` → clear alarm." The 10s Lambda timeout makes interruptions real.

**State-machine invariants (the whole pipeline):**
- Artifacts exist ⇔ `status=FINALIZED ∧ projectionsStatus=COMPLETE`. Any other status/state includes
  no `RESULT#`/contribution rows for the event.
- **`recompute` refuses unless `status=FINALIZED`** — its legit uses are repairing an interrupted
  fan-out and re-deriving artifacts after circuit/settings changes (RFC-0004 §5). After unfinalize,
  the only path back is `finalize`.
- **`finalize` always re-fans out the whole event** (never short-circuits on a marker).
- **`unfinalize`** (admin, audited; records `{who, when, why}`) flips `FINALIZED → IN_PROGRESS` and
  **clears `projectionsStatus` + `projectionsCompleteAt` in the same transaction**, then deletes
  fan-out artifacts (idempotent keyed deletes, best-effort with retry). Score corrections then go
  through edit → `finalize` again.

**Placement authority (one answer):** the official `placement` is the **bracket-adjusted final
order** (RFC-0003: champion 1st, runner-up 2nd, etc.). The qualifying sort is a separate, displayed
**"Qualifying Rank"** and never the stored placement. Circuit points and player projections consume
`placement` only.

## 7. API surface

| Method | Path | Auth | Description |
|---|---|---|---|
| GET | `/events/v1/{eventId}/games` | public | Schedule (QUALIFYING before PLAYOFF; joined with team names) |
| POST | `/events/v1/{eventId}/games/generate` | admin | Generate round robin / swiss (§4) |
| PUT | `/events/v1/{eventId}/games/{gameId}` | admin | Enter score/status (§5); blocked when event FINALIZED |
| GET | `/events/v1/{eventId}/standings` | public | Derived standings (§5) |
| POST | `/events/v1/{eventId}/late-team` | admin | Add a team after generation; regenerates unplayed games preserving COMPLETED/FORFEIT (RFC-0001) |
| POST | `/events/v1/{eventId}/finalize` | admin | Lock + fan-out (§6) |
| POST | `/events/v1/{eventId}/recompute` | admin | Re-run fan-out (ADR-0003) |
| POST | `/events/v1/{eventId}/unfinalize` | admin | Auditor-gated reopen (§6) |
| GET | `/events/v1/players/{playerId}/history` | public | Player participation history from `RESULT#<eventId>#<teamId>` projections (RFC-0005 §6) |
| PATCH | `/events/v1/{eventId}` | admin | Status transitions (`OPENED→IN_PROGRESS→FINALIZED`; registration close stays time-based) |

`Event` schema gains `status` (optional) and `championTeamId?`.

## 8. UI (icaa.world)

- Event page **Standings** + **Schedule** cards render from the API; hardcoded arrays removed once
  Boston data is backfilled (§9).
- Admin: generator form (format/options), score-entry rows, finalize/recompute/
  unfinalize with confirmations.

## 9. Migration

Ordering matters:

1. **Set `status` server-side as optional.** Existing events: upcoming → `OPENED`; past (`endTime <
   now`) → `FINALIZED` **only after** their games are entered (below), else left `IN_PROGRESS`-able —
   do not mark with zero games or standings render empty for events that have results.
2. **Boston backfill is its own reviewable workstream (HIGH effort):** reconstruct full results
   **including the missing playoff games** — the hardcoded 15 Championship games cannot produce the
   canonical 7-0/38-8 player record (a 6-team single round robin is 15 games; 7 games means the
   Championships had playoffs). Map JSX names ("Renegades" vs "Boston Renegades") to the team
   identities minted from registration data. **Verify derived standings == the existing PNG/table
   before cutover.** Then run the finalize fan-out so team pages and player profiles have real
   history (RFC-0005's whole value prop is derived history).
3. Add item types + the `TEAM_NAME` GSI to `infra/main.tf`, `docker-compose.yml` dynamo-setup, and
   `dynamo/db_test.go` together; poll `describe-table` for `ACTIVE` before deploying the search
   endpoint; adapter fails fast at startup if the index is missing.

## 10. Edge cases

- Regenerate scheduling mid-event (blocked once a result exists, per §4); odd fields (virtual byes);
- Team drops out / no-shows (DNS exclusion from placement + points); forfeits incl. double;
- Finalize with an unresolved bracket (must be resolved via results/forfeits — no `qualifyingOnly` mode); interrupted fan-out
  (completion marker + recompute); score correction post-finalize (unfinalize → edit → finalize);
- Late team add preserving results (RFC-0001 `late-team` flow); cross-phase ordering on the schedule.

## 11. Checklist

- [ ] `Event.status` (optional in spec, preserved in `UpdateEvent`) + transitions
- [ ] `games/Game` aggregate + dynamo adapter + handlers (CRUD, generate, standings)
- [ ] Round-robin generator (N/odd-N/20-round arithmetic) + SK-order/balance fixtures
- [ ] Swiss generator (score-group pairing, down-pair, no-rematch, byes) + fixture tests
- [ ] Standings derivation + DNS exclusion + **CONFIRMED-participation-only** placement fixtures
- [ ] Finalize lock/fan-out/completion marker; recompute (refuses unless FINALIZED); unfinalize (audited, clears markers)
- [ ] **`icaaFinalizePending` CloudWatch metric + alarm + SNS + recompute runbook (same PR as finalize)**
- [ ] Circuit points hook (RFC-0004) stubbed into fan-out from day one
- [ ] `TEAM_NAME` GSI in terraform + docker-compose + db_test; GSI-attribute rule on new items
- [ ] Boston backfill workstream (incl. playoffs) + verification + finalize fan-out
- [ ] Event page cards wired to API; admin generator/score/finalize UI
- [ ] Update `system-docs/docs/data.md` + `services.md`