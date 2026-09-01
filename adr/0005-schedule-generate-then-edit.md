# ADR-0005: Schedules & results use generate-then-edit

> Status: **Accepted** · Date: 2026-08-31
> Applies to: RFC-0002, RFC-0003

## Context

Today match schedules and standings are hardcoded for exactly two events in
`icaa.world/src/components/EventPageTemplate.tsx`; every other event shows "coming soon". The
platform needs a real workflow for creating games and reporting results.

## Decision

- Admins **generate** schedules from an event's **CONFIRMED (paid) participation** (RFC-0002 §4)
  using pure, deterministic generators (round robin, swiss, playoff brackets), then **edit**
  results/status as games complete.
- Standings are **derived deterministically from games at read time** (an event has at most a few
  dozen games — trivial to compute), not stored.
- Supported v1 formats: **round robin, swiss, single-elimination playoffs, double-elimination
  playoffs**; the playoff generator consumes the event's **qualifying-round final standings** as its
  seed input.

## Rationale

Generation saves admin labor and removes the hardcoded-data problem; keeping standings derivable
from an immutable game history avoids write-path consistency bugs; explicit closing of a generation
window ("replace only when no games completed") prevents unintentional data loss when admin
regenerates.

## Consequences

- New `schedules/` (or equivalent) domain package in `event-registration` with pure, unit-testable
  generators (circle-method round robin, point-group swiss pairing, seeded-elimination bracket
  trees).
- Fortfeits/cancellations are first-class game statuses; their standings semantics are league rules
  (RFC-0002 §Edge cases), configurable later.