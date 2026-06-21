# KTT-.github

**Organization-wide templates, reusable workflows, and platform documentation for the KiwiTon-Tech polyrepo.**

This repository serves as the central hub for:
- **Reusable GitHub Actions workflows** consumed by all KTT-* services and apps
- **Platform architecture documentation** (service catalog, topology, data model, deployment)
- **Deployment playbooks** for cPanel/WHM hosting
- **GitHub organization profile** (displayed on the org homepage)

---

## 📁 Repository Structure

```
KTT-.github/
├── .github/
│   └── workflows/
│       └── node-cpanel.yml       # Reusable CI workflow (lint + test + build)
├── docs/
│   ├── ARCHITECTURE.md            # Platform architecture (single source of truth)
│   └── CPANEL_DEPLOYMENT.md       # cPanel deployment setup playbook
├── profile/
│   └── README.md                  # GitHub org profile page
└── README.md                      # This file
```

---

## 🏗️ Platform Architecture

The KiwiTon platform is a **polyrepo** of microservices, frontends, and shared libraries:

### Frontends (Cloudflare Pages)
- **[KTT-Public-Web](https://github.com/KiwiTon-Tech/KTT-Public-Web)** — Marketing site (`kiwiton-tech.com`)
- **[KTT-Dashboard-Web](https://github.com/KiwiTon-Tech/KTT-Dashboard-Web)** — User analytics dashboard (`dashboard.kiwiton-tech.com`)
- **[KTT-Admin-Web](https://github.com/KiwiTon-Tech/KTT-Admin-Web)** — Internal admin panel (`admin.kiwiton-tech.com`)

### Backend Services (Node.js on WHM/cPanel)
- **[KTT-Gateway](https://github.com/KiwiTon-Tech/KTT-Gateway)** — Public API gateway (port 5001, exposed via `api.kiwiton-tech.com`)
- **[KTT-Auth-Service](https://github.com/KiwiTon-Tech/KTT-Auth-Service)** — Authentication & sessions (port 5010, internal)
- **[KTT-Analytics-Service](https://github.com/KiwiTon-Tech/KTT-Analytics-Service)** — Google Analytics 4 ingestion (port 5020, internal)
- **[KTT-Email-Service](https://github.com/KiwiTon-Tech/KTT-Email-Service)** — Transactional email + contact forms (port 5030, internal)
- **[KTT-Reports-Service](https://github.com/KiwiTon-Tech/KTT-Reports-Service)** — Saved reports & exports (port 5040, internal)
- **[KTT-Alerts-Service](https://github.com/KiwiTon-Tech/KTT-Alerts-Service)** — Threshold & ops alerts (port 5050, internal)

### Shared Libraries
- **[KTT-DB-Migrations](https://github.com/KiwiTon-Tech/KTT-DB-Migrations)** — Single Prisma schema, migrations, seed scripts
- **[KTT-Contracts](https://github.com/KiwiTon-Tech/KTT-Contracts)** — Shared Zod schemas, error classes, JWT helpers

**Only `KTT-Gateway` is exposed publicly.** All other services bind to `127.0.0.1` and are firewalled to localhost.

For the full architecture, see **[docs/ARCHITECTURE.md](./docs/ARCHITECTURE.md)**.

---

## 🚀 Deployment Model

Deployments are **pulled by cPanel**, not pushed by GitHub Actions.

The shared cPanel host blocks inbound SSH, so we use the inverse pattern:
1. A GitHub App (`KTT-Deploy-Bot`) is installed on the org with `contents:read` on every repo
2. A helper script on the cPanel box mints a short-lived installation token
3. `git pull` runs on the cPanel box using that token over HTTPS
4. PM2 reloads the affected service

**Full setup:** [docs/CPANEL_DEPLOYMENT.md](./docs/CPANEL_DEPLOYMENT.md)

**cPanel layout:**
```
/home/kiwiton/
├── apps/                          # one directory per service repo
│   ├── KTT-Gateway/
│   ├── KTT-Auth-Service/
│   ├── KTT-Analytics-Service/
│   ├── KTT-Email-Service/
│   ├── KTT-Reports-Service/
│   ├── KTT-Alerts-Service/
│   └── ecosystem.config.js        # PM2 process map
├── env/                           # mode 600, never in git
│   ├── gateway.env
│   ├── auth.env
│   └── ...
├── secrets/
│   └── ga4-service-account.json  # mode 600
└── bin/
    ├── ktt-token.sh               # mints installation token
    └── ktt-deploy.sh              # pulls one repo + reloads PM2
```

---

## ⚙️ Reusable CI Workflows

GitHub Actions runs **lint + test + build** on every push and pull request. Deploys are handled by the cPanel pull mechanism described above.

### Available Workflows

#### `node-cpanel.yml`
Lint + test + build for Node.js services and Next.js apps.

**Usage in other repos:**
```yaml
# .github/workflows/ci.yml
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

**Inputs:**
- `app_path` — Absolute path on cPanel (informational, used in summary)
- `node_version` — Node.js version (default: `"20"`)
- `package_manager` — `npm` or `pnpm` (default: `"npm"`)
- `run_tests` — Whether to run tests (default: `true`)
- `run_build` — Whether to run build (default: `true`)
- `working_directory` — Working directory inside repo (default: `"."`)

**Why this repo must be public:**
GitHub Free plan doesn't allow private repos to consume reusable workflows from other private repos. `KTT-.github` is public so all private service repos can use it without requiring GitHub Team.

---

## 🔐 Secrets Management

### CI-time Secrets (GitHub)
- Stored at the **repo level**, not org level (GitHub Free limitation)
- Use `gh secret set` to seed all repos in one shot (see [docs/CPANEL_DEPLOYMENT.md](./docs/CPANEL_DEPLOYMENT.md))

### Runtime Secrets (cPanel)
- Live in `/home/kiwiton/env/<service>.env` (mode 600)
- **Never committed to git**
- Examples: `INTERNAL_JWT_SECRET`, `SMTP_PASS`, `DATABASE_URL`, `REDIS_URL`
- Service account JSON for GA4: `/home/kiwiton/secrets/ga4-service-account.json` (mode 600)

---

## 📚 Documentation

### Core Documents
- **[docs/ARCHITECTURE.md](./docs/ARCHITECTURE.md)** — Platform architecture, service catalog, topology, data model, migration plan
- **[docs/CPANEL_DEPLOYMENT.md](./docs/CPANEL_DEPLOYMENT.md)** — End-to-end cPanel setup playbook (GitHub App, token helper, per-service deploy)

### Key Concepts

**Polyrepo Architecture**
- Each service, app, and shared library is its own GitHub repository
- No monorepo, no pnpm workspace
- Cross-service communication via HTTP only (no source imports)

**Single Database, Schema per Service**
- One PostgreSQL instance: `kiwiton_dashboard`
- One database user: `kiwiton_bolyanatz`
- Schema per service: `auth`, `analytics`, `email`, `reports`, `alerts`
- Prisma multi-schema with per-service roles

**Authentication**
- End-user JWTs: RS256, 15-min access + 30-day refresh (issued by `auth-svc`)
- Service-to-service JWTs: HS256, 60s TTL (minted by `gateway-svc`)
- No service trusts the network — every internal request requires an internal JWT

**Rate Limiting**
- Centralized at the gateway, backed by Redis
- Default: 100 req/min per IP (unauth), 600 req/min per user (auth)
- Stricter buckets on auth endpoints (5 req/min per IP)

---

## 🛠️ Tech Stack

### Frontends
- **Framework:** Next.js 15 (App Router)
- **Styling:** Tailwind CSS v4, daisyUI
- **Deployment:** Cloudflare Pages (`@cloudflare/next-on-pages` or `@opennextjs/cloudflare`)
- **Auth:** NextAuth Credentials provider → `auth-svc` via gateway

### Backend Services
- **Runtime:** Node.js 20 LTS, ESM
- **HTTP:** Hono (`@hono/node-server`)
- **Validation:** Zod (schemas from `KTT-Contracts`)
- **ORM:** Prisma (schema from `KTT-DB-Migrations`)
- **Logging:** Pino (JSON to stdout, PM2 rotates)
- **Queue:** BullMQ (Redis-backed)

### Data Layer
- **Database:** PostgreSQL (`kiwiton_dashboard`)
- **Connection Pooling:** PgBouncer (port 6432, transaction mode)
- **Cache & Queues:** Redis (port 6379)
- **Migrations:** Prisma (run from `KTT-DB-Migrations` via manual workflow)

---

## 📋 Required Org Settings

Before using this repo, ensure the following GitHub org settings are configured:

1. **Deploy keys:** Org → Settings → Member privileges → Deploy keys: **Allowed**
2. **Actions:** Org → Settings → Actions → General → **"Allow all actions and reusable workflows"**
3. **Repo visibility:** `KTT-.github` → **Public** (so private repos can consume reusable workflows)

---

## 🔄 Migration Status

The platform is currently **in progress** (v1.0). See [docs/ARCHITECTURE.md § 9](./docs/ARCHITECTURE.md#9-migration-plan-from-current-state) for the full migration plan.

**Completed:**
- ✅ Polyrepo setup (all repos pushed to GitHub)
- ✅ `KTT-Email-Service` scaffolded
- ✅ `KTT-Gateway` scaffolded

**In Progress:**
- 🔲 `KTT-Contracts` (shared library)
- 🔲 `KTT-DB-Migrations` (unified Prisma schema)
- 🔲 `KTT-Auth-Service` scaffolding
- 🔲 Local dependency linking setup
- 🔲 GA4 service account key rotation

---

## 📞 Contact

**Owner:** KiwiTon Tech  
**GitHub Org:** [KiwiTon-Tech](https://github.com/KiwiTon-Tech)

For questions or issues, open an issue in the relevant service repo or contact the platform team.

---

*Last updated: 2026-06-02*
