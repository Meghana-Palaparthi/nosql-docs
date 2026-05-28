---
title: Phrase Search and Proximity Matching
titleSuffix: Azure DocumentDB
description: Search for exact phrases and word proximity in Azure DocumentDB, where the order and closeness of terms matter.
author: khelanmodi
ms.author: khelanmodi
ms.topic: how-to
ms.date: 05/07/2026
ms.collection:
  - ce-skilling-ai-copilot
---

# Phrase search and proximity matching in Azure DocumentDB

> [!NOTE]
> Full-text search in Azure DocumentDB is in **Gated Preview**. To enable it on your cluster, contact us at [mongodb-feedback@microsoft.com](mailto:mongodb-feedback@microsoft.com).

Phrase search matches query terms that appear together in a specific order, with an optional `slop` tolerance for intervening tokens. It's the right tool when word order is meaningful (multi-word product names, quoted user input, error strings) and when a plain `$search` + `text` query returns too much noise because the tokens are common individually but rare together.

## What is phrase search?

A `$search` + `text` query matches every document that contains the requested tokens, in any order, anywhere in the field. A `$search` + `phrase` query enforces that the tokens appear in the same order the user supplied, with at most `slop` other tokens between them. Phrase precision matters when:

- The user enters a quoted string such as `"bracket controller"`.
- Title, entity, or error-string matching depends on word order.
- The component words are common individually (`bracket`, `controller`) but their combination is the actual signal.

## The `slop` parameter

`slop` is the number of intervening tokens permitted between two adjacent terms in the phrase.

| `slop` | Behavior | Example match for query `"bracket controller"` |
| :---: | --- | --- |
| `0` | Strict adjacency. | Matches `"bracket controller"`. Does not match `"bracket for controller"`. |
| `1` | One intervening token allowed. | Matches `"bracket for controller"`. |
| `3` | Broader proximity. | Matches `"bracket and the new controller"`; useful for titles with adjectives or articles. |

Higher `slop` values increase recall but reduce precision and ranking quality.

## How to run a phrase query

```javascript
// ❌ Plain text query for "bracket controller" matches tokens in any order.
//    Returns "controller for bracket", "bracket without controller", etc.
db.products_10M.aggregate([
  { $search: {
      index: "idx_title_standard",
      text: { query: "bracket controller", path: "title" }
  }},
  { $limit: 20 }
]);
```

```javascript
// ❌ Regex hack: loses BM25 ranking and forces a COLLSCAN.
db.products_10M.find({ title: { $regex: "bracket.*controller" } });
```

```javascript
// ✅ Phrase search with slop: 3 against the BM25 index.
db.products_10M.aggregate([
  {
    $search: {
      index: "idx_title_standard",
      phrase: {
        query: "bracket controller",
        path: "title",
        slop: 3
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

The same rules from [BM25 keyword search](full-text-search-keyword.md) apply: `$search` is the first stage, `index: "<name>"` is set explicitly, and `$limit` is downstream.

## Phrase search with downstream filters

Equality and range filters live in a downstream `$match`, never inside `$search`:

```javascript
db.products_10M.aggregate([
  { $search: {
      index: "idx_title_standard",
      phrase: { query: "bracket controller", path: "title", slop: 1 }
  }},
  { $limit: 100 },
  { $match: { inStock: true } },
  { $project: { _id: 0, title: 1, price: 1, score: { $meta: "searchScore" } } }
]);
```

## Known constraint

> [!IMPORTANT]
> `$search` + `phrase` and `fuzzy` cannot be combined in a single `$search` clause. When you need both ordered matching and typo tolerance, run two queries and fuse the result lists client-side. The Reciprocal Rank Fusion (RRF) implementation in [Hybrid search](full-text-search-hybrid.md#step-3-reciprocal-rank-fusion-rrf) works as a drop-in: pass the phrase hits and the fuzzy hits as the two ranked lists.

## Related pages

- [BM25 keyword search](full-text-search-keyword.md)
- [Fuzzy search (constraint callout)](full-text-search-fuzzy.md#known-constraint)
- [Hybrid search (BM25 + vector)](full-text-search-hybrid.md)
- [Full-text search overview and migration table](full-text-search-overview.md)

## Next step

> [!div class="nextstepaction"]
> [Create a lifetime free-tier cluster for Azure DocumentDB](free-tier.md)