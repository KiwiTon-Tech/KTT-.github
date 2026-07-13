# KTT-Gateway Deployment Guide

Complete guide for deploying and maintaining the KTT-Gateway and KTT-Email-Service on cPanel with PM2 and Apache reverse proxy.

---

## Architecture Overview

```
Browser / Cloudflare Pages Function
        ↓
Cloudflare CDN (api.kiwiton-tech.com)
        ↓
Apache (cPanel) — reverse proxy via userdata conf
        ↓
PM2 → KTT-Gateway (port 5001)
        ↓
PM2 → KTT-Email-Service (port 5030)
        ↓
PostgreSQL + Redis + SMTP (cPanel server)
```

---

## Prerequisites

- cPanel account: `ktt` on `server.t-4.net`
- WHM root access for Apache config changes
- PM2 installed globally: `npm install -g pm2`
- Node.js 18+ available on the server
- Redis running: `ps aux | grep redis`

---

## 1. Directory Structure on Server

```
/home/ktt/
├── apps/
│   ├── KTT-Gateway/        # Deployed app (git pull here)
│   └── KTT-Email-Service/  # Deployed app (git pull here)
├── env/
│   ├── gateway.env         # Environment variables for gateway
│   └── email.env           # Environment variables for email service
├── ecosystem.config.cjs    # PM2 process config
└── logs/                   # PM2 logs (via ~/.pm2/logs/)
```

---

## 2. Environment Variables

### `~/env/gateway.env`

```env
NODE_ENV=production
HOST=0.0.0.0
PORT=5001
LOG_LEVEL=info

REDIS_URL=redis://127.0.0.1:6379/1

INTERNAL_JWT_SECRET=<value from GitHub secret INTERNAL_JWT_SECRET>

EMAIL_SERVICE_URL=http://127.0.0.1:5030
# Add other downstream service URLs as they are deployed
```

### `~/env/email.env`

```env
NODE_ENV=production
HOST=127.0.0.1
PORT=5030
LOG_LEVEL=info

DATABASE_URL=postgresql://email_svc:<password>@127.0.0.1:5432/ktt_dashboard?schema=email

REDIS_URL=redis://127.0.0.1:6379/0

INTERNAL_JWT_SECRET=<same value as gateway — from GitHub secret INTERNAL_JWT_SECRET>

SMTP_HOST=server.t-4.net
SMTP_PORT=465
SMTP_SECURE=true
SMTP_USER=postmaster@kiwiton-tech.com
SMTP_PASS=<cPanel email account password>
EMAIL_FROM_ADDRESS=postmaster@kiwiton-tech.com
EMAIL_FROM_NAME=KiwiTon Tech
```

> **Note:** `REDIS_URL` uses database index `/1` for the gateway and `/0` for the email service to keep their keyspaces separate.

---

## 3. PM2 Ecosystem Config

File: `~/ecosystem.config.cjs`

```js
module.exports = {
  apps: [
    {
      name: 'KTT-Gateway',
      script: 'dist/index.js',
      cwd: '/home/ktt/apps/KTT-Gateway',
      node_args: '--env-file=/home/ktt/env/gateway.env',
      instances: 1,
      autorestart: true,
    },
    {
      name: 'KTT-Email-Service',
      script: 'dist/index.js',
      cwd: '/home/ktt/apps/KTT-Email-Service',
      node_args: '--env-file=/home/ktt/env/email.env',
      instances: 1,
      autorestart: true,
    },
  ],
};
```

Start all services:
```bash
pm2 start ~/ecosystem.config.cjs
pm2 save
pm2 startup  # follow the printed command to enable on reboot
```

---

## 4. Apache Reverse Proxy (WHM)

The cPanel-generated vhost for `api.kiwiton-tech.com` sets `SetHandler proxy:unix:...fcgi` (PHP-FPM) by default. We override this via userdata conf files.

### Standard (HTTP) vhost

```bash
mkdir -p /etc/apache2/conf.d/userdata/std/2_4/ktt/api.kiwiton-tech.com/

cat > /etc/apache2/conf.d/userdata/std/2_4/ktt/api.kiwiton-tech.com/proxy_node.conf << 'EOF'
<Location />
    SetHandler none
    ProxyPass http://127.0.0.1:5001/
    ProxyPassReverse http://127.0.0.1:5001/
</Location>
EOF
```

### SSL (HTTPS) vhost — required for external access

```bash
mkdir -p /etc/apache2/conf.d/userdata/ssl/2_4/ktt/api.kiwiton-tech.com/

cat > /etc/apache2/conf.d/userdata/ssl/2_4/ktt/api.kiwiton-tech.com/proxy_node.conf << 'EOF'
<Location />
    SetHandler none
    ProxyPass http://127.0.0.1:5001/
    ProxyPassReverse http://127.0.0.1:5001/
</Location>
EOF
```

### Rebuild Apache config and restart

```bash
/scripts/rebuildhttpdconf && /scripts/restartsrv_httpd
```

### Verify

```bash
curl -s https://api.kiwiton-tech.com/health
```

Expected response:
```json
{"status":"degraded","checks":{"redis":{"status":"up"},"email":{"status":"up"},...}}
```

> `degraded` is expected until all downstream services (auth, analytics, reports, alerts) are deployed.

---

## 5. Adding a New Email Client (Site)

Each site that uses the contact form needs a row in `email.email_clients`.

### Generate API key and hash

```bash
# On any machine with Node.js
node -e "
const crypto = require('crypto');
const key = crypto.randomBytes(32).toString('hex');
const hash = crypto.createHash('sha256').update(key).digest('hex');
console.log('API Key (give to site):', key);
console.log('API Key Hash (store in DB):', hash);
"
```

### Insert into database

```sql
INSERT INTO email."EmailClient" (
  id, "siteKey", "apiKeyHash", "allowedOrigin",
  recipients, active, "rateLimit", "createdAt", "updatedAt"
) VALUES (
  'cuid_here',
  'your-site-key',
  'sha256_hash_here',
  'https://yoursite.com',
  ARRAY['contact@yoursite.com'],
  true,
  10,
  NOW(),
  NOW()
);
```

### Set in Cloudflare Pages

Go to **Cloudflare Dashboard → Pages → [site] → Settings → Environment variables**:
- `KIWITON_SITE_KEY` = `your-site-key`
- `KIWITON_API_KEY` = `<plaintext key from generation step>`

---

## 6. Deploying Updates

### Gateway or Email Service update

```bash
cd ~/apps/KTT-Gateway   # or KTT-Email-Service
git pull origin main
npm ci
npm run build
pm2 restart KTT-Gateway  # or KTT-Email-Service
```

### Verify after restart

```bash
pm2 status
pm2 logs KTT-Gateway --lines 20 --nostream
curl -s https://api.kiwiton-tech.com/health
```

---

## 7. Troubleshooting

### Gateway returns 500 HTML page

Apache is not proxying to Node. Check:
```bash
grep "proxy_node\|127.0.0.1:5001" /usr/local/apache/conf/httpd.conf
```
If missing, the userdata conf files weren't picked up. Re-create them (Section 4) and rebuild.

### Email service Redis ECONNREFUSED

Redis isn't running or the URL has a wrong password:
```bash
# As root
ps aux | grep redis
grep requirepass /etc/redis.conf
```
If no password: `REDIS_URL=redis://127.0.0.1:6379/0`

### SMTP 530 Relaying not allowed

The `EMAIL_FROM_ADDRESS` is not a valid mailbox on the cPanel server. Either:
- Create the mailbox in cPanel → Email Accounts
- Or use an existing mailbox (e.g. `postmaster@kiwiton-tech.com`)

### Contact form 401 Unauthorized

The `x-ktt-site-key` or `x-ktt-api-key` headers are wrong. Verify:
```bash
psql "$DATABASE_URL" -c "SELECT \"siteKey\", \"allowedOrigin\", active FROM email.\"EmailClient\";"
```

### PM2 process not starting after reboot

```bash
pm2 resurrect
# If that fails:
pm2 start ~/ecosystem.config.cjs
pm2 save
```

---

## 8. Key URLs and Ports

| Service | Internal URL | External URL |
|---|---|---|
| KTT-Gateway | `http://127.0.0.1:5001` | `https://api.kiwiton-tech.com` |
| KTT-Email-Service | `http://127.0.0.1:5030` | Internal only |
| Redis | `redis://127.0.0.1:6379` | Internal only |
| PostgreSQL | `postgresql://127.0.0.1:5432` | Internal only |

---

## 9. GitHub Secrets Reference

These secrets are used by GitHub Actions CI/CD and must also be manually set in `~/env/*.env` on the server:

| Secret | Used in |
|---|---|
| `INTERNAL_JWT_SECRET` | `gateway.env`, `email.env` |
| `REDIS_URL` | `gateway.env`, `email.env` |
| `DATABASE_URL` | `email.env` |
| `SMTP_HOST` / `SMTP_PASS` | `email.env` |
| `CPANEL_HOST` / `CPANEL_USER` / `CPANEL_SSH_KEY` | CI deploy workflow |
