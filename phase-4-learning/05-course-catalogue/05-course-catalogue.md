# Course Catalogue — Browsing and Filtering

The catalogue is the front door of EduFlow. It is what a visitor sees before they have created an account, and it is what turns a browser into a buyer. Get it wrong — slow queries, broken filters, stale data — and the rest of the product does not matter.

This chapter builds a public, paginated, filterable catalogue endpoint and teaches you two concepts that will follow you everywhere: **the N+1 query problem**, and **cursor vs offset pagination**.

---

## The Mongoose models

You need `Course` and `Category` models. Create `src/models/course.model.js` and `src/models/category.model.js`.

The `Category` schema:

```js
// src/models/category.model.js — shape only
import mongoose from 'mongoose';

const categorySchema = new mongoose.Schema({
  name: { type: String, required: true, unique: true },
  slug: { type: String, required: true, unique: true },
});

export const Category = mongoose.model('Category', categorySchema);
```

The `Course` schema:

```js
// src/models/course.model.js — shape only
import mongoose from 'mongoose';

const courseSchema = new mongoose.Schema(
  {
    instructorId: {
      type: mongoose.Schema.Types.ObjectId,
      ref: 'User',
      required: true,
    },
    categoryId: {
      type: mongoose.Schema.Types.ObjectId,
      ref: 'Category',
      default: null,
    },
    title:         { type: String, required: true, trim: true },
    slug:          { type: String, required: true, unique: true },
    description:   { type: String, required: true },
    coverImageUrl: { type: String, default: null },
    price:         { type: Number, required: true, min: 0 },
    status: {
      type: String,
      enum: ['draft', 'pending', 'published', 'rejected'],
      default: 'draft',
    },
    totalLessons:  { type: Number, default: 0 },
    totalDuration: { type: Number, default: 0 }, // seconds
  },
  { timestamps: true }
);
```

Notice `courseSchema.index({ status: 1, createdAt: -1 })` — add this before the model export. The catalogue query will filter on `status: 'published'` and sort by `createdAt` descending. Without this index, Mongoose (and MongoDB) scan the entire `courses` collection for every request. With it, the scan is a tiny fraction of that. Add the index **now**, before the data is large — creating an index after the fact on a large collection is a slow, blocking operation.

```js
courseSchema.index({ status: 1, createdAt: -1 });
courseSchema.index({ categoryId: 1, status: 1 });
courseSchema.index({ instructorId: 1 });

export const Course = mongoose.model('Course', courseSchema);
```

---

## The N+1 problem — learn it here, never forget it

Here is the most common mistake engineers make when building a list endpoint:

```js
// The naive implementation — DO NOT do this
const courses = await Course.find({ status: 'published' });

// Then for each course, fetch the instructor separately
for (const course of courses) {
  course.instructor = await User.findById(course.instructorId);
}
```

This looks harmless. With 10 courses it fires 11 queries — one for the list, one per course for the instructor. With 100 courses it fires 101 queries. With 1,000 it fires 1,001. The load on your database grows linearly with the result set, and at realistic scale this makes a simple list page take several seconds to load.

The correct approach uses Mongoose's `.populate()` to resolve references in one operation:

```js
// Correct — one round trip to the database
const courses = await Course.find({ status: 'published' })
  .populate('instructorId', 'name email')   // resolve instructorId → User (name and email only)
  .populate('categoryId', 'name slug')       // resolve categoryId → Category
  .lean();                                   // return plain JS objects, faster than Mongoose Documents
```

Mongoose issues an additional lookup query for the populated fields, but it batches all IDs in a single query per reference type — not one query per document. The pattern is `O(number of reference types)`, not `O(number of documents)`.

> **Worth reading:** Search "N+1 query problem explained" — the original Ruby on Rails community documented this pattern clearly. Every backend engineer should recognise it on sight.

---

## Offset vs cursor pagination

Your catalogue endpoint must be paginated — returning all courses at once is never acceptable in production.

**Offset pagination** is what most engineers reach for first:

```js
// Offset pagination — readable, but flawed at scale
const courses = await Course.find({ status: 'published' })
  .skip(page * limit)
  .limit(limit);
```

`skip(40)` means "scan and discard the first 40 documents." This is readable and easy to implement. The problem: with large data sets, MongoDB still scans the skipped documents. `skip(1000000)` in a large collection is genuinely slow. It also has a subtle correctness bug: if a new course is published while the user is browsing page 3, the rows shift and the user may see a duplicate or skip one.

**Cursor pagination** solves both problems:

```js
// Cursor pagination — stable and index-friendly
const query = { status: 'published' };

if (cursor) {
  // Continue from where we left off: find courses created before the last seen one
  query.createdAt = { $lt: new Date(cursor) };
}

const courses = await Course.find(query)
  .sort({ createdAt: -1 })
  .limit(limit + 1)   // fetch one extra to determine if there are more pages
  .populate('instructorId', 'name email')
  .lean();
```

Instead of "skip N documents," you say "give me documents before this specific point." The cursor (the `createdAt` timestamp of the last document seen) is stable — it does not shift when new documents are inserted. MongoDB can use the index to jump directly to that point.

The trade-off: cursor pagination cannot jump to page 7. You can only go forward. For EduFlow's catalogue (infinite scroll or "load more"), this is acceptable.

**For this catalogue: use cursor pagination.**

> **Worth reading:** Search "cursor pagination vs offset pagination" — several engineering blogs (Slack, Twitter, Notion) have written about this trade-off at scale.

---

## Scaffold the courses module

```
mkdir -p src/modules/courses
touch src/modules/courses/courses.routes.js
touch src/modules/courses/courses.service.js
touch src/modules/courses/courses.validators.js
```

Mount the router in `app.js` at `/api/courses`.

---

## The catalogue endpoint

```
GET /api/courses
```

Query parameters:

| Parameter | Type | Description |
|-----------|------|-------------|
| `category` | string | category slug to filter by |
| `minPrice` | number | minimum price (inclusive) |
| `maxPrice` | number | maximum price (inclusive) |
| `cursor` | string | ISO timestamp of the last item seen on the previous page |
| `limit` | number | items per page, default 20, max 50 |

Response shape:

```json
{
  "data": [
    {
      "id": "...",
      "title": "Mastering React",
      "slug": "mastering-react",
      "coverImageUrl": "https://...",
      "price": 49.99,
      "instructor": { "id": "...", "name": "Shafiq Rehman" },
      "category": { "id": "...", "name": "Frontend", "slug": "frontend" },
      "totalLessons": 24,
      "totalDuration": 36000
    }
  ],
  "nextCursor": "2024-01-15T12:00:00.000Z",
  "hasMore": true
}
```

Write the `getCatalogue(filters)` service function. It must:

1. Build a Mongoose query from the filters — `status` is always `'published'`; add category and price constraints if provided
2. Apply cursor if provided: `{ createdAt: { $lt: new Date(cursor) } }`
3. Fetch `limit + 1` items — if you get `limit + 1` back, there are more pages; return only the first `limit` and set `hasMore: true`
4. Populate `instructorId` (name only) and `categoryId` (name, slug)
5. Return the shaped response with `nextCursor` set to the last item's `createdAt.toISOString()`

✅ Verify: `GET /api/courses` returns published courses with instructor name and category.
✅ `GET /api/courses?category=frontend` filters correctly.
✅ `GET /api/courses?cursor=LAST_TIMESTAMP` returns the next page.
❌ Do not include draft or pending courses in the public catalogue.

---

## Seed the database for testing

Create `src/seed.js`:

```js
// Seed shape:
// - 3 categories (Frontend, Backend, DevOps)
// - 1 instructor user (verified, role: 'instructor')
// - 10 published courses distributed across categories with varied prices
```

Add to `package.json`:

```json
"scripts": {
  "seed": "node src/seed.js"
}
```

Run: `npm run seed`

---

## Definition of Done

- [ ] `GET /api/courses` returns a paginated list of published courses with instructor name, category, and total lessons
- [ ] The query uses `.populate()` — not a loop of individual User lookups (check with `mongoose.set('debug', true)` temporarily)
- [ ] Category filter, price filter, and cursor pagination all work correctly
- [ ] `?category=nonexistent` returns an empty array, not an error
- [ ] Draft and pending courses do not appear in the catalogue response
- [ ] The compound index `{ status: 1, createdAt: -1 }` exists on the `courses` collection

Write in `learning-log/05-course-catalogue.md`:

1. What is the N+1 problem? Walk through how it would occur without `.populate()`.
2. What is the difference between offset and cursor pagination? Why did you choose cursor here?
3. What does the compound index `{ status: 1, createdAt: -1 }` do, and why does it help this query?

All boxes ticked? Then continue to Chapter 8. If not, that is where today's work is.
