# Instructor Course Management

An instructor's primary job is to create and publish courses. This chapter builds the CRUD endpoints for course management — and adds the authorization rules that ensure one instructor can never touch another's work.

---

## Where the project is now

The catalogue endpoint exists and returns published courses. But those courses were seeded by hand. This chapter adds the routes that let an actual instructor create, edit, and submit a course for review.

## The Lesson model

Before building course management, you need the `Lesson` model — instructors add lessons to their courses. Create `src/models/lesson.model.js`:

```js
// src/models/lesson.model.js — shape only
import mongoose from 'mongoose';

const lessonSchema = new mongoose.Schema(
  {
    courseId: {
      type: mongoose.Schema.Types.ObjectId,
      ref: 'Course',
      required: true,
    },
    title:           { type: String, required: true, trim: true },
    videoUrl:        { type: String, default: null },
    durationSeconds: { type: Number, default: 0 },
    position:        { type: Number, required: true }, // order within the course
  },
  { timestamps: true }
);

lessonSchema.index({ courseId: 1, position: 1 }); // fast sorted lesson retrieval

export const Lesson = mongoose.model('Lesson', lessonSchema);
```

## The course management endpoints

Scaffold the module:

```
mkdir -p src/modules/courses
touch src/modules/courses/courses.routes.js
touch src/modules/courses/courses.service.js
touch src/modules/courses/courses.validators.js
```

All routes in this module require authentication. Routes that modify data also require the `authorize('instructor')` middleware.

| Method | Path | Access | What it does |
|--------|------|--------|-------------|
| `GET` | `/api/courses/mine` | Instructor | List the requesting instructor's own courses |
| `POST` | `/api/courses` | Instructor | Create a new draft course |
| `PATCH` | `/api/courses/:id` | Instructor (owner) | Update course details |
| `POST` | `/api/courses/:id/publish` | Instructor (owner) | Submit for admin review (sets status to `pending`) |
| `DELETE` | `/api/courses/:id` | Instructor (owner) | Delete a draft course |
| `POST` | `/api/courses/:id/lessons` | Instructor (owner) | Add a lesson to a course |
| `PATCH` | `/api/courses/:id/lessons/:lessonId` | Instructor (owner) | Update a lesson |
| `DELETE` | `/api/courses/:id/lessons/:lessonId` | Instructor (owner) | Delete a lesson |

## Ownership enforcement — the right and wrong way

Every service function that modifies a course must verify ownership. The wrong way: check roles in the route and hope the service is never called from elsewhere. The right way: pass `requestingUserId` into the service and check there.

```js
// In courses.service.js — shape only
async function updateCourse(courseId, updates, requestingUserId) {
  const course = await Course.findById(courseId);
  if (!course) throw new NotFoundError('Course not found');
  if (course.instructorId.toString() !== requestingUserId) {
    throw new NotFoundError('Course not found'); // 404, not 403 — see authorization chapter
  }
  Object.assign(course, updates);
  return course.save();
}
```

## Keeping `totalLessons` and `totalDuration` in sync

When a lesson is added or removed, the parent course's `totalLessons` and `totalDuration` fields need to stay accurate. There are two approaches:

**Compute on the fly:** query `Lesson.countDocuments({ courseId })` and sum durations on every catalogue request. This is clean but adds a query per course.

**Denormalise and update:** when a lesson is added, increment `totalLessons` and add to `totalDuration` on the parent course. When deleted, decrement.

Use denormalisation here. The catalogue is read far more often than lessons are added. The update cost is small and predictable:

```js
// After inserting a lesson — shape only
await Course.findByIdAndUpdate(courseId, {
  $inc: { totalLessons: 1, totalDuration: lesson.durationSeconds },
});
```

✅ Use MongoDB's `$inc` operator for counter increments — it is atomic.
❌ Do not read the course, increment in JavaScript, and write it back. A concurrent request between the read and write would produce an incorrect count.

---

## Definition of Done

- [ ] `POST /api/courses` creates a draft course and returns it
- [ ] `GET /api/courses/mine` returns only the requesting instructor's courses
- [ ] `PATCH /api/courses/:id` updates the course; a different instructor's valid token returns 404
- [ ] `POST /api/courses/:id/publish` sets status to `pending`; admin approval is still required before it appears in the catalogue
- [ ] `POST /api/courses/:id/lessons` adds a lesson and increments `totalLessons` on the parent course
- [ ] A student's token attempting any instructor route returns 403

Write in `learning-log/06-instructor-course-management.md`:

1. Why does the ownership check return 404 instead of 403?
2. Why use `$inc` for counter increments instead of read-modify-write?
3. What is the trade-off between computing `totalLessons` on the fly vs denormalising it?
