# ADR-0004: Roster authority is admin-only for v1

> Status: **Accepted** · Date: 2026-08-31
> Applies to: RFC-0001

## Context

Teams become persistent global entities (RFC-0001), so "who may change the roster" is a new
authorization question. The platform currently has a single `admin` role on the JWT
(`roles: ["ADMIN"]`); there is no per-resource permission model.

## Decision

For v1, **only admins** may:

- create teams,
- add/remove/inactivate roster members,
- adjust per-event roster snapshots,
- create **minimal player records** (via `player-profiles-api`, RFC-0005).

Captains/players get **no self-service** yet.

## Rationale

Keeps authorization to the existing `admin` scope — no new roles, scopes, or capability checks are
needed in the shared `auth` library. Admins already perform every analogous operation today (event
creation, roster viewing, CSV export). Self-service can be layered on later once player→login email
identity exists (RFC-0005 §Non-goals), at which point a `captain:<teamId>` scope or a per-resource
check can be added to the validation layer.

## Consequences

- Simpler authz surface; no captain/player role added to JWTs.
- Captains cannot fix their own roster errors — acceptable for v1; an explicit revisit trigger.