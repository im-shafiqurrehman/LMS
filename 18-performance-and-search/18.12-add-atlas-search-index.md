# Search — Full-Text Course Search

Students browsing the catalogue can filter by category and price, but that is not always enough. When a student knows what they want ("docker kubernetes deploy"), they need full-text search.

---

## MongoDB Atlas Search vs basic text index

**Basic MongoDB text index** (`$text` search) supports simple keyword matching. It is built into MongoDB and requires no external service. The limitation: it does not rank results by relevance — all matches are equally weighted.

**MongoDB Atlas Search** uses Lucene under the hood. It supports relevance scoring, fuzzy matching (typo-tolerance), and autocomplete. For a production course platform, relevance matters: "docker" should rank "Docker for Beginners" above "Introduction to Computing (briefly covers Docker)."

For EduFlow, use **MongoDB Atlas Search**. If you are running locally, use the basic text index as a development fallback. Atlas Search is the production target.

---

## Add the search index

In MongoDB Atlas (your cloud-hosted cluster), create a search index on the `courses` collection:

```json
{
  "mappings": {
    "fields": {
      "title": [{ "type": "string", "analyzer": "lucene.standard" }],
      "description": [{ "type": "string", "analyzer": "lucene.standard" }],
      "status": [{ "type": "string" }]
    }
  }
}
```

---

## The search endpoint

`GET /api/courses/search?q=docker`

The service queries using the Mongoose aggregation pipeline with `$search`:

```js
// courses.service.js — shape only
const courses = await Course.aggregate([
  {
    $search: {
      index: 'default',
      text: { query: searchTerm, path: ['title', 'description'], fuzzy: { maxEdits: 1 } },
    },
  },
  { $match: { status: 'published' } },
  { $addFields: { score: { $meta: 'searchScore' } } },
  { $sort: { score: -1 } },
  { $limit: 20 },
]);
```

Note: `.populate()` does not work inside aggregation pipelines. Use `$lookup` stages to join instructor and category data, or make separate queries after the aggregation.

---

## Development fallback — basic text index

While building locally (without Atlas Search), create a basic text index and use `$text` queries:

```js
courseSchema.index({ title: 'text', description: 'text' });

// Fallback search
const courses = await Course.find({
  $text: { $search: searchTerm },
  status: 'published',
}).sort({ score: { $meta: 'textScore' } });
```

---

## Definition of Done

- [ ] `GET /api/courses/search?q=react` returns published courses matching "react" in title or description
- [ ] Results are sorted by relevance (Atlas Search score or text score)
- [ ] Draft and pending courses are excluded from search results
- [ ] An empty or missing `q` parameter returns a 400 error

Write in `learning-log/10-search.md`:

1. What is the difference between a basic MongoDB text index and Atlas Search?
2. Why does relevance scoring matter for a course catalogue search?
