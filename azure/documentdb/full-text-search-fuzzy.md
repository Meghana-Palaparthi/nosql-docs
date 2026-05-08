---
title: "Fuzzy search in Azure DocumentDB — typo-tolerant text matching"
description: Match misspelled terms in Azure DocumentDB full-text search using the fuzzy.maxEdits parameter on $search.text, with guidance on tuning Levenshtein edit distance.
author: khelanmodi
ms.author: khelanmodi
ms.topic: how-to
ms.date: 05/07/2026
ms.collection:
  - ce-skilling-ai-copilot
---

# Fuzzy Search in Azure DocumentDB

Fuzzy search lets `$search.text` match terms that are within a bounded **Levenshtein edit distance** of the user's query. A query for `bracXet` finds documents containing `bracket` because the two strings differ by exactly one character substitution. This page shows how to enable and tune fuzzy matching, and when to reach for the alternative — the [`edgeGram` custom analyzer](full-text-search-custom-analyzers.md). If you're migrating from the legacy `$text` engine, fuzzy matching was previously listed as **Not available**; it's now supported through the same `$search` operator. See the [migration table](full-text-search-overview.md#migrating-from-the-legacy-text-engine) for context.

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
> - Tokens of three characters or fewer — almost everything matches at `maxEdits: 1` on short strings.
> - The default behavior on every endpoint. Fuzziness broadens the candidate set, hurts precision, and increases latency.

## How to enable fuzzy matching

```javascript
// ❌ maxEdits: 3 on short tokens matches almost everything in the corpus.
db.products_10M.aggregate([
  { $search: {
      index: "idx_title_standard",
      text: { query: "bracXet", path: "title", fuzzy: { maxEdits: 3 } }
  }},
  { $limit: 20 }
]);
```

```javascript
// ❌ Wildcard regex — COLLSCAN, no BM25 ranking, no real distance metric.
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

The same rules from [BM25 keyword search](full-text-search-bm25-keyword.md) apply: `$search` is the first stage, `index: "<name>"` is set explicitly, and `$limit` lives downstream of `$search`.

## Tuning `maxEdits`

| `maxEdits` | When to use |
| :---: | --- |
| `1` | Default for short and medium-length user queries. High precision and good recall on single-character typos. |
| `2` | Better recall on longer words at the cost of more noise. Avoid on tokens of four characters or fewer. |
| `≥ 3` | Avoid. Almost everything in the corpus matches and BM25 ranking can no longer separate signal from noise. |

Combine fuzzy queries with a minimum-score threshold (`$match: { score: { $gte: ... } }`) or a fixed `$limit` to drop low-relevance hits.

## Fuzzy vs. edge n-gram prefix matching

Fuzzy and prefix matching solve different problems and have different cost profiles.

| Need | Use |
| --- | --- |
| Tolerate typos in prose, product names, descriptions, search-as-you-type. | `$search.text` + `fuzzy.maxEdits: 1` |
| Match the start of identifiers, SKUs, codes, short tokens. | [`edgeGram` custom analyzer](full-text-search-custom-analyzers.md) |

Edge n-gram indexing is O(log n) and ranked, but it doesn't tolerate typos in the middle of a token. Fuzzy tolerates typos but doesn't help with partial-token prefixes on identifier-like fields.

## Known constraint

> [!IMPORTANT]
> `$search.phrase` and `fuzzy` cannot be combined inside the same `$search` clause. If you need both ordering tolerance and typo tolerance, run a phrase query and a fuzzy query separately and fuse the result lists client-side. Reciprocal Rank Fusion (RRF) is the recommended fusion approach — see [Hybrid search](full-text-search-hybrid.md#step-3-reciprocal-rank-fusion-rrf) for an implementation.

## Related pages

- [BM25 keyword search](full-text-search-bm25-keyword.md)
- [Custom analyzers (edgeGram alternative for IDs and SKUs)](full-text-search-custom-analyzers.md)
- [Phrase search and proximity matching](full-text-search-phrase-proximity.md)
- [Full-text search overview and migration table](full-text-search-overview.md)


## Next step

> [!div class="nextstepaction"]
> [Create a lifetime free-tier cluster for Azure DocumentDB](free-tier.md)