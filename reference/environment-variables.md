# EduFlow — Environment Variables Reference

Every environment variable the application reads. All of these live in `.env` (never committed to Git) and are validated by `src/config/index.js` at startup.

---

## Server

| Variable | Required | Default | Description |
|----------|----------|---------|-------------|
| `NODE_ENV` | No | `development` | Runtime environment. Must be `development`, `production`, or `test`. Controls logging verbosity and Prisma dev logging. |
| `PORT` | No | `3000` | The port Express listens on. Nginx forwards to this port in production. |

---

## Database

| Variable | Required | Default | Description |
|----------|----------|---------|-------------|
| `DATABASE_URL` | ✅ Yes | — | Full MongoDB connection string. Format: `mongodb://user:password@host:port/dbname?authSource=admin`. In development: `mongodb://admin:yourpassword@localhost:27017/eduflow?authSource=admin`. In production: point to your MongoDB Atlas cluster or managed MongoDB instance. |

---

## JWT

| Variable | Required | Default | Description |
|----------|----------|---------|-------------|
| `JWT_SECRET` | ✅ Yes | — | Secret used to sign access tokens. Minimum 32 characters. Generate with: `openssl rand -base64 48`. If this leaks, all access tokens can be forged — rotate immediately. |
| `JWT_REFRESH_SECRET` | ✅ Yes | — | Secret used to sign refresh tokens. Must be **different** from `JWT_SECRET`. Same generation command. |
| `JWT_ACCESS_EXPIRY` | No | `15m` | Access token lifetime. Format: `15m`, `1h`, `2d`. Keep this short — 15 minutes is the standard. |
| `JWT_REFRESH_EXPIRY` | No | `7d` | Refresh token lifetime. Format: `7d`, `30d`. Longer is more convenient; shorter is more secure. |

---

## Cloudinary

Sign up at [cloudinary.com](https://cloudinary.com). Find these in your Cloudinary dashboard under Settings → API Keys.

| Variable | Required | Default | Description |
|----------|----------|---------|-------------|
| `CLOUDINARY_CLOUD_NAME` | ✅ Yes | — | Your Cloudinary cloud name (e.g. `dxyz12abc`). Used in every Cloudinary API call. |
| `CLOUDINARY_API_KEY` | ✅ Yes | — | Cloudinary API key. Public-ish (safe to use in signed URL generation). |
| `CLOUDINARY_API_SECRET` | ✅ Yes | — | Cloudinary API secret. **Never expose client-side.** Used to sign URLs and authenticate uploads. |

---

## Redis

| Variable | Required | Default | Description |
|----------|----------|---------|-------------|
| `REDIS_URL` | ✅ Yes | — | Redis connection URL. In development (Docker): `redis://localhost:6379`. In production: `redis://localhost:6379` (same VPS) or a managed Redis URL. |

---

## Resend (Email)

Sign up at [resend.com](https://resend.com). Create an API key in the Resend dashboard.

| Variable | Required | Default | Description |
|----------|----------|---------|-------------|
| `RESEND_API_KEY` | ✅ Yes | — | Resend API key. Starts with `re_`. Used to send all transactional emails. |
| `EMAIL_FROM` | No | `EduFlow <noreply@yourdomain.com>` | The "from" address for all outgoing emails. Must be a verified domain in Resend. |

---

## Stripe

Sign up at [stripe.com](https://stripe.com). Use test keys during development (they start with `sk_test_`).

| Variable | Required | Default | Description |
|----------|----------|---------|-------------|
| `STRIPE_SECRET_KEY` | ✅ Yes | — | Stripe secret key. **Never expose client-side.** Used to create PaymentIntents and verify webhooks. Dev key: `sk_test_...`. Prod key: `sk_live_...`. |
| `STRIPE_WEBHOOK_SECRET` | ✅ Yes | — | Signing secret for the Stripe webhook endpoint. Find it in your Stripe dashboard under Webhooks, or via the Stripe CLI (`stripe listen` prints it). Used to verify requests are genuinely from Stripe. |

---

## Frontend

| Variable | Required | Default | Description |
|----------|----------|---------|-------------|
| `FRONTEND_URL` | ✅ Yes | — | The URL of the React frontend. Used in the CORS config to allow cross-origin requests. In development: `http://localhost:5173`. In production: `https://yourdomain.com`. |

---

## Development `.env` template

Copy this into your `.env` file and fill in the values. Add this file to `.gitignore` immediately — before any other work.

```dotenv
# ─── Server ────────────────────────────────────────────────────────────────
NODE_ENV=development
PORT=3000

# ─── Database ──────────────────────────────────────────────────────────────
DATABASE_URL=mongodb://admin:yourpassword@localhost:27017/eduflow?authSource=admin

# ─── JWT ───────────────────────────────────────────────────────────────────
# Generate with: openssl rand -base64 48
JWT_SECRET=replace-me-with-a-48-char-random-string
JWT_REFRESH_SECRET=replace-me-with-a-different-48-char-random-string
JWT_ACCESS_EXPIRY=15m
JWT_REFRESH_EXPIRY=7d

# ─── Cloudinary ────────────────────────────────────────────────────────────
CLOUDINARY_CLOUD_NAME=your_cloud_name
CLOUDINARY_API_KEY=your_api_key
CLOUDINARY_API_SECRET=your_api_secret

# ─── Redis ─────────────────────────────────────────────────────────────────
REDIS_URL=redis://localhost:6379

# ─── Email (Resend) ────────────────────────────────────────────────────────
RESEND_API_KEY=re_your_key_here
EMAIL_FROM=EduFlow <noreply@yourdomain.com>

# ─── Stripe ────────────────────────────────────────────────────────────────
STRIPE_SECRET_KEY=sk_test_your_key_here
STRIPE_WEBHOOK_SECRET=whsec_your_webhook_secret_here

# ─── CORS ──────────────────────────────────────────────────────────────────
FRONTEND_URL=http://localhost:5173
```

---

## Production secrets checklist

Before deploying, verify every item:

- [ ] `JWT_SECRET` and `JWT_REFRESH_SECRET` are freshly generated for production (not the same as local dev)
- [ ] `STRIPE_SECRET_KEY` is the **live** key (`sk_live_...`), not the test key
- [ ] `STRIPE_WEBHOOK_SECRET` matches the webhook endpoint registered in the Stripe dashboard for the production URL
- [ ] `DATABASE_URL` points to the production database
- [ ] `FRONTEND_URL` is the production frontend domain with HTTPS
- [ ] The `.env` file on the server is readable only by the deploy user (`chmod 600 .env`)
- [ ] `git log --all -- .env` shows no commits containing `.env` content

---

## How `src/config/index.js` works

The config file uses Zod to parse `process.env` at startup. If any required variable is missing or fails validation, the process prints every failing variable name and exits with code 1. This means:

- A missing `JWT_SECRET` crashes the server at boot, not at 3am when the first authenticated request comes in.
- A `PORT` that is not a number is caught before Express tries to listen on it.
- The rest of the application imports `config` from this file and never reads `process.env` directly — all env access is centralised.

This pattern is called **fail-fast configuration**. It makes deployment errors loud and immediate rather than silent and delayed.
