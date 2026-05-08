---
title: "Custom analyzers in Azure DocumentDB — case-insensitive and prefix matching"
description: Configure custom text analyzers in Azure DocumentDB full-text search to power case-insensitive search and prefix matching on identifiers, SKUs, and codes.
author: khelanmodi
ms.author: khelanmodi
ms.topic: how-to
ms.date: 05/07/2026
ms.collection:
  - ce-skilling-ai-copilot
---

# Custom Analyzers in Azure DocumentDB

A custom analyzer in Azure DocumentDB full-text search controls how a string field is tokenized at index time and at query time. Custom analyzers unlock case-insensitive search on mixed-case content, prefix matching on identifiers and SKUs, and language-specific folding — all things the legacy `$text` engine listed as **Not available**. See the [migration table](full-text-search-overview.md#migrating-from-the-legacy-text-engine) for context.

## Why the default analyzer isn't enough

The default standard analyzer tokenizes on whitespace and punctuation and case-folds the result. That's the right behavior for prose, but it breaks down for identifiers:

- **Identifier splitting.** `"ABX098"` is split on the digit boundary into `ABX` and `098`. A user query for `"AB2"` finds nothing.
- **Mixed-case matching.** A standard-analyzed field already lower-cases tokens at index time, but using the default analyzer doesn't help prefix or partial matching on IDs.
- **Prefix expectations.** Users typing `"AB2"` into a part-number search box expect `"ABX098"`, `"AB29000"`, and `"ab2-blue"` to surface — none of which the standard analyzer can produce.

## Anatomy of a custom analyzer

A custom analyzer has two parts:

1. **Tokenizer.** Splits the raw value into tokens. Supported tokenizers include:
   - `keyword` — the entire value becomes one token. Essential for IDs, SKUs, codes.
   - `standard` — splits on whitespace and punctuation; the implicit default.
   - `pathHierarchy` — emits each progressively longer prefix at a delimiter boundary. Covered in [Hierarchical identifier search](full-text-search-path-hierarchy.md).
2. **Token filter chain.** A list of transformations applied to each token in order. Common filters:
   - `lowerCase` — lower-cases each token. Required for case-insensitive search.
   - `asciiFolding` — converts accented characters to their ASCII equivalents (`café` → `cafe`).
   - `edgeGram` — emits every leading prefix of a token between `minGram` and `maxGram` characters long. The mechanism behind prefix matching.

Each searchable field accepts two analyzer references:

- **`analyzer`** — the index-time tokenization. Determines what's stored in the inverted index.
- **`searchAnalyzer`** *(optional)* — the query-time tokenization. Defaults to the same value as `analyzer`.

Setting `analyzer` and `searchAnalyzer` to **different** analyzers is the trick that makes prefix matching work: the field is indexed as every prefix using `edgeGram`, but the query is tokenized as a single keyword. A query for `"AB2"` becomes the single token `"ab2"`, which finds the prefix `"ab2"` of the indexed value `"abx098"`... wait, that doesn't match. Prefix matching works when the indexed value's prefix tokens include the query token — for example, indexing `"AB2900"` produces prefix tokens including `"ab2"`, so a query for `"AB2"` matches.

## Pattern 1 — case-insensitive search

A `keyword` tokenizer plus a `lowerCase` and `asciiFolding` filter chain produces a single, normalized token per value. Useful for short codes where the entire value should match exactly but case shouldn't matter.

```javascript
// ✅ Case-insensitive exact match on a code field.
db.runCommand({
  createSearchIndexes: "products_10M",
  indexes: [
    {
      name: "idx_customerCode_kw_lc",
      definition: {
        analyzers: [
          {
            name: "kw_lc",
            tokenizer: { type: "keyword" },
            tokenFilters: [
              { type: "lowerCase" },
              { type: "asciiFolding" }
            ]
          }
        ],
        mappings: {
          dynamic: false,
          fields: {
            customerCode: { type: "string", analyzer: "kw_lc" }
          }
        }
      }
    }
  ]
});

db.products_10M.aggregate([
  { $search: {
      index: "idx_customerCode_kw_lc",
      text: { query: "Crm", path: "customerCode" }
  }},
  { $limit: 20 },
  { $project: { _id: 0, customerCode: 1, score: { $meta: "searchScore" } } }
]);
```

## Pattern 2 — prefix matching on IDs and SKUs

Prefix matching on identifier fields uses `edgeGram` at index time and a plain `keyword` analyzer at query time.

```javascript
// ❌ Default analyzer on an ID field — splits "ABX098" on the digit boundary.
//    A query for "AB2" finds nothing.
db.runCommand({
  createSearchIndexes: "products_10M",
  indexes: [{
    name: "idx_partNumber_default",
    definition: {
      mappings: { dynamic: false, fields: { partNumber: { type: "string" } } }
    }
  }]
});
```

```javascript
// ❌ edgeGram on both the index analyzer and the search analyzer —
//    the query is also expanded into prefixes, exploding the candidate set.
partNumber: { type: "string", analyzer: "kw_lc_edge", searchAnalyzer: "kw_lc_edge" }
```

```javascript
// ✅ edgeGram at index time, plain keyword at query time.
db.runCommand({
  createSearchIndexes: "products_10M",
  indexes: [
    {
      name: "idx_partNumber_prefix",
      definition: {
        analyzers: [
          {
            name: "kw_lc",
            tokenizer: { type: "keyword" },
            tokenFilters: [
              { type: "lowerCase" },
              { type: "asciiFolding" }
            ]
          },
          {
            name: "kw_lc_edge",
            tokenizer: { type: "keyword" },
            tokenFilters: [
              { type: "lowerCase" },
              { type: "asciiFolding" },
              { type: "edgeGram", minGram: 1, maxGram: 32 }
            ]
          }
        ],
        mappings: {
          dynamic: false,
          fields: {
            partNumber: {
              type: "string",
              analyzer:       "kw_lc_edge",   // index-time: every prefix
              searchAnalyzer: "kw_lc"         // query-time: one token
            }
          }
        }
      }
    }
  ]
});

// Prefix match — "AB2" finds "AB29000", "ab2-blue", etc.
db.products_10M.aggregate([
  { $search: {
      index: "idx_partNumber_prefix",
      text: { query: "AB2", path: "partNumber" }
  }},
  { $limit: 20 },
  { $project: { _id: 0, partNumber: 1, score: { $meta: "searchScore" } } }
]);

// Exact match still works — the full value matches its own longest edge-gram.
db.products_10M.aggregate([
  { $search: {
      index: "idx_partNumber_prefix",
      text: { query: "ABX098", path: "partNumber" }
  }},
  { $limit: 20 }
]);
```

## Token filter order

> [!IMPORTANT]
> Token filters run in the order you list them. The recommended order for prefix matching is `lowerCase` → `asciiFolding` → `edgeGram`. If `edgeGram` runs before `lowerCase`, every prefix is emitted twice — once in the original case, once lower-cased — doubling the index size for no semantic benefit. If `asciiFolding` runs after `edgeGram`, accented prefixes aren't normalized.

Cap `maxGram` to a realistic identifier length. Leaving `maxGram: 255` for short IDs costs nothing, but applying it to long descriptive strings inflates the index unnecessarily. A typical value for SKUs and part numbers is 32.

## Custom analyzers vs. `$regex` for prefix matching

| Approach | Cost | Ranked | Case-insensitive | Notes |
| --- | --- | :---: | :---: | --- |
| `$regex: /^AB2/i` | `COLLSCAN` on text fields. | No | Only with `/i` flag. | Acceptable on small collections; impractical at scale. |
| `edgeGram` + `$search.text` | O(log n) via the search index. | Yes (BM25) | Built into the analyzer. | Recommended pattern for prefix matching on identifiers. |

## Related pages

- [Hierarchical identifier search (`pathHierarchy` tokenizer)](full-text-search-path-hierarchy.md)
- [Multi-field search index](full-text-search-multifield-index.md)
- [BM25 keyword search](full-text-search-bm25-keyword.md)
- [Full-text search overview and migration table](full-text-search-overview.md)
