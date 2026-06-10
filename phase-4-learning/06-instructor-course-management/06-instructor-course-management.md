# Instructor Course Management

An instructor's relationship with their courses is like a shopkeeper's relationship with their shelves — they stock them, price them, arrange them, and take items down when needed. But in EduFlow, there's an extra gate: before a course reaches students, an admin reviews and approves it. This chapter builds the instructor side of that workflow.

---

## The Lesson schema

Before building course management, add `Lesson` to the schema. An instructor manages lessons as part of managing a course.

```prisma
model Lesson {
  id              String   @id @default(uuid())
  courseId        String
  course          Course   @relation(fields: [courseId], references: [id], onDelete: Cascade)
  title           String
  position        Int      // order within the course
  videoUrl        String?  // Cloudinary URL — added after upload
  durationSeconds Int      @default(0)
  isFree          Boolean  @default(false) // preview lessons visible without enrollment
  createdAt       DateTime @default(now())
  updatedAt       DateTime @updatedAt

  progressRecords ProgressRecord[]

  @@unique([courseId, position])
  @@index([courseId])
}
```

`@@unique([courseId, position])` prevents two lessons in the same course from having the same position. You'll enforce ordering through this field.

Run: `npx prisma migrate dev --name add-lesson`

---

## Scaffold

```bash
touch src/modules/courses/instructor.routes.js
touch src/modules/courses/instructor.service.js
```

Mount `instructor.routes.js` at `/api/instructor` in `app.js`. Every route in this router is protected: `authenticate` + `authorize('INSTRUCTOR')` applied globally on the router.

---

## Course CRUD for instructors

### Create a course

```
POST /api/instructor/courses
```

Request body:

```json
{
  "title": "Mastering Node.js",
  "description": "A comprehensive guide to Node.js in production.",
  "categoryId": "uuid",
  "price": 49.99
}
```

The service function must:

1. Generate a URL-safe `slug` from the title (e.g. `"Mastering Node.js"` → `"mastering-node-js"`). Check for uniqueness; if the slug exists, append a short random suffix.
2. Set `instructorId` from `req.user.id` — never from the request body
3. Set `status` to `DRAFT` — always; the instructor doesn't publish directly
4. Create the record and return it

✅ The `instructorId` is always `req.user.id`. A body field named `instructorId` must be ignored.

### List the instructor's own courses

```
GET /api/instructor/courses
```

Returns all courses where `instructorId = req.user.id`. Include lesson count and enrollment count. This is the instructor's dashboard view — they see all statuses (draft, pending, published, rejected).

### Get a single course with its lessons

```
GET /api/instructor/courses/:id
```

Ownership check: the course's `instructorId` must equal `req.user.id`. Return 404 if not found or not owned.

Include all lessons ordered by `position`.

### Update a course

```
PUT /api/instructor/courses/:id
```

Allowed fields to update: `title`, `description`, `categoryId`, `price`, `coverImageUrl`.

Ownership check required. An instructor cannot update a course that is `PUBLISHED` or `PENDING` — those are under review or live. Return 400 with a clear message: "Published or pending courses cannot be edited. Unpublish the course first."

If `title` changes, regenerate the slug.

### Publish a course (submit for review)

```
POST /api/instructor/courses/:id/submit
```

This transitions the course from `DRAFT` to `PENDING`. The admin will then review it.

The service must validate before transitioning:
- The course has at least 3 lessons
- The course has a cover image URL set
- The course has a description longer than 100 characters

If any check fails, return 400 with which requirement isn't met. Don't transition a course that doesn't meet the minimum bar.

✅ Return 409 if the course is already `PENDING` or `PUBLISHED`.

### Delete a course

```
DELETE /api/instructor/courses/:id
```

Ownership check required. Only `DRAFT` and `REJECTED` courses can be deleted. A `PUBLISHED` or `PENDING` course cannot be deleted — it either has enrolled students or is under review.

Use `onDelete: Cascade` in the Prisma schema (already set on `Lesson`) to ensure lessons are deleted with the course.

---

## Lesson management

### Add a lesson

```
POST /api/instructor/courses/:id/lessons
```

Request body:

```json
{
  "title": "Setting up your environment",
  "isFree": false
}
```

The service must:
- Verify course ownership
- Compute the next position (max existing position + 1, or 1 if the course has no lessons)
- Create the lesson with `videoUrl: null` (the video is uploaded separately)
- Increment `totalLessons` on the course

### Reorder lessons

```
PUT /api/instructor/courses/:id/lessons/reorder
```

Request body:

```json
{
  "lessonIds": ["uuid-1", "uuid-3", "uuid-2"]
}
```

The service assigns `position` values based on the array order. This must be done in a **transaction** — if any update fails, none of them should commit.

```js
// Prisma transaction — shape only
await prisma.$transaction(
  lessonIds.map((id, index) =>
    prisma.lesson.update({ where: { id }, data: { position: index + 1 } })
  )
);
```

✅ Verify the reorder by fetching the lesson list — positions must match the submitted order.

### Delete a lesson

```
DELETE /api/instructor/courses/:id/lessons/:lessonId
```

Verify ownership of the course. Delete the lesson. Decrement `totalLessons` on the course. Re-number remaining lessons so positions are contiguous (no gaps).

---

## Video upload with Cloudinary

Video files are large. Storing them in MongoDB as binary data would bloat the database, make backups slow, and serve files at whatever bandwidth your server has — which is limited and expensive. Cloudinary is a media platform that handles upload, transcoding, storage, and delivery globally. You upload once; they serve everywhere.

Install:

```bash
npm install cloudinary multer multer-storage-cloudinary
```

Configure in `src/lib/cloudinary.js`:

```js
// Shape — initialise the Cloudinary SDK with config values from config/index.js
// Export the configured instance
```

Create `src/middleware/upload.js`:

```js
// Shape — configure multer with CloudinaryStorage
// Store videos in the "eduflow/videos" folder
// Allow only video/mp4, video/webm, video/quicktime
// Reject files over 500MB
```

### The upload endpoint

```
POST /api/instructor/courses/:id/lessons/:lessonId/video
```

Protected by `authenticate`, `authorize('INSTRUCTOR')`, and the upload middleware.

The service must:
1. Verify the instructor owns the course
2. Verify the lesson belongs to the course
3. Get the uploaded file's Cloudinary URL and duration from the Cloudinary response
4. Update the lesson with `videoUrl` and `durationSeconds`
5. Recalculate `totalDuration` on the course (sum of all lesson durations)

> **Worth reading:** Search "Cloudinary Node.js upload guide" — the official Cloudinary docs have a Node.js quickstart that covers the `cloudinary.uploader.upload()` API and the response shape. Read it before implementing the upload.

✅ After upload, the lesson's `videoUrl` is set and the course's `totalDuration` reflects the added lesson.

---

## Definition of Done

- [ ] An instructor can create, list, update, submit, and delete their own courses
- [ ] An instructor cannot access, modify, or delete another instructor's courses (returns 404)
- [ ] Course submission (`DRAFT → PENDING`) is blocked if the minimum requirements aren't met
- [ ] Lessons can be added, reordered (using a transaction), and deleted
- [ ] After lesson deletion, remaining positions are contiguous with no gaps
- [ ] Video upload sets `videoUrl` and updates `totalDuration` on the course
- [ ] A student or unauthenticated user receives 403 or 401 on all `/api/instructor/*` routes

Write in `learning-log/06-instructor-course-management.md`:

1. Why is the `instructorId` taken from `req.user.id` instead of the request body? What attack does this prevent?
2. What is a database transaction and why is it used for lesson reordering?
3. Why are video files stored in Cloudinary rather than in MongoDB?
