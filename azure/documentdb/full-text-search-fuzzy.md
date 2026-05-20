---
title: "Fuzzy Search - Typo-Tolerant Text Matching"
titleSuffix: Azure DocumentDB
description: Add typo tolerance to Azure DocumentDB full-text search so misspelled queries still return the right results.
author: khelanmodi
ms.author: khelanmodi
ms.topic: how-to
ms.date: 05/07/2026
ms.collection:
  - ce-skilling-ai-copilot
---

# Fuzzy search in Azure DocumentDB

> [!NOTE]
> Full-text search in Azure DocumentDB is in **Gated Preview**. To enable it on your cluster, contact us at [mongodb-feedback@microsoft.com](mailto:mongodb-feedback@microsoft.com).

Fuzzy search lets `$search` + `text` match terms that are within a bounded **Levenshtein edit distance** of the user's query. A query for `bracXet` finds documents containing `bracket` because the two strings differ by exactly one character substitution.

## What is fuzzy search?

Levenshtein edit distance counts the number of single-character insertions, deletions, or substitutions required to turn one string into another. With `fuzzy.maxEdits: 1`, a query for `bracXet` matches `bracket` (one substitution), `bracket` matches `brackets` (one insertion), and `cofee` matches `coffee` (one insertion). With `maxEdits: 2`, `recieve` matches `receive` (one substitution plus one transposition counts as two edits in the standard variant).

## When to use fuzzy search

> [!TIP]
> Use fuzzy search for:
>
> - Search-as-you-type UIs and any user-facing search box.
> - Product, catalog, or knowledge-base search where misspellings are common.
> - Log or entity search where input is dictated, OCR'd, or otherwise noisy.

> [!CAUTION]
> Avoid fuzzy search for:
>
> - Programmatic queries where precision matters more than recall.
> - Tokens of three characters or fewer. Almost everything matches at the default `maxEdits: 2` on short strings. Use `maxEdits: 1` or skip fuzzy entirely for very short tokens.
> - The default behavior on every endpoint. Fuzziness broadens the candidate set, hurts precision, and increases latency.

## Running a fuzzy query

```javascript
// ❌ Fuzzy on a short token at the default maxEdits: 2. Almost the whole corpus matches.
db.products_10M.aggregate([
  { $search: {
      index: "idx_title_standard",
      text: { query: "cat", path: "title", fuzzy: {} }
  }},
  { $limit: 20 }
]);
```

```javascript
// ❌ Wildcard regex: COLLSCAN, no BM25 ranking, no real distance metric.
db.products_10M.find({ title: { $regex: ".*br.cket.*" } });
```

```javascript
// ✅ Fuzzy keyword search with maxEdits: 1.
//    The same idx_title_standard index from BM25 keyword search powers this query.
db.products_10M.aggregate([
  {
    $search: {
      index: "idx_title_standard",
      text: {
        query: "bracXet",
        path: "title",
        fuzzy: { maxEdits: 1 }
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

The same rules from [BM25 keyword search](full-text-search-keyword.md) apply: `$search` is the first stage, `index: "<name>"` is set explicitly, and `$limit` lives downstream of `$search`.

## Tuning `maxEdits`

| `maxEdits` | When to use |
| :---: | --- |
| `1` | Stricter typo tolerance. Use for short or medium-length user queries where you want high precision and only single-character typos. |
| `2` (default) | Broader recall on longer words. Used when `maxEdits` is omitted. Avoid on tokens of four characters or fewer; noise dominates. |

> `maxEdits` accepts only `1` or `2`. Any other value is rejected at query time with `'fuzzy.maxEdits' must be 1 or 2`. There is no `0` (use a non-fuzzy `text` query for exact match) and no `≥ 3`.

## Fuzzy with downstream filters

Combine fuzzy queries with a minimum-score threshold (`$match: { score: { $gte: ... } }`) or a fixed `$limit` to drop low-relevance hits.

## Known constraint

> [!IMPORTANT]
> `$search` + `phrase` and `fuzzy` cannot be combined inside the same `$search` clause. If you need both ordering tolerance and typo tolerance, run a phrase query and a fuzzy query separately and fuse the result lists client-side. Reciprocal Rank Fusion (RRF) is the recommended fusion approach. See [Hybrid search](full-text-search-hybrid.md#step-3-reciprocal-rank-fusion-rrf) for an implementation.

## Related pages

- [BM25 keyword search](full-text-search-keyword.md)
- [Phrase search and proximity matching](full-text-search-phrase-proximity.md)
- [Hybrid search (BM25 + vector)](full-text-search-hybrid.md)
- [Full-text search overview and migration table](full-text-search-overview.md)

## Next step

> [!div class="nextstepaction"]
> [Create a lifetime free-tier cluster for Azure DocumentDB](free-tier.md)