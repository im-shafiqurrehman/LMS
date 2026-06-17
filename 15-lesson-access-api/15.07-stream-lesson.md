# Lesson Access and Video Delivery

A student who has enrolled in a course can now watch its lessons. A student who has not enrolled must not be able to — not by navigating to the URL, not by guessing the lesson ID. This chapter builds the access control and the video delivery pipeline.

---

## The access check

Before returning any lesson content, the backend must verify enrollment. This check lives in the service, not a middleware — it is per-resource, not per-route.

```js
// In lessons.service.js — shape only
async function getLesson(lessonId, requestingStudentId) {
  const lesson = await Lesson.findById(lessonId);
  if (!lesson) throw new NotFoundError('Lesson not found');

  const enrollment = await Enrollment.findOne({
    studentId: requestingStudentId,
    courseId:  lesson.courseId,
  });

  if (!enrollment) {
    throw new ForbiddenError('You are not enrolled in this course');
  }

  return lesson;
}
```

✅ A student enrolled in course A can fetch lessons in course A.
✅ A student not enrolled in course A receives 403 when attempting to fetch any lesson in course A.
❌ Do not skip this check for any lesson endpoint. URL-guessing is a real attack vector — a lesson ID in the URL does not prove enrollment.

---

## Video delivery via Cloudinary

Storing video files in MongoDB or on your Express server is never the right approach. A 500MB video file occupies server RAM for every concurrent request, blocks the event loop, and generates enormous egress charges on a standard VPS. You hand video storage and delivery off to Cloudinary — a specialist CDN built for media.

The workflow:

1. Instructor uploads a video to Cloudinary directly from the frontend (using Cloudinary's upload widget with a signed upload preset).
2. Cloudinary stores the video and returns a `public_id`.
3. The backend stores the `public_id` in the lesson's `videoUrl` field.
4. When an enrolled student requests a lesson, the backend generates a **signed, expiring URL** from the `public_id` and returns it.

The signed URL matters. A plain Cloudinary URL never expires — once shared, anyone can download the video forever. A signed URL expires after a short window (e.g. 15 minutes), so a shared link quickly becomes useless.

```js
import { v2 as cloudinary } from 'cloudinary';

// Generate a signed URL expiring in 15 minutes
const signedUrl = cloudinary.url(lesson.videoUrl, {
  resource_type: 'video',
  sign_url: true,
  expires_at: Math.floor(Date.now() / 1000) + 900, // 15 minutes
});
```

Install Cloudinary:

```
npm install cloudinary
```

Add Cloudinary config keys to `.env` and `config/index.js`.

> **Topics to read — protecting your video lessons**
>
> - **Signed / expiring URLs** — search "Cloudinary signed URLs" for the official documentation. A signed URL stops working after the expiry time, so a student cannot share a permanent download link.
> - **Access control on media** — understanding why the URL alone is not enough: the access check on your backend is what determines who receives a URL in the first place.

---

## Scaffold the lessons module

```
mkdir -p src/modules/lessons
touch src/modules/lessons/lessons.routes.js
touch src/modules/lessons/lessons.service.js
```

| Method | Path | Access | What it does |
|--------|------|--------|-------------|
| `GET` | `/api/courses/:courseId/lessons` | Enrolled student | List all lessons in a course (no video URLs) |
| `GET` | `/api/lessons/:lessonId` | Enrolled student | Get a single lesson with a signed video URL |

---

## Definition of Done

- [ ] `GET /api/lessons/:lessonId` returns 403 for a valid token belonging to a non-enrolled student
- [ ] `GET /api/lessons/:lessonId` returns the lesson with a signed Cloudinary URL for an enrolled student
- [ ] The signed URL includes an expiry (verify by inspecting the URL signature parameters)
- [ ] An unenrolled student cannot access lesson content by guessing a lesson ID

Write in `learning-log/08-lesson-access.md`:

1. Why does the enrollment check live in the service rather than a middleware?
2. What is a signed URL and why is it necessary for paid video content?
3. Why should you never store video files in MongoDB or serve them from your Express server?
