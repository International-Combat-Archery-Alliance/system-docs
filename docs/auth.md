# Auth

> **Planned change:** ICAA tokens are signed with **HS256 using a shared SSM secret today**, which
> means every service holds the user-token signing key. We are migrating to **RS256 where `login`
> owns the private key and every other service verifies against `login`'s public
> `/.well-known/jwks.json`** — see [ADR-0007](../adr/0007-user-jwts-rs256.md) (and
> [ADR-0006](../adr/0006-internal-http-machine-auth.md) for machine tokens). Until cutover, the
> paragraphs below describe the current symmetric implementation.

Authentication is split between the `login` service (who *issues* sessions) and the shared `auth` library (which every service uses to *validate* them).

## Model

- **Users are Google accounts.** The frontend does Google's implicit flow and obtains an ID token client-side; there is no password/session on ICAA's own pages.
- **Google tokens never leave the browser.** Only `login` sees them, and only to exchange for ICAA tokens.
- **ICAA sessions are two HttpOnly cookies** shared across the `icaa.world` domain:
  - `ICAA_ACCESS_TOKEN` — HS256 JWT, TTL 1 h, `iss=icaa.world`, `aud=icaa-api`, `kid` header, claims `{email, picture, roles[], token_type=access}`.
  - `ICAA_REFRESH_TOKEN` — separate JWT with a random 32-byte `tokenID` in `sub`, TTL 30 d. Rotated on every refresh (old one is deleted from DynamoDB).
- Cookies are `HttpOnly`, `SameSite=Strict`, `Secure` in prod, `Domain=icaa.world` in prod (so `api.icaa.world` APIs and `icaa.world` pages share the session). `Path=/`.
- **Roles**: single role `ADMIN`. Assignment = membership in the configured admin email list (SSM `/adminEmails`, or everyone in local). The `auth/google` validator exposes an `IsAdmin()` based on `hd=icaa.world` (Google Workspace membership), but `login` currently derives roles only from the admin email list — the `hd` path isn't wired into the issued tokens.

## Flow

```mermaid
sequenceDiagram
    participant B as Browser (SPA)
    participant G as Google (OAuth widget)
    participant L as login API
    participant D as DynamoDB (login-api)
    participant A as Any API (e.g. articles-api)

    B->>G: Google sign-in
    G-->>B: Google ID token
    B->>L: POST /login/google { googleJWT }
    L->>G: validate ID token (audience = frontend client id)
    L->>L: derive roles → sign ICAA access + refresh JWTs
    L->>D: save refresh token record (REFRESH_TOKEN#<id>, TTL)
    L-->>B: Set-Cookie ICAA_ACCESS_TOKEN, ICAA_REFRESH_TOKEN<br/>+ UserInfo JSON (cached in localStorage)

    B->>A: GET/POST with ICAA_ACCESS_TOKEN cookie
    A->>A: validate JWT (SSM signing keys, token_type=access; aud/iss pending RS256 — ADR-0007)
    A-->>B: 200 or 401

    opt any 401 (not /login/refresh)
        B->>L: POST /login/refresh (ICAA_REFRESH_TOKEN cookie)
        L->>D: load + delete old token, create new one (rotation)
        L-->>B: new cookie pair
        B->>A: retry original request
    end
```

## How services validate

Every service implements the same OpenAPI validation middleware (pattern introduced by `login`):

1. Read `ICAA_ACCESS_TOKEN` cookie (or `Authorization: Bearer`).
2. Validate the access token — HMAC check, key lookup by `kid` against SSM `/jwtSigningKeys` (JSON `{currentKey, keys:{<kid>: base64}}`), `token_type=access`. (Note: `iss`/`aud` are **not** currently enforced; the RS256 migration adds them — [ADR-0007](../adr/0007-user-jwts-rs256.md).)
3. Wrap the claims as an ICAA auth token and stash it in the request context.
4. Handlers read it back from context; the OpenAPI `admin` scope requires `roles` to contain `ADMIN`.

`login` is the only holder of signing keys *and* the session store. Every other service only validates with the same keys — that's the trust boundary.

## Refresh / logout semantics

- **Refresh** (`POST /login/refresh`): validates the refresh cookie, loads the stored record (which carries `userEmail`, `picture`, `roles`), deletes it (rotation), mints a new pair, sets cookies. The frontend's `createAuthMiddleware()` single-flights refreshes so parallel 401s trigger one rotation, then retries the failed request.
- **Logout** (`DELETE /login/session`): deletes the refresh record (best-effort), clears both cookies with `Max-Age=0`.
- **Session hydration**: on boot the SPA calls `GET /login/session` when it has no cached `userInfo`; cached info is refreshed under the `AUTHENTICATED`/`UNAUTHENTICATED` `AuthStatus` in localStorage.

## Captcha ≠ auth

Login deliberately has **no CAPTCHA** — it's gated by Google OAuth. Turnstile (`cf-turnstile-response` header) protects the public forms instead: mailing-list signup, event registration, and voting ballots (see [services.md](services.md)).