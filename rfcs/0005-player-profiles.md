# RFC-0005: Player Profiles

> Status: **Draft**
> Date: 2026-08-31
> Depends on: RFC-0002 (history tab only; bio + directory ship first)
> Related: [ADR-0002](../adr/0002-player-reference-validation.md) (reference + snapshot), [ADR-0003](../adr/0003-player-results-projection.md) (derived results)

## 1. Background

The `player-profile-ui` branch of `icaa.world` (Feb 2026, stale vs `main`) renders player profiles
from static JSON committed to `src/components/profiles/` (~48 players), with tabs for
Overview/Season/Lifetime/Recent tournaments and an admin-only edit mode that never persists. The
older `player-profiles-api/DESIGN.md` proposed a Go service following `event-registration`
conventions, with admin-curated `tournamentResults`.

This RFC supersedes that design's data model in one material way: **tournament results are no longer
stored or curated here** — they are **derived** from competition data owned by the events service,
written as a per-player projection at event finalize
([ADR-0003](../adr/0003-player-results-projection.md)). The profiles service owns the player
**master record and bio**. The profile page aggregates both.

## 2. Goals / Non-goals (v1)

### Goals
- A public player master directory: paginated, searchable by name.
- Public read of player profiles (bio + linkable history sourced from the events service).
- Admin-only create/update/archive of profiles. Admin-only creation of **minimal player records**
  (name + email) so rosters can mint UUIDs for players without full profiles (RFC-0001).
- Seed/migrate the existing 48 static JSON profiles into the directory (new UUIDs minted per player).
- Player UUID as the canonical identifier; `email` stored for identity (future self-service).

### Non-goals (defer)
- Player self-service editing (link profile to login account via email). `userInfoContext` exposes
  email today, so this is a plausible later increment, but v1 is admin-written.
- Non-admin profile photo *uploads* — v1 photo changes are **admin-only**, uploaded via `assets-api`
  (see "Photo uploads" below); player/captain self-upload is deferred (PRD-0001 §8).
- Any tournament/result CRUD — results come from events (`GET /events/v1/players/{playerId}/history`).

## 3. Player identity & the events-service boundary

- Canonical id: **UUID** (`id`). Rosters and projections in the events service reference this UUID
  ([ADR-0002](../adr/0002-player-reference-validation.md)).
- `email` is stored on the master record for identity resolution (matching registrations, future
  self-service). It is not unique-enforced in v1 (legacy data may lack emails).
- `slug` does **not** become part of the schema — there is no legacy player-profiles service, so no
  pre-existing identity needs stable URL continuity. The static JSON filenames are transient frontend
  data; the migration mints fresh UUIDs and nothing references the slugs afterwards.
- **Server-side validation:** when the events service writes a roster entry containing a `playerId`,
  it makes a read-only call to this service's internal route to (a) confirm the player exists and
  (b) capture the authoritative current name/email snapshot. Writes fail closed if this service is
  unreachable ([ADR-0002](../adr/0002-player-reference-validation.md), transport =
  [ADR-0006](../adr/0006-internal-http-machine-auth.md) machine tokens).
- **Internal-route contract:** `GET /profiles/internal/players/v1/{id}` (spec-declared,
  `security: [icaaBearerAuth: [m2m:player-profiles]]` — exact scope, per ADR-0006) returns
  `{id, firstName, lastName, email?}` — `email` is optional on the record. **404** = unknown UUID
  (caller fails the write); **200 with `status: ARCHIVED`** = soft-deleted player — a roster *may*
  reference an archived player only for historical backfills, never new adds.
- **PII:** the **internal** route (auth'd) includes `email`; the **public** `GET /players/v1/{id}`
  must **omit `email`** — profiles are public pages; contact data is not.
- The events service stores the **name/email snapshot** with each UUID it references, so competition
  records render even if master data changes (historically accurate by design).
- Players are **soft-deleted** (`status: ARCHIVED`); UUID references never break.

## 4. Data model

Table: `player-profiles` (SAM-created, same shape conventions as other services). Single item type.
Enable **PITR** on this table (like the Terraform-managed `event-registration` one) — profile data is
user-facing and irreplaceable if a botched migration overwrites it.

```
PK = "PLAYER#<uuid>"    SK = "PLAYER#<uuid>"
attributes: id (uuid), email, firstName, lastName, position,
            homeCity, number, experience, profilePicture (URL),
            bio, status (ACTIVE|ARCHIVED), createdAt, updatedAt, Version

GSI1:  GSI1PK = "PLAYER"   GSI1SK = "NAME#<last>#<first>#<uuid>"
       → paginated, name-ordered directory listing (Query + cursor)
GSI2:  GSI1PK = "PLAYER_EMAIL"   GSI1SK = "EMAIL#<email>"
       → email lookup (identity resolution / self-service later)
```

`position` enum (matches the UI + existing JSON): `Rear Guard | Forward | Centerback | Flex`.
`number` and `experience` stay free-text in v1 (legacy format); normalize later.
`profilePicture` is a CDN URL string.

## 5. API design (player-profiles-api)

Base path: `https://api.icaa.world/profiles`. Mirrors the existing DESIGN.md minus result endpoints.

| Method | Path | Auth | Description |
|---|---|---|---|
| GET | `/players/v1` | public | Cursor-paginated directory; optional `search` (name) + `position` filters |
| GET | `/players/v1/{id}` | public | Full profile by UUID |
| POST | `/players/v1` | admin | Create full profile, or a **minimal player** (`firstName`, `lastName`, `email`?) that roster flows use to mint a UUID |
| PUT | `/players/v1/{id}` | admin | Update bio fields (optimistic lock on `Version`) |
| DELETE | `/players/v1/{id}` | admin | Soft-delete (`status: ARCHIVED`) |
| GET | `/internal/players/v1/{id}` | `icaaBearerAuth [m2m:player-profiles]` | Service-to-service existence + snapshot fetch, consumed by `event-registration` at roster-write via a machine token ([ADR-0006](../adr/0006-internal-http-machine-auth.md)). Never linked from the UI; declared in the spec (required by validation middleware + client codegen) |
| POST | `/players/v1/{id}/claim` | *(future)* | Self-claim by email — **not in v1** |

Response shape mirrors `GET /events/v1`: `{ data: [...], cursor, hasNextPage }`. Auth = shared
`icaaCookieAuth`/`icaaBearerAuth`, `admin` scope for writes (same pattern as all services).

### Profile-photo uploads (v1 — admin-only, via assets-api)

- Admins upload through the existing `assets-api` (admin-authenticated presigned upload + confirm,
  under a `profiles` folder) and then set `profilePicture` on the profile record to the resulting
  CDN URL. `player-profiles-api` stays URL-agnostic — it never calls `assets-api` for this, so no new
  service-to-service call is introduced (the first S2S call remains roster validation, ADR-0002).
- **Future — self-service uploads** would need a scoped path with per-user limits and a dedicated
  `profiles/<playerId>` location in the assets service — likely machine-auth or a dedicated
  non-admin presign scope (recorded as a revisit trigger in PRD-0001 §8). Explicitly out of v1.

## 6. Where results come from

The events service exposes player history:

| Method | Path (`event-registration`) | Auth | Description |
|---|---|---|---|
| GET | `/events/v1/players/{playerId}/history` | public | All `RESULT#<eventId>#<teamId>` projections + current team memberships for a player (one entry per team per event — a player on two teams in one event keeps both) |

Response shape (aggregated in the SPA with the bio):
```yaml
history:
  results:
    - eventId, eventName, eventDate, teamId, teamName,
      placement: int, record: {wins, losses, draws?}, pf, pa
  currentTeams: [teamId, teamName, role]
```

The profile page (`icaa.world`) therefore calls two APIs: profiles (bio) + events (history) — the
established SPA-as-aggregator pattern.

> **Dependency:** the history endpoint exists only once RFC-0002's finalize fan-out writes
> `RESULT#<eventId>#<teamId>` items. RFC-0005 can ship **bio + directory first** with the Events tab
> in an explicit empty/pending state; "0005 done" is redefined as "history tab live" only after
> RFC-0002 lands. Backfilled historical events (RFC-0002 §9) are run through finalize so history
> exists from day one.

## 7. Seed / migration

`cmd/seed` (or `make seed`):
1. Reads the 48 JSON files from the `player-profile-ui` branch (copied into this repo under
   `spec/seed/data/`), validates against the OpenAPI schemas.
2. Assembles each profile: id = new UUID, bio fields from JSON, `status: ACTIVE`. The JSON
   filenames are ignored (they were transient frontend data).
3. **Email-backfill pass (a hard prerequisite for RFC-0001's migration — and the existing DESIGN.md
   notes the JSON has no emails):** match profiles to registration data by name and any known
   emails, produce a **dedupe/merge report** for admin review, then set `email` on the records.
   Without this pass, the RFC-0001 email→UUID match would match nothing and mint duplicate minimal
   players.
4. Upserts via the same domain logic as the API (no HTTP round-trip).
5. Reports drift (players in `playerList.json` missing files, and vice versa).

> `player-profiles-api/DESIGN.md` is **superseded** by this RFC (its slug-as-`id` and
> admin-curated `tournamentResults` models contradict UUID identity + derived results). Mark it
> superseded before scaffolding the repo.

Roster migrations for existing team registrations are handled in RFC-0001 §Migration.

## 8. Frontend integration (`icaa.world`)

1. `bun run codegen` for both specs (`profiles`, `events` additions).
2. New `playerProfileQueryClientContext.tsx` (mirror existing contexts).
3. `/player-profiles/:playerId` page (public eventually; `AdminOnlyRoute` initially until data is
   trustworthy). Tabs: Overview (bio) / Events (history from events API) / Pictures.
4. Replace `src/components/profiles/*.json` on the branch — remove once seeded.
5. Roster flows (RFC-0001) use the player directory search to pick/create players.
6. Admin profile-photo upload (assets presign → confirm → set `profilePicture`) on the edit route.

## 9. Edge cases

- **Email uniqueness** — legacy data has missing emails; enforce uniqueness only when present, and
  hold off on hard guarantees until self-service lands. `email` is **not exposed on the public API**
  (PII — §3).
- **Name changes** — master record updates forward; competition snapshots preserve the past (good).
- **Position set** — confirm enum matches the UI select exactly (`Rear Guard | Forward | Centerback | Flex`).
- **Photo management** — v1: admin-only uploads via `assets-api` (presigned upload + confirm,
  `profiles` folder); public pages just render the CDN URL. Self-service uploads are deferred
  (PRD-0001 §8).
- **Player deletion** — archive only; never hard-delete (UUIDs referenced by rosters/projections).

## 10. Implementation checklist

- [ ] Create `player-profiles-api` repo + SAM skeleton (from `event-registration` shape)
- [ ] `spec/api.yaml` + `go generate ./...` (oapi-codegen)
- [ ] Domain (`players/`) + DynamoDB adapter + table in SAM template (**PITR enabled**)
- [ ] Handlers + tests (table-driven, mocked repository)
- [ ] Seed command + migration of 48 JSON profiles
- [ ] Deploy (`ApiMappingKey: profiles`) + verify against prod
- [ ] Codegen + query client + `/player-profiles/:playerId` page; delete static JSON
- [ ] Email backfill pass against registration data
- [ ] Admin profile-photo upload via assets-api (presign + confirm) + SPA edit UI

## Open decisions

League-rule/product choices specific to player profiles. Keep global `D#`s; the
[index](../OPEN-DECISIONS.md) maps every decision to its owning RFC.

| # | Decision | Recommended default | Status |
|---|---|---|---|
| D25 | **Public player pages** — public read immediately, or admin-only until data is trustworthy? | Admin-only initially, public after backfill verified | [ ] |