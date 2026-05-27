---
title: Full-Text Search Overview
titleSuffix: Azure DocumentDB
description: Use full-text search in Azure DocumentDB to deliver relevance-ranked keyword, fuzzy, phrase, and hybrid search natively, without standing up a separate search service.
author: khelanmodi
ms.author: khelanmodi
ms.topic: concept-article
ms.date: 05/07/2026
ms.collection:
  - ce-skilling-ai-copilot
---

# Full-text search in Azure DocumentDB

> [!NOTE]
> Full-text search in Azure DocumentDB is in **Gated Preview**. To enable it on your cluster, contact us at [mongodb-feedback@microsoft.com](mailto:mongodb-feedback@microsoft.com).

Azure DocumentDB full-text search is a BM25-scored, analyzer-driven keyword search engine exposed through MongoDB-compatible primitives. It ranks results by relevance and tolerates user typos through fuzzy matching, without standing up a separate search cluster. It supersedes the legacy `$text` operator and `{ field: "text" }` index type that were backed by a PostgreSQL TSVector implementation.

## What's available today

| Capability | Operator or primitive |
| --- | --- |
| BM25 keyword search | `$search` + `text` |
| Fuzzy (typo-tolerant) search | `$search` + `text` with `fuzzy.maxEdits` |
| Phrase search and proximity matching | `$search` + `phrase` + `slop` |
| Hybrid keyword + vector retrieval | `$search` + `text` + `cosmosSearch` + RRF |

## Coming soon

The following capabilities are in active development and ship in upcoming releases:

- Custom analyzers (case-insensitive search, `edgeGram` prefix matching on IDs and SKUs).
- `pathHierarchy` tokenizer for hierarchical identifier search.
- Multi-field search index (one index, many `mappings.fields` entries).
- `$search` + `compound` for server-side `should` / `must` / `minimumShouldMatch` clauses across multiple fields.

## How Azure DocumentDB full-text search works

A search index in Azure DocumentDB is a separate object from a document index. You create it with the `createSearchIndexes` database command, not with `db.<coll>.createIndex({ field: "text" })` and not with `createIndexes` and a `"textSearch"` key type. The index definition declares which fields are searchable, their types, and the analyzer pipeline used at index time and query time.

Queries run as the first stage of an aggregation pipeline using the `$search` operator. The engine returns documents ordered by BM25 relevance, which you can read through `{ $meta: "searchScore" }` in a downstream `$project` stage. Examples in this section use mongosh-style MongoDB Shell syntax, but the same operations work from any MongoDB-compatible driver.

Four rules apply to every Azure DocumentDB full-text search query:

- **Always target an index by name** with `index: "<name>"` inside `$search`. The engine does not auto-pick when more than one search index exists on the collection.
- **`$search` is always the first stage** of an aggregation pipeline so the search index narrows the candidate set before any other operator runs.
- **There is no `count` or `limit` field inside `$search`.** Cap results with a downstream `{ $limit: N }` stage.
- **Equality and range filters belong in a downstream `$match`,** not inside `$search`. Keeping `$search` index-pure preserves BM25 scoring and avoids unintentional rescans.

Set `dynamic: false` on every index definition and enumerate fields explicitly. `dynamic: true` indexes every string field in the collection and inflates index size unpredictably.

## When to use Azure DocumentDB full-text search

| Mode | Best for | Requires search index | Ranked by BM25 |
| --- | --- | :---: | :---: |
| `$regex` | Cheap exact-substring match on a single field with a preexisting B-tree index. | No | No |
| `$search` + `text` | Standard keyword search on prose, descriptions, titles. | Yes | Yes |
| `$search` + `text` + `fuzzy` | Search-as-you-type, user-facing catalog or log search where typos are common. | Yes | Yes |
| `$search` + `phrase` | Multi-word product names, quoted user input, error strings where word order matters. | Yes | Yes |
| Hybrid (BM25 + vector) | Catalog or knowledge-base search that mixes natural-language queries with exact identifiers; RAG retrieval. | Yes (BM25 + vector) | Fused score |

## Migrating from the legacy `$text` engine

If your application uses the community MongoDB `$text` operator or `{ field: "text" }` index type today, migrate to the new engine using the table below. Fuzzy and proximity capabilities that the legacy engine listed as **Not available** are now first-class with the new `$search` engine.

| Legacy feature | Legacy operator | New equivalent | Reference |
| --- | --- | --- | --- |
| Term-based search | `$text: { $search: "..." }` | `$search` + `text` (BM25-ranked) | [BM25 keyword search](full-text-search-keyword.md) |
| Phrase search | `$text: { $search: "\"a b\"" }` | `$search` + `phrase` with `slop` | [Phrase search](full-text-search-phrase-proximity.md) |
| Fuzzy search | *Not available* | `$search` + `text` + `fuzzy.maxEdits` | [Fuzzy search](full-text-search-fuzzy.md) |
| Proximity search | *Not available* | `$search` + `phrase` + `slop` | [Phrase search](full-text-search-phrase-proximity.md) |
| Wildcard / regex | `$regex` | `$regex` still works for substring patterns but is unranked and forces a `COLLSCAN` on text fields; prefer `$search` | [BM25 keyword search](full-text-search-keyword.md) |

## Related pages

- [BM25 keyword search](full-text-search-keyword.md)
- [Fuzzy search](full-text-search-fuzzy.md)
- [Phrase search and proximity matching](full-text-search-phrase-proximity.md)
- [Hybrid search (BM25 + vector)](full-text-search-hybrid.md)

## Next step

> [!div class="nextstepaction"]
> [Create a lifetime free-tier cluster for Azure DocumentDB](free-tier.md)