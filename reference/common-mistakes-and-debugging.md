# Common Mistakes and How to Debug Them

This is the guide you read when things break. Every mistake here is real — engineers hit these constantly. The pattern for each: what the symptom looks like, what causes it, and the exact fix.

---

## Authentication issues

### "jwt malformed" or "invalid signature" on every request

**Symptom:** Every authenticated request returns 401, even with a freshly issued token.

**Causes (check in order):**
1. You are verifying the access token with `JWT_REFRESH_SECRET` instead of `JWT_SECRET` (or vice versa). Check the `authenticate.js` middleware — it must use `JWT_SECRET`.
2. The `JWT_SECRET` in your `.env` has a trailing space or newline. Open the file in a hex editor or run `cat -A .env | grep JWT_SECRET` to check for `^M` or trailing characters.
3. You are passing the token incorrectly in the `Authorization` header. The format must be exactly `Bearer <token>` — no extra quotes, no `"Bearer: token"`.

**Fix:** Log `process.env.JWT_SECRET.length` at startup and verify it matches the length of the string you put in `.env`.

---

### "TokenExpiredError" during development

**Symptom:** You log in, immediately make a request, and get "token expired."

**Cause:** `JWT_ACCESS_EXPIRY` is set to something very short (e.g. `1s` from a previous debug session) and you forgot to reset it.

**Fix:** Set `JWT_ACCESS_EXPIRY=15m` in `.env` and restart the server. Log the expiry on startup.

---

### Refresh token returns 401 immediately after login

**Symptom:** You log in, get a refresh token, immediately call `/auth/refresh`, and get 401.

**Cause:** The refresh token was not stored in the database. Check `auth.service.js` — the `generateTokenPair` function must create a `RefreshToken` record in Prisma after signing the token.

**Fix:** Add a `SELECT * FROM refresh_tokens ORDER BY created_at DESC LIMIT 5;` query right after login. If the table is empty, the insert is missing.

---

## Database issues

### "Can't reach database server at localhost:27017"

**Symptom:** Server fails to start with a Prisma connection error.

**Cause:** MongoDB is not running.

**Fix:** `docker compose up -d` and then verify with `docker compose ps` — the mongodb container should show `healthy`.

---

### "The table `main.users` does not exist" (SQLite message) or "relation does not exist"

**Symptom:** Prisma queries fail with a missing table error.

**Cause:** Migrations have not been run. The schema file exists but the database doesn't reflect it.

**Fix:** `npx prisma migrate dev` (development) or `npx prisma migrate deploy` (production). Check `prisma/migrations/` — if it's empty, you haven't run any migrations yet.

---

### "Unique constraint failed on the fields: (`email`)" returns 500

**Symptom:** Registering with a duplicate email returns an internal server error instead of 409.

**Cause:** The Prisma unique constraint violation (`P2002`) is not handled. It's thrown as an unhandled error that reaches the generic 500 handler.

**Fix:** In `errorHandler.js`, add a check for `err.code === 'P2002'` and return 409. Or catch the error in `auth.service.js` and throw an `AppError(409)`.

---

### Migration fails: "column already exists"

**Symptom:** `npx prisma migrate dev` fails partway through.

**Cause:** A previous migration ran partially (the script crashed partway) and left some columns in the database. The new migration tries to create the same columns.

**Fix:** Check `prisma/migrations/` for an incomplete migration folder (often missing the `migration.sql` content or having an error in it). Remove it. Then run `npx prisma migrate resolve --rolled-back <migration-name>` to tell Prisma the migration failed, and try again.

---

## Authorization issues

### Instructor can access another instructor's course (IDOR)

**Symptom:** `PUT /api/instructor/courses/:id` succeeds even when `:id` belongs to a different instructor.

**Cause:** The ownership check is missing in the service function. The route only checks the role (`INSTRUCTOR`) but not whether `course.instructorId === req.user.id`.

**Fix:** In `instructor.service.js`, every function that operates on a specific course must:
```js
const course = await prisma.course.findUnique({ where: { id } });
if (!course || course.instructorId !== requestingUserId) {
  throw new AppError('Course not found.', 404);
}
```

---

### Admin routes are accessible by instructors

**Symptom:** An instructor can call `GET /api/admin/metrics` and get a 200.

**Cause:** The `authorize('ADMIN')` middleware is not applied at the router level. It may be applied to some routes but not all.

**Fix:** In `admin.routes.js`:
```js
// Apply to ALL routes in this router
adminRouter.use(authenticate, authorize('ADMIN'));
// Then add routes below this line
```

---

## Stripe / payment issues

### Webhook returns 400: "No signatures found matching the expected signature"

**Symptom:** Stripe webhook events fail immediately with a signature error.

**Cause (most common):** The webhook route is being processed by `express.json()` before the raw body is read. Stripe's signature verification requires the raw, unparsed body.

**Fix:** In `app.js`, mount the webhook router **before** `express.json()`:
```js
// CORRECT order:
app.use('/api/webhooks', stripeWebhookRouter);  // raw body
app.use(express.json());                         // parsed body for everything else
```

---

### Webhook receives events but enrollment is not created

**Symptom:** Stripe sends `payment_intent.succeeded`, your webhook logs it, but no `Enrollment` row appears.

**Cause:** `studentId` or `courseId` is missing from `paymentIntent.metadata`. They must be passed when creating the PaymentIntent in `/api/enrollments/checkout`.

**Fix:** Check the PaymentIntent in your Stripe dashboard → Events → look at the raw event data. Verify the `metadata` field contains `studentId` and `courseId`. If not, fix the `checkout` handler.

---

## Caching issues

### Catalogue shows stale data after a course is published

**Symptom:** An admin approves a course, but the catalogue still doesn't include it even after the approval endpoint confirms success.

**Cause:** Cache invalidation is missing in the admin approval service function.

**Fix:** In `admin.service.js`, after updating the course status to `PUBLISHED`:
```js
const keys = await redis.keys('catalogue:*');
if (keys.length > 0) await redis.del(...keys);
```

---

### Redis errors are crashing the server

**Symptom:** When Redis goes down (or Docker is stopped), every request returns 500.

**Cause:** Redis errors are not caught. Any `redis.get()` or `redis.set()` that fails throws an unhandled error.

**Fix:** Wrap every Redis call in try/catch and fall back gracefully:
```js
let cached = null;
try {
  cached = await redis.get(cacheKey);
} catch (err) {
  console.error('Redis get failed:', err.message); // log and continue
}

if (cached) return JSON.parse(cached);
// proceed with database query
```

---

## File upload issues

### "MulterError: Unexpected field"

**Symptom:** Video upload returns this error.

**Cause:** The frontend is sending the file under a different field name than Multer expects. Multer is configured to accept `upload.single('video')` but the form data uses a different key.

**Fix:** Either fix the frontend to use `video` as the field name, or change `upload.single('video')` to match whatever key the client is using.

---

### Video uploads succeed but `videoUrl` is null in the database

**Symptom:** The upload endpoint returns 200 but the lesson's `videoUrl` is not set.

**Cause:** The Cloudinary storage result is not being read correctly. The `secure_url` is in `req.file.path` when using `multer-storage-cloudinary`.

**Fix:** In the upload handler:
```js
const videoUrl = req.file.path;         // secure_url from Cloudinary
const videoPublicId = req.file.filename; // public_id from Cloudinary
```

---

## General debugging checklist

When something isn't working:

1. **Read the full error** — not just the first line. The bottom of a stack trace tells you more than the top.
2. **Check the terminal logs** — morgan logs every request; Prisma logs every query (if enabled). The answer is usually there.
3. **Query the database directly** — `psql $DATABASE_URL -c "SELECT * FROM users LIMIT 5;"`. Don't trust the application's view of the data; check the source of truth.
4. **Test the endpoint in isolation** — use `curl` or Postman with the exact request body, not through the frontend. Rule out frontend issues first.
5. **Add a `console.log` at the entry point of the function** — verify the function is even being called. If the log never appears, the route isn't reaching the function.
6. **Check the middleware order** — most auth and routing bugs are middleware order problems. Print the middleware stack if needed.
7. **Restart the server** — `nodemon` sometimes gets confused; a manual restart clarifies.
