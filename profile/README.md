# KiwiTon-Tech (KTT)

## Service Catalog

### Frontends (Cloudflare Pages)

[KTT-Public-Web](https://github.com/KiwiTon-Tech/KTT-Public-Web) — public marketing site (`kiwiton-tech.com`).

```
KTT-Public-Web
```

[KTT-Dashboard-Web](https://github.com/KiwiTon-Tech/KTT-Dashboard-Web) — authenticated user analytics dashboard (`dashboard.kiwiton-tech.com`).

```
KTT-Dashboard-Web
```

[KTT-Admin-Web](https://github.com/KiwiTon-Tech/KTT-Admin-Web) — internal staff admin (`admin.kiwiton-tech.com`).

```
KTT-Admin-Web
```

### Backend Services (Node.js on WHM/cPanel)

[KTT-Gateway](https://github.com/KiwiTon-Tech/KTT-Gateway) — public API gateway (`api.kiwiton-tech.com`). Only public ingress.

```
KTT-Gateway
```

[KTT-Auth-Service](https://github.com/KiwiTon-Tech/KTT-Auth-Service) — authentication, sessions, OAuth.

```
KTT-Auth-Service
```

[KTT-Analytics-Service](https://github.com/KiwiTon-Tech/KTT-Analytics-Service) — Google Analytics 4 ingestion + caching.

```
KTT-Analytics-Service
```

[KTT-Email-Service](https://github.com/KiwiTon-Tech/KTT-Email-Service) — transactional email + multi-tenant contact form routing.

```
KTT-Email-Service
```

[KTT-Reports-Service](https://github.com/KiwiTon-Tech/KTT-Reports-Service) — saved reports, scheduled runs, PDF/CSV exports.

```
KTT-Reports-Service
```

[KTT-Alerts-Service](https://github.com/KiwiTon-Tech/KTT-Alerts-Service) — threshold alerts (user) and ops alerts (team).

```
KTT-Alerts-Service
```

### Shared

[KTT-DB-Migrations](https://github.com/KiwiTon-Tech/KTT-DB-Migrations) — single Prisma schema, migrations, seed scripts. Run from CI.

```
KTT-DB-Migrations
```

[KTT-Contracts](https://github.com/KiwiTon-Tech/KTT-Contracts) — zod schemas, GraphQL SDL, JWT helpers, error classes shared across services.

```
KTT-Contracts
```

## Deployment Model

Deployments are **pulled by cPanel**, not pushed by GitHub Actions. The shared cPanel host blocks inbound SSH, so the `rsync`-from-CI pattern doesn't work. We use the inverse: the cPanel server uses a GitHub App (`KTT-Deploy-Bot`) to mint a short-lived installation token and `git pull` directly from GitHub.

```
KTT-Deploy-Bot
```

```
git pull
```

Full setup playbook (one-time per cPanel server, ~15 min): [docs/CPANEL_DEPLOYMENT.md](https://github.com/KiwiTon-Tech/KTT-.github/blob/main/docs/CPANEL_DEPLOYMENT.md).

```
docs/CPANEL_DEPLOYMENT.md
```

Per-service setup (~5 min per repo): see each service's own README.

## Reusable CI Workflows

GitHub Actions runs lint + test + build on every push and pull request. Deploys are handled by the cPanel pull described above.

Other repos consume these via `uses:` — note the `KTT-.github` repo path, not `.github`:

```
uses:
```

```
KTT-.github
```

```
.github
```

```yaml
jobs:
  ci:
    uses: KiwiTon-Tech/KTT-.github/.github/workflows/node-cpanel.yml@main
    with:
      app_path: /home/kiwiton/apps/KTT-Analytics-Service
      node_version: "20"
      run_tests: true
    secrets: inherit
```

Available workflows:

- `node-cpanel.yml` — lint + test + build for Node.js services and Next.js apps.

```
node-cpanel.yml
```

The deploy/rsync steps inside this workflow still exist for hosts that do allow inbound SSH, but are not used by the production cPanel.

```
deploy
```

```
rsync
```

## Free-plan Secret Strategy

GitHub Free disallows org-level secrets on private repos. Workaround:

- Each repo's CI secrets are stored at the repo level, not the org level.
- A small bash loop (using the `gh` CLI) seeds all repos in one shot — see [docs/CPANEL_DEPLOYMENT.md](https://github.com/KiwiTon-Tech/KTT-.github/blob/main/docs/CPANEL_DEPLOYMENT.md).
- Runtime secrets (`INTERNAL_JWT_SECRET`, `GA4_SERVICE_ACCOUNT_JSON`, `SMTP_PASS`, etc.) live in cPanel environment variables per app, **never** in GitHub.

```
gh
```

```
docs/CPANEL_DEPLOYMENT.md
```

```
INTERNAL_JWT_SECRET
```

```
GA4_SERVICE_ACCOUNT_JSON
```

```
SMTP_PASS
```

## Required Org Settings

- Org → Settings → Member privileges → Deploy keys: **Allowed** (GitHub Apps + deploy keys are both used).
- Org → Settings → Actions → General → **"Allow all actions and reusable workflows"**.
- `KTT-.github` repo → visibility: **public** (so private repos can consume the reusable workflows without GitHub Team).

```
KTT-.github
```

## Architecture

See [docs/ARCHITECTURE.md](https://github.com/KiwiTon-Tech/KTT-.github/blob/main/docs/ARCHITECTURE.md) for the full service breakdown, communication flows, data model, and rollout plan.

```
docs/ARCHITECTURE.md
```

## Documents in this repo

- [docs/ARCHITECTURE.md](https://github.com/KiwiTon-Tech/KTT-.github/blob/main/docs/ARCHITECTURE.md) — service catalog, topology, communication patterns, rollout plan.
- [docs/CPANEL_DEPLOYMENT.md](https://github.com/KiwiTon-Tech/KTT-.github/blob/main/docs/CPANEL_DEPLOYMENT.md) — end-to-end cPanel setup playbook (GitHub App, token helper, per-service deploy).
