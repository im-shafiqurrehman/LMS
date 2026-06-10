# Why MongoDB and Prisma?

Before you write a line of application code, you choose a database. That choice shapes every other decision — the schema, the queries, the performance work, the migrations. Getting it wrong early means expensive rewrites later. So let's make it deliberately.

## What EduFlow's data actually looks like

An LMS has relationships everywhere. A `Course` has many `Lessons`. A `User` can enroll in many `Courses`, and a `Course` can have many enrolled users — that's a many-to-many relationship with extra data (enrollment date, payment amount). A `ProgressRecord` belongs to both a `User` and a `Lesson`. A `Certificate` belongs to a `User` and a `Course`.

These relationships need to be enforced — a lesson without a course is orphaned data; an enrollment without a user is a payment record attached to nobody. The question is: where should that enforcement live? In the database schema itself, or in the application code?

## Why MongoDB with Prisma

**MongoDB** is a document database that stores data as JSON-like documents. Unlike SQL databases, it doesn't enforce foreign key constraints at the database level. Instead, it gives you **schema flexibility and horizontal scalability** — you can add fields without migrations, and you can shard data across servers naturally. This is perfect for systems that evolve rapidly and scale to many users.

**Prisma** is an ORM that layers schema validation and type safety on top of MongoDB. When you define your schema in `schema.prisma`, Prisma enforces those relationships at the application level — every query is typed, every relationship is validated before it hits the database. This gives you the benefits of a schema without the rigid constraints of SQL.

The trade-off is clear: **MongoDB trades database-level guarantees for developer flexibility and horizontal scale.**

For EduFlow, this is the right choice because:

1. **Schema evolution.** As you build, you'll discover new fields: referral codes, course tags, video metadata. MongoDB lets you add them without downtime migrations.
2. **Horizontal scale.** When EduFlow grows, you'll shard users by geography. MongoDB sharding is a natural fit; PostgreSQL sharding is complex and painful.
3. **Document-shaped data.** Lessons with videos, enrollments with payment details, courses with rich metadata — these are naturally stored as documents. Prisma keeps them normalized in your code while MongoDB stores them flexibly.
4. **Developer experience.** Prisma's type-safe queries feel natural in TypeScript; the MongoDB adapter generates types from your schema file, keeping your code and data in sync.

> **Worth reading:** Search "MongoDB vs relational databases" — the MongoDB comparison guide explains the trade-offs. Also read "Prisma MongoDB adapter" — Prisma's official docs cover how to use MongoDB with type-safe schemas.

---

## Why Prisma as the ORM

An ORM (Object-Relational Mapper, though Prisma works with both relational and document databases) sits between your application code and the database. With MongoDB, Prisma lets you write type-safe queries in JavaScript/TypeScript, manage your schema migrations (even though MongoDB itself is schemaless), and keep your code synchronized with your data model.

The alternatives to Prisma are raw MongoDB queries (using the native driver) or Mongoose. Prisma wins here for three reasons:

1. **Single source of truth.** You define your models once in `schema.prisma`, and Prisma generates type definitions and migrations. Your code stays in sync with your data.
2. **Type safety.** Every query result is typed — if a `Course` doesn't have a `title` field, TypeScript catches it before runtime.
3. **Schema validation.** Prisma's MongoDB adapter validates data shapes, relationships, and required fields at the application layer — filling the gap that MongoDB's lack of database-level constraints creates.

Unlike SQL, MongoDB also gives you the freedom to denormalize when it makes sense — embed related data in documents instead of joining. Prisma supports this; you'll learn when to do it throughout the course.

> **Worth reading:** Search "Prisma MongoDB adapter setup" — the official Prisma docs on MongoDB are clear and worth reading before the next chapter when you install it.

---

## The schema at a glance

Here is the data model you'll build across the course — not the Prisma schema file (that's yours to write), but the logical picture:

```
User
  id, email, password_hash, role (student | instructor | admin)
  email_verified_at
  created_at

InstructorProfile
  id, user_id (→ User), bio, photo_url, is_approved

Course
  id, instructor_id (→ User), title, slug, description
  cover_image_url, price, category, status (draft | pending | published | rejected)
  created_at, updated_at

Lesson
  id, course_id (→ Course), title, video_url, duration_seconds
  position (order within the course)

Enrollment
  id, student_id (→ User), course_id (→ Course)
  amount_paid, stripe_payment_intent_id
  enrolled_at

ProgressRecord
  id, enrollment_id (→ Enrollment), lesson_id (→ Lesson)
  completed_at

Certificate
  id, enrollment_id (→ Enrollment)
  pdf_url, issued_at

Notification
  id, user_id (→ User), type, payload (JSON), read_at
```

Notice what's here and what isn't. The `amount_paid` on `Enrollment` is a price snapshot — it records what the student actually paid, not the course's current price. This is the same price-snapshot pattern you'll see in any order system: if the instructor changes the price next month, old enrollments keep their original amount. The database enforces the relationship; the snapshot enforces the truth of the transaction.

---

## Key Takeaways

Before moving to the next chapter, you should be able to explain:

- [ ] Why EduFlow uses MongoDB, and what advantages document storage brings (hint: flexibility, horizontal scale)
- [ ] What a foreign key *would* do in a relational database, and what problem MongoDB solves differently (hint: application-layer schema + Prisma validation)
- [ ] What an ORM is, and what Prisma specifically gives you on MongoDB (type safety, schema validation, migrations)
- [ ] When you might denormalize data in MongoDB — embed related documents instead of referencing (hint: speed vs consistency trade-off)

Write your answers in `learning-log/02-why-mongodb-and-prisma.md` and commit before continuing.
