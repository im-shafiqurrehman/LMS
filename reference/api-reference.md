# EduFlow — Complete API Reference

Every endpoint in EduFlow. Use this as your build checklist and as the API reference section of your README.

Legend: 🔓 Public · 🔐 Authenticated (any role) · 👨‍🎓 Student · 👩‍🏫 Instructor · 🛡 Admin

---

## Auth — `/api/auth`

| Method | Path | Auth | Description |
|--------|------|------|-------------|
| `POST` | `/api/auth/register` | 🔓 | Create a new account. Sends OTP verification email. |
| `POST` | `/api/auth/verify-email` | 🔓 | Confirm the OTP. Marks account verified. |
| `POST` | `/api/auth/login` | 🔓 | Validate credentials. Returns `{ accessToken, refreshToken }`. |
| `POST` | `/api/auth/refresh` | 🔓 | Exchange a valid refresh token for a new access token. |
| `POST` | `/api/auth/logout` | 🔐 | Revoke the refresh token. Requires Bearer access token. |
| `POST` | `/api/auth/forgot-password` | 🔓 | Send a password-reset OTP to the email. |
| `POST` | `/api/auth/reset-password` | 🔓 | Verify OTP + update password hash. |
| `GET`  | `/api/auth/me` | 🔐 | Return the decoded token payload (`sub`, `role`). |

---

## Courses — Public Catalogue — `/api/courses`

| Method | Path | Auth | Description |
|--------|------|------|-------------|
| `GET` | `/api/courses` | 🔓 | Paginated catalogue of published courses. Query params: `category`, `minPrice`, `maxPrice`, `cursor`, `limit`. |
| `GET` | `/api/courses/search` | 🔓 | Full-text search over published courses. Query params: `q`, `limit`, `cursor`. |
| `GET` | `/api/courses/:id` | 🔓 | Single published course detail with instructor and category. |
| `GET` | `/api/courses/:id/lessons` | 🔓 | List lesson metadata for a published course. `isCompleted` populated for enrolled students. |

---

## Instructor — Course Management — `/api/instructor`

All routes require `authenticate` + `authorize('INSTRUCTOR')`.

| Method | Path | Auth | Description |
|--------|------|------|-------------|
| `GET` | `/api/instructor/courses` | 👩‍🏫 | List the instructor's own courses (all statuses). Includes lesson + enrollment counts. |
| `POST` | `/api/instructor/courses` | 👩‍🏫 | Create a new course. Status defaults to `DRAFT`. |
| `GET` | `/api/instructor/courses/:id` | 👩‍🏫 | Get a single course with lessons. Ownership enforced (404 if not owned). |
| `PUT` | `/api/instructor/courses/:id` | 👩‍🏫 | Update course fields. Blocked if status is `PUBLISHED` or `PENDING`. |
| `DELETE` | `/api/instructor/courses/:id` | 👩‍🏫 | Delete a course. Only `DRAFT` or `REJECTED` courses can be deleted. |
| `POST` | `/api/instructor/courses/:id/submit` | 👩‍🏫 | Submit for admin review (`DRAFT → PENDING`). Validates minimum requirements (3+ lessons, cover image, description length). |
| `POST` | `/api/instructor/courses/:id/lessons` | 👩‍🏫 | Add a lesson to the course. Position auto-incremented. |
| `PUT` | `/api/instructor/courses/:id/lessons/reorder` | 👩‍🏫 | Reorder lessons. Body: `{ lessonIds: [uuid, uuid, ...] }`. Uses a transaction. |
| `PUT` | `/api/instructor/courses/:id/lessons/:lessonId` | 👩‍🏫 | Update lesson metadata (title, isFree). |
| `DELETE` | `/api/instructor/courses/:id/lessons/:lessonId` | 👩‍🏫 | Delete a lesson. Re-numbers remaining positions. |
| `POST` | `/api/instructor/courses/:id/lessons/:lessonId/video` | 👩‍🏫 | Upload a video file (multipart/form-data). Sets `videoUrl`, `videoPublicId`, and updates `totalDuration`. |
| `GET` | `/api/instructor/dashboard` | 👩‍🏫 | Enrollment counts and lesson completion rates per course. |

---

## Enrollments — `/api/enrollments`

| Method | Path | Auth | Description |
|--------|------|------|-------------|
| `POST` | `/api/enrollments/checkout` | 👨‍🎓 | Create a Stripe PaymentIntent. Returns `{ clientSecret, paymentIntentId }`. |
| `GET` | `/api/enrollments/my` | 👨‍🎓 | List student's enrollments with progress and certificate status. |

---

## Lessons — `/api/lessons`

| Method | Path | Auth | Description |
|--------|------|------|-------------|
| `GET` | `/api/lessons/:id` | 🔐 | Get lesson detail. Returns signed Cloudinary video URL if enrolled (or `isFree`). Returns 403 if unenrolled on a paid lesson. |
| `POST` | `/api/lessons/:id/complete` | 🔐 | Mark a lesson complete. Creates a `ProgressRecord`. Triggers certificate check. |

---

## Certificates — `/api/certificates`

| Method | Path | Auth | Description |
|--------|------|------|-------------|
| `GET` | `/api/certificates/enrollment/:enrollmentId` | 👨‍🎓 | Get certificate for an enrollment. Returns signed PDF download URL. Ownership enforced. |

---

## Webhooks — `/api/webhooks`

No authentication middleware. Stripe signature verified inside the handler.

| Method | Path | Auth | Description |
|--------|------|------|-------------|
| `POST` | `/api/webhooks/stripe` | — | Receives Stripe events. Handles `payment_intent.succeeded` to enroll students. |

---

## Admin — `/api/admin`

All routes require `authenticate` + `authorize('ADMIN')`.

| Method | Path | Auth | Description |
|--------|------|------|-------------|
| `GET` | `/api/admin/metrics` | 🛡 | Platform-wide metrics: user counts, enrollment totals, revenue, top courses. Cached 60s. |
| `GET` | `/api/admin/courses/pending` | 🛡 | Paginated list of courses awaiting review (`status = PENDING`). |
| `GET` | `/api/admin/courses` | 🛡 | All courses with filter by status. |
| `PUT` | `/api/admin/courses/:id/approve` | 🛡 | Approve a pending course. Transitions to `PUBLISHED`. Sends email. Invalidates catalogue cache. |
| `PUT` | `/api/admin/courses/:id/reject` | 🛡 | Reject a course. Body: `{ reason: string }`. Transitions to `REJECTED`. Sends email. |
| `GET` | `/api/admin/users` | 🛡 | List users with optional filters: `role`, `search` (email), `page`, `limit`. |
| `PUT` | `/api/admin/users/:id/deactivate` | 🛡 | Deactivate a user account. Sets `deactivatedAt`. Subsequent requests from that user return 403. |
| `PUT` | `/api/admin/users/:id/reactivate` | 🛡 | Reactivate a deactivated user. Clears `deactivatedAt`. |

---

## System

| Method | Path | Auth | Description |
|--------|------|------|-------------|
| `GET` | `/health` | 🔓 | Sanity check. Returns `{ "status": "ok", "timestamp": "..." }`. |

---

## Common HTTP status codes used in EduFlow

| Code | Meaning | When EduFlow uses it |
|------|---------|----------------------|
| `200` | OK | Successful GET, PUT, POST with no resource creation |
| `201` | Created | Successful POST that creates a new resource (register, create course, create lesson) |
| `204` | No Content | Successful DELETE |
| `400` | Bad Request | Validation failure (Zod error), business rule violation (e.g. "must have 3+ lessons before submitting") |
| `401` | Unauthorized | Missing or invalid access token |
| `403` | Forbidden | Valid token, wrong role; or deactivated account; or accessing a paid lesson without enrollment |
| `404` | Not Found | Resource doesn't exist, or exists but is not owned by the requester (ownership checks return 404 not 403) |
| `409` | Conflict | Duplicate email on register; already enrolled; course already published |
| `422` | Unprocessable Entity | (not used — EduFlow uses 400 for all validation errors) |
| `429` | Too Many Requests | Rate limit exceeded |
| `500` | Internal Server Error | Unhandled exception — should never reach production (use the error handler middleware) |

---

## Request/response conventions

**All request bodies are JSON** (`Content-Type: application/json`) unless the endpoint accepts file uploads, which use `multipart/form-data`.

**All responses are JSON**.

**Authentication** is via the `Authorization` header:
```
Authorization: Bearer <accessToken>
```

**Validation errors** (400) return:
```json
{
  "error": "Validation failed",
  "details": [
    { "field": "email", "message": "Invalid email format" },
    { "field": "password", "message": "Must be at least 8 characters" }
  ]
}
```

**All other errors** return:
```json
{
  "error": "Human-readable error message."
}
```

**Pagination responses** (catalogue, search, admin lists) return:
```json
{
  "data": [...],
  "nextCursor": "uuid-or-null",
  "hasMore": true
}
```

Or for offset-paginated admin endpoints:
```json
{
  "data": [...],
  "total": 142,
  "page": 2,
  "limit": 20,
  "totalPages": 8
}
```
