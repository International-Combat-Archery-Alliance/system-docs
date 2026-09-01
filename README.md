# ICAA System Docs

System design and architecture documentation for the International Combat Archery Alliance platform — the tech behind [icaa.world](https://icaa.world).

This repo is a reference for the overall platform: services and what each owns, data, auth, and infrastructure. Use it as a starting point when building or changing things. Route-level details live in each service's OpenAPI spec, not here.

## Repo inventory

The org has 15 repos. All are Go-based unless noted.

### Frontend
| Repo | Role |
|---|---|
| `icaa.world` | The website — a React + Vite SPA deployed to Cloudflare Pages. The primary consumer of every API. |

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
| `email` | Email sending + subscriber/group management (SES, Gmail, MailerSend, MailerLite) |
| `payments` | Stripe checkout + payment lookup interfaces |
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
| Tracing | OpenTelemetry → New Relic (OTLP, account `7969260`) |
| DNS / CDN / Turnstile | Cloudflare |

## Docs

| Doc | Covers |
|---|---|
| [docs/architecture.md](docs/architecture.md) | System overview + diagrams |
| [docs/services.md](docs/services.md) | Each service: what it owns, storage, integrations (routes live in each spec) |
| [docs/frontend.md](docs/frontend.md) | The `icaa.world` SPA: build, pages, API wiring, env vars |
| [docs/auth.md](docs/auth.md) | Google OAuth, JWT cookies, admin roles, token refresh |
| [docs/data.md](docs/data.md) | DynamoDB single-table designs, S3, Stripe-as-store |
| [docs/infrastructure.md](docs/infrastructure.md) | AWS/Cloudflare deployment, secrets, CI/CD |
| [prd/](prd/README.md) | **Product requirements** — [PRD-0001](prd/0001-competition-platform.md): scope, personas, roadmap, decisions (the north-star product spec) |
| [rfcs/](rfcs/README.md) | Proposed per-feature designs (teams, games/standings, playoffs, circuits, player profiles) |
| [adr/](adr/README.md) | Architecture decision records (the *why* behind cross-cutting decisions) |

## Local development

Everything can run on one machine:

1. `icaa.world` provides a `docker-compose.yml` (DynamoDB, Jaeger, MinIO on the `icaa-shared` network) and `bun run-all-backend`, which drives each backend repo's `make local` target to bring up all services via SAM.
2. Backend services run on `localhost:3000`–`3006`; the SPA dev server runs on `:5173` (`bun run dev`). The port → service mapping lives in `.env.development`.
3. Local DynamoDB tables are bootstrapped with the production schemas; local auth uses the `local-development-signing-key…` JWT key and treats every user as admin.
4. TypeScript API clients in the frontend are regenerated from live specs with `bun run codegen` (needs a running/prod `api.icaa.world`).
5. The `infra` repo needs AWS + Cloudflare credentials and the shared Terraform state bucket before `terraform apply` (see [docs/infrastructure.md](docs/infrastructure.md)).