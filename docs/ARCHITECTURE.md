# KiwiTon Platform Architecture

**Status:** Proposed (v1.0)
**Last updated:** 2026-05-24
**Owners:** KiwiTon Tech

This document is the single source of truth for how the KiwiTon platform is structured: frontends, backend microservices, data, deployment, and the migration path from the current monolith.

---

## 1. Goals & Non-Goals

### Goals

- Cleanly separate the **public marketing site**, the **authenticated user dashboard**, and an **internal admin dashboard**.
- Decompose the current `Analytic-API` monolith into focused **microservices** with single responsibilities.
- Centralize cross-cutting concerns (auth, rate limiting, CORS, observability) in a single **API gateway**.
- Standardize on **PostgreSQL + Prisma** for persistence, **Redis** for cache and queues, and **Node.js + Hono** for services.
- Keep deployment simple: **frontends on Cloudflare Pages**, **backends on WHM/cPanel** (CloudLinux + Node + PM2).
- Eliminate duplication (multiple ORMs, multiple server entrypoints, duplicate user/session schemas).

### Non-Goals

- No MySQL, no SQLite, no Cloudflare D1.
- No Kubernetes / no container orchestration in v1. PM2 on a single host is sufficient.
- No GraphQL federation. The gateway exposes one stitched GraphQL schema; internal calls are REST + zod.
- No multi-region active-active. Single Postgres primary on the WHM box.

---

## 2. High-Level Topology

```
                        ┌──────────────────────┐         ┌─────────────────────┐         ┌────────────────────┐
   public visitors ───► │  web-public          │         │  web-dashboard      │         │  web-admin         │
                        │  (Cloudflare Pages)  │         │  (Cloudflare Pages) │         │  (Cloudflare Pages)│
                        └──────────┬───────────┘         └──────────┬──────────┘         └──────────┬─────────┘
                                   │                                │                               │
                                   │   HTTPS (api.kiwiton-tech.com)                                 │
                                   ▼                                ▼                               ▼
                        ┌─────────────────────────────────────────────────────────────────────────────────┐
                        │                          gateway-svc  (port 5001)                              │
                        │   JWT verify · CORS · rate-limit · request-id · GraphQL stitching · /health     │
                        └─────┬───────────┬────────────┬────────────┬────────────┬────────────────────────┘
                              │           │            │            │            │
                              ▼           ▼            ▼            ▼            ▼
                       ┌───────────┐┌───────────┐┌───────────┐┌───────────┐┌───────────┐
                       │ auth-svc  ││ analytics ││ email-svc ││ reports-  ││ alerts-   │
                       │  :5010    ││  -svc     ││  :5030    ││ svc       ││ svc       │
                       │           ││  :5020    ││           ││  :5040    ││  :5050    │
                       └─────┬─────┘└─────┬─────┘└─────┬─────┘└─────┬─────┘└─────┬─────┘
                             │            │            │            │            │
                             └────────────┴────┬───────┴────────────┴────────────┘
                                               ▼
                                ┌─────────────────────────────────┐
                                │  PgBouncer :6432  →  Postgres   │
                                │  one DB, schemas:               │
                                │  auth · analytics · email ·     │
                                │  reports · alerts               │
                                └─────────────────────────────────┘
                                               │
                                               ▼
                                       Redis :6379
                                  (cache · BullMQ queues)

                                External: GA4 API · SMTP · SMS-via-email gateways
```

**Only `gateway-svc` is exposed publicly.** All other services bind to `127.0.0.1` and are firewalled to localhost. Apache/LiteSpeed in cPanel reverse-proxies `api.kiwiton-tech.com` → `127.0.0.1:5001`.

---

## 3. Repository Layout (Polyrepo)

The platform is a **polyrepo** under the [`KiwiTon-Tech`](https://github.com/KiwiTon-Tech) GitHub org. Each app, service, and shared library is its own repo. There is no monorepo, no pnpm workspace.

### Service Catalog

| Repo | Kind | Deployed to | Port |
|---|---|---|---|
| [`KTT-.github`](https://github.com/KiwiTon-Tech/KTT-.github) | org template | n/a (public) | — |
| [`KTT-Public-Web`](https://github.com/KiwiTon-Tech/KTT-Public-Web) | Next.js app | Cloudflare Pages | — |
| [`KTT-Dashboard-Web`](https://github.com/KiwiTon-Tech/KTT-Dashboard-Web) | Next.js app | Cloudflare Pages | — |
| [`KTT-Admin-Web`](https://github.com/KiwiTon-Tech/KTT-Admin-Web) | Next.js app | Cloudflare Pages | — |
| [`KTT-Gateway`](https://github.com/KiwiTon-Tech/KTT-Gateway) | Node service | WHM/cPanel | 5001 (public via Apache) |
| [`KTT-Auth-Service`](https://github.com/KiwiTon-Tech/KTT-Auth-Service) | Node service | WHM/cPanel | 5010 (internal) |
| [`KTT-Analytics-Service`](https://github.com/KiwiTon-Tech/KTT-Analytics-Service) | Node service | WHM/cPanel | 5020 (internal) |
| [`KTT-Email-Service`](https://github.com/KiwiTon-Tech/KTT-Email-Service) | Node service | WHM/cPanel | 5030 (internal) |
| [`KTT-Reports-Service`](https://github.com/KiwiTon-Tech/KTT-Reports-Service) | Node service | WHM/cPanel | 5040 (internal) |
| [`KTT-Alerts-Service`](https://github.com/KiwiTon-Tech/KTT-Alerts-Service) | Node service | WHM/cPanel | 5050 (internal) |
| [`KTT-DB-Migrations`](https://github.com/KiwiTon-Tech/KTT-DB-Migrations) | shared | run from CI | — |
| [`KTT-Contracts`](https://github.com/KiwiTon-Tech/KTT-Contracts) | shared lib | published to GH Packages | — |

### Per-repo Layout (typical Node service)

```
KTT-<Service>/
├─ src/
│  ├─ index.ts            # Hono app entrypoint
│  ├─ routes/             # HTTP route handlers
│  ├─ lib/                # internal helpers
│  └─ env.ts              # zod-validated env loader
├─ test/
├─ .github/workflows/ci.yml   # consumes KTT-.github/node-cpanel.yml
├─ Dockerfile             # optional (for local dev only)
├─ package.json
├─ tsconfig.json
└─ README.md              # service-specific setup
```

### Cross-Repo Rules

- **Apps** (`KTT-*-Web`) talk only to `KTT-Gateway`. They never call internal services directly.
- **Services** never import each other's source. Cross-service calls go over HTTP using clients generated from `KTT-Contracts`.
- **`KTT-DB-Migrations`** is the only repo that owns the Prisma schema and migration files. CI runs `prisma migrate deploy` from this repo before any service that depends on a new schema version is deployed.
- **`KTT-Contracts`** is published as `@kiwiton-tech/contracts` to GitHub Packages and consumed by every other repo. It contains zod schemas, GraphQL SDL fragments, JWT helpers, error classes, and TypeScript types — nothing runtime-heavy.
- **No circular dependencies.** `KTT-Contracts` depends on nothing in the org. Services depend only on `KTT-Contracts`. Apps depend only on `KTT-Contracts`.
- **Versioning** — `KTT-Contracts` follows semver. Breaking changes require a major bump and a coordinated rollout.

---

## 4. Frontends (Cloudflare Pages)

| App | URL | Purpose | Stack |
|---|---|---|---|
| `web-public` | `kiwiton-tech.com` | Marketing, contact form, public content | Next.js 15, Tailwind v4, daisyUI, OpenNext |
| `web-dashboard` | `dashboard.kiwiton-tech.com` | Authenticated user analytics dashboard | Next.js 15, NextAuth, React Query, Chart.js, Tailwind v4 |
| `web-admin` | `admin.kiwiton-tech.com` | KiwiTon staff: tenant mgmt, contact review, alert config, API keys | Next.js 15, NextAuth (admin role only) |

### Frontend Conventions

- All three deploy to **Cloudflare Pages** via `@cloudflare/next-on-pages` or `@opennextjs/cloudflare`.
- All three import the typed `@kiwiton/sdk` for backend calls — no raw `fetch` to the gateway.
- `web-dashboard` and `web-admin` use **NextAuth Credentials provider** that POSTs to `auth-svc` via the gateway. JWTs returned are stored in HTTP-only cookies.
- Public env (`NEXT_PUBLIC_API_URL=https://api.kiwiton-tech.com`) is the only backend URL the frontends know.

---

## 5. Backend Services

All services share the same baseline stack so they are interchangeable to operate:

- **Runtime:** Node.js 20 LTS, ESM.
- **HTTP framework:** Hono (`@hono/node-server`).
- **Validation:** Zod (schemas live in `KTT-Contracts`).
- **ORM:** Prisma client generated from `KTT-DB-Migrations`.
- **Logging:** Pino → stdout (JSON), PM2 rotates files.
- **Config:** `dotenv`, `.env` loaded from outside the repo (`/home/<cpaneluser>/env/<service>.env`).
- **Health:** every service exposes `GET /health` returning `{status, checks: {...}}`.

### 5.1 `gateway-svc` — Public API Gateway (port 5001)

- **Only** service exposed to the internet (`api.kiwiton-tech.com`).
- Responsibilities:
  - Terminate JWTs (verify against `auth-svc`'s public key cache).
  - CORS for the three frontend origins only.
  - Per-IP and per-user **rate limiting** via Redis (`rate-limiter-flexible`).
  - Request ID injection and structured access logs.
  - **GraphQL endpoint** (`POST /graphql`) — Apollo Server stitched on top of internal services.
  - Thin REST passthroughs where useful (`POST /contact` → `email-svc`).
  - Aggregated `/health` polling each downstream service.
- Mints short-lived **internal JWTs** (HS256, 60s TTL) when calling internal services.
- **Does NOT** talk to Postgres, GA4, or SMTP directly.

### 5.2 `auth-svc` — Authentication & Identity (port 5010, internal)

- Owns Postgres schema **`auth`**: `users`, `sessions`, `oauth_tokens`, `password_resets`.
- Endpoints:
  - `POST /login`, `POST /register`, `POST /refresh`, `POST /logout`
  - `POST /password/reset/request`, `POST /password/reset/confirm`
  - `GET  /oauth/google`, `GET /oauth/google/callback`
  - `GET  /me`
  - `GET  /internal/verify` — gateway uses this to validate JWTs against the revocation list
  - `GET  /internal/users/:id` — service-to-service user lookup
- Migrates the existing `@/Users/zanderbolyanatz/Documents/KTT/Kiwiton User Dashboard/prisma/schema.prisma` `User` / `Session` / `UserPreference` models here.
- The duplicate `user_sessions` table in `@/Users/zanderbolyanatz/Documents/KTT/Analytic-API/db/schema-postgres.sql` is **deleted** as part of migration.

### 5.3 `analytics-svc` — Google Analytics (port 5020, internal)

- Owns Postgres schema **`analytics`**: `analytics_cache`, `api_usage`.
- Wraps the `@google-analytics/data` GA4 client. Service account JSON loaded from `/home/<cpaneluser>/secrets/ga4-service-account.json` (mode 600, **outside** the repo).
- Endpoints (REST, called by gateway resolvers):
  - `GET /accounts`
  - `GET /properties?accountId=...`
  - `GET /views?accountId=...&propertyId=...`
  - `GET /views` (all views, admin)
  - `POST /report` (`{viewId, dateRange, options}`)
  - `GET /realtime/:viewId`
- **Two-tier cache:**
  - L1 — Redis, 30s for realtime, 5–15 min for reports.
  - L2 — Postgres `analytics.analytics_cache` for long-tail and stale-while-revalidate.
- Cache key: `sha256(propertyId | reportType | dateRange | dimensions | metrics)`.
- **Cron warmer** (`node-cron`): every 10 min, pre-fetches the top dashboards for active users.
- **Quota guard:** circuit breaker on GA4 quota errors → serve stale from L2.

### 5.4 `email-svc` — Transactional Email (port 5030, internal)

- Owns Postgres schema **`email`**: `email_logs`, `contact_submissions`.
- Lifts `@/Users/zanderbolyanatz/Documents/KTT/Analytic-API/services/emailService.js` as-is (Nodemailer + SMTP).
- Keeps the **multi-tenant routing** pattern (`CONTACT_ROUTES`, `CONTACT_FROM_NAMES` env vars) — that pattern is correct and stays.
- Endpoints:
  - `POST /internal/send` — `{template, to, vars, siteKey}`
  - `POST /internal/contact` — contact form submission
  - `GET  /internal/logs` — recent send history (admin only)
- **Queue:** BullMQ `email-queue` on Redis. Gateway enqueues; this service consumes. Failures retry with exponential backoff and persist to `email_logs`.
- Templates live in code (`services/email/templates/*.tsx` rendered with `@react-email/render`), not in callers.

### 5.5 `reports-svc` — Saved Reports & Exports (port 5040, internal)

- Owns Postgres schema **`reports`**: `reports`, `saved_views`, `report_runs`.
- Migrates `Report` and `SavedView` models from the dashboard's current Prisma schema.
- Generates **PDF** (Puppeteer or `@react-pdf/renderer`) and **CSV/XLSX** exports.
- Exports written to `/home/<cpaneluser>/reports/<userId>/<reportId>.pdf`; access via signed URLs minted by the gateway.
- **Queue:** BullMQ `reports-queue`. Scheduled runs via `node-cron` enqueue jobs.
- On `report.ready` → enqueues an `email-svc` notification.
- Pulls metrics from `analytics-svc`. **Never** talks to GA4 directly.

### 5.6 `alerts-svc` — Threshold & Ops Alerts (port 5050, internal)

- Owns Postgres schema **`alerts`**: `alert_rules`, `alert_events`, `alert_channels`.
- Two classes of alerts:
  1. **Ops alerts** — service health, error spikes, GA4 quota — sent to KiwiTon team via Gmail SMTP and SMS-via-email gateways. Lifts `@/Users/zanderbolyanatz/Documents/KTT/Analytic-API/services/alertService.js`.
  2. **User alerts** — customer-defined thresholds (e.g. "tell me when bounce rate > 60%").
- Cron every 5 min: pulls relevant metrics from `analytics-svc`, evaluates rules, fires events.
- Delivery channels: email (via `email-svc`), SMS-to-email gateways, future webhooks.
- Kept **separate from `email-svc`** because credentials, recipients, and threat model are operator-level rather than customer-level.

---

## 6. Data Layer (PostgreSQL only)

### 6.1 Single Database, Schema per Service

- **One Postgres instance** on the WHM/cPanel host.
- **One database**: `kiwiton_prod` (and `kiwiton_staging`).
- **Schema per service**:

| Schema | Owner service | Tables |
|---|---|---|
| `auth` | `auth-svc` | `users`, `sessions`, `oauth_tokens`, `password_resets` |
| `analytics` | `analytics-svc` | `analytics_cache`, `api_usage` |
| `email` | `email-svc` | `email_logs`, `contact_submissions` |
| `reports` | `reports-svc` | `reports`, `saved_views`, `report_runs` |
| `alerts` | `alerts-svc` | `alert_rules`, `alert_events`, `alert_channels` |

### 6.2 Per-Service Postgres Roles

Each service connects with its own role that has privileges only on its own schema:

```sql
CREATE ROLE auth_svc      LOGIN PASSWORD '...';
GRANT USAGE   ON SCHEMA auth      TO auth_svc;
GRANT ALL     ON ALL TABLES IN SCHEMA auth TO auth_svc;
-- repeat per service/schema
```

This prevents accidental cross-service writes even though they share an instance.

### 6.3 Prisma Multi-Schema

Single `KTT-DB-Migrations/schema.prisma`:

```prisma
generator client {
  provider        = "prisma-client-js"
  previewFeatures = ["multiSchema"]
}

datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")
  schemas  = ["auth", "analytics", "email", "reports", "alerts"]
}

model User {
  id        String   @id @default(cuid())
  email     String   @unique
  // ...
  @@map("users")
  @@schema("auth")
}

model AnalyticsCache {
  id        String   @id @default(cuid())
  key       String   @unique
  data      Json
  expiresAt DateTime
  @@map("analytics_cache")
  @@schema("analytics")
}
// ...
```

### 6.4 Migrations

- `prisma migrate deploy` runs from `KTT-DB-Migrations` via a manual GitHub Actions workflow only.
- **No service runs migrations at boot.**
- Migration history committed to `KTT-DB-Migrations/migrations/`.
- The hand-written SQL in `KTT-Analytics-Service/db/schema-postgres.sql` and `KTT-Analytics-Service/db/schema.sql` is **deleted** once Prisma is the source of truth.

### 6.5 Connection Pooling

- **PgBouncer** on `127.0.0.1:6432` in transaction mode.
- All services connect through PgBouncer, not directly to Postgres.
- Keeps total Postgres connections sane across 6 PM2 processes.

### 6.6 Backups

- cPanel nightly Postgres dump (built-in).
- Weekly `pg_dump --format=custom` shipped to off-box storage (Cloudflare R2 or S3).
- Restore drill quarterly.

### 6.7 Caching & Queues (Redis)

- **One Redis instance** on the WHM box, `127.0.0.1:6379`.
- Logical separation by key prefix and BullMQ queue name:
  - `cache:analytics:*` — GA4 response cache.
  - `rl:*` — rate limiter buckets.
  - `bull:email-queue:*` — email job queue.
  - `bull:reports-queue:*` — report generation queue.
  - `bull:alerts-queue:*` — alert evaluation queue.

---

## 7. Cross-Cutting Concerns

### 7.1 Authentication

- **End-user JWTs** issued by `auth-svc`, RS256, 15-min access + 30-day refresh, stored in HTTP-only secure cookies on the dashboard.
- **Service-to-service JWTs** signed by the gateway, HS256, 60-second TTL, claims `{svc, aud, exp}`. Verified by every internal service. Shared `INTERNAL_JWT_SECRET` rotated quarterly.
- **No service trusts the network.** Even on localhost, every internal request must carry an internal JWT.

### 7.2 Authorization

- Roles: `USER`, `ADMIN`, `SUPER_ADMIN` (already defined in `@/Users/zanderbolyanatz/Documents/KTT/Kiwiton User Dashboard/prisma/schema.prisma`).
- `web-admin` requires `ADMIN` or `SUPER_ADMIN`.
- Per-route role checks live in each service, not just the gateway.

### 7.3 CORS

- Allowed origins (gateway only):
  - `https://kiwiton-tech.com`
  - `https://dashboard.kiwiton-tech.com`
  - `https://admin.kiwiton-tech.com`
- Internal services do not set permissive CORS — they reject browser origins entirely.

### 7.4 Rate Limiting

- Centralized at the gateway, backed by Redis.
- Default: 100 req/min per IP for unauth, 600 req/min per user for auth.
- Stricter buckets on `/login`, `/register`, `/password/reset` (5 req/min per IP).

### 7.5 Observability

- **Logs:** structured JSON via Pino. PM2 log rotation. Optional shipping to a log aggregator (Better Stack, Loki) later.
- **Tracing:** request-id (`x-request-id`) injected at the gateway, propagated through every internal hop.
- **Metrics:** each service exposes `/metrics` (Prometheus format) — scraped by a single Prometheus on the box (later phase).
- **Health:** gateway aggregates `/health` from each service into a single `/health` response.

### 7.6 Secrets Management

- Secrets live in `/home/<cpaneluser>/env/<service>.env`, mode 600.
- **Never committed.** `.gitignore` enforces this.
- Service account JSON for GA4: `/home/<cpaneluser>/secrets/ga4-service-account.json`, mode 600.
- The currently committed `KTT-Analytics-Service/ga4-service-account.json` must be **rotated and scrubbed from git history** as a Day 1 task.

### 7.7 Error Handling

- Shared error classes in `KTT-Contracts/src/errors.ts`: `BadRequestError`, `UnauthorizedError`, `ForbiddenError`, `NotFoundError`, `ConflictError`, `RateLimitError`, `UpstreamError`, `InternalError`.
- Every service maps these to consistent HTTP status codes via a Hono middleware.
- Gateway translates internal errors to GraphQL errors with stable codes.

---

## 8. Deployment

### 8.1 Frontends — Cloudflare Pages

- One Cloudflare Pages project per app (`KTT-Public-Web`, `KTT-Dashboard-Web`, `KTT-Admin-Web`).
- Pages connects directly to each repo and builds on push to `main`.
- Each Pages project has its own `NEXT_PUBLIC_*` env vars and custom domain.
- The only backend URL the frontends know is `NEXT_PUBLIC_API_URL=https://api.kiwiton-tech.com`.

### 8.2 Backends — WHM/cPanel pull via `KTT-Deploy-Bot`

Deployments are **pulled by cPanel**, not pushed by GitHub Actions. The shared cPanel host blocks inbound SSH, so the standard `rsync`-from-CI pattern doesn't work. Instead:

1. A GitHub App (`KTT-Deploy-Bot`) is installed on the org with `contents:read` on every `KTT-*` repo.
2. A helper script on the cPanel box mints a short-lived installation token from the App's private key.
3. `git pull` runs on the cPanel box using that token over HTTPS.
4. PM2 reloads the affected service.

Full setup is in [`docs/CPANEL_DEPLOYMENT.md`](./CPANEL_DEPLOYMENT.md).

Layout on the cPanel host:

```
/home/kiwiton/
├─ apps/                          # one directory per service repo
│  ├─ KTT-Gateway/
│  ├─ KTT-Auth-Service/
│  ├─ KTT-Analytics-Service/
│  ├─ KTT-Email-Service/
│  ├─ KTT-Reports-Service/
│  ├─ KTT-Alerts-Service/
│  └─ ecosystem.config.js         # PM2 process map (lives outside the repos)
├─ env/                           # mode 600, never in git
│  ├─ gateway.env
│  ├─ auth.env
│  └─ ...
├─ secrets/
│  └─ ga4-service-account.json   # mode 600
└─ bin/
   ├─ ktt-token.sh                # mints an installation token
   └─ ktt-deploy.sh                # pulls one repo + reloads its PM2 process
```

- Apache/LiteSpeed reverse-proxies `api.kiwiton-tech.com` → `127.0.0.1:5001` only.
- All other ports firewalled to localhost (CSF / iptables).
- TLS handled by AutoSSL at the cPanel level.

### 8.3 CI — Reusable Workflows

GitHub Actions runs **lint + test + build only**. Deploys are not in CI.

Every service repo includes a single `.github/workflows/ci.yml` that consumes the shared workflow from `KTT-.github`:

```yaml
name: CI
on: [push, pull_request]
jobs:
  ci:
    uses: KiwiTon-Tech/KTT-.github/.github/workflows/node-cpanel.yml@main
    with:
      app_path: /home/kiwiton/apps/KTT-Auth-Service
      node_version: "20"
      run_tests: true
    secrets: inherit
```

- The reusable workflow lives in [`KTT-.github/.github/workflows/node-cpanel.yml`](../.github/workflows/node-cpanel.yml).
- `KTT-.github` must be **public** so private service repos can consume it on the Free plan.
- CI-time secrets are set per repo with `gh secret set`. Runtime secrets live only in `/home/kiwiton/env/*.env` on the cPanel box, never in GitHub.

### 8.4 Database Migrations

- `KTT-DB-Migrations` is the only repo that owns the Prisma schema.
- A separate manual workflow (`gh workflow run`) on that repo runs `prisma migrate deploy` against staging or production.
- Service deploys assume the schema is already in the right state. **Never** auto-migrate from a service's startup.

### 8.5 Release Order for a Schema-Affecting Change

1. PR `KTT-Contracts` (if API surface changes), version-bump, publish.
2. PR `KTT-DB-Migrations` with the new migration. Merge and run the migration workflow against staging, verify, then production.
3. PR each affected service to consume the new contracts version. Merge and pull each onto cPanel (lowest-level service first: `KTT-Auth-Service`, then `KTT-Gateway` last).
4. PR each affected app. Merge — Cloudflare Pages deploys automatically.

---

## 9. Migration Plan (from current state)

Ordered lowest-risk → highest-risk. Each step is independently shippable. Folder names below refer to the local working directory layout under `/Users/zanderbolyanatz/Documents/KTT/`, where each folder is a separate git repo destined for the `KiwiTon-Tech` org.

| # | Step | Why | Risk |
|---|---|---|---|
| 1 | **Rotate the leaked GA4 service account key** in `KTT-Analytics-Service/ga4-service-account.json`, scrub from git history, move outside the repo | Security | Low |
| 2 | **Pick one server entrypoint** in `KTT-Analytics-Service/api/`. Keep `server-hono.js`, delete `server.js` and `server-cloudflare.js` | Reduce duplication | Low |
| 3 | **Push `KTT-.github`, `KTT-Public-Web`, `KTT-Dashboard-Web`, `KTT-Admin-Web` to GitHub** under `KiwiTon-Tech` org with the polyrepo names already on disk | Foundation | Low |
| 4 | **Set up `KTT-Deploy-Bot` GitHub App + cPanel host** per `CPANEL_DEPLOYMENT.md` | Foundation | Low |
| 5 | **Create `KTT-DB-Migrations`** with unified Prisma multi-schema. Migrate the dashboard's existing schema in. Delete `KTT-Analytics-Service/db/*.sql` | Single source of truth | Medium |
| 6 | **De-duplicate `users` / `user_sessions`** between the two old schemas. One-time data migration script | Correctness | Medium |
| 7 | **Create `KTT-Contracts`** with zod schemas, GraphQL SDL, error classes, JWT helpers. Publish to GitHub Packages | Shared types | Low |
| 8 | **Drop Sequelize** in `KTT-Analytics-Service`. Switch to the Prisma client from `KTT-DB-Migrations`. Remove `sequelize`, `pg-hstore` from `package.json` | Reduce duplication | Medium |
| 9 | **Extract `KTT-Email-Service`** as its own repo. BullMQ queue. Gateway calls it via HTTP | Smallest, isolated | Low |
| 10 | **Extract `KTT-Auth-Service`** as its own repo. Dashboard's NextAuth points at it | Biggest correctness risk | High |
| 11 | **Trim `KTT-Analytics-Service`** to GA4 + caching only. Add Redis L1 cache | Performance win | Medium |
| 12 | **Extract `KTT-Gateway`** as its own repo (the thin remainder of the old monolith) — pure routing/auth/rate-limit/GraphQL stitching | Clean separation | Medium |
| 13 | **Create `KTT-Reports-Service`** and **`KTT-Alerts-Service`** | Feature work | Medium |
| 14 | **Build out `KTT-Admin-Web`** on top of the now-stable internal APIs | New surface | Low |

---

## 10. Open Questions

1. Which **PDF engine** for `reports-svc` — Puppeteer (heavier, pixel-perfect) or `@react-pdf/renderer` (lighter, declarative)?
2. Do we need a **billing service** in v1, or is that v2?
3. Webhook delivery for user alerts — v1 or v2?
4. Where do customer-uploaded assets live? R2 or local disk on the WHM box?

---

## 11. Glossary

- **Gateway** — the single public-facing service; everything ingresses through it.
- **Internal JWT** — short-lived service-to-service token, distinct from end-user JWTs.
- **Schema (Postgres)** — namespace within a database; each service owns one.
- **PgBouncer** — connection pooler in front of Postgres.
- **BullMQ** — Redis-backed job queue used for async work.

---

*This document is intentionally opinionated. Deviating from these patterns requires updating this file in the same PR that introduces the deviation.*
