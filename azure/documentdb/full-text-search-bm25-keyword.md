---
title: "BM25 keyword search in Azure DocumentDB — ranked text retrieval"
description: Run BM25-scored keyword search on Azure DocumentDB using the createSearchIndexes command and the $search.text aggregation operator.
author: khelanmodi
ms.author: khelanmodi
ms.topic: how-to
ms.date: 05/07/2026
ms.collection:
  - ce-skilling-ai-copilot
---

# BM25 Keyword Search in Azure DocumentDB

BM25 is the relevance-ranking algorithm at the heart of Azure DocumentDB full-text search. This page shows how to create a search index covering one text field and run a BM25-scored `$search.text` query against it. If you're migrating from the legacy `$text` operator or `{ field: "text" }` index type, see the [migration table](full-text-search-overview.md#migrating-from-the-legacy-text-engine) on the overview page.

## What is BM25?

BM25 (Best Match 25) ranks each document by how well its terms match the query, taking three signals into account:

- **Term frequency.** Documents that mention a query term more often score higher, with diminishing returns so that a single repeated keyword can't dominate.
- **Inverse document frequency.** Rare terms across the corpus carry more weight than common ones, so matching `bracket` is worth more than matching `the`.
- **Document length normalization.** Long documents aren't unfairly penalized for repeating terms, and short documents aren't unfairly rewarded.

The algorithm is well-studied, defaults are reasonable, and the score returned by Azure DocumentDB is monotonic — higher means more relevant within a single query. Scores are not directly comparable across different queries.

## Creating a search index for keyword search

```javascript
// ❌ Community MongoDB text-index shape — not the Azure DocumentDB FTS path.
//    The new $search engine does not consume this form.
db.products_10M.createIndex({ title: "text" });
```

```javascript
// ✅ Define a search index with createSearchIndexes.
//    Always set dynamic: false and enumerate fields explicitly.
db.runCommand({
  createSearchIndexes: "products_10M",
  indexes: [
    {
      name: "idx_title_standard",
      definition: {
        mappings: {
          dynamic: false,
          fields: {
            title: { type: "string" }
          }
        }
      }
    }
  ]
});
```

Name the index after the field and intent so it's easy to reference from `$search`. After creation the index appears in `db.products_10M.getIndexes()` alongside other indexes on the collection.

## Running a `$search.text` query

```javascript
// ❌ Regex substring search on a large collection — COLLSCAN, unranked,
//    case-sensitive without the /i flag.
db.products_10M.find({ title: { $regex: "bracket", $options: "i" } });
```

```javascript
// ✅ BM25 keyword search with $search.text.
//    Always pass index: "<name>" and cap with a downstream $limit.
db.products_10M.aggregate([
  {
    $search: {
      index: "idx_title_standard",
      text: {
        query: "bracket",
        path: "title"
      }
    }
  },
  { $limit: 20 },
  {
    $project: {
      _id: 0,
      title: 1,
      score: { $meta: "searchScore" }
    }
  }
]);
```

Three rules apply to every `$search.text` query:

- `$search` is the first stage of the aggregation pipeline.
- `index: "<name>"` is set explicitly so the engine knows which search index to use.
- The result set is capped with a downstream `{ $limit: N }` stage. There is no `count` or `limit` field inside `$search`.

## Working with scores

Every document returned by `$search` carries a BM25 score that you read through `{ $meta: "searchScore" }`. Project it whenever ranking matters:

```javascript
db.products_10M.aggregate([
  { $search: { index: "idx_title_standard", text: { query: "bracket", path: "title" } } },
  { $limit: 50 },
  { $project: { _id: 1, title: 1, score: { $meta: "searchScore" } } },
  { $match: { score: { $gte: 1.0 } } }
]);
```

`$search` returns results in BM25-descending order, so an explicit `$sort` is unnecessary unless a downstream stage disturbs that ordering. Use a minimum-score `$match` to drop low-relevance hits and a fixed `$limit` to bound payload size.

## Combining with filters

Equality and range filters live in a downstream `$match`, not inside `$search`. This keeps the search stage index-pure and lets BM25 score the full candidate set before filtering:

```javascript
db.products_10M.aggregate([
  { $search: { index: "idx_title_standard", text: { query: "bracket", path: "title" } } },
  { $limit: 200 },
  { $match: { inStock: true, price: { $lte: 100 } } },
  { $project: { _id: 0, title: 1, price: 1, score: { $meta: "searchScore" } } }
]);
```

When `$search.compound` ships, server-side `should` / `must` clauses across multiple fields will be supported. Until then, multi-field queries follow the [fan-out-and-merge pattern](full-text-search-multifield-index.md#fan-out-and-merge-multi-field-query-workaround).

## When to use BM25 vs. other modes

| Mode | Best for | Trade-off |
| --- | --- | --- |
| `$search.text` (BM25) | Ranked keyword search on prose, titles, and descriptions. | Requires a search index. |
| `$regex` | Single-character substring patterns on small collections or fields with an existing B-tree index. | Unranked; `COLLSCAN` on large collections. |
| Equality (`$eq`) | Exact-match lookup on identifiers or enums. | No tokenization — won't match `"bracket"` inside `"steel bracket"`. |

## Related pages

- [Fuzzy search](full-text-search-fuzzy.md)
- [Phrase search and proximity matching](full-text-search-phrase-proximity.md)
- [Custom analyzers](full-text-search-custom-analyzers.md)
- [Hybrid search (BM25 + vector)](full-text-search-hybrid.md)
- [Full-text search overview and migration table](full-text-search-overview.md)
