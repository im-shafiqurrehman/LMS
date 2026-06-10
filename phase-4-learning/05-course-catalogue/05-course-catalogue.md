# Course Catalogue — Browsing and Filtering

The catalogue is the front door of EduFlow. It's what a visitor sees before they've created an account, and it's what turns a browser into a buyer. Get it wrong — slow queries, broken filters, stale data — and the rest of the product doesn't matter.

This chapter builds a public, paginated, filterable catalogue endpoint and teaches you two concepts that will follow you everywhere: **the N+1 query problem**, and **cursor vs offset pagination**.

---

## The schema additions

You need `Course` and `Category` in the database. Open `prisma/schema.prisma` and add:

```prisma
model Category {
  id      String   @id @default(uuid())
  name    String   @unique
  slug    String   @unique
  courses Course[]
}

model Course {
  id             String       @id @default(uuid())
  instructorId   String
  instructor     User         @relation(fields: [instructorId], references: [id])
  categoryId     String?
  category       Category?    @relation(fields: [categoryId], references: [id])
  title          String
  slug           String       @unique
  description    String
  coverImageUrl  String?
  price          Decimal      @db.Decimal(10, 2)
  status         CourseStatus @default(DRAFT)
  totalLessons   Int          @default(0)
  totalDuration  Int          @default(0)  // seconds
  createdAt      DateTime     @default(now())
  updatedAt      DateTime     @updatedAt

  lessons     Lesson[]
  enrollments Enrollment[]

  @@index([status, createdAt])
  @@index([categoryId, status])
  @@index([instructorId])
}

enum CourseStatus {
  DRAFT
  PENDING
  PUBLISHED
  REJECTED
}
```

Notice `@@index([status, createdAt])`. The catalogue query will filter on `status = PUBLISHED` and sort by `createdAt DESC`. Without this index, MongoDB scans the entire `courses` collection for every request. With it, the scan is a tiny fraction of that. Add the index **now**, before the data is large — adding an index after the fact on a large collection is a slow operation.

Run the migration:

```bash
npx prisma migrate dev --name add-course-and-category
```

---

## The N+1 problem — learn it here, never forget it

Here is the most common mistake engineers make when building a list endpoint:

```js
// The naive implementation — DO NOT do this
const courses = await prisma.course.findMany({ where: { status: 'PUBLISHED' } });

// Then for each course, fetch the instructor name separately
for (const course of courses) {
  course.instructor = await prisma.user.findUnique({ where: { id: course.instructorId } });
}
```

This looks harmless. With 10 courses it fires 11 queries — one for the list, one per course for the instructor. With 100 courses it fires 101 queries. With 1,000 it fires 1,001. The load on your database grows linearly with the size of the result set, and at realistic scale this makes a simple list page take several seconds to load.

The correct approach is to fetch the related data in the same query using Prisma's `include`:

```js
// Correct — one query with a JOIN
const courses = await prisma.course.findMany({
  where: { status: 'PUBLISHED' },
  include: {
    instructor: { select: { id: true, name: true } },
    category: { select: { id: true, name: true, slug: true } },
    _count: { select: { enrollments: true } }
  }
});
```

This is **one query** regardless of how many courses are in the result. MongoDB performs the lookup internally using `$lookup` or Prisma's relation handling. The N+1 pattern is a silent performance killer; learning to spot and prevent it now will save you many late nights later.

> **Worth reading:** Search "N+1 query problem explained" — the original Ruby on Rails community documented this pattern clearly, and every backend engineer should recognise it on sight.

---

## Offset vs cursor pagination

Your catalogue endpoint must be paginated — returning all courses at once is never acceptable in production.

**Offset pagination** is what most engineers reach for first:

```js
db.courses.find().sort({ createdAt: -1 }).skip(40).limit(20);
```

`OFFSET 40` means "skip the first 40 rows." This is readable and easy to implement. The problem: with large data sets, MongoDB still scans and discards the first 40 rows even though you don't return them. `OFFSET 1000000` in a large collection is genuinely slow. It also has a subtle correctness bug: if a new course is published while the user is browsing page 3, the rows shift and the user sees a duplicate or skips one.

**Cursor pagination** solves both problems:

```js
db.courses.find({
  $or: [
    { createdAt: { $lt: lastCreatedAt } },
    { createdAt: lastCreatedAt, _id: { $lt: lastId } }
  ]
}).sort({ createdAt: -1, _id: -1 }).limit(20);
```

Instead of "skip N rows," you say "give me rows before this specific point." The cursor (the `createdAt` and `id` of the last row seen) is stable — it doesn't shift when new rows are inserted. And MongoDB can use the index to jump directly to that point; there's no skipped-row scan.

The trade-off: cursor pagination can't do "jump to page 7." You can only go forward. For EduFlow's catalogue (infinite scroll or "load more"), this is acceptable. For a use case needing "go to page N," offset is the pragmatic choice despite the limitations.

**For this catalogue: use cursor pagination.**

> **Worth reading:** Search "cursor pagination vs offset pagination" — the Prisma blog has an excellent article on this exact topic, with code examples in their query API.

---

## Scaffold the courses module

```bash
mkdir -p src/modules/courses
touch src/modules/courses/courses.routes.js
touch src/modules/courses/courses.service.js
touch src/modules/courses/courses.validators.js
```

Mount the router in `app.js`:
```js
// Mount at /api/courses
```

---

## The catalogue endpoint

```
GET /api/courses
```

Query parameters:

| Parameter | Type | Description |
|---|---|---|
| `category` | string | category slug to filter by |
| `minPrice` | number | minimum price (inclusive) |
| `maxPrice` | number | maximum price (inclusive) |
| `cursor` | string | cursor for next-page fetch (the `id` of the last item on the previous page) |
| `limit` | number | items per page, default 20, max 50 |

Response shape:

```json
{
  "data": [
    {
      "id": "uuid",
      "title": "Mastering React",
      "slug": "mastering-react",
      "coverImageUrl": "https://...",
      "price": "49.99",
      "instructor": { "id": "uuid", "name": "Shafiq Rehman" },
      "category": { "id": "uuid", "name": "Frontend", "slug": "frontend" },
      "enrollmentCount": 342,
      "totalLessons": 24,
      "totalDuration": 36000
    }
  ],
  "nextCursor": "uuid-of-last-item",
  "hasMore": true
}
```

Write the `getCatalogue(filters)` service function. It must:

1. Build a `where` clause from the filters (status always `PUBLISHED`; add category and price constraints if provided)
2. Apply cursor if provided: `where: { ..., id: { lt: cursor } }` combined with ordering by `createdAt DESC, id DESC`
3. Fetch `limit + 1` items — if you get `limit + 1` back, there are more pages; return the first `limit` and set `hasMore: true`
4. Include `instructor` (name only), `category`, and enrollment count via `_count`
5. Return the shaped response

✅ Verify: `GET /api/courses` returns 20 published courses. `GET /api/courses?category=frontend` filters correctly. `GET /api/courses?cursor=LAST_ID` returns the next page.
❌ Don't include draft or pending courses in the public catalogue. The `status: 'PUBLISHED'` filter must always be present.

---

## Seed the database for testing

You need published courses to test the catalogue. Create `prisma/seed.js`:

```js
// Seed shape:
// - 3 categories (Frontend, Backend, DevOps)
// - 1 instructor user (verified, INSTRUCTOR role)
// - 10 published courses distributed across categories with varied prices
```

Add to `package.json`:

```json
"prisma": {
  "seed": "node prisma/seed.js"
}
```

Run: `npx prisma db seed`

---

## Definition of Done

- [ ] `GET /api/courses` returns a paginated list of published courses with instructor name, category, and enrollment count — in one query (check the Prisma query log with `log: ['query']` enabled temporarily)
- [ ] Category filter, price filter, and cursor pagination all work correctly
- [ ] A request for `?category=nonexistent` returns an empty array, not an error
- [ ] Draft and pending courses do not appear in the catalogue response
- [ ] The `_count` for enrollments is accurate after manually inserting an enrollment row in the database

Write in `learning-log/05-course-catalogue.md`:

1. What is the N+1 problem? Walk through a specific example of how it would occur on this catalogue endpoint without `include`.
2. What is the difference between offset and cursor pagination? Why did we choose cursor here?
3. What does the `@@index([status, createdAt])` on the `Course` model actually do in MongoDB, and why does it help this query?
