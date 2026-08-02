# cPanel Deployment Playbook

**Audience:** one-time setup for a cPanel host that runs the KTT backend services.
**Time:** ~15 minutes for the host setup, ~5 minutes per service repo.
**Outcome:** the cPanel server pulls each service from GitHub on demand using a short-lived installation token minted by the `KTT-Deploy-Bot` GitHub App. No inbound SSH from CI required.

---

## Why pull, not push?

The production WHM/cPanel host blocks inbound SSH from arbitrary GitHub Actions runners. The standard `rsync`-over-SSH-from-CI flow therefore can't work. Instead:

1. A GitHub App (`KTT-Deploy-Bot`) is installed on the `KiwiTon-Tech` org with `contents:read` on every `KTT-*` repo.
2. A small helper on the cPanel box exchanges the App's private key for a 1-hour installation access token.
3. `git pull` runs on the cPanel box using that token over HTTPS.
4. PM2 reloads the affected service.

CI on GitHub still runs lint + test + build on every push so broken commits never reach `main`.

---

## Part 1 — One-time host setup (~15 min)

### 1. Create the GitHub App

1. Org → **Settings** → **Developer settings** → **GitHub Apps** → **New GitHub App**.
2. Name: `KTT-Deploy-Bot`. Homepage: `https://kiwiton-tech.com`.
3. Webhook: **disabled**.
4. Permissions:
   - Repository permissions → **Contents: Read-only**.
   - Repository permissions → **Metadata: Read-only** (auto).
5. Where can this App be installed: **Only on this account**.
6. Create the app, then **Generate a private key** and download the `.pem` file.
7. Note the **App ID** (top of the app's settings page).
8. Install the app on the org and grant access to **all `KTT-*` repos**.
9. Note the **Installation ID** (visible in the URL after install).

### 2. Upload the private key to the cPanel box

```bash
# from your laptop
scp KTT-Deploy-Bot.2026-XX-XX.private-key.pem \
    ktt@cpanel-host:/home/ktt/.ssh/ktt-deploy-bot.pem

# on the cPanel box
chmod 600 /home/ktt/.ssh/ktt-deploy-bot.pem
```

### 3. Install the token helper

The helper is one bash file that mints an installation token from the App's JWT.

```bash
# /home/ktt/bin/ktt-token.sh
#!/usr/bin/env bash
set -euo pipefail

APP_ID="${KTT_APP_ID:?set KTT_APP_ID}"
INSTALLATION_ID="${KTT_INSTALLATION_ID:?set KTT_INSTALLATION_ID}"
PEM="${KTT_APP_KEY:-/home/ktt/.ssh/ktt-deploy-bot.pem}"

now=$(date +%s)
iat=$((now - 60))
exp=$((now + 540))   # 9 min, max allowed is 10

header_b64=$(printf '{"typ":"JWT","alg":"RS256"}' | openssl base64 -A | tr '+/' '-_' | tr -d '=')
payload_b64=$(printf '{"iat":%d,"exp":%d,"iss":"%s"}' "$iat" "$exp" "$APP_ID" \
              | openssl base64 -A | tr '+/' '-_' | tr -d '=')
sig=$(printf '%s.%s' "$header_b64" "$payload_b64" \
      | openssl dgst -sha256 -sign "$PEM" \
      | openssl base64 -A | tr '+/' '-_' | tr -d '=')
jwt="$header_b64.$payload_b64.$sig"

curl -sS -X POST \
  -H "Authorization: Bearer $jwt" \
  -H "Accept: application/vnd.github+json" \
  "https://api.github.com/app/installations/${INSTALLATION_ID}/access_tokens" \
  | grep -oE '"token": *"[^"]+"' | cut -d'"' -f4
```

```bash
chmod +x /home/ktt/bin/ktt-token.sh

# put these in ~/.bashrc or a sourced env file
export KTT_APP_ID=123456
export KTT_INSTALLATION_ID=987654321
export KTT_APP_KEY=/home/ktt/.ssh/ktt-deploy-bot.pem
```

Test it:

```bash
$ /home/ktt/bin/ktt-token.sh
ghs_xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
```

### 4. Install the per-service deploy script

```bash
# /home/ktt/bin/ktt-deploy.sh
#!/usr/bin/env bash
set -euo pipefail

REPO="${1:?usage: ktt-deploy.sh <REPO> [<REF>]}"
REF="${2:-main}"
APP_DIR="/home/ktt/apps/${REPO}"

TOKEN=$(/home/ktt/bin/ktt-token.sh)
URL="https://x-access-token:${TOKEN}@github.com/KiwiTon-Tech/${REPO}.git"

if [ ! -d "${APP_DIR}/.git" ]; then
  git clone --depth=1 --branch "$REF" "$URL" "$APP_DIR"
else
  git -C "$APP_DIR" remote set-url origin "$URL"
  git -C "$APP_DIR" fetch --depth=1 origin "$REF"
  git -C "$APP_DIR" reset --hard "origin/$REF"
fi

cd "$APP_DIR"

if [ -f requirements.txt ]; then
  # Python service: ensure the virtualenv exists and install dependencies.
  if [ ! -d .venv ]; then
    /opt/alt/python311/bin/python3.11 -m venv .venv
  fi
  .venv/bin/pip install --upgrade pip setuptools wheel
  .venv/bin/pip install -r requirements.txt
elif [ -f package.json ]; then
  # Node service: NODE_ENV=development for install+build so devDependencies
  # (e.g. typescript) are installed even though the host's default NODE_ENV
  # is production — `npm ci` respects NODE_ENV and silently omits
  # devDependencies otherwise.
  NODE_ENV=development npm ci
  NODE_ENV=development npm run build --if-present || true
  npm prune --omit=dev
else
  echo "No requirements.txt or package.json found in $APP_DIR" >&2
  exit 1
fi

# Use `restart`, not `reload`, in fork mode — `reload` can race on the port
# and throw EADDRINUSE if the old process hasn't released it yet.
pm2 restart "$REPO" || pm2 start /home/ktt/apps/ecosystem.config.cjs --only "$REPO"
pm2 save
```

```bash
chmod +x /home/ktt/bin/ktt-deploy.sh
```

> **If a deploy still shows `EADDRINUSE` or a stale env var (wrong `PORT`,
> wrong `DATABASE_URL`) after this script runs:** PM2's saved process dump
> can carry over stale env from a previous run or even a different service.
> Do a clean restart: `pm2 delete <REPO>` then
> `pm2 start /home/ktt/apps/ecosystem.config.cjs --only <REPO>`, and set any
> known-correct values explicitly in that service's `env` block in
> `ecosystem.config.cjs` (see below) so they can't be overridden by a stale
> dump or the shell's inherited environment.

### 5. PM2 ecosystem file

> **Actual production file is `ecosystem.config.cjs` (CommonJS extension),
> not `.js`** — and the working services use `script: 'node'` +
> `args: '--env-file <path> dist/index.js'` rather than `env_file`, since
> `--env-file` is a native Node.js flag (Node 20+) and doesn't require any
> extra PM2 config. Any var that has previously come from a stale PM2 dump
> (commonly `PORT` or `DATABASE_URL`) should be pinned explicitly in that
> app's `env` block so PM2 sets it directly before spawning the process,
> overriding anything inherited from the shell or a prior dump.

```js
// /home/ktt/apps/ecosystem.config.cjs
module.exports = {
  apps: [
    { name: 'KTT-Gateway',           cwd: '/home/ktt/apps/KTT-Gateway',           script: 'node', args: '--env-file /home/ktt/env/gateway.env dist/index.js' },
    { name: 'KTT-Auth-Service',      cwd: '/home/ktt/apps/KTT-Auth-Service',      script: 'node', args: '--env-file /home/ktt/env/auth.env dist/index.js' },
    {
      name: 'KTT-Analytics-Service',
      cwd: '/home/ktt/apps/KTT-Analytics-Service',
      script: 'node',
      args: '--env-file /home/ktt/env/analytics.env dist/index.js',
      // Pinned explicitly: a stale PM2 dump previously carried over
      // PORT=5030 and another service's DATABASE_URL, overriding --env-file.
      env: {
        PORT: '5020',
        DATABASE_URL: 'postgresql://analytics_svc:<url-encoded-password>@127.0.0.1:5432/kiwiton_dashboard?schema=analytics',
      },
    },
    { name: 'KTT-Email-Service',     cwd: '/home/ktt/apps/KTT-Email-Service',     script: 'node', args: '--env-file /home/ktt/env/email.env dist/index.js' },
    { name: 'KTT-Reports-Service',   cwd: '/home/ktt/apps/KTT-Reports-Service',   script: 'node', args: '--env-file /home/ktt/env/reports.env dist/index.js' },
    { name: 'KTT-Alerts-Service',    cwd: '/home/ktt/apps/KTT-Alerts-Service',    script: 'node', args: '--env-file /home/ktt/env/alerts.env dist/index.js' },
    { name: 'KTT-Inventory-API',   cwd: '/home/ktt/apps/KTT-Inventory-API',   script: '/home/ktt/apps/KTT-Inventory-API/.venv/bin/python', args: '-m gunicorn -b 127.0.0.1:5050 --workers 2 --timeout 60 wsgi:app', env: { APP_ENV: 'production', HOST: '127.0.0.1', PORT: '5050' } },
  ],
};
```

### 6. Apache reverse proxy

Only `KTT-Gateway` is exposed. In WHM → cPanel → **Apache Configuration** → **Include Editor**, add for `api.kiwiton-tech.com`:

```apache
<VirtualHost *:443>
  ServerName api.kiwiton-tech.com
  ProxyPreserveHost On
  ProxyPass        / http://127.0.0.1:5001/
  ProxyPassReverse / http://127.0.0.1:5001/
</VirtualHost>
```

All other service ports are firewalled to `127.0.0.1` only via CSF.

### 7. PostgreSQL + Redis

- Create the Postgres database (`kiwiton_dashboard`) and per-service roles described in `ARCHITECTURE.md` §6 / `KTT-DB/README.md`. The Inventory API uses a separate `ktt_toro_prod` database with a `ktt_toro_owner` role.
- PgBouncer is **not** currently installed — services connect directly to `127.0.0.1:5432`. (Original plan called for PgBouncer on `6432`; revisit if connection pooling becomes a bottleneck.)
- Install Redis (`127.0.0.1:6379`, password-protected).
- **Check `pg_hba.conf` before assuming GRANTs are the problem.** cPanel's
  default config (`/var/lib/pgsql/data/pg_hba.conf`) commonly ships with a
  `samerole` rule:
  ```
  host samerole all  127.0.0.1   255.255.255.255   md5
  ```
  `samerole` only allows a role to connect to a database whose name matches
  a role it's a member of. Per-service roles (`analytics_svc`, `auth_svc`,
  etc.) are **not** members of a role named `kiwiton_dashboard`, so they're
  rejected at the auth layer — before any `GRANT` is even checked. Prisma
  surfaces this as a generic `P1010 "denied access on the database"`, which
  looks identical to a missing-GRANT error. Add an explicit rule (as root)
  and reload:
  ```bash
  sudo bash -c 'echo "host    kiwiton_dashboard    analytics_svc,auth_svc,email_svc,reports_svc,alerts_svc,gateway_svc,inventory_svc    127.0.0.1/32    md5" >> /var/lib/pgsql/data/pg_hba.conf'
  sudo bash -c 'echo "host    ktt_toro_prod        ktt_toro_owner    127.0.0.1/32    md5" >> /var/lib/pgsql/data/pg_hba.conf'
  sudo -u postgres pg_ctl reload -D /var/lib/pgsql/data
  ```
  Verify directly with `psql -h 127.0.0.1 -U <svc> -d <database> -c "SELECT 1;"` before debugging GRANTs.

---

## Part 2 — Per-service setup (~5 min per repo)

For each `KTT-*` service repo:

1. **First-time deploy** on the cPanel box:
   ```bash
   /home/ktt/bin/ktt-deploy.sh KTT-Auth-Service main
   ```
2. **Create the runtime env file** at `/home/ktt/env/<service>.env` (mode 600). Never commit this.
3. **Add the CI workflow** at `.github/workflows/ci.yml` in the repo:
   ```yaml
   name: CI
   on: [push, pull_request]
   jobs:
     ci:
       uses: KiwiTon-Tech/KTT-.github/.github/workflows/node-cpanel.yml@main
       with:
         app_path: /home/ktt/apps/KTT-Auth-Service
         node_version: "20"
         run_tests: true
       secrets: inherit
   ```
4. **(Optional) Auto-deploy on push to `main`** — add a webhook receiver later that runs `ktt-deploy.sh <REPO>` when GitHub fires a `push` event. Until that exists, deploys are a single SSH command.

---

## Part 3 — Seeding repo-level secrets (Free plan)

GitHub Free can't share secrets across private repos at the org level. Seed each repo individually with `gh`:

```bash
# Adjust the secret list per service
SECRETS=(INTERNAL_JWT_SECRET DATABASE_URL REDIS_URL)
REPOS=(KTT-Gateway KTT-Auth-Service KTT-Analytics-Service KTT-Email-Service KTT-Cigar-Hub KTT-Inventory-API KTT-Reports-Service KTT-Alerts-Service KTT-DB)

for repo in "${REPOS[@]}"; do
  for s in "${SECRETS[@]}"; do
    gh secret set "$s" --repo "KiwiTon-Tech/$repo" --body "${!s}"
  done
done
```

Runtime secrets (the actual values that services read at startup) live in `/home/ktt/env/*.env` on the cPanel box. They are **not** stored in GitHub — only CI-time secrets are.

---

## Troubleshooting

| Symptom | Likely cause |
|---|---|
| `ktt-token.sh` returns empty | App private key path wrong, or App ID / Installation ID env vars missing. |
| `git clone` 403 | The App isn't installed on that repo, or the token expired (>1 h since mint). |
| `pm2 restart`/`reload` says process not found | First-time deploy — use `pm2 start /home/ktt/apps/ecosystem.config.cjs --only <name>` once. |
| Service starts then crashes | Missing env file or wrong path in `ecosystem.config.cjs`. Check `pm2 logs <name> --err --lines 20 --nostream`. If the log is empty despite repeated restarts, run the exact `node --env-file ... dist/index.js` command manually to see the raw crash. |
| `EADDRINUSE` on the service's own port | Stale PM2 dump or a leftover process still holding the port. Do a clean `pm2 delete <name>` + `pm2 start ... --only <name>` instead of `restart`/`reload`. |
| Wrong `PORT` / `DATABASE_URL` after deploy (`pm2 env <id>` shows an unexpected value) | PM2's dumped env carried over a stale value (sometimes from a *different* service). Pin the correct value explicitly in that app's `env` block in `ecosystem.config.cjs`, then `pm2 delete` + `pm2 start` (not `restart`). |
| `tsc: command not found` during deploy | Host's global `NODE_ENV=production` makes `npm ci` skip devDependencies (including `typescript`). `ktt-deploy.sh` explicitly sets `NODE_ENV=development` for the install/build steps — confirm that override is present. |
| Prisma `P1010 "denied access on the database"` even though GRANTs look correct | Check `pg_hba.conf`'s `samerole` rule — see Part 1 §7 above. This is the most common root cause on this host and is easy to mistake for a GRANT problem. |
| Apache 502 on `api.kiwiton-tech.com` | Gateway not running or listening on wrong port. `pm2 status` and `curl 127.0.0.1:5001/health`. |
