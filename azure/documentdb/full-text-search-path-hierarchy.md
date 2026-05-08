---
title: "Hierarchical identifier search in Azure DocumentDB — pathHierarchy tokenizer"
description: Index hierarchical identifiers like BN-747-ENG-2024.05 in Azure DocumentDB full-text search with the pathHierarchy tokenizer so any ancestor segment matches the full value.
author: khelanmodi
ms.author: khelanmodi
ms.topic: how-to
ms.date: 05/07/2026
ms.collection:
  - ce-skilling-ai-copilot
---

# Hierarchical Identifier Search in Azure DocumentDB

The `pathHierarchy` tokenizer lets Azure DocumentDB full-text search match a hierarchical identifier by any ancestor segment. A query for `BN-747` matches `BN-747`, `BN-747-ENG`, and `BN-747-ENG-2024.05`. This page shows how to define a `pathHierarchy` analyzer and when to choose it over the default analyzer or `edgeGram`. The legacy `$text` engine had no equivalent.

## What are hierarchical identifiers?

Hierarchical identifiers encode parent-child relationships in their value:

- `BN-747-ENG-2024.05` — manufacturing lot with revision and date.
- `/regions/eu/ops` — slash-delimited resource path.
- `com.example.service` — reverse-DNS service identifier.

The search requirement is the same in each case: querying any prefix at a delimiter boundary should match the full value. A query for `BN-747` should find every document whose hierarchical identifier starts with `BN-747`, regardless of how many additional segments follow.

## Why other analyzers don't work for hierarchical IDs

| Analyzer | Behavior on `BN-747-ENG-2024.05` | Why it's wrong |
| --- | --- | --- |
| Default (standard) | Splits on `-` into `BN`, `747`, `ENG`, `2024`, `05`. | Matches any segment in any order. A query for `ENG` matches every document with `ENG` anywhere — too noisy for hierarchical lookups. |
| `edgeGram` (character prefix) | Emits every character prefix: `B`, `BN`, `BN-`, `BN-7`, `BN-74`, `BN-747`, etc. | Matches non-boundary prefixes like `BN-7` or `BN-74` that aren't legitimate ancestors. |
| `pathHierarchy` ✅ | Emits `BN`, `BN-747`, `BN-747-ENG`, `BN-747-ENG-2024.05`. | Matches only true ancestors at the delimiter boundary. |

## How `pathHierarchy` works

With `delimiter: "-"`, the `pathHierarchy` tokenizer expands `BN-747-ENG-2024.05` into four tokens — one for each ancestor in the hierarchy:

```text
BN
BN-747
BN-747-ENG
BN-747-ENG-2024.05
```

A `$search.text` query for any of those four values matches the document. A query for `747` alone — which is not an ancestor at the delimiter boundary — does not.

## Creating a pathHierarchy search index

```javascript
// ✅ pathHierarchy tokenizer with case-folding for typed input.
db.runCommand({
  createSearchIndexes: "products_10M",
  indexes: [
    {
      name: "idx_basicNumber_path_prefix",
      definition: {
        analyzers: [
          {
            name: "basic_path",
            tokenizer: { type: "pathHierarchy", delimiter: "-" },
            tokenFilters: [
              { type: "lowerCase" },
              { type: "asciiFolding" }
            ]
          }
        ],
        mappings: {
          dynamic: false,
          fields: {
            basicNumber: { type: "string", analyzer: "basic_path" }
          }
        }
      }
    }
  ]
});
```

## Querying hierarchical identifiers

```javascript
// ✅ Any of these queries finds BN-789-ENG-2024.05.
db.products_10M.aggregate([
  { $search: {
      index: "idx_basicNumber_path_prefix",
      text: { query: "bn-789", path: "basicNumber" }
  }},
  { $limit: 20 },
  { $project: { _id: 0, basicNumber: 1, score: { $meta: "searchScore" } } }
]);
```

The query `bn-789` matches because `BN-789` is one of the ancestor tokens emitted by the analyzer. The query `bn-789-eng` also matches. A query for `789` alone does not — it isn't a delimiter-boundary ancestor.

The standard `$search` rules still apply: `$search` is the first stage, `index: "<name>"` is set explicitly, and `$limit` is downstream.

## Choosing the right delimiter

| Delimiter | Common use |
| :---: | --- |
| `-` (dash) | Manufacturing lots, revision IDs, codes like `BN-747-ENG-2024.05`. |
| `.` (dot) | Reverse-DNS identifiers like `com.example.service`. |
| `/` (slash) | Resource paths like `/regions/eu/ops`. |

A `pathHierarchy` analyzer accepts a single delimiter. To support multiple delimiters on the same field, define two analyzers and reference one of them per searchable field, or normalize the value at write time.

## Known constraint

> [!IMPORTANT]
> `pathHierarchy` does not match mid-hierarchy segments. A query for `747` alone will not match `BN-747-ENG` because `747` is not an ancestor token. If your application also needs mid-hierarchy lookups, index the same field a second time inside a [multi-field search index](full-text-search-multifield-index.md) using a `keyword` + `lowerCase` + `edgeGram` analyzer. The `pathHierarchy` index covers ancestor lookups; the second mapping covers substring lookups.

Use `db.products_10M.aggregate([...]).explain("executionStats")` to confirm `$search` is using the expected index.

## Related pages

- [Custom analyzers (case-insensitive and edgeGram patterns)](full-text-search-custom-analyzers.md)
- [Multi-field search index](full-text-search-multifield-index.md)
- [Full-text search overview and migration table](full-text-search-overview.md)
