---
title: Multi-Vector Search with MaxSim (Preview)
titleSuffix: Azure Cosmos DB for NoSQL
description: Learn how multi-vector search stores multiple vectors at a single embedding path and uses MaxSim to retrieve documents with token-level or passage-level relevance.
author: jcodella
ms.author: jacodel
ms.service: azure-cosmos-db
ms.subservice: nosql
ms.topic: concept-article
ms.date: 05/20/2026
ms.update-cycle: 180-days
ms.collection:
  - ce-skilling-ai-copilot
appliesto:
  - ✅ NoSQL
ai-usage: ai-assisted
---

# Multi-vector search (preview)

[!INCLUDE[Preview](../includes/notice-preview.md)]

Multi-vector search lets you store more than one vector at the same embedding path in an Azure Cosmos DB item. Instead of storing a single vector, the embedding path stores an array of vectors. This pattern is useful for retrieval approaches that represent one document with multiple embeddings, such as ColBERT-style embeddings.

Use multi-vector search when one item contains multiple semantic units that should be compared independently during retrieval. For example, a product, document, memory, or passage group might have one vector for each token, phrase, chunk, image region, or other subunit. Multi-vector search keeps those vectors with the source item while still returning the item as the search result.


> [!IMPORTANT]
> As multi-vector search is gradually rolling out across Azure regions, availability may vary, and the feature might not yet be accessible in your subscription or region. 

> [!NOTE]
> Currently only the quantizedFlat vector index type is supported for multi-vector search. Support for DiskANN indexes is planned for the future.

## SDK support

Multi-vector search is supported in the latest Azure Cosmos DB SDKs for .NET, Python, and Java. For .NET, use the latest preview version of the Azure Cosmos DB .NET SDK.

## How multivectors are stored

A standard vector path stores one vector as an array of numbers. A multivector path stores an array of vectors at the same path.

```json
{
  "id": "doc-001",
  "title": "Contoso hiking backpack",
  "content": "A lightweight backpack for multiday hiking trips.",
  "embedding": [
    [0.12, -0.08, 0.44, 0.31],
    [0.02, 0.19, -0.21, 0.07],
    [0.38, -0.16, 0.09, 0.24]
  ]
}
```

In this example, `/embedding` is the embedding path. Each nested array is an individual vector for the same item.

## MaxSim distance function

`MaxSim` is the distance function used for multi-vector comparison. It compares a query multivector with the vectors stored on an item and uses the strongest matching vector-level similarities to score the item.

This behavior is useful for late-interaction retrieval models. In ColBERT-style retrieval, each document is represented by multiple vectors instead of a single pooled embedding. `MaxSim` helps preserve fine-grained semantic matching because individual query vectors can match different vectors in the stored document representation.

## Configure a vector policy for multivectors

Configure the vector embedding path to point to the property that contains the array of vectors. The `dimensions` value describes each individual vector in the multivector array.

```json
{
  "vectorEmbeddings": [
    {
      "path": "/embedding",
      "dataType": "float32",
      "distanceFunction": "MaxSim",
      "dimensions": 4
    }
  ]
}
```

After you configure the vector policy and vector index, insert items whose embedding path contains an array of vectors. Query-time vectors should use the same dimensionality as the vectors stored in the item.

## Query with multi-vector search

Query multi-vector data the same way you query regular vector data: use the [`VectorDistance`](/cosmos-db/query/vectordistance) system function in a query, project the similarity score if you need it, and sort by `VectorDistance` to return the most similar items first.

For multi-vector search, the indexed path contains an array of vectors and the query argument is a vector from the same dimensionality and embedding model. Each nested vector in the query should have the same dimensionality as the vectors in the indexed path.

```nosql
SELECT TOP 10
  c.title,
  VectorDistance(c.embedding, [[0.12, -0.08, 0.44, 0.31], [0.02, 0.19, -0.21, 0.07]]) AS MaxSimScore
FROM c
ORDER BY VectorDistance(c.embedding, [[0.12, -0.08, 0.44, 0.31], [0.02, 0.19, -0.21, 0.07]])
```

> [!IMPORTANT]
> Always use a `TOP N` clause in the `SELECT` statement. Without `TOP N`, the query can try to return more results than your application needs, which can increase request unit (RU) consumption and latency.

## When to use multi-vector search

Use multi-vector search for retrieval workloads where a single vector loses too much detail.

Common scenarios include:

- ColBERT-style text embeddings that keep token-level or phrase-level vectors.
- Long documents where different passages carry different meaning.
- Product catalogs where one item has multiple descriptions, attributes, or modalities.
- Agent memory stores where one memory item contains multiple turns, observations, or extracted facts.
- Multimodal retrieval where a single item contains multiple text, image, or other embedding representations.

## Design considerations

Multi-vector search can improve retrieval quality for workloads that benefit from fine-grained matching, but it also changes storage and query behavior.

| Design area | Consideration |
| --- | --- |
| Item size | Multiple vectors increase item size. Keep items below Azure Cosmos DB item size limits. |
| Dimensions | Each nested vector in the multivector array should use the same dimensionality. |
| Model alignment | Use the same embedding model family for stored vectors and query vectors. |
| Indexing | Configure the vector policy and vector index before you load production data. |
| Cost and latency | Test retrieval quality, RU consumption, and latency with representative data. |

## Related content

- [Vector search in Azure Cosmos DB for NoSQL](../vector-search.md)
- [Distance functions](distance-functions.md)
- [Hybrid search in Azure Cosmos DB for NoSQL](hybrid-search.md)
- [`VectorDistance` system function](/cosmos-db/query/vectordistance)
- [Vector embeddings in Azure Cosmos DB](vector-embeddings.md)
