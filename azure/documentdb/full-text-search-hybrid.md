---
title: "Hybrid search in Azure DocumentDB — combining BM25 and vector retrieval"
description: Combine BM25 keyword search and DiskANN vector search on a single Azure DocumentDB collection, then fuse the result lists with Reciprocal Rank Fusion (RRF) for higher recall and precision than either alone.
author: khelanmodi
ms.author: khelanmodi
ms.topic: how-to
ms.date: 05/07/2026
ms.collection:
  - ce-skilling-ai-copilot
---

# Hybrid Search in Azure DocumentDB

Hybrid search runs a BM25 keyword query and a vector similarity query against the same collection and fuses the result lists into a single ranked list. It typically gives higher recall and precision than either mode alone, because each mode covers a failure case of the other. Azure DocumentDB supports both index types on the same cluster — search indexes for BM25 and `cosmosSearch` for vectors — providing native vector indexing alongside document data, enabling RAG and similarity search without introducing a separate vector store.

## Why hybrid search?

BM25 keyword search misses paraphrases and synonyms. A query for `water-resistant jacket` won't rank `waterproof coat` highly, even though they describe the same product. Vector search using sentence embeddings handles that gracefully — the two phrases are close in embedding space.

Vector search misses exact identifiers and rare terms. Embeddings don't preserve `SKU-4821-A` well, and a user typing the SKU expects the exact product, not the semantically nearest one. BM25 handles that case directly.

Running both arms in parallel and fusing the result lists captures the strengths of each: lexical exactness from BM25, semantic generalization from vectors. Reciprocal Rank Fusion (RRF) is the recommended fusion algorithm because it ignores raw scores — which aren't comparable across different scoring systems — and combines ranks instead.

## When to use hybrid search

> [!TIP]
> Reach for hybrid search when:
>
> - Your catalog or product search mixes natural-language queries with SKUs, part numbers, or other exact identifiers.
> - Your knowledge-base or documentation search needs synonym tolerance but should still match titles exactly.
> - Your RAG pipeline needs both lexical grounding (so rare entity names aren't lost) and semantic generalization (so paraphrased questions still find the right passages).

## Architecture: two indexes, one collection

Both indexes live on the same Azure DocumentDB cluster, on the same collection, against the same documents. There is no replication or synchronization layer to manage — write a document once and both indexes pick it up.

The two indexes use different commands:

- **BM25 search index** — `createSearchIndexes` (the new search engine).
- **Vector index** — `createIndex` with `cosmosSearch` and `cosmosSearchOptions` (the vector engine).

## Step 1 — creating both indexes

```javascript
// ✅ BM25 keyword arm — createSearchIndexes (NOT createIndexes with "text").
db.runCommand({
  createSearchIndexes: "products",
  indexes: [
    {
      name: "idx_description_fts",
      definition: {
        mappings: {
          dynamic: false,
          fields: {
            description: { type: "string" }
          }
        }
      }
    }
  ]
});

// ✅ Vector arm — DiskANN vector index for the embedding field.
//    Swap "vector-diskann" for "vector-hnsw" or "vector-ivf" to use the
//    HNSW or IVF index kinds; cosine, L2, and inner-product similarities
//    are all supported. See vector-search.md for the full option matrix.
db.products.createIndex(
  { embedding: "cosmosSearch" },
  {
    name: "desc_diskann",
    cosmosSearchOptions: {
      kind: "vector-diskann",
      dimensions: 1536,
      similarity: "COS"
    }
  }
);
```

## Step 2 — querying both arms

Each arm runs as its own aggregation pipeline. The keyword arm follows the standard `$search` rules from [BM25 keyword search](full-text-search-bm25-keyword.md) — `index: "<name>"`, `$search` first, `$limit` downstream.

```javascript
// Keyword arm — BM25 hits with searchScore.
const kwHits = await db.products.aggregate([
  { $search: {
      index: "idx_description_fts",
      text: { query: userQuery, path: "description" }
  }},
  { $limit: 50 },
  { $project: { _id: 1, kw: { $meta: "searchScore" } } }
]).toArray();

// Vector arm — DiskANN nearest-neighbor search.
const qv = await embed(userQuery);   // your embedding model of choice
const vecHits = await db.products.aggregate([
  { $search: { cosmosSearch: { path: "embedding", query: qv, k: 50 } } },
  { $project: { _id: 1, vec: { $meta: "searchScore" } } }
]).toArray();
```

## Step 3 — Reciprocal Rank Fusion (RRF)

RRF assigns each document a fused score of `1 / (k + rank)` from each list it appears in, sums those contributions across lists, and ranks by the total. A typical value of `k` is 60. Documents that appear high in either list get a strong contribution; documents that appear in both get added contributions.

```javascript
// ✅ Reciprocal Rank Fusion across an arbitrary number of ranked lists.
function rrf(lists, k = 60) {
  const scores = new Map();
  for (const list of lists) {
    list.forEach((doc, rank) => {
      const id = doc._id.toString();
      const cur = scores.get(id) ?? 0;
      scores.set(id, cur + 1 / (k + rank + 1));
    });
  }
  return [...scores.entries()]
    .sort((a, b) => b[1] - a[1])
    .slice(0, 10)
    .map(([id, score]) => ({ _id: id, score }));
}

const fused = rrf([kwHits, vecHits]);
```

The same `rrf()` helper fuses any pair of ranked lists — fuzzy + phrase, multi-field fan-out results, or the keyword + vector combination shown here.

## Server-side RRF with `$unionWith`

When you'd rather keep fusion inside a single aggregation pipeline — no application-layer code, one round trip — combine the keyword and vector arms with `$unionWith`. Each arm computes its own per-rank reciprocal contribution; a final `$group` sums them per document.

```javascript
// ✅ End-to-end hybrid search in one aggregation pipeline.
//    The vector arm runs first; the $unionWith inlines the keyword arm;
//    the final $group sums the per-arm RRF contributions per document.
const k = 60;     // RRF constant; 60 is a common default
const topN = 10;  // final result depth

db.products.aggregate([
  // --- Vector arm ----------------------------------------------------------
  { $search: { cosmosSearch: { path: "embedding", query: qv, k: 50 } } },
  { $group: { _id: null, hits: { $push: "$$ROOT" } } },
  { $unwind: { path: "$hits", includeArrayIndex: "rank" } },
  {
    $project: {
      _id: "$hits._id",
      title: "$hits.title",
      rrf: { $divide: [1, { $add: ["$rank", k, 1] }] }
    }
  },

  // --- Keyword arm (inlined) ----------------------------------------------
  {
    $unionWith: {
      coll: "products",
      pipeline: [
        { $search: {
            index: "idx_description_fts",
            text: { query: userQuery, path: "description" }
        }},
        { $limit: 50 },
        { $group: { _id: null, hits: { $push: "$$ROOT" } } },
        { $unwind: { path: "$hits", includeArrayIndex: "rank" } },
        {
          $project: {
            _id: "$hits._id",
            title: "$hits.title",
            rrf: { $divide: [1, { $add: ["$rank", k, 1] }] }
          }
        }
      ]
    }
  },

  // --- Fuse ----------------------------------------------------------------
  {
    $group: {
      _id: "$_id",
      title: { $first: "$title" },
      score: { $sum: "$rrf" }
    }
  },
  { $sort: { score: -1 } },
  { $limit: topN }
]);
```

Use the server-side variant when you want a single round-trip and no client-side fusion code. Use the [client-side `rrf()` helper](#step-3reciprocal-rank-fusion-rrf) when you also want to fuse in additional ranked lists — phrase results, per-field fan-out results, or hits from a third retriever — without rewriting the pipeline each time.

## Tuning hybrid search

- **Keep per-arm depth modest.** Set `$limit` for the keyword arm and `k` for the vector arm to 20–100. RRF doesn't benefit from deep lists; quality plateaus quickly past the top results from each arm.
- **Weight the more reliable signal.** When one arm consistently outperforms the other for your workload, weight its contribution: `score += w / (k + rank)` with `w` between 1.0 and 2.0 for the favored arm. In the `$unionWith` variant, multiply the per-arm `$divide` expression by the weight before the final `$group`.
- **Tune the RRF constant per arm.** The `$unionWith` example uses the same `k` for both arms. Using a larger `k` for the keyword arm (for example, `k = 60` for vector and `k = 10` for keyword) penalizes lower-ranked keyword hits more aggressively when the keyword arm is noisier.
- **Choose the vector index kind for your scale.** DiskANN is the default for production catalogs with millions of vectors. HNSW gives lower-latency lookups at higher memory cost; IVF gives faster builds and lower memory cost at the price of recall. See [Vector search](vector-search.md) for the full matrix.
- **Cache embeddings for popular queries.** Vector arm latency is dominated by the embedding API call, not the DiskANN lookup. Caching the embeddings for the most common queries cuts hybrid latency to roughly the keyword arm's latency.

## Roadmap: `$search.compound` and a multi-field keyword arm

When `$search.compound` ships, the keyword arm of a hybrid query can cover multiple fields server-side instead of fanning out per field. Today, if you want the BM25 arm to span `title`, `description`, and `tags`, follow the [fan-out-and-merge pattern](full-text-search-multifield-index.md#fan-out-and-merge-multi-field-query-workaround) and pass each per-field result list into `rrf()` alongside the vector list. The fusion math doesn't change; only the keyword arm collapses to a single server-side query.

## Related pages

- [BM25 keyword search](full-text-search-bm25-keyword.md)
- [Multi-field search index](full-text-search-multifield-index.md)
- [Custom analyzers](full-text-search-custom-analyzers.md)
- [Full-text search overview and migration table](full-text-search-overview.md)
