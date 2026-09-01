# ADR-0001: Competition domain co-located in `event-registration`

> Status: **Accepted** · Date: 2026-08-31
> Applies to: RFC-0001, RFC-0002, RFC-0003, RFC-0004

## Context

The platform has no service-to-service messaging, one DynamoDB table per service, and atomic
multi-entity writes via `TransactWriteItems`. Teams, rosters, event participation, games, standings,
schedules, playoffs, and circuits all describe one world model — *who plays, what happened, who won,
who qualifies* — and they share transaction surfaces (team registration/payment, event finalize) and
lifetimes.

## Decision

**Teams, rosters, event participation, games, standings, schedules, playoffs, and circuits all live
in the `event-registration` service** (the de-facto "competition" service). The only separate
bounded context is **players** (master + bio in `player-profiles-api`), referenced by UUID.
The `/events` API base path stays.

## Rationale

Splitting teams or circuits out would force cross-table reads or SPA-mediated writes for every
transaction that spans an event (roster snapshots at payment time, mid-event roster corrections,
points-on-finalize) — creating *more* denormalized copies, not fewer — and eventual consistency
where a single transaction suffices. Co-location means "is this team real? is this roster legal?"
are **local table reads**, never service calls. A playback check: with teams co-located, no
inter-service call is required anywhere in RFCs 0001–0004; only *players* cross a boundary
([ADR-0002](0002-player-reference-validation.md)).

## Consequences

- `event-registration` grows into a large-but-cohesive domain service. The repo name undersells it;
  a cosmetic rename (e.g. `competition-api`, path unchanged) can be considered later.
- Multi-entity operations stay single-transaction (registration, finalize).
- The Terraform-managed `event-registration` table gains new item types and one new index
  (see RFC-0001/0002); `infra/main.tf` and the local `docker-compose` dynamo-setup must be updated in
  lockstep.

## Revisit triggers (conditions that would justify extracting a service)

- External organizations onboard and need isolated, multi-tenant league management.
- Team identity must exist fully independently of events (e.g., open public team registration/federation).
- Independent deploy cadence or ownership for one of these sub-domains emerges.