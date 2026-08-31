# Data

Most state lives in DynamoDB, one table per service, all in `us-east-1`. A couple of services keep their data in external systems (Stripe for donations, S3 for files). There is no SQL database and no shared microservice-to-microservice data store.

## DynamoDB conventions across services

- **Single-table design** with composite keys:
  - `PK` / `SK` (both strings) for the primary access pattern.
  - `GSI1PK` / `GSI1SK` (projection ALL) for time-ordered listings (nearly everything sorts newest-first).
- **Item discrimination** by attributes (e.g. `Type`, `status`, `Status`) rather than separate tables.
- **Optimistic locking**: metadata items carry a `Version`; conditional writes / `TransactWriteItems` enforce it (`409 VersionConflict` on mismatch).
- **TTL** for short-lived data (pending uploads, refresh tokens, idempotency records) so DynamoDB does the cleanup.
- **Transactions** (`TransactWriteItems`) are used where a request must atomically write several related items or keep folder/event stat counters consistent.
- Pagination is cursor-based (`ExclusiveStartKey` → base64 cursor in responses).

## Tables

### `articles-api` — created by SAM (articles-api)
| Item type | Keys | Notes |
|---|---|---|
| `ARTICLE` (draft/published) | `PK=ARTICLE#<slug>`, `SK=ARTICLE#<slug>` | `GSI1PK=ARTICLE`, `GSI1SK=STATUS#<p>#<status>#UPDATED_AT#…` for status-filtered, time-ordered listing. `Version` lock. `Content` = Editor.js blocks as JSON string. |

### `assets-api` — created by SAM (assets-api), TTL enabled
| Item type | Keys | Notes |
|---|---|---|
| `FILE` / `FOLDER` | `PK=PATH#<parent>`, `SK=NAME#<name>` | Root folder is a known `PATH#/`/`NAME#/` UUID. Folders hold `ContentCount`, bumped in the same transaction as child create/delete. `Version` lock. Pending uploads get a 1 h TTL until confirmed. |

### `event-registration` — created by Terraform (`infra`), PITR + deletion protection
| Item type | Keys | Notes |
|---|---|---|
| `EVENT` | `PK=EVENT#<id>`, `SK=EVENT#<id>` | `GSI1PK=EVENT`, `GSI1SK=EVENT#<StartTime>#<id>` (list by start time). Stores prices, team size range, TZ, rules/image refs, sign-up stats, `MailingListGroupID`. |
| `REGISTRATION` | `PK=EVENT#<eventId>`, `SK=REGISTRATION#<email>` | `Type` `BY_INDIVIDUAL`/`BY_TEAM` (team keyed by captain email), `Paid` flag, `Version`. |
| `REGISTRATION_INTENT` | `PK=EVENT#<eventId>`, `SK=REG_INTENT#<email>` | Links pending Stripe session; 30-min expiry. Registration + intent + event-stats update are written in **one transaction**; paying or expiry deletes the intent. |

### `login-api` — created by SAM (login), TTL enabled
| Item type | Keys | Notes |
|---|---|---|
| `REFRESH_TOKEN` | `PK=REFRESH_TOKEN#<id>`, `SK=REFRESH_TOKEN#<id>` | Stores `userEmail`, `picture`, `roles`, `expiresAt` (30 d), `ttl`. Deleted on rotation/logout. |

### `voting-api` — created by SAM (voting-api), TTL enabled
| Item type | Keys | Notes |
|---|---|---|
| `POLL` | `PK=POLL#<id>`, `SK=#METADATA` | `GSI1PK=POLL`, `GSI1SK=POLL#<StartTime>#<id>`. Groups/options embedded; `Version`. |
| `RESULTS` | `PK=POLL#<id>`, `SK=RESULTS` | `{TotalVotes, Counts{optionId: int}}` — created atomically with the poll; incremented per vote with retries. |
| `VOTE_RECORD` | `PK=POLL#<id>`, `SK=IDEMPOTENCY#<key>` | Ballot replay guard, TTL 24 h. Checked **before** captcha so retries are cheap. |

## S3 — `assets.icaa.world`

- Public-read bucket with CORS for `https://icaa.world` and `https://*.icaa.world`.
- Object keys are flat: `<uuid>.<ext>` (no folders); the folder hierarchy lives only in DynamoDB.
- Writes are browser-direct via **presigned POST** (from `POST /assets/v1/upload-url`); the API confirms via `HeadObject`.
- CDN base URL for the API is `ASSETS_CDN_BASE_URL` (server-side env var in `assets-api`, default `https://assets.icaa.world`). The API returns full CDN URLs on asset objects, and all `<img>`/PDF URLs on the site point there.

## Stripe as a store

- **Donations** have no local record — money, metadata, and aggregate queries all live in Stripe (PaymentIntent metadata `item_type=donation` makes them searchable for the admin dashboard).
- **Registrations** are multi-writer: the source of truth for "paid" is the DynamoDB `REGISTRATION` row, updated from the verified webhook — but the payment itself is a Stripe Checkout session (`PaymentSessionId` stored on the intent).

## SSM Parameter Store (secrets/config, per service)

| Parameter | Used by |
|---|---|
| `/jwtSigningKeys` | all services (JWT validation), login (signing) |
| `/adminEmails` | login (role assignment) |
| `/newrelic-license-key` | all services (OTLP export) |
| `/stripeSecretKey`, `/stripeEndpointSecret` | donation-api, event-registration |
| `/cfTurnstileSecretKey` | event-registration, mailing-list-api, voting-api |
| `/mailerLiteApiKey` | mailing-list-api, event-registration |
| `/mailerSendApiKey` | event-registration |