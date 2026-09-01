# ADR-0007: User JWTs migrate from HS256 (shared secret) to RS256 (login-owned key, JWKS verification)

> Status: **Accepted** · Date: 2026-08-31 (revised after round-2 adversarial review)
> Motivation: the security fix behind the machine-token work in [ADR-0006](0006-internal-http-machine-auth.md)

## Context

Today user **access and refresh tokens are HS256** signed with a single symmetric secret in SSM
`/jwtSigningKeys`, and **every service holds that same secret** to validate access tokens
(`auth/token/service.go` signs HS256; five stacks read the param at startup and **fail boot if it's
missing or unparseable** (`cmd/config.go` `os.Exit(1)`)). Because HS256 is symmetric, "can verify"
and "can sign" are the same thing: any Lambda can forge user tokens — including `ADMIN` tokens.

Code-level facts that constrain this migration (verified round 2):
- The auth lib hardcodes `*jwt.SigningMethodHMAC` in both validators and **never validates `iss` or
  `aud`** (despite docs claiming it).
- Every `openapivalidate.go` has only an `admin` scope validator; unknown scopes → 401.
- A session stays alive only while its refresh record exists in `login-api`
  (`REFRESH_TOKEN#<id>`); the access token is short-lived. **Deleting those records is a real
  global logout.**

## Decision

- **Algorithm:** user access + refresh tokens move from **HS256 to RS256**. `kid` namespaces:
  `user-*` (this ADR) and `machine-*` (ADR-0006). **No alg-confusion:** the verifier binds `alg` to
  the `kid` family — `user-*`/`machine-*` kids are only ever verified with `*jwt.SigningMethodRSA`
  and a parsed `*rsa.PublicKey`. Parsing uses `jwt.WithValidMethods(["RS256"])` and
  `jwt.WithAudience("icaa-api")` + `WithIssuer("icaa.world")` + `WithLeeway(30s)` — the new verifier
  **enforces aud/iss** (the current one does not). Regression test that must exist: a token signed
  HS256 (or `alg=none`) using any key bytes is rejected.
- **Key ownership / params (new names; `/jwtSigningKeys` untouched until cutover):**
  - `/userJwtSigningKeys` — **login-only** (IAM per-ARN), RSA private key(s).
  - `/jwtSigningKeys` — **stays byte-identical until cutover** (services still boot against it
    during the deploy), then **deleted entirely** — we do not retain or re-scope symmetric material
    for a grace window (see cutover).
  - Public keys: login publishes to `GET /login/.well-known/jwks.json` **and** to an SSM param
    `/jwtPublicKeys` **in the same rotation step** — the SSM copy is the **permanent availability
    floor**.
- **Service verification:** fetch JWKS at startup **non-fatally** (log + start empty; never abort
  boot), **also load `/jwtPublicKeys` from SSM**, cache both, lazily refetch on unknown `kid` —
  **consults JWKS then SSM (union)** (singleflight + minimum 30s interval + negative-cache unknown
  kids). Fail closed (401) only at verification time when no key exists. Keep the last-known-good
  set per instance. A failed `/jwtPublicKeys` write during rotation surfaces as a CloudWatch alarm
  (login-side) so the SSM floor cannot rot silently.
- **Rotation:** mint keypair → add `kid` → flip `currentKid` → publish public keys to JWKS + SSM in
  the same step; retain old `user-*` kids until prior refresh-period tokens expire (or the next
  forced logout). Rotation never requires a service redeploy.

## Cutover (simplified: forced re-login — accepted as the one-time cost)

**No dual-verify, no symmetric-key retention, no kid/alg census.** We deliberately accept that every
user re-logs-in once. The only load-bearing ordering rule: **services can verify RS256 before login
is allowed to issue it.**

1. **`auth` v0.4:** RS256-only verification (JWKS + SSM public keys), alg↔kid binding, aud/iss
   enforcement. **No HS256 acceptance.** (Removing the old path entirely also removes the
   alg-confusion surface.)
2. **Deploy every service** (events, articles, assets, donation, voting): bump the lib, load JWKS +
   SSM public keys, **and remove `/jwtSigningKeys` from the startup fetch in each `cmd/config.go`
   (plus the matching SSM IAM statement in each `template.yml`)** — removing the lib usage alone is
   not enough, because those fetches `os.Exit(1)` on a missing param; add a CI check (grep) that no
   non-login repo still references `jwtSigningKeys`. From this deploy onward, existing HS256 cookies
   401 and sessions die. **Steps 2 and 3 are a single maintenance window** — between them the site is
   down for every user (refresh still succeeds against HS256 login, and re-login issues tokens every
   service rejects), so do not pause between them.
3. **Flip `login`, then purge, back-to-back in the maintenance window:** start issuing RS256, then
   **purge all `REFRESH_TOKEN#*` records** in `login-api` (the authoritative global logout) and
   remove HS256 signing material. Two load-bearing details:
   - **Flip-before-purge ordering is required:** if the purge ran first, warm pre-flip `login`
     containers (old code until AWS recycles them) could accept HS256 refresh cookies and re-create
     records after the purge, resurrecting sessions.
   - **Residual access-token window:** access tokens are stateless — the purge ends *renewal*, not
     *validity*; tokens minted between flip and purge stay valid until their ≤1h TTL. Back-to-back
     execution bounds this to minutes (no `iat`/deploy-time machinery needed).

   The purge runs **out-of-band** (login's IAM has `GetItem`/`PutItem`/`DeleteItem` only — no `Scan`):
   `scan --filter-expression "begins_with(PK, :p)"` with `:p = REFRESH_TOKEN#`, **dry-run count
   first**, then delete. The filter must be exact — `CLIENT#` machine-credential records live in the
   same table and a sloppy filter kills the m2m path (ADR-0006).

   **Frontend change is REQUIRED (one line), not optional:** the SPA's 401→refresh-fail path does
   not clear cached `userInfo` (verified in `authMiddleware.ts` — it returns the 401 untouched), so
   without a change, cached users see authenticated-but-broken UI for up to an hour after the purge.
   Bump the `userInfo`/`authStatus` localStorage keys in this deploy **or** wire refresh-failure →
   clear into `authMiddleware`. Then users re-login via Google and get RS256 cookies.
4. **Retire `/jwtSigningKeys` safely:** rename the param to a tombstone at cutover, observe ≥ 24h
   (the Lambda warm-container recycle bound) with **zero** `missing SSM parameter` logs across the
   five service log groups, then delete it. Renaming has identical failure semantics to deletion, so
   it is a live probe for any stack that still fetches it.
5. **Cutover verification (scripted `make smoke-prod`, per repo):** fresh Google login → RS256 token
   → 200 on every service; an HS256-fixture token → 401 everywhere; unknown `kid` → JWKS/SSM refetch
   then clean 401/200 as appropriate. This suite is also what defines "botched" for the rollback
   below.

**Rollback of a botched flip** (defined by step 5's smoke failing) is: revert the `auth` pin in all
five service repos and redeploy **all five** — restoring `/jwtSigningKeys` **before** their first
old-version cold start (their config fetches exit on a missing param) — then redeploy the previous
`login` image. Neither "restore the param" nor "redeploy old login" alone does anything post-step-2:
no deployed stack reads the param anymore, and an HS256 login issues tokens every RS256-only service
rejects. Pre-write the revert checklist; run the step-5 smoke before touching anything.

### Why this is acceptable

- One-time UX cost for a niche-sport audience; an auth-scheme overhaul forcing re-login is normal.
- Native sessions (Google) make re-login low-friction — no passwords to remember.
- It eliminates: the dual-verify path (and its forgery surface), the 30-day symmetric-key retention,
  and the census/classification of in-flight tokens — the three most intricate pieces of the
  previous plan.

## Verification availability (login is NOT the site's availability root cause)

- Services must never abort boot because login or its JWKS is unreachable — the SSM `/jwtPublicKeys`
  floor keeps verification working during a login blip.
- JWKS served with `Cache-Control: public, max-age=300, stale-while-revalidate=3600` (one header for
  both user + machine keys). **The SSM `/jwtPublicKeys` floor is the load-bearing availability
  mechanism** — no edge cache exists in front of `api.icaa.world` today; the header only helps if
  one is ever added.

## Local dev

- Local mode uses a **development RSA keypair** (private in local login config; public in the shared
  local verification config), replacing the `local-development-signing-key` default. **No HS256 path
  survives anywhere.**
- **Prod code paths assert `!isLocal()` before ever installing the dev key** — `LOCAL` must be an
  explicit opt-in config, never reachable by env drift; add a test that a token signed with the dev
  private key is rejected when `LOCAL` is false.

## Consequences

- **`login` is the only forger.** A compromised web service can verify user tokens but cannot mint
  them.
- Services no longer read a symmetric signing secret; auth config is "login JWKS URL + SSM public
  keys, cached, non-fatal, fail-closed at verify".
- One-time re-login for all users during cutover (accepted).
- `docs/auth.md`, `docs/data.md` (SSM table), `docs/services.md`, and `docs/architecture.md`
  (component table + "frontend is the only consumer") must be updated at cutover.

## Rejected alternatives

- **Do nothing** — every Lambda holds the user-token signing key. Rejected once asymmetric machinery
  is being built anyway.
- **Graceful 30-day dual-verify window** — more moving parts (dual-verify code, symmetric-key
  retention, token census) for a migration cost (one re-login) the org is happy to pay. Rejected in
  favor of the forced re-login above.
- **Repurpose `/jwtSigningKeys` for RSA** — would crash every not-yet-migrated service at boot.
  Rejected; new params instead.
- **Per-service keypairs / KMS day one** — unnecessary moving parts / added latency; KMS recorded as
  a later upgrade path.

## Checklist

- [ ] `auth` lib: RSA verification, alg↔kid binding, `WithValidMethods`/`Audience`/`Issuer`/`Leeway`,
      JWKS cache (singleflight + negative cache + min-interval), **HS256/alg-none rejected**
      regression test
- [ ] `login`: RS256 issue; `GET /login/.well-known/jwks.json` declared in spec (`security: []`);
      `/userJwtSigningKeys` login-only; publish public keys to JWKS + SSM at once
- [ ] All services: bump `auth`, JWKS + SSM-public load (non-fatal), **remove `/jwtSigningKeys`
      from `cmd/config.go` fetch + `template.yml` IAM; CI grep gate**
- [ ] Cutover (single maintenance window): flip login → purge `REFRESH_TOKEN#*` (exact filter,
      dry-run, out-of-band) → **frontend userInfo/cache-key bump** → rename `/jwtSigningKeys`
      tombstone → observe 24h → delete → smoke (RS256 fixture 200, HS256 fixture 401)
- [ ] Local dev RS256 keypair; dev key unreachable in prod (`!isLocal()` + test)
- [ ] Update `docs/auth.md`, `docs/data.md`, `docs/services.md`, `docs/architecture.md` at cutover