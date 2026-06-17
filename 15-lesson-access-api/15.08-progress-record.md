# Progress Tracking and Certificates

Students mark lessons complete; when every lesson in a course is complete, a certificate is generated. This chapter builds both — the progress tracking model and the certificate PDF pipeline.

---

## The ProgressRecord and Certificate models

Create `src/models/progressRecord.model.js`:

```js
// src/models/progressRecord.model.js — shape only
import mongoose from 'mongoose';

const progressRecordSchema = new mongoose.Schema({
  enrollmentId: {
    type: mongoose.Schema.Types.ObjectId,
    ref: 'Enrollment',
    required: true,
  },
  lessonId: {
    type: mongoose.Schema.Types.ObjectId,
    ref: 'Lesson',
    required: true,
  },
  completedAt: { type: Date, default: Date.now },
});

progressRecordSchema.index({ enrollmentId: 1, lessonId: 1 }, { unique: true });

export const ProgressRecord = mongoose.model('ProgressRecord', progressRecordSchema);
```

The unique compound index on `{ enrollmentId, lessonId }` ensures a student cannot mark the same lesson complete twice.

Create `src/models/certificate.model.js`:

```js
const certificateSchema = new mongoose.Schema({
  enrollmentId: {
    type: mongoose.Schema.Types.ObjectId,
    ref: 'Enrollment',
    required: true,
    unique: true,
  },
  pdfUrl:   { type: String, required: true },
  issuedAt: { type: Date, default: Date.now },
});

export const Certificate = mongoose.model('Certificate', certificateSchema);
```

---

## Marking a lesson complete

`POST /api/progress/:lessonId`  
Auth: enrolled student

The service must:

1. Verify the student is enrolled (same check as lesson access).
2. Insert a `ProgressRecord`. If one already exists (unique index violation), return 200 — idempotent.
3. Count completed lessons: `ProgressRecord.countDocuments({ enrollmentId })`.
4. Count total lessons in the course: `Lesson.countDocuments({ courseId })`.
5. If they are equal (all lessons complete), trigger certificate generation.

---

## Certificate generation

Certificates are PDF files generated server-side using **pdfkit** and stored in Cloudinary.

Install pdfkit:

```
npm install pdfkit
```

The generation function (in a `src/lib/certificate.js` helper):

```js
// shape only — generate a PDF in memory and upload to Cloudinary
import PDFDocument from 'pdfkit';
import { v2 as cloudinary } from 'cloudinary';

export async function generateCertificate({ studentName, courseTitle, completedAt }) {
  const doc = new PDFDocument();
  // Build the certificate design: student name, course title, completion date, logo

  // Pipe the PDF stream to Cloudinary's upload stream
  // Return the Cloudinary secure_url of the stored PDF
}
```

Once uploaded, store the `pdfUrl` in a `Certificate` document. If a certificate already exists for this enrollment (re-triggering edge case), return the existing one.

---

## The progress endpoint

`GET /api/progress/:courseId`  
Auth: enrolled student

Returns:

```json
{
  "completedLessons": ["lessonId1", "lessonId2"],
  "totalLessons": 5,
  "percentComplete": 40,
  "certificate": null
}
```

If the course is complete, `certificate` contains `{ pdfUrl, issuedAt }`.

---

## Definition of Done

- [ ] `POST /api/progress/:lessonId` marks the lesson complete and returns 200
- [ ] A second request to mark the same lesson complete returns 200 (idempotent, no error)
- [ ] When all lessons in a course are complete, a certificate PDF is generated and stored in Cloudinary
- [ ] `GET /api/progress/:courseId` returns accurate completed lesson IDs and percentage
- [ ] The certificate `pdfUrl` is returned once the course is complete

Write in `learning-log/09-progress-and-certificates.md`:

1. What does "idempotent" mean? Why is it important that marking a lesson complete is idempotent?
2. Why is the certificate generated server-side rather than on the frontend?
3. Why store the PDF in Cloudinary rather than generating it on demand at every request?
