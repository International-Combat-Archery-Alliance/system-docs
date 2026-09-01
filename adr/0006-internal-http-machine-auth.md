# ADR-0006: Service-to-service HTTP with JWT machine auth (RS256)

> Status: **Accepted** · Date: 2026-08-31 (revised after round-2 adversarial review)
> Applies to: ADR-0002 (the first consumer: player existence verification from `event-registration`
> → `player-profiles-api`)
> Related: [ADR-0007](0007-user-jwts-rs256.md) — user tokens migrate to the same RS256/JWKS model

## Context

The platform has no service-to-service communication today, but [ADR-0002](0002-player-reference-validation.md)
needs `event-registration` to read a player from `player-profiles-api` at roster-write time.
Requirements:

1. Transport preference is plain HTTP.
2. The S2S call must be distinguishable from ordinary browser traffic.
3. It must be machine-authenticated.
4. **The auth must be declared in the OpenAPI spec** — not gateway/IAM magic.
5. **No non-`login` service ever holds a private signing key.** Only `login` signs; every other
   service verifies with public keys served by `login`.
6. **`login`'s availability must not be the platform's availability root cause.**

## Decision

### Transport & identity

- **Transport:** HTTP/1.1 from one Lambda to another via the callee's existing HTTP API.
- **Machine identity:** each caller has `clientId` + `clientSecret` in SSM (`/m2m/<clientId>/secret`).
  Secrets are **≥ 32 bytes of CSPRNG output**, validated by `login` via client-credentials exchange
  (`POST /login/v1/m2m-tokens`), verified by bcrypt hash + constant-time compare (appendix).

### Tokens: RS256, login is the only signer

- Machine JWTs are **RS256**, signed by `login` with a keypair namespaced `machine-*`, kept separate
  from user-token `user-*` keys (ADR-0007). Private key in `/machineJwtSigningKeys` — **login-only
  IAM** (per-ARN). Store **one keypair per SSM parameter** (or use the Advanced tier) — a Standard
  `SecureString` is capped at 4 KB and two RSA-2048 keys already exceed it on the first rotation.
  Upgrade path: sign via AWS KMS (no plaintext private key at all).
- **Claims:** `{sub: <clientId>, token_type: "machine", roles: ["m2m:<callee-scope>"], aud:
  "<callee>-api", iss: "icaa.world", iat, exp (5 min)}`, header `{"alg":"RS256","kid":"machine-*","typ":"JWT"}`.
  **`aud` is per-callee** (the callee service's own audience, e.g. `profiles-api`), NOT the global
  `icaa-api` — so a leaked machine token is not replayable against a second service even if it
  declared the same scope string. `ValidateMachineToken` enforces the callee's specific `aud` +
  exact scope.
- **Scope is exact, never prefix.** The callee's internal route requires the **literal scope for that
  callee** (e.g. `m2m:player-profiles` for `player-profiles-api`). A token minted for `clientId A`
  with `roles: ["m2m:player-profiles"]` is **not** valid on a route requiring `m2m:other`. The auth
  lib provides a **distinct `ValidateMachineToken`** that never flows through `ValidateAccessToken`
  (two separate token paths by design — user routes reject `token_type: machine`; internal routes
  reject `token_type: access`).

### Public-key distribution (login is not an availability root cause)

- `login` serves **`GET /login/.well-known/jwks.json`** (declared in `login/spec/api.yaml` with
  `security: []` — the full prod URL includes the `/login` mapping key; API Gateway v2 strips it and
  `BaseNamePrefix` re-adds it).
- **SSM public-key floor (permanent, not a rollout crutch):** `login` publishes its public keys to
  **both** the JWKS **and** `/jwtPublicKeys` (SSM) in the same rotation step. Services load SSM first,
  JWKS at startup (startup fetch is **non-fatal**: never abort boot), cache both, and **lazily
  refetch on an unknown `kid`** — **consulting JWKS then SSM (union)** (singleflight + minimum 30s
  interval + negative-cache unknown kids). Fail closed (401) only at **verification** time when no
  key exists. Keep last-known-good per instance. A failed `/jwtPublicKeys` write during rotation
  raises a CloudWatch alarm (login-side) so the floor cannot rot silently.
- **JWKS caching headers:** `Cache-Control: public, max-age=300, stale-while-revalidate=3600`; the
  SSM `(/jwtPublicKeys)` floor — not edge caching — is the load-bearing availability mechanism
  (no edge cache fronts `api.icaa.world` today).

### Spec-declared callee route

The callee's OpenAPI declares the internal route with `security: [{icaaBearerAuth: ["m2m:<scope>"]}]`
(the scheme already exists everywhere). The existing in-process OpenAPI validation middleware enforces
it via the machine-token path. **No API Gateway authorizer is required** (an API Gateway JWT
authorizer on login's JWKS remains a later option). The route **MUST be in the public spec** (the
validation middleware rejects undeclared paths; handlers are only generated; the caller's client is
codegen'd from it).

### Caller flow

A **code-generated client** (in the consuming repo, pinned to the callee git tag) plus a runtime
wrapper that:

- mints/refreshes the cached machine token via `login` (client secret from SSM), refreshing at
  **~80% of TTL**;
- on a callee **401**, treats it as cache-invalidation: discard the cached token, **refetch once,
  retry** (not just network-error retries) — absorbs clock skew and mid-flight rotations;
- validates required fields on the response (generated clients unmarshal silently);
- retries, timeouts, OTel spans.

The callee's CI spec-diffs (`oasdiff`) before deploy.

### m2m endpoint hardening

`POST /login/v1/m2m-tokens` is internet-facing and NOT Google-gated, so it needs real rate-limiting.
**API Gateway HTTP API v2 has no per-route throttling/usage plans** (that is a REST-API v1 feature),
so the control is two layers:

**Layer 1 — edge rate limiting (optional, cost-conscious):** prefer **Cloudflare Rate Limiting** —
`api.icaa.world` DNS is on Cloudflare, and if the record is **proxied** (orange-cloud) a single
rate-limiting rule on `/login/v1/m2m-tokens` is **free** on the Free plan (keyed by IP, blocks before
AWS). AWS WAF is **not free** ($5/mo ACL + $1/mo rule + per-request) — only use it if the record is
DNS-only and the org accepts ~$6/mo; otherwise skip the edge layer. Zone queries confirm whether the
record is proxied. Managed rule groups/bot control are out of scope (extra cost).

**Layer 2 — DynamoDB per-`clientId` fixed-window counter + lockout (in `login`).** Targets the real
threat: a *valid* clientId being hammered, bcrypt CPU burn, and brute-force guesses. New item family
in the existing `login-api` table:

```
PK = "RATE#<clientId>"    SK = "RATE#<clientId>"
attributes: windowCount, failCount, lockedUntil?, ttl   (ttl = end of the 60s window)
```

- **Init-or-claim** the window: `UpdateItem` with `AttributeNotExists` on create + `ttl`; DynamoDB
  TTL cleans stale windows for free.
- **Count atomically:** `ADD windowCount 1` (safe across concurrent Lambda instances);
  `windowCount > limit` (e.g. 30/min) → `429`.
- **Lockout:** on a failed `invalid_client`, `ADD failCount 1`; at N failures (e.g. 5) →
  `SET lockedUntil = now + 15 min`; any request while locked → `429`; success resets `failCount`.
- **Runs BEFORE bcrypt**, so failed attempts never burn Lambda CPU.
- Fixed-window has a soft boundary-burst edge (fine for DoS/brute-force protection); upgrade to a
  single-item token bucket (`SET tokens = LEAST(threshold, tokens + FLOOR((now-updatedAt)/interval))`
  in one `UpdateItem`) if tight-burst ever matters.
- Optional micro-filter: a per-instance in-memory recent-failures map (instances churn, so it is a
  fast-path filter, never the mechanism of record).

- Fix a documented **bcrypt cost** (10–12) so the CPU budget per attempt is known, and **equalize
  the unknown-`clientId` path** with a dummy bcrypt compare so clientId existence is not a timing
  oracle.
- **CloudWatch alarm on `invalid_client` 401 spikes / lockout events** (leak/brute-force signal) and
  on caller-side roster-write M2M failures — wired to the Layer-2 countermeasure as the response, not
  just an alert.
- **Rotation runbook couples both stores:** update the bcrypt `secretRounds[]` + the SSM value
  together, then **roll the caller** (`sam deploy` or recycle) — the caller reads SSM at startup, so
  a rotated secret silently breaks roster writes until its instances recycle. Keep the previous round
  in `secretRounds[]` until all callers have recycled, then remove it. Documented, not implicit.
- **Internal-route residual:** a leaked machine token can read `{name, email}` for any player UUID
  (UUIDs are public in the directory) — rate-limit/anomaly-alarm the internal route, and keep the
  scope string unique per callee.

### Origin bypass (api.icaa.world is proxied via Cloudflare)

Cloudflare's edge rate limiting only covers traffic that actually transits Cloudflare. Two bypass
vectors exist and must be assumed: the **default `execute-api` endpoint**
(`https://<api-id>.execute-api.<region>.amazonaws.com`) is publicly reachable and does not go through
Cloudflare, and **origin-IP discovery** (DNS history, certificate transparency) lets someone connect
to the API Gateway origin directly while presenting `Host: api.icaa.world`.

This does not break the security model — the DDB `RATE#` limiter runs app-level in `login` and caps
floods regardless of path — but the edge controls are defense-in-depth only. Free hardening to close
the bypass:

1. **Host-header validation in the shared middleware** — reject any request whose `Host` isn't
   `api.icaa.world` (localhost in dev). Kills the default-endpoint bypass before any handler;
   fits the existing middleware + `BaseNamePrefix` pattern.
2. **API Gateway custom-domain mutual TLS + Cloudflare Authenticated Origin Pulls** — truststore
   contains Cloudflare's origin-pull CA; the origin refuses any TLS handshake that isn't Cloudflare's
   client cert. Free (not WAF); covers the custom domain; combine with (1) for the default endpoint.
3. `CF-Connecting-IP` is only trusted once the request is known to have come from Cloudflare
   (established by (1)/(2)); only then may the rate limiter key on real client IP as well as `clientId`.
4. AWS WAF remains the paid fallback if any of the above proves unworkable; it is otherwise out of
   scope for the ~$0 monthly budget.

### Local dev

- Local mode: development RSA keypair (private in local login config; public in local JWKS + dev
  verification config). Wrapper machine-auth uses an explicit `LOCAL` env flag (**never** inferred
  from hostname); prod base URLs asserted `https://`; prod code asserts `!isLocal()` before dev keys
  are installable.
- Spec paths include the full prefix: `localhost:<port>/profiles/internal/players/v1/{id}` locally,
  `https://api.icaa.world/profiles/internal/players/v1/{id}` in prod.

## Rejected alternatives

- **HS256 with a shared secret** — forces the signing key on every service. Rejected.
- **SSM-only public distribution** — rotation required touching every service; JWKS + SSM-floor
  is the single source with an availability floor. SSM remains only for login-scoped private keys
  and the public-key floor.
- **IAM authorization (SigV4)** — gateway-config, not spec-declarable; awkward with the org's single
  catch-all Lambda Web Adapter routes. Rejected.
- **Self-signed tokens** — removes the mint call but removes central authority + requires a
  per-client public-key directory at every callee. Rejected.
- **API keys / mTLS / long-lived shared header secret** — respectively static-secret/HTTP-API-v2
  limitations, non-OAS-expressible, and no identity/expiry. Rejected.

## Consequences

- Only `login` mints machine tokens; services verify with public keys and cannot forge them.
- Public keys come from JWKS **with an SSM fallback** and a non-fatal startup fetch — login
  availability degrades verification temporarily, never boots services.
- User tokens migrate to the same model under [ADR-0007](0007-user-jwts-rs256.md); keypairs are
  namespaced `user-*` / `machine-*`.
- `login` owns the token endpoints + private keys; the shared `auth` lib gains a machine-token
  verifier + JWKS cache + the `token_type` separation rules.

## Deploy order / verification (includes IAM + spec steps)

1. `login`: declare `POST /login/v1/m2m-tokens` and `GET /login/.well-known/jwks.json` in its spec
   (`security: []`); provision `/m2m/<clientId>/secret` (SSM, caller role) + bcrypt `CLIENT#<id>`
   record (login table); `/machineJwtSigningKeys` (login-only) + `/jwtPublicKeys` (all services);
   add edge rate limiting (Cloudflare rule if proxied, else WAF only if the ~$6/mo is accepted) +
   the DDB `RATE#` limiter/lockout + alarms (§m2m endpoint hardening).
2. **IAM:** grant caller role `ssm:GetParameter` on `/m2m/<clientId>/secret`; login role on
   `/machineJwtSigningKeys` **and `ssm:PutParameter` on `/machineJwtSigningKeys`,
   `/userJwtSigningKeys`, and `/jwtPublicKeys`** (rotation cannot run without PutParameter).
   Deploy both stacks.
3. Callee: declare the internal route + `m2m:<scope>` in its spec; deploy.
4. Caller: generate client pinned to the callee tag; wire wrapper; deploy.
5. **Smoke after every deploy** (scripted — `make smoke-prod`, owned per repo): machine token → 200;
   no token → 401; user `access` token → 401; wrong-scope machine token → 401; expired/revoked
   `kid` → 401 + JWKS refetch.

---

## Appendix: client-secret verification

`POST /login/v1/m2m-tokens` parses the `clientId:clientSecret` credentials, `GetItem`s
`CLIENT#<clientId>`, and validates via **bcrypt** hash + constant-time compare
(`bcrypt.CompareHashAndPassword`). Unknown/inactive clients short-circuit to `401 invalid_client`
before any crypto work. `secretRounds[]` supports rotation with a grace window; revocation = mark
inactive/delete. Outstanding tokens bounded by the 5-minute TTL; the caller's 80%-TTL refresh and
401-invalidate-retry absorb skew.