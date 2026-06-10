# Lesson Access and Video Delivery

A student enrolls in a course and sees the lesson list. They click a lesson. What happens next is not as simple as returning a video URL from the database — because that URL, once returned, is shareable. Anyone with the link can watch the lesson without paying. This chapter builds the access layer that prevents that.

---

## The threat model

A naive implementation:

```json
GET /api/lessons/uuid
→ { "videoUrl": "https://res.cloudinary.com/eduflow/video/upload/v1234/abc.mp4" }
```

A student enrolls, gets this URL, posts it to a forum. Anyone clicks it and watches the lesson — forever, for free. The URL has no expiry, no access check, no relationship to enrollment status.

The correct approach is **signed, expiring URLs**. Cloudinary supports them: instead of returning the permanent URL, you generate a short-lived signed URL on demand — valid for, say, 2 hours. After it expires, the link dies. Someone who shares the link gets a playback error two hours later. You can also add additional restrictions (IP, referer header) if needed.

> **Topics to read — protecting your video lessons:**
>
> - **Signed / expiring URLs** — search "Cloudinary signed URLs Node.js." A signed URL includes a cryptographic signature and an expiry timestamp; Cloudinary validates both before serving the file. *Why: you're about to serve paid lessons and need to stop link-sharing.*
> - **Access control on media** — the pattern of generating the signed URL only after an authorization check. *Why: only paying students should ever receive the signed URL, let alone the file itself.*

---

## The ProgressRecord schema

Add to `prisma/schema.prisma`:

```prisma
model ProgressRecord {
  id           String     @id @default(uuid())
  enrollmentId String
  enrollment   Enrollment @relation(fields: [enrollmentId], references: [id], onDelete: Cascade)
  lessonId     String
  lesson       Lesson     @relation(fields: [lessonId], references: [id], onDelete: Cascade)
  completedAt  DateTime   @default(now())

  @@unique([enrollmentId, lessonId])
  @@index([enrollmentId])
}
```

Run: `npx prisma migrate dev --name add-progress-record`

---

## Scaffold the lessons module

```bash
mkdir -p src/modules/lessons
touch src/modules/lessons/lessons.routes.js
touch src/modules/lessons/lessons.service.js
```

Mount at `/api/lessons` in `app.js`.

---

## List lessons for a course

```
GET /api/courses/:courseId/lessons
```

**Public** — anyone can see the lesson list for a published course. This powers the course detail page: the student sees lesson titles and durations before enrolling.

The response must **not** include `videoUrl` — only metadata:

```json
{
  "lessons": [
    {
      "id": "uuid",
      "title": "Setting up your environment",
      "position": 1,
      "durationSeconds": 480,
      "isFree": true,
      "isCompleted": false  // always false for unauthenticated users
    }
  ]
}
```

For authenticated enrolled students, `isCompleted` reflects their actual progress. For everyone else, it's `false`.

The service function takes an optional `studentId`. If provided, join against `ProgressRecord` to compute `isCompleted` per lesson.

✅ The query must not leak `videoUrl` at any point — exclude the field in the Prisma `select`.

---

## Get a lesson (with video access)

```
GET /api/lessons/:id
```

Protected: `authenticate` only (role check happens in the service).

The service must:

1. Find the lesson and its parent course
2. If the lesson is `isFree: true` — generate and return the signed URL (free preview)
3. If the lesson is `isFree: false`:
   - Check enrollment: call `isEnrolled(req.user.id, course.id)` from the enrollments service
   - If not enrolled — return 403: "Enroll in this course to access this lesson."
   - If enrolled — generate and return the signed URL

**Generating the signed URL:**

```js
// Shape — use Cloudinary SDK to generate a signed URL
// cloudinary.url(publicId, {
//   sign_url: true,
//   expires_at: Math.floor(Date.now() / 1000) + 7200,  // 2 hours from now
//   resource_type: 'video'
// })
```

You need the Cloudinary **public ID** of the video (the path within your Cloudinary account), not the full URL. Store the public ID when uploading (in the previous chapter you stored the full URL; extract and store the public ID instead, or derive it from the URL).

Response:

```json
{
  "lesson": {
    "id": "uuid",
    "title": "Setting up your environment",
    "position": 1,
    "durationSeconds": 480,
    "signedVideoUrl": "https://res.cloudinary.com/eduflow/video/upload/s--XXXX--/e_/v1234/abc.mp4",
    "urlExpiresAt": "2025-01-01T14:00:00Z"
  }
}
```

✅ Test with an enrolled student — receives the signed URL.
✅ Test with an unenrolled authenticated student — receives 403.
✅ Test with no auth header — receives 401.
✅ Test the signed URL after 2 hours (or reduce expiry temporarily to 10 seconds for testing) — playback fails.

---

## Mark a lesson complete

```
POST /api/lessons/:id/complete
```

Protected: `authenticate`.

The service must:

1. Verify the user is enrolled in the lesson's course (same check as above)
2. Upsert a `ProgressRecord` — `@@unique([enrollmentId, lessonId])` prevents duplicates

After creating the progress record, check if all lessons in the course now have a completion record for this enrollment. If yes, trigger certificate generation (next chapter).

```js
// Shape — check if all lessons are complete
const totalLessons = await prisma.lesson.count({ where: { courseId } });
const completedLessons = await prisma.progressRecord.count({ where: { enrollmentId } });

if (completedLessons >= totalLessons) {
  // trigger certificate generation
}
```

---

## Definition of Done

- [ ] `GET /api/courses/:id/lessons` returns lesson metadata without `videoUrl` for all users
- [ ] `isCompleted` is accurate for enrolled students and `false` for everyone else
- [ ] `GET /api/lessons/:id` returns a signed Cloudinary URL for enrolled students
- [ ] An unenrolled authenticated student receives 403 on a paid lesson
- [ ] A free preview lesson (`isFree: true`) returns a signed URL to any authenticated user
- [ ] `POST /api/lessons/:id/complete` creates a progress record; a second call on the same lesson does not duplicate it
- [ ] The signed URL cannot be used after it expires

Write in `learning-log/08-lesson-access-and-video.md`:

1. Why does the lesson detail endpoint generate a signed URL rather than returning the stored video URL directly?
2. Where does the access check live — in middleware, or in the service function? Why?
3. What does `isFree` enable, and how does it change the access logic?
