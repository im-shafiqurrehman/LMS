# Why MongoDB?

Before you write a line of application code, you choose a database. That choice shapes every other decision — the schema, the queries, the performance work, the migrations. Getting it wrong early means expensive rewrites later. So let's make it deliberately.

## What EduFlow's data actually looks like

An LMS has relationships everywhere. A `Course` has many `Lessons`. A `User` can enroll in many `Courses`, and a `Course` can have many enrolled users — that's a many-to-many relationship with extra data (enrollment date, payment amount). A `ProgressRecord` belongs to both a `User` and a `Lesson`. A `Certificate` belongs to a `User` and a `Course`. A `Question` belongs to a `Course` and a `User`, and an `Answer` belongs to a `Question` and an `Instructor`.

These relationships need to be enforced — a lesson without a course is orphaned data; an enrollment without a user is a payment record attached to nobody. The question is: where should that enforcement live? In the database schema itself, or in the application code?

## Why MongoDB (without an ORM)

**MongoDB** is a document database that stores data as JSON-like documents. Unlike SQL databases, it doesn't enforce foreign key constraints at the database level. Instead, it gives you **schema flexibility and horizontal scalability** — you can add fields without migrations, and you can shard data across servers naturally. This is perfect for systems that evolve rapidly and scale to many users.

**Why no ORM?** We're using the native MongoDB driver directly. Here's why:

1. **Simplicity.** ORMs add abstraction layers that hide what's actually happening. With the native driver, you write queries that map directly to MongoDB operations. When something goes wrong, you see the actual query, not an ORM-generated one.

2. **Performance.** Every ORM adds overhead — query translation, result mapping, relationship loading. The native driver has zero overhead. You get exactly what MongoDB returns, no transformation layers.

3. **Learning.** Understanding MongoDB's native query language makes you a better developer. You learn about indexes, aggregation pipelines, and document structure directly. These skills transfer regardless of which language or framework you use later.

4. **Control.** ORMs make assumptions about how to structure your data. With the native driver, you decide when to embed documents vs. reference them. You control the query shape, the projection, and the aggregation strategy.

5. **No migration hell.** MongoDB is schemaless at the database level. With an ORM, you still need to track schema changes in migration files. Without an ORM, you add fields to your documents as needed — the database doesn't care.

The trade-off is clear: **MongoDB trades database-level guarantees for developer flexibility and horizontal scale. The native driver trades ORM convenience for simplicity, performance, and direct control.**

For EduFlow, this is the right choice because:

1. **Schema evolution.** As you build, you'll discover new fields: referral codes, course tags, video metadata. MongoDB lets you add them without downtime migrations.

2. **Horizontal scale.** When EduFlow grows, you'll shard users by geography. MongoDB sharding is a natural fit; PostgreSQL sharding is complex and painful.

3. **Document-shaped data.** Lessons with videos, enrollments with payment details, courses with rich metadata — these are naturally stored as documents. You structure them exactly how your application needs them.

4. **Performance at scale.** The native driver has zero overhead. Every query you write is exactly what MongoDB executes. No hidden N+1 queries, no lazy loading surprises.

> **Worth reading:** Search "MongoDB native driver Node.js" — the official MongoDB Node.js driver documentation is excellent. Also read "MongoDB vs relational databases" — the MongoDB comparison guide explains the trade-offs.

---

## The schema at a glance

Here is the data model you'll build across the course — the logical picture of how documents relate:

```
User
  _id, email, password_hash, role (student | instructor | admin)
  email_verified_at
  created_at

InstructorProfile
  _id, user_id (→ User), bio, photo_url, is_approved

Course
  _id, instructor_id (→ User), title, slug, description
  cover_image_url, price, category, status (draft | pending | published | rejected)
  created_at, updated_at

Lesson
  _id, course_id (→ Course), title, video_url, duration_seconds
  position (order within the course)

Enrollment
  _id, student_id (→ User), course_id (→ Course)
  amount_paid, stripe_payment_intent_id
  enrolled_at

ProgressRecord
  _id, enrollment_id (→ Enrollment), lesson_id (→ Lesson)
  completed_at

Certificate
  _id, enrollment_id (→ Enrollment)
  pdf_url, issued_at

Question
  _id, course_id (→ Course), user_id (→ User)
  content, created_at

Answer
  _id, question_id (→ Question), instructor_id (→ User)
  content, created_at

Notification
  _id, user_id (→ User), type, payload (JSON), read_at
```

Notice what's here and what isn't. The `amount_paid` on `Enrollment` is a price snapshot — it records what the student actually paid, not the course's current price. This is the same price-snapshot pattern you'll see in any order system: if the instructor changes the price next month, old enrollments keep their original amount.

The `Question` and `Answer` collections enable the Q&A system — students ask questions about courses, and instructors answer them. This is a critical feature for learning engagement.

---

## How relationships work in MongoDB (without an ORM)

Without an ORM, you manage relationships manually. Here's the pattern:

**One-to-many (Course → Lessons):** Store the `course_id` in each Lesson document. Query: `db.lessons.find({ course_id: courseId })`

**Many-to-many (Users ↔ Courses via Enrollments):** Create an Enrollment collection with both `student_id` and `course_id`. Query: `db.enrollments.find({ student_id: userId })`

**Embedded vs. referenced:** When related data is small and rarely changes alone, embed it. For example, a Course's tags could be an array inside the Course document. When related data is large or changes independently, reference it by ID. Lessons are referenced by course_id because they're substantial entities.

You'll implement these patterns throughout the course. The native driver gives you full control over when to embed vs. reference.

---

## Key Takeaways

Before moving to the next chapter, you should be able to explain:

- [ ] Why EduFlow uses MongoDB, and what advantages document storage brings (hint: flexibility, horizontal scale)
- [ ] Why we use the native MongoDB driver instead of an ORM (hint: simplicity, performance, learning, control)
- [ ] How relationships work in MongoDB without an ORM (hint: store IDs, query by those IDs, choose embed vs. reference)
- [ ] When you might embed data in MongoDB vs. reference it (hint: size, change frequency, query patterns)

Write your answers in `learning-log/02-why-mongodb.md` and commit before continuing.
