# RFC-0004: Circuits

> Status: **Draft** · Date: 2026-08-31 (revised after adversarial review)
> Depends on: RFC-0001 (teams), RFC-0002 (events/finalize)
> Decisions: ADR-0001 (co-location), ADR-0003 (fan-out at finalize)
> Open product choices: §Open decisions (below)

## 1. Background

A **circuit** is one-to-many events across which teams earn points, with a **main
event/championship** at the end that the top teams qualify for. Point systems and roster-stability
rules vary by circuit and are **configurable per circuit**. This RFC wires points into the event
finalize fan-out (RFC-0002 §6) using **idempotent per-event contribution items** (ADR-0003 review
fix — safer than read-modify-write standing rows), and models roster eligibility so it cannot be
gamed by late roster additions.

## 2. Goals / Non-goals (v1)

### Goals
- Circuit entity: details + member events + participating teams + configurable settings.
- **Per-circuit points table** (placement → points), applied from each member event's final
  placements when the event finalizes.
- Circuit **standings** (total points, eligibility, qualification flags) in API + UI.
- Qualification target: the circuit's main event, filled by top **eligible** teams.
- **Roster eligibility evaluated on frozen event snapshots**, not the live ACTIVE roster.

### Non-goals (defer)
- Player/individual circuits (v1 is team circuits; free-agent events coexist but are not scored).
- Transfer points, team splits/merges mid-circuit, wildcards, tiebreaker tournaments.
- Circuit entry fees or application workflow (admin assigns teams).

## 3. Data model (additions to the `event-registration` table)

```
Circuit master:        PK = "CIRCUIT#<circuitId>"      SK = "CIRCUIT#<circuitId>"
                       id, name, description, startDate, endDate,
                       status (PLANNED|ACTIVE|COMPLETED), mainEventId,
                       settings (§4), Version

Circuit membership:    PK = "CIRCUIT#<circuitId>"      SK = "EVENT#<eventId>"
                       circuitId, eventId, orderIndex

Circuit team entry:    PK = "CIRCUIT#<circuitId>"      SK = "TEAM#<teamId>"
                       circuitId, teamId, joinedAt, Version

Circuit contribution:  PK = "CIRCUIT_CONTRIBUTION#<circuitId>"   SK = "EVENT#<eventId>#TEAM#<teamId>"
                       circuitId, eventId, teamId, points, placement,
                       eligible (bool — snapshot met minRosterSize at this event)
                       (idempotent keyed put — NO read-modify-write; aggregation on read)
```

Circuit standings are **aggregated on read** from contribution items (tens of teams × few events —
trivial) and sorted in the handler; `dropLowest` and eligibility exclusions are **read-time
projections**, never stored mutations. The circuits list endpoint is documented as Scan/in-memory
sort at ICAA scale (acceptable; adding a `CIRCUIT` GSI later would require amending the pinned
only-EVENT/TEAM GSI rule + its adapter test — RFC-0001/0002).

## 4. Circuit settings (point system & roster rules)

```yaml
settings:
  points:
    placement: {1: 100, 2: 80, 3: 65, 4: 55, 5: 45, 6: 35, 7: 25, 8: 15}   # 1..N by placement
  scoring:
    minEvents: 2                 # min member events to qualify
    dropLowest: 0                # drop N lowest-scoring events (read-time)
    fieldSizeEffect: SCALE       # NONE | SCALE — REQUIRES DECISION (default recommendation: SCALE)
  roster:
    minRosterSize: 6             # snapshot size required to EARN points at an event
    maxRosterSize: 8
    eligibilityMode: HOLD         # HOLD | FORFEIT | UNRANKED — REQUIRES DECISION (default: HOLD)
    lockHoursBeforeEvent: 0       # future (roster lock window), not enforced v1
  qualification:
    numQualifiers: 8
    requireEligible: true         # must be ELIGIBLE at qualification time
    shortfall: RUN_SHORT          # resolved (D19): admin-managed — computed list is a suggestion
    declines: NEXT_ELIGIBLE       # resolved (D20): admin-managed — next-eligible suggestion, admin confirms
```

## 5. Points flow (the finalize fan-out)

When a member event finalizes (RFC-0002 §6, ADR-0003):

1. Compute the bracket-adjusted `placement` per team, plus which teams are **DNS / never placed**
   (zero completed/forfeit games → RFC-0002 §5). **DNS teams earn no points.**
2. For each placed circuit-entrant team: `points = settings.points.placement[p]`; `eligible = team's
   event snapshot size ≥ minRosterSize` (the `rosterSizeAtEvent` frozen at finalize).
3. **Write a `CIRCUIT_CONTRIBUTION#…` item per (event, team) — idempotent keyed put** (no
   accumulate-then-write, no races between concurrent finalizes/recomputes). Part of the
   post-transaction fan-out with `UnprocessedItems` retry, **tracked by the `projectionsStatus:
   PENDING → COMPLETE` / `projectionsCompleteAt` marker lifecycle** (RFC-0002 §6).
4. **Recompute = RFC-0002's `recompute`** (deterministic re-derivation from committed games + frozen
   snapshots). The circuit `recompute` endpoint (§8) is the alias. Point-table edits, backfills, and
   membership changes trigger recompute — which must **delete stale contribution items** for removed
   member events/teams (keyed puts only add/overwrite), never mutate a stored `totalPoints`.

## 6. Roster eligibility (the "keep your points" rule) — non-gameable

- **Earn points at event E:** team's **snapshot at E** meets `minRosterSize` (stored as
  `eligible` on the contribution item at finalize; the snapshot is frozen — post-finalize edits can't
  help).
- **Keep/rank points (circuit level):** `eligibilityMode`:
  - `HOLD` (recommended default): points retained; team marked `INELIGIBLE` (excluded from
    qualification) until their *current snapshot across the season* meets the bar.
  - `FORFEIT`: contributions from ineligible events are **excluded at read time** (never a stored
    mutation).
  - `UNRANKED`: teams below the bar are excluded from the standings list at read time.
- **The bar is evaluated on event snapshots, never the live ACTIVE roster** (the live roster can be
  padded the night before qualification; snapshots cannot). A team is considered stable if its
  season snapshots (e.g. min snapshot size over member events, or the last N — product choice) meet
  `minRosterSize`. Retro/prospective semantics are a decision (§Open decisions).
- When the roster-evaluation rule is violated, teams get the `eligibilityMode` treatment **at read
  time** — the standings endpoint computes eligibility flags and hides/excludes rows accordingly.

## 7. Qualification

- Sort circuit standings by `totalPoints` (default tiebreak: points → best single placement → event
  count → name — configurable).
- Take top `numQualifiers` teams meeting `minEvents` + eligible; resolve **shortfall/declines** per
  `settings.qualification` (decisions D19/D20 in §Open decisions). **The computed list is a
  suggestion**: the championship field is **admin-managed** — admins review and set the final
  `qualifiedTeamIds` on the CIRCUIT item.
- The **main event as a member event is a decision** — if allowed, its own finalize happens after
  qualification and its results must NOT re-rank the already-computed qualification list (guard).
- Qualification is an admin action (`POST .../qualify`); the qualified list populates the
  championship participation via RFC-0001 (admin-confirmed).

## 8. API surface

| Method | Path | Auth | Description |
|---|---|---|---|
| GET | `/events/v1/circuits` | public | List (Scan/in-memory at v1 scale) |
| POST | `/events/v1/circuits` | admin | Create circuit |
| GET | `/events/v1/circuits/{circuitId}` | public | Detail + settings + events + teams |
| PATCH | `/events/v1/circuits/{circuitId}` | admin | Update settings / status |
| POST | `/events/v1/circuits/{circuitId}/events` | admin | Add member event(s) |
| DELETE | `/events/v1/circuits/{circuitId}/events/{eventId}` | admin | Remove member event (recompute) |
| POST | `/events/v1/circuits/{circuitId}/teams` | admin | Add team |
| GET | `/events/v1/circuits/{circuitId}/standings` | public | Aggregated points + eligibility flags |
| POST | `/events/v1/circuits/{circuitId}/recompute` | admin | Re-derive standings (alias of RFC-0002 recompute) |
| POST | `/events/v1/circuits/{circuitId}/qualify` | admin | Compute qualified teams |

## 9. Open product questions (schema is ready; org must decide)

Consolidated in §Open decisions below; the ones blocking correct circuit implementation:
`eligibilityMode` default + whose snapshots count toward the "stay stable" bar (retro vs prospective);
`fieldSizeEffect` (small-event farming); tie splits for shared placements; `dropLowest` timing;
points for playoff depth (now well-defined via `placement`);
main-event-as-member-event; mid-season joins (points from join date, default).
(Qualification shortfall/declines — D19/D20 — were **resolved** at design review: admin-managed field.)

## 10. UI (icaa.world)

- `/circuits` list; `/circuits/:circuitId` detail + **standings table** (points, eligible/qualified
  badges); event pages link to their circuit.
- Admin console: circuit builder (settings incl. points table), membership management, recompute +
  qualify actions (scoped like all admin UI — net-new work).

## 11. Checklist

- [ ] `circuits/` domain: Circuit, settings, membership, contribution aggregates + ports
- [ ] Circuit CRUD + membership + team entry endpoints + tests
- [ ] Contribution writes in finalize fan-out (idempotent, completion-marked) + recompute
- [ ] Eligibility from frozen snapshots (`rosterSizeAtEvent`), read-time modes
- [ ] Qualification + shortfall/declines handling; main-event guard
- [ ] Frontend circuit pages + admin builder
- [ ] Update `system-docs/docs/data.md` + `services.md`

## Open decisions

League-rule/product choices specific to circuits. Keep global `D#`s; the
[index](../OPEN-DECISIONS.md) maps every decision to its owning RFC.

| # | Decision | Recommended default | Status |
|---|---|---|---|
| D10 | **Points for playoff depth** — the circuit points table keys on `placement`, so a semifinalist beats a round-1 loser. Confirm the default table (1:100, 2:80, 3:65, 4:55, 5:45, 6:35, 7:25, 8:15) or provide a league table. | Default table ($5) | [ ] |
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