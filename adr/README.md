# Architecture Decision Records (ADRs)

ADRs capture **decisions** that shape multiple RFCs or the platform itself — the *why* behind the
design, so it survives. They complement `../rfcs/` (which describe *what* will be built).

Each ADR is a standalone file with a stable status:

- **Accepted** — decided; the design should follow it.
- **Proposed** — under discussion.
- **Superseded** — replaced by a later ADR (linked in its header).

## Index

| # | Title | Status | Date |
|---|---|---|---|
| 0001 | [Competition domain co-located in event-registration](0001-competition-domain-co-location.md) | Accepted | 2026-08-31 |
| 0002 | [Player references validated server-side (reference + snapshot)](0002-player-reference-validation.md) | Accepted | 2026-08-31 |
| 0003 | [Player results derived via per-player projection](0003-player-results-projection.md) | Accepted | 2026-08-31 |
| 0004 | [Roster authority is admin-only for v1](0004-roster-authority-v1.md) | Accepted | 2026-08-31 |
| 0005 | [Schedules & results use generate-then-edit](0005-schedule-generate-then-edit.md) | Accepted | 2026-08-31 |
| 0006 | [Service-to-service HTTP with JWT machine auth](0006-internal-http-machine-auth.md) | Accepted | 2026-08-31 |
| 0007 | [User JWTs migrate to RS256 (login-owned key, JWKS verification)](0007-user-jwts-rs256.md) | Accepted | 2026-08-31 |

Numbering is global (not per-RFC). RFCs and system docs reference ADRs by number.