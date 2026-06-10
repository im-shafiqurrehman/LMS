# Progress Tracking and Certificates

When a student finishes every lesson in a course, something should happen. Not just a database row changing — something they can hold, share, and show. EduFlow generates a PDF certificate, stores it in Cloudinary, and makes it available for download. This chapter builds the completion detection and the certificate generation.

---

## The Certificate schema

Add to `prisma/schema.prisma`:

```prisma
model Certificate {
  id           String     @id @default(uuid())
  enrollmentId String     @unique
  enrollment   Enrollment @relation(fields: [enrollmentId], references: [id])
  pdfUrl       String
  issuedAt     DateTime   @default(now())
}
```

`@unique` on `enrollmentId` ensures one certificate per enrollment — no duplicates regardless of how many times the trigger fires.

Run: `npx prisma migrate dev --name add-certificate`

---

## Scaffold the certificates module

```bash
mkdir -p src/modules/certificates
touch src/modules/certificates/certificates.service.js
touch src/modules/certificates/certificates.routes.js
```

---

## Completion detection

In the lesson completion handler from the previous chapter, you check whether all lessons are complete. Extract this into a proper function in `certificates.service.js`:

```js
// checkAndIssueCertificate(enrollmentId, courseId)
// 1. Count total published lessons in the course
// 2. Count ProgressRecords for this enrollment
// 3. If equal and both > 0, call generateCertificate(enrollmentId)
// 4. Return whether a certificate was issued
```

Call this from `POST /api/lessons/:id/complete` after creating the progress record.

✅ Use a database transaction to prevent a race condition: if the student marks two lessons complete simultaneously (two concurrent requests), both might pass the count check and trigger two certificate generations. Wrap the count-check and certificate-create in a transaction.

---

## Generating the PDF

You will use the **`pdfkit`** library to build a certificate PDF in memory, then upload the buffer directly to Cloudinary.

```bash
npm install pdfkit
```

The certificate must include:
- EduFlow branding (a header with the platform name)
- Student's full name (pulled from their profile — add a `name` field to the User or a separate Profile model if not already present)
- Course title
- Instructor name
- Completion date
- A unique certificate ID (the `Certificate.id`)

```js
// generateCertificate(enrollmentId) — shape
// 1. Fetch the enrollment with student name, course title, instructor name
// 2. Create a PDFDocument in memory (pdfkit)
// 3. Build the certificate layout:
//    - Title: "Certificate of Completion"
//    - Body: "This certifies that [name] has successfully completed [course]..."
//    - Date, certificate ID
// 4. End the document and collect the buffer
// 5. Upload the buffer to Cloudinary (resource_type: 'raw', folder: 'eduflow/certificates')
// 6. Create the Certificate record with the Cloudinary URL
// 7. (Optionally) trigger the "certificate ready" email notification
```

Cloudinary upload from a buffer:

```js
// Shape — upload a buffer to Cloudinary
cloudinary.uploader.upload_stream(
  { resource_type: 'raw', folder: 'eduflow/certificates', public_id: `cert_${enrollmentId}` },
  (error, result) => { /* handle result.secure_url */ }
)
```

> **Interesting to read:** The PDF format was created by Adobe in 1993 as a way to represent documents independent of any application, hardware, or operating system. It became an ISO open standard in 2008. The spec runs over 700 pages — `pdfkit` abstracts essentially all of it. Search "PDF specification history" if you want the full story.

---

## Download the certificate

```
GET /api/certificates/enrollment/:enrollmentId
```

Protected: `authenticate`.

The service must:
1. Find the certificate by `enrollmentId`
2. Verify that `enrollment.studentId === req.user.id` (a student can only download their own certificate)
3. Generate a signed, expiring Cloudinary URL for the PDF (same pattern as video)
4. Return the URL and issued date

Response:

```json
{
  "certificate": {
    "id": "uuid",
    "issuedAt": "2025-06-01T10:00:00Z",
    "downloadUrl": "https://res.cloudinary.com/.../cert_uuid.pdf?signature=..."
  }
}
```

---

## Student progress overview

```
GET /api/enrollments/my
```

Protected: `authenticate` + `authorize('STUDENT')`.

Returns all of the student's enrollments with progress:

```json
{
  "enrollments": [
    {
      "id": "uuid",
      "course": { "id": "uuid", "title": "Mastering Node.js", "coverImageUrl": "..." },
      "enrolledAt": "2025-05-01T00:00:00Z",
      "totalLessons": 24,
      "completedLessons": 18,
      "progressPercent": 75,
      "certificate": null  // or { "id": "uuid", "issuedAt": "..." }
    }
  ]
}
```

This powers the student's dashboard. The `completedLessons` count comes from a `_count` on `ProgressRecord`, and `progressPercent` is computed in the service layer.

---

## Definition of Done

- [ ] Marking the final lesson complete triggers certificate generation
- [ ] A certificate PDF is generated, uploaded to Cloudinary, and the record is stored in the database
- [ ] Marking the same final lesson complete twice does not generate two certificates
- [ ] `GET /api/certificates/enrollment/:id` returns a signed download URL for the student's own certificate
- [ ] Another student's enrollment ID returns 404
- [ ] `GET /api/enrollments/my` returns accurate progress percentages and certificate status for all enrollments

Write in `learning-log/09-progress-and-certificates.md`:

1. What race condition exists in the "check if course is complete" logic, and how does a database transaction prevent it?
2. Why is the certificate PDF stored in Cloudinary rather than generated on every download request?
3. Why does the certificate download endpoint generate a signed URL rather than returning the raw `pdfUrl` from the database?
