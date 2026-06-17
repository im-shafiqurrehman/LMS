# Caching and Performance

The catalogue page is the most-read endpoint in EduFlow. Every visitor — logged in or not — hits it. It also changes infrequently: a course does not get published every second. This is the textbook use case for caching.

---

## What is a cache and why it matters here

A cache is a fast, temporary store of expensive computation results. Instead of running a MongoDB query on every catalogue request, you run it once, store the result in Redis (an in-memory store), and return the cached result on subsequent requests.

The trade-off: a cache may return **stale data** — results that were true a moment ago but have since changed. For a course catalogue, stale data for 5 minutes is acceptable. For a bank balance, it is not. Know your tolerance.

---

## Redis setup

Install the Redis client:

```
npm install ioredis
```

Create `src/lib/redis.js`:

```js
// src/lib/redis.js — shape only
import Redis from 'ioredis';
import { config } from '../config/index.js';

export const redis = new Redis(config.redisUrl);

redis.on('error', (err) => console.error('Redis error:', err));
```

---

## The cache-aside pattern

The pattern for caching a read operation:

1. Check the cache first: `const cached = await redis.get(cacheKey)`.
2. If the cache has a value, return it immediately — no database query.
3. If not, query the database, store the result in the cache with a TTL (time-to-live), and return it.

```js
// In catalogue service — shape only
const cacheKey = `catalogue:${JSON.stringify(filters)}`;

const cached = await redis.get(cacheKey);
if (cached) return JSON.parse(cached);

const courses = await Course.find(...).populate(...);

await redis.set(cacheKey, JSON.stringify(courses), 'EX', 300); // 5-minute TTL
return courses;
```

---

## Cache invalidation

A 5-minute TTL means stale data for up to 5 minutes. For most catalogue changes this is acceptable. But when a new course is published or an existing course is updated, you may want to clear the cache immediately.

Invalidation strategy: when a course's status changes to `published`, delete all catalogue cache keys:

```js
// After a course is published — shape only
const keys = await redis.keys('catalogue:*');
if (keys.length) await redis.del(keys);
```

✅ Use a namespace prefix (`catalogue:*`) so you can invalidate all catalogue variants at once.
❌ Do not skip invalidation for status changes — a newly published course should appear in the catalogue immediately, not after 5 minutes.

> **Interesting to read.** Facebook at its peak ran Memcached (similar to Redis) with hundreds of servers holding terabytes of cached data. Their 2013 paper "Scaling Memcache at Facebook" describes the challenges of cache invalidation at that scale — search it when you deploy.

---

## Definition of Done

- [ ] A catalogue request whose results are cached returns the cached value (verify by checking Redis with `redis-cli keys "catalogue:*"`)
- [ ] A catalogue request after cache expiry queries the database and refreshes the cache
- [ ] Publishing a course clears the catalogue cache
- [ ] Redis connection errors do not crash the server — the service falls back to the database

Write in `learning-log/11-caching.md`:

1. What is cache staleness, and how did you handle it in EduFlow?
2. What is the cache-aside pattern? Draw the decision flow.
3. Why is deleting `catalogue:*` keys on publish the right invalidation strategy here?
