---
title: Tips for optimizing vector indexing & search performance
description: Performance tuning and best practices for vector search in Azure Cosmos DB for NoSQL
author: jcodella
ms.author: jacodel
ms.service: azure-cosmos-db
ms.subservice: nosql
ms.topic: concept-article
ms.date: 03/23/2026
appliesto:
  - ✅ NoSQL
---

# Tips for optimizing vector indexing & search performance

This guide covers practical optimization strategies for semantic search in Azure Cosmos DB for NoSQL. It focuses on the decisions that most affect cost, latency, and recall: embedding shape and precision, index selection, partitioning, throughput sizing, ingestion patterns, SDK concurrency, and query-time tuning.

## Things to consider

Most teams get the best results by making decisions in this order:

1. Choose the embedding model, dimension count, and numeric precision. Azure OpenAI models are a common choice for text embeddings.
2. Choose the vector index type for your expected dataset size and query shape.
3. Choose a partitioning strategy that balances your workloads to spread CRUD operations across partitions while still allowing vector search queries to be scoped to a single or small set of partitions.
4. Plan throughput and provisioning for both ingestion and search.
5. Optimize ingestion with bulk mode or concurrency patterns that match your SDK.
6. Configure SDK query concurrency and latency settings based on your query patterns.
7. Tune query-time controls such as `searchListSizeMultiplier`, `quantizedVectorListMultiplier`, and `filterPriority` as needed to balance the trade-off between recall, latency, and RU cost.
8. Measure RU consumption, latency, and throttling under realistic load, then iterate.

## Quick reference: Best practices

### Tuning vector dimensions

Many modern embedding models let you configure the output dimension of the vector. For example, `text-embedding-3-large` supports dimensions ranging from 256 up to 3072. Higher dimensions can capture richer semantic meaning and improve recall and relevance, but they also increase storage size and can raise RU cost slightly for inserts and searches.

| Dimensions | Pros | Cons |
| ----------- | ------ | ------ |
| Lower (for example, 256-512) | Smaller storage footprint, lower insert, and search RU cost, faster queries | Can miss finer semantic distinctions |
| Higher (for example, 1536-3072) | Captures more nuanced meaning, higher recall, and relevance | Larger storage, slightly higher RU cost per insert and search |

Start with the model's default dimension. If recall or relevance is insufficient, increase dimensions. If cost or latency is a concern and recall is already high, reduce dimensions.

### Reducing vector precision: float16 and int8

You can store vectors at lower precision than the model's native `float32` output by setting the `dataType` in the vector embedding policy to `float16` or `int8`. This trades recall and relevance for storage and cost savings:

| Data type | Storage vs. float32 | Impact on recall | Best for |
| ----------- | --------------------- | ----------------- | ---------- |
| `float32` | 1x (baseline) | None - full precision | Read-heavy workloads where recall is critical |
| `float16` | 0.5x | Small for most text workloads | General-purpose cost optimization |
| `int8` | 0.25x | Moderate - noticeable for some queries | **Write-heavy workloads** where insert cost matters most |

### Improving recall at query time

When using lower-precision data types (for example, `float16`, `int8`) or choosing smaller vector dimension sizes, you can often recover some recall by increasing query-time candidate list multipliers. This raises search RU cost and latency, but it can still be a good overall tradeoff for write-heavy systems that care more about ingestion cost than absolute per-query efficiency.

The following SQL example shows how to increase both query-time multipliers in a vector search query. Use this pattern when you want to trade additional RU cost for improved recall with lower-precision vectors.

```sql
SELECT TOP 10 c.title, c.name
FROM c
ORDER BY VectorDistance(c.embedding, @embedding, false, {
    searchListSizeMultiplier:10,
    quantizedVectorListMultiplier:10
})
```

The trade-off is straightforward:

- Higher `searchListSizeMultiplier` values improve DiskANN recall but increase query RU cost.
- Higher `quantizedVectorListMultiplier` values improve recall for quantized candidate evaluation but also increase query RU cost.
- If a filtered DiskANN query is returning weak results, `filterPriority` lets you change the balance between the query filter and the vector search itself.

This pattern is especially useful for **write-heavy vector workloads** where you want smaller vectors and cheaper ingestion, then use query-time multipliers to recover some recall.

| Query-time option | Typical values | Effect |
| ------------------ | ---------------- | -------- |
| `searchListSizeMultiplier` | `5`, `10`, `20` | Improves DiskANN recall at higher RU and latency |
| `quantizedVectorListMultiplier` | `5`, `10`, `20` | Improves recall for quantized candidate lists at higher RU and latency |
| `filterPriority` | `0.0`, `0.1`, `0.5`, `1.0` | Changes the balance between the `WHERE` filter and the vector search in filtered DiskANN workloads |

For most applications, `float16` is the first precision reduction to try. Move to `int8` only after validating recall on your own benchmark set.

## Choosing a vector index type

Azure Cosmos DB for NoSQL offers three vector index types. Choosing the right one for your scenarios is the highest-impact optimization decision for vector search.

> [!NOTE]
> You can always add new path configurations or remove existing ones, but you cannot change the settings of a vector embedding policy or vector indexing policy directly. To do so you must first drop the existing vector policy and/or index, then add it back with new configuration. 


### Comparison matrix

| Characteristic | `flat` | `quantizedFlat` | `diskANN` |
| ---------------- | -------- | ----------------- | ----------- |
| Max dimensions | 505 | 4,096 | 4,096 |
| Recall (accuracy) | 100 % (exact kNN) | ~95-100 % (tunable) | ~90-100 % (tunable)|
| Latency | High for large datasets | Balanced, best for small or scoped data | **Lowest** for large data |
| RU cost per search | **Highest** at scale | Balanced | **Lowest** at scale |
| Sweet spot | < few thousand small vectors | < ~50K vectors per search scope or partition | > 50K vectors per search scope or partition |

### When to use `quantizedFlat`

- Most multitenant workloads with moderate dataset sizes per tenant, especially when using hierarchical partition keys.
- Moderate-sized datasets (up to ~50K vectors in total or scoped via partition key filters, such as multitenant scenarios).
- You want better latency and RU cost vs. `flat` at scale. 
- Scenarios where you regularly apply `WHERE` filters that narrow the search space.

### When to use `diskANN`

- **Most document search production workloads** - especially when your search is over more than 50K vectors even with scoping to one or a few partitions.
- Multitenant scenarios when each tenant can reach a high number of vectors (> 50K) and you want to use hierarchical partition keys with sharded DiskANN for tenant isolation and performance.
- You need the lowest latency and lowest RU cost per query.
- Multi-million or billion-scale vector datasets.

The following JSON example shows a basic `diskANN` vector index configuration for a container.

```json
// DiskANN index configuration
{
    "indexingMode": "consistent",
    "automatic": true,
    "includedPaths": [{ "path": "/*" }],
    "excludedPaths": [
        { "path": "/_etag/?" }
    ],
    "vectorIndexes": [
        {
            "path": "/embedding",
            "type": "diskANN"
        }
    ]
}
```

### Quantization options for DiskANN and quantizedFlat

Both `diskANN` and `quantizedFlat` support optional quantization parameters to tune the accuracy-performance trade-off:

The following JSON example shows how to set quantization and indexing parameters on a vector index definition. Use this when you want to tune the balance between recall, index size, and ingestion cost.

```json
{
    "vectorIndexes": [
        {
            "path": "/embedding",
            "type": "diskANN",
            "quantizerType": "product",
            "quantizationByteSize": 64,
            "indexingSearchListSize": 100
        }
    ]
}
```

| Parameter | Default | Description |
| ----------- | --------- | ------------- |
| `quantizationByteSize` | Dynamic (system decided) | Number of bytes used for the quantized vector. Lower = smaller index / faster search at the cost of recall. Range: 1-512. |
| `indexingSearchListSize` | 100 (`diskANN`) | Size of the candidate list during DiskANN index build. Higher = better recall at the cost of slower builds and ingestion. Range: 10-500. |
| `vectorIndexShardKey` | (none) | Partition-scoped DiskANN index. See [Sharded DiskANN](#sharded-diskann-and-partition-key-design). |
| `quantizerType` | `product` | (Optional) the method used for quantizing vectors. See details below. |

You can also optionally configure a `quantizerType` within each vectorIndexes entry. This controls how vectors are quantized prior to indexing.
  - **product** (default)
Uses standard product quantization. Provides balanced performance and accuracy for most workloads.
  - **spherical** (public preview)
This quantization method can improve quantization time, leading to slightly faster indexing times and performance. It can also provide higher and more stable recall over time with very high dimensional embeddings. Currently available in public preview. 

## Partition key strategies for insert and query throughput

Partition key design has an outsized effect on both write throughput and vector search QPS. A poor choice creates hot partitions that cap throughput at a single physical partition's limit (~10,000 RU/s max per physical partition).

### Goals

1. **Distribute writes evenly.** Avoid hotspots where all inserts hit the same partition.
2. **Minimize cross-partition vector searches.** Target a single logical partition (or a small set of partitions) where possible to reduce RU cost and latency.
3. **Match your dominant query pattern.** If most searches filter by a specific field, that field could be a strong candidate for a partition key, or to include in a hierarchical partition key.

If the dominant filter has low cardinality, do not use it directly as the partition key unless the workload is genuinely small and has no potential for scaling to large degrees. This will help to avoid the 20GB limit on logical partitions. Prefer a synthetic key or hierarchical partition key so you preserve write distribution while still allowing efficient filtered search.

### Pattern 1: Synthetic partition key for write distribution

When the natural key has low cardinality or temporally skewed writes.

### Pattern 2: Hierarchical partition keys for multitenant workloads

[Hierarchical partition keys (HPK)](../hierarchical-partition-keys.md) solve two challenges for multitenant vector search:

1. **Large tenants exceed the 20 GB logical partition limit** when partitioned only by `tenantId`.
2. **Searching across a single tenant's data** should be efficient, not a full cross-partition fan-out.

> [!NOTE]
> If you want to use vector search on collections with hierarchical partition keys, please reach out to the team at: cosmossearch@microsoft.com to configure your account to use it optimally.

**Recommended hierarchy:**

The following plain-text example shows one way to structure a hierarchical partition key for multitenant workloads. It separates tenant scope, sub-scope, and document identity into distinct levels.

```text
Level 1: /tenantId                 → Tenant isolation
Level 2: /userId                   → Sub-scoping within tenant
Level 3: /documentId               → Fine-grained, can be GUID
```

**Scoping vector searches to the partition key:**

Scope vector searches to the **full hierarchical partition key** whenever possible. This usually targets a single logical partition and is typically the cheapest, lowest-latency search path. If the full key is not available, scope to at least the **top-level path** (e.g., `/tenantId`) so the engine prunes partitions belonging to other tenants. Cross-partition vector searches that hit every physical partition are usually the most expensive option and should be avoided in multitenant workloads.

Partition keys can be specified as a `WHERE` clause in a query or as a query request option. [Learn more here](../how-to-query-container.md)

**Choosing the right index type with HPK:**

- If each logical partition, or the subset of vectors your query targets, contains roughly **50K vectors or fewer**, a `quantizedFlat` index works well. It offers brute-force search over quantized vectors and avoids the overhead of building an ANN graph for small partitions.
- For larger partitions (> 50K vectors), use `diskANN` with `vectorIndexShardKey` set to the partition key path. This builds a separate DiskANN graph per partition key value, so scoped searches stay within that value's local graph, delivering the lowest latency and RU cost. See [Sharded DiskANN](#sharded-diskann-and-partition-key-design) for configuration details.

**Benefits of HPK for multitenant vector search:**

> [!NOTE]
> If you want to use vector search on collections with [hierarchical partition keys](../hierarchical-partition-keys.md), please reach out to team at: cosmossearch@microsoft.com to configure your account to optimally use the partitioning scheme during search.

| Benefit | Detail |
| --------- | -------- |
| No 20-GB ceiling per tenant | Each `(tenantId, category)` pair is a separate logical partition. |
| Efficient tenant-scoped search | The engine prunes partitions belonging to other tenants. |
| Natural data isolation | Tenant data is physically separated for compliance and security. |
| Flexible search scoping | Query at any HPK level - tenant-wide, category, or exact partition. |

## Sharded DiskANN and partition key design

**Sharded DiskANN** (also called partition-scoped DiskANN) builds a separate DiskANN index graph per logical partition key value rather than one global graph across the entire physical partition. This is configured via the `vectorIndexShardKey` property on the index.

### When to use Sharded DiskANN

- Your queries **always** include a filter on a specific property (for example, `tenantId`, `category`).
- You want searches scoped to a partition key value to stay entirely within that value's local DiskANN graph, avoiding scans of unrelated vectors.
- Multitenant scenarios where tenant isolation extends to the index level for performance or noisy-neighbor protection.
- You can tolerate slightly higher index build time in exchange for lower per-query RU cost and latency on scoped searches.

### Configuration

The `vectorIndexShardKey` can be the same path as the container's partition key, or the first level of a hierarchical partition key, or a separate property you wish to isolate vector search on.

The following JSON example shows a `diskANN` index that is sharded by `tenantId`. Use this when you want tenant-scoped searches to stay within a tenant-specific vector index.

```json
{
    "indexingMode": "consistent",
    "automatic": true,
    "includedPaths": [{ "path": "/*" }],
    "excludedPaths": [
        { "path": "/_etag/?" }
    ],
    "vectorIndexes": [
        {
            "path": "/embedding",
            "type": "diskANN",
            "vectorIndexShardKey": ["/tenantId"]
        }
    ]
}
```

### Sharded DiskANN with hierarchical partition keys

Suppose you want to perform vector search on individual tenants, across a group of categories. You can combine sharded DiskANN with hierarchical partitioning. Set `vectorIndexShardKey` to the first level of your hierarchical key:

The following JSON example shows a hierarchical partition key definition combined with a tenant-scoped `diskANN` index. This is useful when queries are usually scoped by tenant, but data is still subdivided further within the tenant.

```json
{
    "partitionKeyPaths": ["/tenantId", "/category"],
    "vectorIndexes": [
        {
            "path": "/embedding",
            "type": "diskANN",
            "vectorIndexShardKey": ["/tenantId"]
        }
    ]
}
```

This essentially creates one DiskANN index per `tenantId`, spanning all categories for that tenant. Queries scoped to a tenant search that tenant's graph. Queries scoped to `tenantId + category` search only the relevant subset.

## Throughput planning (RU/s)

Vector workloads can consume more RU/s than traditional point-read-heavy applications, especially during ingestion and cross-partition search when the number of partitions is large. Treat the estimates below as directional guidance only. Actual cost depends on item size, vector dimensions, numeric precision, partition fan-out, filter selectivity, and index state.

### RU cost estimates

| Operation | Approximate RU cost | Notes |
| ----------- | ------------------- | ------- |
| Point read (1 KB item by ID + PK) | ~1 RU | Cheapest operation in Cosmos DB |
| Point write / upsert (1-KB item) | ~5 RUs | Larger items cost more |
| Insert with 1536-dim `float32` vector + `quantizedFlat` | ~35 RUs | Usually lower insert overhead than DiskANN because there is no graph to maintain |
| Insert with 1536-dim `float32` vector + `diskANN` | ~65 RUs | Often higher than `quantizedFlat` |
| Vector search (`TOP 10`, `diskANN`, no filters except for a single partition key) | up to ~45 RUs | Best fit for large partitions and production workloads |
| Vector search (`TOP 10`, `diskANN`, no filters, cross-partition) | Up to ~45 RUs x N partitions hit | Fan-out dominates cost; scope by partition key whenever possible |
| Vector search (`TOP 10`, `quantizedFlat`, no filters except for partition key) | < 40 RUs (on small datasets) | Good for scoped partitions or subsets of roughly 40-50K documents or fewer |
| Vector search (with filters) | Depends on data size & filter selectivity | Selective filters and good partition scoping can reduce vector work |

> [!NOTE]
> These RU charges are only rough estimates. Actual RU charges may vary depending on container size, partitions, query filters, TOP clause values, search parameters, and other factors. Always measure actual RU consumption through the SDK's `RequestCharge` property or Azure Monitor. 

### Capacity formula

The following plain-text formula shows a simple way to estimate required RU/s from your expected read, write, and search rates. Use it as an initial sizing model before load testing.

```text
Required RU/s = (writes & deletes/sec × RU_per_write)
              + (searches/sec × RU_per_search)
              + (read/modify/other operations/sec × RU_per_operation)
```

**Example:** An application performing 100 vector searches/sec (45 RU each) and 50 inserts/sec (36 RU each) using a quantizedFlat vector index:

The following plain-text calculation applies the RU sizing formula to a specific workload. It gives you a starting point for choosing an autoscale ceiling.

```text
= (100 × 45) + (50 × 36) + 10 % buffer
= 4,500 + 1,800 + 630
= 6,930 RU/s → provision ~7,000 RU/s using Autoscale.
```

### Provisioning options

| Provisioning model | Useful scenarios |
| ------------------ | ---------------- |
| **Manual throughput** | Steady search or ingestion workloads that run all day at consistently high utilization; production workloads where you already know the RU/s envelope and want predictable always-on capacity; low-latency services where waiting for autoscale headroom changes is less desirable than holding fixed capacity. |
| **Autoscale** | Variable or bursty semantic search traffic, especially when query volume changes by hour or day; bulk vector ingestion followed by quieter search periods; new or growing applications where peak demand is still being discovered, but you still want provisioned throughput guarantees. |
| **Serverless** | Intermittent and unpredictable traffic with long idle periods, where pre-provisioning capacity would sit unused; early-stage projects, proofs of concept, and teams just getting started with Azure Cosmos DB; development, testing, prototyping, or new applications with bursty traffic, a low average-to-peak ratio, or traffic patterns you cannot forecast yet. Note that Serverless is capped at 5,000 RU/s per physical partition, so it may not be suitable for high-throughput vector insert or search workloads. |

The following C# example shows how to provision a container with autoscale throughput using the Azure Cosmos DB .NET SDK. It's useful when you want Cosmos DB to absorb bursty search or ingestion traffic without manually changing RU/s.

```csharp
// Autoscale example - scales 1,000 to 10,000 RU/s
await database.CreateContainerAsync(
    containerProperties,
    throughputProperties: ThroughputProperties.CreateAutoscaleThroughput(
        maxThroughput: 10000));
```

### Measuring actual cost

The following C# example shows how to accumulate `RequestCharge` across query pages in the .NET SDK. Use this to measure the real RU cost of a representative search workload.

```csharp
// .NET - inspect RequestCharge
using FeedIterator<SearchResult> feed = container.GetItemQueryIterator<SearchResult>(queryDef);
double totalRU = 0;
while (feed.HasMoreResults)
{
    FeedResponse<SearchResult> response = await feed.ReadNextAsync();
    totalRU += response.RequestCharge;
}
Console.WriteLine($"Vector search cost: {totalRU:F2} RU");
```

The following Python example shows how to read the request charge from Cosmos DB response headers after executing a query. Use it when you want to baseline RU cost in Python workloads.

```python
# Python - RequestCharge is in response headers
results = container.query_items(
    query=query,
    parameters=params,
    enable_cross_partition_query=True,
    populate_query_metrics=True,
)
items = [item for item in results]
print(f"Request charge: {container.client_connection.last_response_headers['x-ms-request-charge']} RU")
```

## Tips for optimizing insert throughput

| Tip | Detail |
| ----- | -------- |
| **Bulk mode** (.NET `AllowBulkExecution = true`) | Batches inserts for higher throughput. Essential for vector ingestion. |
| **`asyncio` in the Python SDK** | Use `asyncio` to parallelize inserts across partitions. The Python SDK doesn't have a built-in bulk mode, so concurrency is key for performance. See [GitHub samples](https://github.com/Azure/azure-sdk-for-python/blob/main/sdk/cosmos/azure-cosmos/samples/concurrency_sample.py). |
| **Enable autoscale** | Temporarily absorbs ingestion bursts, then scales down during search-only phases. |
| **Prefer lighter vector formats for write-heavy workloads** | `float16` and `int8` reduce storage and insert cost. Use query-time multipliers later if you need to recover recall. |
| **For very large workloads, use the Spark connector** | For large-scale backfills or distributed vector ingestion pipelines, use the [Azure Cosmos DB Spark connector](../tutorial-spark-connector.md) so writes can be parallelized across executors instead of pushed through a single application instance. |

### Enabling bulk execution (.NET, Java, JavaScript)

The Azure Cosmos DB API allows you to batch multiple document insert operations with vectors for higher throughput, which reduces round trips between the client SDK and the server and lets the vector indexer parallelize indexing across items in the batch. Enable this by setting the `AllowBulkExecution` option on the client. The SDK groups operations into batches of up to 100 operations each and dispatches them automatically.

> [!NOTE]
> The Python SDK does not support bulk execution at this time. Use the async client (`azure.cosmos.aio.CosmosClient`) with concurrent requests instead. See the async guidance in the SDK concurrency section below.

The following C# example shows how to enable bulk execution and set aggressive retry behavior for ingestion-heavy workloads in the .NET SDK. Use this client configuration when you want the SDK to batch writes and absorb throttling during large imports.

```csharp
// .NET - enable bulk execution on the client
CosmosClientOptions cosmosClientOptions = new()
{
    ConnectionMode = ConnectionMode.Direct,
    AllowBulkExecution = true,
    // Handle throttling with generous retry settings during ingestion.
    // https://learn.microsoft.com/azure/cosmos-db/nosql/how-to-migrate-from-bulk-executor-library
    MaxRetryAttemptsOnRateLimitedRequests = 100,
    MaxRetryWaitTimeOnRateLimitedRequests = TimeSpan.FromSeconds(600)
};

using CosmosClient client = new(endpoint, key, cosmosClientOptions);
```

With bulk enabled, create all insert tasks and await them concurrently so the SDK can batch and pipeline the operations:

The following C# example shows the write pattern that lets the SDK batch and dispatch item creates efficiently. It creates one task per document and then awaits them together.

```csharp
Container container = client.GetContainer("mydb", "documents");
List<Task> tasks = new();
foreach (var doc in documents)
{
    tasks.Add(container.CreateItemAsync(doc, new PartitionKey(doc.TenantId)));
}
await Task.WhenAll(tasks);
```

## SDK concurrency tuning

Vector search throughput starts at the SDK layer. For cross-partition queries, the main knob is how many partitions the SDK can query in parallel. Keep queries scoped to one partition whenever you can. When you cannot, raise the SDK's parallelism setting so one query can hit multiple partitions concurrently.

### .NET

Use `QueryRequestOptions.MaxConcurrency` for cross-partition queries. `-1` lets the SDK choose a sensible value. This is a good baseline before you start setting explicit concurrency values, but if you desire higher throughput or performance characteristics, you can try setting a specific positive value to increase parallelism beyond the SDK default.

The following C# example shows how to let the .NET SDK choose an effective concurrency level for a cross-partition vector query.  

```csharp
var query = new QueryDefinition(
    "SELECT TOP 10 c.id, VectorDistance(c.embedding, @embedding) AS score " +
    "FROM c ORDER BY VectorDistance(c.embedding, @embedding)")
    .WithParameter("@embedding", embedding);

var iterator = container.GetItemQueryIterator<SearchResult>(
    query,
    requestOptions: new QueryRequestOptions
    {
        MaxConcurrency = -1
    });

var results = await iterator.ReadNextAsync();
```

### Python

The Azure Cosmos DB Python SDK does not expose the same max-concurrency knob for a single query. When you need parallelism, issue multiple partition-scoped queries with `asyncio`, then merge and rerank the combined results. Additional information and samples about async parallelism in the Python SDK are available on [GitHub](https://github.com/Azure/azure-sdk-for-python/tree/main/sdk/cosmos/azure-cosmos).

The following Python example shows one way to run partition-scoped vector queries concurrently with `asyncio`. It gathers results from multiple partitions and merges them client-side.

```python
import asyncio
from azure.cosmos.aio import CosmosClient

QUERY = (
    "SELECT TOP 10 c.id, VectorDistance(c.embedding, @embedding) AS score "
    "FROM c ORDER BY VectorDistance(c.embedding, @embedding)"
)

async def search_partition(container, partition_key: str, embedding: list[float]) -> list[dict]:
    results = []
    query = container.query_items(
        query=QUERY,
        parameters=[{"name": "@embedding", "value": embedding}],
        partition_key=partition_key,
    )
    async for item in query:
        results.append(item)
    return results

async def search_many_partitions(partition_keys: list[str], embedding: list[float]) -> list[dict]:
    async with CosmosClient(url=ENDPOINT, credential=KEY) as client:
        container = client.get_database_client("mydb").get_container_client("documents")
        batches = await asyncio.gather(
            *(search_partition(container, partition_key, embedding) for partition_key in partition_keys)
        )
        merged = [item for batch in batches for item in batch]
        return sorted(merged, key=lambda item: item["score"])[:10]

async def main() -> None:
    results = await search_many_partitions(["tenantA", "tenantB"], embedding)
    print(results)

asyncio.run(main())
```

### Java

Use `CosmosQueryRequestOptions.setMaxDegreeOfParallelism(...)` for cross-partition queries. `-1` lets the SDK pick the effective concurrency.

The following Java example shows how to enable SDK-managed query parallelism for a cross-partition vector query. It is the Java equivalent of using automatic concurrency selection in .NET.

```java
SqlQuerySpec query = new SqlQuerySpec(
    "SELECT TOP 10 c.id, VectorDistance(c.embedding, @embedding) AS score " +
    "FROM c ORDER BY VectorDistance(c.embedding, @embedding)",
    Collections.singletonList(new SqlParameter("@embedding", embedding)));

CosmosQueryRequestOptions options = new CosmosQueryRequestOptions();
options.setMaxDegreeOfParallelism(-1);

CosmosPagedFlux<SearchResult> results =
    container.queryItems(query, options, SearchResult.class);
```

### JavaScript / TypeScript

In the JavaScript SDK, cross-partition parallelism is controlled with `maxDegreeOfParallelism` when `enableQueryControl` is enabled.

The following TypeScript example shows how to enable query control and set parallelism for a cross-partition vector query. Use this pattern when you need lower latency from the JavaScript SDK on fan-out queries.

```typescript
const querySpec = {
  query:
    "SELECT TOP 10 c.id, VectorDistance(c.embedding, @embedding) AS score " +
    "FROM c ORDER BY VectorDistance(c.embedding, @embedding)",
  parameters: [{ name: "@embedding", value: embedding }],
};

const iterator = container.items.query(querySpec, {
  enableQueryControl: true,
  maxDegreeOfParallelism: 8,
});

const { resources } = await iterator.fetchNext();
```

If a query can be scoped to a single logical partition, it is generally best to do so. Cross-partition parallelism helps latency, but it still fans out work and charges additional RUs for each partition hit.

### Choosing a concurrency value

Setting `MaxConcurrency` (or the equivalent in other SDKs) to `-1` lets the SDK pick a reasonable default. For latency-sensitive applications, you can set a specific positive value to increase parallelism beyond the SDK default:

The following C# example shows how to set an explicit concurrency target for a query in the .NET SDK. Use this when you have measured fan-out latency and want tighter control over client-side parallelism.

```csharp
var requestOptions = new QueryRequestOptions()
{
    MaxConcurrency = 64,
};
FeedIterator<IdWithSimilarityScore> queryResultSetIterator =
    container.GetItemQueryIterator<IdWithSimilarityScore>(queryDefinition, null, requestOptions);
```

### Latency best practices

A few additional practices help minimize end-to-end query latency:

- **Co-locate clients with your resources.** Ensure your application, Azure Cosmos DB account, and AI model endpoints are in the same Azure region.
- **Reuse client instances.** Create the `CosmosClient` as a singleton (one per `AppDomain` in .NET). The SDK caches connections, metadata, and session tokens that speed up subsequent calls.
- **Warm up the SDK before measuring.** The first query after process start incurs setup costs such as DNS, TCP, TLS, and service metadata. Issue a lightweight warm-up query before evaluating production latency.

## Tuning performance at search time

The `VectorDistance()` system function is the main query-time tuning surface for semantic search. Use it to control brute-force behavior, metric overrides, and candidate-list tuning.

The following SQL syntax block shows the full `VectorDistance` signature, including the optional brute-force flag and tuning object. Use it as a reference when constructing parameterized vector queries.

```sql
VectorDistance(<vector_expr_1>, <vector_expr_2>, <bool_expr>, <obj_expr>)
```

### Parameter reference

| Parameter | Type | Default | Description |
| ----------- | ------ | --------- | ------------- |
| `vector_expr_1` | array expression | *(required)* | The stored vector path, such as `c.embedding`. |
| `vector_expr_2` | array expression | *(required)* | The query vector. Parameterize this value in SDK code. |
| `bool_expr` | boolean | `false` | If `true`, forces brute-force search. If `false`, uses a vector index when one is available. |
| `obj_expr` | object literal | `{}` | Optional JSON-like object literal for query-time vector tuning. |

### Object options

| Option | Type | Description |
| -------- | ------ | ------------- |
| `distanceFunction` | string | Overrides the distance function for the query. Valid values: `Cosine`, `DotProduct`, `Euclidean`. |
| `dataType` | string | Overrides the vector data type. Valid values: `Float32`, `Float16`, `Int8`, `Uint8`. |
| `searchListSizeMultiplier` | number | Multiplier for the DiskANN search list size. Higher values usually improve recall at the cost of more RU and latency. Typical values: `5`, `10`, `20`. |
| `quantizedVectorListMultiplier` | number | Multiplier for the quantized vector candidate list. Higher values usually improve recall at the cost of more RU and latency. Typical values: `5`, `10`, `20`. |
| `filterPriority` | number | Relative priority of the `WHERE` filter versus the vector search in DiskANN. Takes any float between 0.0 and 1.0. With a higher priority the search path will bias more on filter matches. This will trade a small recall hit with fewer RUs consumed. To mitigate the impact of reduced recall, try increasing the searchListSizeMultiplier or quantizedVectorListMultiplier.|

> [!TIP]
> Keep the embedding vector parameterized, but keep the options object literal explicit in the query text. These options are tuning directives, not user input.

### Examples

#### Basic vector search (use index, policy defaults)

The following SQL example shows the simplest indexed vector search query using policy defaults. It is the baseline pattern for most application queries.

```sql
SELECT TOP 10 c.title, VectorDistance(c.embedding, @embedding) AS score
FROM c
ORDER BY VectorDistance(c.embedding, @embedding)
```

This is the most common pattern. It uses the vector index (`diskANN` or `quantizedFlat`) and the distance function configured in the container's vector embedding policy.

#### Increase DiskANN recall with `searchListSizeMultiplier`

The following SQL example shows how to raise the `searchListSizeMultiplier` for a filtered `diskANN` query. Use it when you want to improve recall before considering an index rebuild.

```sql
SELECT TOP 10 c.id, c.title,
    VectorDistance(c.embedding, @embedding, false, {searchListSizeMultiplier:10}) AS score
FROM c
WHERE c.tenantId = @tenantId
ORDER BY VectorDistance(c.embedding, @embedding, false, {searchListSizeMultiplier:10})
```

Use this when:

- You already have good partition scoping and a `diskANN` index.
- Recall is slightly low and you want to increase the ANN candidate set before rebuilding the index.
- You can tolerate moderately higher RU cost and latency per search.

#### Increase quantized candidate set with `quantizedVectorListMultiplier`

The following SQL example shows how to raise `quantizedVectorListMultiplier` for a scoped query. Use it when quantized search is fast enough, but recall needs improvement.

```sql
SELECT TOP 10 c.id, c.title,
    VectorDistance(c.embedding, @embedding, false, {quantizedVectorListMultiplier:10}) AS score
FROM c
WHERE c.category = @category
ORDER BY VectorDistance(c.embedding, @embedding, false, {quantizedVectorListMultiplier:10})
```

Use this when:

- You use `quantizedFlat`, or you rely on quantized vectors and need to recover some recall.
- The query is already scoped to a manageable subset of data.
- You want a query-time knob before changing vector precision or rebuilding the index.

#### Bias filtered DiskANN queries with `filterPriority`

The following SQL example shows how to combine `filterPriority` with another query-time tuning option in a filtered `diskANN` query. Use it when you need to rebalance scalar filtering and semantic similarity.

```sql
SELECT TOP 10 c.id, c.title,
    VectorDistance(c.embedding, @embedding, false, {filterPriority:0.5, searchListSizeMultiplier:10}) AS score
FROM c
WHERE c.tenantId = @tenantId
  AND c.category = @category
  AND c.status = 'published'
ORDER BY VectorDistance(c.embedding, @embedding, false, {filterPriority:0.5, searchListSizeMultiplier:10})
```

Use this when:

- Your query has meaningful filters and a vector search, and you want to adjust the balance between them for better recall or lower latency and RU cost.
- Your query has broad filters, and you want to experiment with how strongly the vector search should influence candidate selection.

As a practical starting point:

- Try `0.0` This effectively starts vector similarity search and does post-filtering on candidate vectors.
- Try `0.5` for a balanced starting point.
- Try `1.0` This prioritizes candidate vectors that match the filters. This tends to converge faster and has lower RU costs.

## Monitoring and diagnostics

### Key metrics to watch

| Metric | Where to find it | What it tells you |
| -------- | ----------------- | ------------------- |
| **Normalized RU Consumption** | Azure Monitor | % of provisioned throughput used. > 80 % → scale up or optimize. |
| **Total Request Units** | Azure Monitor / SDK | Actual RU spend per query. Use for capacity planning. |
| **429 (Throttled) requests** | Azure Monitor | You're exceeding provisioned RU/s. Scale up or optimize queries. |
| **P99 server-side latency** | Azure Monitor | Track vector search latency. Spikes can indicate hot partitions or index build lag. |
| **Partition fan-out** | Query shape + diagnostics | If a query isn't scoped to a partition key, fan-out often dominates both RU cost and latency. |

### Track RU per query in code

Always log the `RequestCharge` for vector search queries during development to baseline costs:

The following C# example shows how to log per-page diagnostics and total RU charge in the .NET SDK. Use it to validate both query cost and execution behavior during testing.

```csharp
// .NET
using FeedIterator<SearchResult> feed = container.GetItemQueryIterator<SearchResult>(queryDef);
double totalRU = 0;
while (feed.HasMoreResults)
{
    FeedResponse<SearchResult> response = await feed.ReadNextAsync();
    totalRU += response.RequestCharge;
    // Log diagnostics for troubleshooting
    Console.WriteLine(response.Diagnostics.ToString());
}
Console.WriteLine($"Total RU: {totalRU:F2}");
```

The following Python example shows the async paging pattern for iterating query results while retaining access to request metadata. Use it to instrument cost and latency in Python-based tests.

```python
# Python (async)
async for page in container.query_items(
    query=query,
    parameters=params,
    enable_cross_partition_query=True,
).by_page():
    for item in page:
        pass  # process items
    # Access request charge from page response headers
```

## Related content

- [Vector search overview](vector-search-overview.md)
- [VectorDistance system function](/cosmos-db/query/vectordistance)
- [DiskANN + Azure Cosmos DB—Microsoft Mechanics Video](https://www.youtube.com/watch?v=MlMPIYONvfQ)
- [Hierarchical partition keys](../hierarchical-partition-keys.md)
- [Autoscale throughput](../provision-throughput-autoscale.md)
- [.NET SDK vector search how-to](../how-to-dotnet-vector-index-query.md)
- [Python SDK vector search how-to](../how-to-python-vector-index-query.md)
- [Java SDK vector search how-to](../how-to-java-vector-index-query.md) 
- [JavaScript SDK vector search how-to](../how-to-javascript-vector-index-query.md)


