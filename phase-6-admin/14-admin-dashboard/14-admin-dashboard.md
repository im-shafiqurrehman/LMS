# Admin Dashboard

Every platform needs an operator — someone who can approve courses, suspend bad actors, and see how the platform is actually doing. The admin dashboard is EduFlow's control panel. It's not customer-facing, so it doesn't need to be beautiful, but it must be correct and complete.

This chapter builds the backend for the admin dashboard: course approval workflow, user management, and platform metrics.

---

## Scaffold the admin module

```bash
mkdir -p src/modules/admin
touch src/modules/admin/admin.routes.js
touch src/modules/admin/admin.service.js
```

Mount at `/api/admin` in `app.js`. Every route in this router requires `authenticate` + `authorize('ADMIN')` — applied at the router level, not per-route. Adding authorization to each route individually is a mistake that eventually leaves one route unguarded.

```js
// admin.routes.js — apply auth at router level
adminRouter.use(authenticate, authorize('ADMIN'));
// All routes added below this line are automatically protected
```

---

## Course approval workflow

### List pending courses

```
GET /api/admin/courses/pending
```

Returns all courses with `status = 'PENDING'`. Include instructor name, category, lesson count, and submission date (`updatedAt` when status last changed — you may need to track this explicitly or use `updatedAt` as a proxy).

Paginated (offset pagination is fine here — admins aren't scrolling infinite feeds).

### Approve a course

```
PUT /api/admin/courses/:id/approve
```

The service must:

1. Find the course, verify it's `PENDING`
2. Transition to `PUBLISHED`
3. Send the "course approved" email to the instructor
4. Invalidate the catalogue cache (the new course must appear)
5. Return the updated course

Return 409 if the course isn't `PENDING` — an already-published course shouldn't be accidentally re-published.

### Reject a course

```
PUT /api/admin/courses/:id/reject
```

Request body:

```json
{ "reason": "Video quality is too low. Please re-record with better audio." }
```

The service must:

1. Find the course, verify it's `PENDING` or `PUBLISHED`
2. Transition to `REJECTED`
3. Store the rejection reason (add a `rejectionReason String?` field to the `Course` model if not already present)
4. Send the "course rejected" email to the instructor with the reason included
5. Invalidate the catalogue cache if the course was `PUBLISHED`

✅ An instructor can then edit their `REJECTED` course and resubmit it (transition back to `DRAFT`, edit, re-submit to `PENDING`). Verify this flow end-to-end.

---

## User management

### List users

```
GET /api/admin/users
```

Query params: `role` (filter by role), `search` (partial email match), `page`, `limit`.

Returns user list with `id`, `email`, `role`, `emailVerifiedAt`, `createdAt`, enrollment count.

### Deactivate a user

```
PUT /api/admin/users/:id/deactivate
```

Add `deactivatedAt DateTime?` to the `User` model. When set, the `authenticate` middleware must reject the user's tokens with 403 ("Your account has been deactivated").

Add the deactivation check in `authenticate.js`:

```js
// After jwt.verify() succeeds, look up the user's deactivatedAt
// If deactivatedAt is not null, return 403
```

This adds a database lookup to every authenticated request — an important trade-off. Discuss this in your learning log: why is a lookup required here that isn't required for normal auth?

✅ After deactivating a user, their existing access token must be rejected on the next request.

---

## Platform metrics

```
GET /api/admin/metrics
```

Returns:

```json
{
  "totalUsers": 1420,
  "totalStudents": 1280,
  "totalInstructors": 140,
  "totalCourses": 85,
  "publishedCourses": 62,
  "totalEnrollments": 4320,
  "revenueTotal": "189430.00",
  "revenueThisMonth": "12400.00",
  "topCoursesByEnrollment": [
    { "id": "uuid", "title": "Mastering React", "enrollmentCount": 342 }
  ]
}
```

Use `prisma.$queryRaw` for the aggregations — multiple `COUNT`, `SUM`, and `GROUP BY` queries are cleaner in raw SQL than Prisma's query API for this use case.

Cache this endpoint aggressively — metrics don't change per-second and are expensive to compute:

```js
redis.set('admin:metrics', JSON.stringify(result), 'EX', 60); // 1 minute TTL
```

---

## Definition of Done

- [ ] `PUT /api/admin/courses/:id/approve` transitions a PENDING course to PUBLISHED, sends email, and the course appears in the catalogue on the next request
- [ ] `PUT /api/admin/courses/:id/reject` transitions the course to REJECTED with a stored reason and sends the rejection email
- [ ] An instructor can edit a REJECTED course and resubmit it
- [ ] A deactivated user's access token is rejected with 403 on the next authenticated request
- [ ] `GET /api/admin/metrics` returns correct aggregated figures (verify against direct SQL queries)
- [ ] All admin routes return 403 to non-admin users

Write in `learning-log/14-admin-dashboard.md`:

1. Why is authorization applied at the router level rather than per-route in the admin module?
2. The deactivation check adds a database lookup to every authenticated request. What trade-off does this create, and what would a token blocklist in Redis solve instead?
3. Why is the metrics endpoint cached for 60 seconds rather than a longer TTL?
