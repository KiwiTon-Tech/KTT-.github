# KTT Platform Sprint Plan

**Prepared:** June 2026  
**Audience:** Ryan  
**Source of truth:** [`ARCHITECTURE.md`](./ARCHITECTURE.md)

---

## Current State Assessment

| Service | Status | Notes |
|---|---|---|
| `KTT-Gateway` | ✅ Scaffolded | Routes, proxy, rate-limit, JWT wiring all done |
| `KTT-Email-Service` | ⚠️ Scaffolded, **has bugs** | Field name mismatches vs Prisma schema (see bugs below) |
| `KTT-Analytics-Service` | ✅ Scaffolded | GA4 + cache + usage routes functional |
| `KTT-Contracts` | ✅ Done | Schemas, errors, JWT helpers published |
| `KTT-DB` | ✅ Schema only | Multi-schema Prisma schema exists, **no migrations run** |
| `KTT-Auth-Service` | ❌ Missing | Gateway proxies to `:5010` but repo doesn't exist yet |
| `KTT-Reports-Service` | ❌ Missing | Gateway proxies to `:5040` but repo doesn't exist yet |
| `KTT-Alerts-Service` | ❌ Missing | Gateway proxies to `:5050` but repo doesn't exist yet |
| `KTT-Dashboard-Web` | ⚠️ Old monolith | Still has its own Prisma schema, not on new gateway API |
| `KTT-Admin-Web` | ❌ Skeleton | Just a README and empty directories |
| `KTT-Frontend` (public) | ✅ Deployed | Marketing site operational |

---

## 🔴 Bugs to Fix Before Any Service Goes to Production

These are **breaking runtime bugs** in `KTT-Email-Service`:

1. **`EmailLog` field name mismatch** — `send.ts` and `contact.ts` write `toEmail` / `fromEmail` but `schema.prisma` defines `to` / `from`
2. **Invalid `EmailStatus`** — code writes `status: 'QUEUED'` but the enum only has `PENDING | SENT | FAILED | BOUNCED`
3. **`worker.ts` field mismatch** — updates `errorMessage` on failure but schema field is `error`
4. **`ContactSubmission` missing fields** — code passes `subject`, `status`, `userAgent` to `prisma.contactSubmission.create()` but none of those fields exist in the schema
5. **GA4 service account key committed** — `KTT-Analytics-Service/ga4-service-account.json` is live in the repo; this is a **Day 1 security issue**, key must be rotated immediately and scrubbed from git history

---

## Sprint 1 — Security & Foundations *(highest priority)*

**Goal:** Unblock safe deployment; establish shared DB and CI infrastructure.

- [ ] Rotate the GA4 service account key — revoke in GCP console, generate new, move to `/home/kiwiton/secrets/ga4-service-account.json`, scrub from git history with `git filter-repo`
- [ ] Fix all 4 `KTT-Email-Service` Prisma/code mismatches (bugs 1–4 above) — align field names, fix enum value, add missing schema fields to `ContactSubmission`
- [ ] Run `prisma migrate dev` in `KTT-DB` to create the first migration baseline and verify all 5 schemas apply cleanly to a local Postgres instance
- [ ] Set up `KTT-Deploy-Bot` GitHub App and cPanel pull scripts per `CPANEL_DEPLOYMENT.md`
- [ ] Wire CI — ensure all service repos have the `.github/workflows/ci.yml` calling the reusable `node-cpanel.yml`

---

## Sprint 2 — KTT-Auth-Service *(blocks everything else)*

**Goal:** Implement the one missing critical service. Gateway can't verify user JWTs without a live JWKS endpoint.

- [ ] Scaffold `KTT-Auth-Service` at port `:5010`
  - `POST /v1/login` — bcrypt verify, issue RS256 access + refresh tokens, create `Session`
  - `POST /v1/register` — hash password, create `User`, send welcome email via `email-svc`
  - `POST /v1/refresh` — validate refresh token, rotate session, issue new access token
  - `POST /v1/logout` — delete session
  - `POST /v1/password-reset` + `POST /v1/password-reset/confirm`
  - `GET /v1/me` + `PATCH /v1/me`
  - `GET /.well-known/jwks.json` — RS256 public key endpoint (gateway caches for 5 min)
  - `GET /internal/verify` — JWT revocation check for the gateway
- [ ] Generate RS256 key pair and document rotation process
- [ ] Migrate old `KTT-Dashboard-Web` users — one-time data migration script to move existing `users` / `sessions` rows into the `auth` schema

---

## Sprint 3 — Email Service Hardening + Testing

**Goal:** Make `KTT-Email-Service` production-safe with regression coverage.

- [ ] Add `EmailClient` model to schema — `contact.ts` calls `verifySiteKey()` which looks up a client by `siteKey`/`apiKey`; this model is referenced in `auth.ts` but not in `schema.prisma`
- [ ] Add `subject`, `status`, `userAgent` fields to `ContactSubmission` — or remove from code (decide and align)
- [ ] Add `QUEUED` value to `EmailStatus` enum — or change code to use `PENDING` with an explicit sub-state column
- [ ] Write integration tests in `test/` for `POST /v1/contact` and `POST /internal/v1/send` (currently test dir is empty across all services)
- [ ] Verify BullMQ retry config matches `WORKER_MAX_ATTEMPTS` from env

---

## Sprint 4 — Analytics Service Refinement

**Goal:** Make `KTT-Analytics-Service` match the architecture spec.

- [ ] Audit and delete leftover dead code in `_archive/`
- [ ] Add `GET /v1/accounts` and `GET /v1/properties?accountId=` endpoints (referenced in architecture, not yet in routes)
- [ ] Add cron warmer (`node-cron`, every 10 min) pre-fetching top dashboards for active users
- [ ] Add quota circuit breaker — on GA4 quota error, serve stale L2 cache instead of failing
- [ ] Expose `GET /metrics` in Prometheus format (all services should expose this per §7.5)
- [ ] Write tests for `metrics.ts` route handlers with a mock GA4 client

---

## Sprint 5 — KTT-Reports-Service

**Goal:** Scaffold the reports service so the dashboard can offer PDF/CSV exports.

- [ ] Scaffold `KTT-Reports-Service` at port `:5040`
  - `POST /v1/reports` — create saved report config
  - `GET /v1/reports` — list user's reports
  - `POST /v1/reports/:id/run` — enqueue BullMQ job
  - `GET /v1/reports/:id/runs` — list run history + file download URL
  - `GET /v1/views` + `POST /v1/views` — saved views CRUD
- [ ] Choose PDF engine — recommend `@react-pdf/renderer` (lighter than Puppeteer on cPanel)
- [ ] BullMQ worker — `reports-queue`, generates PDF/CSV, writes to disk, updates `ReportRun.status`
- [ ] On completion — enqueue `email-svc` `report-ready` template notification

---

## Sprint 6 — KTT-Alerts-Service

**Goal:** Scaffold ops + user threshold alerting.

- [ ] Scaffold `KTT-Alerts-Service` at port `:5050`
  - `POST /v1/rules` + `GET /v1/rules` + `PATCH /v1/rules/:id` + `DELETE /v1/rules/:id`
  - `GET /v1/events` — alert history
  - `POST /v1/channels` + CRUD for delivery channels
- [ ] Cron evaluator (every 5 min) — pull metrics from `analytics-svc`, evaluate `AlertRule.condition` JSON against thresholds, fire `AlertEvent`, dispatch via `email-svc`
- [ ] Port health/error-spike/quota ops alerts from old `alertService.js`

---

## Sprint 7 — KTT-Dashboard-Web Migration

**Goal:** Move the dashboard off its own Prisma schema and onto the new gateway API.

- [ ] Remove `prisma/schema.prisma` from `KTT-Dashboard-Web` — all persistence goes through the gateway now
- [ ] Replace direct DB calls with `fetch` calls via the `@kiwiton/sdk` client
- [ ] Implement NextAuth Credentials provider — POST to `gateway /v1/auth/login`, store RS256 JWT in HTTP-only cookie
- [ ] Wire analytics UI to `gateway /v1/analytics/metrics/report` and `/metrics/realtime`
- [ ] Wire reports UI to `gateway /v1/reports/*`
- [ ] Wire alerts UI to `gateway /v1/alerts/*`

---

## Sprint 8 — KTT-Admin-Web

**Goal:** Build the internal operations dashboard.

- [ ] Bootstrap `KTT-Admin-Web` — Next.js 15, NextAuth (`ADMIN`/`SUPER_ADMIN` role gate), Tailwind v4, shadcn/ui
- [ ] Contact review page — list/search `ContactSubmission` records via gateway
- [ ] Tenant/user management — list users, role assignment, deactivation
- [ ] Analytics property management — assign GA4 properties to users
- [ ] Alert config UI — CRUD for `AlertRule` and `AlertChannel`
- [ ] Email log viewer — list `EmailLog` with status filters

---

## Ongoing / Cross-Cutting

- [ ] **Test coverage** — all `test/` directories are currently empty; each sprint should ship tests for its work
- [ ] **PgBouncer setup** on cPanel host — required before multiple services go live
- [ ] **Redis ACL configuration** — namespace isolation between services
- [ ] **PM2 `ecosystem.config.js`** — add `KTT-Auth-Service`, `KTT-Reports-Service`, `KTT-Alerts-Service` entries as they are created
- [ ] **Prometheus scrape config** — after all services expose `/metrics`
- [ ] **Weekly offsite backup** — Postgres dump to R2/S3 (see `ARCHITECTURE.md` §6.6)

---

**Critical path:** Sprint 1 bugs → Sprint 2 (Auth Service) → Sprint 3 (Email hardening) → all others can proceed in parallel. Nothing reaches production safely without Sprints 1 and 2.
