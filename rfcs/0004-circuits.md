# RFC-0004: Circuits

> Status: **Draft** · Date: 2026-08-31 (revised after adversarial review)
> Depends on: RFC-0001 (teams), RFC-0002 (events/finalize)
> Decisions: ADR-0001 (co-location), ADR-0003 (fan-out at finalize)
> Open product choices: `OPEN-DECISIONS.md`

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
    shortfall: RUN_SHORT          # RUN_SHORT | NEXT_ELIGIBLE | CANCEL — REQUIRES DECISION
    declines: NEXT_ELIGIBLE       # how to fill declined slots — REQUIRES DECISION
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
  `minRosterSize`. Retro/prospective semantics are a decision (`OPEN-DECISIONS.md`).
- When the roster-evaluation rule is violated, teams get the `eligibilityMode` treatment **at read
  time** — the standings endpoint computes eligibility flags and hides/excludes rows accordingly.

## 7. Qualification

- Sort circuit standings by `totalPoints` (default tiebreak: points → best single placement → event
  count → name — configurable).
- Take top `numQualifiers` teams meeting `minEvents` + eligible; resolve **shortfall/declines** per
  `settings.qualification` (decisions in `OPEN-DECISIONS.md`). Record `qualifiedTeamIds` on the
  CIRCUIT item.
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

All consolidated in `OPEN-DECISIONS.md`; the ones blocking correct circuit implementation:
`eligibilityMode` default + whose snapshots count toward the "stay stable" bar (retro vs prospective);
`fieldSizeEffect` (small-event farming); tie splits for shared placements; `dropLowest` timing;
points for playoff depth (now well-defined via `placement`); qualification shortfall/declines;
main-event-as-member-event; mid-season joins (points from join date, default).

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