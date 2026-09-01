# RFC-0001: Teams & Rosters

> Status: **Draft** · Date: 2026-08-31 (revised after adversarial review)
> Depends on: RFC-0005 (player UUIDs)
> Decisions: ADR-0001 (co-location), ADR-0002 (player validation), ADR-0003 (fan-out), ADR-0004 (admin authority)

## 1. Background

Today a "team" is just a `TeamRegistration`: a `REGISTRATION` item keyed by `EVENT#<eventId>` +
captain's email, carrying `teamName`, `homeCity`, `captainEmail`, and a `players[]` array. There is
**no cross-event team identity**. This RFC introduces a persistent **Team** entity, roster
membership that evolves over time, per-event **participation with roster snapshots**, and the
APIs/UI to manage them — **without breaking the paid registration flow**.

## 2. Goals / Non-goals (v1)

### Goals
- A global `Team` entity (UUID) that persists across events and circuits.
- Roster membership with history (join/leave, status); per-event **roster snapshots**.
- Team pages: roster, events played, records, links to events/players.
- The paid team-registration flow keeps working (captain email remains the money key).
- Admin-only management (ADR-0004).

### Non-goals (defer)
- Captain/player self-service; per-game lineups; player transfers/waivers; federation.
- **Auto-affiliation of legacy inline-`players[]` registrations to teams at payment time** — that
  work is done by the §9 migration tool instead.

## 3. Data model (additions to the `event-registration` table)

```
Team master:            PK = "TEAM#<teamId>"               SK = "TEAM#<teamId>"
                        id, name, normalizedName, homeCity, status (ACTIVE|ARCHIVED),
                        captainPlayerId, createdAt, updatedAt, Version

Team name reservation:  PK = "TEAM_NAME#<normalizedName>"  SK = "TEAM_NAME#<normalizedName>"
                        teamId, createdAt            (create-conditional; released on archive)

Roster membership:      PK = "TEAM#<teamId>"               SK = "MEMBER#<playerId>"
                        playerId, name, email (snapshot), role (CAPTAIN|MEMBER),
                        status (ACTIVE|INACTIVE), joinedAt, leftAt?, Version

Player current teams:   PK = "PLAYER#<playerId>"           SK = "MEMBER#<teamId>"
                        teamId, teamName, role, joinedAt, leftAt?
                        (same partition family as RESULT# items from RFC-0002/ADR-0003)

Event participation:    PK = "EVENT#<eventId>"             SK = "TEAM#<teamId>"
                        eventId, teamId, registrationId (→ REGISTRATION item), seed,
                        status (REGISTERED|CONFIRMED|WITHDRAWN|DNS),
                        rosterSnapshot [playerId...], rosterSizeAtEvent, Version

Team history:           PK = "TEAM#<teamId>"               SK = "EVENT#<eventId>"
                        eventId, eventName, eventDate, registrationId, status, result
```

**GSI rule (blocking/pinning):** only **EVENT masters** (`GSI1PK="EVENT"`) and **TEAM masters**
(`GSI1PK="TEAM"`, `GSI1SK="NAME#<normalizedName>#<teamId>"` — the new `TEAM_NAME` index) carry
`GSI1PK`/`GSI1SK`. Participation, membership, history, RESULT#, GAME, and CIRCUIT items carry **no
GSI attributes — enforced by an adapter test that proves `GetEvents` (GSI1 query,
`begins_with(GSI1SK,"EVENT#")`) returns exactly event masters after every item type is written.**
This prevents any new item type from silently corrupting the site's events listing.

The `TEAM_NAME` GSI must be added to **`infra/main.tf`, `icaa.world/docker-compose.yml`
dynamo-setup, and `dynamo/db_test.go` together**, and the adapter should fail fast at startup if the
index isn't `ACTIVE` (it backfills asynchronously after `terraform apply`).

Roster changes (add/reactivate/deactivate) write the team-side `MEMBER#` row and the player-side
`MEMBER#` row **in one transaction** — they cannot drift. Team history adjacency is part of the
event-registration transaction (§6), not a post-hoc write.

## 4. Player identity integration (server-side validation)

- Rosters store **player UUID + name/email snapshot**, sourced from the player directory
  (RFC-0005). Rosters never mint players: v1 minimal-player creation is admin-only via the profiles
  API (SPA or admin tooling).
- At roster-write time the events service calls the profiles **internal route** (machine token,
  ADR-0006) to verify the UUID and capture the authoritative current name/email. Writes **fail
  closed** if the profiles service is unreachable (ADR-0002). This is on **admin-path roster
  mutations only** — never on the public payment path.

## 5. Business rules

- **Team name** is unique case-insensitively (`normalizedName`), enforced via the
  `TEAM_NAME#<normalizedName>` reservation item created with `AttributeNotExists` in the team-create
  transaction (DynamoDB has no unique indexes; this closes the race). Released when the team is
  archived.
- **Captain** must be an ACTIVE roster member, identified by `captainPlayerId` (player UUID) with the
  captain's **email on the roster row** (the money key). Changing captain = admin update (old member
  drops to `MEMBER`).
- **Event eligibility** uses the existing `event.allowedTeamSizeRange` check against the event
  **roster snapshot** size. Snapshot defaults to the team's ACTIVE roster at entry; admins adjust it
  (someone can't make it), bounded by the range.
- **Roster change effects:** eligibility for circuits uses the snapshot (and `rosterSizeAtEvent`)
  frozen at finalize — post-finalize snapshot edits cannot retroactively change it (RFC-0004).
- **A team has at most one participation row per event** — enforced by `AttributeNotExists` on the
  participation key (a second entry → 409; explicit re-activation path).
- **Registration close stays time-based** — the existing `registrationCloseTime` is the sole trigger;
  moving the event to `IN_PROGRESS` does **not** auto-close it (decision D8, §Open decisions).
  Late teams use the admin `late-team` flow (RFC-0002 §7) which regenerates unplayed games preserving
  results.
- **Archived teams** cannot enter new events; history and projections remain.
- Players may be on **≥ 1 team at once** in v1 (open rule — §Open decisions).

## 6. Registration flow (teams) — the payment lifecycle

**New flow (SPA submits `teamId` — only ships once teams data is trustworthy):**
1. Load team (local read); must be ACTIVE.
2. Snapshot active roster; validate against `event.allowedTeamSizeRange`.
3. **One transaction:** `REGISTRATION` (keyed `REGISTRATION#<captainEmail>` — captain resolved
   server-side from the team; never client-supplied), `REG_INTENT` (Stripe), participation
   (`REGISTERED`, snapshot, `AttributeNotExists`), team-history row, event counters.
4. Stripe checkout metadata `EMAIL` = captain email (unchanged — the webhook resolves this row).
5. **Paid webhook transaction:** `REGISTRATION → paid`, delete `REG_INTENT`, participation →
   `CONFIRMED`. **The webhook write must PRESERVE `teamId`** — today `UpdateRegistrationToPaid`
   (`dynamo/registration.go`) re-marshals the registration via `registrationDynamo` and does a full
   `Put` (Version-conditioned); if `teamId` isn't carried through that mapping, a backfilled row gets
   its stamp silently deleted on payment. Either add `TeamID` to the dynamo model or convert the
   paid update to a minimal, attribute-preserving `UpdateItem SET Paid=true, Version=+1`.
   Regression test: stamp → `UpdateRegistrationToPaid` → `GetRegistration` still returns `teamId`.
6. **Expiry webhook transaction:** delete `REGISTRATION` + `REG_INTENT` + participation + team-history
   row, roll back event counters.

All three transaction shapes (intent/paid/expired) change **in one deployment with regression tests**
regressing the current money flow (including the `teamId`-preservation test above).

**Legacy flow (SPA's current inline-`players[]` payload) stays byte-for-byte as it is today** — no
team/participation items are created on this path, so no phantom teams, no S2S on the payment path,
and no name-based merges at runtime. Those registrations are affiliated to global teams by the §9
migration tool only.

## 7. API surface

| Method | Path | Auth | Description |
|---|---|---|---|
| GET | `/events/v1/teams` | public | Cursor-paginated directory; `search` by name |
| POST | `/events/v1/teams` | admin | Create team + initial roster (reserves name) |
| GET | `/events/v1/teams/{teamId}` | public | Team detail + active roster + recent events/records |
| PUT | `/events/v1/teams/{teamId}` | admin | Update name/city/captain/status (reserve name) |
| POST | `/events/v1/teams/{teamId}/roster` | admin | Add member (machine-token validate + snapshot) |
| PUT | `/events/v1/teams/{teamId}/roster/{playerId}` | admin | Role/reactivate |
| DELETE | `/events/v1/teams/{teamId}/roster/{playerId}` | admin | Deactivate (`leftAt`, both MEMBER rows tx) |
| GET | `/events/v1/{eventId}/teams` | admin | Participants + roster snapshots |
| PATCH | `/events/v1/{eventId}/teams/{teamId}/roster` | admin | Adjust event snapshot (blocked when FINALIZED) |
| POST | `/events/v1/{eventId}/late-team` | admin | Add team after generation (RFC-0002 §7) |
| POST | `/events/v1/{eventId}/registrations` | public | Extended for `teamId` (§6) |

## 8. UI (icaa.world)

- `/teams` directory; `/teams/:teamId` (roster, events played + records, links).
- Registration form: team selection + roster preview (swaps in **after** teams data is trustworthy —
  flag-gated; the old inline form stays live until then).
- Admin console: team/roster management, snapshot editor. **Scope these honestly** — every admin
  write UI is net-new (today's Edit/Delete in `EventRegistrationTable.tsx` are stubs).

## 9. Migration (data is small; done as admin-run scripts)

1. **Prerequisite:** RFC-0005's **email-backfill pass runs first** (seed JSON has no emails; without
   it the email→UUID match matches nothing). Match heuristically on (email, name) and emit a
   **dedupe/merge report** before minting any minimal players.
2. Group legacy `TeamRegistration`s by `(normalizedName, captainEmail)` — explicit **review report /
   merge tool**, never an auto-merge, for ambiguous names.
3. Mint `TEAM` + `MEMBER#` rows (UUID via profiles, name/email snapshot); participation + snapshots
   from each registration's `players[]`.
4. **Stamp `teamId` onto existing REGISTRATION rows with an additive `UpdateItem SET` only** (never a
   full `Put` — a stale full Put can race the paid-webhook write and flip `Paid` back to `false`).
   **Also fix the other direction:** `UpdateRegistrationToPaid` must not delete the stamp (see §6).
   Dry-run with a drift report; PITR is already enabled.
5. Enjoy migration runs twice with a zero-diff assertion (idempotency test).

## 10. Edge cases

- Duplicate/renamed teams; archived teams re-registering; captain leaves roster (block until
  reassigned); player on two teams; snapshot edits vs. paid registration; legacy registrations
  without emails; team with an empty active roster mid-season; same team entering twice (409 via
  create-condition); a team whose checkout expired (lifecycle §6); email clash with existing team at
  registration (reservation 409).

## 11. Checklist / rollout stages

**Stage A (additive, zero risk to payments):** teams API + admin UI + `TEAM_NAME` GSI (all 3 schema
sources) + roster-write validation (machine token).
**Stage B (backfill):** run §9 (after RFC-0005 email pass), verify, London merge report.
**Stage C (registration flow):** participation lifecycle transaction changes (intent/paid/expired +
regression tests), then the SPA form swap when teams are trustworthy.

- [ ] GSI `TEAM_NAME` in terraform + docker-compose + db_test; adapter index check
- [ ] `teams/` domain: Team, RosterEntry, Participation aggregates + ports
- [ ] Dynamo adapter + transactions (intent / paid / expired incl. participation lifecycle; paid write preserves `teamId`)
- [ ] Name reservation item + 409 on clash
- [ ] OpenAPI endpoints + codegen + handlers + tests
- [ ] Profiles client (codegen'd, machine token, response validation) wired to roster writes
- [ ] Migration tooling + dry-run/idempotency + email-pass prerequisite
- [ ] Frontend `/teams` pages; registration form swap (flag-gated stage C)
- [ ] Update `system-docs/docs/data.md` + `services.md`

## Open decisions

League-rule/product choices specific to teams & rosters. Keep global `D#`s; the
[index](../OPEN-DECISIONS.md) maps every decision to its owning RFC.

| # | Decision | Recommended default | Status |
|---|---|---|---|
| D8 | **Late registration** — when does event sign-up close? **Resolved:** keep the existing time-based `registrationCloseTime` — no change to the live paid flow (moving an event to IN_PROGRESS does not auto-close). Late teams still enter via the admin late-team flow. | Keep time-based close (no change) | [x] (2026-09-01) |
| D22 | **Player on multiple teams** — allowed in v1 (projection key already supports it via `RESULT#<eventId>#<teamId>`). | Allowed; review when self-service lands | [ ] |
| D23 | **Team name uniqueness** — enforced via name reservation + 409. Confirm (vs best-effort + merge). | Enforce via reservation | [ ] |
| D24 | **Post-launch team merges** — two teams turn out to be the same org: ship a merge operation or explicitly defer (keep survivor, re-point future events)? | Defer mechanics; future | [ ] |
| D26 | **Free agents** — individual (ByIndividual) registrations coexist but earn no circuit points in v1. Confirm. | Unscored in v1 | [ ] |