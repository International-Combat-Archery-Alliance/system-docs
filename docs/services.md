# Services

All backend services are Go, REST-ish, deployed as AWS Lambda container images via AWS SAM, mounted under `https://api.icaa.world/<prefix>`. They share this shape:

- Hexagonal layout: `cmd/` (entrypoint/wiring), `api/` (handlers + OpenAPI validation), a domain package, `dynamo/` (DynamoDB repo).
- OpenAPI spec in `spec/api.yaml`, generated server via `oapi-codegen`.
- HTTP API route `{proxy+}` on a single Lambda function; prod adds `BaseNamePrefix(logger, "/<prefix>")` so spec paths (`/<prefix>/...`) match after gateway prefixing.
- Env vars come from SSM Parameter Store in prod; locally they come from `SAM local` env / `docker-compose`.
- Middleware chain (common): OpenAPI validation → CORS → Swagger UI → access logging → OTel → trace flush.
- Secrets available in every function: SSM `/jwtSigningKeys` and `/newrelic-license-key`.

## Route index

| Service | Public routes | Admin routes |
|---|---|---|
| `login` | `POST /login/google`, `GET /login/session`, `DELETE /login/session`, `POST /login/refresh` | — |
| `articles-api` | `GET /articles/v1`, `GET /articles/v1/{slug}` | CRUD under `/articles/v1`, `…/publish`, `…/unpublish` |
| `assets-api` | `GET /assets/v1?path=`, `GET /assets/v1/by-path?path=` | `POST /assets/v1/folders`, `POST /assets/v1/upload-url`, `DELETE /assets/v1/by-path`, `POST /assets/v1/by-path/confirm`, `POST /assets/v1/by-path/replace-url` |
| `donation-api` | `POST /donations/v1` | `GET /donations/v1`, `GET /donations/v1/per-state` |
| `event-registration` | `GET /events/v1`, `GET /events/v1/{id}`, `POST /events/v1/{eventId}/register` (deprecated), `POST /events/v1/{eventId}/registrations`, `POST /events/v1/registration/webhook` (Stripe) | `POST /events/v1`, `PATCH /events/v1/{id}`, `GET /events/v1/{eventId}/registrations`, `POST /events/v1/admin/test-email`, `POST /events/v1/admin/test-mailerlite` |
| `mailing-list-api` | `POST /mailing-list/signup` | — |
| `voting-api` | `GET /voting/v1/polls`, `GET /voting/v1/polls/{id}`, `POST /voting/v1/polls/{id}/votes`, `GET /voting/v1/polls/{id}/results` | `POST /voting/v1/polls`, `PATCH /voting/v1/polls/{id}`, `DELETE /voting/v1/polls/{id}` |

---

## login — `/login`

Google OAuth exchange + session management. Issues the auth cookies used by every other service. See [auth.md](auth.md).

- `POST /login/google` — accepts a Google ID token (`{ googleJWT }`), validates it (audience is the frontend's OAuth client), issues `ICAA_ACCESS_TOKEN` + `ICAA_REFRESH_TOKEN` cookies, returns `UserInfo`.
- `GET /login/session` — current user from access cookie/bearer.
- `DELETE /login/session` — logout (revokes refresh token, clears cookies).
- `POST /login/refresh` — rotate refresh cookie into a new access + refresh pair.
- Storage: DynamoDB `login-api` — refresh-token records (`PK/SK = REFRESH_TOKEN#<id>`, TTL 30 d).
- Env/SSM: `/jwtSigningKeys`, `/adminEmails`. Local default: everyone is admin.

## articles-api — `/articles`

Blog/news article CMS.

- `GET /articles/v1` (list published, paginated by cursor+limit), `GET /articles/v1/{slug}` (404 for drafts).
- Admin: `POST /articles/v1` (draft by default, 409 duplicate slug), `PATCH /articles/v1/{slug}` (optimistic lock via `version`), `DELETE`, `POST …/publish`, `POST …/unpublish`.
- Storage: DynamoDB `articles-api`. Single item type `ARTICLE`; `PK/SK = ARTICLE#<slug>`; `GSI1PK = ARTICLE`, `GSI1SK = STATUS#<p>#<status>#UPDATED_AT#…` for status-filtered time-ordered listings.
- Content body is Editor.js blocks serialized to JSON.
- No external integrations.

## assets-api — `/assets`

File/folder management for website content (images, PDFs, logos). All asset URLs on the site come from here.

- Paths are virtual folders (`/images/carousel`, …). Upload non-admin flow:
  1. `POST /assets/v1/upload-url` (admin) → S3 presigned POST + creates a `pending` file record (1 h TTL).
  2. Browser uploads directly to S3.
  3. `POST /assets/v1/by-path/confirm` (admin) → verifies with `HeadObject`, marks `confirmed`.
- Listing: `GET /assets/v1?path=` returns `AdminAsset` for admins, trimmed `PublicAsset` otherwise. Get-one: `GET /assets/v1/by-path?path=`. Delete requires empty folder and removes S3 + DynamoDB item.
- Storage: DynamoDB `assets-api` — mixed `FILE`/`FOLDER` items; `PK = PATH#<parent>`, `SK = NAME#<name>`; folders keep a `ContentCount`; `Version` optimistic locking. S3 `assets.icaa.world` (public-read, flat keys `<uuid><ext>`, ≤ 50 MB), CDN URL from `ASSETS_CDN_BASE_URL`.

## donation-api — `/donations`

One-off donations via Stripe embedded checkout.

- `POST /donations/v1` (public) — validates amount (≥ 100 minor units), creates Checkout session (`UIMode=EmbeddedPage`, `item_type=donation` metadata), returns `{ clientSecret }`; frontend runs embedded checkout and handles success itself (`/donation/success`).
- `GET /donations/v1` (admin) — paginated Stripe-charge listing; `GET /donations/v1/per-state` — aggregate counts by state/currency from billing addresses.
- Storage: none — all data lives in Stripe (searchable via PaymentIntent metadata). No webhooks consumed.
- Env/SSM: `/stripeSecretKey`, `/stripeEndpointSecret`; returns to `STRIPE_RETURN_URL` (default `https://www.icaa.world/donation/success`).

## event-registration — `/events`

The largest service: events, registrations, paid checkout, confirmations, mailing-list groups. This is the only service with write access to the Terraform-managed `event-registration` table.

- Event CRUD: public list (`GET /events/v1`, sorted by start time desc) + get (`GET /events/v1/{id}`); admin create/patch (create also provisions a MailerLite group for the event).
- Registration flows:
  - `POST /events/v1/{eventId}/register` — deprecated free signup.
  - `POST /events/v1/{eventId}/registrations` — paid signup: Turnstile-checked, creates registration + intent + Stripe checkout in one transaction, returns `{ clientSecret, expiresAt, registration }`.
  - `POST /events/v1/registration/webhook` — Stripe webhook (not in the OpenAPI spec): verifies `Stripe-Signature`, on `checkout.session.completed` marks registration paid, sends confirmation email (MailerSend, `info@icaa.world`), adds subscriber to the event's MailerLite group; on `expired` deletes the registration + intent.
- Admin extras: `POST /admin/test-email`, `POST /admin/test-mailerlite`.
- Storage: DynamoDB `event-registration`. Items (`EVENT#<id>`, `REGISTRATION#<email>` under the event, `REG_INTENT#<email>` for pending checkouts, 30-min expiry) — see [data.md](data.md).
- Integrations: Stripe, MailerSend, MailerLite, Cloudflare Turnstile.

## mailing-list-api — `/mailing-list`

Public newsletter signup.

- `POST /mailing-list/signup` — Turnstile-checked; finds-or-creates the "ICAA Mailing List" MailerLite group at startup, adds `{email, name?}`. Maps email-provider errors to 422/429/500.
- Storage: none. Env/SSM: `/mailerLiteApiKey`, `/cfTurnstileSecretKey`. No auth.

## voting-api — `/voting`

Polls (e.g. match MVP votes) with idempotent, captcha-protected ballot casting.

- Polls: public `GET /voting/v1/polls` (+ `/{id}`) with computed status (`Upcoming|Active|Closed`); admin create/patch (`version` query param for optimistic lock)/delete.
- `POST /voting/v1/polls/{id}/votes` — public; requires Turnstile header **and** an `Idempotency-Key` header. Vote record is checked before captcha so retries short-circuit; same key + different ballot → `409 IdempotencyConflict`. Validates against `min/maxSelections`/`maxSelectionsPerGroup` and the poll window.
- `GET /voting/v1/polls/{id}/results` — result visibility gated by `resultsVisibility` (`Live|AfterClose|AdminOnly`) and `publicResultsLevel` (`Full|Percentages|Rankings|None`); admins always see full results.
- Storage: DynamoDB `voting-api`. Polls (`POLL#<id>`, groups/options embedded), atomic `RESULTS` counters (incremented with retries), `IDEMPOTENCY#<key>` records (TTL 24 h).

---

## Shared libraries at a glance

| Lib | What it offers | Used by |
|---|---|---|
| `auth` | `google.Validator` (Google ID token → `AuthToken`); `token.TokenService` (HS256 ICAA JWTs: access 1 h / refresh 30 d, `kid` key rotation); `RefreshTokenStore` interface | all services (auth cookies), `login` (issue) |
| `middleware` | `AccessLogging`, `OTELHandler`/`FlushTraces`, `CorsMiddleware`/`DefaultCorsConfig`, `BaseNamePrefix`, `HostSwaggerUI`, auth/logger/IP context helpers | all services |
| `captcha` | `Validator` interface + `cfturnstile` implementation (siteverify with idempotency key) | event-registration, mailing-list, voting |
| `email` | `Sender` (SES/Gmail/MailerSend) + `SubscriberManager` (MailerLite, `FindOrCreateGroup`) + categorized errors | event-registration, mailing-list |
| `payments` | `CheckoutManager` (`CreateCheckout`, `ConfirmCheckout` webhook) + `PaymentQuerier` (list/search charges) — Stripe impl | donation-api, event-registration |
| `telemetry` | `Init` (OTLP gRPC, New Relic `api-key`), `InstrumentedHTTPClient`, `InstrumentAWSConfig`, `RunWithSpan` | all services |