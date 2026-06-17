# Rate Limiting

Auth endpoints are the most attacked surface of any web API. A botnet can attempt thousands of login requests per second, trying leaked username/password pairs (credential stuffing) or common passwords (brute force). Without rate limiting, the only constraint is your server's capacity.

---

## What rate limiting does

Rate limiting tracks how many requests a client has made in a time window and rejects requests above a threshold with `429 Too Many Requests`. For EduFlow's auth endpoints, a reasonable limit is 10 requests per 15 minutes per IP address.

---

## Redis-backed rate limiting with express-rate-limit

Install the packages:

```
npm install express-rate-limit rate-limit-redis
```

Create a rate limiter configuration in `src/middleware/rateLimiter.js`:

```js
// src/middleware/rateLimiter.js — shape only
import rateLimit from 'express-rate-limit';
import { RedisStore } from 'rate-limit-redis';
import { redis } from '../lib/redis.js';

export const authRateLimiter = rateLimit({
  windowMs: 15 * 60 * 1000,   // 15 minutes
  max: 10,                     // 10 requests per window per IP
  standardHeaders: true,       // Return rate limit info in headers
  legacyHeaders: false,
  store: new RedisStore({
    sendCommand: (...args) => redis.call(...args),
  }),
  message: {
    error: 'Too many attempts. Please wait 15 minutes and try again.',
  },
});
```

Apply it to auth routes:

```js
// In auth.routes.js
import { authRateLimiter } from '../../middleware/rateLimiter.js';

authRouter.post('/register',       authRateLimiter, registerHandler);
authRouter.post('/login',          authRateLimiter, loginHandler);
authRouter.post('/forgot-password', authRateLimiter, forgotPasswordHandler);
```

## Why Redis-backed, not in-memory

`express-rate-limit` can store rate limit counters in memory by default. In a single-server development environment, this works. In production with multiple server instances, each instance has its own in-memory counter — a client could bypass the limit by routing requests across instances.

Redis is a shared store that all instances read from and write to. The counter is accurate regardless of how many server instances are running.

---

## Definition of Done

- [ ] 11 consecutive `POST /api/auth/login` requests from the same IP within 15 minutes — the 11th returns 429
- [ ] The 429 response includes a clear, human-readable error message
- [ ] The rate limiter uses Redis as its store (verify by checking Redis keys with `redis-cli keys "rl:*"`)
- [ ] Catalogue and lesson routes are not rate limited

Write in `learning-log/13-rate-limiting.md`:

1. What is a credential stuffing attack and how does rate limiting mitigate it?
2. Why is in-memory rate limiting insufficient for a horizontally scaled application?
