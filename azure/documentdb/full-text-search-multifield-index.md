---
title: "Multi-field search index in Azure DocumentDB"
description: Define one Azure DocumentDB full-text search index that maps multiple fields, each with its own analyzer pair, and query across them with the fan-out-and-merge pattern.
author: khelanmodi
ms.author: khelanmodi
ms.topic: how-to
ms.date: 05/07/2026
ms.collection:
  - ce-skilling-ai-copilot
---

# Multi-Field Search Index in Azure DocumentDB

When an application needs to search several identifier-like fields on the same collection — `serialNumber`, `basicNumber`, `customerCode`, `partNumber`, `lineNumber` — the right pattern is **one** search index with multiple field mappings, not one index per field. Each field keeps its own analyzer pair, so the same index can simultaneously power prefix search on IDs, hierarchical lookup on lot numbers, and standard search on titles.

## Why one index, not many

| Per-field indexes | Multi-field index |
| --- | --- |
| Each new field adds an index build, storage cost, and write-amplification overhead. | One index covers every searchable field on the collection. |
| Reload requires coordinating multiple index rebuilds. | Reload is atomic — drop and recreate one index. |
| Adding a field to search means a new index. | Adding a field means a new entry in `mappings.fields`. |

Storage and write amplification scale with the number of indexes, so consolidating into one index that maps many fields is the operationally correct pattern.

## How multi-field indexes work

Each entry in `mappings.fields` declares its own type and analyzer pair. Fields with different "shapes" can coexist in one index:

- A prose field (`description`) using the default standard analyzer.
- An identifier field (`partNumber`) using `kw_lc_edge` at index time and `kw_lc` at query time for [prefix matching](full-text-search-custom-analyzers.md#pattern-2--prefix-matching-on-ids-and-skus).
- A hierarchical identifier (`basicNumber`) using a [`pathHierarchy` analyzer](full-text-search-path-hierarchy.md).

Define each analyzer once at the top of the index definition and reference it by name from each field that needs it.

## Creating a multi-field search index

```javascript
// ❌ One search index per field — multiplies build time, storage, and write amplification.
db.runCommand({ createSearchIndexes: "products_10M",
  indexes: [{ name: "idx_sn",
    definition: { mappings: { dynamic: false, fields: { serialNumber: { type: "string" } } } } }] });
db.runCommand({ createSearchIndexes: "products_10M",
  indexes: [{ name: "idx_bn",
    definition: { mappings: { dynamic: false, fields: { basicNumber: { type: "string" } } } } }] });
// ...one createSearchIndexes call per field, multiplied for every searchable field.
```

```javascript
// ✅ One index, five fields, one analyzer pair reused per field.
db.runCommand({
  createSearchIndexes: "products_10M",
  indexes: [
    {
      name: "idx_identifiers_multifield",
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
            serialNumber: { type: "string", analyzer: "kw_lc_edge", searchAnalyzer: "kw_lc" },
            basicNumber:  { type: "string", analyzer: "kw_lc_edge", searchAnalyzer: "kw_lc" },
            customerCode: { type: "string", analyzer: "kw_lc_edge", searchAnalyzer: "kw_lc" },
            partNumber:   { type: "string", analyzer: "kw_lc_edge", searchAnalyzer: "kw_lc" },
            lineNumber:   { type: "string", analyzer: "kw_lc_edge", searchAnalyzer: "kw_lc" }
          }
        }
      }
    }
  ]
});
```

## Querying one field at a time (current pattern)

`$search.compound` (server-side multi-clause `should` / `must` / `minimumShouldMatch`) is on the roadmap but is **not yet supported**. Today, target one field per `$search` call:

```javascript
// ✅ Single-field query against the multi-field index.
db.products_10M.aggregate([
  {
    $search: {
      index: "idx_identifiers_multifield",
      text: { query: "Crm", path: "customerCode" }
    }
  },
  { $limit: 20 },
  {
    $project: {
      _id: 0,
      customerCode: 1,
      basicNumber: 1,
      score: { $meta: "searchScore" }
    }
  }
]);
```

## Fan-out-and-merge (multi-field query workaround)

To search across every mapped field today, fan the query out per field and merge the result lists in the application layer, keeping the highest BM25 score per document:

```javascript
const fields = ["serialNumber", "basicNumber", "customerCode", "partNumber", "lineNumber"];
const results = new Map();

for (const path of fields) {
  const hits = await db.products_10M.aggregate([
    { $search: { index: "idx_identifiers_multifield", text: { query, path } } },
    { $limit: 50 },
    { $project: { _id: 1, [path]: 1, score: { $meta: "searchScore" } } }
  ]).toArray();

  for (const doc of hits) {
    const key = doc._id.toString();
    const prev = results.get(key);
    if (!prev || doc.score > prev.score) results.set(key, doc);
  }
}

const merged = [...results.values()]
  .sort((a, b) => b.score - a.score)
  .slice(0, 20);
```

The same pattern works for ranked-list fusion when one of the per-field queries is a phrase or fuzzy query — see [Reciprocal Rank Fusion (RRF)](full-text-search-hybrid.md#step-3reciprocal-rank-fusion-rrf) for a fusion variant that doesn't depend on raw BM25 scores being comparable across queries.

## Roadmap: `$search.compound`

```javascript
// ❌ compound is not yet supported on Azure DocumentDB — returns an error today.
db.products_10M.aggregate([
  { $search: {
      index: "idx_identifiers_multifield",
      compound: {
        should: [
          { text: { query: "<Q>", path: "serialNumber"  } },
          { text: { query: "<Q>", path: "basicNumber"   } },
          { text: { query: "<Q>", path: "customerCode"  } },
          { text: { query: "<Q>", path: "partNumber"    } },
          { text: { query: "<Q>", path: "lineNumber"    } }
        ],
        minimumShouldMatch: 1
      }
  }}
]);
```

When `$search.compound` ships, the fan-out loop above becomes a single server-side `should` clause with `minimumShouldMatch: 1`. The application code is deletable; the index definition stays the same. Designing your search index as a multi-field index today makes the eventual switch a one-line change in the query layer.

## Related pages

- [Custom analyzers (analyzer pairs used per field)](full-text-search-custom-analyzers.md)
- [Hierarchical identifier search (`pathHierarchy` tokenizer)](full-text-search-path-hierarchy.md)
- [Hybrid search (BM25 + vector)](full-text-search-hybrid.md)
- [Full-text search overview and migration table](full-text-search-overview.md)
