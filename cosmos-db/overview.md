---
title: Cosmos DB Overview
description: Learn about Cosmos DB, a distributed NoSQL database for low-latency apps. It offers elastic scaling, global replication, and AI capabilities in Azure and Fabric.
ms.date: 05/27/2026
ai-usage: ai-generated
---

# What is Cosmos DB (in Azure and Fabric)

Cosmos DB is a distributed NoSQL database engine built for predictable low latency, elastic horizontal scale, and global availability. Use Cosmos DB to store semi-structured JSON data with automatic indexing, fast querying, and support for SQL-like queries, geospatial operations, full-text, and vector search. This overview explains the core capabilities, design goals, and how Cosmos DB helps you build globally distributed, low-latency applications in Azure and Microsoft Fabric.

## Core design goals and capabilities

The engine handles flexible, nested JSON documents without predefining a schema, enabling schema-agnostic storage and iteration. The engine copies data across different regions. This setup routes requests to the closest region for fast reads. The system scales by splitting data into logical partitions. These partitions map to physical partitions. This design lets containers scale throughput and storage on their own.

Applications can choose from multiple consistency models to trade off latency and correctness. The engine uses a Request Unit (RU) model that provides predictable throughput and cost abstraction for reads, writes, and queries. The engine indexes all data automatically by default. You can create custom indexing policies to optimize query performance. These policies support range indexes, spatial indexes, composite indexes, and vector indexes. The rich query engine supports declarative SQL-like queries, aggregates, scalar functions, and integration with other APIs built on the same engine.

## Common operational concerns

Avoid hot partitions and stay within logical partition limits by choosing a partition key that balances data distribution and query patterns. For indexing, rely on the default all-properties indexing for fast development, then narrow indexing policies or add composite and vector indexes to optimize costs and query latency for production workloads. Pick a consistency level appropriate for your correctness and latency requirements. Session is a common default for many applications.

## Scenarios

The Cosmos DB engine is designed for low-latency globally distributed applications such as gaming, e-commerce, and IoT ingestion. The engine supports real-time analytics and hybrid search. It uses built-in full-text and vector search features. You can use it as a base for AI and ML feature stores. It also works well for embedding indexes. This flexibility is especially true when you use Fabric connections.

## Implementations

The Cosmos DB engine is implemented in two services that share the same core technology while providing different operational models and integration capabilities.

### Azure Cosmos DB

Azure Cosmos DB is a fully managed cloud database service that uses the Cosmos DB engine to provide support for querying items with flexible schemas and native support for JSON. It offers global distribution with multi-region replication, allowing applications to achieve low-latency reads and writes across geographic regions. The service provides fine-grained control over throughput provisioning, indexing policies, and consistency levels, enabling you to optimize performance and cost for your specific workload. Azure Cosmos DB integrates with Azure services. The service supports software development kits (SDKs) for .NET, Java, Python, Node.js, and Go. This compatibility makes it suitable for mission-critical applications. These applications require predictable performance and high availability.

For more information about Azure Cosmos DB, see [Azure Cosmos DB documentation](/azure/cosmos-db).

### Cosmos DB in Microsoft Fabric

Cosmos DB in Microsoft Fabric is an AI-optimized NoSQL database with a simplified management experience that uses the same Cosmos DB engine and infrastructure. Cosmos DB in Fabric is tightly integrated into Fabric, providing autonomous defaults optimized for most application workloads and eliminating typical database management tasks. Data in Cosmos DB appears automatically in Fabric OneLake. The data uses Delta Parquet format. This format enables analytics that run in near real-time. You can run queries that execute across different databases. You can create Power BI visualizations. The service integrates with data science tools. These tools include notebooks and Lakehouse. The service includes built-in AI features. These features include full-text search, hybrid search, and vector indexing. These tools make it easy to build AI applications. You can build with less friction while you keep the flexible data model. The service maintains automatic scaling and fast performance that the core engine provides.

For more information about Cosmos DB in Microsoft Fabric, see [Cosmos DB in Microsoft Fabric](/fabric/database/cosmos-db).
