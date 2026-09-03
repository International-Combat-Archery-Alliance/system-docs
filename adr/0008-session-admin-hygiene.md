# ADR-0008: Session & admin hygiene in the existing `login` (security hardening)

> Status: **Accepted** · Date: 2026-08-31
> Motivation: round-4 threat model found live-code defects that predate (and are not fixed by)
> the RS256 migration in [ADR-0007](0007-user-jwts-rs256.md).
>
> Update 2026-09: item 1 (logout revocation) is fixed — `DELETE /login/session` accepts the
> refresh cookie (OR'd with the access cookie) and deletes the record when presented (INT-6).
> Items 2–3 remain open.

## Context

The round-4 security threat model verified three defects in the **current** `login` implementation
that exist today, independent of any planned change:

1. **Logout does not revoke the refresh token.** `DELETE /login/session` is scoped to
   `icaaCookieAuth` (the access cookie), and the handler reads the refresh-token id from context via
   `GetRefreshTokenIDFromCtx` — which is only ever populated by `icaaRefreshCookieAuth`
   (verified: `session.go`, `openapivalidate.go`). The `REFRESH_TOKEN#<id>` record is therefore
   **never deleted** on logout; a previously-captured refresh cookie stays valid for up to 30 days.
   `docs/auth.md`'s "logout deletes the refresh record" is false.
2. **Refresh rotation has a reuse race.** `refresh.go` does Get → Delete → Save non-atomically,
   tolerates a failed Delete ("continue anyway"), and has **no reuse detection**
   (`replacedBy` tracking). Concurrent use of the same refresh cookie (attacker racing a victim) can
   yield **two parallel live sessions**; a failed Delete lets a replayed old token rotate again.
3. **Admin revocation has a ≤30-day tail.** Roles are frozen in the refresh record at issue and
   re-issued on every refresh. Removing an email from `/adminEmails` does not take effect until the
   refresh record expires (≤30 days) or the token-cache clears. There is no per-token revocation
   (e.g., `jti` denylist) and no de-admin purge.

## Decision

Fix all three in the current code as a standalone hardening workstream (independent of the RS256
migration):

1. **Logout revokes:** scope `DELETE /login/session` to require the refresh cookie (or resolve the
   refresh record by principal via a new `GSI1` on `userEmail`) and delete the record in the handler.
2. **Atomic, reuse-detected rotation:** make refresh rotation atomic (conditional delete + server-side
   singleflight), and store a `replacedByTokenID` link on the new record. If an old token is ever
   presented after rotation ("reuse detected"), **alert and kill both** (delete the record; require
   re-login).
3. **De-admin purge:** when `/adminEmails` changes (or a periodic job), delete `REFRESH_TOKEN#*`
   records for affected users (login owns the table — do it in-band via a `userEmail` index) so a
   removed admin's `ADMIN` tail is ≤1h (access-token TTL), not up to 30 days.

## Also captured (broader hardening backlog, not code bugs)

- **Restricted payment keys:** `event-registration`/`donation-api` hold the full Stripe secret today.
  Track down **restricted keys** (no refund/disbursement rights) and drop donation's unused
  `stripeEndpointSecret`; treat the Turnstile secret as a real credential (it un-gates all public
  forms).
- **PITR on all tables:** only `login-api` (SAM) and the Terraform-managed `event-registration` table
  have PITR. Enable it for articles/assets and the planned `player-profiles` table (already required
  by RFC-0005).
- **No-header-leak middleware test:** enforce that the auth middleware never logs token material and
  that OpenAPI validation errors don't echo wrapped verifier internals verbatim into client responses.
- **Observability for auth-health:** alarms for sustained unknown-`kid` refetches, empty JWKS cache
  > 30 min, and m2m `invalid_client` spikes (see ADR-0006).

## Consequences

- The RS256/ADR-0007 cutover's forced re-login makes the logout/rotation fixes *temporarily* moot in
  prod, but they remain defects in the codebase and must be fixed before (or alongside) — otherwise a
  captured refresh cookie outlives the migration.
- These are security fixes with their own deploy + regression tests; do them as their own small
  workstream, not a checkbox inside a feature RFC.

## Checklist

- [x] `DELETE /login/session` revokes the refresh record (cookie-scoped or principal-indexed)
- [ ] Atomic refresh rotation + `replacedByTokenID` reuse detection (alert + kill both)
- [ ] De-admin purge on `/adminEmails` change
- [ ] Restricted Stripe keys for event-registration/donation; drop unused donation endpoint secret
- [ ] PITR enabled on remaining tables; auth-health alarms
- [ ] No-token-in-logs + no-error-echo regression tests