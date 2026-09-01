# Services

All backend services are Go, REST-ish, deployed as AWS Lambda container images via AWS SAM, mounted under `https://api.icaa.world/<prefix>`. They share this shape:

- Hexagonal layout: `cmd/` (entrypoint/wiring), `api/` (handlers + OpenAPI validation), a domain package, `dynamo/` (DynamoDB repo).
- Each service declares its routes in an OpenAPI spec (`spec/api.yaml`, generated server via `oapi-codegen`), served live at `<prefix>/openapi.json` + Swagger UI. **This document describes what each service owns, not its routes** — for route-level detail, read the spec.
- Common request behavior — OpenAPI validation, CORS, access logging, OTel tracing, Swagger UI — comes from the shared `middleware` library rather than per-service code.
- Env vars come from SSM Parameter Store in prod; locally they come from `SAM local` env / `docker-compose`.
- Secrets available in every function: SSM `/jwtSigningKeys` and `/newrelic-license-key`.

## At a glance

| Service | Base path | Owns |
|---|---|---|
| `login` | `/login` | Sessions & auth cookies: Google OAuth exchange, token refresh, logout. **Becoming the token authority**: RS256 signing keys (private, login-only), `GET /.well-known/jwks.json` for public key distribution, and `POST /login/v1/m2m-tokens` for scoped machine-to-machine tokens (ADR-0006/0007) |
| `articles-api` | `/articles` | Blog/news article CMS + publish workflow |
| `assets-api` | `/assets` | Virtual folder/file management, presigned S3 uploads, CDN URLs |
| `donation-api` | `/donations` | Creating one-off Stripe checkout sessions (Stripe is the store) |
| `event-registration` | `/events` | Events, registrations, paid checkout, confirmation emails, MailerLite groups |
| `mailing-list-api` | `/mailing-list` | Newsletter signups (MailerLite) |
| `voting-api` | `/voting` | Polls with idempotent, captcha-protected ballots |

---

## login — `/login`

Google OAuth exchange and session management. Issues the auth cookies every other service validates. See [auth.md](auth.md) for the full flow.

- Accepts a Google ID token from the frontend, validates it, and returns `UserInfo` plus `ICAA_ACCESS_TOKEN` / `ICAA_REFRESH_TOKEN` cookies; refresh rotates the pair, logout revokes + clears.
- Storage: DynamoDB `login-api` — refresh-token records (`PK/SK = REFRESH_TOKEN#<id>`, TTL 30 d).
- Env/SSM: `/jwtSigningKeys`, `/adminEmails`. Local default: everyone is admin.
- Routes: `spec/api.yaml` in the `login` repo.

## articles-api — `/articles`

Blog/news article CMS with a draft → published workflow.

- Drafts by default; publishing is explicit; drafts 404 on the public read path; edits are optimistic-locked via `version`; duplicate slugs are rejected (409).
- Storage: DynamoDB `articles-api`. Single item type `ARTICLE`; `PK/SK = ARTICLE#<slug>`; `GSI1PK = ARTICLE`, `GSI1SK = STATUS#<p>#<status>#UPDATED_AT#…` for status-filtered time-ordered listings.
- Content body is Editor.js blocks serialized to JSON.
- No external integrations.
- Routes: `spec/api.yaml` in the `articles-api` repo.

## assets-api — `/assets`

File/folder management for website content (images, PDFs, logos). All asset URLs on the site come from here.

- Paths are virtual folders (`/images/carousel`, …). Assets upload **browser-direct to S3**: an admin call returns a presigned POST and creates a `pending` file record (1 h TTL), the browser uploads straight to S3, then an admin call verifies with `HeadObject` and marks it `confirmed`.
- Listings show full `AdminAsset` data to admins and trimmed `PublicAsset` to everyone else; deleting a folder requires it to be empty and removes the S3 object + DynamoDB item.
- Storage: DynamoDB `assets-api` — mixed `FILE`/`FOLDER` items; `PK = PATH#<parent>`, `SK = NAME#<name>`; folders keep a `ContentCount`; `Version` optimistic locking. S3 `assets.icaa.world` (public-read, flat keys `<uuid><ext>`, ≤ 50 MB), CDN URLs from `ASSETS_CDN_BASE_URL`.
- Routes: `spec/api.yaml` in the `assets-api` repo.

## donation-api — `/donations`

One-off donations via Stripe embedded checkout.

- Validates the amount (≥ 100 minor units), creates a Checkout session (`UIMode=EmbeddedPage`, `item_type=donation` metadata), returns `{ clientSecret }`; the frontend runs the embedded checkout and handles success itself (`/donation/success`).
- Storage: none — all data lives in Stripe (searchable via PaymentIntent metadata). No webhooks consumed (the endpoint secret is read at startup but unused — see [architecture.md](architecture.md)).
- Env/SSM: `/stripeSecretKey`, `/stripeEndpointSecret`; returns to `STRIPE_RETURN_URL` (default `https://www.icaa.world/donation/success`).
- Routes: `spec/api.yaml` in the `donation-api` repo.

## event-registration — `/events`

The largest service: events, registrations, paid checkout, confirmations, mailing-list groups. The only service with write access to the Terraform-managed `event-registration` table.

- Event CRUD (public read, admin create/update; creating an event also provisions its MailerLite group) plus a paid registration flow: Turnstile-checked, creates registration + intent + Stripe checkout in one transaction, returns `{ clientSecret, expiresAt, registration }`. A deprecated free-signup route still exists.
- A Stripe webhook (handled **outside** the OpenAPI spec) verifies `Stripe-Signature`: on `checkout.session.completed` it marks the registration paid, sends the confirmation email (MailerSend, `info@icaa.world`), and adds the subscriber to the event's MailerLite group; on `expired` it deletes the registration + intent. Admin test endpoints exercise email + MailerLite.
- Storage: DynamoDB `event-registration`. Items (`EVENT#<id>`, `REGISTRATION#<email>` under the event, `REG_INTENT#<email>` for pending checkouts, 30-min expiry) — see [data.md](data.md).
- Integrations: Stripe, MailerSend, MailerLite, Cloudflare Turnstile.
- Routes: `spec/api.yaml` in the `event-registration` repo (except the webhook above).

## mailing-list-api — `/mailing-list`

Public newsletter signup.

- Turnstile-checked; finds-or-creates the "ICAA Mailing List" MailerLite group at startup and adds `{email, name?}`. Maps email-provider errors to 422/429/500.
- Storage: none. Env/SSM: `/mailerLiteApiKey`, `/cfTurnstileSecretKey`. No auth.
- Routes: `spec/api.yaml` in the `mailing-list-api` repo.

## voting-api — `/voting`

Polls (e.g. match MVP votes) with idempotent, captcha-protected ballot casting.

- Polls carry a computed status (`Upcoming|Active|Closed`); ballot casting requires a Turnstile token **and** an `Idempotency-Key` header. The vote record is checked before captcha so retries short-circuit; same key + different ballot → `409 IdempotencyConflict`. Selections are validated against `min/maxSelections`/`maxSelectionsPerGroup` and the poll window.
- Results are gated by `resultsVisibility` (`Live|AfterClose|AdminOnly`) and `publicResultsLevel` (`Full|Percentages|Rankings|None`); admins always see full results.
- Storage: DynamoDB `voting-api`. Polls (`POLL#<id>`, groups/options embedded), atomic `RESULTS` counters (incremented with retries), `IDEMPOTENCY#<key>` records (TTL 24 h).
- Routes: `spec/api.yaml` in the `voting-api` repo.

---

## Shared libraries at a glance

| Lib | What it offers | Used by |
|---|---|---|
| `auth` | Google ID-token validation; ICAA JWT (access/refresh) signing & validation with `kid` key rotation; refresh-token store interface | all services (validate), `login` (issue) |
| `middleware` | Reusable HTTP middleware (logging, CORS, OTel, Swagger UI hosting, auth-context plumbing) | all services |
| `captcha` | CAPTCHA validator interface + Cloudflare Turnstile implementation (siteverify with idempotency key) | event-registration, mailing-list, voting |
| `email` | Email sending (SES/Gmail/MailerSend) + MailerLite subscriber/group management, with categorized errors | event-registration, mailing-list |
| `payments` | Stripe checkout-session creation, webhook verification, and charge listing/search | donation-api, event-registration |
| `telemetry` | OpenTelemetry init (OTLP → New Relic) + instrumented HTTP/AWS clients | all services |