# Key Engineering Decisions

Every significant technology choice and design decision in EduFlow, with the reasoning behind it. This document belongs in your README and is the source material for your portfolio write-up. It is also your primary defence in the viva.

Read this after building. It should read like things you already know — because you built them. If any section surprises you, that chapter's learning log is incomplete.

---

## 1. MongoDB over PostgreSQL

**The decision:** Use a document database (MongoDB) rather than a relational database (PostgreSQL).

**Why:** EduFlow's data is naturally document-shaped. Courses, lessons, and progress records have nested structures that map cleanly to JSON documents. MongoDB's schema flexibility allows the data model to evolve without migrations — adding a new field to a course doesn't require a schema change across the entire table.

MongoDB with Prisma provides the best of both worlds: the flexibility of a document database with the type safety and validation of an ORM. Prisma enforces relationships at the application level, ensuring referential integrity while allowing the schema to evolve.

The query patterns also favour MongoDB for this use case. "Give me all courses with their lesson counts and enrollment counts" is a single aggregation pipeline. MongoDB Atlas Search provides powerful full-text search capabilities without needing a separate search service.

**What PostgreSQL would have cost:** Rigid schema that requires migrations for every field change. Complex JOINs for nested data. Less natural fit for document-shaped data. Scaling horizontally requires more complex sharding strategies.

**When PostgreSQL wins:** When the data has strict, complex relationships with frequent cross-table joins, or when ACID transactions across multiple tables are critical. Financial systems with complex transactional requirements are better suited to PostgreSQL.

---

## 2. JWT with two tokens over sessions

**The decision:** Use JWT access tokens (15-minute expiry) and JWT refresh tokens (7-day expiry, stored hashed in the database) rather than server-side sessions.

**Why:** JWTs are stateless — any server in the fleet can verify any token without a shared state store. This makes horizontal scaling clean: add servers, point the load balancer at them, and they all work immediately. A session system requires all servers to share the session store (typically Redis), which is an extra infrastructure dependency on the critical auth path.

The two-token pattern resolves JWT's main weakness — non-revocability — by keeping one of the two tokens in the database. The access token is short-lived (15 minutes) so a leaked one expires quickly. The refresh token is long-lived but stored, so it can be revoked on logout by deleting the record.

**What sessions would have cost:** A database or Redis lookup on every authenticated request (to validate the session). In a single-server setup, this is fine; at scale, the session store becomes a bottleneck and a single point of failure.

**The trade-off we accepted:** An active access token cannot be immediately revoked (without a blocklist). If a user's account is deactivated, their current access token remains valid for up to 15 minutes. EduFlow mitigates this with the `deactivatedAt` check in the `authenticate` middleware — a database lookup on every authenticated request, for this specific case. This is a deliberate choice: the lookup cost is accepted because immediate deactivation matters operationally.

---

## 3. Cloudinary for video storage, not MongoDB or the filesystem

**The decision:** Store video files in Cloudinary (a specialised media platform) rather than in the database as binary data or on the server filesystem.

**Why:** Video files are large — a single lesson can be 1–5GB. Storing binary data in MongoDB bloats the database, makes backups slow and enormous, and serves files at the database server's network bandwidth rather than a CDN. Storing on the filesystem works for a single server but breaks immediately when you add a second server (they don't share a filesystem).

Cloudinary solves all of this: it stores files permanently, serves them from a global CDN (fast everywhere), handles transcoding (different resolutions, formats), and provides signed URL generation for access control. The per-request cost is zero — you pay for storage and bandwidth at Cloudinary's scale, not your server's.

**What MongoDB storage would have cost:** Bloated database backups. Slow restores. Every video request hits the database connection pool. No CDN — full-speed delivery from a single VPS for every viewer globally.

**What filesystem storage would have cost:** Files lost on server wipe. Files unavailable on horizontal scale. No CDN. Backup complexity.

---

## 4. Stripe webhooks drive enrollment, not the payment redirect

**The decision:** Enrollment is created when Stripe sends a `payment_intent.succeeded` webhook, not when the student's browser is redirected back to the success page.

**Why:** A browser redirect is unreliable. The tab can close. The connection can drop. The student can navigate away. If enrollment depended on the redirect completing, every dropped connection would be a lost enrollment — a student who paid but can't access their course. Support tickets, refund requests, and lost trust follow.

Stripe's webhook is server-to-server. It retries until it receives a 200 response. The student's browser state is irrelevant. Enrollment happens on the server, confirmed by Stripe, regardless of what the client does.

**The idempotency requirement:** Because Stripe retries webhooks, the same event can arrive more than once. The enrollment handler uses `upsert` with a `@@unique([studentId, courseId])` constraint — if the enrollment already exists, the upsert is a no-op. This makes the handler safe to call multiple times for the same payment.

---

## 5. Cursor pagination over offset for the catalogue

**The decision:** The course catalogue uses cursor-based pagination rather than `LIMIT/OFFSET`.

**Why:** `OFFSET N` in MongoDB is not a skip — MongoDB must read and discard the first N documents before returning your page. At large offsets, this is a collection scan. At 100,000 courses and page 50 (offset 1000), every catalogue request scans and discards 1,000 documents.

Cursor pagination avoids this: instead of "skip N documents," it says "give me documents after this specific point." MongoDB uses the index to jump directly to that point. Query performance is constant regardless of which page you're on.

The cursor-based response also handles real-time inserts correctly. With offset, if a course is published while a user is browsing page 3, all subsequent pages shift by one — the user sees a duplicate or skips a course. A cursor is anchored to a specific record; new inserts don't affect it.

**The trade-off:** Cursor pagination cannot jump to an arbitrary page ("go to page 7"). It only supports forward traversal. For EduFlow's infinite-scroll catalogue, this is acceptable. For an admin table where "jump to page 47" is useful, offset is the better choice — which is why the admin user list uses offset.

---

## 6. Redis for caching with explicit invalidation

**The decision:** Cache the catalogue endpoint in Redis with a 5-minute TTL, and explicitly invalidate the cache when a course is published or unpublished.

**Why:** The catalogue is the highest-traffic endpoint in the system — hit by every visitor, logged in or not, multiple times per session. It reads the same data repeatedly: published courses, their categories, their instructor names. Hitting MongoDB on every request is wasteful when the data changes infrequently.

Redis stores the result in memory — reads take microseconds vs milliseconds for a database query. For a popular page, this reduces database load by 90%+ and makes the page noticeably faster.

**The invalidation strategy:** Both an explicit invalidation (when a course status changes) and a TTL (5 minutes) protect against stale data. The explicit invalidation ensures the change is visible immediately. The TTL is a safety net — if a code path forgets to invalidate (a future bug), the cache repairs itself within 5 minutes rather than serving stale data indefinitely.

**What a pure TTL strategy would have cost:** A published course might not appear in the catalogue for up to 5 minutes after approval. For an admin who just approved a course and immediately checks the catalogue, this would look like a bug.

---

## 7. Signed, expiring Cloudinary URLs for video access

**The decision:** Video lesson access is controlled via signed, short-lived Cloudinary URLs rather than by returning the raw video URL from the database.

**Why:** A permanent Cloudinary URL, once returned, is shareable. A student enrolls, gets the URL, posts it on a forum. Anyone can watch the lesson for free, forever. The enrollment check becomes irrelevant — the URL itself is the access control, and permanent URLs have no access control.

Signed URLs expire (2 hours in EduFlow). After expiry, the URL is invalid — Cloudinary rejects requests against it. The only way to get a fresh signed URL is through EduFlow's API, which checks enrollment first. Link-sharing becomes harmless: the link expires in 2 hours.

**The trade-off:** A signed URL is generated on every lesson request (a Cloudinary SDK call). This is fast (local computation, not a network call) but adds a small amount of latency. The security guarantee is worth the cost.

---

## 8. pdfkit for certificate generation, stored rather than generated on demand

**The decision:** Generate the certificate PDF once at course completion, upload it to Cloudinary, and store the URL. Return a signed URL to the download request rather than generating the PDF on every download.

**Why:** PDF generation with pdfkit takes 50–200ms and is CPU-intensive. Generating the PDF on every download request would block the Node.js event loop on every certificate download — a poor user experience and a resource cost that grows with download frequency.

Generating once and storing means the first download pays the generation cost; every subsequent download is just a signed URL lookup (microseconds). For a certificate that a student might download dozens of times, the economics are clear.

**What on-demand generation would have cost:** CPU spike on every certificate download. Slower response times. Node.js event loop blockage (PDFKit is synchronous).

---

## 9. Rate limiting on auth endpoints with Redis store

**The decision:** Rate limit `POST /auth/login`, `POST /auth/register`, and `POST /auth/forgot-password` to 10 requests per 15 minutes per IP, using Redis as the count store.

**Why:** Auth endpoints are the most obvious attack surface. Without rate limiting, an attacker can brute-force passwords at hundreds of thousands of guesses per hour. With a 15-minute window of 10 attempts, an attacker trying a password list against a single account gets 10 guesses per 15 minutes — at that rate, even a weak password is safe from an unsophisticated attack.

Redis is used as the store rather than in-memory because the rate limit count must be shared across all servers. A rate limiter that counts in memory on each server gives each server its own window — a 3-server deployment effectively triples the allowed request rate.

**The trade-off:** Redis adds a network hop to every auth request to increment the counter. This is ~1ms at local Redis speeds. For an auth endpoint that runs bcrypt (300ms), this overhead is imperceptible.
