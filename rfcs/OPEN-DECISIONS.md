# Open team decisions — index

League-rule and product choices the ICAA team must confirm. **Each RFC owns the decisions that apply
to it**, in an `## Open decisions` section — marked `[ ]` until confirmed, `[x] (date)` once decided
— so a question lives next to the design it constrains. This page is the **index** of where each
decision lives, plus a couple of cross-cutting notes that aren't per-feature questions.

## Index

| # | Decision (short) | Lives in |
|---|---|---|
| D1 | Draws in games | RFC-0002 [§Open decisions](0002-games-schedules-standings.md#open-decisions) |
| D2 | Forfeit/withdrawal score | RFC-0002 [§Open decisions](0002-games-schedules-standings.md#open-decisions) |
| D3 | Mid-event withdrawal | RFC-0002 [§Open decisions](0002-games-schedules-standings.md#open-decisions) |
| D4 | Double forfeit | RFC-0002 [§Open decisions](0002-games-schedules-standings.md#open-decisions) |
| D5 | DNS / never-played teams | RFC-0002 [§Open decisions](0002-games-schedules-standings.md#open-decisions) |
| D6 | Bye representation | RFC-0002 [§Open decisions](0002-games-schedules-standings.md#open-decisions) |
| D7 | Swiss tiny fields | RFC-0002 [§Open decisions](0002-games-schedules-standings.md#open-decisions) |
| D8 | Late registration | RFC-0001 [§Open decisions](0001-teams-and-rosters.md#open-decisions) |
| D9 | Placement authority | RFC-0003 [§Open decisions](0003-playoffs.md#open-decisions) |
| D10 | Points for playoff depth | RFC-0004 [§Open decisions](0004-circuits.md#open-decisions) |
| D11 | Stuck brackets | RFC-0003 [§Open decisions](0003-playoffs.md#open-decisions) |
| D12 | `fieldSizeEffect` | RFC-0004 [§Open decisions](0004-circuits.md#open-decisions) |
| D13 | `eligibilityMode` | RFC-0004 [§Open decisions](0004-circuits.md#open-decisions) |
| D14 | Stability bar | RFC-0004 [§Open decisions](0004-circuits.md#open-decisions) |
| D15 | Roster lock window | RFC-0004 [§Open decisions](0004-circuits.md#open-decisions) |
| D16 | Tied placements | RFC-0004 [§Open decisions](0004-circuits.md#open-decisions) |
| D17 | Drop-lowest | RFC-0004 [§Open decisions](0004-circuits.md#open-decisions) |
| D18 | Mid-season join | RFC-0004 [§Open decisions](0004-circuits.md#open-decisions) |
| D19 | Qualification shortfall | RFC-0004 [§Open decisions](0004-circuits.md#open-decisions) |
| D20 | Qualified-team declines | RFC-0004 [§Open decisions](0004-circuits.md#open-decisions) |
| D21 | Main event as a member event | RFC-0004 [§Open decisions](0004-circuits.md#open-decisions) |
| D22 | Player on multiple teams | RFC-0001 [§Open decisions](0001-teams-and-rosters.md#open-decisions) |
| D23 | Team name uniqueness | RFC-0001 [§Open decisions](0001-teams-and-rosters.md#open-decisions) |
| D24 | Post-launch team merges | RFC-0001 [§Open decisions](0001-teams-and-rosters.md#open-decisions) |
| D25 | Public player pages | RFC-0005 [§Open decisions](0005-player-profiles.md#open-decisions) |
| D26 | Free agents | RFC-0001 [§Open decisions](0001-teams-and-rosters.md#open-decisions) |

## Cross-cutting (not per-feature product questions)

- **D27 — ADR-0006 status** — resolved (2026-08-31): accepted as written. Recorded in the
  [ADR-0006](../adr/0006-internal-http-machine-auth.md) header; no open product question remains.
- **D28 — `docs/architecture.md` update** — a follow-up task, not a decision: when the first
  service-to-service call lands, amend `docs/architecture.md`'s "frontend is the only consumer"
  wording. Track it with the ADR-0006 implementation work.

## How to use this

- Every decision has a global `D#`, but **lives in the RFC whose design it constrains**. Confirm it
  in that RFC's `## Open decisions` table (mark `[ ]` → `[x] (date)`).
- Where the default is already implemented as described, flipping a decision may require a small
  config/migration; where noted, it changes behavior only (no schema churn).
- The items previously marked "decide before code" were **resolved at design review (2026-09-01)**:
  D8 (keep time-based registration close), D2/D3/D4 (admin-recorded forfeits/withdrawals), D11 (no
  `qualifyingOnly` mode), and D19/D20 (admin-managed qualification). The remaining open rows are
  safe to ship with their recorded defaults.