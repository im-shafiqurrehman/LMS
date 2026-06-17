# Why MongoDB and Mongoose?

Before you write a line of application code, you choose a database and decide how your application will talk to it. These two choices shape every other decision — the schema, the queries, the performance work. Getting them wrong early means expensive rewrites later. So let us make them deliberately.

## What EduFlow's data actually looks like

An LMS has relationships everywhere. A `Course` has many `Lessons`. A `User` can enroll in many `Courses`, and a `Course` can have many enrolled users — that is a many-to-many relationship with extra data (enrollment date, payment amount). A `ProgressRecord` belongs to both a `User` and a `Lesson`. A `Certificate` belongs to a `User` and a `Course`. A `Question` belongs to a `Course` and a `User`, and an `Answer` belongs to a `Question` and an `Instructor`.

These relationships need to be managed — a lesson without a course is orphaned data; an enrollment without a user is a payment record attached to nobody. The question is: where should that structure live? In the database schema itself, or in the application layer?

## Why MongoDB

**MongoDB** is a document database that stores data as JSON-like documents. Unlike SQL databases, it does not enforce foreign key constraints at the database level. Instead, it gives you **schema flexibility and horizontal scalability** — you can add fields without migrations, and you can shard data across servers naturally. This is well-suited for systems that evolve rapidly and scale to many users.

For EduFlow, MongoDB is the right choice because:

1. **Schema evolution.** As you build, you will discover new fields — referral codes, course tags, video metadata. MongoDB lets you add them without downtime migrations.
2. **Document-shaped data.** Lessons with videos, enrollments with payment details, courses with rich metadata — these are naturally stored as documents, structured exactly how your application needs them.
3. **Horizontal scale.** When EduFlow grows, you can shard users by geography. MongoDB sharding is a natural fit.

> **Worth reading:** Search "MongoDB vs relational databases" — the official MongoDB comparison guide explains the trade-offs clearly.

## Why Mongoose as the ODM

MongoDB is schemaless at the database level — it will accept any document shape you throw at it. This is powerful, but it creates a problem: nothing stops a bug in your code from writing `{ emal: "alice@example.com" }` (a typo) or `{ price: "free" }` (a string where a number belongs) directly into the database. Once bad data is in, cleaning it is painful.

**Mongoose** (an Object Document Mapper, or ODM) solves this by putting a schema layer in your application code. You define the shape and constraints of each document type once, and Mongoose enforces them before any write reaches MongoDB.

### What Mongoose gives you that the native driver does not

**1. Schema definition and built-in validation.**
You define a schema once — the fields, their types, whether they are required, their default values — and Mongoose validates every document against it before saving. A typo in a field name fails immediately with a clear error, not silently as a new field on every document.

```js
// Example shape — you will write this in full in chapter 3.05
// The schema tells Mongoose: email is required, must be a string,
// must be unique, and role must be one of these three values.
const userSchema = new mongoose.Schema({
  email:        { type: String, required: true, unique: true },
  passwordHash: { type: String, required: true },
  role:         { type: String, enum: ['student', 'instructor', 'admin'], default: 'student' },
});
```

**2. Middleware (hooks).**
Mongoose lets you run code automatically before or after certain operations. The most useful example in EduFlow: before saving a user, you can hash their password in a `pre('save')` hook. The hash happens automatically on every save and update — you cannot forget to do it in a route handler.

```js
// Shape only — you write this in chapter 3.05
userSchema.pre('save', async function (next) {
  if (this.isModified('passwordHash')) {
    this.passwordHash = await bcrypt.hash(this.passwordHash, 12);
  }
  next();
});
```

**3. Populate — reference resolution.**
MongoDB stores references as IDs (a `courseId` field on a Lesson, a `userId` field on an Enrollment). The native driver requires you to make separate queries to resolve those references. Mongoose's `.populate()` method resolves them in a single, readable line:

```js
// Fetch a course and resolve its instructorId into the full User document
const course = await Course.findById(id).populate('instructorId', 'name email');
```

Behind the scenes, Mongoose issues the lookup efficiently. You write one line; the database does the work.

**4. Timestamps and virtuals.**
Setting `{ timestamps: true }` in your schema options adds `createdAt` and `updatedAt` fields automatically — maintained by Mongoose, never forgotten. Virtuals add computed properties that appear in query results but are never persisted to the database.

**5. A query API that reads like your application code.**
Mongoose's chainable query API — `.find()`, `.findOne()`, `.findById()`, `.sort()`, `.skip()`, `.limit()` — is designed to match how developers think about data, not how MongoDB's raw query format works. The native driver is powerful, but Mongoose's API is clearer and harder to misuse.

## The trade-off

Mongoose adds an abstraction layer on top of MongoDB. That means a small performance overhead per query, and some situations where you need to drop down to the native driver to run operations Mongoose does not expose cleanly (certain aggregation pipelines, bulk writes, and changestreams). For the overwhelming majority of EduFlow's queries, Mongoose is the correct tool. You will use Mongoose throughout this course.

> **Worth reading:** Search "Mongoose vs native MongoDB driver" — the Mongoose documentation itself has a section explaining when to use each.

---

## The schema at a glance

Here is the data model you will build across the course — the logical picture of how Mongoose models relate:

```
User
  _id, email, passwordHash, role (student | instructor | admin)
  emailVerifiedAt, verificationOtp, otpExpiresAt
  timestamps (createdAt, updatedAt)

InstructorProfile
  _id, userId (→ User), bio, photoUrl, isApproved
  timestamps

Course
  _id, instructorId (→ User), title, slug, description
  coverImageUrl, price, category, status (draft | pending | published | rejected)
  totalLessons, totalDuration
  timestamps

Lesson
  _id, courseId (→ Course), title, videoUrl, durationSeconds
  position (order within the course)
  timestamps

Enrollment
  _id, studentId (→ User), courseId (→ Course)
  amountPaid, stripePaymentIntentId
  enrolledAt

ProgressRecord
  _id, enrollmentId (→ Enrollment), lessonId (→ Lesson)
  completedAt

Certificate
  _id, enrollmentId (→ Enrollment)
  pdfUrl, issuedAt

Question
  _id, courseId (→ Course), userId (→ User)
  content, timestamps

Answer
  _id, questionId (→ Question), instructorId (→ User)
  content, timestamps

RefreshToken
  _id, tokenHash, userId (→ User)
  expiresAt, timestamps
```

Notice the `amountPaid` on `Enrollment`. This is a price snapshot — it records what the student actually paid, not the course's current price. If the instructor changes the price next month, old enrollments keep their original amount.

---

## How references work in Mongoose

MongoDB stores a reference as an ObjectId field. You access the referenced document with `.populate()`:

```js
// One-to-many (Course → Lessons): store courseId on each Lesson
// Query all lessons in a course:
const lessons = await Lesson.find({ courseId: course._id });

// Many-to-many (Users ↔ Courses via Enrollments):
// Store both studentId and courseId on each Enrollment
const enrollments = await Enrollment.find({ studentId: user._id });

// Resolve references:
const course = await Course.findById(id)
  .populate('instructorId', 'name email');
```

When to embed vs reference: embed data that is small, rarely changes independently, and is always needed with its parent (for example, a lesson's position inside a course could be an embedded array). Reference data that is substantial, changes on its own, or is queried independently (a `User` document is referenced by many other documents — it is never embedded).

---

## Key Takeaways

Before moving to the next chapter, you should be able to explain:

- [ ] Why EduFlow uses MongoDB, and what advantages document storage brings
- [ ] Why we use Mongoose instead of the native MongoDB driver, and what Mongoose adds
- [ ] What Mongoose schema validation does and why it matters at the application level
- [ ] How Mongoose's `.populate()` resolves references, and when you would use it
- [ ] When to embed data in a MongoDB document vs reference it by ObjectId

Write your answers in `learning-log/02-why-mongodb.md` and commit before continuing.
