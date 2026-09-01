# PRD-0001: Competition Platform

> **Status:** Draft · **Date:** 2026-09-01 · **Owner:** Andrew Mellen
> **Scope:** Tournament management (teams, events, playoffs, circuits) · Player profiles ·
> Accounts & auth · The website · Platform foundation
>
> This is the first PRD written for the ICAA platform, and the **canonical product spec** for the
> ICAA Competition Platform — the *what* and *why* we are building. It is the reference we check RFCs
> against when we revise or implement them, and the single document a non-engineer (league board,
> organizers) can read to understand scope, priorities, and decisions.
>
> **Context — this is new functionality on an existing, live platform.** The platform already runs in
> production (accounts & logins, news articles, donations, paid event registration, voting, the
> website). This PRD does not re-build or re-platform that; it defines a coherent program of **new
> features layered on top of the existing system**. Nothing here assumes a greenfield — every change
> must preserve what already works, above all the paid registration flow.
>
> **Reading guide:** product goals, personas, scope, sequencing, and decisions live here, in user
> terms. *How* each feature is designed (data, interfaces, migration) lives in the linked **RFCs**;
> cross-cutting technical decisions live in the linked **ADRs**; and the not-yet-decided league-rule
> choices live in **OPEN-DECISIONS.md**. If an RFC and this PRD ever disagree on *what* we are
> building, this document wins — fix the RFC.

---

## 1. What we're building

A product-level upgrade to the ICAA platform that turns each tournament from a static page into a
real, connected competition: teams that persist across events, live schedules and standings, playoff
brackets, points-based circuits, and living player profiles — on a security foundation that lets the
platform's own parts work together safely.

Today the platform's competition features are effectively **static**: only two past events have any
schedule/standings content (built by hand) and every other event shows "coming soon"; a "team" only
exists within a single event's registration, with no identity across events; player profiles are
static files; and there is no points/circuit system. The PRD below defines the product we are working
towards — **added onto the existing live platform**, not replacing it.

## 2. Who it's for

| Persona | Today's pain | What they get |
|---|---|---|
| **League / tournament organizers (admins)** | Results are hand-entered or hardcoded; no teams-of-record; no points system; profiles are static files | Admin-managed teams & rosters, schedule + score entry, playoffs, circuit points, player directory |
| **Team captains** | A "team" resets every event; no history or records to point to | A persistent team with roster, events played, records (registration still admin-mediated in v1) |
| **Players / members** | No page that shows their tournament history | A player profile with bio + verified event history & records |
| **Spectators / visitors** | Events with no schedule, standings, or brackets; no way to explore teams/players | Public, always-current schedules, standings, brackets, directories, and player pages |

## 3. Goals & non-goals

### Goals (product-level)
- **Persistent competition identity:** a team and a player are the same entity across events, so
  history and records accumulate instead of being re-created each time.
- **Every event is live:** schedule, standings, and (where applicable) playoff brackets render from
  current data — no event shows "coming soon", no hand-entered result data.
- **Admin can run the full lifecycle** of an event and a season: enter teams → generate schedules →
  record scores → run playoffs → finalize → award circuit points — without manual spreadsheet work.
- **Player profiles are living pages:** bio + derived, trustworthy history (never re-keyed by an
  admin).
- **Circuits give the season structure:** teams earn points across member events and qualify for a
  championship.
- **The money path keeps working through every phase:** the paid team-registration flow keeps working
  end-to-end; rule changes that touch it (e.g. how late registration closes) are surfaced for a
  decision before they land.

### Non-goals (deferred within this program — not what we're building here)
- Captain/player **self-service** roster and profile editing (v1 is admin-managed).
- Player transfers / waivers / team splits-and-merges; free-agent circuit scoring.
- Live score-entry timers and per-game lineups; roster **lock** windows (documented, not enforced).
- Player circuit points, profile photo **uploads** (URL in v1), personalized dashboards.

## 4. Feature scope

Each feature is specified by an RFC. Product-level description here; **design lives in the RFC.**

### 4.0 Platform foundation — accounts & security *(enabler, not a user-facing feature)*
Required before the competition features can depend on each other; owned by
**[ADR-0006/0007/0008](../adr/README.md)**. In plain terms, this is about **who's allowed to do what
and making it safe for the platform's own parts to talk to each other** (needed once teams need to
check that a roster player is a real player):

- **Safer logins** ([ADR-0007](../adr/0007-user-jwts-rs256.md)) — today a leaked signing key could
  forge admin logins. This changes verification so only the account system can issue logins.
- **Trusted connections between systems** ([ADR-0006](../adr/0006-internal-http-machine-auth.md)) —
  lets the platform's parts verify each other securely when one needs data from another.
- **Cleaner sessions & admin control** ([ADR-0008](../adr/0008-session-admin-hygiene.md)) — logging
  out actually ends a session, sessions can't be replayed, and removing an admin takes effect
  quickly.

> **Operational note:** the login upgrade ships as a **brief, scheduled maintenance window**. Because
> logging in currently only meaningfully matters for admins, the only direct user-visible change is
> that **admins sign back in once** afterward. The team schedules the window to avoid in-flight paid
> registrations.

### 4.1 Player Profiles — [RFC-0005](../rfcs/0005-player-profiles.md)
A real player directory and individual pages. Admins manage profiles (including creating lightweight
player records so rosters can refer to real players). Public pages are shown only after data is
verified. **User outcome:** every rostered player has a page and (once history ships) a verified
record.

### 4.2 Teams & Rosters — [RFC-0001](../rfcs/0001-teams-and-rosters.md)
Persistent global teams with evolving rosters, a per-event "who's on the team" record, a public team
directory, and team pages (roster, events, records). Admin-managed in v1; event sign-up only uses
real teams at the end, once team data is trustworthy. **User outcome:** a team is a lasting thing
with history, not a one-off registration.

### 4.3 Games, Schedules & Standings — [RFC-0002](../rfcs/0002-games-schedules-standings.md)
Events get a real lifecycle, and schedules are **generated then edited**: round-robin and swiss
schedules, score/results entry, standings derived live from games, and a finalize step that locks the
event and publishes results. **User outcome:** any event page shows a real, current schedule +
standings.

### 4.4 Playoffs — [RFC-0003](../rfcs/0003-playoffs.md)
Single-elimination (first) and double-elimination (later) brackets generated from final qualifying
standings, rendered as a tree, with a decided champion and an unambiguous final placement order.
**User outcome:** events can crown a real champion and everyone knows the final order.

### 4.5 Circuits — [RFC-0004](../rfcs/0004-circuits.md)
Points across a chain of member events, configurable per circuit (points tables, roster-eligibility
rules, qualification), with standings and qualification to a main championship event. Roster
eligibility is evaluated on the roster as recorded for each event, so it can't be gamed by last-minute
changes. **User outcome:** a season has a leaderboard, and the championship field is filled by merit.

## 5. How we ship it (product roadmap)

Ordering below is the **recommended, product-framed** sequence — the authoritative staged build plan
stays in [rfcs/README.md](../rfcs/README.md#recommended-build-order); detailed checklists live inside
each RFC.

| Phase | Ships | Product context | Depends on |
|---|---|---|---|
| **0 — Foundation** | The accounts & security foundation ([ADR-0006/7/8](../adr/README.md)) | The groundwork that lets the platform's parts depend on one another safely | — |
| **1 — Player identity** | Player profiles: bio + directory ([RFC-0005](../rfcs/0005-player-profiles.md)) | Players get real identities the rest of the platform can rely on | 0 |
| **2 — Team identity** | Teams & rosters ([RFC-0001](../rfcs/0001-teams-and-rosters.md)) | Teams get persistent identity and histories, added alongside what exists today — signing up for an event doesn't change yet | 1 |
| **3 — Events go live** | Games, schedules & standings ([RFC-0002](../rfcs/0002-games-schedules-standings.md)) | Every event shows a real schedule/standings and can publish results; recreate the history of the two past events | 2 |
| **4 — Playoffs** | Playoffs ([RFC-0003](../rfcs/0003-playoffs.md)) | Events can crown a champion with a clear final order | 3 |
| **5 — Circuits** | Circuits ([RFC-0004](../rfcs/0004-circuits.md)) | Teams earn points across a season and qualify for a championship; points build on the playoff order from Phase 4 | 3, 4 |
| **6 — Complete experience** | Sign-up uses real teams; profiles show live history ([RFC-0001](../rfcs/0001-teams-and-rosters.md), [RFC-0005](../rfcs/0005-player-profiles.md)); double-elimination playoffs ([RFC-0003](../rfcs/0003-playoffs.md)) | Now that team identity is trustworthy, signing up can use it; profiles get their history tab | 3, 4, 5 |

> **Hard rule:** the paid team-registration path is never broken. Phases 1–2 add new capabilities
> without touching sign-up; the change to sign-up lands **last** (Phase 6), after team data is
> trustworthy.
>
> **Scheduling nuance (Phase 1):** player profiles (bio + directory) could deliver before the
> security foundation — it's a fresh area that doesn't depend on existing account systems. It's shown
> after Phase 0 only because later team work needs that foundation.
>
> **Ordering within Phase 6:** double-elimination playoff support lands before switching event sign-up
> to use real teams.

## 6. Product decisions to confirm (the open league-rule/rule choices)

These are real product choices (mostly league rules) that the team must confirm. The recommended
default is recorded so implementation isn't blocked. **The full log (28 items) is in
[OPEN-DECISIONS.md](../rfcs/OPEN-DECISIONS.md); this is the shortlist that must be decided *before
corresponding work starts*.** In user terms:

| Decision | The question, in plain language | Recommended default |
|---|---|---|
| **D8 — When registration closes** | When does a team stop being able to sign up for an event, and how do late teams get in? | Registration closes when the event starts; late teams only via admin. *Changes the live paid flow — decide first.* |
| **D2/D3/D4 — Forfeits & withdrawals** | What's the score when a team forfeits or withdraws mid-event? Do both teams get a loss if both no-show? | Played games stand; remaining games vs a withdrawn team = a default forfeit win (5–0); double-forfeit = both lose 0–0. Fixed per event when the schedule is made. |
| **D11 — Stuck-bracket escape hatch** | If a playoff bracket can't resolve (disputed/no-show game), can the event still be finalized? | Finalize refuses by default; an explicit "qualifying-only" mode is an opt-in that must be reachable so no event gets permanently stuck. |
| **D19/D20 — Not enough qualifiers / declines** | If fewer teams qualify than slots, or a qualified team declines, what happens? | Run with a short field; fill declined slots from the next-eligible teams. *External people-and-money consequences — decide first.* |

The remaining decisions (draws, byes, points tables, roster-stability rules, eligibility rules,
players on multiple teams, when pages become public, etc.) are safe to ship with their recorded
defaults, but are all listed for confirmation in
[OPEN-DECISIONS.md](../rfcs/OPEN-DECISIONS.md).

## 7. How we'll know it's working (product-level success criteria)

"Done" for this program is not just "the capabilities exist" — it's measurable product outcomes:

- **No event shows "coming soon":** every event page shows a live schedule and standings; the two
  hand-built event pages become the real record (Phase 3) and history reads from live data.
- **Teams are persistent:** a team's page shows its history across its actual events, and (in the final
  phase) signing up for an event lets you pick your real team.
- **Player pages are real:** all 48 static player profiles become live pages whose history is drawn
  from the events they actually played — not re-keyed by an admin.
- **Admin runs an event end-to-end:** an admin can take one event from sign-up through scheduling,
  scores, and (where applicable) playoffs and results with correct standings and placements — no
  spreadsheets.
- **Circuits compute:** standings and a championship field are produced from a season of member events
  without manual point entry.
- **The money path is unchanged:** the experience of registering and paying for an event is unchanged
  for teams and captains through every phase.
- **Security foundation holds:** a security review confirms only the account system can issue logins,
  logging out actually ends a session, and the platform's parts verify each other instead of trusting
  page input.

## 8. Future program candidates (out of scope now, with revisit triggers)

These are consciously **not** part of this program, but are natural candidates for a future PRD. Each
is listed with the trigger that would make us revisit it:

- **Captain/player self-service** — revisit once a player's login is linked to their profile (v1 keeps
  rosters and profiles admin-managed).
- **Player transfers & waivers, team splits/merges** — revisit after persistent teams are live and real
  post-launch pain appears.
- **Federation / external-org multi-tenancy** — revisit if other organizations want to run isolated
  leagues/events on the platform.
- **Free-agent (individual) circuit scoring / player circuit points** — revisit after team circuits
  are proven.
- **Profile photo uploads** — revisit once there's a natural upload path (URL-only in v1).

For what's deferred purely *within* this program, see §3 Non-goals.

## 9. References

| Doc | What it owns |
|---|---|
| [RFC-0001](../rfcs/0001-teams-and-rosters.md) | Teams & Rosters design |
| [RFC-0002](../rfcs/0002-games-schedules-standings.md) | Games, Schedules & Standings design |
| [RFC-0003](../rfcs/0003-playoffs.md) | Playoffs design |
| [RFC-0004](../rfcs/0004-circuits.md) | Circuits design |
| [RFC-0005](../rfcs/0005-player-profiles.md) | Player Profiles design |
| [ADR-0001–0008](../adr/README.md) | Cross-cutting technical decisions (why) |
| [rfcs/OPEN-DECISIONS.md](../rfcs/OPEN-DECISIONS.md) | Live league-rule/product decision log |
| [rfcs/README.md](../rfcs/README.md) | RFC lifecycle + staged build order |
| [docs/](../docs) | How the system *is* today (reference) |

## 10. Risks & rollout notes (supplemental — product terms)

These are the things most likely to disrupt the live system; the team should plan around them:

- **Historical data completeness (the two past events).** Recreating their full results is a
  high-effort, reconstructive job: some playoff games are **missing** from the current hand-built data
  and team names were written inconsistently (e.g. "Renegades" vs "Boston Renegades"). The recreated
  history must be verified correct before it becomes the public record.
- **Paid-registration safety.** The register→pay→confirm flow is the business's money path. The one
  change that touches it (associating a real team during checkout) must be carefully protected so a
  payment can never be silently dropped or mis-attributed.
- **Publishing results can be interrupted.** Publishing an event's results may be interrupted partway;
  the system is designed to resume and retry, and to raise an alarm if a publish never completes — the
  recovery step is to re-run the publish, confirm it finished, and clear the alarm.
- **Auth upgrade window.** The login overhaul is a brief, scheduled maintenance window; admins sign
  back in once afterward (see §4.0). Schedule it to avoid in-flight registrations.
