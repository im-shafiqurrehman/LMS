# Rate Limiting

Auth endpoints are an obvious target. A `POST /auth/login` that accepts unlimited requests is an open invitation to brute-force password guessing. At 100 requests per second, an attacker can try 360,000 passwords per hour against a single account. With a common password list and no rate limiting, a typical account is compromised in minutes.

Rate limiting caps how many requests a client can make in a time window. It is one of the cheapest security measures you can add and one of the most impactful.

---

## What to rate limit and how

Rate limiting applies differently depending on what's being protected:

**Auth endpoints** (register, login, forgot-password): rate-limit by IP address. An attacker cycling through password guesses from one IP should be cut off fast. A reasonable limit: 10 requests per 15 minutes per IP.

**General API endpoints**: rate-limit by authenticated user ID to prevent one account from hammering the API. A reasonable limit: 100 requests per minute per user. This is a gentler guard against scraping or misbehaving clients.

**Webhook endpoints**: don't rate-limit — they're called by Stripe's servers, not end users, and blocking them would cause payment events to be lost.

---

## Install express-rate-limit with Redis store

Using Redis as the store for rate limiting ensures that if you run multiple servers, the rate limit counts are shared across all of them. A purely in-memory limiter would count separately on each server — giving each server its own limit rather than a shared one.

```bash
npm install express-rate-limit rate-limit-redis
```

Configure in `src/middleware/rateLimiter.js`:

```js
// Shape — auth limiter
import rateLimit from 'express-rate-limit';
import RedisStore from 'rate-limit-redis';
import { redis } from '../lib/redis.js';

export const authLimiter = rateLimit({
  windowMs: 15 * 60 * 1000,  // 15 minutes
  max: 10,                     // 10 requests per window per IP
  standardHeaders: true,       // Return rate limit info in the RateLimit-* headers
  legacyHeaders: false,
  store: new RedisStore({
    sendCommand: (...args) => redis.call(...args)
  }),
  handler: (req, res) => {
    res.status(429).json({
      error: 'Too many requests. Please wait before trying again.',
      retryAfter: Math.ceil(req.rateLimit.resetTime / 1000)
    });
  }
});

// Shape — general API limiter (by user ID for authenticated routes)
export const apiLimiter = rateLimit({
  windowMs: 60 * 1000,  // 1 minute
  max: 100,
  keyGenerator: (req) => req.user?.id || req.ip,
  // ...
});
```

---

## Apply the limiters

In `auth.routes.js`, apply `authLimiter` to the sensitive routes:

```js
router.post('/register', authLimiter, validate(registerSchema), registerHandler);
router.post('/login', authLimiter, validate(loginSchema), loginHandler);
router.post('/forgot-password', authLimiter, forgotPasswordHandler);
```

The limiter runs before validation — there's no point validating a request that's already over the limit.

In `app.js`, apply `apiLimiter` globally to all routes under `/api` except `/api/webhooks`:

```js
app.use('/api', apiLimiter);
app.use('/api/webhooks', webhookRouter);  // mount webhooks before the limiter, or exclude them
```

---

## Testing rate limits

```bash
# Send 11 login requests rapidly from the same IP — the 11th should receive 429
for i in $(seq 1 11); do
  curl -s -o /dev/null -w "%{http_code}\n" -X POST http://localhost:3000/api/auth/login \
    -H "Content-Type: application/json" \
    -d '{"email":"test@example.com","password":"wrong"}'
done
```

The first 10 should return 401 (wrong password). The 11th should return 429.

Check the response headers on successful requests:

```
RateLimit-Limit: 10
RateLimit-Remaining: 8
RateLimit-Reset: 1700001800
```

These tell the client how many requests they have left and when the window resets. A good client uses these headers to back off gracefully.

✅ The Stripe webhook endpoint does not rate-limit (test by sending more than 100 webhook events — they should all succeed).

---

## Definition of Done

- [ ] `POST /auth/login` returns 429 after 10 requests within 15 minutes from the same IP
- [ ] The 429 response includes a human-readable error message and a `retryAfter` value
- [ ] The rate limit counts are stored in Redis (restart the server — the counts survive the restart)
- [ ] The Stripe webhook endpoint is not rate-limited
- [ ] Successful auth requests include `RateLimit-*` headers showing the current window state

Write in `learning-log/13-rate-limiting.md`:

1. What attack does auth rate limiting prevent? Walk through the attack with and without the limiter.
2. Why use Redis as the rate limiter's store rather than in-memory?
3. Why are webhook endpoints excluded from rate limiting?
