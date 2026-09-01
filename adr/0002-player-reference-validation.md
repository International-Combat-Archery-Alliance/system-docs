# ADR-0002: Player references validated server-side (reference + snapshot)

> Status: **Accepted** · Date: 2026-08-31
> Applies to: RFC-0001, RFC-0005
> Transport detail: [ADR-0006](0006-internal-http-machine-auth.md)

## Context

Rosters (events service) must reference players whose master data lives in `player-profiles-api`.
A UUID alone doesn't prove a player exists — a hand-crafted API call could submit a bogus UUID — so
the guarantee cannot rely on the SPA having performed a search first. The weakness would grow once
captains get self-service roster access.

## Decision

Players are identified by **UUID** everywhere, sourced/minted only by `player-profiles-api`
([RFC-0005](../rfcs/0005-player-profiles.md)). At **roster-write time** the events service makes a
**read-only call to the profiles service** for the submitted `playerId`, which:

1. **verifies the player exists**, and
2. **captures the authoritative current name/email** from the response as the rosters **snapshot**.

Writes **fail closed** if the profiles service is unreachable — the client-supplied name is never
trusted. Player master data is **soft-deleted** (`status: ARCHIVED`), never hard-deleted, so old
UUIDs and snapshots keep rendering.

## Rationale

Server-side verification (not "by construction") is warranted because self-service is planned and
verification is cheap today — roster writes are rare admin-path operations. Verification and
authoritative snapshot capture share one round-trip. Reads stay snapshot-only, so no service call
ever sits on a hot path.

## Consequences

- Snapshot staleness is intended: competition records freeze names as of write time, even if the
  player's master record changes later (historical fidelity).
- The events service has a **hard dependency on profiles + the machine-token path for roster
  writes** — acceptable; admin ops can fail loudly and be retried.
- Local dev: the events service uses a `LOCAL` mode (dev machine token / local login) so the auth
  header path is exercised without prod secrets ([ADR-0006](0006-internal-http-machine-auth.md)).

## Supersedes

Nothing; this replaces the earlier (unwritten) notion of relying on SPA-orchestrated UUIDs alone.