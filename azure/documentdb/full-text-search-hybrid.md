---
title: "Hybrid Search - Combining BM25 and Vector Retrieval"
titleSuffix: Azure DocumentDB 
description: Combine keyword and vector search in Azure DocumentDB to deliver higher recall and precision than either approach alone, on a single collection.
author: khelanmodi
ms.author: khelanmodi
ms.topic: how-to
ms.date: 05/07/2026
ms.collection:
  - ce-skilling-ai-copilot
---

# Hybrid search in Azure DocumentDB

> [!NOTE]
> Full-text search in Azure DocumentDB is in **Gated Preview**. To enable it on your cluster, contact us at [mongodb-feedback@microsoft.com](mailto:mongodb-feedback@microsoft.com).

Hybrid search runs a BM25 keyword query and a vector similarity query against the same collection and fuses the result lists into a single ranked list. It typically gives higher recall and precision than either mode alone, because each mode covers a failure case of the other. Azure DocumentDB supports both index types on the same cluster (search indexes for BM25 and `cosmosSearch` for vectors), providing native vector indexing alongside document data and enabling RAG and similarity search without introducing a separate vector store.

## Why hybrid search?

BM25 keyword search misses paraphrases and synonyms. A query for `water-resistant jacket` won't rank `waterproof coat` highly, even though they describe the same product. Vector search using sentence embeddings handles that gracefully (the two phrases are close in embedding space).

Vector search misses exact identifiers and rare terms. Embeddings don't preserve `SKU-4821-A` well, and a user typing the SKU expects the exact product, not the semantically nearest one. BM25 handles that case directly.

Running both queries in parallel and fusing the result lists captures the strengths of each: lexical exactness from BM25, semantic generalization from vectors. Reciprocal Rank Fusion (RRF) is the recommended fusion algorithm because it ignores raw scores (which aren't comparable across different scoring systems) and combines ranks instead.

## When to use hybrid search

> [!TIP]
> Reach for hybrid search when:
>
> - Your catalog or product search mixes natural-language queries with SKUs, part numbers, or other exact identifiers.
> - Your knowledge-base or documentation search needs synonym tolerance but should still match titles exactly.
> - Your RAG pipeline needs both lexical grounding (so rare entity names aren't lost) and semantic generalization (so paraphrased questions still find the right passages).

## Architecture: two indexes, one collection

Both indexes live on the same Azure DocumentDB cluster, on the same collection, against the same documents. There is no replication or synchronization layer to manage. Write a document once and both indexes pick it up. Both full-text and vector search are included with your cluster at no extra cost.

The two indexes use different commands:

- **BM25 search index:** `createSearchIndexes` (the new search engine).
- **Vector index:** `createIndex` with `cosmosSearch` and `cosmosSearchOptions` (the vector engine).

## Step 1: creating both indexes

```javascript
// ✅ BM25 query: createSearchIndexes (NOT createIndexes with "text").
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

// ✅ Vector query: DiskANN vector index for the embedding field.
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

## Step 2: running both queries

Each query runs as its own aggregation pipeline. The keyword query follows the standard `$search` rules from [BM25 keyword search](full-text-search-keyword.md): `index: "<name>"`, `$search` first, `$limit` downstream.

```javascript
// Assumes db is a connected MongoDB database (from MongoClient.connect().db("..."))
// and embed() is your embedding function (for example, an OpenAI client call).
const userQuery = "water-resistant jacket";

// Keyword query: BM25 hits with searchScore.
const kwHits = await db.products.aggregate([
  { $search: {
      index: "idx_description_fts",
      text: { query: userQuery, path: "description" }
  }},
  { $limit: 50 },
  { $project: { _id: 1, kw: { $meta: "searchScore" } } }
]).toArray();

// Vector query: DiskANN nearest-neighbor search.
const qv = await embed(userQuery);   // your embedding model of choice
const vecHits = await db.products.aggregate([
  { $search: { cosmosSearch: { path: "embedding", query: qv, k: 50 } } },
  { $project: { _id: 1, vec: { $meta: "searchScore" } } }
]).toArray();
```

## Step 3: Reciprocal Rank Fusion (RRF)

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

The same `rrf()` helper fuses any pair of ranked lists: fuzzy + phrase, or the keyword + vector combination shown here.

## Server-side RRF with `$unionWith`

When you'd rather keep fusion inside a single aggregation pipeline (no application-layer code, one round trip), combine the keyword and vector queries with `$unionWith`. Each query computes its own per-rank reciprocal contribution; a final `$group` sums them per document.

```javascript
// ✅ End-to-end hybrid search in one aggregation pipeline.
//    The vector query runs first; the $unionWith inlines the keyword query;
//    the final $group sums the per-query RRF contributions per document.
//    Reuses the userQuery and qv variables defined in Step 2.
const k = 60;     // RRF constant; 60 is a common default
const topN = 10;  // final result depth

db.products.aggregate([
  // --- Vector query ----------------------------------------------------------
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

  // --- Keyword query (inlined) ----------------------------------------------
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

Use the server-side variant when you want a single round-trip and no client-side fusion code. Use the [client-side `rrf()` helper](#step-3-reciprocal-rank-fusion-rrf) when you also want to fuse in additional ranked lists (phrase results or hits from a third retriever) without rewriting the pipeline each time.

## Tuning hybrid search

- **Keep per-query depth modest.** Set `$limit` for the keyword query and `k` for the vector query to 20–100. RRF doesn't benefit from deep lists; quality plateaus quickly past the top results from each query.
- **Weight the more reliable signal.** When one query consistently outperforms the other for your workload, weight its contribution: `score += w / (k + rank)` with `w` between 1.0 and 2.0 for the favored query. In the `$unionWith` variant, multiply the per-query `$divide` expression by the weight before the final `$group`.
- **Tune the RRF constant per query.** The `$unionWith` example uses the same `k` for both queries. Using a larger `k` for the keyword query (for example, `k = 60` for vector and `k = 10` for keyword) penalizes lower-ranked keyword hits more aggressively when the keyword query is noisier.
- **Choose the vector index kind for your scale.** DiskANN is the default for production catalogs with millions of vectors. HNSW gives lower-latency lookups at higher memory cost; IVF gives faster builds and lower memory cost at the price of recall. See [Vector search](vector-search.md) for the full matrix.
- **Cache embeddings for popular queries.** Vector query latency is dominated by the embedding API call, not the DiskANN lookup. Caching the embeddings for the most common queries cuts hybrid latency to roughly the keyword query's latency.

## Related pages

- [BM25 keyword search](full-text-search-keyword.md)
- [Full-text search overview and migration table](full-text-search-overview.md)

## Next step

> [!div class="nextstepaction"]
> [Create a lifetime free-tier cluster for Azure DocumentDB](free-tier.md)