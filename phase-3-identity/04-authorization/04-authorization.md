# Authorization — Roles and Data Isolation

Authentication answered: "Who are you?" Authorization answers: "What are you allowed to do?" They are different problems, and confusing them is the source of serious security bugs.

## The threat: IDOR (Insecure Direct Object Reference)

Here is the most common authorization mistake in web APIs. An instructor makes a request to edit their course:

```
PATCH /api/courses/abc123
{ "title": "Updated title" }
```

The server checks that the user is authenticated (has a valid token). It finds course `abc123` and updates it. What it does not check: whether the instructor making the request is the one who owns `abc123`.

This means any authenticated user — including a student — could modify another instructor's course, simply by guessing or discovering a course ID. This is an IDOR vulnerability. It is not subtle; it is a routine bug in systems that mistake "logged in" for "authorised."

The fix is simple but must be explicit: every write operation on a resource must verify that `resource.ownerId === req.user.id`.

## The authorize middleware

Add an `authorize` middleware to `src/middleware/authorize.js` that enforces role-based access:

```js
// src/middleware/authorize.js — shape only
export function authorize(...roles) {
  return (req, res, next) => {
    if (!roles.includes(req.user?.role)) {
      return res.status(403).json({ error: 'Forbidden' });
    }
    next();
  };
}
```

Usage in a route:

```js
// Only instructors and admins can reach this route
router.post('/courses', authenticate, authorize('instructor', 'admin'), createCourse);
```

The `authenticate` middleware runs first — it verifies the token and sets `req.user`. Then `authorize` checks the role. If the role is not allowed, the request stops at 403 before the handler runs.

## Ownership checks in services

Role checks are not enough. An instructor with a valid token can call instructor-restricted routes — but they should only be able to modify _their own_ courses. Ownership is checked in the service layer, not in middleware:

```js
// In courses.service.js — shape only
async function updateCourse(courseId, updates, requestingUserId) {
  const course = await Course.findById(courseId);
  if (!course) throw new NotFoundError('Course not found');

  // Ownership check — return the same 404 whether the course doesn't exist
  // or belongs to someone else. Never leak "this course exists but is not yours."
  if (course.instructorId.toString() !== requestingUserId) {
    throw new NotFoundError('Course not found');
  }

  Object.assign(course, updates);
  return course.save();
}
```

Notice the service throws a 404, not a 403. Returning 403 ("Forbidden") confirms that the resource exists, which helps an attacker enumerate IDs. A 404 reveals nothing.

## Role matrix for EduFlow

| Action | Student | Instructor | Admin |
|--------|---------|------------|-------|
| Browse course catalogue | Yes | Yes | Yes |
| Enroll in a course | Yes | No | No |
| Watch enrolled lessons | Yes | No | No |
| Create a course | No | Yes | No |
| Edit own course | No | Yes | No |
| Edit another instructor's course | No | No | No |
| Approve/reject courses | No | No | Yes |
| View platform metrics | No | No | Yes |
| Deactivate user accounts | No | No | Yes |

Implement the `authorize` middleware and write a simple test matrix: for each route in `auth.routes.js` and any routes you add in later chapters, confirm that forbidden roles receive 403 and the correct roles reach the handler.

## Definition of Done

- [ ] The `authorize` middleware exists in `src/middleware/authorize.js`
- [ ] Applying `authorize('instructor')` to a route returns 403 for a student's valid token
- [ ] Applying `authorize('admin')` returns 403 for an instructor's valid token
- [ ] Ownership check in the course update service returns 404 when the requesting user is not the owner
- [ ] An admin can still reach instructor-restricted routes (admin role is separate from instructor role — include both where needed)

Write in `learning-log/04-authorization.md`:

1. What is an IDOR vulnerability? Give a concrete example using EduFlow.
2. Why does the ownership check return 404 instead of 403?
3. What is the difference between authentication and authorization, stated in one sentence each?

All boxes ticked? Then continue to Chapter 7 — Course Catalogue. If not, that is where today's work is.
