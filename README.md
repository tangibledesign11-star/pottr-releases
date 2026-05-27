# Pottr

**A modern collections modeling layer for WordPress.**

> **v0.9.3** · Beta · [pottr.tangble.com](https://pottr.tangble.com)

Pottr is a WordPress plugin that gives you a high-performance, headless data layer — without touching WordPress's native post types or CPT system. Define your data model through the admin UI, store structured content in a dedicated DB layer, and expose it through a clean REST API with JWT auth and plan-based feature gates.

> **Beta** — actively developed. APIs are stable but may evolve before v1.0.

---

## What it does

WordPress is a great CMS and hosting platform. But its native data model (posts, meta) is a poor fit for modern app backends — it's slow to query, awkward to type, and requires hacks to expose cleanly via an API.

Pottr adds a proper data layer on top of it:

- **Collections** — define typed schemas (text, number, boolean, date, rich\_text, relation, media, JSON) through the admin UI or via the Blueprint API
- **REST API** — CRUD endpoints at `/wp-json/pottr/v1/` with consistent JSON responses
- **JWT Auth** — per-app user tables, completely decoupled from `wp_users`
- **Blueprint provisioning** — idempotent app setup: declare your schema, call once on deploy, never worry about drift
- **Webhooks** — real-time event delivery (`entry.created`, `entry.updated`, `entry.deleted`) with HMAC-SHA256 signatures
- **Structured logging** — every API operation is logged to `wp_pottr_logs`
- **Plan-based limits** — Free / Pro / Business tiers enforced at the API level

---

## Requirements

| Requirement | Minimum |
|---|---|
| WordPress | 6.0 |
| PHP | 7.4 |
| MySQL | 5.7 / MariaDB 10.3 |

---

## Quick start

### 1. Install & activate

Install and activate Pottr from **WP Admin → Plugins**.

On activation, Pottr:
- Creates its DB tables (`wp_pottr_entries`, `wp_pottr_logs`, `wp_pottr_webhooks`, etc.)
- Generates a public key (`pk_live_…`) and secret key (`sk_live_…`)

You can start a **14-day Pro trial** from the license page: **WP Admin → Pottr → License**.

### 2. Get your API keys

Go to **WP Admin → Pottr → API Keys**. You'll see a ready-to-copy `.env` snippet:

```env
POTTR_URL=https://your-site.com
POTTR_PUBLIC_KEY=pk_live_...
POTTR_SECRET_KEY=sk_live_...
```

- **Public key** (`pk_live_…`) — safe for client-side use. Allows auth endpoints, reads, and user-scoped writes.
- **Secret key** (`sk_live_…`) — server-side only. Required for schema management, webhooks, and blueprint provisioning.

### 3. Bootstrap your app

Send one `POST /auth/bootstrap` call on every deploy to declare your schema. It's fully idempotent — safe to run on every app start:

```bash
curl -X POST https://your-site.com/wp-json/pottr/v1/auth/bootstrap \
  -H "Authorization: Bearer pk_live_..." \
  -H "Content-Type: application/json" \
  -d '{
    "app": "my-app",
    "version": "1.0.0",
    "collections": [
      {
        "name": "Projects",
        "slug": "project",
        "fields": [
          { "key": "title",   "type": "text",   "required": true },
          { "key": "user_id", "type": "number", "required": true }
        ]
      }
    ],
    "auth": {
      "slug": "members",
      "name": "Members",
      "fields": [
        { "key": "display_name", "type": "text", "required": true }
      ]
    }
  }'
```

Response:

```json
{
  "status": "provisioned",
  "app": "my-app",
  "version": "1.0.0",
  "auth_ready": true,
  "collections_ready": ["project"],
  "collections_failed": []
}
```

---

## REST API overview

All endpoints are under `/wp-json/pottr/v1/`. Every request must include a valid key or JWT in the `Authorization` header:

```
Authorization: Bearer <pk_live_ | sk_live_ | jwt_token>
```

### Collections (CRUD)

| Method | Endpoint | Description |
|---|---|---|
| `GET` | `/collections` | List all collections |
| `GET` | `/collections/{slug}` | List entries in a collection |
| `GET` | `/collections/{slug}/{id}` | Get a single entry |
| `POST` | `/collections/{slug}` | Create an entry |
| `PUT` | `/collections/{slug}/{id}` | Replace an entry |
| `PATCH` | `/collections/{slug}/{id}` | Partial update |
| `DELETE` | `/collections/{slug}/{id}` | Soft-delete an entry |

### Auth (per-app)

| Method | Endpoint | Description |
|---|---|---|
| `POST` | `/auth/{app}/signup` | Register a new user |
| `POST` | `/auth/{app}/login` | Login, returns JWT |
| `GET` | `/auth/{app}/me` | Get current user profile |
| `PUT` | `/auth/{app}/me` | Update profile |
| `POST` | `/auth/{app}/logout` | Revoke token |
| `POST` | `/auth/{app}/forgot-password` | Send reset email |
| `POST` | `/auth/{app}/reset-password` | Confirm reset with token |
| `POST` | `/auth/bootstrap` | Idempotent app provisioning (public key) |

### Schema (secret key only)

| Method | Endpoint | Description |
|---|---|---|
| `GET` | `/schema` | List all collection schemas |
| `GET` | `/schema/{collection}` | Get field schema for one collection |

### Webhooks (secret key only)

| Method | Endpoint | Description |
|---|---|---|
| `GET` | `/webhooks` | List registered webhooks |
| `POST` | `/webhooks` | Register a new webhook |
| `DELETE` | `/webhooks/{id}` | Delete a webhook |

### Blueprint (secret key only)

| Method | Endpoint | Description |
|---|---|---|
| `POST` | `/blueprint` | Provision collections via secret key |
| `GET` | `/blueprint/{app}` | Get blueprint status for an app |

---

## Webhook delivery

Webhooks fire immediately after entry create, update, or delete. Your endpoint receives:

| Header | Value |
|---|---|
| `Content-Type` | `application/json` |
| `X-Pottr-Signature` | `sha256=<HMAC-SHA256 of body>` |
| `X-Pottr-Event` | e.g. `entry.created` |
| `X-Pottr-Webhook-ID` | Numeric webhook ID |

**Verifying signatures (Node.js):**

```js
const crypto = require('crypto');
const sig = 'sha256=' + crypto.createHmac('sha256', webhookSecret).update(rawBody).digest('hex');
if (sig !== req.headers['x-pottr-signature']) throw new Error('Invalid signature');
```

---

## Security

Pottr was built with security as a first-class concern:

- API key validation on every request (public vs. secret scope enforced)
- JWT tokens with per-user revocation
- Rate limiting per IP (with configurable trusted proxy support via `POTTR_TRUSTED_PROXIES`)
- SSRF protection on webhook URLs — private/internal IP ranges are rejected
- Entry ownership enforcement — users can only modify their own data
- Password reset tokens are single-use
- HMAC-SHA256 webhook signature verification

---

## Changelog

See [CHANGELOG.md](CHANGELOG.md) for the full version history.

---

## Contributors

- [@SarafPushkar0796](https://github.com/SarafPushkar0796)

---

## License

[GPL-2.0-or-later](LICENSE)

---

Built by [tangble](https://tangble.com) — a product studio building focused software for thoughtful work.