# ICAA System Docs

System design and architecture documentation for the International Combat Archery Alliance platform — the tech behind [icaa.world](https://icaa.world).

This repo is a reference for the overall platform: services, their routes, data, auth, and infrastructure. Use it as a starting point when building or changing things.

## Repo inventory

The org has 15 repos. All are Go-based unless noted.

### Frontend
| Repo | Role |
|---|---|
| `icaa.world` | The website — a React 19 + Vite SPA deployed to Cloudflare Pages. The primary consumer of every API. |

### Backend services (AWS Lambda + API Gateway, via AWS SAM)
| Repo | Base path on `api.icaa.world` | Role |
|---|---|---|
| `login` | `/login` | Google OAuth, issues JWT cookies, session management |
| `articles-api` | `/articles` | Blog/news article CRUD + publish workflow |
| `assets-api` | `/assets` | File/folder management, presigned S3 uploads |
| `donation-api` | `/donations` | One-off Stripe donations |
| `event-registration` | `/events` | Event CRUD, registrations, paid checkout, confirmation emails |
| `mailing-list-api` | `/mailing-list` | Newsletter signup (MailerLite) |
| `voting-api` | `/voting` | Polls/voting with idempotent ballots |

### Shared Go libraries (consumed as semver-tagged Go modules)
| Repo | Role |
|---|---|
| `auth` | Google ID-token validation + ICAA JWT (access/refresh) signing & validation, roles |
| `middleware` | Reusable HTTP middleware (logging, CORS, OTel, auth-context plumbing, Swagger UI, base path) |
| `captcha` | CAPTCHA validator interface + Cloudflare Turnstile implementation |
| `email` | Email `Sender` / `SubscriberManager` interfaces; SES, Gmail, MailerSend, MailerLite |
| `payments` | Payment `CheckoutManager`/`PaymentQuerier` interfaces; Stripe implementation |
| `telemetry` | OpenTelemetry init (OTLP to New Relic), instrumented HTTP/AWS clients |

### Infrastructure
| Repo | Role |
|---|---|
| `infra` | Terraform for shared AWS resources (DynamoDB `event-registration` table, SES identity, API domain) |

## Run conditions (production)

| Thing | Value |
|---|---|
| Frontend | `https://icaa.world` (Cloudflare Pages, static assets) |
| API gateway | `https://api.icaa.world/<service-prefix>/...` |
| Asset CDN | `https://assets.icaa.world/<uuid>.<ext>` |
| AWS account / region | `197461532156` / `us-east-1` |
| Tracing | OpenTelemetry → New Relic (OTLP) |
| DNS / CDN / Turnstile | Cloudflare |

## Docs

| Doc | Covers |
|---|---|
| [docs/architecture.md](docs/architecture.md) | System overview + diagrams |
| [docs/services.md](docs/services.md) | Each service: routes, storage, integrations |
| [docs/frontend.md](docs/frontend.md) | The `icaa.world` SPA: build, routes, API wiring, env vars |
| [docs/auth.md](docs/auth.md) | Google OAuth, JWT cookies, admin roles, token refresh |
| [docs/data.md](docs/data.md) | DynamoDB single-table designs, S3, Stripe-as-store |
| [docs/infrastructure.md](docs/infrastructure.md) | AWS/Cloudflare deployment, secrets, CI/CD |

## Local development

Everything can run on one machine:

1. `icaa.world` has a `docker-compose.yml` (DynamoDB, Jaeger, MinIO) and `make local` targets that spin up all backend services via SAM on a shared Docker network.
2. Backend services run the Lambda binary locally on ports `3000`–`3006`:
   `events=3000, login=3001, assets=3002, donations=3003, articles=3004, mailing-list=3005, voting=3006`.
3. Frontend dev server runs on `:5173` (`bun run dev`); `.env.development` points each API client at the local SAM ports.
4. Local DynamoDB tables are bootstrapped with the production schemas; local auth uses the `local-development-signing-key…` JWT key and treats every user as admin.
5. TypeScript API clients in the frontend are regenerated from live specs with `bun run codegen` (needs a running/prod `api.icaa.world`).
6. The `infra` repo uses Cloudflare + AWS providers and will need credentials + a state bucket (`icaa-terraform-state`) before `terraform apply`.