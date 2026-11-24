# Pottr

**The backend layer for web apps built on WordPress.**

[![Latest release](https://img.shields.io/github/v/release/tangibledesign11-star/pottr-releases?label=latest&color=blue)](https://github.com/tangibledesign11-star/pottr-releases/releases/latest)
[![License: GPL v2+](https://img.shields.io/badge/License-GPLv2%2B-blue.svg)](https://www.gnu.org/licenses/gpl-2.0.html)
[![WordPress 6.0+](https://img.shields.io/badge/WordPress-6.0%2B-blue.svg)](https://wordpress.org)
[![PHP 7.4+](https://img.shields.io/badge/PHP-7.4%2B-indigo.svg)](https://php.net)

Pottr is a WordPress plugin that adds structured collections, a REST API, separate app-user logins and signed webhooks, so you can build client portals, directories, booking tools and other apps on WordPress without a second backend. Your client keeps the WordPress admin they know, and the data stays in their own database.

> **Beta.** Pottr is in active development. The current version is always the latest [release](https://github.com/tangibledesign11-star/pottr-releases/releases/latest).

**This repository publishes the plugin zip.** Each release has the installable `pottr-vX.Y.Z.zip`, its `.sha256` checksum and the release notes. Documentation, pricing and support are at [pottr.tangble.com](https://pottr.tangble.com).

---

## Install

Pottr is **not on WordPress.org**. It is distributed from [pottr.tangble.com](https://pottr.tangble.com) and from this repository.

1. Download `pottr-vX.Y.Z.zip` from the [latest release](https://github.com/tangibledesign11-star/pottr-releases/releases/latest) (or from pottr.tangble.com).
2. In WordPress, go to **Plugins > Add New > Upload Plugin**, choose the zip and activate it. Or use WP-CLI: `wp plugin install pottr-vX.Y.Z.zip --activate`.
3. Open **Pottr** in the admin menu. A **Getting started** checklist walks you through your first collection and your API keys.

Optional: check the download against the `.sha256` file on the release page (`shasum -a 256 pottr-vX.Y.Z.zip`).

**Updates** arrive in the normal WordPress update screen. Pottr checks pottr.tangble.com about twice a day, and **Pottr > License > Check for updates** checks right now. Updates are free on every plan and need no license key.

### Requirements

| Requirement | Minimum |
|---|---|
| WordPress | 6.0 |
| PHP | 7.4 |
| Database | MySQL 5.7 or MariaDB 10.2.3 |

---

## What you get

- **Collections.** Define typed collections and fields in the admin, or declare them from your app with a Blueprint. Entries are stored as JSON documents in their own table (`wp_pottr_entries`), not in posts or post meta, so your WordPress tables stay small.
- **REST API.** CRUD for every collection at `/wp-json/pottr/v1/`, with filtering, sorting, pagination, single-level relation expansion (`?expand=`) and optimistic concurrency.
- **App users.** Separate sign-in for the people who use your app, kept in per-app tables and never in `wp_users`. Short-lived access tokens, single-use refresh tokens and per-device sessions.
- **Blueprints.** Describe your app's collections and user fields in one JSON file, send it on deploy, and Pottr adds what is missing. It never deletes or changes existing fields.
- **Zero-config connect.** An app can ask for the collections it needs and an administrator approves it with one click, or a plugin on the same site can call `pottr_connect_app()`.
- **Webhooks.** Signed (HMAC-SHA256) POST requests when entries change, with a delivery log and a manual Retry for failed deliveries.
- **Media uploads.** Upload files through the API and get back the URL and metadata to attach to entries.
- **Live events** (Pro and Business). A Server-Sent Events stream of entry changes.
- **Logs and usage.** API activity, webhook deliveries and plan usage in the dashboard.

---

## Quick start

### 1. Get your API keys

Open **Pottr > API Keys**. You get a ready-to-copy `.env` snippet:

```env
POTTR_URL=https://your-site.com
POTTR_PUBLIC_KEY=pk_live_...
POTTR_SECRET_KEY=sk_live_...
```

- **Public key** (`pk_live_...`) is for browsers and mobile apps. User-owned collections still require a signed-in app user.
- **Secret key** (`sk_live_...`) is for servers only and has full access. It is accepted only in the `Authorization` header, never in a URL.

### 2. Declare your app

Send a Blueprint on every deploy. It is additive and safe to repeat:

```bash
curl -X POST https://your-site.com/wp-json/pottr/v1/auth/bootstrap \
  -H "Authorization: Bearer pk_live_..." \
  -H "Content-Type: application/json" \
  -d '{
    "app": "my-app",
    "version": "1.0.0",
    "collections": [
      {
        "slug": "project",
        "name": "Projects",
        "fields": [
          { "key": "title", "type": "text", "required": true },
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

`POST /blueprint` does the same with the secret key. The response lists each collection as `created`, `existing` or `failed` (for example when a plan limit is reached).

### 3. Create and read entries

```bash
curl -X POST https://your-site.com/wp-json/pottr/v1/collections/project \
  -H "Authorization: Bearer sk_live_..." \
  -H "Content-Type: application/json" \
  -d '{ "title": "Client portal", "user_id": 1 }'

curl "https://your-site.com/wp-json/pottr/v1/collections/project?limit=10" \
  -H "Authorization: Bearer sk_live_..."
```

When a browser app uses the public key and signs a user in, send both headers:

```http
Authorization: Bearer pk_live_...
X-Pottr-User-Token: <app-user access token>
```

---

## REST API overview

Everything lives under `/wp-json/pottr/v1/`. Send a key or an app-user token as `Authorization: Bearer ...`.

| Area | Routes |
|---|---|
| Collections | `GET`, `POST /collections/{slug}` and `GET`, `PUT`, `PATCH`, `DELETE /collections/{slug}/{id}` |
| Schema (secret key) | `GET /schema`, `GET /schema/{collection}` |
| Blueprints | `POST /blueprint` (secret key), `POST /auth/bootstrap` (public key), `GET /blueprint/{app}` |
| App users | `POST /auth/signup`, `/auth/login`, `/auth/refresh`, `/auth/logout`, `/auth/forgot-password`, `/auth/reset-password`; `GET /auth/me`, `/auth/sessions`; `DELETE /auth/sessions/{family_id}` |
| Webhooks (secret key) | `GET`, `POST /webhooks`; `PUT`, `DELETE /webhooks/{id}`; `POST /webhooks/{id}/test` |
| Media | `POST /media`, `DELETE /media/{id}` |
| Live events (Pro, Business) | `GET /events` |
| Connect | `POST /connect/request`, `GET /connect/status` |

Identify the app for user routes with the `X-Pottr-App` header. The full route list, request and response shapes and the database schema are in the [developer documentation](https://pottr.tangble.com/docs).

---

## Webhooks

Pottr sends a POST request when an entry is created, updated or deleted. Delivery runs in the background with a five-second timeout. A failed delivery is logged and can be retried from the delivery log; Pottr does not retry on its own.

| Header | Value |
|---|---|
| `Content-Type` | `application/json` |
| `X-Pottr-Signature` | `sha256=<HMAC-SHA256 of the raw body>` |
| `X-Pottr-Event` | For example `entry.created` |
| `X-Pottr-Webhook-ID` | The webhook's ID |

Verify the signature before you trust a request (Node.js):

```js
const crypto = require('crypto');

const expected = 'sha256=' + crypto.createHmac('sha256', webhookSecret).update(rawBody).digest('hex');
const received = req.headers['x-pottr-signature'] || '';
const valid = expected.length === received.length &&
  crypto.timingSafeEqual(Buffer.from(expected), Buffer.from(received));
if (!valid) throw new Error('Invalid signature');
```

Webhooks send a generic signed POST. To reach Zapier, Make or Slack, point the webhook at their incoming-webhook URL or at your own receiver.

---

## Plans

Pottr is one plugin. Download it, install it and you are on **Free**. **Pro** and **Business** raise the limits and unlock live events and webhook diagnostics; you unlock them by pasting a license key under **Pottr > License**, with no second plugin to install.

| | Free | Pro | Business |
|---|---|---|---|
| Collections | 3 | 15 | Unlimited |
| Entries per collection | 1,000 | 10,000 | Unlimited |
| API calls per month | 5,000 | 100,000 | Unlimited |
| Webhooks | 1 | 10 | Unlimited |
| Log retention | 3 days | 30 days | 365 days |
| Media upload size | 10 MB | 25 MB | 50 MB |
| Live events and webhook diagnostics | no | yes | yes |
| Sites | any | 1 | unlimited |

Cancelling or an expired license returns the site to the Free limits and never deletes data. Pricing and checkout: [pottr.tangble.com](https://pottr.tangble.com).

---

## Security

- Every request needs a valid key or app-user token. Public and secret scopes are enforced separately.
- Secret keys are accepted only in the `Authorization` header. Tokens and keys are never read from URLs (the live-events stream is the one exception, because browsers cannot set headers on it, and it never accepts the secret key).
- App-user access tokens are signed and expire after 15 minutes. Refresh tokens are single-use, stored hashed, and replaying one revokes that user's sessions.
- Rate limiting per client, with trusted-proxy support.
- Webhook URLs are checked to block private and internal addresses, and every delivery is signed.
- Users can only reach their own entries in user-owned collections.

Found a vulnerability? Please email [hello@tangble.com](mailto:hello@tangble.com) rather than opening a public issue.

---

## Help and releases

- **Documentation:** [pottr.tangble.com/docs](https://pottr.tangble.com/docs)
- **Support:** in WordPress, open **Pottr > Settings > Support** and use **Copy site info**, then email [hello@tangble.com](mailto:hello@tangble.com) with the details. The site info lists versions, your plan and basic counts, and never includes keys, your license key, your site address or app data. More at [pottr.tangble.com/support](https://pottr.tangble.com/support).
- **Release notes:** every version is on the [Releases](https://github.com/tangibledesign11-star/pottr-releases/releases) page. The same notes are in the plugin under **Pottr > Changelog**.

This repository only holds releases. Please do not open issues or pull requests here; use the support email above.

---

## License

Pottr is open-source software licensed under the **GNU General Public License v2.0 or later**. The full license text is included in the plugin zip.

---

Pottr is a product from [tangble](https://tangble.com), a studio that designs and builds focused web applications.
