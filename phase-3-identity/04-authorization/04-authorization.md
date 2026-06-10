# Authorization — Roles and Data Isolation

Authentication answers "who are you?" Authorization answers "what are you allowed to do?" They are completely separate concerns, and confusing them is one of the most common security mistakes in web development.

At this point, EduFlow knows who every user is — the `authenticate` middleware proves it. But every authenticated user can reach every route. A student could theoretically call `DELETE /api/courses/123` and delete another instructor's course. An instructor could call the admin dashboard. These are authorization failures, and they are exploitable the moment the application is deployed.

This chapter draws the lines.

---

## The two layers of authorization

Authorization in EduFlow works in two layers, and you need both.

**Layer 1 — Role checks.** Some routes are only for certain roles. The admin dashboard is only for admins. Course creation is only for instructors. A role check is a middleware that reads `req.user.role` and rejects the request if the role doesn't match. It runs before the route handler.

**Layer 2 — Ownership checks.** Within a role, access is further scoped by ownership. An instructor can edit their own courses, not other instructors' courses. A student can see their own enrollment records, not other students'. Ownership is not a middleware — it's a check inside the service function, because you need the database to answer it.

Most authorization bugs live in Layer 2. Engineers add role checks and consider the job done. Then a logged-in instructor changes the `courseId` in the URL to another instructor's course ID and successfully edits it — because the route checked their role but not their ownership. This is called an **Insecure Direct Object Reference (IDOR)** and it's in the OWASP Top 10.

> **Worth reading:** Search "OWASP IDOR broken object level authorization" — the OWASP API Security Top 10 entry on Broken Object Level Authorization (BOLA) is worth fifteen minutes. It shows exactly how this attack pattern works in production APIs.

---

## Building the authorize middleware (Layer 1)

Create `src/middleware/authorize.js`. This is a **middleware factory** — a function that takes one or more allowed roles and returns a middleware function:

```js
// authorize.js — behaviour spec
// export function authorize(...roles)
// Returns a middleware that:
//   1. Reads req.user.role (set by authenticate)
//   2. If the role is in the allowed list — calls next()
//   3. If not — responds 403 Forbidden: { error: "You don't have permission to access this resource." }
```

Usage on a route (spec, not code):

```js
// Only admins can access this route
router.get('/admin/metrics', authenticate, authorize('ADMIN'), handler)

// Both admins and instructors can access this route
router.get('/courses/mine', authenticate, authorize('INSTRUCTOR', 'ADMIN'), handler)
```

`authorize` always runs after `authenticate` — it depends on `req.user` being set.

✅ Return 403 (Forbidden), not 401 (Unauthorized). 401 means "you're not logged in." 403 means "you're logged in, but you don't have access." The distinction matters for frontend error handling.

---

## Ownership checks (Layer 2)

Ownership checks live inside service functions, not middleware. Here is the pattern, using course editing as an example:

```js
// auth.service.js — pattern spec
async function updateCourse(courseId, updateData, requestingUserId) {
  const course = await prisma.course.findUnique({ where: { id: courseId } });

  if (!course) {
    throw { status: 404, message: "Course not found." };
  }

  // Ownership check — the course must belong to the requesting instructor
  if (course.instructorId !== requestingUserId) {
    // Return 404, not 403 — this leaks less information
    // A 403 tells the attacker the resource exists; a 404 doesn't
    throw { status: 404, message: "Course not found." };
  }

  return prisma.course.update({ where: { id: courseId }, data: updateData });
}
```

The deliberate choice of 404 over 403 on ownership failure deserves a moment. If an attacker is probing your API with different IDs, a 403 tells them "this resource exists, but it's not yours." A 404 tells them nothing. Returning 404 for resources the requester cannot access is the standard hardened pattern. Use it everywhere ownership matters.

✅ Always perform the ownership check inside the service function — never trust a `userId` or `instructorId` from the request body. Read the actual resource and compare against `req.user.id`.
❌ Don't put ownership logic in middleware — it requires a database query that depends on the route's parameters, which varies per route.

---

## Apply authorization to every route you've built

Go through every route in `auth.routes.js` and the routes you'll add in subsequent chapters. For each, decide:

- Is it public? (No middleware.)
- Does it require authentication? (Add `authenticate`.)
- Does it also require a specific role? (Add `authorize(role)`.)

The matrix for EduFlow's major routes:

| Route | Auth required | Role required |
|---|---|---|
| `POST /auth/register` | No | — |
| `POST /auth/login` | No | — |
| `GET /courses` | No | — |
| `POST /courses` | Yes | INSTRUCTOR |
| `PUT /courses/:id` | Yes | INSTRUCTOR (+ ownership check in service) |
| `DELETE /courses/:id` | Yes | INSTRUCTOR (+ ownership check in service) |
| `POST /courses/:id/enroll` | Yes | STUDENT |
| `GET /lessons/:id/video` | Yes | STUDENT (+ enrollment check in service) |
| `GET /admin/metrics` | Yes | ADMIN |
| `PUT /admin/courses/:id/approve` | Yes | ADMIN |

Build this matrix out fully as you add routes in the next chapters. Keep it updated — it becomes your access control document.

---

## Scaffold the enrollment check (preview)

You'll build enrollment access control fully in Chapter 8. But plan for it now: a student accessing a lesson needs two checks — are they authenticated? and are they enrolled in this course?

The enrollment check is **not** a role check (any `STUDENT` role passes the role check). It is a record-level ownership check: does a row exist in `enrollments` where `student_id = req.user.id` and `course_id = the course this lesson belongs to`?

Design this check as a service-layer function now — `isEnrolled(studentId, courseId)` returning a boolean — and wire it into the lesson access service when you get there. Designing the query now, even before the `Enrollment` table exists, is good practice: you think about the access pattern before you choose the index.

---

## Definition of Done

- [ ] `authorize('ADMIN')` middleware rejects a STUDENT with 403 and an INSTRUCTOR with 403 on an admin-only route
- [ ] `authorize('INSTRUCTOR')` rejects a STUDENT with 403
- [ ] A `PUT /courses/:id` request from a valid INSTRUCTOR but with another instructor's course ID returns 404 (not 403, not 200)
- [ ] `req.user` is never mutated inside a route handler — it is read-only
- [ ] No route relies on a `userId` from the request body for ownership — it always reads from `req.user.id`

Write your answers in `learning-log/04-authorization.md`:

1. What is the difference between authentication and authorization? Give a concrete example of each in EduFlow.
2. What is an IDOR vulnerability? Walk through a specific example of how it could be exploited in EduFlow if ownership checks were missing.
3. Why does the ownership check return 404 instead of 403 when a resource exists but doesn't belong to the requester?
4. Why does the `authorize` middleware run *after* `authenticate` and not before?
