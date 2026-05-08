---
title: "Full-Text Search in Azure DocumentDB — BM25 keyword and hybrid retrieval"
description: Run BM25-scored, analyzer-driven full-text search natively inside Azure DocumentDB using the createSearchIndexes command and the $search aggregation stage.
author: khelanmodi
ms.author: khelanmodi
ms.topic: concept-article
ms.date: 05/07/2026
ms.collection:
  - ce-skilling-ai-copilot
---

# Full-Text Search in Azure DocumentDB

Azure DocumentDB full-text search is a BM25-scored, analyzer-driven keyword search engine exposed through MongoDB-compatible primitives. It ranks results by relevance, supports custom tokenization and token filters, and tolerates user typos through fuzzy matching — without standing up a separate search cluster. It supersedes the legacy `$text` operator and `{ field: "text" }` index type that were backed by a PostgreSQL TSVector implementation. Search indexes, document indexes, and vector indexes all live on the same Azure DocumentDB cluster, so keyword, semantic, and hybrid retrieval share one operational footprint.

## What's new in this release

| Capability | Operator or primitive |
| --- | --- |
| BM25 keyword search | `$search` + `text` |
| Phrase search and proximity matching | `$search` + `phrase` + `slop` |
| Fuzzy (typo-tolerant) search | `$search` + `text` with `fuzzy.maxEdits` |
| Custom analyzers (case-insensitive, prefix matching) | `keyword` / `lowerCase` / `asciiFolding` / `edgeGram` |
| Hierarchical identifier search | `pathHierarchy` tokenizer |
| Multi-field search index | One index, many `mappings.fields` entries |
| Hybrid keyword + vector retrieval | `$search` + `text` + `cosmosSearch` + RRF |

## How Azure DocumentDB full-text search works

A search index in Azure DocumentDB is a separate object from a document index. You create it with the `createSearchIndexes` database command — not with `db.<coll>.createIndex({ field: "text" })` and not with `createIndexes` and a `"textSearch"` key type. The index definition declares which fields are searchable, their types, and the analyzer pipeline used at index time and query time.

Queries run as the first stage of an aggregation pipeline using the `$search` operator. The engine returns documents ordered by BM25 relevance, which you can read through `{ $meta: "searchScore" }` in a downstream `$project` stage. Examples in this section use mongosh-style MongoDB Shell syntax, but the same operations work from any MongoDB-compatible driver.

Four rules apply to every Azure DocumentDB full-text search query:

- **Always target an index by name** with `index: "<name>"` inside `$search`. The engine does not auto-pick when more than one search index exists on the collection.
- **`$search` is always the first stage** of an aggregation pipeline so the search index narrows the candidate set before any other operator runs.
- **There is no `count` or `limit` field inside `$search`.** Cap results with a downstream `{ $limit: N }` stage.
- **Equality and range filters belong in a downstream `$match`,** not inside `$search`. Keeping `$search` index-pure preserves BM25 scoring and avoids unintentional rescans.

Set `dynamic: false` on every index definition and enumerate fields explicitly. `dynamic: true` indexes every string field in the collection and inflates index size unpredictably.

The `$search` + `compound` operator (server-side `should` / `must` / `minimumShouldMatch` clauses across multiple fields) is on the roadmap but is not yet supported. Until it ships, query one field at a time and merge results in the application layer — the [multi-field search index](full-text-search-multifield-index.md) page documents the fan-out-and-merge pattern.

## When to use Azure DocumentDB full-text search

| Mode | Best for | Requires search index | Ranked by BM25 |
| --- | --- | :---: | :---: |
| `$regex` | Cheap exact-substring match on a single field with a pre-existing B-tree index. | No | No |
| `$search` + `text` | Standard keyword search on prose, descriptions, titles. | Yes | Yes |
| `$search` + `text` + `fuzzy` | Search-as-you-type, user-facing catalog or log search where typos are common. | Yes | Yes |
| `$search` + `phrase` | Multi-word product names, quoted user input, error strings where word order matters. | Yes | Yes |
| `edgeGram` custom analyzer | Prefix matching on identifiers, SKUs, part numbers, short codes. | Yes | Yes |
| `pathHierarchy` tokenizer | Hierarchical IDs (`BN-747-ENG-2024.05`, `/regions/eu/ops`, `com.example.svc`). | Yes | Yes |
| Hybrid (BM25 + vector) | Catalog or knowledge-base search that mixes natural-language queries with exact identifiers; RAG retrieval. | Yes (BM25 + vector) | Fused score |

## Migrating from the legacy `$text` engine

If your application uses the community MongoDB `$text` operator or `{ field: "text" }` index type today, migrate to the new engine using the table below. Capabilities that the legacy engine listed as **Not available** — fuzzy search, proximity, custom analyzers, faceting — are now first-class with the new `$search` engine.

| Legacy feature | Legacy operator | New equivalent | Reference |
| --- | --- | --- | --- |
| Term-based search | `$text: { $search: "..." }` | `$search` + `text` (BM25-ranked) | [BM25 keyword search](full-text-search-bm25-keyword.md) |
| Phrase search | `$text: { $search: "\"a b\"" }` | `$search` + `phrase` with `slop` | [Phrase search](full-text-search-phrase-proximity.md) |
| Prefix query | `$regex: /^pattern/` | `edgeGram` custom analyzer + `$search` + `text` | [Custom analyzers](full-text-search-custom-analyzers.md) |
| Fuzzy search | *Not available* | `$search` + `text` + `fuzzy.maxEdits` | [Fuzzy search](full-text-search-fuzzy.md) |
| Proximity search | *Not available* | `$search` + `phrase` + `slop` | [Phrase search](full-text-search-phrase-proximity.md) |
| Custom analyzers | *Not available* | `createSearchIndexes` with custom `analyzers` pipeline | [Custom analyzers](full-text-search-custom-analyzers.md) |
| Multi-match / multi-field | `createIndex({ a: "text", b: "text" })` with weights | Multi-field search index + fan-out-and-merge | [Multi-field search index](full-text-search-multifield-index.md) |
| Faceted search | *Not available* | `$match` + `$group` downstream of `$search` | [BM25 keyword search](full-text-search-bm25-keyword.md) |
| Boolean operators | `$text: { $search: "+a -b" }` | `$search` + `text`; server-side `should` / `must` ships with `$search` + `compound` (roadmap) | [Multi-field search index](full-text-search-multifield-index.md) |
| Wildcard / regex | `$regex` | `$regex` still works for substring patterns but is unranked and forces a `COLLSCAN` on text fields; prefer `$search` | [BM25 keyword search](full-text-search-bm25-keyword.md) |
| Language stemming | `default_language: "en"` | Standard analyzer covers common cases; extend with custom token filters | [Custom analyzers](full-text-search-custom-analyzers.md) |
| Synonym support | *Not available* | Not yet supported natively; planned | — |

The migration story is straightforward: every capability the legacy engine offered remains available, and most of what it could not do — fuzzy matching, proximity, custom analyzers, hierarchical identifier search, hybrid retrieval — is now part of the same `$search` surface area.

## Limitations and roadmap

A few capabilities aren't yet supported natively. Most have a documented workaround you can use today.

| Capability | Status | Workaround |
| --- | --- | --- |
| `$search` + `compound` (server-side `should` / `must` / `minimumShouldMatch` across multiple fields) | Roadmap | Fan-out per field and merge in the application layer. See [Multi-field search index](full-text-search-multifield-index.md#fan-out-and-merge-multi-field-query-workaround). |
| Combining `$search` + `phrase` with `fuzzy` in a single clause | Not supported | Run a phrase query and a fuzzy query separately and fuse the result lists with [Reciprocal Rank Fusion](full-text-search-hybrid.md#step-3-reciprocal-rank-fusion-rrf). |
| Query-time term boosting (per-clause weight in `$search`) | Roadmap (ships with `$search` + `compound`) | Use BM25's natural inverse-document-frequency weighting; apply weighting at fusion time in client code or `$unionWith`. |
| Native autocomplete operator | Not yet exposed | Implement type-ahead with the [`edgeGram` custom analyzer](full-text-search-custom-analyzers.md#pattern-2--prefix-matching-on-ids-and-skus). |
| Faceted search as a first-class operator | Not yet exposed | `$match` and `$group` downstream of `$search` produce facet counts. See [BM25 keyword search](full-text-search-bm25-keyword.md#combining-with-filters). |
| Synonym support | Roadmap | Expand synonyms client-side and `OR` the expanded terms into the query string until native support ships. |

## Next steps

- [BM25 keyword search](full-text-search-bm25-keyword.md)
- [Fuzzy search](full-text-search-fuzzy.md)
- [Phrase search and proximity matching](full-text-search-phrase-proximity.md)
- [Custom analyzers](full-text-search-custom-analyzers.md)
- [Hierarchical identifier search](full-text-search-path-hierarchy.md)
- [Multi-field search index](full-text-search-multifield-index.md)
- [Hybrid search (BM25 + vector)](full-text-search-hybrid.md)


## Next step

> [!div class="nextstepaction"]
> [Create a lifetime free-tier cluster for Azure DocumentDB](free-tier.md)