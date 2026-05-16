---
title: Aggregation pipeline examples and limitations
titleSuffix: Azure Cosmos DB for MongoDB
description: Learn how to use the MongoDB aggregation pipeline in Azure Cosmos DB for MongoDB, including common examples, $lookup cross-collection join limitations, and workarounds for cross-collection aggregation scenarios.
author: gahl-levy
ms.author: gahllevy
ms.service: azure-cosmos-db
ms.subservice: mongodb
ms.topic: tutorial
ms.date: 05/13/2026
ai-usage: ai-assisted
appliesto:
  - ✅ MongoDB
---

# Aggregation pipeline examples and limitations

[!INCLUDE[Note - Recommended services](includes/note-recommended-services.md)]

The aggregation pipeline lets you perform advanced data analysis and transformation on your collections in Azure Cosmos DB for MongoDB. A pipeline is a sequence of stages, each of which filters, reshapes, or computes values on the documents flowing through it. This article shows common single-collection pipeline patterns, explains limitations around cross-collection aggregation with `$lookup`, and suggests workarounds when native aggregation isn't sufficient.

## Prerequisites

- An Azure Cosmos DB for MongoDB account. If you don't have one, [create an account](connect-account.yml).
- A MongoDB-compatible client or shell connected to your account.

## Basic syntax

```javascript
db.<collection>.aggregate([
    { <stage1> },
    { <stage2> },
    // ...
    { <stageN> }
])
```

Each stage receives the documents output by the previous stage and passes its own output to the next.

## Single-collection pipeline examples

The examples in this section use a `sales` collection with documents in the following format:

```json
{
  "_id": "ord-001",
  "date": "2024-03-15",
  "category": "electronics",
  "item": "laptop",
  "price": 1200,
  "quantity": 2,
  "tags": ["sale", "featured"]
}
```

### Filter documents with $match

Use `$match` to pass only documents that satisfy a condition. Place `$match` as early as possible in the pipeline so that it can use indexes and reduce the number of documents processed by later stages.

```javascript
db.sales.aggregate([
    { $match: { category: "electronics", price: { $gt: 500 } } }
])
```

### Group and compute totals with $group

Use `$group` to group documents by a field and apply accumulator expressions such as `$sum`, `$avg`, `$min`, and `$max`.

```javascript
db.sales.aggregate([
    {
        $group: {
            _id: "$category",
            totalRevenue: { $sum: { $multiply: ["$price", "$quantity"] } },
            orderCount: { $sum: 1 },
            avgPrice: { $avg: "$price" }
        }
    }
])
```

### Shape output with $project

Use `$project` to include, exclude, or compute fields in the output documents. Set a field to `1` to include it, `0` to exclude it, or provide an expression to compute a new value.

```javascript
db.sales.aggregate([
    {
        $project: {
            item: 1,
            price: 1,
            revenue: { $multiply: ["$price", "$quantity"] },
            discounted: { $cond: [{ $gt: ["$price", 1000] }, true, false] }
        }
    }
])
```

### Sort results with $sort

Use `$sort` to order documents by one or more fields. Use `1` for ascending order and `-1` for descending order.

```javascript
db.sales.aggregate([
    { $sort: { price: -1, date: 1 } }
])
```

### Limit results with $limit and $skip

Use `$limit` to cap the number of output documents. Use `$skip` to bypass a specified number of documents, which is useful for pagination when combined with `$sort`.

```javascript
db.sales.aggregate([
    { $sort: { date: -1 } },
    { $skip: 20 },
    { $limit: 10 }
])
```

### Flatten arrays with $unwind

Use `$unwind` to deconstruct an array field so that each element produces a separate output document.

```javascript
db.sales.aggregate([
    { $unwind: "$tags" },
    { $group: { _id: "$tags", count: { $sum: 1 } } },
    { $sort: { count: -1 } }
])
```

### Add computed fields with $addFields

Use `$addFields` to add new fields to documents without having to respecify all existing fields, unlike `$project`.

```javascript
db.sales.aggregate([
    {
        $addFields: {
            totalValue: { $multiply: ["$price", "$quantity"] },
            year: { $substr: ["$date", 0, 4] }
        }
    }
])
```

### Multi-stage pipeline example

The following pipeline combines several stages to find the top five product categories by total revenue for the first quarter of 2024:

```javascript
db.sales.aggregate([
    // Stage 1: filter to Q1 2024
    { $match: { date: { $gte: "2024-01-01", $lt: "2024-04-01" } } },

    // Stage 2: compute per-document revenue
    { $addFields: { revenue: { $multiply: ["$price", "$quantity"] } } },

    // Stage 3: group by category and sum revenue
    { $group: { _id: "$category", totalRevenue: { $sum: "$revenue" } } },

    // Stage 4: sort by revenue descending
    { $sort: { totalRevenue: -1 } },

    // Stage 5: return top 5 only
    { $limit: 5 },

    // Stage 6: rename _id for readability
    { $project: { _id: 0, category: "$_id", totalRevenue: 1 } }
])
```

## Cross-collection aggregation with $lookup

The `$lookup` stage performs a left outer join between the current collection (local) and another collection (foreign) in the same database. The following example joins `orders` to a `customers` collection using a matching field.

```javascript
db.orders.aggregate([
    {
        $lookup: {
            from: "customers",
            localField: "customerId",
            foreignField: "_id",
            as: "customerInfo"
        }
    },
    { $unwind: "$customerInfo" },
    {
        $project: {
            orderId: "$_id",
            customerName: "$customerInfo.name",
            total: 1
        }
    }
])
```

### $lookup limitations in Azure Cosmos DB for MongoDB

`$lookup` is partially supported in Azure Cosmos DB for MongoDB. The following limitations apply across all supported server versions:

- **Uncorrelated subqueries aren't supported.** The extended `$lookup` syntax introduced in MongoDB 3.6 that uses `let` and `pipeline` fields isn't supported. Using those fields returns the error "`let` isn't supported." Use the basic `from` / `localField` / `foreignField` / `as` form instead.

  ```javascript
  // This syntax is NOT supported:
  {
      $lookup: {
          from: "inventory",
          let: { ordItem: "$item" },
          pipeline: [ { $match: { $expr: { $eq: ["$sku", "$$ordItem"] } } } ],
          as: "inventoryDocs"
      }
  }
  ```

- **`$graphLookup` isn't supported** in API versions 4.2, 6.0, and 7.0. If your workload requires recursive or graph traversal queries, see [Workarounds for cross-collection scenarios](#workarounds-for-cross-collection-scenarios).
- **Both collections must reside in the same database.** Cross-database `$lookup` isn't supported.
- **Performance considerations.** Because Azure Cosmos DB for MongoDB is a distributed system, `$lookup` joins that span large collections or collections in separate logical partitions can consume significant request units (RUs). Evaluate your RU consumption with the [capacity planner](estimate-ru-capacity-planner.md) and consider denormalizing data to reduce join frequency.

## Cross-collection aggregation limitations

Azure Cosmos DB for MongoDB is optimized for high-throughput single-collection access patterns. When you need to aggregate data across multiple collections, keep the following constraints in mind:

| Capability | Support |
| --- | --- |
| Basic `$lookup` (localField / foreignField) | ✅ Yes |
| `$lookup` with `let` and `pipeline` (uncorrelated subqueries) | ✖️ No |
| `$graphLookup` (graph traversal) | ✖️ No (versions 4.2–7.0) |
| `$lookup` across databases | ✖️ No |
| Transactions spanning multiple collections | ✖️ No (multi-document transactions are limited to a single nonsharded collection) |

These constraints apply regardless of which server version (3.6 through 7.0) your account uses.

## Workarounds for cross-collection scenarios

When native aggregation pipeline capabilities aren't sufficient for your cross-collection or export scenario, consider the following alternatives.

### Application-side aggregation

Query each collection independently, then combine and aggregate the results in your application. This approach avoids `$lookup` limitations and gives you full control over join logic.

```javascript
// Query collection A
const orders = await db.collection("orders").find({ status: "completed" }).toArray();
const customerIds = orders.map(o => o.customerId);

// Query collection B with matching IDs
const customers = await db.collection("customers")
    .find({ _id: { $in: customerIds } })
    .toArray();

// Join in application memory
const customerMap = new Map(customers.map(c => [c._id.toString(), c]));
const enriched = orders.map(o => ({
    ...o,
    customer: customerMap.get(o.customerId.toString())
}));
```

This pattern is most suitable for small-to-medium result sets. For large exports, use one of the Azure service integrations described next.

### Azure Synapse Link for analytics

[Azure Synapse Link for Azure Cosmos DB](../configure-synapse-link.md) automatically synchronizes your operational data to Azure Synapse Analytics without impacting your transactional workloads. In Azure Synapse, you can run cross-collection analytical queries using Apache Spark or serverless SQL pools, which support full join semantics.

Use Azure Synapse Link when you need:

- Complex analytical queries across multiple collections.
- Historical trend analysis or reporting without impacting production RU consumption.
- Near-real-time analytics with minimal ETL overhead.

### Azure Data Factory for export and transformation

[Azure Data Factory](https://azure.microsoft.com/services/data-factory/) can read from multiple Azure Cosmos DB for MongoDB collections, apply transformations in data flows, and write the results to a destination such as Azure Blob Storage, Azure SQL Database, or Azure Synapse Analytics.

Use Azure Data Factory when you need:

- Scheduled batch export of data from one or more collections.
- Cross-collection joins as part of an ETL pipeline.
- Integration with downstream systems that don't connect directly to Azure Cosmos DB.

## Related content

- [Supported features and syntax in Azure Cosmos DB for MongoDB 7.0](feature-support-70.md)
- [Configure Azure Synapse Link for Azure Cosmos DB](../configure-synapse-link.md)
- [Query data with Azure Cosmos DB for MongoDB](tutorial-query.md)
- [Troubleshoot query performance in Azure Cosmos DB for MongoDB](troubleshoot-query-performance.md)
- [Estimate request units with the Azure Cosmos DB capacity planner](estimate-ru-capacity-planner.md)
