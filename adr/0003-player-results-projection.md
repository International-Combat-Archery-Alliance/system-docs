# ADR-0003: Player results derived via per-player projection

> Status: **Accepted** · Date: 2026-08-31 (revised 2026-08-31)
> Applies to: RFC-0002, RFC-0005

## Context

Player profile pages need each player's tournament history (events played, team each event, record,
placement). That data is a reverse-lookup join over competition data naturally indexed by event, not
by player — deriving it from raw games/participations on every page view is an expensive derivation.
The existing `player-profiles-api` DESIGN.md proposal stored `tournamentResults` admin-curated
instead; that avoids derivation but re-enters data the events service already owns.

## Decision (Option A, revised after adversarial review)

When an event is **finalized** ([RFC-0002](../rfcs/0002-games-schedules-standings.md) §Finalize),
the events service writes, for every rostered player, a projection item in the `event-registration`
table:

```
PK = "PLAYER#<playerId>"    SK = "RESULT#<eventId>#<teamId>"
attributes: eventId, eventName, eventDate, teamId, teamName,
            placement (int, bracket-adjusted final order), record (wins/losses), pf, pa
```

- **Key includes the team** (`RESULT#<eventId>#<teamId>`) so a player on two teams in one event
  cannot overwrite one result — idempotency is per (player, event, team). History reads are still a
  single partition query (`Query(PK=PLAYER#<playerId>)`).
- Player history is exposed by `GET /events/v1/players/{playerId}/history`. `player-profiles-api`
  does **not** store tournament results.

## Writes are always post-transaction (revised)

Finalize consistency was reviewed: storing per-team results computed from a pre-lock read of GAME
items races concurrent score edits, and the "fits in one 100-action transaction" claim was knife-edge
(≈89 actions at 8×8, 133+ at 12 teams, and the math omitted history mirrors + circuit rows).

New shape:

1. **Atomic part (one transaction):** EVENT → `status: FINALIZED` (+Version lock). That is the entire
   lock. Team-result rows are **not** snapshot-stored from a pre-lock read; standings derived on read
   (ADR-0005) are the single source of truth. (Optional: write each `EVENT_TEAM` result row with a
   `ConditionCheck` per QUALIFYING game version if a cheap result cache is ever wanted — not in v1.)
2. **Post-transaction idempotent fan-out:** player `RESULT#` projections and, when applicable,
   circuit contribution rows ([RFC-0004](../rfcs/0004-circuits.md)) are written in **chunked batches
   (≤ 25 items)** with a `UnprocessedItems` retry loop. All writes are keyed-by-(player,event,team)
   puts — re-running is always safe.
3. **Completion marker:** the EVENT first stores `projectionsStatus: PENDING` and, once the fan-out
   finishes, `projectionsCompleteAt`. A monitor (or a scheduled alarm) flags `FINALIZED` events
   whose `projectionsCompleteAt` never arrived so an interrupted finalize is visible, not silent.
4. **Single recompute path:** `POST /events/v1/{eventId}/recompute` (admin) re-runs the fan-out
   deterministically from the committed games/standings and the snapshots **frozen at finalize** (the
   editable participation snapshot is re-read only when unfinalizing). This is the *only* repair
   path, and it is what RFC-0002/0004 point all refresh operations at.

## Why not event-driven CDC

A DynamoDB Streams + consumer Lambda adds infrastructure and eventual consistency for a write that
can happen immediately after finalize. Rejected for this platform's scale; the recompute path heals
any interruption.

## Consequences

- Profiles reads are cheap and consistent with the source of truth (written at finalize, not on read).
- The events table carries a denormalized projection by design (standard single-table pattern).
- Placement ties: teams sharing a placement get the same integer; points mapping for tied/playoff
  depth is a circuit setting ([RFC-0004](../rfcs/0004-circuits.md) and `OPEN-DECISIONS.md`).