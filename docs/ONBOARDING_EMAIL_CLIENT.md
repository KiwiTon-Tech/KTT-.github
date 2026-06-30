# Onboarding a New Email Client

Use this guide every time a new site needs to send contact form emails through the KTT Email Service.

---

## Step 1 — Register the client in the database

SSH into the server and run psql:

```bash
psql -U ktt_admin -h 127.0.0.1 -d ktt_dashboard
```

Generate a UUID and API key:

```bash
cat /proc/sys/kernel/random/uuid   # → UUID
node -e "require('crypto').randomBytes(32).toString('hex')" | tr -d '\n' > /tmp/apikey.txt
cat /tmp/apikey.txt   # copy this — it's the raw API key
node -e "const c=require('crypto'),k=require('fs').readFileSync('/tmp/apikey.txt','utf8').trim();console.log(c.createHash('sha256').update(k).digest('hex'))"  # → SHA-256 hash
```

Insert the client (replace all `<PLACEHOLDER>` values):

```sql
INSERT INTO email.email_clients (id, "siteKey", "apiKeyHash", "allowedOrigin", recipients, "fromName", "rateLimit", "createdAt", "updatedAt")
VALUES (
  '<UUID>',
  '<site-key>',                          -- e.g. 'acme-corp'
  '<SHA256_OF_API_KEY>',
  ARRAY['https://<domain.com>'],
  ARRAY['<recipient@domain.com>'],
  '<From Name>',
  60,
  now(),
  now()
);
```

Clean up: `rm /tmp/apikey.txt`

---

## Step 2 — Install the SDK in the Next.js project

```bash
npm install @kiwiton-tech/email-sdk
```

---

## Step 3 — Add the environment variable

Add to the site's `.env` (and to the deployment environment variables):

```
KIWITON_API_KEY=<raw_api_key_from_step_1>
```

---

## Step 4 — Create the API route

Create `app/api/contact/route.ts`:

```ts
import { createContactRoute } from '@kiwiton-tech/email-sdk/next';

export const POST = createContactRoute({
  siteKey: '<site-key>',
  apiKey: process.env.KIWITON_API_KEY!,
  baseUrl: 'https://api.kiwiton-tech.com',
});
```

---

## Step 5 — Update the contact form

Change the form's `fetch` target from any third-party service to `/api/contact`:

```ts
const response = await fetch('/api/contact', {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({ name, email, subject, message }),
});
```

---

## Step 6 — Test

```bash
curl -X POST http://127.0.0.1:5030/v1/contact \
  -H "Content-Type: application/json" \
  -H "x-ktt-site-key: <site-key>" \
  -H "x-ktt-api-key: <raw_api_key>" \
  -d '{"name":"Test","email":"test@example.com","subject":"Test","message":"Hello"}'
```

Expected: `{"success":true,...}`

---

## Reference

| Field | Description |
|---|---|
| `siteKey` | Unique slug for the site, e.g. `acme-corp` |
| `apiKey` | Raw 64-char hex key — stored only in env vars, never in DB |
| `apiKeyHash` | SHA-256 of the raw key — stored in DB |
| `allowedOrigin` | Array of allowed origin URLs |
| `recipients` | Array of email addresses that receive contact submissions |
| `fromName` | Display name in the From header |
| `rateLimit` | Max requests per minute per IP |
