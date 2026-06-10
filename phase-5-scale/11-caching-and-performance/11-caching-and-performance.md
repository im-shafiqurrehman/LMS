# Caching and Performance

The course catalogue is the most-read page in EduFlow. It's hit by every visitor — logged in or not — and it reads the same data over and over: published courses, their categories, their instructor names, their enrollment counts. Most of the time, this data hasn't changed. Yet every request hits the database and runs the full query.

Caching solves this by storing the result of an expensive operation and returning the stored result for subsequent identical requests. Done right, it makes a frequently-read page 10–50x faster. Done wrong, it serves stale data to users and is harder to debug than just not having a cache.

This chapter adds Redis caching to the catalogue endpoint and teaches you the discipline that makes caching correct: **invalidation**.

---

## What caching is and what it costs you

A cache is a faster storage layer that sits in front of slower storage. Redis is an in-memory key-value store — reads take microseconds vs milliseconds for a database query. For a catalogue page with millions of visits, the difference is enormous.

The cost: **cache coherence**. When a course is published, updated, or unpublished, the catalogue cache has old data. If you don't invalidate the cache at that moment, users see stale course listings. The rule is simple but often forgotten: **every write that changes cached data must also clear or update the cache.**

There is a second cost: **complexity**. A bug in the cache layer can silently serve wrong data. A missing invalidation means users see ghost courses. Treat cache logic with the same care as database logic — it affects correctness, not just performance.

> **Worth reading:** Phil Karlton famously said "there are only two hard things in computer science: cache invalidation and naming things." Search "cache invalidation strategies" — the Martin Fowler article on cache patterns is worth reading before you implement.

---

## Redis setup

You already have Redis running via Docker Compose. Install the client:

```bash
npm install ioredis
```

Configure in `src/lib/redis.js`:

```js
// Create an ioredis client using config.redisUrl
// Handle connection errors (log them; don't crash the server)
// Export as `redis`
```

The connection error handling is important: if Redis goes down, the catalogue should still work — it just falls back to hitting the database. Your caching code must handle Redis failures gracefully (try/catch around every Redis operation).

---

## Caching the catalogue

The cache key strategy for the catalogue must encode all the parameters that affect the result:

```
catalogue:category=frontend:minPrice=0:maxPrice=100:cursor=uuid:limit=20
```

Build the key from the sorted, serialised query parameters so different filter combinations get different cache entries, and the same parameters always produce the same key.

In `courses.service.js`, wrap the catalogue query with cache logic:

```js
// Cache-aside pattern — shape
async function getCatalogue(filters) {
  const cacheKey = buildCatalogueKey(filters);

  // 1. Try to read from cache
  const cached = await redis.get(cacheKey);
  if (cached) return JSON.parse(cached);

  // 2. Cache miss — query the database
  const result = await queryCatalogue(filters);

  // 3. Store in cache with a TTL (time to live)
  await redis.set(cacheKey, JSON.stringify(result), 'EX', 300); // 5 minutes

  return result;
}
```

A 5-minute TTL means stale data lives for at most 5 minutes even if invalidation fails. Choose the TTL based on how often the data changes and how stale is acceptable.

✅ Test: make the same catalogue request twice. The second should be noticeably faster (log timing in the service layer).
✅ Verify in Redis: `redis-cli keys "catalogue:*"` should show the cached key.

---

## Cache invalidation

Every time a course's status changes to or from `PUBLISHED`, the catalogue cache must be cleared. The cleanest approach: **invalidate by prefix**. All catalogue cache keys start with `catalogue:`. When a course is published or unpublished, delete every key matching `catalogue:*`.

```js
// Invalidate all catalogue cache entries
const keys = await redis.keys('catalogue:*');
if (keys.length > 0) {
  await redis.del(...keys);
}
```

Call this from:
- `POST /instructor/courses/:id/submit` (course submitted for review — pending courses don't appear in catalogue, so no invalidation needed here, but keep it for when rejection→publish cycles matter)
- Admin course approval (when a course transitions to `PUBLISHED`)
- Admin course rejection (when a `PUBLISHED` course is suspended)

✅ Test the invalidation: cache the catalogue, publish a new course through the admin route, then request the catalogue again. The new course must appear — it's not served from the stale cache.

---

## Fixing N+1 queries with `include`

You already learned the N+1 pattern in the catalogue chapter. Now audit the rest of your codebase. Run Prisma's query logging temporarily:

```js
const prisma = new PrismaClient({ log: ['query'] });
```

Then make requests to:
- `GET /api/instructor/courses` (the instructor's course list)
- `GET /api/enrollments/my` (the student's enrollment list)
- `GET /api/courses/:id/lessons` (lesson list with progress)

In each response, count the logged queries. More than 1–2 queries for a list endpoint is a red flag. For each N+1 you find, fix it with `include` or `select` in the Prisma query, then re-verify the query count drops to 1.

---

## Definition of Done

- [ ] `GET /api/courses` reads from Redis on cache hits (verify by checking Redis keys)
- [ ] A cache miss queries the database and stores the result with a 5-minute TTL
- [ ] Publishing or unpublishing a course invalidates the catalogue cache; the next request reflects the change
- [ ] Redis connection failure does not crash the server — the catalogue falls back to a database query
- [ ] No N+1 queries exist on the catalogue, instructor course list, or student enrollment list endpoints (verified with Prisma query logging)

Write in `learning-log/11-caching-and-performance.md`:

1. What is the cache-aside pattern? Walk through a cache hit and a cache miss step by step.
2. What is cache invalidation, and what goes wrong if you skip it?
3. What does TTL mean and why does it exist even when you have explicit invalidation?
4. What would happen if the Redis key prefix strategy used different key structures for the same request (e.g. sorting parameters differently) — what bug would that introduce?
