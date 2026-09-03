# ADR-0009: Player history via participation index + team-history stamping

> Status: **Accepted** · Date: 2026-09-02
> Supersedes: [ADR-0003](0003-player-results-projection.md)
> Applies to: RFC-0001, RFC-0002, RFC-0005

## Context

Player profile pages need each player's tournament history (events played, team each event, record,
placement). [ADR-0003](0003-player-results-projection.md) chose a per-player `RESULT#` projection
written at finalize: for every rostered player, an accumulated stats row
(`PLAYER#<playerId> / RESULT#<eventId>#<teamId>` carrying record/pf/pa/placement).

Design review (2026-09-02) found the per-player projection to be the *only* part of finalize that
does not fit DynamoDB's 100-action transaction cap (≈89 actions at 8×8, 133+ at 12 teams, and the
original math omitted history mirrors + circuit rows). That forced the whole burden that followed:
chunked post-transaction fan-out (≤25/write, `UnprocessedItems` retry), a completion marker
(`projectionsStatus`/`projectionsCompleteAt`), and a visibility mechanism (CloudWatch alarm + SNS)
for interrupted publishes. The per-player rows also duplicate team-owned data: any score correction
would re-fan-out or drift.

Review conclusion: player pages don't need stored stats — they need a **cheap index from player to
(event, team)** and a read-time derivation from per-team data the events service already owns. Read
cost is then bounded by the player's event count, not the event's size — the exact premise
("expensive derivation") that motivated the projection in the first place.

## Decision

No per-player stat rows exist. Two coupled pieces:

1. **Player event index** — one lightweight item per (player, event, team), written transactionally
   at **participation creation** (RFC-0001 §6), never at finalize:

   ```
   PK = "PLAYER#<playerId>"   SK = "HISTORY#<eventId>#<teamId>"
   attributes: eventId, teamId, teamName, eventName, eventDate
   ```

   - Bounded: ≤ roster size items, created in the registration intent transaction, rolled back by
     the expiry transaction, extended by the snapshot-adjust and late-team admin paths.
   - Invariant: index rows mirror `participation.rosterSnapshot`.
   - Key includes the team, so a player on two teams in one event keeps both entries.

2. **Team-history result stamping at finalize** — RFC-0001's `TEAM#<teamId> / EVENT#<eventId>`
   items already exist (with a `result` field, today empty). Finalize computes the final standings
   and bracket-adjusted `placement` (single derivation rule, RFC-0003 §6), then **stamps
   `result {record, pf, pa, placement}`** on each participant's team-history item — RFC-0004's
   contribution rows in the same transaction. At 16 teams this is ≤ ~48 actions: **one bounded
   `TransactWriteItems`**, idempotent keyed puts.

3. **Player history reads** — `GET /events/v1/players/{playerId}/history`:
   `Query(PLAYER#<playerId>)` over the index (N = events played) + one `BatchGetItem` over the
   team-history rows → assemble the response (record/pf/pa/placement). Response shape is identical
   to RFC-0005 §6; a veteran player with 20 events is 1 Query + 1 BatchGetItem.

## Consequences

- **Finalize is synchronous and bounded**: a lock transaction (`EVENT → FINALIZED`) + one stamping
  transaction (`finalizeCompleteAt = now` included for admin-UI visibility). No chunked fan-out, no
  completion-marker machinery, no outbox/worker, no alarms. An interrupted finalize is a *visible
  request error*; `finalize`/`recompute` re-run and re-stamp idempotently.
- **No replication**: player pages always reflect committed games. A score correction years later
  (unfinalize → edit → finalize) propagates to every page automatically — nothing to reconcile.
- **Mid-event**: index rows exist from participation; stats appear once finalize stamps them
  (history tab shows an explicit pending state until then — display decision, RFC-0005 §6).
- **Placement authority unchanged** (RFC-0003 §6): placement is derived once at finalize and stamped
  on team-history rows; circuits and player history both consume it from there.
- **GSI rule unaffected**: index and team-history items carry no GSI attributes (RFC-0001/0002).
- **The `icaaFinalizePending` CloudWatch alarm requirement is dropped** (decision D30,
  OPEN-DECISIONS.md): with nothing running in the background there is nothing to monitor.

## Revisit trigger

If the platform reaches hundreds of events / thousands of players with heavy profile traffic,
per-read derivation may warrant reintroducing a cached projection (the stored-projection tradeoff
returning). Not a v1 concern; recorded here so the choice is deliberate.