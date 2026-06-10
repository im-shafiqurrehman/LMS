# Search — Finding Courses

A student arrives on EduFlow with something in mind: "machine learning," "Node.js," "guitar for beginners." The catalogue filters help when they know what category to browse. Search helps when they know what they want to learn but not how it's categorised. This chapter adds full-text search over course titles and descriptions using MongoDB Atlas Search.

---

## Why not regex search

The tempting approach is a MongoDB regex query:

```js
db.courses.find({
  $or: [
    { title: /node.js/i },
    { description: /node.js/i }
  ]
});
```

This works for small datasets. The problem: regex queries cannot use an index effectively. With 10,000 courses, this is a full collection scan on every search request. And it has no concept of relevance — a course with "node.js" in the first word of the title ranks the same as one with it buried in the 500th word of the description.

**MongoDB Atlas Search** solves both problems. It uses a pre-built search index over the document's fields, so searches are fast. And it has built-in relevance ranking that scores results by how well they match — terms in the title can be weighted higher than terms in the description.

> **Worth reading:** Search "MongoDB Atlas Search tutorial" — the official MongoDB documentation explains how to set up search indexes and the various search operators available.

---

## Adding the search index

MongoDB Atlas Search uses indexes defined in the Atlas UI or via the MongoDB Atlas API. For EduFlow, you'll create a search index on the `courses` collection.

**In MongoDB Atlas:**

1. Navigate to your cluster → Search → Create Search Index
2. Select the `courses` collection
3. Configure the index with the following JSON:

```json
{
  "mappings": {
    "dynamic": false,
    "fields": {
      "title": {
        "type": "string",
        "analyzer": "lucene.standard"
      },
      "description": {
        "type": "string",
        "analyzer": "lucene.standard"
      },
      "status": {
        "type": "string"
      }
    }
  }
}
```

This configuration:
- Sets `dynamic: false` to only index the fields we specify
- Uses the standard Lucene analyzer for text processing
- Includes `status` so we can filter by published courses

The index is automatically kept current when documents are inserted or updated — no triggers needed.

---

## The search endpoint

```
GET /api/courses/search?q=node+js&limit=20
```

Use MongoDB's `$search` aggregation stage:

```js
const results = await prisma.$runCommandRaw({
  aggregate: 'Course',
  pipeline: [
    {
      $search: {
        index: 'default',
        compound: {
          must: [
            {
              text: {
                query: queryString,
                path: ['title', 'description'],
                fuzzy: {}
              }
            }
          ],
          filter: [
            {
              text: {
                query: 'PUBLISHED',
                path: 'status'
              }
            }
          ]
        }
      }
    },
    {
      $project: {
        id: 1,
        title: 1,
        slug: 1,
        cover_image_url: 1,
        price: 1,
        score: { $meta: 'searchScore' }
      }
    },
    {
      $sort: { score: -1, created_at: -1 }
    },
    {
      $limit: limit
    }
  ]
});
```

The search configuration:
- `compound.must` requires the query terms to match
- `compound.filter` ensures only published courses are returned
- `fuzzy: {}` enables fuzzy matching for typos (e.g., "jvscript" matches "javascript")
- `$meta: 'searchScore'` includes the relevance score in results

Response shape (same as catalogue but with a `relevanceScore` field):

```json
{
  "data": [...],
  "query": "node js",
  "total": 12
}
```

✅ Test: searching "javascript" returns courses with that word in title or description. Courses with it in the title rank higher than those with it only in the description.
✅ Test: searching "jvscript" (typo) returns results with fuzzy matching enabled.
✅ Test: searching with an empty query (`q=`) returns 400, not an empty result.

---

## Definition of Done

- [ ] `GET /api/courses/search?q=term` returns ranked results for published courses
- [ ] Title matches rank higher than description-only matches
- [ ] An empty query (`q=`) returns 400, not an empty result or an error
- [ ] The search uses the Atlas Search index, not a regex scan
- [ ] Fuzzy matching works for typos (test with "jvscript" → "javascript")

Write in `learning-log/10-search.md`:

1. Why is regex search slow, and what MongoDB feature does Atlas Search use instead?
2. What is a MongoDB Atlas Search index, and how does it differ from a regular MongoDB index?
3. What advantage does Atlas Search have over traditional full-text search? (Hint: fuzzy matching and multi-language support.)
