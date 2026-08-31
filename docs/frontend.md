# Frontend — `icaa.world`

The website is a **React 19 + TypeScript SPA** built with **Vite** and **TailwindCSS 4 (shadcn/ui)**, deployed as static assets on **Cloudflare Pages** with SPA fallback (client-side routing). No SSR.

## Key stack

- Routing: `react-router-dom` v7 (`<BrowserRouter>`)
- Data fetching: TanStack Query + `openapi-fetch`/`openapi-react-query`, typed clients generated per backend into `src/api/*.d.ts` (`bun run codegen` pulls each service's `/openapi.json` from `api.icaa.world`)
- Forms: React Hook Form + Zod; widgets: `@react-oauth/google`, `react-turnstile`, `@stripe/react-stripe-js`, EditorJS (article content), maplibre/react-map-gl
- Observability: New Relic browser agent + OpenTelemetry web SDK (dev traces to local OTLP endpoint)
- Tooling: `bun` (install/lint/build/dev)

## Routes (in `src/App.tsx`)

| Route | Page |
|---|---|
| `/` | Home |
| `/about-icaa`, `/about-sport`, `/official-rules`, `/our-communities` | Static content |
| `/events`, `/events/:eventId/event-details` | Event listing / details |
| `/events/:eventId/register-free-agent`, `/register-team` | Registration forms (Turnstile-guarded) |
| `/events/:eventId/payment`, `/success` | Stripe embedded checkout / confirmation |
| `/news/:slug` | Articles |
| `/mailing-list` | Newsletter signup (Turnstile-guarded) |
| `/donate`, `/donation/success` | Donation (Stripe embedded checkout) |
| `/admin` | Admin console (`AdminOnlyRoute`) |
| `/vote/:pollId` | Voting |
| `/espn`, `/espn/rules` | ESPN page |
| `/contact` | Posts to Formspree (`https://formspree.io/f/xblkevky`) |

## API wiring

- One context provider per backend in `src/context/` (`*QueryClientContext.tsx`). All clients set `credentials: 'include'` and wrap requests with `createAuthMiddleware()` (`src/lib/authMiddleware.ts`).
- Base URLs come from Vite env vars, all `https://api.icaa.world` in prod; `.env.development` points at local SAM ports (`localhost:3000`–`3006`).
- **Auth middleware**: on any 401 (except `/login/refresh` itself), single-flight `POST /login/refresh` (cookie-based token rotation), then retry the failed request. See [auth.md](auth.md).
- Session state: `userInfo` + `AuthStatus` in localStorage (`src/context/userInfoContext.tsx`); hydrated via `GET /login/session` on boot. Route guards: `ProtectedRoute` and `AdminOnlyRoute` (checks `userInfo.roles` includes `ADMIN`).

## Production env vars

| Var | Purpose |
|---|---|
| `VITE_{EVENT,LOGIN,ASSETS,DONATION,ARTICLES,MAILING_LIST,VOTING}_API_URL` | API base URLs (all `https://api.icaa.world`) |
| `VITE_TURNSTILE_SITE_KEY` | Turnstile widget site key |
| `VITE_STRIPE_PUBLISHABLE_KEY` | Stripe embedded checkout |
| `VITE_NEW_RELIC_{ACCOUNT_ID,BROWSER_KEY,APPLICATION_ID}` | New Relic browser monitoring |
| `VITE_ENABLE_VOTE_TIMELOCK` | Feature flag for voting timelock |
| `VITE_OTEL_EXPORTER_OTLP_ENDPOINT` | Dev-only trace endpoint |

The Google OAuth `clientId` (`1008624351875-q36btbijttq83bogn9f8a4srgji0g3qg.apps.googleusercontent.com`) is hardcoded in `App.tsx` — it must match the audience `login` validates against.

## Payment flows

- **Donation**: `POST /donations/v1` → `clientSecret` → `<StripeEmbeddedCheckout>`; success page is `/donation/success`.
- **Event payment**: `POST /events/v1/{id}/registrations` → `clientSecret` → embedded checkout; success via `/events/:eventId/success`.

## Building / deploying

- `bun run build` → `dist/`; `wrangler.jsonc` serves `./dist` as static assets with single-page-application 404 handling (project `icaa-world`).
- Dev: `bun run full-dev` starts frontend + all backends via docker compose / `make local`; Turnstile uses Cloudflare's always-pass test keys locally.